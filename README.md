# sign-image-fips

Sign a container image in FIPS mode from GitHub Actions, on standard hosted
runners. No FIPS-enabled kernel or self-hosted infrastructure required.

`sign-image-fips` builds cosign against a CMVP-validated cryptographic module,
signs your image, records the signature in the Rekor transparency log, and
verifies the result in a single composite action step.

## Approach

This action builds cosign with `GOFIPS140`, which links the Go Cryptographic Module 
(CMVP Certificate #5247) and makes the binary operate in FIPS mode. Because that is 
a property of the *binary*, not the host, the signing runs correctly on ordinary 
`ubuntu-latest`. A self-check verifies the compiled binary carries the validated module 
and defaults to FIPS mode, otherwise the action fails.

## Usage

```yaml
jobs:
  publish:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v7

      - name: Log in to GHCR
        uses: docker/login-action@v4
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      # ... build and push your image, capturing its digest ...

      - name: Sign image
        uses: usnistgov/sign-image-fips@v1
        with:
          image: ghcr.io/your-org/your-app@sha256:abc123...
          private-key: ${{ secrets.SIGNING_KEY }}
```

**Sign by digest, not tag.** A tag can later point at a different image than the
one you signed. Passing an immutable `@sha256:...` reference guarantees the
signature covers the exact image you built.

**The registry login is your responsibility.** The action assumes the job is
already authenticated to the registry it will push the signature to. It inherits
that session; it does not log in for you.

## Inputs

| Input                    | Required | Default                       | Description |
| ------------------------ | -------- | ----------------------------- | ----------- |
| `image`                  | yes      | --                            | Image reference to sign. Use a digest. |
| `private-key`            | yes      | --                            | PEM signing key. Passed via env, not the command line. |
| `public-key`             | no       | (derived)                     | PEM public key for verification. If omitted, derived from the private key. |
| `rekor-server`           | no       | `https://rekor.sigstore.dev`  | Transparency log URL. Used when `tlog-upload` is true. |
| `tlog-upload`            | no       | `true`                        | Publish the signature to the transparency log. Set `false` for an offline run that doesn't write a permanent entry. |
| `verify-inclusion-proof` | no       | `true`                        | Run an independent rekor-cli Merkle inclusion-proof check. Only meaningful when `tlog-upload` is true. |
| `cosign-release`         | no       | `v2.4.1`                      | cosign version to build. |
| `go-version`             | no       | `1.25.x`                      | Go toolchain. Must be 1.24+ for `GOFIPS140`. |
| `gofips140-version`      | no       | `v1.0.0`                      | Frozen Go Crypto Module snapshot (v1.0.0 = CMVP #5247). |
| `rekor-cli-release`      | no       | `v1.3.10`                     | rekor-cli version (inclusion-proof check only). |
| `rekor-cli-sha256`       | no       | (pinned)                      | SHA-256 of the rekor-cli linux-amd64 binary. |

## Outputs

| Output            | Description |
| ----------------- | ----------- |
| `public-key-file` | Path to the public key used for verification. |

## Key handling

You provide a standard PEM signing key. Internally the action imports it into
cosign's key format, signs, and shreds the on-disk key material; the private key
travels via an environment variable and is never placed on the command line.

Use a signing key you control. For CI you may prefer a dedicated key whose public
half you publish. The action can derive the public key from the private key, so 
supplying `public-key` is optional -- provide it explicitly when you want to publish 
a stable verification key.

## Verifying a signed image

With the default (public) transparency log, verification needs only your public
key -- the Rekor trust root ships with cosign via TUF:

```bash
cosign verify --key cosign.pub ghcr.io/your-org/your-app@sha256:...
```

If you point `rekor-server` at a **private** Rekor, verifiers must also supply
that instance's public key through `SIGSTORE_REKOR_PUBLIC_KEY`, because cosign's
verify path trusts Rekor via TUF and does not honor `--rekor-url` on verify.

## Transparency log behavior

By default every signature is published to the public Rekor log. Those entries
are permanent and world-readable by design and are expected when signing 
publicly-available images.

For testing, set `tlog-upload: false` to sign without writing a permanent entry. 
Verification then runs with `--insecure-ignore-tlog` and the inclusion-proof check 
is skipped automatically.

## Requirements

- A standard GitHub-hosted runner (`ubuntu-latest`, amd64) -- or any runner with
  Go 1.24+ available for the toolchain step.
- The job authenticated to the target registry before this step.
- `jq` and `curl` on the runner (present on GitHub-hosted images).

## How it works (step by step)

1. **Preflight** -- validate inputs. (No kernel FIPS check; not needed.)
2. **Set up Go** and **build cosign** with `GOFIPS140` (cached across runs).
3. **FIPS self-check** -- confirm the built cosign carries module #5247 and
   defaults to FIPS mode; fails otherwise.
4. **Sign** -- import the key, sign the image, and (by default) upload the tlog
   entry, all through the validated module.
5. **Verify** -- cosign checks the signature and, when applicable, the tlog SET.
6. **Inclusion proof** (optional) -- rekor-cli independently verifies the Merkle
   inclusion proof.