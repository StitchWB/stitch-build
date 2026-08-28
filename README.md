# stitch-build

Public build farm for the **Stitch Manager** open-core project.

This repository exists so that the heavy [Nuitka] compilation of Stitch
provider plugins runs on **free public-repo GitHub Actions minutes** instead of
the private monorepo's limited quota. It holds **no product source code** — the
private source is checked out read-only at build time and is never committed
here.

## How it works

```
StitchWB/Stitch-Account-Manager (private)          stitch-build (this repo)
        │  read-only fine-grained token                     │
        └──────────────► checkout ──────────────────────────┤
                                                            │ pack + Nuitka compile
                                                            │ zip
                                                            │ ENCRYPT (artifact-pub.pem)
                                                            ▼
                                            GitHub Release  plugins-<version>
                                            (*.zip.enc — encrypted assets)
                                                            │
        ┌─────────────── download (public URL) ─────────────┘
        │  decrypt (private key, Actions secret)
        │  sign (ed25519)  →  publish to distribution server
        ▼
StitchWB/Stitch-Account-Manager (private)
```

## Security model

- **Source never lands here.** The monorepo is fetched read-only via a
  fine-grained token with `Contents: Read` on that repo only. Nothing from it
  is committed to this repo.
- **Artifacts are encrypted.** Each compiled plugin zip is encrypted with
  `artifact-pub.pem` (RSA-OAEP + AES-256-GCM, see
  `stitch_plugin_tools.artifact_crypto` in the monorepo) before being uploaded.
  The release assets are therefore safe to sit in a public channel — without the
  private key (held **only** as an Actions secret in the private monorepo) they
  are useless.
- **No signing/publish secrets here.** This repo never holds the plugin signing
  key or the distribution admin key. Signing and publishing happen in the
  private monorepo after decryption.

## Residual trust

A read-only token to the private monorepo is stored as a secret here. It is
masked in logs and never passed to fork pull requests, and only this repo's
maintainers can trigger the workflow. If you are a maintainer, keep write access
to this repo tightly scoped.

## Triggering a build

Actions → **Build provider plugins** → *Run workflow* (optionally pass a
semver `version`; defaults to `YYYY.M.D+<short-sha>`). The encrypted assets are
published to a release tagged `plugins-<version>`.

[Nuitka]: https://nuitka.net/
