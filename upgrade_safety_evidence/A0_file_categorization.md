# A0: File categorization (v7.6.12 -> HEAD, 97 files changed)

Baseline: `v7.6.12` (real GA tag, ancestor of HEAD). `git diff --stat v7.6.12 HEAD`: 97 files
changed, 4062 insertions, 292 deletions.

## Bucket 1 - Central variable files (2) - audited in A1/A2
- roles/variables/defaults/main.yml
- roles/variables/vars/main.yml

## Bucket 2 - Component defaults/main.yml (7) - audited, JAVA_HOME-only, confirmed harmless
- roles/control_center/defaults/main.yml
- roles/kafka_broker/defaults/main.yml
- roles/kafka_connect/defaults/main.yml
- roles/kafka_controller/defaults/main.yml
- roles/kafka_rest/defaults/main.yml
- roles/ksql/defaults/main.yml
- roles/schema_registry/defaults/main.yml

## Bucket 3 - Task/playbook/template files (51) - covered by A3's grep sweep
playbooks/control_center.yml, playbooks/kafka_broker.yml, playbooks/kafka_connect.yml,
playbooks/kafka_connect_replicator.yml, playbooks/kafka_controller.yml, playbooks/kafka_rest.yml,
playbooks/ksql.yml, playbooks/rootless_bootstrap.yml, playbooks/tasks/certificate_authority.yml,
roles/common/tasks/collect_support_bundle.yml, roles/common/tasks/config_validations.yml,
roles/common/tasks/custom_java_install.yml, roles/common/tasks/debian.yml,
roles/common/tasks/fips-redhat.yml, roles/common/tasks/idp_certs.yml,
roles/common/tasks/main.yml, roles/common/tasks/rbac_setup.yml, roles/common/tasks/redhat.yml,
roles/common/tasks/rootless_lifecycle.yml, roles/common/tasks/rootless_prereqs.yml,
roles/common/tasks/rootless_service_state.yml, roles/common/tasks/rootless_start_service.yml,
roles/common/tasks/secrets_protection.yml, roles/common/tasks/ubuntu.yml,
roles/common/tasks/validate_package_availability.yml,
roles/common/templates/rootless.service.j2, roles/common/templates/rootless_component.env.j2,
roles/control_center/tasks/main.yml, roles/control_center/tasks/restart_and_wait.yml,
roles/kafka_broker/tasks/dynamic_groups.yml, roles/kafka_broker/tasks/main.yml,
roles/kafka_broker/tasks/rbac.yml, roles/kafka_broker/tasks/restart_and_wait.yml,
roles/kafka_connect/tasks/connect_plugins.yml, roles/kafka_connect/tasks/main.yml,
roles/kafka_connect/tasks/restart_and_wait.yml, roles/kafka_connect_replicator/tasks/main.yml,
roles/kafka_connect_replicator/tasks/rbac_replicator.yml,
roles/kafka_connect_replicator/tasks/rbac_replicator_consumer.yml,
roles/kafka_connect_replicator/tasks/rbac_replicator_monitoring.yml,
roles/kafka_connect_replicator/tasks/rbac_replicator_producer.yml,
roles/kafka_connect_replicator/tasks/restart_and_wait.yml, roles/kafka_controller/tasks/main.yml,
roles/kafka_controller/tasks/rbac.yml, roles/kafka_controller/tasks/restart_and_wait.yml,
roles/kafka_rest/tasks/main.yml, roles/kafka_rest/tasks/restart_and_wait.yml,
roles/kerberos/tasks/main.yml, roles/ksql/tasks/main.yml, roles/ksql/tasks/restart_and_wait.yml,
roles/schema_registry/tasks/main.yml, roles/schema_registry/tasks/restart_and_wait.yml

Note: `roles/kafka_broker/tasks/main.yml`'s "Set Permissions on /var/lib/kafka" and
`roles/kafka_controller/tasks/main.yml`'s "Set Permissions on /var/lib/controller" were
specifically investigated (hardcoded literals rather than variable-derived) - both are already
correctly guarded `when: not (rootless_enabled | bool)`, confirmed by direct read. No root or
rootless bug.

## Bucket 4 - Docs/molecule/test-only (36) - informational, not deploy-time risk
.gitignore, .semaphore/rootless_become_check.py, .semaphore/rootless_privileged_tag_check.py,
.semaphore/sanity_tests.sh, .semaphore/tests/test_rootless_become_check.py,
.semaphore/tests/test_rootless_privileged_tag_check.py, ROOTLESS_DEPLOYMENT_STEPS.md,
docs/VARIABLES.md, docs/sample_inventories/non_root_deployment.yml,
docs/sample_inventories/non_root_deployment_rbac_ldap.yml,
docs/sample_inventories/non_root_deployment_rbac_oauth.yml,
molecule/broker-scale-up/molecule.yml, molecule/certificates.yml,
molecule/connect-scale-up/molecule.yml, molecule/rbac-mds-kerberos-debian/molecule.yml,
molecule/rbac-mds-kerberos-mtls-custom-rhel/{molecule,verify}.yml,
molecule/rbac-mds-mtls-custom-kerberos-rhel/molecule.yml,
molecule/rbac-mds-mtls-custom-rhel-fips/{molecule,verify}.yml,
molecule/rbac-mds-mtls-existing-keystore-truststore-ubuntu/molecule.yml,
molecule/rbac-mds-plain-custom-rhel-fips/molecule.yml,
molecule/rbac-mds-scram-custom-rhel/molecule.yml,
molecule/rootless-{oauth-ubuntu2004,plain-debian10,rbac-ldap-rhel9}/*.yml (12 files),
test_roles/confluent.test.oauth/tasks/main.yml

## Coverage plan
- Bucket 1: A1 (Jinja evaluation) + A2 (doc cross-check)
- Bucket 2: already audited (this doc)
- Bucket 3: A3's grep sweep for old literals (systematic, catches sins of omission across all 51
  files without manual file-by-file reading) + spot-read of anything the grep flags
- Bucket 4: no action needed (test/doc-only, no deploy-time path risk)
