---
title: Known Issues
author: tom
date: 2024-07-11 13:37:00 +0800
description: A summary of known issues and their solutions.
categories: [Issues, Troubleshooting]
tags: [Issues, Troubleshooting, Problems, Bugs]
---

### Work-horse terminated unexpectedly
**Issue**: The worker container is terminated unexpectedly with the following error message:
```
extractor-worker-high-1  | 06:32:32 Moving job to FailedJobRegistry (Work-horse terminated unexpectedly; waitpid returned 9 (signal 9); )
extractor-worker-high-1  | 06:32:32 Cleaning registries for queue: high-python
```
- **Solution**: This issue is most likely caused by the worker container running out of memory. 
You can increase the memory limit for the worker container by setting the `DOCKER_MEMORY_LIMIT` environment variable in 
the `.env` file. For example, to set the memory limit to 20GB, add the following line to the `.env` file:
  ```
  DOCKER_MEMORY_LIMIT=20GB
  ```
After setting the memory limit, restart the worker container to apply the changes. Depending on your operating system
you might need to adjust the memory limit of the docker daemon as well.

- **Workaround**: If increasing the memory limit does not solve the issue, you can try to reduce the number of workers
or the number of elements in the queue to process.



### Initial setup of the database fails 
Sometimes when you try to deploy a new instance of FMD, the initialisation of the database fails with the following error message:
```
....
mongo-db-1               | {"t":{"$date":"2024-10-29T09:32:29.237+00:00"},"s":"E",  "c":"CONTROL",  "id":20557,   "ctx":"initandlisten","msg":"DBException in initAndListen, terminating","attr":{"error":"IllegalOperation: Attempted to create a lock file on a read-only directory: /data/db"}}
...
```
This can happen if the `mongo-db-1` container is not able to write to the `/data/db` directory within the container.
Usually that means that there was a problem creating the directory for the database due to permissions on the host system. 
If that is the case, subsequent attempts to start the container will fail because the directory already exists but is not writable and
the container will not be able to connect to the database. The following error message will be shown in the logs:
```
backend-worker           | pymongo.errors.ServerSelectionTimeoutError: mongo-db-1:27017: [Errno -2] Name or service not known (configured timeouts: socketTimeoutMS: 20000.0ms, connectTimeoutMS: 20000.0ms), Timeout: 30s, Topology Description: <TopologyDescription id: 6720aba298caf29a9cb0f029, topology_type: Unknown, servers: [<ServerDescription ('mongo-db-1', 27017) server_type: Unknown, rtt: None, error=AutoReconnect('mongo-db-1:27017: [Errno -2] Name or service not known (configured timeouts: socketTimeoutMS: 20000.0ms, connectTimeoutMS: 20000.0ms)')>]>
backend-worker exited with code 0
```
To overcome this issue you can remove the directories (`mongo_database` and `django_database`) and manually 
create them with the correct permissions. The environment variables `LOCAL_MONGO_DB_PATH_NODE1` and `DJANGO_SQLITE_DATABASE_MOUNT_PATH`
should be set to the correct paths where the directories should be created on the host.
```
# Check if the environment variables are set correctly in the .env file:
cat .env

# Delete the mongo_database and django_database directories -> Replace <YOUR_PATH> with the correct path:
rm -R <YOUR_PATH>/mongo_database
rm -R <YOUR_PATH>/django_database

# Creates the mongo_database directory -> Replace <LOCAL_MONGO_DB_PATH_NODE1> with the correct path:
mkdir -p <LOCAL_MONGO_DB_PATH_NODE1>

# Create the django_database directory -> Replace <DJANGO_SQLITE_DATABASE_MOUNT_PATH> with the correct path:
mkdir -p <DJANGO_SQLITE_DATABASE_MOUNT_PATH>
```
After creating the directories you can restart the deployment and the containers should start successfully.
```
docker compose up
```

