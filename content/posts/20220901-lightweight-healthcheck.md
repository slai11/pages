---
title: "20220901 Lightweight Healthcheck"
date: 2022-09-01T21:04:49+08:00
tags:
- fragments
---

An interesting pattern I noticed while exploring the deployment process at
GitLab was the use of `pgrep` to perform healthchecks in the Dockerfile or even
in Kubernetes (technically you could). 

First define a `healthcheck` file:
```
#!/bin/bash

set -e

/usr/bin/pgrep -f <process-to-check>
```

If there is something, the exit code is 0, else it returns exit code 1. That is
a pretty nifty trick.

Second, add this line into your Dockerfile:

```
HEALTHCHECK --interval=30s --timeout=30s --retries=5 CMD /healthcheck
```

In a kubernetes setting, simply configure your `livenessProbe/readinessProbe` and you should be
good to go.

```
livenessProbe:
  exec:
    command:
    - /healthcheck
  initialDelaySeconds: 5
  periodSeconds: 5
```

Is this an overkill? Do we need to healthcheck our workers if the container is going to restarts anyway? You could monitor container or pod restarts and fire alerts on those signals instead. *shrugs*

