---
name: container-debug
description: Debug container runtime issues — image build failures, runtime errors, entrypoint/PID1, user/permissions, volume mounts, exit codes, multi-arch, distroless. Covers `docker`, `podman`, `nerdctl`, and pod-level container internals via `kubectl debug`. NOT for orchestrator scheduling (use kubernetes-debug).
---

# Inspect (read-only)
```
docker ps -a --format 'table {{.ID}}\t{{.Names}}\t{{.Status}}\t{{.Image}}'
docker inspect <CID> | jq '.[0] | {State, Config, Mounts, HostConfig:{NetworkMode,RestartPolicy}}'
docker logs --tail=200 --timestamps <CID>
docker top <CID>
docker stats --no-stream <CID>
docker diff <CID>          # filesystem changes vs image
```

# Exit codes
- `0` clean ; `1` generic ; `125` docker daemon error ; `126` not executable ; `127` not found ; `137` SIGKILL/OOM ; `139` SIGSEGV ; `143` SIGTERM

# Build failures
```
docker build --progress=plain --no-cache -t <T> .
docker history <T>
```
- Layer too large → multi-stage, `.dockerignore`, `--squash` (experimental)
- Cache miss → reorder COPY of files that change least → most
- Multi-arch: `docker buildx build --platform linux/amd64,linux/arm64 --push -t <T> .`

# Run-time issues
- **PID 1 ignores SIGTERM** → use `tini`, or shell-form CMD; pid1 must reap zombies
- **Permission denied on mount** → UID/GID mismatch with host; `chown` in entrypoint or `--user`
- **No such file (works locally)** → wrong arch (`docker inspect <IMG> | jq '.[0].Architecture'`) or distroless lacks shell
- **CrashLoop with no logs** → entrypoint exits immediately; run with `--entrypoint sh` to inspect

# In-cluster (without shell in image)
```
kubectl debug -it <POD> -n <NS> --image=busybox --target=<container>      # shares pidns
kubectl debug -it <POD> -n <NS> --image=busybox --copy-to=<POD>-debug --share-processes
```

# Image provenance / vulns
```
docker image inspect <T> --format '{{.RepoDigests}}'
trivy image --severity HIGH,CRITICAL <T>
```

# Cleanup (destructive)
- `docker system prune -af --volumes` — removes ALL unused; rollback: none. Confirm host is not multi-tenant.
