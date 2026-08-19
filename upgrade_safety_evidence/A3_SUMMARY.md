# A3: Repo-wide grep sweep for missed literal path conversions

Repo-wide `grep` for every old literal path value this PR's ternary conversions replaced
(`/var/lib/kafka`, `/var/lib/controller`, `/var/lib/zookeeper`, `/var/lib/kafka-streams`,
`/var/lib/confluent/control-center`, `/var/log/kafka`, `/var/log/confluent`, `/opt/confluent`,
`/opt/confluent-cli`, `/opt/jolokia`, `/opt/prometheus`, `/etc/security/keytabs`,
`/var/ssl/private`, `/usr/local/bin/confluent`, `/etc/krb5.conf`, `/usr/share/java/connect_plugins`)
across `roles/` and `playbooks/` (excluding the already-audited central variable files and
molecule fixtures), plus a separate sweep of all 60 `roles/**/templates/*.j2` files - the
highest-risk spot for a missed literal, since templates are what actually render component
config files. Raw hits: `A3_grep_sweep_raw_hits.txt`.

## Result: 23 hits outside the central files, 0 in templates, all classified safe

**10 hits - `/etc/krb5.conf` comparison guards** (one per component's `defaults/main.yml`
java-args list plus two `health_check.yml` files):
```
{% if kerberos_client_config_file_dest != '/etc/krb5.conf' %}-Djava.security.krb5.conf=...{% endif %}
```
This is intentional and correct: it's comparing the variable's *current resolved value* against
the literal to decide whether to add the `-D` JVM flag. For root brownfield,
`kerberos_client_config_file_dest` resolves to exactly `/etc/krb5.conf` (confirmed in A1), so this
evaluates false and no flag is added - identical to pre-PR behavior. Not a literal-path
assignment at all.

**5 hits - hardcoded `/var/ssl/private*` directory-creation tasks** (`kafka_rest/tasks/main.yml`,
`kafka_connect_replicator/tasks/rbac_replicator_producer.yml`,
`kafka_connect_replicator/tasks/rbac_replicator_monitoring.yml`,
`kafka_controller/tasks/rbac.yml`, `common/tasks/idp_certs.yml`, `kafka_broker/tasks/rbac.yml`).
Checked each directly: **every one is already guarded `when: not (rootless_enabled | bool)`**
(some combined with other conditions like `rbac_enabled|bool`). Root: literal, unchanged, runs
exactly as before. Rootless: correctly skipped (the actual rootless-safe SSL directory creation
happens elsewhere via `ssl_file_dir_final`, per the earlier `a34c4f7be` fix). No root-brownfield
impact either way.

**4 hits - `/var/lib/controller/` and `/var/lib/kafka/` permission tasks**
(`kafka_controller/tasks/main.yml`, `kafka_broker/tasks/main.yml`) - already investigated in A0;
both guarded `when: not (rootless_enabled | bool)`. No bug.

**Templates**: 0 hits across all 60 `.j2` files - no config-rendering template contains a stale
hardcoded literal.

## Verdict

Zero "sins of omission" found - no location exists where this PR's diff should have converted a
literal path to a `deployment_path`-aware ternary but didn't. Every hit is either an intentional
literal comparison, or an already-correctly-guarded root-only/rootless-skipped task using a
literal that was never touched by this PR at all.
