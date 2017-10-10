# Diff on Configs
Examples of several diffs on Cisco NX-OS using the [nxos_config Ansible module](http://docs.ansible.com/ansible/latest/nxos_config_module.html).

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
TASK [sanitized output "Lines added to running-config that are not present in startup-config"] ****************************************************************************
ok: [n9k] => {
    "difference": [
        "  ip address 4.4.4.4/24"
    ]
}
```
This output gives no context but just shows one of various methods to possibly get the desired output in the format desired.


 ---
 ![Ansible Red Hat Engine](ansible-engine-small.png)

 In addition to open source Ansible, there is Red Hat® Ansible® Engine which includes support and an SLA for the nxos_facts module shown above.

 Red Hat® Ansible® Engine is a fully supported product built on the simple, powerful and agentless foundation capabilities derived from the Ansible project.  Please visit [ansible.com](https://www.ansible.com/ansible-engine) for more information.
