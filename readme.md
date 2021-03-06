server-nagios ansible role
================================

This is an ansible role to set up nagios on a network. The idea is to use ansible inventory and vars to "automatically" set up hosts and services in nagios.

The target network is 10-20 devices and 1-3 admins, not large scale enterprise.


Design decisions
---------------------

* this is a stand-alone role

    Others part of ansible will have to handle firewalls, package management, users, logging and whatever else you want of a server.

    Currently, `apache2` is installed as part of the role. This might change.

* ansible groups becomes nagios hostgroups

    Devices in a group will automatically be in a nagios group of the same name
    Groups named `all` and `ungrouped` are ignored by default.

* provisioning to nagios is independent of fact collection

    All the info that nagios needs are in `group_vars` and `host_vars`, so it is sufficient to have the devices in inventory.

* notifications use email

    Proper emailing must be set up on the system, e.g using `dpkg-reconfigure exim4-config`

* notification is specified per contact

    Default is to use email. A notify list containing X, Y and Z gets converted to `notify-host-by-X`, `notify-host-by-Y` and `notify-host-by-Z`, which must exist, probably defined in `custom-objects`.

* configurations are split into default, ansible and non-ansible.

    The configuration is default in `/usr/local/nagios/etc`. Main config file is `nagios.cfg` (ansible controlled) and the subdirectories contains other config. Path is specified using `nagios_base_dir`.

    `ansible-hosts` contains the autogenerated config by ansible.

    `custom-objects` contains whatever manual stuff the admin want also, like non-default commands and non-ansible hosts.

* role is usable `out-of-the-box`, ie. no config is need to have an minimal setup.

Setup
-------------

See `tests` directory for details

We download nagios from [github](https://github.com/NagiosEnterprises/nagioscore/releases) and the version specified in `nagios_version` must match that.

Nagios plugins are also downloaded from [github](https://github.com/nagios-plugins/nagios-plugins/releases) and `nagios_plugins_version` must match a version there.

### Minimal config

The role may be used without setting any `host_vars` or `group_vars`. See [test-defaults/test.yml](https://github.com/moozer/ansible-role-nagios/blob/master/tests-defaults/test.yml) for an example

So a playbook like this is minimum

```
- hosts: all

  roles:
  - {role: server-nagios, become: true }

```

This assumes that the roles is pulled to the directory `roles/server-nagios`.

Notifications will be minimal, localhost will have some checks, and every other device in `inventory` will be a host with no service checks.



### Small example

* In the example the nagiosserver is called `nagioshost` and there is another host called `otherhost`


#### `inventory`

* the two servers are in the `servers` group and one in the `otherservers` group

```
[servers]
nagioshost
otherhostA

[otherservers]
otherhostB
```

#### `playbook.yml`

* in the playbook we define how to provision

```
- hosts: nagioshost

  roles:
  - { role: server-nagios, become: true }

# provision other hosts if applicable
#- hosts: otherhost?
#
#  roles:
#  - {}
```

### Defining variables for the host groups

* one group called `servers` and another called `otherservers`

#### `group_vars/all.yml` or `group_vars/all/nagios.yml`

* define the users that will be referenced in notifications
* define contact info for the default nagiosadmin user
* Both users A and B will be notified by `email` (since it is default), and `userA` will also be notified using `othermean`.
* User A is in the admin group and will receive notification for that group

```
nagios_contacts:
  - name: userA
    mail: userA@somewhere
    notify:
      - email
      - othermean
  - name: userB
    mail: userB@somewhere

nagios_admin_group_contact: userA
```

#### `group_vars/servers.yml` or `group_vars/servers/nagios.yml`

* define who should be notified when something happens
* define the tests that are to be used for all groups. The text written will be used directly as check command. See the nagios documentation for [details](https://assets.nagios.com/downloads/nagioscore/docs/nagioscore/3/en/plugins.html).

```
nagios_group_contacts:
  - userA
  - userB

nagios_group_tests:
  ping: "check_ping!100.0,20%!500.0,60%"
  ssh: check_ssh
```

#### `group_vars/otherservers.yml` or `group_vars/otherservers/nagios.yml`

* the group `otherservers` are not to be included in nagios
* hosts in the groups will, unless something else is defined in hostvars, be excluded from nagios.

```
nagios_exclude_group: True
nagios_exclude_host: True
```

### Defining for the hosts

* host specific tests are not implemented yet

#### `host_vars/nagioshost.yml` or `host_vars/nagioshost/nagios.yml`

* `userA` is the owner and will be notified. Default owner is `nagiosadmin`.

```
nagios_host_owner: userA

nagios_admin_user: nagiosadmin
nagios_admin_pass: somepass
```

#### `host_vars/otherhostA.yml` or `host_vars/otherhostA/nagios.yml`

* `userB` is the owner and will be notified.
* Specific to `otherhostA`, HTTP will also be checked. Other checks from the group are still included.

```
nagios_host_owner: userB

nagios_host_checks:
  http: check_http
```
