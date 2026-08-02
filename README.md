# setup-aura

Installs the [Aura](https://github.com/aura-config/aura-lang) configuration-language
CLI and puts it on `PATH`.

```yaml
- uses: aura-config/setup-aura@v1
  # `version` defaults to `latest`. Pin it — `version: "0.1.1"` — when a
  # reproducible pipeline matters more than picking up fixes automatically.

- run: aura check deploy.aura --strict
```

## What it does

1. Resolves the version — `latest`, `v0.1.0` and `0.1.0` are all accepted.
2. Maps the runner to a release target triple.
3. Downloads the archive **and its `.sha256`**, and verifies the checksum before
   unpacking. The language pins its own dependencies by hash in `aura.lock`; an
   installer that skipped the same check would be the weak link.
4. Unpacks and appends the directory to `GITHUB_PATH`.

## Inputs

| Input | Default | |
| --- | --- | --- |
| `version` | `latest` | `latest`, `v0.1.0` or `0.1.0` |
| `github-token` | `${{ github.token }}` | Used for the release API |

## musl on Linux

On x86_64 Linux this installs the **musl** build rather than the gnu one.

A gnu binary is compiled against the builder's glibc and then refuses to start on
anything older — the usual way a released Rust tool breaks for its first users.
Re-verified against the 0.1.1 artifacts: the gnu build requires `GLIBC_2.34` and does
not reach `main` on Ubuntu 20.04, Debian 11, CentOS 8 or Amazon Linux 2. The musl
build is `static-pie` and has no floor at all.

It is not a trade against speed, which was worth checking rather than assuming:
evaluating a real manifest on ext4 took 1.12 ms with musl against 1.40 ms with
gnu. A static binary skips the dynamic loader, and over a process this short-lived
that outweighs musl's slower allocator.

aarch64 Linux stays on gnu only because no aarch64-musl artifact is published yet.

## Usage in CI

Two recipes that cover most of what a pull request wants to know:

```yaml
# Analysis only — fast, and it never evaluates anything.
- run: aura check manifests/*.aura --strict

# Prove a manifest performs no I/O at all. Decided statically, so it holds for
# branches this run would not have taken.
- run: aura check --hermetic manifests/*.aura

# Evaluate exactly as locked, without writing anything.
- run: aura eval deploy.aura --frozen --dry-run
```

## Releases

Binaries live in [`aura-config/aura-lang`](https://github.com/aura-config/aura-lang/releases)
— this action only fetches them, the same split as `setup-node` and nodejs.org.
This repository is versioned independently: `@v1` moves as the installer changes,
regardless of the language's own version.

## Licence

MIT or Apache-2.0, at your option — the same as Aura itself.
