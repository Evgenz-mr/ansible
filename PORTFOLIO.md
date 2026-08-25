# Productionization notes

The original playbook is useful for showing Ansible fundamentals, but a production implementation should evolve toward the following design:

```text
inventories/
  dev/
  prod/
roles/
  java/
  tomcat/
playbooks/
  site.yml
```

## Improvements expected in production

1. Move configurable values into role defaults/group vars.
2. Keep credentials out of `tomcat-users.xml`; source them from Ansible Vault or an external secret store.
3. Use handlers so Tomcat restarts only when configuration changes.
4. Add idempotency testing and disposable integration environments.
5. Pin supported Java/Tomcat versions and document upgrade testing.
6. Add least-privilege ownership and systemd hardening directives.
7. Separate installation, configuration, deployment and verification responsibilities.

The `ansible_roles` repository is the primary portfolio location for those reusable-role patterns.
