# Phase B: live EC2 root brownfield upgrade test

## Setup

3x RHEL9 EC2 instances (t3.xlarge), self-mapped `/etc/hosts` to avoid AWS hairpin-NAT:
- `upgradenode1`: `zookeeper` + `control_center`
- `upgradenode2`: `kafka_broker` (broker 1)
- `upgradenode3`: `kafka_broker` (broker 2) + `schema_registry` + `kafka_connect` + `kafka_rest` + `ksql`

Inventory (`upgrade_test_inventory.yml`): deliberately vanilla - `installation_method: package`,
plaintext (no SSL/RBAC/Kerberos), and **none** of this PR's new variables
(`deployment_path`/`deployment_user`/`deployment_group`/`rootless_enabled`) mentioned at all -
exactly what an untouched, pre-existing customer inventory looks like.

Two isolated `git worktree`s used throughout so the "old" and "new" code never touch the live
branch checkout: `v7.6.12` (pre-PR baseline) and the current HEAD.

## Step 1: deploy with the pre-PR (`v7.6.12`) code

`ansible-playbook playbooks/all.yml` from the `v7.6.12` worktree: **`failed=0`** across all 3
hosts (`ansible_deploy_log.txt`). Created a real topic (`upgrade-safety-test`, 3 partitions, RF 2)
and produced 10,000 real messages. Snapshotted ground truth via direct SSH (not trusting
Ansible's own summary) - see `B_pre_upgrade/`:
- Inode + mtime listing of `/var/lib/kafka/data` on both brokers and `/var/lib/zookeeper`.
- Rendered `log.dirs` (both brokers), `dataDir` (ZK), `confluent.controlcenter.data.dir` (C3),
  `ksql.streams.state.dir` (ksqlDB) - all confirmed as the expected literal paths
  (`/var/lib/kafka/data`, `/var/lib/zookeeper`, `/var/lib/confluent/control-center`,
  `/var/lib/kafka-streams`).
- Systemd unit `FragmentPath`/`ExecStart` for every component.
- Message manifest: consumed all 10,000 messages, sorted, checksummed
  (`message_manifest.sha256`: `e2b03bd3...`).

## Step 2: the "upgrade" - switch only the code, nothing else

Re-ran the **identical** `ansible-playbook playbooks/all.yml -i <same file>`, this time from the
HEAD worktree. Target hosts and inventory file never changed.

- First attempt hit a transient EC2 network blip mid-run (all 3 hosts briefly unreachable
  simultaneously - a session-wide connectivity issue, not a code problem; `upgradenode1`/
  `upgradenode2` had already completed with `changed=0` by that point). Re-ran cleanly.
- Clean re-run: **`failed=0`** (`ansible_upgrade_run2_clean.log`). `changed=0` on
  `upgradenode1`/`upgradenode2`. `upgradenode3` showed `changed=4` on
  `kafka_broker`/`kafka_connect`'s "Create \*Config directory"/"Create Logs Directory" tasks -
  investigated and found **byte-identical task definitions in both `v7.6.12` and HEAD** setting
  different ownership (`cp-kafka` vs `cp-kafka-connect`) on the same default log directory
  (`/var/lib/kafka` co-located components share `kafka_broker_default_log_dir` ==
  `kafka_connect_default_log_dir` == `/var/log/kafka` by design) - a pre-existing, PR-unrelated
  co-location ownership-contention artifact of *this test's own topology choice*, not a path
  change and not a regression. The actual data-directory permission task
  ("Set Permissions on Data Dirs" for `/var/lib/kafka/data`) showed `ok:` throughout - stable.

## Step 3: post-upgrade verification

All checks in `B_pre_upgrade/` re-captured in `B_post_upgrade/` and diffed:

- **`upgrade-safety-test` topic's partition files on both brokers: byte-for-byte identical
  inode numbers and mtimes**, pre- vs. post-upgrade - the strongest available proof; Kafka's own
  process never even re-opened these files across the upgrade, let alone wrote them to a
  different location.
- ZooKeeper's entire `/var/lib/zookeeper` directory listing: identical.
- Every rendered config value (`log.dirs` ×2, `dataDir`, `confluent.controlcenter.data.dir`,
  `ksql.streams.state.dir`): identical.
- Original 10,000-message manifest re-consumed after the upgrade: **checksum matches exactly**
  (`e2b03bd3...` = `e2b03bd3...`) - the data itself, not just the path, survived intact and
  readable through the upgraded code.
- Produced 500 new messages post-upgrade; all 500 present on re-consume - the cluster is fully
  functional going forward, not just frozen-but-readable.
- All systemd units (`confluent-server` ×2, `confluent-zookeeper`, `confluent-control-center`,
  `confluent-schema-registry`, `confluent-kafka-connect`, `confluent-kafka-rest`,
  `confluent-ksqldb`) `active`, `NRestarts=0` on every unit since the upgrade - no crash/restart
  loop.

## Verdict

Every Phase C checklist item for Phase B passes cleanly. No data or config directory path moved,
no data was lost or became unreadable, and the cluster remained fully functional across a real
pre-PR → post-PR upgrade with an untouched, vanilla root inventory.
