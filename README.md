# ansible-resources

A small practice repo for learning Ansible fundamentals against Cisco IOS
devices — playbook structure, inventory management, group/host variables,
and the `cisco.ios` collection modules. Built as a hands-on exercise for
getting comfortable with YAML syntax and Ansible's network automation
patterns, not as a production deployment tool.

## What's in here

```
.
├── inventory.ini                  # Static inventory: switches + ASBR routers
├── playbook.yaml                  # Two-play example: health check + config push
├── group_vars/
│   └── asbr_routers.yml           # Shared vars for the asbr_routers group
└── host_vars/
    └── router00.yml               # Per-host vars (IPs, OSPF/HSRP settings)
```

## Concepts practiced

- **Inventory structure** — grouping hosts (`access_switches`, `core_switches`,
  `asbr_routers`) and setting connection-wide vars under `[all:vars]`
  (`ansible_connection`, `ansible_network_os`, credentials, privilege escalation).
- **Multi-play playbooks** — separating a read-only "pre-deployment health
  check" play from a second play that pushes configuration, so checks run
  before any changes are attempted.
- **`cisco.ios` collection modules** — `ios_ping`, `ios_command` for
  show/read-only operations, and `ios_config` for pushing CLI configuration,
  including nested config with `parents` (e.g. entering an interface context
  before applying sub-commands).
- **Variable layering** — `group_vars/` for settings shared across a group,
  `host_vars/` for per-device values (interface IPs, HSRP priorities, OSPF
  cost/priority) that get interpolated into the playbook via Jinja2
  (`{{ g000_ip }}`, `{{ ospf_password }}`, etc.).
- **Registering and using command output** — capturing `show` command results
  with `register`, then referencing them in later tasks (e.g. backing up
  `show running-config` output before making changes).

## Known issues / things left as an exercise

This was written and iterated on as a learning exercise, so a few rough
edges are intentionally still here rather than fixed:

- `router01` was never added to `inventory.ini` with a real `ansible_host` —
  only `router00` is fully wired up. Adding `router01` back in (with its own
  `host_vars/router01.yml`) is a good next step.
- The `Save backup to local file` task references `ansible_date_time`, which
  isn't populated because `gather_facts: false` is set on both plays. On IOS
  devices, fact-gathering doesn't work the same way it does for Linux hosts,
  so the more reliable fix is generating a timestamp on the controller
  instead, e.g.:
  ```yaml
  dest: "./backups/{{ inventory_hostname }}-{{ lookup('pipe', 'date +%Y%m%d-%H%M%S') }}.cfg"
  ```
- Credentials in `inventory.ini` (`ansible_password`, `ansible_become_password`)
  are plaintext lab credentials for a local sandbox. In any environment
  beyond a personal lab, these should move to `ansible-vault` or be passed
  via `--ask-pass`/`--ask-become-pass` instead of being committed.
- `ospf_password` in `host_vars/router00.yml` is similarly a placeholder and
  should be vaulted rather than stored in plaintext for anything beyond
  local practice.

## Requirements

```bash
pip install ansible paramiko
ansible-galaxy collection install cisco.ios
```

## Usage (against a lab device)

```bash
ansible-playbook -i inventory.ini playbook.yaml
```

This repo has not been run against production or shared infrastructure —
it's scoped to a personal lab environment for practicing Ansible/YAML
syntax and Cisco IOS module usage.
