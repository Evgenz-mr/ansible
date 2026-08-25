# Ansible Tomcat Provisioning Lab

A focused configuration-management lab for provisioning and operating a Tomcat host with Ansible.

## What it demonstrates

- inventory-driven automation
- package and service management
- application-server configuration
- templated/configuration file delivery
- repeatable provisioning with privilege escalation
- CI syntax validation for Ansible content

## Repository map

- `tomcat_provision.yml` — original provisioning playbook
- `inventory` — lab inventory example
- `tomcat.service` — systemd unit example
- `context.xml` / `tomcat-users.xml` — application-server configuration examples
- `PORTFOLIO.md` — productionization notes and design improvements

## Validation

```bash
ansible-playbook -i inventory tomcat_provision.yml --syntax-check
ansible-playbook -i inventory tomcat_provision.yml --check
```

## Scope

This repository is intentionally kept as a focused historical lab. New reusable infrastructure automation belongs in the separate `ansible_roles` portfolio repository. Before using this playbook against current Linux/Tomcat versions, review package names, Java compatibility, file ownership, secret handling and service hardening.
