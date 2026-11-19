<!-- Documents reproducible, attested binary releases for Cargo Wrapper. -->
# Releasing Cargo Wrapper

Releases use checksum-pinned `cargo-dist` 0.32.0 and GitHub Actions. A `vMAJOR.MINOR.PATCH` tag builds native archives on Linux, macOS and Windows, generates checksums and a source archive, attests every published file, and creates the GitHub Release. No installer is generated: users verify and extract an archive, then place the binary as `cargo` ahead of the real Cargo on `PATH` (see the README, "Install a precompiled release").

## Release targets

- `x86_64-unknown-linux-gnu`
- `aarch64-unknown-linux-gnu`
- `x86_64-apple-darwin`
- `aarch64-apple-darwin` (built on `macos-15`, because GitHub no longer provides the default `macos-14` runner)
- `x86_64-pc-windows-msvc`

## Build the host archive locally

```sh
./scripts/install-pinned-dist.sh
dist --version
dist plan --tag=v0.2.3
dist build --artifacts=local --target=x86_64-unknown-linux-gnu --tag=v0.2.3
```

Use the version from `Cargo.toml`. Add `--allow-dirty` only for a private smoke test. Artifacts appear in `target/distrib/`.

```sh
cd target/distrib
shasum -a 256 -c cargowrapper-x86_64-unknown-linux-gnu.tar.xz.sha256
tar -tJf cargowrapper-x86_64-unknown-linux-gnu.tar.xz
```

A single host cannot build all native targets. The GitHub workflow supplies the Linux, macOS and Windows matrix.

## Cut a release

Make the version bump part of the pull request that is squash-merged, so the default branch has one commit for the release. Then, on the merged default branch:

```sh
cargo test --all-targets --locked
cargo clippy --all-targets --locked -- -D warnings
cargo fmt --all --check
dist plan --tag=v0.2.3
git tag -a v0.2.3 -m "Cargo Wrapper 0.2.3"
git push origin v0.2.3
```

The workflow publishes only matching version tags. Any verification, build, packaging or attestation failure prevents publication.

## Verification after publishing

When the Release workflow succeeds, the separate `verify-published` workflow (`.github/workflows/verify-release.yml`) runs on a native runner for each target. It downloads the published archive, checks its checksum and its build provenance with `gh attestation verify`, unpacks it, and runs `cargowrapper wrapper`. It runs after publishing, so a failure cannot un-publish a release; it tells the maintainer that a published archive is broken or mismatched. It lives in its own file so regenerating `release.yml` with `cargo-dist` never removes it.

## Release trust model

The workflow downloads the exact cargo-dist 0.32.0 executable and checks its repository-pinned SHA-256 digest. External GitHub Actions are pinned to full commits. The publishing job alone receives attestation identity-token permission.

After downloading an artifact, verify both checksum and repository provenance:

```sh
gh attestation verify \
  cargowrapper-x86_64-unknown-linux-gnu.tar.xz \
  --repo danielemiliogarcia/cargowrapper \
  --signer-workflow danielemiliogarcia/cargowrapper/.github/workflows/release.yml
```

Checksums detect corruption and attestations identify the producing repository and workflow. Neither proves that source code is safe.
