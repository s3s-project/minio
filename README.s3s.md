# s3s branch — unofficial MinIO image builds

This branch is **not** upstream MinIO. It exists so that [s3s](https://github.com/s3s-project/s3s)
can keep running its MinIO-backed end-to-end tests from an image it controls.

## Why this branch exists

MinIO stopped publishing free community images in October 2025 and removed the
`minio/minio` Docker Hub repository in September 2026; the upstream repository is
[no longer maintained](https://github.com/minio/minio#readme) and ships source only.
The last image ever published (`RELEASE.2025-09-07T16-13-09Z`) predates the fix for
[GHSA-jjjj-jwhf-8rgr](https://github.com/minio/minio/security/advisories/GHSA-jjjj-jwhf-8rgr).

Building from a pinned upstream commit gives s3s a reference implementation that is
both available and patched, without depending on any registry we do not control.

- This branch sits on top of `master`, which is kept identical to `minio/minio` `master`;
  everything s3s adds lives here only.
- Pinned upstream commit: [`.minio-source-ref`](./.minio-source-ref) — the upstream
  `master` commit this branch is based on, so the recorded revision always names the
  exact tree the image is built from.
- Build recipe: [`Dockerfile.s3s`](./Dockerfile.s3s) (Go build stage → alpine runtime).
- Publish workflow: [`.github/workflows/publish-ghcr.yml`](./.github/workflows/publish-ghcr.yml)
  → `ghcr.io/s3s-project/minio`.

## Patches carried on this branch

Upstream is unmaintained, so this branch is not only build files: it also carries the
minimum patch needed for the server to behave the way the E2E suite expects.

- `cmd/api-response.go`: `trackingResponseWriter` (added upstream in `52eee5a2f`) embeds
  the `http.ResponseWriter` interface, and embedding an interface does not promote
  `Flush`. Every helper that type-asserts to `http.Flusher` — `internal/http.Flush` — thus
  became a silent no-op for streamed responses: `ListenBucketNotification` delivered
  nothing until the HTTP server's write buffer happened to fill up, so clients that wait
  for one event, such as mint's `minio-java` suite, hung until their timeout. The patch
  forwards `Flush` to the wrapped writer.

Because the patched source lives on this branch, the Corresponding Source for a published
image is this repository at the tag it was built from, not the pinned upstream commit
alone.

## Images

- `ghcr.io/s3s-project/minio:<yyyymmddhhmm>` — one tag per publish, the UTC timestamp of
  the tag that triggered it.
- `ghcr.io/s3s-project/minio:edge` — the most recent publish.
- Every image carries `org.opencontainers.image.revision` = the upstream commit it was
  built from, so an image's source is always recoverable even though the tag does not
  name it.

Consumers should pin the digest rather than a tag. s3s does that in
[`scripts/minio.env`](https://github.com/s3s-project/s3s/blob/main/scripts/minio.env).

## Building locally

```bash
docker build -f Dockerfile.s3s -t minio-s3s .
```

The build derives its version metadata from `.minio-source-ref` the same way upstream's
`buildscripts/gen-ldflags.go` does at that commit, so `minio --version` reports the
upstream source identity. Behind a network where the default Go module proxy is
unreachable, pass a mirror: `--build-arg GOPROXY=https://<mirror>,direct`.

## Publishing

```bash
git tag -a "$(date -u +%Y%m%d%H%M)" -m "MinIO community build from upstream $(cat .minio-source-ref)"
git push origin s3s --follow-tags
```

There is no scheduled rebuild: the upstream source is frozen, so rebuilds happen only
when someone asks for one (`workflow_dispatch`, or a fresh tag) — for example to pick up
base image security updates. Pushing a tag that does not point at this branch (such as a
mirrored upstream `RELEASE.*` tag) is skipped by the workflow's guard job.

## License and provenance

MinIO is licensed under the [GNU AGPLv3](./LICENSE); this branch redistributes only
unmodified upstream source, and the built image bundles `LICENSE` and `NOTICE`. The
Corresponding Source is this repository at the commit recorded in `.minio-source-ref`.

This is an unofficial community build. It is **not affiliated with, sponsored by, or
endorsed by MinIO, Inc.** "MinIO" is a trademark of its respective owner.
