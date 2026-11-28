# Kumikit plugin registry

The official registry of [Kumikit](https://github.com/kumikit/kumikit) plugins. Kumikit
reads only the index this repository's CI builds and publishes:

```
https://kumikit.github.io/registry/index.json
```

Kumikit fetches it only when you browse, search, install, update or check for updates.
It never fetches it at startup or in the background. The format is described in
[10-distribution](https://github.com/kumikit/kumikit/blob/main/docs/spec/10-distribution.md#registry).

## Layout

```
publishers/<publisher>.toml       # a publisher's key, and who may change its files
plugins/<publisher>/<name>.toml   # one plugin: metadata and versions
starter.toml                      # ids offered on the first-run screen
allow-confusable.toml             # reviewed exceptions to the confusable-id check
```

## Before 1.0

Kumikit's plugin interfaces are not frozen yet
([ADR 0020](https://github.com/kumikit/kumikit/blob/main/docs/decisions/0020-interface-freeze-after-m5.md)).
Until 1.0 this registry accepts only publishers marked `early = true`, added by a
maintainer. Open an issue if you would like to publish. Anyone can run their own registry
with the same checker.

## Publishing a plugin

1. **Make a publisher key, once.** Keep the private key out of every repository. It
   cannot be replaced, because users pin it on their first install.

   ```sh
   kumikit plugin keygen --name acme     # prints ed25519:…
   ```

2. **Pack and sign each release**, then attach the `.kkp` to a release (any `https` URL
   that will not change works):

   ```sh
   kumikit plugin pack --key <private key file> path/to/plugin
   kumikit plugin verify acme.probe-0.1.0.kkp
   sha256sum acme.probe-0.1.0.kkp
   ```

3. **Open a pull request.** For a new publisher, add `publishers/acme.toml`:

   ```toml
   name = "Acme"
   key = "ed25519:…"          # from keygen
   github = ["jane"]          # GitHub logins that may change Acme's files
   early = true
   ```

   For each plugin, add or extend `plugins/acme/probe.toml`:

   ```toml
   id = "acme.probe"
   name = "Probe"
   description = "One line."
   repository = "https://github.com/acme/kumikit-probe"
   license = "MIT"
   kind = ["provider"]

   [[versions]]
   version = "0.1.0"
   url = "https://github.com/acme/kumikit-probe/releases/download/v0.1.0/acme.probe-0.1.0.kkp"
   sha256 = "…"
   api = "^0.1"                                   # copied from the manifest
   permissions = { spawn = ["some-agent"] }       # copied from the manifest (core plugins)
   ```

   UI plugins copy `isolation` and `network` from the manifest's `[ui]`, and
   `requires-core` from its `[requires] core`, instead of `permissions`.

## What CI checks

Every pull request runs `kumikit-registry check`. It downloads each new version and
checks:

- the SHA-256, and that the package is signed by the publisher's key;
- the manifest, with the same validation as an install, and that the copied fields equal it;
- that the id starts with the publisher's name, and is not confusingly close to an existing one;
- that no publisher key changed, and no published version was changed or removed (a
  version can only be marked `yanked = true`);
- that the pull request's author is in the publisher's `github` list.

To run it yourself:

```sh
cargo install --locked --git https://github.com/kumikit/kumikit kumikit-registry-tool
kumikit-registry check . --require-early
```

On `main`, CI builds `index.json` and publishes it to GitHub Pages.
