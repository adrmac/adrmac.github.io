---
title: "Serverless deployment with Cloud Run - repos, containers, and webpacks"
date: 2026-03-06
---

As a front-end / UX engineer I'm generally famiilar with the concept of serverless architecture, but in a contract interview recently I realized I could stand to do a deeper dive. I set up a Python API [esp32_ui](https://github.com/postoccupancy/esp32_ui) / [esp32_api](https://github.com/postoccupancy/esp32_api) earlier this year, and deployed it on Google Cloud Run, a serverless platform. I'll walk through some of the finer points and write up the contextual research I'm doing to understand it better.

A quick definition -- "serverless" in a cloud architecture context means a backend API that is hosted in such a way that it doesn't require a server running 24-7. Serverless processes can be "spun up" on demand, then go back to zero, after storing whatever data they might need for another time in a separate database. This makes them "stateless," as opposed to "stateful." The tradeoff between serverless vs. always-on virtual machine is essentially about how frequently processes are run, how much persistent storage or configuration is needed, and whether it makes more sense to pay for cloud compute on-demand or as an indefinite subscription.

These are mostly backend decisions, and as a front-end/UX engineer, the interface for a serverless API may not differ much from what I'm used to. It's important to know that serverless platforms can have a 'cold start' when spinning up for the first time. If the UI is slow, is it your React code, or is it the serverless container spinning up? Are you accounting for that in the UX loading states? For a prolonged user session, do you need to keep the container warm?


**About containers**

Let's talk about what a container actually is. The first question in the Cloud Run deployment UI is whether to 'Connect a repository,' or 'Deploy a container' -- either way, the app will deploy inside a container. 

A container is an encapsulated runtime environment with all the necessary system dependencies for an app to run. 

In my case, I'm going for development velocity and an iterative source-to-production workflow. I don't want to plow through a lot of config to get my app running. I want to connect my GitHub repo directly. This means Cloud Run will builds a container for me, and rebuild it whenever I push new updates to my GitHub repo.

The alternative would be to deploy from an existing container, which I might need to do if I had a more complex app requiring a specific system environment. Right now, creating a container environment for my esp32_api app is optional--I'm using standard Python packages that can be readily installed from many different OS systems. The [Orcasound](https://github.com/orcasound/orcasite) app has a container to specifically run Linux OS, Elixir, Node, Python, Postgres, and various other system libraries, to make sure that all developers are working with the same toolset. A large company might containerize for security and compliance. 

Just because Orcasound has a container doesn't mean it would be deployable on Cloud Run -- it wouldn't be, because it's an always-on server (deployed on Heroku, see note below) -- but it is an example of an app I work on inside a container. If I ever needed to deploy a container to Cloud Run, I would need to create it and store to a registry like the Google Artifact Registry. 

**About virtual machines**

Virtual machines are cloud servers. When companies say they are going "serverless" they might mean they are moving from an on-premises server to the cloud. But in fact this is still a "serverful" architecture from a developer's perspective.

Orcasound has a serverful architecture using Heroku. This is a managed platform that runs on top of Amazon Web Services (AWS) EC2, a large virtual machine. Each Heroku customer gets their own container called a Dyno that operates like an always-on VM, while under the hood it shares a large VM with other customers. Each Dyno boots up a full Linux system environment, so it can't start and stop instantly like a serverless container. 

**About serverless web servers**

Instead of a VM, Cloud Run uses a proprietary stack where each container lives in its own Google-managed gVisor sandbox. Google keeps a massive pool of "warm" resources ready to execute code at a millisecond's notice, as long as the code is "stateless" and can go back to nothing. AWS App Runner is a direct competitor to Cloud Run, although it lacks a 'zero' state and requires at least one instance. 

These platforms are for deploying an entire "serverless web server" like a Python API. This phrase is not a contradiction, because "serverless" refers to the hosting platform, and "web server" refers to the code. 

**About serverless functions**

There is another class of platform for "serverless functions" -- including AWS Lambda, or function targets inside of Cloud Run. These are designed for short-lived, event-driven tasks -- so could be an option for ESP32 sensor posting data.

**About cold starts**

Because my esp32_api serverless web server receives data every 2 seconds, it most likely stays alive and never spins down -- idle time is about 15 seconds. This means the React UI might benefit from accessing an already-live container without cold start -- until there is enough traffic that Cloud Run needs to cold-start a new container to handle concurrent load.

To monitor cold starts in GCP:
- Cloud Run Metrics Tab: Look at the "Container Instance Count" graph. If you see the line drop to 0 and then jump to 1, the first request at that "jump" was a cold start.
- Cloud Logging: Search your logs for the phrase `Defaulting to latest available Python runtime`. This line appears during the build/start phase. More importantly, look for requests with a very high latency compared to others. A typical request might take 50ms, while a cold start might take 2–5 seconds.

There is also a trick for ensuring the container never cold starts -- set 'minimum instances' to 1 so it never scales to 0. This results in a small minimum monthly fee ($1-5/mo). AWS App Runner defaults to 1 and doesn't allow 0.

**About Dockerfiles**

Cloud Run asks whether to build the container based on a Dockerfile I provide, or to use a default Python webpack provided by Google. 

A Dockerfile is a set of instructions for how to build a container -- specifying an exact OS, Python version, dependency installation command (e.g. `pip install -r requirements.txt` is the default), and system-level libraries like maybe `ffmpeg` for audio processing or `libpq` for databases that a standard buildpack might miss.

Right now I don't have specific system dependencies beyond the Python version. I'm not currently doing any audio processing, although I might want that in the future. For now, I want to keep it simple. So I take the webpack route and accept the defaults. 

**About webpacks**

Even with the default Python webpack, there are some assumptions and gotchas that are good to be aware of. 

1. The Python version -- this one slipped past me as a backend noob. While I was familiar with the convention of documenting Python package dependencies in a `requirements.txt` file, I didn't realize that this only captures Python packages, not the Python version. For this, Cloud Run looks for a separate file called `.python-version` -- which is exactly the file that the `pyenv` version manager generates when setting a directory-level version via `pyenv local [version]`. (Cloud Run would also recognize a `GOOGLE_PYTHON_VERSION` environment variable. Otherwise, it defaults to the most recent version of Python.)

2. Webpack inputs -- the webpack UI requests inputs that look a little arcane until you understand what it's asking.
- Build context directory -- this is just the application root, e.g. `/` but in my case it is `/server` 
- Entrypoint -- they want the command used to start the server. You can leave this blank, which is safest, because herein lies another gotcha. Locally, I go to the `/server` directory and run `uvicorn app.main:app --host 0.0.0.0 --port 8000` because I want to ensure the exact host and port I'm running on. But Cloud Run runs on port 8080, so specifying 8000 would lead to an error. The default is `uvicorn app.main:app` so leaving it blank works fine (see 'About entrypoints' below). 
- Function target -- this is only for 'serverless function' deployments. Leave blank for a web server.

**About entrypoints**

You can manually set the entrypoint (the command used to start the app) in the Cloud Run UI or in a `Procfile`, or leave it blank to have Cloud Run detect it automatically. The entrypoint detection process will wrap the app code in one of two kinds of Python web server interfaces:
1. Gunicorn = a WSGI (web server gateway interface), the traditional standard for Python, used by Flask/Django
2. Uvicorn = an ASGI (asynchronous server gateway interface), the newer standard used for high-performance, asynchronous apps like FastAPI

If Cloud Run detects FastAPI, it will assume an ASGI environment and use Uvicorn. If not, and no other framework default is specified, it defaults to Gunicorn.


**About virtual environments (venv)**

A venv is used in Python development, not so much in production deployment. But I was a little hazy about the difference between a venv and a container so needed to clarify. 

Basically a venv is a Python-native feature that only captures Python dependencies, while a container is a snapshot of an entire system environment including OS and system-level dependencies. 

The Python version is a system-level dependency. When you activate a venv it will _try_ to use the same version of Python it was created with, but it doesn't bring the Python installation with it, just the Python packages. 

In production, Cloud Run installs dependencies from requirements.txt directly to the container environment, so no venv involved. 


**About Docker Compose**

The Orcasound app has a Dockerfile, but it also has a Docker Compose setup. This is because it has actually has multiple containers working together:
- web = app container
- db = PostGIS database container. PostGIS is an extension installed inside the Docker container that adds geospatial capabilities to PostgreSQL. 
- cache = Redis container. Redis (Remote Dictionary Server) is an exceptionally fast in-memory data store that keeps data in RAM rather than on a physical disk.

The Redis cache likely handles Websocket connections for the live audio streams. For ESP32, it could be a way to show a live temperature reading without querying the main Postgres db every time. You would post the latest reading to Redis for the UI to grab instantly.

Digging into these should be another post. But these are the relevant files:
1. `.devcontainer/devcontainer.json` - This triggers VS Code to automatically set up a development container when opening the repo. It points to three docker-compose.yml files that get merged together.
2. `docker-compose.yml` -- the base setup
3. `docker-compose.dev.yml` -- overrides for things needed for local dev
4. `../docker-compose.yml` -- VS Code-specific settings in the `.devcontainer` folder, lets VS Code attach to the container as a development shell
5. `Dockerfile` - defines how to build a container image. An image is a frozen snapshot of a filesystem and runtime. It is a template for creating containers. Docker Compose creates containers from images and defines how containers work together. 


