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
