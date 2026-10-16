---
tags:
  - gRPC
  - gNOI
  - Operation
  - Ping
  - SR Linux
---

# Automating network reachability tests using gNOI


|                             |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| --------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Activity name**           | Automating network reachability tests using gNOI |
| **Activity ID**           | 21 |
| **Short Description**       | Automate network reachability ICMP Ping test using gRPC gNOI and Ansible |
| **Difficulty**              | Beginner |
| **Tools used**              | [gNMIc](https://gnmic.openconfig.net/), [gNOIc](https://gnoic.kmrd.dev/), [Ansible for SR Linux](https://learn.srlinux.dev/ansible/collection/) |
| **Topology Nodes**          | :material-router: leaf21 |
| **References**              | [SR Linux documentation](https://documentation.nokia.com/srlinux/)<br/>[gRPC website](https://grpc.io/)<br/>[gNxI protobuf reference](https://gnxi.srlinux.dev/) |


You are a NOC engineer starting your shift now. There is a report of packet loss in a core link which may impact reachability to multiple applications from the edge of the data center network. You have been tasked to check the reachability of 10 applications from all 1000 multi-vendor leaf switches each containing 10 VRFs - 1 for each application. After completing this task in under 30mins, you are also required to take a configuration backup of all 1000 switches.

If these actions are performed manually, you will be issuing 10K ping commands and another 1K scp commands to take the configuration backup. Add another 2-3 hours to consolidate the output and find potential failures.

This is where automation comes to the rescue !

In this activity, you will create a proof of concept for your solution using a subset of nodes.

## Objective

On your group's hackathon instance, you will create an Ansible playbook to check the reachability of `devices` from `leaf21`. The playbook will use gRPC Network Operations Interface (gNOI) remote procedure calls (RPCs) to perform this task. At the end of the activity, take configuration backups of `leaf21`, `leaf22` and `leaf23` and store them on your hackathon instance.

In this activity, you will:

1. Use gNOI to perform a ping test to a remote destination from a SR Linux switch.
2. Automate the ping operation using an Ansible playbook
3. Use gNOI to backup the configuration file for a SR Linux device.

## Technology explanation

### gRPC
gRPC is an open source Remote Procedure Call (RPC) framework that implements gRPC services. To use a gRPC service, a gRPC client and a gRPC server are required. In this activity, the gRPC server is the router and the gRPC client is the server.  The client sends a gRPC request to the server who will then process the request and respond back to the client.

```
gRPC (client) --------(Request)-----------> gRPC (server)
gRPC (client) <-------(Response)----------- gRPC (server)
```

Examples of gRPC servers include SR OS and SR Linux.
Examples of gRPC clients include gNMIc, gNOIc and gNSIc.

### gNOI
gNOI (gRPC Network Operations Interface) is a gRPC service for executing operational commands on a network device. gNOI supports many services for different operations including, File Get, File Put, System Ping, System Traceroute and others.  Refer to the [gNxI reference](https://gnxi.srlinux.dev/) to see the details of all the gNOI RPCs supported on SR Linux.

For this activity, our focus will be on gNOI System service. Some of the common RPCs in this service are:

1. `Ping` - Ping executes the `ping` command on the target and streams back the results
2. `Traceroute` - Traceroute executes the `traceroute` command on the target and streams back the results
3. `Time` - Time returns the current time on the target
4. `Reboot` - Reboot causes the target to reboot
5. `SwitchControlProcessor` - SwitchControlProcessor will switch from the current route processor to the provided route processor.

### Ansible
[Ansible](https://www.ansible.com/) is a suite of software tools that enables infrastructure as code. The suite is open-source and includes software provisioning, configuration management and application deployment functionality. Ansible is designed with several key principles in mind. It has to be simple to understand, readable, extensible and there should be a gentle learning curve. The language used for defining what Ansible should do is YAML. Check out [some of the documentation](https://docs.ansible.com/ansible/latest/reference_appendices/YAMLSyntax.html) for the reasoning behind it and to see how YAML is used.

#### Inventory

In any Ansible deployment an inventory must be defined. This inventory includes the target hosts that could be used as well as some key information about them. This could include credentials, reachability information or some specific details required to be able to connect to the device.

#### Plays

Solutions built as Ansible playbooks are broken down into smaller pieces known as plays that are called from a playbook. A play may refer to a mapping of a role to one or more target hosts, though in general it refers to an ordered grouping of tasks mapped to specific hosts.

#### Playbooks

Finally, a playbook is what is executed from the CLI, potentially with some additional input flags. A playbook can call plays and import roles to perform complicated workflows.


## Preparing the switch

### gRPC configuration on SR Linux

Before starting the tasks, let's verify the gRPC configuration on `leaf21`.

gRPC is enabled without TLS under gRPC server called `insecure-mgmt` and the gNOI service you will use for this activity is enabled within it.

Verify the config of this grpc server.

/// tab | cmd

``` bash
info flat system grpc-server insecure-mgmt
```

///
/// tab | expected output

``` bash
set / system grpc-server insecure-mgmt admin-state enable
set / system grpc-server insecure-mgmt rate-limit 65000
set / system grpc-server insecure-mgmt network-instance mgmt
set / system grpc-server insecure-mgmt port 57400
set / system grpc-server insecure-mgmt trace-options [ request response common ]
set / system grpc-server insecure-mgmt services [ gnmi gnoi gnsi gribi p4rt ]
set / system grpc-server insecure-mgmt unix-socket admin-state enable
```
///

Based on the config, you will notice that the insecure gRPC server is listening on port `57400`.

### Client Verification

You will be using gNOIc as then client for the gNOI service.

Verify this client is installed on your group's hackathon instance.

/// tab | gnoic

``` bash
gnoic version
```

///
/// tab | gnoic expected output

``` bash
version : 0.2.1
 commit : 27bc5a6
   date : 2026-02-27T18:18:08Z
 gitURL : https://github.com/karimra/gnoic
   docs : https://gnoic.kmrd.dev
```
///


It is also acceptable to have newer versions of these clients.

If clients are not installed or if you need to update the clients to the latest version, refer to the source listed below.

- [gNOIc](https://gnoic.kmrd.dev/)

### Ansible verification

Ansible is installed on your hackathon instance. Verify by executing the following command on your instance.

/// tab | ansible

``` bash
ansible --version
```

///
/// tab | ansible expected output

``` bash
ansible [core 2.16.3]
  config file = None
  configured module search path = ['/home/nokia/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python3/dist-packages/ansible
  ansible collection location = /home/nokia/.ansible/collections:/usr/share/ansible/collections
  executable location = /usr/bin/ansible
  python version = 3.12.3 (main, Mar 23 2026, 19:04:32) [GCC 13.3.0] (/usr/bin/python3)
  jinja version = 3.1.2
  libyaml = True
```
///

If Ansible is not installed, run the below commands to install Ansible.

/// details | Ansible installation steps (if required only)
    type: code
```bash
sudo apt install python3-pip -y
sudo apt install ansible -y
```
///


### Test gNOIc connectivity to SR Linux device

Perform a connectivity check for gNOI to ensure gRPC is running on `leaf21`. You will use the `admin` user for all gNOI operations. The user password is provided in the sheet distributed during the hackathon.

For this test, use the gNOI `System` service `Time` RPC to get the current timestamp on `leaf21`.

/// tab | cmd

``` bash
gnoic -a clab-srexperts-leaf21 -u admin -p $EVENT_PASSWORD --insecure system time
```
///

/// tab | expected output

``` bash
+-----------------------------+------------------------------------------+---------------------+
|         Target Name         |                   Time                   |      Timestamp      |
+-----------------------------+------------------------------------------+---------------------+
| clab-srexperts-leaf21:57400 | 2026-04-25 22:32:58.749420494 +0300 EEST | 1745609578749420494 |
+-----------------------------+------------------------------------------+---------------------+
```
///

If you received the current system timestamp from `leaf21`, this confirms that gNOI is enabled on the router.

## Tasks

**You should read these tasks from top-to-bottom before beginning the activity**.  

It is tempting to skip ahead but tasks may require you to have completed previous tasks before tackling them.

### Use gNOIc to perform a remote ping from SR Linux

Our first task is to prepare the `gnoic` command to check the reachability of an IP address from `leaf21`.  You will
be using the `gNOI System Ping` RPC for this purpose. 

Check the reachability of `10.46.2.45` which is the system IP of `leaf23` from `leaf21`.

Run the below command from your host VM. Replace the `$EVENT_PASSWORD` with the value on your hackathon sheet.

/// tab | cmd

``` bash
gnoic -a clab-srexperts-leaf21 -u admin -p $EVENT_PASSWORD --insecure system ping --destination 10.46.2.45 --ns default --count 1 --wait 1s
```
///

/// tab | expected output

``` bash
56 bytes from 10.46.2.45: icmp_seq=1 ttl=63 time=7.409705ms
--- 10.46.2.45 ping statistics ---
1 packets sent, 1 packets received, 0.00% packet loss
round-trip min/avg/max/stddev = 7.410/7.410/7.410/0.000 ms
```
///

You confirmed that the IP is reachable from `leaf21`. The `--ns` option in the above command means, name space, and indicates the VRF used for the reachability test.

To get the output in `json` format, add the `--format json` option.

/// tab | cmd

``` bash
gnoic -a clab-srexperts-leaf21 -u admin -p $EVENT_PASSWORD --insecure system ping --destination 10.46.2.45 --ns default --count 1 --wait 1s --format json
```
///

/// tab | expected output

``` json
{
  "target": "clab-srexperts-leaf21:57400",
  "response": {
    "source": "10.46.2.45",
    "time": 4239013,
    "bytes": 56,
    "sequence": 1,
    "ttl": 63
  }
}
{
  "target": "clab-srexperts-leaf21:57400",
  "response": {
    "source": "10.46.2.45",
    "time": 4439022,
    "sent": 1,
    "received": 1,
    "min_time": 4239013,
    "avg_time": 4239013,
    "max_time": 4239013
  }
}
```
///

### Create Ansible playbook to check reachability of multiple IPs

With the knowledge gained from the above information, create an Ansible playbook to check the reachability of the following IPs from `leaf21` and `leaf23`. Keep the VRF name as a variable. For our exercise, all IPs should be reachable using `default` VRF.

You may use either CLI or JSON formatted response.

| IP | VRF | Description |
|----------|----------|----------|
| 10.46.2.44 | default | System IP on `leaf22` |
| 10.46.2.41 | default | System IP on `spine21` |
| 10.46.2.42 | default | System IP on `spine22` |

To take this challenge a notch up - add logic to return an error when an IP is not reachable. Test this using a random IP (like `3.3.3.3`) in your Ansible inventory.

**Expected result:** 

At the end of the playbook execution, for each leaf, there will be 3 successful task executions and 1 failed task execution (for the incorrect IP).

/// details | TIP: Executing Ansible playbok
    type: tip
To execute an ansible playbook along with an ansible inventory file:

```bash
ansible-playbook -i inventory-file.yml playbook-file.yml
```

///

If you need ideas or the complete solution, refer to the playbook below.

/// details | Solution
    type: success

Here's a sample inventory file. Make sure to replace the $EVENT_PASSWORD with the value provided in your hackathon sheet.

/// details | Ansible inventory
    type: code
```yaml
all:
  children:
    nokia_switches:
      hosts:
        # Define each switch and its management IP here
        leaf21:
          ansible_host: clab-srexperts-leaf21
        leaf23:
          ansible_host: clab-srexperts-leaf23

      vars:
        # Common credentials for all switches in this group
        gnoi_port: 57400
        gnoi_user: admin
        gnoi_pass: $EVENT_PASSWORD
        
        # The universal ping list
        ping_targets:
          - { ip: "10.46.2.44", vrf: "default" }
          - { ip: "10.46.2.41", vrf: "default" }
          - { ip: "10.46.2.42", vrf: "default" }
          - { ip: "3.3.3.3", vrf: "default" }
```
///

Here's the Ansible playbook:

/// details | Ansible playbook
    type: code
```yaml
---
- name: gNOI Ping for SReXperts
  hosts: nokia_switches
  gather_facts: false
  connection: local

  tasks:
    - name: Run gNOIc
      ansible.builtin.shell: >
        gnoic -a {{ ansible_host }}:{{ gnoi_port }} 
        -u {{ gnoi_user }} 
        -p {{ gnoi_pass }} 
        --insecure 
        system ping 
        --destination {{ item.ip }} 
        --ns {{ item.vrf }}
        --count 1
        --wait 1s
      loop: "{{ ping_targets }}"
      register: raw_ping_results
      changed_when: false
      ignore_errors: true 

    - name: Report Successful Pings
      vars:
        is_ok: "{{ '0.00% packet loss' in item.stdout and '1 packets received' in item.stdout }}"
      ansible.builtin.debug:
        msg: "[{{ inventory_hostname }}] {{ item.item.ip }} ({{ item.item.vrf }}) -> OK"
      when: is_ok
      loop: "{{ raw_ping_results.results }}"
      loop_control:
        label: "{{ item.item.ip }}"

    - name: Report Failed Pings
      vars:
        is_ok: "{{ '0.00% packet loss' in item.stdout and '1 packets received' in item.stdout }}"
      ansible.builtin.fail:
        msg: "[{{ inventory_hostname }}] {{ item.item.ip }} ({{ item.item.vrf }}) -> NOK"
      when: not is_ok
      loop: "{{ raw_ping_results.results }}"
      loop_control:
        label: "{{ item.item.ip }}"
      register: failure_check
      ignore_errors: false
```
///

Below is an example execution result of this playbook.

/// details | Ansible execution result
    type: code
```yaml
PLAY [gNOI Ping for SReXperts] *********************************************************************************************************************

TASK [Run gNOIc] ***********************************************************************************************************************************
ok: [leaf21] => (item={'ip': '10.46.2.44', 'vrf': 'default'})
ok: [leaf23] => (item={'ip': '10.46.2.44', 'vrf': 'default'})
ok: [leaf21] => (item={'ip': '10.46.2.41', 'vrf': 'default'})
ok: [leaf23] => (item={'ip': '10.46.2.41', 'vrf': 'default'})
ok: [leaf21] => (item={'ip': '10.46.2.42', 'vrf': 'default'})
ok: [leaf23] => (item={'ip': '10.46.2.42', 'vrf': 'default'})
ok: [leaf21] => (item={'ip': '3.3.3.3', 'vrf': 'default'})
ok: [leaf23] => (item={'ip': '3.3.3.3', 'vrf': 'default'})

TASK [Report Successful Pings] *********************************************************************************************************************
ok: [leaf21] => (item=10.46.2.44) => {
    "msg": "[leaf21] 10.46.2.44 (default) -> OK"
}
ok: [leaf23] => (item=10.46.2.44) => {
    "msg": "[leaf23] 10.46.2.44 (default) -> OK"
}
ok: [leaf21] => (item=10.46.2.41) => {
    "msg": "[leaf21] 10.46.2.41 (default) -> OK"
}
ok: [leaf23] => (item=10.46.2.41) => {
    "msg": "[leaf23] 10.46.2.41 (default) -> OK"
}
ok: [leaf21] => (item=10.46.2.42) => {
    "msg": "[leaf21] 10.46.2.42 (default) -> OK"
}
skipping: [leaf21] => (item=3.3.3.3) 
ok: [leaf23] => (item=10.46.2.42) => {
    "msg": "[leaf23] 10.46.2.42 (default) -> OK"
}
skipping: [leaf23] => (item=3.3.3.3) 

TASK [Report Failed Pings] *************************************************************************************************************************
skipping: [leaf21] => (item=10.46.2.44) 
skipping: [leaf21] => (item=10.46.2.41) 
skipping: [leaf23] => (item=10.46.2.44) 
skipping: [leaf21] => (item=10.46.2.42) 
skipping: [leaf23] => (item=10.46.2.41) 
skipping: [leaf23] => (item=10.46.2.42) 
failed: [leaf21] (item=3.3.3.3) => {"ansible_loop_var": "item", "changed": false, "item": {"ansible_loop_var": "item", "changed": false, "cmd": "gnoic -a clab-srexperts-leaf21:57400  -u admin  -p SReXperts2026!  --insecure  system ping  --destination 3.3.3.3  --ns default --count 1 --wait 1s\n", "delta": "0:00:01.121205", "end": "2026-04-22 23:37:06.353067", "failed": false, "invocation": {"module_args": {"_raw_params": "gnoic -a clab-srexperts-leaf21:57400  -u admin  -p SReXperts2026!  --insecure  system ping  --destination 3.3.3.3  --ns default --count 1 --wait 1s\n", "_uses_shell": true, "argv": null, "chdir": null, "creates": null, "executable": null, "expand_argument_vars": true, "removes": null, "stdin": null, "stdin_add_newline": true, "strip_empty_ends": true}}, "item": {"ip": "3.3.3.3", "vrf": "default"}, "msg": "", "rc": 0, "start": "2026-04-22 23:37:05.231862", "stderr": "", "stderr_lines": [], "stdout": "--- 3.3.3.3 ping statistics ---\n1 packets sent, 0 packets received, 100.00% packet loss\nround-trip min/avg/max/stddev = 0.000/0.000/0.000/0.000 ms", "stdout_lines": ["--- 3.3.3.3 ping statistics ---", "1 packets sent, 0 packets received, 100.00% packet loss", "round-trip min/avg/max/stddev = 0.000/0.000/0.000/0.000 ms"]}, "msg": "[leaf21] 3.3.3.3 (default) -> NOK"}
failed: [leaf23] (item=3.3.3.3) => {"ansible_loop_var": "item", "changed": false, "item": {"ansible_loop_var": "item", "changed": false, "cmd": "gnoic -a clab-srexperts-leaf23:57400  -u admin  -p SReXperts2026!  --insecure  system ping  --destination 3.3.3.3  --ns default --count 1 --wait 1s\n", "delta": "0:00:01.125527", "end": "2026-04-22 23:37:06.417328", "failed": false, "invocation": {"module_args": {"_raw_params": "gnoic -a clab-srexperts-leaf23:57400  -u admin  -p SReXperts2026!  --insecure  system ping  --destination 3.3.3.3  --ns default --count 1 --wait 1s\n", "_uses_shell": true, "argv": null, "chdir": null, "creates": null, "executable": null, "expand_argument_vars": true, "removes": null, "stdin": null, "stdin_add_newline": true, "strip_empty_ends": true}}, "item": {"ip": "3.3.3.3", "vrf": "default"}, "msg": "", "rc": 0, "start": "2026-04-22 23:37:05.291801", "stderr": "", "stderr_lines": [], "stdout": "--- 3.3.3.3 ping statistics ---\n1 packets sent, 0 packets received, 100.00% packet loss\nround-trip min/avg/max/stddev = 0.000/0.000/0.000/0.000 ms", "stdout_lines": ["--- 3.3.3.3 ping statistics ---", "1 packets sent, 0 packets received, 100.00% packet loss", "round-trip min/avg/max/stddev = 0.000/0.000/0.000/0.000 ms"]}, "msg": "[leaf23] 3.3.3.3 (default) -> NOK"}

PLAY RECAP *****************************************************************************************************************************************
leaf21                     : ok=2    changed=0    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0   
leaf23                     : ok=2    changed=0    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0   
```
///

///

## Additional Task

### Use gNOIc to get the configuration file from SR Linux
Our final task is to setup a remote configuration file backup service on the host VM.

Here are some guidelines to help you:

- The default SR Linux config file is `/etc/opt/srlinux/config.json`
- Use `gNOI file stat` RPC to check the presence of the config file. Use option `--path` to specify the config file path.
- Create a directory on your host VM to store the config backups
- Use `gNOI file get` RPC to transfer the config file to your backup directory. Use option `--file` to speficy the config file path and `--dst` to specify the destination directory.
- Verify the config is now present on your hackathon instance
- Create an Ansible playbook to take configuration backups of `leaf22` and `leaf23`.


/// details | Solution
    type: success

Use the `gNOI File Stat` RPC to check the presence of the config file. The default configuration file for SR Linux is located at `/etc/opt/srlinux/config.json`.

/// tab | cmd

``` bash
gnoic -a clab-srexperts-leaf21:57400 -u admin -p $EVENT_PASSWORD --insecure file stat --path /etc/opt/srlinux/config.json
```
///

/// tab | expected output

``` bash
+-----------------------------+------------------------------+---------------------------+------------+------------+--------+
|         Target Name         |             Path             |       LastModified        |    Perm    |   Umask    |  Size  |
+-----------------------------+------------------------------+---------------------------+------------+------------+--------+
| clab-srexperts-leaf21:57400 | /etc/opt/srlinux/config.json | 2026-04-24T19:16:13+03:00 | -rw-rw-r-- | -----w--w- | 121885 |
+-----------------------------+------------------------------+---------------------------+------------+------------+--------+
```
///

You confirmed that the configuration file exists and you also obtained information on the last modified timestamp and, the size of the file.

Now copy this file to our host using gNOI.

Create a directory called `my-backups` on your host to store the configuration backup file.

```bash
mkdir ~/my-backups
```

Use the `gNOI File Get` RPC to retrieve the file from `leaf21`.

/// tab | cmd

``` bash
gnoic -a clab-srexperts-leaf21:57400 -u admin -p $EVENT_PASSWORD --insecure file get --file /etc/opt/srlinux/config.json --dst ~/my-backups
```
///

/// tab | expected output

``` bash
INFO[0000] "clab-srexperts-leaf11:57401" received 64000 bytes 
INFO[0000] "clab-srexperts-leaf11:57401" received 57885 bytes 
INFO[0000] "clab-srexperts-leaf11:57401" file "/etc/opt/srlinux/config.json" saved 
```
///

Let's check that the file get operation was successful and the file exists on your groups hackathon instance.

/// tab | cmd

``` bash
ls -lrt ~/my-backups/etc/opt/srlinux
```
///

/// tab | expected output

``` bash
❯ ls -lrt ~/my-backups/etc/opt/srlinux
total 120
-rw-r--r-- 1 nokia nokia 121885 Apr 25 23:00 config.json
```
///

/// note

The destination file is created on the same path as the origin. In this case `/etc/opt/srlinux/`<br>

///

You may open the configuration backup to verify the contents.

Re-use the Ansible solution provided above to build an Ansible playbook for configuration backups.

///

## Summary

Congratulations! By this point you should have successfully completed the activity to check the reachability of your applications and take the configuration backup of all your switches. With the work you have put in, you should have acquired some new knowledge of the following topics:

- Using gRPC to perform activities on a remote router
- Using gNOI service and Ansible to automate network reachability tests
- Using the gNOI service to manipulate the file system
- Using gNOI service and Ansible to automate configuration backups

Good job in completing all the tasks in this activity! 
