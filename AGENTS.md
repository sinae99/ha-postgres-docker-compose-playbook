# AGENTS.md

## Purpose

This repository is a small, local PostgreSQL high-availability test environment. Future AI agents must act as interactive infrastructure engineering assistants: inspect the repository first, gather missing requirements before editing, explain operational impact, make the smallest consistent change, validate it, and clearly distinguish validated configuration from deployment that was actually executed.

Do not assume this repository contains Kubernetes, Helm, Terraform, cloud provisioning, remote-host provisioning, backups, monitoring, CI, or production hardening. Those capabilities are not present in the current tree. If a user requests one of them, explain that it is outside the current implementation and ask for the required design decisions before adding anything.

## Project Overview

The current architecture is:

- One `pg_auto_failover` monitor container, `pg-monitor`.
- Three PostgreSQL containers managed by `pg_auto_failover`, `pg-node1`, `pg-node2`, and `pg-node3`.
- All four containers run through one Docker Compose project named `postgres-local-ha`.
- The inventory names four logical hosts, but all of them use `ansible_connection=local` and `127.0.0.1`; this is a single-machine test topology, not a multi-host failure domain.
- The monitor and each PostgreSQL node persist data in separate Docker named volumes.
- Containers communicate over the Compose-created default network using service names. Host port bindings are loopback-only:
  - Monitor PostgreSQL: `127.0.0.1:5500`.
  - Node 1 PostgreSQL: `127.0.0.1:5541`.
  - Node 2 PostgreSQL: `127.0.0.1:5542`.
  - Node 3 PostgreSQL: `127.0.0.1:5543`.
- PostgreSQL listens on port `5432` inside every container.
- The image is `citusdata/pg_auto_failover:latest`, controlled by `postgres_image` in `inventory/group_vars/all.yml`.
- The current setup explicitly disables SSL and uses `trust` authentication for local test clients. Treat this as unsafe for production or network-exposed use.

The Compose commands initialize `pg_auto_failover` state only when the relevant configuration file is absent, then run `pg_autoctl`. Existing named-volume state is therefore significant: restarting containers is not the same as rebuilding the cluster.

## Repository Structure

Important files and their responsibilities:

```text
README.md                                  Documented quick-start and cleanup commands
ansible.cfg                                Inventory, role path, and Ansible defaults
compose.yaml                               Checked-in Compose stack used for direct startup
inventory/hosts.ini                        Logical monitor/node groups; all map to localhost
inventory/group_vars/all.yml               Image, ports, service names, DSN, and DB variables
playbooks/site.yml                         Ordered monitor, node, DB initialization, and report flow
playbooks/clean.yml                        Cleanup tasks tagged `clean` and `nuke`
roles/pg_monitor/tasks/main.yml            Renders, validates, starts, and prepares the monitor
roles/pg_monitor/templates/docker-compose.yml.j2
                                            Jinja source for the Compose stack
roles/pg_node/tasks/main.yml               Starts and prepares one mapped PostgreSQL node
roles/report/tasks/main.yml                Prints Compose and final pg_auto_failover state
```

There are currently no Helm charts, `values.yaml` files, Terraform configurations, Kubernetes manifests, Dockerfiles, Ansible dependency files, test suites, CI definitions, or backup jobs in the repository. Inspect the tree again before relying on that statement after future changes.

## Configuration Model

The main configuration source is `inventory/group_vars/all.yml`:

- `postgres_image`: image used by the Jinja Compose template.
- `postgres_stack_dir`: defaults to `{{ playbook_dir }}/..`; with the checked-in playbooks this resolves to the repository root.
- `pg_monitor_service`: currently defined as `pg-monitor`; inspect usages before changing or relying on it because the current tasks use the literal service name in several commands.
- `pg_monitor_host_port`: host port for the monitor, currently `5500`.
- `pg_monitor_dsn`: node-to-monitor DSN, currently addressed to `pg-monitor:5432` with SSL disabled.
- `pg_node_services`: ordered service list `pg-node1`, `pg-node2`, `pg-node3`.
- `pg_node_host_ports`: ordered host port list `5541`, `5542`, `5543`.
- `app_database_name`: database initialized on the detected primary, currently `appdb_utf8`.

The ordering relationship between `inventory/hosts.ini`, `groups['pg_nodes']`, `pg_node_services`, and `pg_node_host_ports` is essential. `roles/pg_node/tasks/main.yml` calculates the node index from inventory group order, then indexes both lists. A node-count or ordering change must update all related structures and the explicit node deployment sequence in `playbooks/site.yml`; do not change only one list.

`roles/pg_monitor/templates/docker-compose.yml.j2` is the source that the monitor role writes to the repository-root `compose.yaml`. The checked-in `compose.yaml` is also the direct-start baseline, but running the monitor role can overwrite it. When changing Compose behavior for Ansible deployments, update the template and ensure the checked-in Compose file remains intentionally synchronized; do not make an unreviewed manual change that the next playbook run will erase.

## Deployment Flow

The README documents this sequence:

```bash
docker compose up -d
ansible-playbook playbooks/site.yml
```

The playbook itself performs the service starts as well, but its order and side effects must be understood before proposing a shortcut:

1. On `pg_monitor`, the `pg_monitor` role creates the stack directory if needed, renders `compose.yaml` from the Jinja template, validates it with `docker compose config --quiet`, starts `pg-monitor`, waits for host port `5500`, appends `host  all  all  0.0.0.0/0  trust` to the monitor's `pg_hba.conf` if absent, and reloads PostgreSQL configuration.
2. The playbook checks monitor state with `pg_autoctl show state --pgdata /var/lib/postgres/pgaf`.
3. It deploys `pg-node1`, verifies state, deploys `pg-node2`, verifies state, then deploys `pg-node3` and verifies state again. Each node role starts only the mapped Compose service, waits up to 180 seconds for its host port, adds the same broad test `pg_hba.conf` trust rule, and reloads configuration.
4. The playbook checks `pg_is_in_recovery()` on each service in `pg_node_services` to find the current primary. It fails instead of initializing the database if no primary is detected.
5. On the detected primary, it creates `app_database_name` if missing using UTF-8 encoding, `template0`, and `C` locale settings, then sets UTF-8 client encoding for that database and the `postgres` role.
6. The `report` role prints `docker compose ps` and the final `pg_autoctl show state` output.

The playbook uses `gather_facts: false`, no privilege escalation, and commands executed locally through Docker Compose. It does not provision Docker, install Ansible, configure a remote VM, establish TLS, create users/passwords, or configure backups.

## Interactive Agent Workflow

When a user asks to deploy, modify, or customize this infrastructure:

### Discover first

- Read the relevant README, inventory, group variables, playbook, role tasks, and Compose template before editing.
- Trace the requested setting from the user-facing variable through Ansible and into the rendered Compose file or SQL command.
- Identify whether the requested change affects named volumes, service names, host ports, inventory ordering, the monitor DSN, or primary detection.
- Explain the current architecture when it affects the request, especially that all logical hosts are local containers on one machine.

### Gather requirements before editing

Do not immediately modify files when important requirements are missing. Ask concise questions based on the existing configuration, and do not ask for facts already established by the repository. Depending on the request, clarify:

- Is this still a local development/test deployment, or is production/staging use intended?
- How many monitor and PostgreSQL nodes are required, and should they remain on one host or be distributed across real hosts?
- What host CPU, RAM, storage, and Docker volume constraints apply? The current repository does not define resource limits.
- Which host ports, bind addresses, networks, DNS names, and service names are required?
- Should the image remain `citusdata/pg_auto_failover:latest`, or should a specific version be pinned?
- What PostgreSQL authentication, TLS, and network exposure are required? The current setup uses disabled SSL and broad `trust` rules only for local testing.
- Which database name, encoding, locale, users, and application connection details are required?
- What persistence, backup, restore, replication, failover, and recovery objectives apply? The repository creates volumes and a backup directory inside nodes but implements no backup procedure.
- Is the requested operation allowed to stop containers, recreate services, change cluster membership, or remove data?

For changes involving the Compose or Ansible configuration, use the existing variables and ordered lists as the starting point. Do not invent a parallel configuration model.

### Plan and obtain confirmation

After requirements are known, summarize:

- The requested topology and configuration.
- The exact files that need modification.
- The intended change and its operational impact.
- Whether a container restart, downtime, failover, reinitialization, port conflict, or data migration is possible.
- Any security implications, especially around `trust`, disabled SSL, `latest` image tags, and exposed bind addresses.

Ask for explicit confirmation before destructive or high-impact operations. This includes changing cluster membership, recreating stateful services, deleting volumes, changing authentication for an existing cluster, or applying changes to a production-like environment.

## Implementation Rules

- Prefer a focused change to `inventory/group_vars/all.yml`, `inventory/hosts.ini`, the existing playbook, role, or Compose template over introducing a new tool or file.
- Preserve the current monitor/node/service/volume relationships unless the user explicitly requests a topology change.
- Keep environment-specific values parameterized through existing Ansible variables where practical; do not hardcode a user's ports, addresses, or image tag into multiple files.
- When changing node count or order, update inventory groups, `pg_node_services`, `pg_node_host_ports`, the Compose services and volumes, and the sequential sections of `playbooks/site.yml` together. The current implementation is explicitly written for three nodes and is not a generic loop.
- When changing the database name, update or verify `app_database_name` and remember that the task display name in `playbooks/site.yml` currently says `appdb_utf8` even though the SQL uses the variable.
- When changing the image, inspect both the template and the checked-in `compose.yaml`; the template is rendered over the latter by the monitor role.
- Do not silently alter unrelated services, authentication, ports, persistence, or cleanup behavior.
- Do not add secrets to Git, command output, Compose files, or Ansible debug output. Ask users to provide secret references or secure injection requirements rather than requesting secret values unnecessarily.
- Do not add claims of multi-host HA, automated backups, disaster recovery, or production readiness unless those capabilities are implemented and validated.

## Validation

Before declaring a configuration change complete, run the narrowest applicable checks and report their results:

```bash
docker compose config --quiet
ansible-playbook playbooks/site.yml --syntax-check
ansible-playbook playbooks/clean.yml --syntax-check
```

Run commands from the repository root so `ansible.cfg` is discovered. In restricted environments where Ansible cannot write its default home temp directory, set writable temporary locations, for example:

```bash
mkdir -p /tmp/ansible-local /tmp/ansible-remote
HOME=/tmp ANSIBLE_LOCAL_TEMP=/tmp/ansible-local ANSIBLE_REMOTE_TEMP=/tmp/ansible-remote \
  ansible-playbook playbooks/site.yml --syntax-check
```

For an actual deployment or operational verification, use the repository's commands and inspect the result rather than inferring success:

```bash
docker compose ps
docker compose exec -T -u postgres pg-monitor \
  pg_autoctl show state --pgdata /var/lib/postgres/pgaf
ansible-playbook playbooks/site.yml
```

Only claim deployment success after the command has actually run and the container status and final `pg_autoctl` report have been reviewed. If Docker is unavailable, the image cannot be pulled, or the user did not authorize execution, report the configuration as unverified and list the manual commands needed.

Check consistency after edits:

- Every service referenced by the playbooks exists in the rendered Compose file.
- Every node inventory mapping resolves to a matching service and port list entry.
- The monitor DSN uses the actual Compose monitor service and internal port.
- Host ports are available and bind to the intended address.
- Named volumes remain attached to the intended stateful service.
- The SQL database name and Ansible variable agree.
- The Compose template and checked-in `compose.yaml` are intentionally aligned.

There is no repository test suite, linter configuration, or CI workflow currently present. Do not claim those checks were run unless future repository inspection finds and executes them.

## Operations and Cleanup

Inspect the running stack without changing data:

```bash
docker compose ps
docker compose config --quiet
docker compose logs --tail=100 pg-monitor pg-node1 pg-node2 pg-node3
docker compose exec -T -u postgres pg-monitor \
  pg_autoctl show state --pgdata /var/lib/postgres/pgaf
```

The supported Ansible cleanup commands are:

```bash
# Stop and remove containers, preserving named database volumes.
ansible-playbook playbooks/clean.yml --tags clean

# Stop and remove containers and remove all named database volumes.
# This permanently removes the monitor and node data.
ansible-playbook playbooks/clean.yml --tags nuke
```

`playbooks/clean.yml` runs on the `pg_monitor` logical host but executes Docker Compose against `postgres_stack_dir`. The `clean` task uses `docker compose down --remove-orphans`; the `nuke` task uses `docker compose down --volumes --remove-orphans`.

Never run the `nuke` tag, `docker compose down -v`, `docker volume rm`, `kubectl delete pvc`, `terraform destroy`, or an equivalent destructive command without explicit user authorization and a clear statement of the data-loss consequences. This repository does not provide a backup or restore workflow, so volume deletion must be treated as irreversible unless the user has an independently verified backup.

## Safety Boundaries

- Treat `monitor_pgdata`, `node1_pgdata`, `node2_pgdata`, and `node3_pgdata` as sensitive state.
- Preserve persistent volumes during ordinary edits and restarts.
- Treat the current `trust` authentication and `PGSSLMODE=disable` settings as test-only. Do not expose the loopback bindings or broaden them for real clients without an explicit security review and requirements gathering.
- Be especially cautious with `latest`: changing or pulling it can change database behavior without a repository diff. Ask whether the user wants an immutable image version.
- Do not overwrite a production or production-like Compose file without confirming the target environment and taking a backup or rollback plan into account.
- State clearly whether you only rendered or validated configuration, or actually started/stopped containers.
- If the requested behavior cannot be determined from the repository, say what must be inspected or ask the user instead of guessing.
