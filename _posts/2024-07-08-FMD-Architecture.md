---
title: FMD Architecture
author: tom
date: 2024-07-08 13:37:00 +0800
categories: [Documentation, Architecture]
tags: [Architecture, API, Overview]
---

To give you a better understanding of the FirmwareDroid (FMD) architecture, we will provide an overview of the 
different components and how they interact with each other. A good starting point of the components can 
be found in our research paper [FirmwareDroid: Towards Automated Static Analysis of Pre-Installed Android Apps](https://ieeexplore.ieee.org/document/10172951).
However, the implementation mentioned in the paper is already a bit outdated, and 
we will provide you with the most recent information in this post.

## Architecture Overview
The following image gives you an overview of the different components and how they interact with each other:

![FirmwareDroidOverview](https://firmwaredroid.github.io/commons/FirmwareDroidOverview.png)

The main idea of FirmwareDroid is to be able to analyze Android firmware images and the pre-installed applications (APKs)
at scale. The architecture of FMD is designed to be scalable and modular. The main components of the FMD architecture
are based on docker containers and are orchestrated by docker compose. A redis queue is used to manage the different
tasks, workers, and queues. We use RQ (Redis Queue) as a simple Python library for queueing jobs and 
processing them in the background with workers. Workers are docker containers that are responsible for different tasks
like extracting firmware, scanning APKs, or analyzing the extracted firmware. The main webserver is a Django application
that serves the client-side of the FMD application. The client-side is a React application that served by Django and
gunicorn. The main database is a MongoDB database that stores the extracted firmware and the analysis results.

### Main directories and files
Following is a brief overview of the main directories and files in the FirmwareDroid repository (state of 2024-07-08):
- `setup.py`: A standalone script that installs the necessary environment files and sets up the project.
- `blob_storage`: The main storage directory for all the data (databases and blobs).
- `docker`: Contains the Dockerfiles and scripts to build the Docker containers.
- `firmware-droid-client`: The client-side (web interface) of the FMD application.
- `requirements`: The Python requirements for the different tools and components.
- `source`: The source code of the FMD application.
  - `models`: The database models.
  - `static`: The static files (CSS, JS, images).
  - `api`: The definition of the GraphQL API schema and endpoints.
  - `webserver`: The Django webserver configuration. 
- `Dockerfile_BASE`: Base Dockerfile for the FMD application and all derived docker containers.

### Docker Containers
Following is a brief overview of the different docker containers used in the FirmwareDroid application 
(state of 2024-07-08):
- `backend-work`: The main webserver that serves the FMD application.
- `extractor-worker`: A worker container that extracts the firmware and handles files.
- `apk_scanner-worker`: A worker container that is responsible for APK scanning with various static analysis tools.
- `nginx`: The reverse proxy that forwards the requests to the backend and the client. Used to provide TLS termination.
- `mongo-db-1`: The MongoDB database that stores the extracted firmware and the analysis results. Running as a replica 
set.

By default, the docker containers are started with the `docker-compose.yml` file in the root directory of the server. 
The docker-compose.yml consumes the `.env` file in the root directory of the server to set the environment variables
for the different containers. 

### Environment Variables
The FMD application uses environment variables to configure the different components. The environment variables are
stored in the `.env` file in the root directory of the server. Additionally, there exists a `env` directory, that 
contains the environment files for the different docker containers.

### RQ Worker Queues
The queues in the RQ worker (see [RQ](https://python-rq.org/)) are used to manage the different tasks and workers. 
The following queues are used in the FMD application (state of 2024-07-08):
- `high-python`: The high-privilege queue for Python workers that have the access right to mount directories. This queue
is mainly used for the extraction of firmware and should not be used for other tasks.
- `default-python`: The default-privilege queue for Python workers that scan APKs and analyze the extracted firmware.

The queues are initialized in the `settings.py` file of the Django application. The following snippet shows the 
default configuration:
```
RQ_QUEUES = {
    'high-python': {
        'HOST': REDIS_HOST,
        'PORT': 6379,
        'DB': 0,
        'PASSWORD': REDIS_PASSWORD,
        'DEFAULT_TIMEOUT': 60 * 60 * 24 * 14,
        'DEFAULT_RESULT_TTL': 60 * 60 * 24 * 3,
    },
    'default-python': {
        'HOST': REDIS_HOST,
        'PORT': 6379,
        'DB': 0,
        'PASSWORD': REDIS_PASSWORD,
        'DEFAULT_TIMEOUT': 60 * 60 * 24 * 14,
        'DEFAULT_RESULT_TTL': 60 * 60 * 24 * 3,
    },
}
```
Additional queues can be added by extending the `RQ_QUEUES` dictionary in the `settings.py` file.

Tasks can be enqueued in the different queues by using the `django_rq` library. The following snippet shows how to
enqueue a task in a specific queue:
```
queue_name = 'high-python'
func_to_run = 'path.to.your.function'
queue = django_rq.get_queue(queue_name)
job = queue.enqueue(func_to_run, job_timeout=ONE_WEEK_TIMEOUT)
```
When workers are listening, they will pick up the tasks from the different queues in a first-in-first-out manner. Every
worker is assigned to a specific queue and will only process tasks from this queue. Workers are spawned by the
`rqworker` command and can be scaled up and down depending on the workload. The following snippet shows how to start
a worker for a specific queue within a docker container:
```
# Snippet from the docker-compose.yml
...
command: rqworker --logging_level INFO --name extractor-worker-high-1 --url redis://:${REDIS_PASSWORD}@redis:6379/0 high-python
...
```
