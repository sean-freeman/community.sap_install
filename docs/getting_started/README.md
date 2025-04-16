# Getting started

In this folder you will find sample files, a few additional tips for using the provided ansible roles, as well as references to further information.

- [How to run playbooks](#how-to-run-playbooks)
  - [Inventory and variable parameters](#inventory-and-variable-parameters)
  - [Security parameters](#security-parameters)
  - [Other useful options](#other-useful-options)
  - [Improve readability of playbook output in terminal](#improve-readability-of-playbook-output-in-terminal)

## How to run playbooks

Playbook execution requires a minimum set of parameters, depending on the ansible configuration settings and the environment to be used in.

The parameters shown in the below examples can all be combined according to the respective needs. The example commands are kept short for readability.

Simplest example:

```bash
ansible-playbook sap_hana_cluster_deploy.yml
```

_The parameter examples in the following sections can be combined according to the individual environment and needs._

### Inventory and variable parameters

Examples with different ways to use inventory and target system parameters:

```bash

ansible-playbook -i /path/to/inventory-file -l node1,node2 sap_hana_cluster_deploy.yml
ansible-playbook -l ~node[12] sap_hana_cluster_deploy.yml

ansible-playbook [...] -e 'my_variable=value'
ansible-playbook [...] -e @path/to/vars-file.yml
ansible-playbook [...] -e @path/to/vars-file.yml -e 'additional_variable=value'
```

- `-i` provide path to inventory host file
- `-l` limit the target hosts to the name(s).
  Prefix with `~` to use regex for host name patterns.
- `-e` (`--extra-vars`) allows to define variables on command execution.
  Variables can also be included from a file using the `@filename` notation.
  `-e` can be used multiple times.
  These extra-vars will overwrite the same variable names defined in the playbook or included files. See [variable precedence documentation](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#variable-precedence-where-should-i-put-a-variable).

### Security parameters

Examples showing some typical privilege escalation options:

```bash
ansible-playbook [...] -k
ansible-playbook [...] -k -K
ansible-playbook [...] -u target-user -k -K
```

- `-u` is used to target a different remote user than the current user on the ansible control node.
- `-k` will prompt for the remote user's login password.
- `-K` will prompt for the remote user's sudo password. This is needed when `become` is used to run tasks as another user or root. Instead of sudo the privilege escalation method can also be changed (please see official ansible documentation).

Example if encrypted files or variables are used in a play:

```bash
ansible-playbook [...] --ask-vault
```

This will prompt for the vault password, which was provided during creation of the ansible-vault encrypted contents.

Please see [secure-your-passwords](secure-your-passwords.md) for more information.

### Other useful options

There are also other very useful parameters for individual use cases.

```bash
ansible-playbook [...] --list-hosts

ansible-playbook [...] --list-tasks

ansible-playbook [...] -t tasks-tagged-name-a
ansible-playbook [...] -t tasks-tagged-name-a,tasks-tagged-name-b

ansible-playbook [...] --start-at-task "Name of task to start with and proceed"

ansible-playbook [...] -C
ansible-playbook [...] --step
```

These are not all available options, but ones that may help getting familiar with the tasks and changes the playbook would execute. Especially for a collection of complex roles this could help keeping an overview and defining steps, if desired.

- `--list-hosts` helps verifying the hosts that are going to be targeted. The playbook is not run.
- `--list-tasks` displays the tasks the playbook is going to run. Tasks from dynamically included files will not be visible and only accessed during runtime.
- `-t` or `--tags` limits the tasks to be executed to those with the tag name(s) provided.
  Tags are displayed in square brackets in the `--list-tasks` overview.
- `--start-at-task` will run the playbook starting with this named task. If there are duplicate task names it will start at the first in the list (see `--list-tasks`).
  Be careful to choose a task which covers pre-requisites, i.e. tasks that discover information which is used in subsequent tasks have to be run to fulfill conditionals.
- `-C` attempts a dry-run of the playbook without applying actual changes. This is limited to simple tasks that do not require other changes already been done in previous tasks.
- `--step` this executes the playbook but will prompt for every task to be run or skipped. At the prompt it can also be told to continue and not ask again, however. Useful to slow down execution and review each tasks result before proceeding with the next task.

### Improve readability of playbook output in terminal
Note: For terminals with dark background, replace the color code `30m` by `37m`.
In case you need to make an invisible font readable on a terminal with dark background, run the following command in the terminal:
```yaml
printf "\033[37mreadable font\n"
```
In case you need to make an invisible font readable on a terminal with bright background, run the following command in the terminal:
```yaml
printf "\033[30mreadable font\n"
```

Execution of `sap_general_preconfigure` playbook with a nice compact and colored output, this time for two hosts:
```console
ansible-playbook sap.yml -l host_1,host_2 -e "{sap_general_preconfigure_assert: yes, sap_general_preconfigure_assert_ignore_errors: yes}" |
awk '{sub ("    \"msg\": ", "")}
  /TASK/{task_line=$0}
  /fatal:/{fatal_line=$0; nfatal[host]++}
  /...ignoring/{nfatal[host]--; if (nfatal[host]<0) nfatal[host]=0}
  /^[a-z]/&&/: \[/{gsub ("\\[", ""); gsub ("]", ""); gsub (":", ""); host=$2}
  /SAP note/{print "\033[30m[" host"] "$0}
  /FAIL:/{nfail[host]++; print "\033[31m[" host"] "$0}
  /WARN:/{nwarn[host]++; print "\033[33m[" host"] "$0}
  /PASS:/{npass[host]++; print "\033[32m[" host"] "$0}
  /INFO:/{print "\033[34m[" host"] "$0}
  /changed/&&/unreachable/{print "\033[30m[" host"] "$0}
  END{print ("---"); for (var in npass) {printf ("[%s] ", var); if (nfatal[var]>0) {
        printf ("\033[31mFATAL ERROR!!! Playbook might have been aborted!!!\033[30m Last TASK and fatal output:\n"); print task_line, fatal_line
     }
     else printf ("\033[31mFAIL: %d  \033[33mWARN: %d  \033[32mPASS: %d\033[30m\n", nfail[var], nwarn[var], npass[var])}}'
```
