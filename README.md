# Diff on Configs
Examples of several diffs on Cisco NX-OS using the [nxos_config Ansible module](http://docs.ansible.com/ansible/latest/nxos_config_module.html).

- [Check Mode](#running-config) `--check`
- [Running Config vs Startup Config](#running-config-vs-startup-config) `diff_against: startup`
- [Changed Config vs Running Config](#changed-config-vs-running-config) `diff_against: running`
- [Running Config vs Intended Config](#running-config-vs-intended-config) `diff_against: intended`

# Check Mode
The full playbook for this example is located here: [check.yml](check.yml)

When using the `src:` keyword with the [nxos_config module](http://docs.ansible.com/ansible/latest/nxos_config_module.html) it will merge the configuration provided in `src:` with the running-config on the target network host.  Ansible has a special `--check` flag where it will show you what "would" change but not actually change it.  In the following example there is a changed.txt file with two interfaces configured:
```
interface Ethernet1/4
  no switchport
  ip address 4.4.4.4/24
interface Ethernet1/5
  no switchport
  ip address 5.5.5.5/24
```
For the following playbook output `Ethernet1/4` has already been configured, but `Ethernet1/5` *has not*.  When registering the output from the nxos_config module it will save any changes (the diff) to `registered_variable_name.updates`

```
[root@localhost difftests]# ansible-playbook check.yml --check

PLAY [n9k] ********************************************************************

TASK [Going to merge change.txt to running-config] ****************************
changed: [n9k]

TASK [the only config we need to merge is]*************************************
ok: [n9k] => {
    "test_diff.updates": [
        "interface Ethernet1/5",
        "no switchport",
        "ip address 5.5.5.5/24"
    ]
}

PLAY RECAP
******************************************************************************
n9k                        : ok=2    changed=1    unreachable=0    failed=0
```

# Running-Config vs Startup-Config
The full playbook for this example is located here: [running_vs_startup.yml](running_vs_startup.yml)

It might be beneficial to see if the running-config is different from the startup-config (so if a power-loss or reboot was performed the switch and/or router would be in the same state it was prior.)  To compare the running-config to the startup-config with the [nxos_config module](http://docs.ansible.com/ansible/latest/nxos_config_module.html) use the `diff_against: startup`

```
- name: diff against the startup config
  nxos_config:
    diff_against: startup
    provider: "{{ provider }}"
  register: test_diff
```

The playbook must also be run with the `--diff` option

`ansible-playbook running_vs_startup.yml --diff`

This will produce output like this:
```
TASK [diff against the startup config] ***************************************
--- before
+++ after
@@ -47,6 +47,7 @@
   ip address 3.3.3.3/24
 interface Ethernet1/4
   no switchport
+  ip address 4.4.4.4/24
 interface Ethernet1/5
 interface Ethernet1/6
 interface Ethernet1/7
 ```
The diff shown here is in a standard GNU format known as the [Unified Format](http://www.gnu.org/software/diffutils/manual/diffutils.html#Detailed-Unified).

This line here specifically `@@ -47,6 +47,7 @@`

The first `47,6` refers to the startup-config.  Starting from line 47 (which is the `ip address 3.3.3.3/24` line). It will show 6 lines for context.  The second `+47,7` is the running-config.  Again starting from line 47, but this time there will be 7 lines shown (1 line difference as shown with the `+`).  

It might be desired to save the output from the before and/or after config to a file, or just print out the lines that are different (or many other various scenarios).  By registering the output from the diff to `test_diff` we can use a simple [Python filters](http://docs.ansible.com/ansible/latest/playbooks_filters.html).  In this example demonstration we just want the one line (`ip address 4.4.4.4/24`).  The playbook [running_vs_startup.yml](running_vs_startup.yml) shows more detail but the main Python filter is just `difference`

`set_fact: difference="{{ after | difference(before) }}"`

The playbook can now output exactly just the 1 line:
```
TASK [sanitized output "Lines added to running-config that are not present in startup-config"] **
ok: [n9k] => {
    "difference": [
        "  ip address 4.4.4.4/24"
    ]
}
```
This output gives no context but just shows one of various methods to possibly get the desired output in the format desired.

# Changed Config vs Running Config
The full playbook for this example is located here: [change_vs_running.yml](change_vs_running.yml)

The difference between using `diff_against: running` and `--check` is that it will actually make the change to the config, but show the diff as it performs the change.  As of Ansible 2.4 the `--check` and the `--diff` cannot be run at the same time.

```
[root@localhost difftests]# ansible-playbook change_vs_running.yml --diff

PLAY [n9k] ********************************************************************

TASK [diff against the startup config] ****************************************
--- before
+++ after
@@ -49,6 +49,8 @@
   no switchport
   ip address 4.4.4.4/24
 interface Ethernet1/5
+  no switchport
+  ip address 5.5.5.5/24
 interface Ethernet1/6
 interface Ethernet1/7
 interface Ethernet1/8

changed: [n9k]

TASK [Take config we are merging and sets it to a variable called *before*] ***
ok: [n9k]

TASK [This takes the running config and sets it to a variable called *after*] *
ok: [n9k]

TASK [Create a line-to-line diff of change-config to startup-config] **********
ok: [n9k]

TASK [sanitized output "Lines added to running-config that are not present in change.txt"] *
ok: [n9k] => {
    "difference": [
        "  ip address 5.5.5.5/24"
    ]
}

PLAY RECAP ********************************************************************
n9k                        : ok=5    changed=1    unreachable=0    failed=0
```

# Running Config vs Intended Config
The full playbook for this example is located here: [running_vs_intended.yml](running_vs_intended.yml)

To use compare the running config to an intended config you must use `diff_against: intended`.  This actually compares the running config to the intended config (which is the opposite of the rest of the diff modes, including just doing a check).  This means it is actually assuming intended_config is the final state of the box,  e.g. if the `src: change.txt` was just a couple interfaces you wanted to merge (as in the other examples) the diff would show that change.txt was missing a lot of config (including every other interface that was not part of `change.txt`) vs showing what *would* change (with a `--check`).  Most likely `intended` would be used with a previous backup, a templated file or some other *complete* config vs a piece of config being merged into the running config.

There is a [backup.yml](backup.yml) in this repo used for this example.
```
- name: create a backup
  nxos_config:
    backup: yes
    provider: "{{provider}}"
```

We can use the backup full name or we can just copy the backup config somewhere.  We can manually edit the config (lets add `6.6.6.6/24 to interface1/6`) then run the playbook with this task:
```
- name: diff against the startup config
  nxos_config:
    diff_against: intended
    provider: "{{ provider }}"
    intended_config: "{{ lookup('file', 'backup.txt') }}"
````
The playbook will now show the change of the running config VS the intended config (our local file) which just shows the `6.6.6.6/24`
```
[root@localhost difftests]# ansible-playbook intended_vs_running.yml --diff

PLAY [n9k] *******************************************************************

TASK [diff against the startup config] ***************************************
--- before
+++ after
@@ -50,8 +50,6 @@
   no switchport
   ip address 5.5.5.5/24
 interface Ethernet1/6
-  no switchport
-  ip address 6.6.6.6/24
 interface Ethernet1/7
 interface Ethernet1/8
 interface Ethernet1/9

changed: [n9k]

PLAY RECAP *******************************************************************
n9k                        : ok=1    changed=1    unreachable=0    failed=0
```

 ---
 ![Ansible Red Hat Engine](ansible-engine-small.png)

 In addition to open source Ansible, there is Red Hat® Ansible® Engine which includes support and an SLA for the nxos_facts module shown above.

 Red Hat® Ansible® Engine is a fully supported product built on the simple, powerful and agentless foundation capabilities derived from the Ansible project.  Please visit [ansible.com](https://www.ansible.com/ansible-engine) for more information.
