---
name: clean-logs
description: Clear Docker container logs. Use when user asks to clean logs, clear logs, flush logs, or reset logs for any docker container.
disable-model-invocation: true
---

Truncate a Docker container's logs to zero.

## Steps

1. If `$ARGUMENTS` is given → use it as the container name/id:
```bash
truncate -s 0 $(docker inspect --format='{{.LogPath}}' $ARGUMENTS) && echo "✅ $ARGUMENTS logs cleared"
```

2. If no argument → ask the user which container:
```bash
docker ps --format "table {{.Names}}\t{{.Status}}"
```
Then wait for the user to specify, and run step 1 with that name.

## Notes
- Does not restart the container — safe to run while container is live
- Requires sudo if permission denied:
  `sudo truncate -s 0 $(docker inspect --format='{{.LogPath}}' $ARGUMENTS)`