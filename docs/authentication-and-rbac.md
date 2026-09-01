# Authentication, RBAC, and skill maintainership

This guide explains who can maintain a skill, how Artifactory knows who they
are, and how it decides which actions they may perform. It focuses on the human
and organizational model first, then maps that model to JFrog concepts.

## The short answer

JFrog does not infer maintainership from the `author` field in `SKILL.md`.
`author` is descriptive package metadata, similar to a name printed on a book.
It gives credit and context, but it is not proof of identity and does not grant
access.

A skill maintainer is a user, group, or project member that your organization
has deliberately given the appropriate permissions for the skill's repository
or artifact path. In practical terms, a maintainer usually needs permission to:

- Read existing releases.
- Publish new versions.
- Correct package metadata when appropriate.
- Remove or overwrite a bad release in exceptional circumstances.

Permission to manage **people's access** is a separate and more powerful
responsibility. A skill maintainer does not automatically need it.

## Four concepts that sound similar

### Identity

An identity is the person or automated system taking an action. Examples are
`nickrwann`, `alice`, or `skills-release-bot`.

Identity is the answer to: **who is acting?**

For a human, the identity may originate from a local JFrog account or an
external identity provider such as Okta, Microsoft Entra ID, or LDAP. For
automation, it is normally a deliberately named non-human subject or service
identity.

### Authentication

Authentication is the process of proving that the actor controls an identity.
A password, browser login, or access token is a credential used to prove that
claim.

Authentication is the answer to: **can you prove you are that identity?**

For CLI and API activity, the credential is usually a JFrog Access Token. The
token is presented with a request, and JFrog Access validates it. A valid token
gets the actor through the identity check; it does not automatically make the
actor an administrator.

### Authorization and RBAC

Authorization is the decision about what an authenticated identity may do.
Role-Based Access Control, or RBAC, grants abilities through roles or groups
instead of configuring every person independently.

Authorization is the answer to: **now that we know who you are, what are you
allowed to do here?**

For example, Alice and Bob can both authenticate successfully. Alice may be in
the `skill-maintainers` group and can publish or delete a release. Bob may be in
the `skill-consumers` group and can only search and install. Authentication
succeeded for both people; authorization produced different results.

### Authorship

Authorship is a claim in the skill's metadata about who created or is credited
for its content.

Authorship is the answer to: **who does this package say created it?**

Because authorship is editable text, it should not be used as an access rule.
The authenticated account in Artifactory's access records is operationally
more meaningful than the manifest's author string.

## How a request is evaluated

The request path can be pictured as:

```text
Human or CI job
      │
      │ presents password, browser session, or access token
      ▼
JFrog Access: authentication
      │
      │ resolves the identity and its groups/project roles
      ▼
Artifactory: authorization
      │
      │ checks action + repository + artifact path
      ▼
Accepted or denied operation
      │
      └── recorded with identity and context in platform logs
```

The authorization question is not simply “is Alice a maintainer?” It is more
specific:

```text
May Alice DEPLOY this path in skills-local?
May Alice DELETE this version in skills-local?
May Bob DOWNLOAD this path from skills-local?
```

That specificity is what makes it possible to let someone maintain one skill
without granting control over every package or the entire Artifactory server.

## Credentials and access tokens

An access token is a bearer credential. Anyone holding it can act with the
authority represented by that token until it expires or is revoked. “Bearer”
means possession is enough; there is normally no second prompt when a CLI uses
it.

Treat a token like a password:

- Do not place it in `SKILL.md`, source files, Git history, or screenshots.
- Prefer an expiration appropriate to the task.
- Give it only the groups, project roles, and audience it needs.
- Revoke it when it is no longer required or might have leaked.
- Use a secret manager for CI rather than a developer's personal token.

JFrog tokens have a subject, scope, audience, issuer, issue time, and optional
expiry. In plain language:

| Token property | Meaning |
| --- | --- |
| Subject | Identity represented by the token |
| Scope | Groups, roles, or permissions represented by it |
| Audience | JFrog services or instances that should accept it |
| Issuer | JFrog instance that created it |
| Expiry | Time after which it can no longer be used |

A CLI profile connects a friendly local name to the Artifactory URL and
credential:

```bash
jf config add local-skills \
  --url http://localhost:8082 \
  --access-token "$JFROG_ACCESS_TOKEN" \
  --interactive=false

jf config use local-skills
```

This avoids repeatedly typing the token, but it does not make the token safe to
share. The local JFrog configuration is still sensitive user data.

## Repository permissions in everyday language

Skills packages use ordinary Artifactory repository permissions. These are the
capabilities most relevant to a producer/consumer workflow:

| Permission | Everyday meaning | Skills example |
| --- | --- | --- |
| Read | See and retrieve existing material | Search, list, download, or install a skill |
| Deploy/Cache | Add new material | Publish a new skill version |
| Annotate | Change attached properties | Correct or enrich artifact metadata |
| Delete/Overwrite | Remove or replace material | Recall a release or replace the same version path |
| Manage | Delegate permissions to others | Let a trusted lead change most access assignments in a permission target |

`Manage` is about managing authorization, not about maintaining the skill's
content. Under Permissions V2, a user with Manage permission cannot grant or
revoke Manage itself. This prevents a delegated permission manager from simply
creating another permission manager.

## Two ways to model RBAC

JFrog exposes two related approaches. Which one fits depends on how much of the
platform you are organizing.

### Permission targets: repository-centered control

A permission target selects repositories and optional path patterns, then
assigns actions to users or groups. It is direct and easy to visualize for a
small registry.

For this lab, a permission-target model might be:

```text
skills-consumers
  repository: skills-local
  paths:      all skill paths
  group:      skill-consumers
  actions:    Read

skills-publishers
  repository: skills-local
  paths:      all skill paths
  group:      skill-publishers
  actions:    Read, Deploy/Cache

skills-maintainers
  repository: skills-local
  paths:      all skill paths
  group:      skill-maintainers
  actions:    Read, Deploy/Cache, Annotate, Delete/Overwrite
```

To let one team maintain only `my-skill`, scope its target to the stored path:

```text
my-skill/**
```

The exact stored layout is `<slug>/<version>/<slug>-<version>.zip`, so the slug
is the natural top-level path boundary. Always test path rules with a non-admin
account; an admin account can hide mistakes because it already has broad
access.

### Project roles: project-centered control

JFrog Projects organize repositories, builds, environments, and members under
a project boundary. A Project Admin defines roles and assigns users or groups
as project members. Actions granted by a project role apply to the relevant
project resources and environments.

Use project roles when the Skills registry is one part of a larger team-owned
area and you want the same membership model across multiple repositories. A
conceptual project could contain:

```text
Project: agent-platform

Role: Skill Consumer
  members: application developers
  actions: read project artifacts

Role: Skill Publisher
  members: skill authors and release automation
  actions: read and deploy project artifacts

Role: Skill Maintainer
  members: designated release maintainers
  actions: read, deploy, annotate, and delete project artifacts

Project Admin
  members: small platform-owner group
  responsibility: project roles, members, repositories, and resources
```

Project roles are broader than a single skill path. If the requirement is “this
team may maintain only this one slug,” a repository permission target with a
path pattern may express that boundary more directly. If the requirement is
“this team owns all assets in this project,” project roles are the more natural
model.

## Who should be allowed to maintain a skill?

Maintainership should describe an operational duty, not popularity or original
authorship. A maintainer should be expected to:

- Review changes before release.
- Apply the versioning policy consistently.
- Publish or approve new versions.
- Respond to unsafe or broken releases.
- Communicate deprecations and major changes.
- Protect release credentials.
- Verify packages from a consumer's point of view.

Someone can remain credited as the original author without retaining current
maintainer access. Likewise, a release engineer may maintain the package
without claiming authorship of its content.

## Recommended roles for a small team

The following separation is understandable without being overly elaborate:

| Persona | Identity type | Permissions | Purpose |
| --- | --- | --- | --- |
| Platform administrator | Human admin | Platform-wide administration | Bootstrap Artifactory and recover the platform |
| Project/registry owner | Small human group | Project administration or delegated Manage | Define membership and repository policy |
| Skill maintainer | Human group | Read, Deploy, Annotate, Delete/Overwrite | Own releases and emergency corrections |
| Skill publisher | Human group | Read, Deploy | Publish new versions without rewriting history |
| CI publisher | Non-human identity | Read, Deploy | Publish approved automated releases |
| Consumer | Human or workload group | Read | Search and install released skills |

The Platform Admin account should not be the everyday publishing identity. It
has far more authority than publishing requires, which makes both mistakes and
credential leaks more consequential.

## Workflow example: solo producer

Nick is the only person experimenting with the registry.

1. Nick uses the administrator account once to complete onboarding and create
   `skills-local`.
2. Nick creates or uses a regular producer identity for daily CLI work.
3. That identity receives Read and Deploy permissions for `skills-local`.
4. Nick edits and reviews the skill in Git.
5. Nick creates a personal access token and configures `jf` locally.
6. Nick publishes a new semantic version.
7. The access log records the authenticated identity performing the deploy.
8. Nick uses the admin identity only when changing repository configuration or
   access policy.

Even for one person, separating daily publishing from administration makes the
model clearer and reduces the impact of accidentally exposing the CLI token.

## Workflow example: team with human maintainers

A team has Alice and Diego as maintainers, while several developers only
consume skills.

1. The registry owner creates `skill-maintainers` and `skill-consumers`
   groups.
2. Alice and Diego join `skill-maintainers`.
3. Application developers join `skill-consumers`.
4. The maintainer permission grants Read, Deploy, Annotate, and
   Delete/Overwrite on the approved skill paths.
5. The consumer permission grants only Read.
6. Alice publishes `my-skill` version `1.3.0` using her own token.
7. Diego can respond if the release must be removed, but consumers cannot
   alter the registry.
8. Logs identify Alice's authenticated deploy separately from Diego's later
   actions.

The team can change who maintains the skill by changing group membership. It
does not need to edit the skill package or rewrite its author metadata.

## Workflow example: CI is the only publisher

A team wants releases to come only from reviewed commits.

1. Human maintainers review and approve source changes in Git.
2. A named identity such as `skills-release-bot` receives Read and Deploy, but
   not Delete/Overwrite.
3. The pipeline receives a short-lived or appropriately expiring token from a
   protected CI secret store.
4. The pipeline verifies the expected version and publishes the package.
5. Human maintainers retain Delete/Overwrite for emergency response.
6. The access log attributes normal deployments to `skills-release-bot`, while
   Git records which humans reviewed the source.

This separates approval from mechanical publication. It also prevents the CI
credential from rewriting or deleting release history if it is misused.

## Workflow example: one team maintains one skill

The Data team should maintain `data-analysis`, but not `deployment-helper`.

1. Both skills live in `skills-local`.
2. The Data maintainers receive permissions scoped to `data-analysis/**`.
3. The Platform maintainers receive permissions scoped to
   `deployment-helper/**`.
4. Both groups may receive Read across the repository if cross-team discovery
   is desired.
5. A deploy outside a group's allowed path is denied even though the user is a
   valid, authenticated producer elsewhere in the repository.

This is the clearest example of authentication and authorization being
different: the Data producer's identity is valid, but its authority ends at a
path boundary.

## Offboarding a maintainer

When a maintainer changes teams or leaves:

1. Remove the user from the maintainer group or project role.
2. Revoke outstanding personal tokens associated with that access where
   appropriate.
3. Review recent security/access logs if the departure or timing warrants it.
4. Transfer source-review ownership in Git separately.
5. Do not rewrite historical `author` metadata merely because maintainership
   changed; authorship history and current authority are different facts.

Removing a role changes future authorization. Revoking tokens ensures an
already-issued credential cannot continue to be presented.

## Responding to a leaked publishing token

1. Revoke the affected token immediately.
2. Temporarily remove or reduce the identity's publishing role if needed.
3. Inspect the Artifactory access log for accepted or denied Deploy, Annotate,
   and Delete operations by that identity.
4. Compare published versions with reviewed Git releases.
5. Remove unauthorized packages using a trusted maintainer identity.
6. Issue a new, narrower or shorter-lived token only after the cause is
   understood.

Changing the `author` field does nothing to contain a leaked token because the
token—not the manifest text—is the credential used for authentication.

## Responding to a bad release

1. A maintainer confirms the exact slug and version.
2. The maintainer previews deletion with `jf agent skills delete --dry-run`.
3. The maintainer deletes only the affected version if removal is necessary.
4. The producer publishes a corrected new semantic version.
5. Consumers are told which version to update to.

Deleting a registry version does not remotely erase copies already installed
on consumer machines. That is why communication and a corrected release remain
part of the response.

## What Artifactory records

Artifactory's access log records security-related operations such as accepted
or denied logins, downloads, deployments, searches, annotations, and deletions.
Entries include context such as username, source IP, repository path, and
authentication type where applicable.

The separate security audit trail records changes to users, groups, permission
targets, and access tokens. Together, these logs answer two different audit
questions:

- **Access log:** who attempted or performed an artifact operation?
- **Security audit log:** who changed the identities, tokens, or rules that
  made those operations possible?

Logs provide accountability for the registry operation. Git history and review
records provide accountability for the source content. A mature producer
workflow uses both.

## A sensible policy for this lab

Start with the following model after Artifactory onboarding:

1. Keep the built-in administrator only for setup and recovery.
2. Create a normal producer identity for interactive experimentation.
3. Give it Read and Deploy on `skills-local`.
4. Create a separate `skill-maintainers` group with Read, Deploy, Annotate, and
   Delete/Overwrite.
5. Give consumers Read only.
6. Do not grant Manage until there is a real need to delegate permission
   administration.
7. Later, introduce `skills-release-bot` when publication moves into CI.

This makes the boundaries visible without requiring enterprise-scale identity
infrastructure on day one.

## References

- [JFrog authentication and token management](https://docs.jfrog.com/administration/docs/jfrog-authentication-and-token-management-overview)
- [Access tokens](https://docs.jfrog.com/administration/docs/access-tokens)
- [Permissions](https://docs.jfrog.com/administration/docs/permissions)
- [Permission targets](https://docs.jfrog.com/artifactory/docs/permission-targets)
- [Project roles and members](https://docs.jfrog.com/projects/docs/project-roles-and-members-concepts)
- [Manage project roles](https://docs.jfrog.com/projects/docs/manage-project-roles)
- [Access log](https://docs.jfrog.com/administration/docs/access-log)
- [Security audit trail](https://docs.jfrog.com/administration/docs/audit-trail-log)
- [Skills repositories](https://docs.jfrog.com/artifactory/docs/skills-repositories)
