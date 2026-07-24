---
name: liferay-logs
description: Show Liferay Docker container logs. Use when the user asks to check logs, watch logs, see liferay output, tail liferay, or monitor deployment in liferay.
disable-model-invocation: true
---

Stream logs from the Liferay container using Docker Compose.

## Steps

1. Run command on liferay-dev container logs
   ```bash
   docker logs liferay-dev
   ```

2. Stream the Liferay service logs:
   ```bash
   docker compose logs liferay-dev--tail=100 --follow
   ```

3. Watch for key events and summarize what you see:
   - ✅ `Server startup in` → deployment complete
   - 📦 `STARTED` followed by a bundle name → module deployed successfully
   - ❌ `ERROR` or `FAILED` lines → show these prominently so the user can act
   - ⏳ No output yet → tell the user Liferay may still be starting

4. If the user only wants a snapshot (not a live stream), run without `--follow`:
   ```bash
   docker logs liferay-dev --tail=200
   ```