# Promptfile

A Promptfile declares prompt dependencies for a project. It specifies where prompts come from, how to fetch them, and how to verify their authenticity.

Prompts are the highest-privilege input to an LLM system. A Promptfile makes authorship verifiable and tampering auditable.

Full spec at [promptfile.md](https://promptfile.md).

## Format

A Promptfile is a TOML file named `Promptfile` with no extension.

```toml
version = 1

[prompts]
coder    = {git = "https://github.com/alice/prompts", ref = "v1", path = "coder"}
reviewer = {git = "https://github.com/alice/prompts", release = "v1", asset = "reviewer.tar.gz"}
claude   = {git = "https://github.com/alice/claude-md", ref = "v1", inline = true}
```

### Version

`version = 1` is required. The only supported version.

### Prompts

Each entry under `[prompts]` maps an alias to a source. The alias is the local name used to reference the dependency.

### Source modes

Every entry must have a `git` URL and exactly one of `ref` or `release`.

**Clone mode** (`ref`) clones the repository and checks out the specified ref. The ref is auto-detected as a tag, branch, or commit SHA.

```toml
coder = {git = "https://github.com/alice/prompts", ref = "v1"}
```

**Release mode** (`release`) downloads a signed tarball from a GitHub or Codeberg release. Requires a `.sigstore.json` bundle alongside the tarball. Unsigned releases are rejected unless the implementation provides a force flag.

```toml
reviewer = {git = "https://github.com/alice/prompts", release = "v1"}
```

### Optional fields

| Field    | Applies to   | Description                                                                 |
|----------|-------------|-----------------------------------------------------------------------------|
| `path`   | clone only  | Subdirectory within the repo to extract                                     |
| `asset`  | release only| Non-standard asset filename (default: `{repo}.tar.gz`)                      |
| `inline` | both        | Place a single-file prompt directly in the working directory instead of a subdirectory |

### Constraints

- `ref` and `release` are mutually exclusive
- `path` is invalid on release entries
- `asset` is invalid on clone entries
- `git` must be an HTTPS or SSH URL pointing to a git repository

## Lockfile

Resolving dependencies produces a `Promptfile.lock` that pins exact versions:

```toml
version = 1

[[prompt]]
name     = "coder"
source   = "git"
git      = "https://github.com/alice/prompts"
ref      = "v1"
ref_type = "tag"
commit   = "a1b2c3d4e5f6..."
path     = "coder"
digest   = "sha256:..."

[[prompt]]
name     = "reviewer"
source   = "release"
git      = "https://github.com/alice/prompts"
release  = "v1"
asset    = "reviewer.tar.gz"
digest   = "sha256:..."
signer   = "alice@github.com"
```

The lockfile records:

- **commit**: the resolved commit SHA (clone mode)
- **ref_type**: the auto-detected ref type (`tag`, `branch`, or `commit`)
- **digest**: SHA-256 content digest (both modes)
- **signer**: identity from the Sigstore certificate (release mode)

## Signing

Release mode uses [Sigstore](https://sigstore.dev) keyless signing. Authors sign tarballs with `cosign sign-blob`, producing a `.sigstore.json` bundle containing the signature, certificate, and transparency log proof.

The bundle is self-contained. Verification requires no contact with the author or any registry. The signer's identity (email or OIDC URI) is extracted from the Fulcio certificate embedded in the bundle.

## Implementations

- [impromptu](https://github.com/calebfaruki/impromptu) -- Go CLI
