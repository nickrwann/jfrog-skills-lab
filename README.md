# JFrog Skills Lab

A small learning repository for exploring JFrog Artifactory Skills
repositories, including publishing and installing agent skills locally.

## Quick start

This lab uses the official Artifactory Pro image with its bundled database.
It is intended for evaluation only.

1. Install Docker Engine and the Docker Compose plugin inside WSL. This lab
   uses the native Linux daemon managed by systemd; Docker Desktop is not
   required.
2. Start Artifactory:

   ```bash
   docker compose up -d
   docker compose logs -f artifactory
   ```

3. Open <http://localhost:8082/> and complete Artifactory onboarding.
4. Create a local repository with package type **Skills** and key
   `skills-local`.

Stop the lab without deleting its data:

```bash
docker compose down
```

The named Docker volume is intentionally retained. Running
`docker compose down --volumes` also removes the Artifactory configuration and
stored skills.

## Goals

- Run an evaluation instance of JFrog Artifactory locally.
- Create a Skills repository.
- Publish a sample skill.
- Install the skill into Codex and other compatible coding agents.

## References

- [JFrog Skills repositories documentation](https://docs.jfrog.com/artifactory/docs/skills-repositories)
- [JFrog Skills CLI reference](https://docs.jfrog.com/artifactory/docs/jf-skills)
- [JFrog agent commands](https://docs.jfrog.com/artifactory/docs/agent-commands)
- [Artifactory Docker installation](https://docs.jfrog.com/installation/docs/docker)
- [Producer guide](docs/producer-guide.md)
- [Consumer guide](docs/consumer-guide.md)
