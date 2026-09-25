# Nagios NRPE and Fail2Ban with Ansible

> [!WARNING]
> Archived and no longer maintained; kept for reference. Written in 2017 for CentOS 6 and 7, which are both end-of-life.

- [Nagios NRPE and Fail2Ban with Ansible](#nagios-nrpe-and-fail2ban-with-ansible)
  - [Overview](#overview)
    - [What this playbook does](#what-this-playbook-does)
  - [Using it](#using-it)
    - [Inventory](#inventory)
    - [Variable files](#variable-files)
    - [Running the playbook](#running-the-playbook)

## Overview

An Ansible playbook that sets up CentOS 6 and 7 servers to be monitored by a Nagios server over NRPE, registers them with that server, and optionally locks them down with Fail2Ban.

I wrote this in January 2017 as the Linux sysadmin for NTSG, a research group at the University of Montana. Part of that job was setting up monitoring and Fail2Ban on our servers. Adding a host meant changes on the host itself, its firewall, and the Nagios server's config, and this does all three in one run.

**Tech:** Ansible, Nagios, NRPE, Fail2Ban, firewalld, iptables, EPEL, CentOS 6/7

### What this playbook does

On each monitored host, it:

- installs NRPE and the standard Nagios plugins from EPEL, plus two community plugins (`show_users` and `check_mountpoints`)
- adds a common set of checks (users, load, zombie and total processes, swap, root disk, mount points)
- opens the NRPE port to the Nagios server only, with firewalld on CentOS 7 or iptables on CentOS 6
- writes a Nagios host definition from inventory variables

Then it:

- copies every host definition to the Nagios server and restarts Nagios
- installs Fail2Ban, with jails for SSH, sendmail, and NRPE, on the hosts whose group asks for it
- backs up every file it changes first, like my AD playbooks ([ntsg-combining-ad-nis-ansible](https://github.com/angiebrr/ntsg-combining-ad-nis-ansible) and [ntsg-configuring-ad-in-ansible](https://github.com/angiebrr/ntsg-configuring-ad-in-ansible))

## Using it

### Inventory

Copy `hosts.example` to `hosts`. The playbook expects two groups:

- `nagios_clients`: the hosts to monitor. The example builds it from child groups (`linux_servers`, `web_servers`) so each set of hosts can share Nagios templates and host groups.
- `nagios_server`: the one Nagios server.

Each monitored host (or its group) sets:

| Variable | What it's for |
|---|---|
| `host_alias` | The longer name shown in Nagios |
| `host_templates` | Nagios templates the host inherits from, e.g. `generic-host,linux-server` |
| `host_groups` | Nagios host groups the host belongs to |
| `install_fail2ban` | Whether to install and configure Fail2Ban on the host |

### Variable files

Paths and settings shared by all roles are in `defaults/local.yml`. The other files in `defaults/` (`client.yml`, `server.yml`, `fail2ban.yml`, `backups.yml`) list each role's defaults, commented out; uncomment a line to override it. The usual candidates are the firewalld zone (`work` by default), Fail2Ban's ban time and retry limits, and the Nagios server's host config directory (`/etc/nagios/conf.d/hosts` by default).

### Running the playbook

```bash
$ ansible-playbook -i hosts site.yml
```

The plays run with the `debug` strategy, so a failed task drops you into Ansible's debugger instead of stopping the run. Changed files are backed up and moved to `/etc/backups` at the end.
