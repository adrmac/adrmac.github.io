---
title: "Serverless deployment with Cloud Run - repos, containers, and webpacks"
date: 2026-03-06
---

I had a job interview last week where I took some questions on serverless architecture. As a front-end / UX engineer I'm generally famiilar with the concept, and when I set up a Python API [esp32_api](https://github.com/postoccupancy/esp32_ui) / [esp32_api](https://github.com/postoccupancy/esp32_api) earlier this year, I deployed it on Google Cloud Run, a serverless platform. But I haven't done as much of a deep dive into the details and tradeoffs of it as maybe I should. So here are some takeaways from my experience with serverless cloud architecture so far. 

A quick definition -- "serverless" in a cloud architecture context means a backend API that is hosted in such a way that it doesn't require a server running 24-7. Serverless processes can be "spun up" on demand, then go back to zero, after storing whatever data they might need for another time in a separate database. This makes them "stateless," as opposed to "stateful." The tradeoff between serverless vs. always-on virtual machine is essentially about how frequently processes are run, how much persistent storage or configuration is needed, and whether it makes more sense to pay for cloud compute on-demand or as an indefinite subscription.

These are mostly backend decisions, and as a front-end/UX engineer, the interface for a serverless API may not differ much from a server. But it's important to know that serverless platforms can have a 'cold start' when spinning up for the first time. If the UI is slow, is it your React code, or is it the serverless container spinning up? Are you accounting for that in the UX loading states? For a prolonged user session, do you need to keep the container warm?

I will also walk through the process of setting up a serverless container in Cloud Run, explaining concepts along the way.

**About containers**

The first question in the Cloud Run deployment UI is whether to 'Connect a repository,' or 'Deploy a container' -- either way, the app will deploy inside a container. 

A container is an encapsulated runtime environment with all the necessary system dependencies for an app to run. 

In my case, I'm going for development velocity and an iterative source-to-production workflow, so I want to connect my GitHub repo directly. This means Cloud Run automatically builds a container for me, and rebuilds it whenever I push new updates.

The alternative would be to deploy from an existing container, which I might do if I had a more complex app requiring a specific system environment. Right now, containerization for my esp32_api app is optional--I'm using standard Python packages that can be readily installed--but might be convenient depending on who I'm working with. The [Orcasound](https://github.com/orcasound/orcasite) app is containerized to specifically run with Linux OS, Elixir, Node, Python, Postgres, and various other system libraries, to make sure that all developers are working with the same toolset. A large company might containerize to ensure security and compliance. 

This doesn't mean the Orcasound container would necessarily be deployable in Cloud Run -- it wouldn't be, because it's an always-on server (deployed on Heroku, see note below) -- but it is an example of an app with an existing container. If I went this route, I would need to first store the container to a registry like the Google Artifact Registry. 

**About virtual machines**

Virtual machines are cloud servers. When companies say they are going "serverless" they might mean they are moving from an on-premises server to the cloud, while still using a "serverful" architecture.

Orcasound has a serverful cloud architecture using Heroku, a managed service that runs on top of an Amazon Web Services (AWS) EC2 virtual machine. Instead of exposing the underlying VM, Heroku provides Dynos, which are isolated, lightweight Linux containers that operate like always-on VMs. Multiple customers' Dynos share the same large VM. Dynos need to boot up a full Linux environment, so don't start and stop instantly like a serverless container. 

By contrast, the serverless architecture for ESP32_api uses Cloud Run, a managed service within Google Cloud Platform (GCP). Instead of a large VM with many tenants, Cloud Run runs each container in its own proprietary Google-managed gVisor sandbox on a shared Google OS. Google keeps a massive 
pool of "warm" resources ready to execute code at a millisecond's notice. 

**About cold starts**

Because my esp32_api serverless container receives data every 2 seconds from my sensor, it most likely stays alive and never spins down -- idle time is about 15 seconds. This means the React UI might benefit from accessing an already-live container without cold start -- until there is enough traffic that Cloud Run needs to cold start a new container to handle concurrent load.

To monitor cold starts in GCP:
- Cloud Run Metrics Tab: Look at the "Container Instance Count" graph. If you see the line drop to 0 and then jump to 1, the first request at that "jump" was a cold start.
- Cloud Logging: Search your logs for the phrase `Defaulting to latest available Python runtime`. This line appears during the build/start phase. More importantly, look for requests with a very high latency compared to others. A typical request might take 50ms, while a cold start might take 2–5 seconds.

There is also a trick for ensuring the container never cold starts -- set 'minimum instances' to 1 so it never scales to 0. This results in a small minimum monthly fee ($1-5/mo).

**About Dockerfiles**

Cloud Run asks whether to build the container based on a Dockerfile I provide, or to use a default Python webpack provided by Google. 

A Dockerfile is a set of instructions for how to build a container -- specifying an exact OS, Python version, dependency installation command (e.g. `pip install -r requirements.txt` is the default), and system-level libraries like maybe `ffmpeg` for audio processing or `libpq` for databases that a standard buildpack might miss.

Right now I don't have specific system dependencies. My app uses standard Python with FastAPI and should work in any OS. I'm not currently doing any audio processing, although I might want that in the future. For now, I want to keep it simple. So I take the webpack route and accept the defaults. 

**About webpacks**

Using the default webpack, there are some assumptions and gotchas that are good to be aware of. 

1. The Python version -- this one slipped past me as a backend noob. While I was familiar with the convention of documenting Python package dependencies in a `requirements.txt` file, I didn't realize that this only captures Python packages, not the Python version. For this, Cloud Run looks for a separate file called `.python-version` -- which is the file that the commonly used `pyenv` version manager generates when setting a directory-level version via `pyenv local [version]`. Cloud Run would also recognize a `GOOGLE_PYTHON_VERSION` environment variable. Otherwise, it defaults to the most recent version.

2. Webpack inputs -- despite being a streamlined path, the webpack UI requests input that looks a little arcane until you understand what it's asking. You also need to avoid another gotcha.
- Build context directory -- this is just the application root, e.g. `/` but in my case it is `/server` 
- Entrypoint -- they want the command used to start the server. You can leave this blank, which is safest, because herein lies another gotcha. Locally, I go to the `/server` directory and run `uvicorn app.main:app --host 0.0.0.0 --port 8000` because I want to ensure the exact host and port I'm running on. But Cloud Run runs on port 8080, so specifying 8000 would lead to an error. The default is `uvicorn app.main:app` so leaving it blank works fine (see 'About entrypoints' below). 
- Function target -- this doesn't apply, it's for 'function' oriented deployments instead of full APIs. Leave blank.


**About entrypoints**

You can manually set the entrypoint in the Cloud Run UI or in a `Procfile`, or leave it blank to have Cloud Run detect it automatically. The entrypoint detection process will wrap the app code in one of two kinds of Python web server interfaces:
1. Gunicorn = a WSGI (web server gateway interface), the traditional standard for Python, used by Flask/Django
2. Uvicorn = an ASGI (asynchronous server gateway interface), the newer standard used for high-performance, asynchronous apps like FastAPI

If Cloud Run detects FastAPI, it will assume an ASGI environment and use Uvicorn. If not, and no other framework default is specified, it defaults to Gunicorn.


**About virtual environments (venv)**

A venv is used in Python development, not so much in production deployment. But I was a little hazy about the difference between a venv and a container so needed to clarify. Basically a venv is a Python-native feature that only captures Python dependencies, while a container is a snapshot of an entire system environment including OS and system-level dependencies. The Python version is a system-level dependency. When you activate a venv it will _try_ to use the same version of Python it was created with, but it doesn't bring the Python installation with it, just the Python packages. In production, Cloud Run doesn't use a venv at all, it installs dependencies from requirements.txt directly to the container environment. 


**About Docker Compose**

The Orcasound app doesn't just have a Dockerfile -- it also has a Docker Compose setup with a few elements to be familiar with. 

Relevant files:
1. `.devcontainer/devcontainer.json` - This triggers VS Code to automatically set up a development container when opening the repo. It points to three docker-compose.yml files that get merged together.
2. `docker-compose.yml` -- the base setup
3. `docker-compose.dev.yml` -- overrides for things needed for local dev
4. `../docker-compose.yml` -- VS Code-specific settings in the `.devcontainer` folder, lets VS Code attach to the container as a development shell
5. `Dockerfile` - defines how to build a container image. An image is a frozen snapshot of a filesystem and runtime. It is a template for creating containers. Docker Compose creates containers from images and defines how containers work together. 
