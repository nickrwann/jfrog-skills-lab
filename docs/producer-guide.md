# Producer guide

This guide covers managing a skill as its producer: authoring the source,
publishing versions, controlling who may publish, and maintaining releases.

A producer is the person or team responsible for what a skill does and which
versions are made available to other people. Producing is broader than writing
`SKILL.md`: it includes reviewing changes, deciding when a version is ready,
publishing it under a known identity, and correcting or retiring it when
necessary.

## Lifecycle at a glance

1. **Author:** Develop the skill as ordinary source code and documentation.
2. **Review:** Confirm that the instructions and included files are safe and
   useful.
3. **Identify:** Authenticate to Artifactory as the person or automation doing
   the release.
4. **Release:** Assign a version and publish a packaged snapshot.
5. **Verify:** Confirm that the exact package can be found and installed.
6. **Distribute:** Allow approved consumers to discover and download it.
7. **Maintain:** Publish newer versions or deliberately remove a bad release.

The lifecycle separates editable work from released work. This lets consumers
depend on a known snapshot while producers continue developing the next one.

## Responsibility model

| System | Producer responsibility |
| --- | --- |
| Git | Editable source, reviews, ownership rules, commits, and release tags |
| Artifactory | Authenticated packages, versions, metadata, permissions, audit events, and distribution |
| JFrog CLI | Packaging, publishing, listing, searching, and deleting releases |

Artifactory is the package registry, not the source editor. Keep the canonical
skill in Git and publish release artifacts from a clean, reviewed source tree.

In less technical terms, Git is the workshop where the skill is built and its
history is discussed. Artifactory is the controlled distribution shelf where
finished, labeled versions are placed. The JFrog CLI carries a release from the
workshop to that shelf and later helps consumers retrieve it.

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

The goal is to make the skill understandable on its own. Someone reviewing the
source should be able to see its instructions and supporting behavior without
also needing access to the producer's local machine or private credentials.

## Manifest and authorship

The manifest is the skill's label. It tells people what the package is called,
what it is for, who presents themselves as its author, and which release they
are looking at. Good metadata helps people make sense of the catalog before
they install anything.

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

This distinction matters because anybody who can edit a file could type a name
into `author`. Artifactory therefore does not treat that text as proof. It uses
the logged-in account and its recorded permissions when deciding whether a
publish action is allowed. Teams can still use the author field for human
credit while relying on authenticated activity for operational accountability.

Artifactory indexes manifest values such as name, version, description, author,
tags, display name, summary, changelog, and fingerprint as `skill.*`
properties for package display and search.

## Release workflow

A release is a frozen, named snapshot of the skill. Consumers should be able to
install version `1.2.0` today or later and receive the same contents. New work
belongs in a new version instead of silently changing what an old version
means.

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

Semantic versions communicate the expected size of a change:

- A patch release such as `1.2.1` normally fixes behavior without changing the
  skill's intended contract.
- A minor release such as `1.3.0` normally adds compatible capability.
- A major release such as `2.0.0` signals that consumers may need to review or
  adapt how they use the skill.

JFrog orders the versions, but the producer is responsible for applying those
meanings consistently.

Artifactory stores releases at:

```text
<slug>/<version>/<slug>-<version>.zip
```

Treat each published version as immutable. Replacing the same path requires
Delete/Overwrite permission and weakens reproducibility; normally fix the
source and publish a new version.

## Correcting or removing a release

Removing a release is similar to recalling a shipped product: it can protect
future consumers, but it cannot erase copies that people already installed.
Use deletion for material mistakes, unsafe content, or policy violations—not
as the normal way to revise a skill. Publish a corrected version and explain
the change whenever that is sufficient.

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

Access control answers **what is this identity allowed to do?** This is also
called authorization. It is separate from authentication, which answers **who
is making the request?** A person can successfully log in but still lack
permission to publish or delete a skill.

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

The practical goal is to limit the consequences of mistakes. A regular
publisher usually needs permission to add new versions but not permission to
rewrite or delete existing ones. A smaller maintainer group can hold the more
powerful Delete/Overwrite permission for exceptional corrections.

For a complete explanation of identities, groups, project roles, permission
targets, maintainers, CI publishers, tokens, and audit records, see
[Authentication, RBAC, and maintainership](authentication-and-rbac.md).

## Authentication

Authentication is how Artifactory knows which human or automated process is
talking to it. Think of the access token as a temporary machine-readable badge:
the token represents an identity, and Artifactory looks up that identity's
permissions before accepting the command.

The token is not the author name and is not part of the skill. It should be
handled like a password. A named CLI profile remembers the server and stores
the credential in JFrog's local configuration so producers do not paste it
into every command or accidentally commit it with the source.

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

A personal identity makes it clear which person performed an interactive
release. A service identity makes automated releases traceable to a particular
pipeline rather than to whichever developer originally configured it. Tokens
should be revocable and limited in scope and lifetime where practical.

The detailed authentication guide includes end-to-end examples for a solo
producer, a team, and an automated release pipeline:
[Authentication, RBAC, and maintainership](authentication-and-rbac.md).

## Signing and scanning

Signing and scanning answer different trust questions:

- **Signing asks:** did this package come through the expected producer and has
  the signed evidence remained intact?
- **Scanning asks:** does the package appear to contain risky or malicious
  content?

Neither one makes a skill logically correct. Review and testing still matter,
but signing and scanning add useful checks around the release process.

JFrog CLI supports evidence signing during publish with `--signing-key` and
`--key-alias`. Consumers automatically verify available evidence during
installation. Keep private signing material outside Git and inject it through a
protected local or CI secret store.

The CLI can also request a post-publish Xray scan. Scanning capabilities and
semantic skill analysis depend on the installed JFrog products and license. Do
not assume the open-beta Skills package type automatically includes every
security product.

## Verification and distribution

Publishing successfully only proves that the upload completed. A producer
should also verify the consumer experience: confirm the package appears in the
catalog, install it into a clean test workspace, and check that the intended
files and version arrived. This catches packaging mistakes that source-only
tests can miss.

Distribution is controlled access to the released package. Granting consumers
Read permission lets them discover and install it without also allowing them
to publish, edit metadata, overwrite releases, or delete anything.

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
