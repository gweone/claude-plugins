---
name: build
description: Build the project using Gradle. Use when the user asks to build, compile, run gradlew, or run a clean build.
disable-model-invocation: true
---

Run a full Gradle clean build from the workspace root.

## Steps

1. Confirm the current directory is the workspace root (where `gradlew` exists).
   - If not, `cd` to the workspace root first.

2. Run the build:
   ```bash
   ./gradlew clean build
   ```

3. Wait for the build to complete.

4. Report the result:
   - If **SUCCESS** → tell the user the build passed and list any `.jar` or `.zip` output files found under:
     - `modules/*/build/libs/*.jar`
     - `client-extensions/*/dist/*.zip`
   - If **FAILED** → show the first error block from the output (the `> Task :...` failure lines) so the user can act on it immediately.

## Notes
- Do not skip `clean` — always run `clean build` together to avoid stale output.
- If the user specifies a module (e.g. "build only my-module"), append `--project-dir modules/my-module` or use `:my-module:clean :my-module:build` instead.
- Do not deploy after building unless the user explicitly asks.