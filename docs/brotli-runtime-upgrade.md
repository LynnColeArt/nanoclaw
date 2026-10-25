# Brotli runtime security upgrade

CVE-2026-111497 affects the Brotli decoder bundled with older Node.js and Bun
releases. This checkout uses Node.js 22.23.4 for the host and local agent image,
and Bun 1.4.4 for the agent runner. Supported host alternatives are Node.js
24.22.1+ on 24.x and 26.12.1+ on 26.x or a later supported release.

## Detect

Run `node --version` on the host. For each agent image you use, inspect both
runtimes without starting an agent:

```sh
docker run --rm --entrypoint node IMAGE --version
docker run --rm --entrypoint bun IMAGE --version
```

Replace `IMAGE` with the local image reference shown by the build or your pinned
registry reference. An existing image is unchanged by a checkout update.

## Why

An attacker-controlled Brotli response or other compressed input can reach the
native decoder through HTTP clients or decompression APIs. Upgrading application
packages alone does not replace the embedded runtime.

## Fix

1. Install the Node version in `.nvmrc` (or a supported patched alternative),
   reinstall host dependencies with `pnpm install --frozen-lockfile`, and rebuild
   with `pnpm run build`.
2. For local images, run `./container/build.sh build` to rebuild the image with
   Node 22.23.4 and Bun 1.4.4. Rebuild derived images as well.
3. Registry images are independent of this Dockerfile. The vendor image digest
   in `versions.json` has not been replaced by this source change. Verify its
   Node/Bun versions and vendor backports before using it; use a newly verified
   patched digest or the local build if necessary. An existing
   `NANOCLAW_AGENT_IMAGE_REF` override also needs verification.
4. Restart the host using your existing service manager and restart affected
   agent groups so running processes use the new binaries.

## Verify

Repeat the version commands against the actual rebuilt or replacement image.
Node must be at least 22.23.4 on 22.x, 24.22.1 on 24.x, or 26.12.1 on 26.x;
Bun must be at least 1.4.4. Run the host tests and agent-runner tests before
resuming normal use.

## Roll back

Keep the previous checkout and configuration for operational recovery, but do not
restore an affected runtime to service. If the new runtime is incompatible,
stop affected groups until a patched replacement image is verified. Reverting
source alone is not a safe rollback for this security fix.
