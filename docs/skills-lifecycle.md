# Skills registry architecture and lifecycle

## Responsibilities

| Component | Responsibility |
| --- | --- |
| Git repository | Editable source, review history, branches, and collaboration |
| Artifactory Skills repository | Authenticated package storage, metadata indexing, search, versions, permissions, and downloads |
| JFrog CLI | Publish, install, update, list, search, and delete skill versions |
| Coding-agent harness | Loads an installed copy from its native skills directory |

Artifactory is a registry, not a source editor. Edit and review a skill in Git,
then publish a packaged version to Artifactory.

## Version workflow

1. Edit and test the skill source locally.
2. Set an explicit semantic version in the `SKILL.md` frontmatter.
3. Commit and tag the source version in Git.
4. Publish it:

   ```bash
   jf agent skills publish ./path/to/skill --repo skills-local
   ```

5. Install or update a consumer:

   ```bash
   jf agent skills install my-skill \
     --harness codex \
     --project-dir . \
     --repo skills-local

   jf agent skills update my-skill \
     --harness codex \
     --project-dir . \
     --repo skills-local
   ```

Artifactory stores each package at
`<slug>/<version>/<slug>-<version>.zip`. The CLI normally reads `version` from
`SKILL.md`; `--version` can override it. If a version is omitted in interactive
use, the CLI prompts for one. In quiet or CI mode it can select the next minor
version automatically.

Treat published versions as immutable. Replacing an existing path requires
Artifactory's Delete/Overwrite permission. Prefer publishing a new semantic
version so consumers and audits have a stable history.

After installation, the CLI writes `.jfrog/skill-info.json` inside the
installed skill. It uses that record to compare registry versions and target
updates. Failed updates restore the previous installed copy from a temporary
backup.

## Identity and access

The `author` value parsed from `SKILL.md` is descriptive package metadata. It
does not grant ownership or publishing rights.

Actual authority comes from the JFrog identity used by the UI, CLI, or REST
API. Artifactory can manage users, groups, access tokens, projects, roles, and
permission targets. For a Skills repository, the relevant repository
permissions are:

| Permission | Typical Skills use |
| --- | --- |
| Read | Search, list, download, and install |
| Deploy/Cache | Publish a new skill version |
| Delete/Overwrite | Delete a version or replace an existing version path |
| Annotate | Change artifact properties/metadata |
| Manage | Delegate most permissions within a permission target |

Admin or Project Admin permission is required to create the Skills repository.
For a team, prefer groups and narrowly scoped permission targets rather than
granting every publisher administrative access. Repository include/exclude
patterns can narrow access to selected artifact paths.

## Metadata

Artifactory parses YAML frontmatter from `SKILL.md` and indexes values such as
name, version, description, author, tags, display name, summary, changelog, and
fingerprint as `skill.*` properties. Those values drive package display and
search; they should not be confused with authenticated user identity.

## References

- [Skills repositories](https://docs.jfrog.com/artifactory/docs/skills-repositories)
- [JFrog CLI for Skills](https://docs.jfrog.com/artifactory/docs/jf-skills)
- [Skills and Agent Plugins CLI](https://docs.jfrog.com/artifactory/docs/agent-commands)
- [JFrog permissions](https://docs.jfrog.com/administration/docs/permissions)
- [Permission targets](https://docs.jfrog.com/artifactory/docs/permission-targets)
