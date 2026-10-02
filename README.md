# PostgreSQL HA with Docker Compose

HA uses `pg_auto_failover`: one monitor and three PostgreSQL nodes on one
Docker network.

```bash
docker compose up -d
ansible-playbook playbooks/site.yml
```

Check or stop the cluster:

```bash
docker compose ps
ansible-playbook playbooks/clean.yml --tags clean
```

Use `--tags nuke` to remove database volumes too.

## Customize with AI

To deploy a different number of PostgreSQL instances or customize the
infrastructure:

1. Open this repository in Cursor, Codex, or another agent-enabled IDE.
2. Tell the agent to read `AGENTS.md` and inspect the repository first.
3. Describe the deployment you want, including the number of instances and any
   environment-specific requirements.

The agent will ask for missing requirements, explain the required changes, and
guide you through validation and deployment.
