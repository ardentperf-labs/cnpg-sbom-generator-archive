# Extraction and publisher handoff

## Source

Extracted from `cnpg-extensions/postgres-extensions-containers` commit
[`cf8e7be48c45fa52e4719410c5e18058ebcc396b`](https://github.com/cnpg-extensions/postgres-extensions-containers/tree/cf8e7be48c45fa52e4719410c5e18058ebcc396b/sbom-generator).
The generator, dependency pins, tests, and fixtures are preserved. Its former
`sbom-generator/` directory is now the repository root. Release build context,
workflow paths, documentation, and Renovate scope are adjusted for that layout.
This repository retains the source repository's Apache-2.0 license.

## Staging

The new repository is `ardentperf-labs/cnpg-sbom-generator`. CI runs checks on
pushes and pull requests. Publishing requires `SBOM_PUBLISH_ENABLED=true`.
Leave it unset while the old workflow is active: concurrency groups serialize
runs only within one repository, not between publishers.

The image destination remains owner-relative:
`ghcr.io/<repository-owner>/cnpg-sbom-generator`. In ardentperf-labs this name
is also used by the existing fork. Before enabling a staging publication,
disable that fork's generator publisher and grant this repository Actions write
access to the package. Manual `publish=true` moves only `test`; manual
`publish_latest=true` on main moves `latest`. Both require the publication gate.

## Final handoff to cnpg-extensions

1. Transfer this repository to `cnpg-extensions/cnpg-sbom-generator` and verify
   Actions permissions and available native AMD64/ARM64 runners.
2. Disable/remove `.github/workflows/sbom-generator.yml` in the old
   `cnpg-extensions/postgres-extensions-containers` repository and wait for its
   active publishing runs to finish. Keep consumers pinned to their existing
   checked digest throughout the transition.
3. In the existing GHCR `cnpg-sbom-generator` package settings, grant the new
   repository Actions write access and update its linked repository. A repository
   transfer alone does not grant access to a pre-existing package.
4. Update any generator Cosign verification policy to trust the new workflow
   identity below. Old digests retain the old signing identity.
5. Set `SBOM_PUBLISH_ENABLED=true` in the new repository. Dispatch the workflow
   on main with `publish_latest=true`. Require both architecture checks and the
   signed, checked publication to succeed before using the resulting digest.
6. Update the generator digest in CNPG Extensions and CloudNativePG consumers
   and run their affected extension builds. The production image path stays
   `ghcr.io/cnpg-extensions/cnpg-sbom-generator`, so there is no registry rename.
7. Remove the old generator source, generator-only Renovate rules, and obsolete
   workflow exclusions from the consumer repository. Retain consumer digest
   update rules and extension integration. Install/configure Renovate here and
   require the `Automerge gate` status before enabling repository auto-merge.

Verify a newly published generator with:

```bash
cosign verify "ghcr.io/cnpg-extensions/cnpg-sbom-generator@$DIGEST" \
  --certificate-identity='https://github.com/cnpg-extensions/cnpg-sbom-generator/.github/workflows/sbom-generator.yml@refs/heads/main' \
  --certificate-oidc-issuer='https://token.actions.githubusercontent.com'
```

Staging signatures use `ardentperf-labs` in that identity. The generator does
not change consumer image signing or SLSA workflows.

For rollback, restore the previous checked digest in consumers and hold automated
digest updates. Disable the new publisher before re-enabling the old one. Retain
old commit images and their signatures.
