# Producer guide

This guide covers managing a skill as its producer: authoring the source,
publishing versions, controlling who may publish, and maintaining releases.

## Responsibility model

| System | Producer responsibility |
| --- | --- |
| Git | Editable source, reviews, ownership rules, commits, and release tags |
| Artifactory | Authenticated packages, versions, metadata, permissions, audit events, and distribution |
| JFrog CLI | Packaging, publishing, listing, searching, and deleting releases |

Artifactory is the package registry, not the source editor. Keep the canonical
skill in Git and publish release artifacts from a clean, reviewed source tree.

## Suggested source layout

```text
skills/
└── my-skill/
    ├── SKILL.md
    ├── references/
    ├── scripts/
    └── assets/
```

Only add supporting directories that the skill actually needs. Never put
Artifactory tokens, signing-key passphrases, or private keys in the skill tree.

## Manifest and authorship

JFrog reads the semantic version and package metadata from the YAML frontmatter
in `SKILL.md`. A minimal producer-owned manifest can look like:

```yaml
---
name: my-skill
description: Describe when and why an agent should use this skill.
version: 0.1.0
author: Nick Wanner
---
```

The `author` value is descriptive metadata. It does not establish registry
ownership or grant publishing permission. The authenticated JFrog user or
access token, combined with Artifactory permissions, controls what the producer
may publish or change.

Artifactory indexes manifest values such as name, version, description, author,
tags, display name, summary, changelog, and fingerprint as `skill.*`
properties for package display and search.

## Release workflow

Use an explicit semantic version for every release:

1. Edit and test the skill locally.
2. Review and merge the source change in Git.
3. Update `version` and, when useful, the changelog metadata in `SKILL.md`.
4. Commit and tag the source with the same version.
5. Publish the reviewed directory:

   ```bash
   jf agent skills publish ./skills/my-skill \
     --repo skills-local
   ```

6. Confirm the package in Artifactory or from the CLI:

   ```bash
   jf agent skills list --repo skills-local
   ```

The CLI can override the manifest version:

```bash
jf agent skills publish ./skills/my-skill \
  --repo skills-local \
  --version 1.2.0
```

Prefer keeping the Git tag, `SKILL.md`, and published package version identical
instead of relying on an override. If the manifest omits a version, interactive
publishing prompts for one. Quiet/CI publishing can choose the next minor
semantic version automatically, but an explicit release version is easier to
review and reproduce.

Artifactory stores releases at:

```text
<slug>/<version>/<slug>-<version>.zip
```

Treat each published version as immutable. Replacing the same path requires
Delete/Overwrite permission and weakens reproducibility; normally fix the
source and publish a new version.

## Correcting or removing a release

Preview deletion first:

```bash
jf agent skills delete my-skill \
  --version 1.0.0 \
  --repo skills-local \
  --dry-run
```

Then omit `--dry-run` only when the exact version should be removed. The delete
command otherwise acts immediately without an interactive confirmation.

## Producer access control

Admin or Project Admin permission is required to create the local Skills
repository. After creation, use groups and narrowly scoped permission targets
instead of making every producer an administrator.

| Role | Suggested repository permissions |
| --- | --- |
| Consumer | Read |
| Skill publisher | Read, Deploy/Cache |
| Release maintainer | Read, Deploy/Cache, Annotate, Delete/Overwrite |
| Permission delegate | Manage plus only the operational permissions needed |

- Read permits listing and downloading packages.
- Deploy/Cache permits publishing new artifact paths.
- Annotate permits changing artifact properties.
- Delete/Overwrite permits deleting releases and replacing existing paths.
- Manage permits delegating non-Manage permissions within the permission
  target; it does not make someone a platform administrator.

Permission targets can apply to users or groups and can use repository path
include/exclude patterns. That supports separate teams or namespaces within a
repository, although separate repositories may be easier to reason about when
teams need strongly isolated release authority.

## Authentication

Configure a named JFrog CLI profile with an identity/access token rather than
putting credentials in the repository:

```bash
jf config add local-skills \
  --url http://localhost:8082 \
  --access-token "$JFROG_ACCESS_TOKEN" \
  --interactive=false

jf config use local-skills
jf config show
```

Use a personal token for interactive development and a narrowly scoped service
identity/token for CI publishing. The token identity—not the manifest's
`author` string—is the security principal Artifactory evaluates.

## Signing and scanning

JFrog CLI supports evidence signing during publish with `--signing-key` and
`--key-alias`. Consumers automatically verify available evidence during
installation. Keep private signing material outside Git and inject it through a
protected local or CI secret store.

The CLI can also request a post-publish Xray scan. Scanning capabilities and
semantic skill analysis depend on the installed JFrog products and license. Do
not assume the open-beta Skills package type automatically includes every
security product.

## Producer checklist

- Source reviewed and committed.
- Explicit semantic version set.
- Git tag and package version agree.
- No credentials or private signing material included.
- Publisher uses a named, least-privilege identity.
- Existing versions are not overwritten during a normal release.
- Published package is listed and install-tested from a consumer workspace.

## References

- [Skills repositories](https://docs.jfrog.com/artifactory/docs/skills-repositories)
- [JFrog CLI for Skills](https://docs.jfrog.com/artifactory/docs/jf-skills)
- [JFrog permissions](https://docs.jfrog.com/administration/docs/permissions)
- [Permission targets](https://docs.jfrog.com/artifactory/docs/permission-targets)
