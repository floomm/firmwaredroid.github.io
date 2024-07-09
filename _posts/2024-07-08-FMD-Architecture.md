---
title: FMD Architecture
author: Tom
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
- `nginx`: The reverse proxy that forwards the requests to the backend and the client.
- `mongo-db-1`: The MongoDB database that stores the extracted firmware and the analysis results. Running as a replica 
set.
