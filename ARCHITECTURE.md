# Architecture and implementation

The final image is built `FROM scratch`: it retains extension files and libraries,
but no package database. The generator uses the builder's package evidence to
identify the files actually shipped, excluding packages with no matched files.
It runs as a BuildKit scanner; its tools are not added to the extension image.

See [How it works](README.md#how-it-works) for the complete architecture diagram.

Files are copied to different locations in the scratch image: for example,
`/usr/share/doc/gcc-12-base/copyright` becomes `/licenses/gcc-12-base/copyright`.
The final path therefore cannot be used directly to look up Debian ownership.
We match each shipped file to its original builder file, then retain the package
that owned that builder file.

Ownership requires matching SHA256 content. Exact paths and shared path suffixes
rank candidates; ambiguous matches stay unclaimed. The composer retains owning
packages and their relationships, and passes through Syft package identities,
license references, and extracted license text. It updates file paths, IDs, and
checksums, repairs relationships, and recalculates package verification codes for
the shipped subset. Original copyright notices remain intact. Hash-based license
references can be difficult to interpret; see the [Syft license-reference usability work](WORKAROUNDS.md#syft-readability-of-hash-based-license-references)
and [checksum-based notice retrieval](README.md#retrieve-an-original-notice).

Builder scans should mirror BuildKit's default Syft generator as closely as
possible. Review the [Syft call](generator.py) against the BuildKit scanner settings
linked there: scan base path, source alias, `image,file` cataloger defaults, and
added `sbom-cataloger`. Tests exercise installed packages, embedded SBOMs, and
exclusion of source-only Go dependencies.

## Compatibility contract

The reference project is
[cloudnative-pg/postgres-containers](https://github.com/cloudnative-pg/postgres-containers),
as described in the [compatibility goal](README.md#compatibility). Preserve the
same SBOM and provenance formats and user-facing structure, use the same
generation tools as far as possible, and keep inspection and verification
commands interchangeable apart from image and publisher parameters.

When changing scanner dependencies, SPDX composition, attestation packaging, or
release workflows, compare the resulting documents and commands with that
project. Check Docker Buildx SBOM extraction and provenance inspection, Cosign
verification, and Grype and Grant reporting. For consumers that publish separate
signed SLSA provenance, also compare its generation and verification workflow.
Record any necessary differences and their compatibility impact.

The existing integration tests cover BuildKit/SPDX/Grype/Grant behavior. They do
not establish complete parity with every image and publishing workflow in the
reference project; maintainers must assess that compatibility when relevant
behavior changes.

## Code map

| Area | Entry points |
| --- | --- |
| Syft lifecycle, final inventory, BuildKit statement | [generator.py](generator.py) |
| File ownership, package selection, metadata preservation, document identity | [compose.py](compose.py) |
| Optional hook and its context | [hooks.py](hooks.py) |
| Final SPDX conformance check | [spdx_validation.py](spdx_validation.py), [explicit SPDX Tools dependency](Dockerfile) |
| Scanner image and dependencies | [Dockerfile](Dockerfile) |
| Extension integration | [consumer integration](README.md#consumer-integration) |
| Generator checks, publication, and digest updates | [sbom-generator.yml](.github/workflows/sbom-generator.yml), [release.py](release.py), [renovate.json](renovate.json) |
| Unit checks and real BuildKit/SPDX/Grype/Grant fixture | [tests](tests), [integration](tests/integration) |

Consumers expose the `builder` stage with
`ARG BUILDKIT_SBOM_SCAN_STAGE=builder` and select the scanner image through
BuildKit's SBOM attestation option. The generator wraps its SPDX document in an
in-toto statement; BuildKit attaches it to the output image. Consumer image
signing and any separate SLSA provenance are handled by the consumer's workflows.

## Downstream augmentation

A builder may supply `/usr/local/share/cnpg-sbom/augment_spdx.py` with
`augment_spdx(document, context) -> dict`. It receives the scoped document as
`document` and the full, unprocessed Syft result as `context.builder_document`.
[HookContext](hooks.py) also exposes API version 1, extension name, platform, and
builder/final paths. After the hook returns, the generator calculates the final
document namespace and validates the complete SPDX, including hook additions.

The hook executes trusted Python in the generator environment. Builder-installed
dependencies are unavailable there; downstream builds can prepare JSON evidence
for the hook to read. Missing hooks leave the document unchanged; hook errors or
invalid final SPDX fail the build. SPDX validation checks conformance, not whether
added components actually belong to the payload.

PGRX is a downstream consumer of this interface: it supplies Cargo component and
license evidence. The core generator has no Cargo or PGRX processing dependency.
The [fixture hook](tests/integration/fixture/augment_spdx.py) demonstrates adding
a package that Grant can report.

## Local checks

From the repository root:

```bash
docker build -t cnpg-sbom-generator:local .
docker run --rm --entrypoint python3 \
  --volume "$PWD:/workspace:ro" --workdir /workspace \
  cnpg-sbom-generator:local -m unittest discover -s tests

# Extract an SPDX document for the validator check below.
IMAGE=ghcr.io/cnpg-extensions/pgsphere@sha256:54e6f0b0a9d9e4c80db8c6edcac98b451159c1fbaa91e429a92d4978f593518d
docker buildx imagetools inspect "$IMAGE" \
  --format '{{ json (index .SBOM "linux/amd64").SPDX }}' \
  > pgsphere.spdx.json
docker run --rm --entrypoint python3 --volume "$PWD:/workspace:ro" \
  cnpg-sbom-generator:local /opt/cnpg-sbom-generator/spdx_validation.py \
  /workspace/pgsphere.spdx.json
```

Delete `pgsphere.spdx.json` after the check, or supply another extracted
SBOM at that path. See the [user guide](README.md#verify-the-image-and-extract-its-sbom)
for publisher verification.

The validator accepts extracted SPDX or the generator's in-toto statement and
exits nonzero on invalid output. Native AMD64 and ARM64 CI checks also build the
scanner and exercise a real BuildKit attestation, Debian notice ownership, hook
augmentation, Grype distribution detection, and Grant license reporting.

See the [user and release guide](README.md) for inspection and publication commands.

## Release promotion

The generator image itself includes BuildKit's default SBOM and detailed
provenance, and the workflow signs its checked digest with Cosign. This does not
use the extension images' separate SLSA reusable workflow.

The publish job is serialized. Once its digest passes checks, a `main` push
updates `latest` even if unrelated commits have since advanced `main`.
`release.py` verifies the immutable commit tag and promoted alias against the
checked digest. GitHub does not guarantee queue order, so serialization alone
does not enforce source-commit order for publications.
