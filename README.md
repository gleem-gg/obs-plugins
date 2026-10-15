# Gleem OBS plugin catalog

The OBS plugins a renter can pick for a [Gleem OBS Hosting](https://gleem.gg/hosting) rental, and how they are built.

Plugins are not part of the runtime image. When a rental starts, the machine agent downloads the plugins the renter picked from `get.gleem.gg/obs-plugins/`, checks each one against the hash in the signed index, and mounts it read-only into the rental container. There, runtime-init points OBS at it.

## Recipes

Each catalog plugin is one file in `recipes/`:

| Field | |
|---|---|
| `id` | Catalog id, `[a-z0-9-]`. What the renter's choice is stored as. |
| `name`, `description` | Shown on the rental form. |
| `version`, `revision` | Upstream version, plus our build revision. Bump `revision` to ship a rebuild of the same upstream version. Published files never change. |
| `repo`, `commit` | Upstream source, pinned by commit. |
| `license`, `homepage` | Shown next to the plugin. |
| `cmake` | Extra CMake arguments. |

The build arguments come from the plugin's Flathub manifest (`flathub/com.obsproject.Studio.Plugin.*`) where one exists. The plugins are GPL. This repository and the pinned upstream commits are the corresponding source for every binary we publish.

## ABIs

A plugin only fits the OBS build it was compiled against. `abis` lists the ABI keys to build for. A key names one OBS build of the runtime image (`obs-<version>-<base>`), as the image records it in `/etc/gleem/obs-abi` and in its `gg.gleem.obs-abi` label. Every build runs in `ghcr.io/gleem-gg/obs-runtime-sdk:<abi>`, which is the `obs` stage of [obs-runtime](https://github.com/gleem-gg/obs-runtime) that the runtime image's OBS comes from.

When the runtime moves to a new OBS release:
1. Add its key to `abis`.
2. Let CI publish the catalog for it.
3. Only then point the website at it (`HOSTING_RUNTIME_OBS_ABI`, together with `HOSTING_RUNTIME_DIGEST`).

## Building locally

```
scripts/build obs-32.2.2-trixie recipes/advanced-masks.json
```

This writes `dist/<abi>/<id>-<version>.tar.zst` and its catalog entry. To build against images you built yourself, set `SDK_IMAGE` and `RUNTIME_IMAGE`. Before packaging, the script checks with `ldd` inside the runtime image that every library the plugin needs is there.

Each archive is laid out the way it is mounted at `/opt/gleem/obs-plugins/<module>`:

```
bin/<module>.so
data/...
LICENSE
```

## Publishing

CI runs `scripts/publish` for every ABI on each push to `main` that changes `abis`, `recipes/` or `scripts/`. It builds only versions that are not published yet. It then writes a new `index-<serial>.json`, signs it with Gleem's release key (trusted comment `obs-plugins <abi> <serial>`) and moves `<abi>/latest` to the new serial. The website reads `latest`, checks the signature and trusted comment, and offers what the index lists.
