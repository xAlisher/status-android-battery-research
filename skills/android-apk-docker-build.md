---
id: android-apk-docker-build
title: Build Status Android APK locally via Docker (ContainerBuilds.mk)
created: 2026-06-04
project: status-android-battery-research
---

## Problem
`make apk-debug` via `ContainerBuilds.mk` fails in a local environment due to permission
mismatches, missing cache mounts, shallow-clone version detection, and missing Maven artifacts.

## Recipe

Five fixes required for `mobile/ContainerBuilds.mk` docker run command:

### 1. Root permissions
```makefile
--user root \
-e HOME=/home/jenkins \
```
The Docker image runs as `jenkins` (uid 1001). Host dirs are mode 750, inaccessible to other.
`--user root` + explicit `HOME` lets the container write everywhere.

### 2. Git safe.directory
First bash command inside container:
```bash
git config --global --add safe.directory "*"
```
Otherwise `git rev-parse --show-toplevel` fails → `STATUS_DESKTOP=""` → all derived paths break.

### 3. Cache volume paths (match HOME)
```makefile
-v $(HOME)/.docker-build-cache/go-build:/home/jenkins/.cache/go-build \
-v $(HOME)/.docker-build-cache/go-mod:/home/jenkins/go/pkg/mod \
-v $(HOME)/.docker-build-cache/gradle:/home/jenkins/.gradle \
-v $(HOME)/.docker-build-cache/m2:/home/jenkins/.m2 \
```
Go writes to `$HOME/go/pkg/mod` and `$HOME/.cache/go-build`. With `HOME=/home/jenkins`,
mounting to `/root/...` silently leaves caches on the overlay → fills root disk.

### 4. version.sh shallow clone
```bash
git -C /path/to/repo tag v2.38.0-battery-fixes HEAD
```
`scripts/version.sh` runs `git describe --tags`. Fails on a shallow clone with no tag ancestor.
Tag the tip commit once before building.

### 5. mobileui-android Maven publish
Add this step explicitly before `make apk-debug`:
```bash
cd ui/StatusQ/build/android/StatusQ/_deps/mobileui-src/android/qt6 && \
./gradlew publishToMavenLocal -q
```
CMake's `mobileui_android_publish` target only runs when StatusQ is dirty. On incremental
builds it's skipped, leaving `~/.m2` empty → Gradle fails with `Could not find im.status:mobileui-android:0.0.0-local`.

## Why
The build image was designed for CI (fresh checkout, root filesystem). Local reuse requires
explicit workarounds for each assumption CI makes. Tackle them in order: permissions →
git → cache → version → maven.

## Output
APK at `mobile/android/build/outputs/apk/debug/android-debug.apk` (287 MB, package
`app.status.mobile.debug`). Copy to accessible path: the `make` touch step will fail
with permission denied if run as non-root, but the APK is already built.

## See also
- `companion-pr-timing-race.md` — submodule timing after APK build
