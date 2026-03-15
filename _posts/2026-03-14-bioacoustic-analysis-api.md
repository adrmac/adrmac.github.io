---
title: "My next API - bio-acoustic analysis of Orcasound hydrophones"
date: 2026-03-14
---

My next project is to build on what I've learned in the ESP32 API project to set up a new FastAPI for Orcasound. I don't need to invent any new bio-acoustic analysis code -- that has all been created already by Masters of Data Science (MSDS) students at the University of Washington. 

The `orcasound/ambient-sound-analysis` repo they built is a reusable Python package that provides: 
- `orcasound/orca-hls-utils` - package for fetching Orcasound HLS `.ts` audio clips from AWS S3 object storage

- `ffmpeg` calls for converting the clips into `.wav` and computes Power Spectral Density (PSD) data -- essentially a numeric matrix of average noise levels at all frequencies over 1-second intervals

- `boto3` calls for storing the PSD `.parquet` files back to the same AWS S3 bucket

- a custom `NoiseAccessor` that reads the archived PSD data and assembles dataframes for requested time windows and resolutions

- visualization utilities: `plot_spec` for generating spectrograms, and `plot_bb` for broadband (all frequencies) noise time series

- `daily_noise.py` summary metrics and trends

- a Streamlit dashboard that can be run locally

- AWS Batch support `ec2_batch` -- tooling for batch processing over long time ranges

### My mission

The data science team has built the functionality, so what remains is for a web developer like me to make it accessible -- by setting up a hosted API that exposes HTTP endpoints for calling the NoiseAccessor, plots, and daily noise summary on live data from the Orcasound Next/React interface. 

My goal is to add a new level of scientific information to the Orcasound hydrophone user experience that significantly broadens the platform's usefulness and appeal. 

### Step 1: Setting up a Docker container

First off, I need to set up a new runtime environment for my API, which I intend to eventually deploy to a production container on Cloud Run. 

The ambient-sound-analysis repo is set up as a Python package, as evidenced by the presence of the `pyproject.toml` and `setup.py` files. This means I can install it to my API container environment with `pip install` and it will bring along its own Python package dependencies like `pandas`, `librosa`, or `orca-hls-utils` (from `ambient-sound-analysis/requirements.txt`). It has two system-level dependencies that I need to install separately -- these are:

1. `ffmpeg` - audio processing package for converting HLS to WAV - this isn't actually necessary for my API because I'm only planning on fetching the processed PSD data. The conversions are a separate process, maybe run once to fill out the full data archive, and automated on some interval to update it from live streams.

2. `Python 3.9` - this is the trickiest part of the whole setup, because the package's Python dependencies break with more recent Python versions. I cannot, for example, reuse the standard Orcasound devcontainer environment because it only has Python 3.12, and the package doesn't install successfully. My choices are either to pin my API to 3.9, or try to update the dependencies. 

As always, I want to keep things simple, so am just going to stick with 3.9 for this API for now. I am creating a `Dockerfile` that specifies this for my dev container. 

Alternatively, I could do what I did last time for `esp32_api` -- use `pyenv` to set a local Python version in my working directory, and create a `venv` (virtual environment) to install Python dependencies from `requirements.txt` for development. The `Dockerfile` isn't absolutely necessary for this project, but I'm doing it anyway to get more comfortable. 

The `Dockerfile` requires unique Docker syntax but can be relatively easily mastered. It has a few key sections:

1. `FROM` - selects the base image the container will start from. The `python3.9-slim` variant is a smaller Debian Linux image. Later commands use the Debian package manager, `apt-get`. Debian is an open-source operating system (OS) based on the Linux kernel.
```Dockerfile
FROM python:3.9-slim
```

2. `RUN` - run shell commands during the image build process to install system level dependencies. Here I am taking a few suggestions from Codex for the baseline tools.  
```Dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    git \
    curl \
    && rm -rf /var/lib/apt/lists/*
```
   - `apt-get update` downloads the latest list of available Debian packages

   - `apt-get install` installs specific Linux tools into the image. In this case:

       - `build-essential` adds basic compilation tools like `gcc` and `make`

       - `git` lets the container pull code or dependencies from Git repos

       - `curl` is a simple command-line networking utility

       - Later add `ffmpeg` if needed

The flags mean:

   - `-y` automatically answers "yes" to install prompts

   - `--no-install-recommends` avoids pulling in extra optional packages

At the end, `rm -rf /var/lib/apt/lists/*` deletes the downloaded package lists after installation. Those are only needed during the image build, so deleting them afterward makes the final image smaller. 


3. `WORKDIR` - set the default working directory inside the container. 

For the production `Dockerfile`, the conventional package folder name is `/src` or `/app`.
```Dockerfile
WORKDIR /app
```

4. `COPY` / `RUN` - `COPY` moves files from the repo into the image, and `RUN` runs them. 

```Dockerfile
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .
```

The last part `COPY . .` is huge, it means copy everything from the current build context on your machine (the first `.` -- e.g. the repo root) into the current working directory inside the image (the second `.` -- this is the `WORKDIR` set above)

The common pattern is to copy and install `requirements.txt` first, so Docker caches this separately. If the app code changes but `requirements.txt` does not, this allows Docker to skip reinstalling everything from scratch. 


5. `CMD` - defines the default command that runs when the container starts. 
```Dockerfile
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8080"]
```

   - `app.main:app` is standard Python syntax -  "import the `app` object from the `main.py` file inside the `app/` package." 

   - `--host 0.0.0.0` means "listen on all network interfaces inside the container," not just on `localhost`. The app needs to be reachable from outside the container, including by Cloud Run or VS Code port forwarding.

   - `--port 8080` -- this is the port that Cloud Run uses by default, so planning ahead using it here.


### Step 2: Setting up a local devcontainer in VS Code

This part was nightmarishly error-prone to get set up correctly and took several tries. The goal is to have a Python 3.9 container running in VS Code, with both `ambient-sound-analysis-api` and `ambient-sound-analysis` side by side in the same workspace. A major gotcha was to make sure and to it exactly in this sequence:
1. Open one base repo
2. Reopen that repo in a container
3. Add other folders to the workspace from inside the container
4. Save the workspace as a `.code-workspace` file

Do not try to do something crazy like adding folders to the workspace before starting the container.

To make VS Code open a repo in a local dev container, it needs to see a `.devcontainer/` folder with two files at a minimum:

1. `.devcontainer/devcontainer.json` -- configuration file that looks like this:

```json
{
  "name": "ambient-sound-analysis-api",
  "build": {
    "dockerfile": "./Dockerfile", // note this is ./ not ../ 
    "context": ".." // this is ../, it allows the Dockerfile to access the root
  },
  "workspaceFolder": "/workspaces/orcasound/ambient-sound-analysis-api", // the workspace needs a default folder
  "workspaceMount": "source=${localWorkspaceFolder}/../..,target=/workspaces,type=bind", // this is the critical part -- it tells VS Code to create a 'workspaces' folder in the container, and give it a 'source' to access -- the localWorkspaceFolder is where the container was created (needs a .devcontainer/), and '${localWorkspaceFolder}/../..' gives access to the parent directory two levels up
  "customizations": {
    "vscode": {
      "extensions": [ // these are the VS Code extensions that should be installed in the container by default
        "ms-python.python",
        "ms-python.vscode-pylance",
        "ms-azuretools.vscode-docker",
        "charliermarsh.ruff"
      ]
    }
  },
  "postCreateCommand": "cd /workspaces/orcasound/ambient-sound-analysis-api && python -m pip install --upgrade pip && python -m pip install -r requirements.txt", // this installs the requirements after the base image is created, instead of in the Dockerfile, so you can pip install new packages from the terminal without rebuilding the container
  "remoteUser": "root"
}
```

- `.devcontainer/Dockerfile` -- this is similar to the production `Dockerfile` above, but sets the `WORKDIR` to the `/workspaces` folder. It skips installing dependencies from `requirements.txt` because not all repos have the same ones. We also don't need to start the web server on container build. 

```Dockerfile
FROM python:3.9-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    git \
    curl \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /workspaces
```

### Step 3: Install dependencies

The `ambient-sound-analysis-api` repo has a short `requirements.txt` file:

```
fastapi
uvicorn[standard]
orcasound_noise @ git+https://github.com/orcasound/ambient-sound-analysis.git

```

This is one of the benefits of having `ambient-sound-analysis` as an installable package -- it brings all of its other dependency packages (`librosa`, `matplotlib`, `orca-hls-utils`, etc) with it. Good thing we installed `git` in the container. 


