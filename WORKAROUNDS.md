# Upstream workarounds

Tracking the CloudNativePG upstream project is a high-priority goal. This register
covers intentional downstream workarounds and compatibility adaptations; choices
inherited from upstream do not require separate justification here.

On dependency updates, check the linked issue, pull request, or release contract,
verify the replacement, and remove temporary measures and their entries together
when they are no longer needed. Some entries describe packaging adaptations
rather than pending upstream bug fixes. Where no upstream issue is linked, say so
instead of implying that a fix is scheduled. Last reviewed: 2026-10-19.

## Renovate: Debian security package lookups

- **Upstream:** [issue #44330](https://github.com/renovatebot/renovate/issues/44330)
  and [PR #35865](https://github.com/renovatebot/renovate/pull/35865), both open.
- **Problem:** the native `deb` datasource requests `Packages.gz`, but Bookworm
  security indexes are published as `Packages.xz`. Renovate skips the failed
  component and can report no updates even when a newer security package exists.
- **Workaround:** the two SBOM generator Debian rules in [renovate.json](renovate.json)
  use `custom.debian-bookworm` to read the full version from Debian's package
  details page for the requested architecture. The `plain` format lets JSONata
  inspect the HTML version heading, or the architecture's version table cell
  when Debian lists different binary rebuilds across architectures.
  This works with hosted Renovate and keeps Debian version ordering, existing
  grouping, and manual merge review.
  Package details include security updates and version epochs; download filenames
  omit epochs and must not be used as the version source. Architecture-independent
  packages also resolve through the architecture-specific details URLs.
- **Limits:** this depends on Debian's website layout and refresh cadence. A
  missing or changed version heading or ambiguous version cell fails the lookup
  instead of producing a guessed version. Package availability and dependency
  compatibility still need validation by the generator build.
- **Remove when:** the hosted Renovate release supports the security indexes.
  Restore the `deb` registry URLs for `bookworm`, `bookworm-updates`, and
  `bookworm-security` on each architecture, remove the datasource overrides and
  custom datasource, and verify a security-only update is proposed on both
  architectures with the correct epochs.

## Syft: readability of hash-based license references

- **Upstream:** [issue #3527: Unable to classify complex licenses](https://github.com/anchore/syft/issues/3527).
  This [comment](https://github.com/anchore/syft/issues/3527#issuecomment-6295475239)
  discusses the usability of hash-based results.
  [PR #5448](https://github.com/anchore/syft/pull/5448), currently open, proposes
  more descriptive custom references for possible license matches below the
  configured coverage threshold.
- **Problem:** complex Debian copyright notices can appear in license reports
  as hash-based custom references. The license evidence is retained, but a hash
  alone gives users little help understanding it.
- **Current approach:** the SBOM generator passes through Syft's license
  references and extracted text while scoping package and file records to the
  shipped payload. Original notices are shipped in `/licenses`. Users can start with a checksum in Grant and retrieve the
  original notice. See [reading license results](GUIDE.md#reading-license-results).
- **Limits:** a hash identifies text; it does not explain its terms. PR #5448
  proposes `LicenseRef-Possible-<ID>-<hash>` names containing candidate license
  IDs and a shortened source-text hash. These remain custom references, not
  assertions of standard SPDX licenses. Complex notices may still need manual
  review, and scanner accuracy remains an upstream Syft concern.
- **Revisit when:** a released Syft version includes the upstream usability
  improvement. Upgrade the pin and check how candidate references and their
  extracted text appear in SPDX and Grant, then update the checksum-based
  reading guidance as appropriate.
