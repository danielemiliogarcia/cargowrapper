# Try Cargo Wrapper without installing it

A hands-on walkthrough of the whole flow: the core wrapper, the Supply Chain Guard plugin, and
minimum publish age. Nothing is installed on your system. You do not clone any repository and you
do not compile anything: the verified release binaries are downloaded into one throwaway folder, and
you remove that folder with a single `rm -rf` when you are done. The wrapper is on `PATH` only in
the shell you open, and the plugin registry lives in that folder too.

These commands are for Linux and macOS shells (bash or zsh). On Windows the same ideas apply, but
the commands differ; see the [README](README.md) for the Windows install steps.

## What you need

- Rust installed with `rustup`, so the real Cargo is at `$HOME/.cargo/bin/cargo`. The wrapper
  forwards to it, and the demo project uses it. Cargo 1.100 or newer is needed for the
  minimum-publish-age part; the rest works on older Cargo.
- `curl`.
- The [GitHub CLI](https://cli.github.com/) (`gh`), signed in with `gh auth login`. It downloads the
  releases and verifies that each file was built by its repository's release workflow.

## 1. Set up the throwaway folder

```sh
export DEMO=$HOME/cw-demo
mkdir -p "$DEMO"/{bin,plugins,data,config,cache,tmp}

# Keep everything the tools write inside the demo folder.
export CARGO_WRAPPER_PLUGIN_DIR="$DEMO/plugins"
export XDG_DATA_HOME="$DEMO/data" XDG_CONFIG_HOME="$DEMO/config" XDG_CACHE_HOME="$DEMO/cache"
export TMPDIR="$DEMO/tmp"
```

## 2. Download the wrapper

Download the release archive for your platform, verify it, and put the binary in the demo folder
as `cargo`. No clone and no build.

```sh
CORE=danielemiliogarcia/cargowrapper
CORE_TAG=v0.2.2                                  # the release this guide was written for

case "$(uname -s)/$(uname -m)" in
  Linux/x86_64)          target=x86_64-unknown-linux-gnu ;;
  Linux/aarch64|Linux/arm64) target=aarch64-unknown-linux-gnu ;;
  Darwin/arm64)          target=aarch64-apple-darwin ;;
  Darwin/x86_64)         target=x86_64-apple-darwin ;;
  *) echo "No prebuilt binary for this platform; see the README for building from source." >&2 ;;
esac
archive="cargowrapper-$target.tar.xz"

cd "$TMPDIR"
gh release download "$CORE_TAG" --repo "$CORE" --pattern "$archive" --pattern "$archive.sha256"
echo "$(cut -d ' ' -f 1 "$archive.sha256")  $archive" | shasum -a 256 -c -
gh attestation verify "$archive" --repo "$CORE" \
  --signer-workflow "$CORE/.github/workflows/release.yml" \
  --source-ref "refs/tags/$CORE_TAG" --deny-self-hosted-runners

tar -xJf "$archive"
install -m 755 "cargowrapper-$target/cargowrapper" "$DEMO/bin/cargo"

export PATH="$DEMO/bin:$HOME/.cargo/bin:$PATH"   # wrapper first, real Cargo still reachable
cargo wrapper                                    # prints: Cargo wrapper is active.
which cargo                                      # should be $HOME/cw-demo/bin/cargo
```

Check the repository's releases page for a newer tag. Do not put the wrapper in
`$HOME/.cargo/bin`: it forwards to the real Cargo there. If you only want the download and
checksum, plain `curl -LO` on the release asset URLs works too, but then you skip the provenance
check.

## 3. The core wrapper on its own

```sh
cd "$DEMO" && cargo new hello && cd hello
cargo add itoa            # passes through; the core wrapper does not block add
cargo build               # runs `cargo build --locked`
cargo update              # refused with a warning
cargo forceupdate         # the explicit way to run `cargo update`
```

The wrapper adds `--locked` to `build`, `test`, `run`, `check`, `bench`, `clippy`, `doc`, `rustc`,
`rustdoc` and `install`. On Cargo 1.100+, `cargo install` already uses the packaged lockfile, so
`--locked` does nothing there. On older Cargo it makes install use the packaged lockfile.

## 4. Add the Supply Chain Guard plugin (verified release, into the same folder)

Download the installer from the guard's release, verify where it came from, then run it. The
installer verifies the plugin bundle the same way before it installs anything.

```sh
GUARD=danielemiliogarcia/cargo-wrapper-supply-chain-guard
GUARD_VERSION=0.2.0

cd "$TMPDIR"
gh release download "v$GUARD_VERSION" --repo "$GUARD" -p install.sh
gh attestation verify install.sh --repo "$GUARD" \
  --signer-workflow "$GUARD/.github/workflows/release.yml" \
  --source-ref "refs/tags/v$GUARD_VERSION" --deny-self-hosted-runners

chmod +x install.sh
CARGO_WRAPPER_SUPPLY_CHAIN_GUARD_VERSION="$GUARD_VERSION" ./install.sh

cargo wrapper plugin enable supply-chain-guard
cargo wrapper plugin list
cargo supply-chain-guard cargo-vet status         # "Helper: ready" means cargo-vet evidence works
```

Now dependency changes go through review:

```sh
cd "$DEMO/hello"
cargo add serde                            # blocked: dependency changes must use the force commands
cargo forceadd serde                       # resolves in isolation, then reviews the exact artifacts
cargo forceadd serde --dump-audit-prompt   # writes a batch-review report instead of prompting
cargo forceadd serde --accept-all-once     # accepts unresolved findings once, non-interactively
```

Other guarded commands: `cargo forcegenerate-lockfile`, `cargo forceremove <dep>`,
`cargo remove <dep>` and `cargo forceupdate`.

Notes:

- For crates that cargo-vet does not cover, you get an interactive review. `R` (reject) is the
  default, so pressing Enter is safe.
- Put the batch flags directly after the command. Do not put `--` before them.
- Crates already covered by cargo-vet audits are cleared without a prompt, and the evidence is
  printed instead.

## 5. Minimum publish age (Cargo 1.100 native)

Cargo 1.100 enforces a minimum publish age natively. Try it with an environment variable:

```sh
export CARGO_REGISTRY_GLOBAL_MIN_PUBLISH_AGE="30 days"
cargo forceadd syn        # picks versions that are at least 30 days old
```

Or check it in per project, in `.cargo/config.toml`:

```toml
[registry]
global-min-publish-age = "30 days"
```

### See the guard's warning for a policy it cannot see

The guard resolves candidates in a private `CARGO_HOME`, so a policy that lives only in your Cargo
home config is applied by the final Cargo run but not by the guard's resolution. The guard prints a
warning when it detects this. To see it without touching your real `$HOME/.cargo/config.toml`, use
a throwaway Cargo home:

```sh
unset CARGO_REGISTRY_GLOBAL_MIN_PUBLISH_AGE
mkdir -p "$DEMO/home" && ln -sfn "$HOME/.cargo/bin" "$DEMO/home/bin"
printf '[registry]\nglobal-min-publish-age = "30 days"\n' > "$DEMO/home/config.toml"
export CARGO_HOME="$DEMO/home" RUSTUP_HOME="${RUSTUP_HOME:-$HOME/.rustup}"
cd "$DEMO/hello"
cargo forceadd syn        # prints the warning
```

It warns only when Cargo is 1.100+, the environment variable is unset, the project's
`.cargo/config.toml` does not set the policy, and the Cargo home config does. Setting the
environment variable, or adding the setting to the project's `.cargo/config.toml`, silences it.

## 6. Clean up

Close the shell. The `PATH`, `CARGO_HOME` and plugin-directory changes disappear with it. Then:

```sh
rm -rf "$HOME/cw-demo"
```

That removes the wrapper, the plugin, its data and the demo project. Two tools keep their own small
caches outside the folder: Cargo, which caches the crates it downloads for the demo project
(`$HOME/.cargo/registry`), and the GitHub CLI (`$HOME/.cache/gh`). They are harmless and are
reused next time.

## Next steps

- Install the wrapper for real: follow "Install a precompiled release" in the [README](README.md).
  It uses the same verified download, into a directory on your `PATH` instead of a temporary folder.
- Install Supply Chain Guard permanently: see its README. The steps are the ones in part 4, without
  the temporary-folder variables.
- Cargo 1.99 or older: the legacy
  [age-policy](https://github.com/danielemiliogarcia/cargo-wrapper-age-policy) plugin provides the
  minimum publish age that Cargo does not enforce there. Cargo 1.100+ does not need it.

## Useful environment variables

| Variable | Purpose |
|---|---|
| `CARGO_WRAPPER_PLUGIN_DIR` | Where the wrapper keeps its plugin registry. Use a temp directory to experiment. |
| `CARGO_WRAPPER_LOG_FILE` | Log forwarded Cargo commands to a file. |
| `CARGO_REGISTRY_GLOBAL_MIN_PUBLISH_AGE` | Native Cargo 1.100 minimum publish age. Also visible to the guard's isolated resolution. |
| `CARGO_WRAPPER_SUPPLY_CHAIN_GUARD_CONFIG` | Override the guard's config file path. |
| `CARGO_WRAPPER_SUPPLY_CHAIN_GUARD_OFFLINE` | Skip the cargo-vet import refresh and use the cached snapshot. |
