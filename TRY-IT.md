# Try Cargo Wrapper without installing it

A hands-on walkthrough of the whole flow: the core wrapper, the Supply Chain Guard plugin, and
minimum publish age. Nothing here is permanent. The wrapper is on `PATH` only in the shell you
open, and the plugin registry is a temporary directory. When you are done, close the shell and
delete one folder.

These commands are for Linux and macOS shells (bash or zsh). On Windows the same ideas apply, but
the commands differ; see the [README](README.md) for the Windows install steps.

## What you need

- Rust installed with `rustup`, so the real Cargo is at `$HOME/.cargo/bin/cargo`.
- Cargo 1.100 or newer for the minimum-publish-age part (the rest works on older Cargo).
- A clone of this repository, and (for part 4) a clone of
  [cargo-wrapper-supply-chain-guard](https://github.com/danielemiliogarcia/cargo-wrapper-supply-chain-guard).

## 1. Build the binaries with the real Cargo

Use your normal Cargo for this step, before the wrapper is on `PATH`.

```sh
export CW_CORE=/path/to/cargowrapper
export CW_GUARD=/path/to/cargo-wrapper-supply-chain-guard

(cd "$CW_CORE" && cargo build --release)
(cd "$CW_GUARD" && cargo build --release)    # only needed for part 4
```

## 2. Open a shell with the wrapper first on `PATH`

```sh
export DEMO=$HOME/cw-demo
export CARGO_WRAPPER_PLUGIN_DIR=$DEMO/plugins     # throwaway plugin registry
mkdir -p "$DEMO/bin" "$DEMO/plugins"
cp "$CW_CORE/target/release/cargowrapper" "$DEMO/bin/cargo"
export PATH="$DEMO/bin:$HOME/.cargo/bin:$PATH"    # wrapper first, real Cargo still reachable

cargo wrapper                                     # prints: Cargo wrapper is active.
which cargo                                       # should be $HOME/cw-demo/bin/cargo
```

Do not put the wrapper in `$HOME/.cargo/bin`. It forwards to the real Cargo there.

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

## 4. Add the Supply Chain Guard plugin

```sh
cargo wrapper plugin install supply-chain-guard \
  "$CW_GUARD/target/release/cargo-wrapper-plugin-supply-chain-guard"
cargo wrapper plugin enable supply-chain-guard
cargo wrapper plugin list
```

Now dependency changes go through review:

```sh
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
- The verified release installer bundles a pinned cargo-vet helper. A binary you built from source
  does not have it, so cargo-vet evidence may be missing and the guard falls back to its static
  review. For real use, install the verified release as described in the guard's README.

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

## Next steps

- Install the wrapper for real: `cargo run -- wrapper-install` from this repository. It shows its
  plan and asks before changing anything.
- Install Supply Chain Guard from a verified release: see its README.
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
