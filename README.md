# CNPG SBOM generator

Standalone BuildKit SBOM scanner for PostgreSQL extension container images.
It scans the builder with Syft, scopes the inventory to files shipped in the
final image, preserves package and license evidence, and validates SPDX 2.3.
Optional Python hooks can add extension-specific evidence.

Staged at **ardentperf-labs/cnpg-sbom-generator**, with eventual ownership by
**cnpg-extensions**. Both CNPG Extensions and CloudNativePG can consume the
published scanner image without carrying its source or release workflow.

## Consumer integration

Expose the stage containing the package database and original files:

```dockerfile
FROM debian:bookworm-slim AS builder
ARG BUILDKIT_SBOM_SCAN_STAGE=builder
# Install/build the extension and collect its runtime files and license notices.

FROM scratch
COPY --from=builder /extension/ /
```

Use the generator as a BuildKit attestation scanner, pinned to a checked digest:

```bash
GENERATOR='ghcr.io/cnpg-extensions/cnpg-sbom-generator:latest@sha256:<checked-index-digest>'
docker buildx build \
  --attest "type=sbom,generator=$GENERATOR" \
  --provenance=mode=max --tag "$EXTENSION_IMAGE" --push .
```

For existing postgres-extensions-containers workflows, set their
`sbom_generator` input to that same image reference. Continue exposing the
`builder` stage. No source checkout of this repository is needed.
The current upstream digest remains valid during extraction; see
[MIGRATION.md](MIGRATION.md) for publisher handoff and signature identity changes.

## Development

```bash
docker buildx build --load --tag cnpg-sbom-generator:local .
docker run --rm --entrypoint python3 \
  --volume "$PWD:/workspace:ro" --workdir /workspace \
  cnpg-sbom-generator:local -m unittest discover -s tests
```

CI builds and tests on native AMD64 and ARM64, including a real BuildKit
attestation, SPDX validation, Grype, and Grant. The release workflow publishes
both architectures, signs the checked image with Cosign, and promotes its digest
to `latest` or `test`. Publication is disabled unless repository variable
`SBOM_PUBLISH_ENABLED` is `true`; checks still run normally.

- [Usage, hooks, and release guide](GUIDE.md)
- [Reviewer guide](REVIEWER-GUIDE.md)
- [Dependency workarounds](WORKAROUNDS.md)
- [Migration and provenance](MIGRATION.md)

Licensed under Apache-2.0; original copyright notices are preserved.
