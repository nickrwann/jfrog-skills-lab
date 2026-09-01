# Consumer guide

This guide covers discovering, installing, pinning, and updating skills from an
Artifactory Skills repository.

## Configure the registry

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

```bash
jf agent skills list --repo skills-local
jf agent skills search "keyword" --repo skills-local
```

## Install for Codex

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

## Track and update installs

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

## Trust and access

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
