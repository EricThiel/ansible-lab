# Hands On - Ansible for Network Work Management


# Setup and Preparation
## Devnet Sandbox
This lab was written to be run using the [DevNet Open NX-OS with Nexus 9K Sandbox](https://devnetsandbox.cisco.com/RM/Diagram/Index/0e22761d-f813-415d-a557-24fa0e17ab50?diagramType=Topology).  This sandbox is a basic CentOS 7 workstation with typical development tools and software installed.  Specifically used in this lab are Python 3.6 and a Nexus 9kv virtual switch. 

If you are doing this lab on your own, you'll need to reserve an instance of this sandbox before beginning.  If you are doing this as part of a guided workshop, the instructor will assign you a pod.  

### Steps to complete preparation 
1. Using either AnyConnect or OpenConnect, establish a VPN to your pod.  
1. SSH to the Devbox at IP `10.10.20.20` using credentials `root / cisco123` 

    ```bash
    ssh root@10.10.20.20
    ```

1. Clone the code samples to the devbox from GitHub and change into the directory. 

    ```bash
    cd code
    git clone https://github.com/securenetwrk/ansible-lab.git
    cd ansible-lab
    ```

1. Create a Python 3.6 virtual environment and install Python libraries for exercises. 
 
    ```bash
    python3.6 -m venv venv
    source venv/bin/activate 
    pip install -r requirements.txt
    ```

    This step will load essential python modules into your working environment including:
    
    ```bash
    ansible
    scp
    netaddr
    ```

1. Now that you have your requirements set, let's review some of what has been set up.
 

# Intro

1. `ansible.cfg` contains some basic settings for our instance of Ansible. 

    ```bash
    [defaults]
    host_key_checking = False
    inventory      = ./inventory
    roles_path     = ./
    
    # Ignore some expected output
    deprecation_warnings=False
    display_skipped_hosts = false
    
    [persistent_connection]
    connect_timeout = 150
    command_timeout = 140
    ```
    
1. `inventory/hosts` contains our default inventory hosts needed for this workshop

    ```bash
    [all:vars]
    ansible_python_interpreter="/usr/bin/env python"

    [localhost]
    127.0.0.1

    [nx]
    10.10.20.58 username=admin password=Cisco123
    
    ```
    
1. `group_vars/nx.yaml` contains variables that will apply to inventory group `nx`. These variables will be used by our playbooks throughout the workshop.

    ```yaml
    ansible_connection: network_cli
    ansible_network_os: nxos
    ansible_user: "{{username}}"
    ansible_ssh_pass: "{{password}}"
    ansible_remote_user: "{{username}}"
    remote_user: "{{username}}"
    ansible_httpapi_pass: "{{password}}"
    ansible_password: "{{password}}"
    
    target_version: 9.2(2)
    domain_name: sdlab.cisco.com
    ntp_servers: 
      - 8.8.8.8
      - 1.1.1.1
    log_servers:
      - 1.1.1.1
      - 1.1.1.2
    
    local_users_full:
      - name: cisco
        roles: network-admin
        configured_password: "{{password}}"
      - name: admin
        roles: network-admin
        configured_password: "{{password}}"
      - name: admin2
        roles: network-admin
        configured_password: "{{password}}"
    
    siteid: 51
    tenants:
      - tenant_name: finance
        tenant_num: 11
        segments:
          - vlan_num: 1
            name: "{{siteid}}-finance-data"
            subnet: "10.{{siteid}}.111.0/24"
          - vlan_num: 2
            name: "{{siteid}}-finance-voice"
            subnet: "10.{{siteid}}.112.0/24"
      - tenant_name: engineering
        tenant_num: 12
        segments:
          - vlan_num: 1
            name: "{{siteid}}-engineering-data"
            subnet: "10.{{siteid}}.121.0/24"
          - vlan_num: 2
            name: "{{siteid}}-engineering-voice"
            subnet: "10.{{siteid}}.122.0/24"
      - tenant_name: hr
        tenant_num: 13
        segments:
          - vlan_num: 1
            name: "{{siteid}}-hr-data"
            subnet: "10.{{siteid}}.131.0/24"
          - vlan_num: 2
            name: "{{siteid}}-hr-voice"
            subnet: "10.{{siteid}}.132.0/24"
      - tenant_name: facilities
        tenant_num: 14
        segments:
          - vlan_num: 1
            name: "{{siteid}}-facilities-data"
            subnet: "10.{{siteid}}.141.0/24"
          - vlan_num: 2
            name: "{{siteid}}-facilities-voice"
            subnet: "10.{{siteid}}.142.0/24"
    ```


# NX OS configurations

## Enable NX-API
* Execute `ansible-playbook 1_nxapi.yml`
  * In the first example, we have a simple playbook designed to enable NX-API on an NX-OS switch. 


  ```yaml
  ---
  - name: Enable NX-API before any other tasks
    hosts: nx
    gather_facts: no

    tasks:

    - name: Enable NXAPI access with default configuration
      nxos_nxapi:
    state: present
    enable_http: yes
    enable_https: yes
  ```

1. `hosts: nx` - This line tells Ansible to target all hosts defined in the "nx" group in the inventory file. 

1. `tasks:` - This header begins the list of tasks to be run on each host

1. `nxos_nxapi:` - Within each task is a module to be executed. The first (and only) task in this playbook will use the nxos_nxapi module


## Gather information, upload new 'os' if current is out of date
* Execute `ansible-playbook 2_sh_facts.yml`
  * In the second example, we have a simple playbook designed to gather information about the switch, and take specific actions based on that information. 


  ```yaml
  ---
  - name: Gather information about nxos device
    hosts: nx
    gather_facts: no

    tasks:

    - name: Gather facts (ops)
      when: ansible_network_os == 'nxos'
      nxos_facts:
    gather_subset: all
  
    - name: Version is too old
      when: ansible_net_version < target_version
      debug: msg="Version {{ansible_net_version}} is older than {{target_version}}"

    - name: Version is too new
      when: ansible_net_version > target_version
      debug: msg="Version {{ansible_net_version}} is newer than {{target_version}}"

    - name: Display current OS version
      when: ansible_net_version == target_version
      debug: msg="Version {{ansible_net_version}} is already at current target version"

    - name: Copy correct NXOS binary to switch if version is not correct
      when: ansible_net_version != target_version
      nxos_file_copy: 
    local_file: "./9.2.2.txt"
    remote_file: "9.2.2.txt"
  ```

1. `gather_facts: no` - For networking equipment we generally want to skip the built in fact checker, and call a device-specific one as a task. 

1. `- name: Gather facts (ops)` - This task gathers basic information about the target including version details using a module designed specifically for that platform. 

1. `when: ansible_network_os == 'nxos'` - Only execute on devices defined as `nxos` os type

1. `nxos_facts:` - Use the device-specific module `nxos_facts` to gather information about nxos devices

1. `when: ansible_net_version < target_version` - This task looks at the `ansible_net_version` created by nxos_facts and takes action `when` the current version number is smaller than the `target_version` specified in the variable file. In this case the action is to print a warning about the version being old. 

1. `when: ansible_net_version > target_version` - Same as above, except if the version is too new.

1. `when: ansible_net_version == target_version` - Same as above, but xecute this task only if the device has the exact target version.

1. `when: ansible_net_version != target_version` - Execute this task only on hosts that do not have the correct target version. 

1. `nxos_file_copy:` - This module can copy files to an NXOS device filesystem using SCP

1. `local_file: "./9.2.2.txt"` - Specify the local file to copy to the device. For demo purposes we are using a "fake" NX OS binary to save time on the copy.  

1. `remote_file: "9.2.2.txt"` - Specify the target file name on the device bootflash. 


## Review target devices for gold config compliance
* Execute `ansible-playbook 3_gold_config.yml -C -v`
  * For the initial run, we are using `-C -v` to tell ansible to only 'check' the device and report what changes it _would_ do. This is a very important capability when creating new playbooks to test before running them against production devices. 

  ```yaml
  ---
  - name: Check if devices match the Gold config
    hosts: nx
    gather_facts: no
  
    tasks:
    - name: Check gold config status
      nxos_config:
        lines:
          - clock timezone PST 8 0
          - logging timestamp milliseconds
          - no ip source-route
        backup: yes
      when: ansible_network_os == 'nxos'
  
    - name: Set domain attributes
      nxos_system:
        domain_lookup: False
        domain_name: "{{domain_name}}"
  
    - name: Set standard logging settings
      nxos_logging:
        aggregate:
          - { dest: console, dest_level: 7 }
          - { dest: logfile, dest_level: 6, name: mylog }
        state: present
  
    - name: Enable standard features
      loop:
        - scp-server
        - sftp-server
        - interface-vlan
      nxos_feature:
        feature: "{{item}}"
        state: enabled
  
    - name: Configure VTY and console
      when: ansible_network_os == 'nxos'
      loop: 
        - line vty
        - line console
      nxos_config:
        lines:
          - exec-timeout 525600
        parents: "{{ item }}"
  
    - name: Set NTP
      when: ansible_network_os == 'nxos'
      loop: "{{ ntp_servers }}"
      nxos_config:
        lines:
          - ntp server {{ item }} use-vrf default
  
    - name: Create temp user
      when: ansible_network_os == 'nxos'
      nxos_config:
        lines:
          - username deleteme password C1sco12345 role network-admin
  ```

1. `nxos_config:` - The nxos_config module provides a catch-all for configuration changes. For this task, we are using it to ensure several best practice settings are on the device. 

1. `lines:` - nxos_config expects one or more lines of configuration to be passed to it. For this task we pass a list with multiple configuration lines

1. `backup: yes` - nxos_config supports saving a local copy of the configuration when executing. This is a good way to document before and after state for your playbook. 

1. `nxos_system:` - The nxos_system module can be used to set system-level attributes. In this task, we are setting dns related system attributes. 

1. `domain_name: "{{domain_name}}"` - Using the nxos_system module, and the `domain_name` variable which could be set universally, per group, or per host, we set the system's lookup domain name. If `domain_name` is not set the play will fail. 

1. `nxos_logging:` - The nxos_logging module is used to set attributes for the system logging facility. 

1. `aggregate:` - Many modules support passing in an "aggregate", which is a sequence or mapping containing multiple attributes. This is much more efficient when creating or modifying multiple items with different parameters 

1. `- { dest: console, dest_level: 7 }` - Inform the nxos_logging module that console logging should be set to level 7 messages.

1. `- { dest: logfile, dest_level: 6, name: mylog }` - Inform the nxos_logging module that file logging should be set to use file name 'mylog' and capture level 6 messages. 

1. `state: present` - For most modules, a state must be set. If 'present', Ansible will ensure the configuration exists. If 'absent', it will make sure the configuration is removed. 

1. `loop:` - loop is a powerful mechanism within Ansible to iterate over sequences or mappings and execute the task multiple times. When iterating over a sequence, each loop the variable `item` will contain the current sequence item. For this example, the task will loop 3 times, each time enabling one of the features listed under the loop parameter.

1. `nxos_feature:` - The nxos_feature module is used to manage the enabled features on an nxos device. 

1. `feature: "{{item}}"` - For loop number one, this will tell nxos_feature to enable scp-server, for loop two it will enable sftp-server, and for loop 3 it will enable interface-vlan. 

1. `- name: Configure VTY and console` - This task will loop over 2 terminal types, vty and console, and apply an exec-timeout to each

1. `parents: "{{ item }}"` - In order to set the exec-timeout on each type of terminal, we must first execute a command to enter that config mode. "parents" allows typing a command before applying the config within the nxos_config module. 
    * In this case, for loop 1 it will run `line vty` prior to running `exec-timeout 525600`. 
    * For loop 2, it will run `line console` prior to running `exec-timeout 525600`.

1. `- name: Set NTP` - Make note in this task the explicit setting of `use-vrf default`. Often the command `ntp server 1.1.1.1` is used, but in the configuration it actually appears with the use-vrf at the end. This is important to include for Ansible so it understands that the configuration is correct. If that was left off in the task, it would always think the configuration had changed and would re-apply the config. 

1. `- name: Create temp user` - This task was created to demonstrate the issue mentioned above. This task creates a user by passing a plain-text password, but when it is stored in the configuration the password is obfuscated. As a result, every run will attempt to re-apply the configuration. Sometimes this can be solved as with ntp above, but often you will have one or more tasks that always believe the config has changed. It is best to group these tasks together so that auditing the results to detect actual config drift is easier. 

* Execute `ansible-playbook 3_gold_config.yml`
  * This time we left off the `-C -v` so the playbook will execute the defined tasks. 
  * If you execute it a second time, what do you expect for the results? Try it and see
     <details>
     <summary> Answer </summary>
     * All tasks should show 'ok' except the last task. The last task will show 'changed' every time because it applies the password in plaintext, but the password is stored in the configuration encrypted. 
     </details>
  * How could this issue be avoided?
     <details>
     <summary> Answer </summary>
     * If you manually add and capture the configuration from the device, in this case with an encrypted password, you can often use that exact config line on other devices as well. 
     </details>


## blah
* Execute `blah`
  * blah 

  ```yaml


  ```




1. `blah` - blah

1. `blah` - blah

1. `blah` - blah

1. `blah` - blah

1. `blah` - blah





## blah
* Execute `blah`
  * blah 

  ```yaml


  ```
1. `blah` - blah

1. `blah` - blah

1. `blah` - blah

1. `blah` - blah

1. `blah` - blah


## blah
* Execute `blah`
  * blah 

  ```yaml


  ```
1. `blah` - blah

1. `blah` - blah

1. `blah` - blah

1. `blah` - blah

1. `blah` - blah


## blah
* Execute `blah`
  * blah 

  ```yaml


  ```
1. `blah` - blah

1. `blah` - blah

1. `blah` - blah

1. `blah` - blah

1. `blah` - blah


