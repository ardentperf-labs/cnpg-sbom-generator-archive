# CNPG SBOM generator

A standalone BuildKit SBOM scanner for container images assembled from a builder
stage. It uses Syft to identify packages in the builder, matches them to files
shipped in the final image, and produces a validated SPDX 2.3 software bill of
materials with package, checksum, and license evidence.

The main consumer is [cnpg-extensions](https://github.com/cnpg-extensions), which
uses it for PostgreSQL extension images. Any compatible BuildKit build can use
the published scanner image without checking out this repository.

## How it works

Minimal images often contain copied binaries and libraries but no package
database. Scanning the final filesystem alone loses the evidence needed to
identify those packages.

The generator scans the builder stage and inventories the final filesystem.
It matches shipped files to builder files using paths and checksums, retains
their owning packages, and preserves license metadata. BuildKit attaches the
resulting SPDX document to the output image as an SBOM attestation. The scanner
and its dependencies are not added to the final image.

## Consumer integration

Name the stage containing the package database and original files `builder`,
and expose it to the scanner:

```dockerfile
FROM debian:bookworm-slim AS builder
ARG BUILDKIT_SBOM_SCAN_STAGE=builder
# Build or install your software and collect runtime files in /payload.
# Include the corresponding license notices in that payload.

FROM scratch
COPY --from=builder /payload/ /
```

Select a checked generator image digest and pass it to Docker Buildx:

```bash
GENERATOR='ghcr.io/cnpg-extensions/cnpg-sbom-generator:latest@sha256:<checked-index-digest>'
IMAGE='your-registry/your-image:tag'

docker buildx build \
  --attest "type=sbom,generator=$GENERATOR" \
  --provenance=mode=max --tag "$IMAGE" --push .
```

Replace the digest placeholder with a published image index digest. Pinning the
digest keeps builds reproducible when the `latest` tag moves. Generator images
support Linux AMD64 and ARM64.

Extract the attached SBOM for a platform:

```bash
docker buildx imagetools inspect "$IMAGE" \
  --format '{{ json (index .SBOM "linux/amd64").SPDX }}' \
  > image.spdx.json
```

The SPDX document can be used with tools such as Grype for vulnerability reports
and Grant for license reports. Original license notices must be included in the
final image; an SBOM does not replace them.

## Add custom inventory

Builds can supply a Python hook at
`/usr/local/share/cnpg-sbom/augment_spdx.py` in the builder stage. Its
`augment_spdx(document, context)` function receives the scoped SPDX document and
builder evidence, and returns the updated document. This allows consumers to add
components that the builder scan cannot identify.

Hooks execute as trusted code inside the scanner. Hook errors or invalid SPDX
fail the build. See the [hook guide](GUIDE.md#add-extension-specific-inventory)
and [example hook](tests/integration/fixture/augment_spdx.py).

## Development

Build the scanner and run its unit tests from the repository root:

```bash
docker buildx build --load --tag cnpg-sbom-generator:local .
docker run --rm --entrypoint python3 \
  --volume "$PWD:/workspace:ro" --workdir /workspace \
  cnpg-sbom-generator:local -m unittest discover -s tests
```

CI builds and tests on native AMD64 and ARM64 runners. Integration checks exercise
real BuildKit attestations, SPDX validation, package ownership, custom hooks,
Grype, and Grant.

- [Usage and release guide](GUIDE.md)
- [Reviewer guide](REVIEWER-GUIDE.md)
- [Dependency workarounds](WORKAROUNDS.md)

## License

[Apache-2.0](LICENSE).
