# Hands On - Ansible for Network Work Management

# Index:
* [Setup and Preparation](https://github.com/securenetwrk/ansible-lab#setup-and-preparation)
* [Intro](https://github.com/securenetwrk/ansible-lab#intro)
* [NX OS configurations](https://github.com/securenetwrk/ansible-lab#nx-os-configurations)
  * [Enable NX-API](https://github.com/securenetwrk/ansible-lab#enable-nx-api)
  * [Gather information, upload new 'os' if current is out of date](https://github.com/securenetwrk/ansible-lab#gather-information-upload-new-os-if-current-is-out-of-date)
  * [Review target devices for gold config compliance](https://github.com/securenetwrk/ansible-lab#review-target-devices-for-gold-config-compliance)
  * [Check if devices have all expected users, and purge any extra users](https://github.com/securenetwrk/ansible-lab#check-if-devices-have-all-expected-users-and-purge-any-extra-users)
  * [Set port descriptions on ports, and shut down unused ports](https://github.com/securenetwrk/ansible-lab#set-port-descriptions-on-ports-and-shut-down-unused-ports)
  * [Use roles to create VRFs](https://github.com/securenetwrk/ansible-lab#use-roles-to-create-vrfs)

---

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
 
---

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

---

# NX OS configurations

## Enable NX-API
---
* Execute `ansible-playbook 1_nxapi.yml`
---
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
---
* Execute `ansible-playbook 2_sh_facts.yml`
---
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
---
* Execute `ansible-playbook 3_gold_config.yml -C -v`
---
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

---
* Execute `ansible-playbook 3_gold_config.yml` again
---
  * This time we left off the `-C -v` so the playbook will execute the defined tasks. 
  * If you execute it a second time, what do you expect for the results? Try it and verify
     <details>
     <summary> Answer </summary>
     * All tasks should show 'ok' except the last task. The last task will show 'changed' every time because it applies the password in plaintext, but the password is stored in the configuration encrypted. 
     </details>
  * How could this issue be avoided?
     <details>
     <summary> Answer </summary>
     * If you manually add and capture the configuration from the device, in this case with an encrypted password, you can often use that exact config line on other devices as well. 
     </details>


## Check if devices have all expected users, and purge any extra users
---
* Execute `ansible-playbook 5_users.yml`
---
  * In this example we will manage local users across all devices. As was demonstrated in the past lab, using `-C -v` to confirm what a playbook like this will do before running in production is always a good idea to avoid unintended consequences.  

  ```yaml
  ---
  - name: Check if devices have all expected users, and purge any extra users
    hosts: nx
    gather_facts: no
  
    tasks:
    - name: Create local device users
      nxos_user:
        aggregate:  "{{ local_users_full }}"
        state: present
        purge: yes
        update_password: on_create
  ```

1. `nxos_user:` - This module is exclusively for managing local user accounts on NX-OS devices

1. `aggregate:  "{{ local_users_full }}"` - In this instance, we are passing in a variable that contains attributes for multiple users. If you look at `group_vars/nx.yaml` you will see a section that looks like the following which lists multiple users, their role, and their password, which is enough for the module to add any missing users:

  ```yaml
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
  ```

1. `purge: yes` - The purge parameter tells the module that the list of users passed in with aggregate is complete, and any additional users found on any device should be removed.  

1. `update_password: on_create` - This setting tells the module that if the user already exists, do not attempt to change the password. Since the module does not know the current password, this allows for safe creation of new users without potentially changing the password for existing users. 


## Set port descriptions on ports, and shut down unused ports
---
* Execute `ansible-playbook 6_unused_port.yml`
---
  * This playbook is designed to check the current state of ports, and take action based on their state.  

  ```yaml
  ---
  - name: Set port descriptions on ports, and shut down unused ports
    hosts: nx
    gather_facts: no
  
    tasks:
    - name: Gather port status
      nxos_command:
        commands:
          - show interface status | json
      register: if_state
  
    - name: Set description on access ports
      loop: '{{ if_state.stdout[0].TABLE_interface.ROW_interface  }}'
      when: 
        - item.state == 'connected'
        - item.name is not defined
        - "'u' not in item.vlan"
      nxos_interface: 
        name: '{{item.interface}}'
        description: 'Access port - vlan {{item.vlan}}'
  
    - name: Set description on routed ports
      loop: '{{ if_state.stdout[0].TABLE_interface.ROW_interface  }}'
      when: 
        - item.state == 'connected'
        - item.name is not defined
        - item.vlan == 'routed'
      nxos_interface: 
        name: '{{item.interface}}'
        description: Routed port
  
    - name: Set description on trunk ports
      loop: '{{ if_state.stdout[0].TABLE_interface.ROW_interface  }}'
      when: 
        - item.state == 'connected'
        - item.name is not defined
        - item.vlan == 'trunk'
      nxos_interface: 
        name: '{{item.interface}}'
        description: Trunk port
  
    - name: Set description and shut down unused ports
      loop: '{{ if_state.stdout[0].TABLE_interface.ROW_interface  }}'
      when: 
        - item.state == 'notconnect'
        - item.interface|length <= 11
      nxos_interface: 
        name: '{{item.interface}}'
        description: Unused port
        admin_state: down

  ```

1. `nxos_command:` - The nxos_command module is a catch-all for running non-configuration commands

1. `- show interface status | json` - Using the command module, this task executes a show interface status command, with the output formatted in JSON for easier parsing

1. `register: if_state` - Store the output of any commands run into the variable `if_state` for later reference

1. `- name: Set description on access ports` - This task will use the information gathered above to detect access ports and set a description

1. `loop: '{{ if_state.stdout[0].TABLE_interface.ROW_interface  }}'` - This task will loop over each item within the JSON output stored in if_state. `stdout[0]` refers to the output from the first command in that task, and `TABLE_interface.ROW_interface` is where NX-OS stores the output of the command within JSON. Inside ROW_interface is a list of interfaces and their attributes. 

1. `when:` - We only want to execute each of the following tasks under certain circumstances. When passed a list of conditions, by default the task will require all to be true to execute

1. `item.state == 'connected'` - The next threee tasks should only be run against interfaces that have a state of `connected`

1. `- item.name is not defined` - The next threee tasks should only be run against interfaces that do not already have a description set

1. `- "'u' not in item.vlan"` - This task will only execute if the current 'vlan' does not the letter u. This condition excludes ports that are `routed` or `trunk`, but not ports assigned to a single vlan

1. `nxos_interface:` - This module can set various attributes on an interface on an NX-OS device

1. `name: '{{item.interface}}'` - For each loop, configure the port with the name specified in the 'interface' field of the show interface status output. (e.g. Ethernet1/1, Ethernet1/2)

1. `description: 'Access port - vlan {{item.vlan}}'` - For each interface the task loops over, if the port is an access port, set the description to `Access port - vlan ` followed by the VLAN it is a member of.

1. `- name: Set description on routed ports` - This task will use the information gathered above to detect routed ports and set a description

1. `- item.vlan == 'routed'` - Only execute if the port is in routed mode

1. `description: Routed port` - Set the port description to 'Router port' if not otherwise set

1. `- name: Set description on trunk ports` - This task will use the information gathered above to detect trunk ports and set a description

1. `- item.vlan == 'trunk'` - Only execute if the port is in trunk mode

1. `description: Trunk port` - Set the port description to 'Trunk port' if not otherwise set

1. `- name: Set description and shut down unused ports` - Set description and shut down unused ports

1. `- item.state == 'notconnect'` - Detect if the port is disconnected

1. `- item.interface|length <= 11` - To speed up the playbook for demo purposes, only run on ports with 11 or less characters (skip Ethernetx/yy and Ethernetx/zzz)

1. `description: Unused port` - Set the port description to 'Unused port' if the link is currently down

1. `admin_state: down` - Admin disable any ports currently in down state


## Use roles to create VRFs
* We will now look at leveraging Ansible roles created by someone else to accomplish common tasks. We will start with a basic VRF role
  * Browse to [Ansible Galaxy](http://galaxy.ansible.com) and see the existing roles that you can choose from. Any of these roles can be downloaded with the `ansible-galaxy` command
  * In the left nav bar, click on search, and search for `network_vrf`. Within that role Read Me, you can see details about supported versions, sample playbooks, and a list of variables supported. There is also a link to the GitHub repo if you have suggestions or issues to submit.
---
* On your jump host, execute `ansible-galaxy install securenetwrk.network_vrf -p roles/` to download the role to your local host
---

  ```yaml
  ---
  - name: Use roles to create VRFs
    hosts: nx
    gather_facts: false
  
    roles: 
      - securenetwrk.network_vrf

  ```
1. `roles:` - Instead of defining specific tasks, this playbook will assign roles to the hosts defined with `hosts:`

1. `- securenetwrk.network_vrf` - Assign the role you just downloaded `securenetwrk.network_vrf` to the hosts in group `nx`. Within that role are two key files. 

    * roles/securenetwrk.network_vrf/tasks/main.yml

    ```yaml
    ---
    # tasks file for network_vrf
    - name: NX-OS Devices
      include_tasks: nxos.yml
      when: ansible_network_os == "nxos" and tenants is defined    
    ```

1. `main.yml` - This task list simply checks the ansible_network_os of the hosts the role is applied to, and if it is  NX-OS it will execute the tasks from nxos.yml. nxos.yml relies on multiple variables which have been set in group_vars/nx.yaml

    * roles/securenetwrk.network_vrf/tasks/nxos.yml

    ```yaml
    ---
    ############################################################
    #  VLAN Configuration
    ############################################################
    
    - name: Configure VLANs as defined for tenants
      tags: [nxapi, vrf]
      nxos_vlan:
        vlan_id: "{{item.0.tenant_num}}{{item.1.vlan_num}}"
        name: "{{item.1.name}}"
        state: present
      loop: "{{ tenants|subelements('segments') }}"
    
    #############################################################
    # VRF Configuration
    #############################################################
    
    - name: ensure VRF are created
      nxos_vrf:
        vrf: "{{item.tenant_name}}-vrf"
        state: present
      loop: "{{tenants}}"
    
    - name: Create VLAN interface
      nxos_interface:
        name: "Vlan {{item.0.tenant_num}}{{item.1.vlan_num}}"
      loop: "{{ tenants|subelements('segments') }}"
    
    - name: Assign interfaces to VRF
      nxos_vrf_interface:
        vrf: "{{item.0.tenant_name}}-vrf"
        interface: "vlan{{item.0.tenant_num}}{{item.1.vlan_num}}"
      loop: "{{ tenants|subelements('segments') }}"
    
    #############################################################
    # Layer 3 Configuration
    #############################################################
    
    - name: Ensure ip address is configured on Interfaces
      nxos_l3_interface:
        name: "vlan{{item.0.tenant_num}}{{item.1.vlan_num}}"
        state: present
        ipv4: "{{ item.1.subnet | ipaddr('1') | ipaddr('address')}}/{{item.1.subnet | ipaddr('prefix') }}"
      loop: "{{ tenants|subelements('segments') }}"
    ```

    ```yaml
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

1. `tags: [nxapi, vrf]` - Tags can be used to call selective tasks within a playbook without executing the full playbook. 

1. `loop: "{{ tenants|subelements('segments') }}"` - Ansible supports a number of capabilities with loop. In this example, we are looping through the list of 'segments' defined within each 'tenant'. 
    * Additional documentation on loops available at [docs.ansible.com](https://docs.ansible.com/ansible/latest/user_guide/playbooks_loops.html)

1. `nxos_vlan:` - Our first step will be to create VLANs

1. `vlan_id: "{{item.0.tenant_num}}{{item.1.vlan_num}}"` - When looping through subelements, item.0 represents the top level list (tenants) and item.1 represents the inner values (vlan_num=1, vlan_num=2). In this example we combine the tenant_num from the top level (e.g. 11, 12, 13) with the vlan_num from the subelement (e.g. 1, 2) to form a VLAN Id (e.g. 111, 112, 121, 122). 

1. `name: "{{item.1.name}}"` - For each VLAN, a name is defined in group_vars combining the siteid for that site (e.g. 51) with the intent for the vlan (e.g. finance-data). This name is used as the VLAN name. 

1. `nxos_vrf:` - Our second step will be to create the VRFs

1. `loop: "{{tenants}}"` - This task loops only over the list of tenants, creating one VRF per

1. `vrf: "{{item.tenant_name}}-vrf"` - Create a VRF based on the name of the tenant name (e.g. engineering-vrf)

1. `nxos_interface:` - Our third step will be to create the SVIs for the routed VLANs

1. `name: "Vlan {{item.0.tenant_num}}{{item.1.vlan_num}}"` - Create an SVI interface for each vlan created in step 1. (e.g. Int VLAN111)

1. `nxos_vrf_interface:` - Our fourth step will be to associate each SVI to the appropriate VRF

1. `vrf: "{{item.0.tenant_name}}-vrf"` - Specifiy the VRF to apply 

1. `interface: "vlan{{item.0.tenant_num}}{{item.1.vlan_num}}"` - Loop through all segments, associating each SVI to the appropriate VRF

1. `nxos_l3_interface:` - Our final step will be defining the IP address for each SVI

1. `name: "vlan{{item.0.tenant_num}}{{item.1.vlan_num}}"` - Loop through SVIs one segment at a time

1. `state: present` - Tell Ansible to ensure the defined IP is present

1. `ipv4: "{{ item.1.subnet | ipaddr('1') | ipaddr('address')}}/{{item.1.subnet | ipaddr('prefix') }}"` - Use the `ipaddr` filter to dynamically extract details about the subnet. 
    * `{{ item.1.subnet | ipaddr('1') | ipaddr('address')}}` uses the subnet defined in nx.yaml (e.g. 10.51.111.0/24) and returns the value of the first usable IP (10.51.111.1)
    * `{{item.1.subnet | ipaddr('prefix') }}` uses the subnet defined in nx.yaml (e.g. 10.51.111.0/24) and returns the value of the netmask (/24)
    * The [ipaddr filter](https://docs.ansible.com/ansible/latest/user_guide/playbooks_filters_ipaddr.html) is a very useful tool for turning a subnet definition into specific IPs. 




<!---
## blah
---
* Execute `ansible-playbook `
---
  * blah 

  ```yaml


  ```
1. `blah` - blah

1. `blah` - blah

1. `blah` - blah

1. `blah` - blah

1. `blah` - blah


--->
