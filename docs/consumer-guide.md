# Consumer guide

This guide covers discovering, installing, pinning, and updating skills from an
Artifactory Skills repository.

A consumer is a person, project, or automated environment that uses a released
skill without controlling its publication. The consumer's main decisions are
whether to trust the source, where to install the skill, which version to use,
and when to accept an update.

## Configure the registry

Configuration tells the CLI which Artifactory server to contact and which
identity to present. It is the registry equivalent of signing in to a private
app. The identity normally needs only Read permission because a consumer is
retrieving packages, not changing the catalog.

Configure a named JFrog CLI profile using an identity with Read permission:

```bash
jf config add local-skills \
  --url http://localhost:8082 \
  --access-token "$JFROG_ACCESS_TOKEN" \
  --interactive=false

jf config use local-skills
```

Never place the access token in the repository.

## Discover skills

Discovery is browsing the approved internal catalog. Search and list commands
show what the current identity is allowed to see; a missing result can mean the
skill does not exist, but it can also mean the consumer lacks access.

```bash
jf agent skills list --repo skills-local
jf agent skills search "keyword" --repo skills-local
```

## Install for Codex

Installation downloads a released snapshot and copies it into a directory the
selected coding agent understands. It is not a live link back to Artifactory:
the local copy stays at its installed version until an update command changes
it.

Install the latest version into the current project's `.codex/skills`:

```bash
jf agent skills install my-skill \
  --harness codex \
  --project-dir . \
  --repo skills-local
```

Pin an exact version when reproducibility matters:

```bash
jf agent skills install my-skill \
  --harness codex \
  --project-dir . \
  --repo skills-local \
  --version 1.2.0
```

For a cross-agent installation, use `--harness cross-agent`; JFrog installs it
under `.agents/skills`. A global Codex installation uses `--harness codex
--global` and installs under `~/.codex/skills`.

Project installation keeps the dependency close to one codebase and makes its
scope visible. Global installation makes a skill available across projects but
also increases the number of places affected by an update. Use the narrowest
scope that matches the intended audience.

Pinning is like recording an exact dependency version: it favors repeatability.
Installing the latest version favors convenience and faster uptake of producer
changes. Neither policy is universally right; stable automation usually
benefits from pins, while an experimental personal workspace may prefer the
latest release.

## Track and update installs

Artifactory does not silently rewrite the installed directory. The local
metadata file acts like a receipt: it records where the skill came from and
which version was installed so the CLI can later compare it with the registry.

The CLI writes `.jfrog/skill-info.json` inside the installed skill. It records
the source repository, installed version, harness, scope, and project directory
for version comparisons and updates.

Check installed skills for newer registry versions:

```bash
jf agent skills list --harness codex --check-updates
```

Preview and apply an update:

```bash
jf agent skills update my-skill \
  --harness codex \
  --project-dir . \
  --repo skills-local \
  --dry-run

jf agent skills update my-skill \
  --harness codex \
  --project-dir . \
  --repo skills-local
```

The default update target is the latest semantic version. A specific version
can be selected with `--version`. If replacement fails, the CLI restores the
previous installed copy from a temporary backup.

An update is therefore an explicit consumer decision. Previewing first is
useful in controlled environments, and major-version changes deserve the same
review given to other meaningful dependency changes.

## Trust and access

Trust is not a single checkbox. Consumers should consider who operates the
registry, which authenticated identity published the release, what the skill
claims in its metadata, whether evidence verifies, and whether the contents
have been reviewed or scanned. The visible author name alone is not proof of
origin.

- Read permission is normally sufficient for discovery and installation.
- Manifest `author` is descriptive metadata, not proof of the publisher's
  registry identity.
- When a producer publishes signed evidence, installation verifies it using
  public keys stored in Artifactory.
- Pin versions for stable automation; use `latest` only where automatic uptake
  is intentional.

## References

- [Skills repositories](https://docs.jfrog.com/artifactory/docs/skills-repositories)
- [JFrog CLI for Skills](https://docs.jfrog.com/artifactory/docs/jf-skills)
- [Skills and Agent Plugins CLI](https://docs.jfrog.com/artifactory/docs/agent-commands)
