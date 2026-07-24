---
name: deploy
description: Deploy built artifacts to the Liferay dev environment. Use when the user asks to deploy, copy jars, copy zips, push to liferay, or deploy a module or client extension.
disable-model-invocation: true
---

Copy built artifacts from the workspace into the Liferay dev deploy folder.

## Deploy HOT Target Path
```
/opt/github/sharpps.dpdns.org/docker/liferay-dev/mount/deploy/
```

## Source Paths
- **Modules** (`.jar`): `modules/<module-dir>/build/libs/*.jar`
- **Client Extensions** (`.zip`): `client-extensions/<cx-dir>/dist/*.zip`

## Steps

### 1. Check source files exist
```bash
ls modules/<module-dir>/build/libs/*.jar
ls client-extensions/<cx-dir>/dist/*.zip
```
If nothing found → stop and tell the user to run `/build` first.

### 3. Copy and confirm each file
```bash
cp modules/<module-dir>/build/libs/<file>.jar \
   /opt/github/sharpps.dpdns.org/docker/liferay-dev/mount/deploy/ \
   && echo "✅ <file>.jar copied"

cp client-extensions/<cx-dir>/dist/<file>.zip \
   /opt/github/sharpps.dpdns.org/docker/liferay-dev/mount/deploy/ \
   && echo "✅ <file>.zip copied"
```
- If `cp` succeeds → `&& echo` prints ✅
- If `cp` fails → echo is skipped, the shell error is shown instead

## Notes
- Liferay will do Hot deploy automatically, Do not verify file exists in target after copy — Liferay processes and deletes it immediately
- Do not run a build automatically — if artifacts missing, ask user to run `/build` first
- If user says "deploy all", loop through all modules and client-extensions that have output files