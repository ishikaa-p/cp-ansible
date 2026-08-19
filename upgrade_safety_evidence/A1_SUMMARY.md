# A1: Jinja-evaluation cross-check result

Method: rendered every one of the 47 path variables (plus `*_default_user`/`*_default_group`
pairs) via real Ansible/Jinja evaluation, with `deployment_path` left unset, at both `v7.6.12`
(pre-PR baseline) and the current HEAD (`f0e1c0402`) using isolated `git worktree`s so the
comparison exercises real variable resolution rather than a hand-written string-matching script.

- Inventory: `audit_inventory.yml` - single `localhost` host in every relevant group
  (zookeeper/kafka_broker/kafka_controller/schema_registry/kafka_connect/kafka_rest/ksql/
  control_center/kafka_connect_replicator) so all `_final_properties`-style nested lookups
  resolve without inventory-shape errors.
- Playbook: `audit_vars.yml` - `import_role: confluent.platform.variables` then `debug` every
  variable, written to a JSON file.
- Run against both worktrees with `ANSIBLE_COLLECTIONS_PATH` pointed at each worktree's own
  properly-namespaced root (`worktrees/{mergebase,head}_root/ansible_collections/confluent/platform`)
  so each run uses that commit's own code, never the live branch checkout.

## Result

`diff A1_variable_dump_v7.6.12.json A1_variable_dump_head.json` (see `A1_diff.txt`):

**Only 2 lines differ**, and both are variables that don't exist at all pre-PR (dumped as
`NEW_VAR_NOT_IN_BASELINE` there since referencing them at v7.6.12 would error otherwise):
- `deployment_group`: new in this PR, resolves to `"confluent"` at HEAD - matches the literal
  every `*_default_group` ternary already hardcoded pre-PR.
- `rootless_lifecycle_bin_dir`: new in this PR, resolves to `/opt/confluent/rootless-bin` at HEAD
  (derived from `archive_destination_path`, which itself is confirmed unchanged at `/opt/confluent`).

**Every other audited variable - all 47 path variables covered by the PR's ternary conversions,
plus every `*_default_user`/`*_default_group` pair - is byte-identical between v7.6.12 and HEAD**
when `deployment_path` is left unset, as any untouched brownfield inventory would leave it.

Conclusion: no pre-existing root-brownfield path, keytab, jmx/jolokia config path, CLI path, SSL
directory, log directory, data directory, or user/group default changed as a result of this PR.
