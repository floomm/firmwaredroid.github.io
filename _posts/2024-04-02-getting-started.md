---
title: Getting Started
author: Tom
date: 2024-04-02 13:37:00 +0800
categories: [Setup, Tutorial]
tags: [installation, getting started]
---

1. Clone the repository and change to the FirmwareDroid directory
    ```bash
    git clone --recurse-submodules https://github.com/FirmwareDroid/FirmwareDroid.git
    
    cd FirmwareDroid
    ```

2. Install python packages for the setup:

    ```bash
    pip3 install jinja2
    ```

3. Run the setup script:

    ```bash
    python3 ./setup.py
    ```

4. Build the docker base images (takes some time).

    ```bash
    chmod +x ./docker/build_docker_images.sh
    ./docker/build_docker_images.sh
    ```

5. Set correct access rights for the blob storage and start the containers

    ```bash
    chown -R $(id -u):$(id -g) ./blob_storage
    docker-compose up
    ```
    The first start takes some time because the database will change the mode to a replica-set.
    Wait until the logscreen stops moving (takes usually 2 to 5 minutes).

6. Open the browser under [https://fmd.localhost](https://fmd.localhost). By default, a self-signed certificates is used, and you will encounter a TLS warning.

7. Log-into the application. Password and username can be found in the `.env` file within the root directory of the server.

    ```bash
    cat .env
    ...
    DJANGO_SUPERUSER_USERNAME=XXXXX-XXXX
    DJANGO_SUPERUSER_PASSWORD=XXXXX-XXXX-XXXX
    ...
    ```

8. After log-in, you can explore the following routes:
    - [GraphQL API (/graphql/)](https://fmd.localhost/graphql/)
    - [User-Management (/admin/)](https://fmd.localhost/admin/)
    - [RQ-Job Management (/django-rq/)](https://fmd.localhost/django-rq/)
