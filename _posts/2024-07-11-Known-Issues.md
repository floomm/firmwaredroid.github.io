---
title: Known Issues
author: Tom
date: 2024-07-11 13:37:00 +0800
description: Short summary of the post.
categories: [Issues, Troubleshooting]
tags: [Issues, Troubleshooting, Problems, Bugs]
toc: true
comments: false
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