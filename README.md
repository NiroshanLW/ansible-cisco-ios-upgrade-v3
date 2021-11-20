Challenge

Upgrading 1-15 switches can be done manually. But what about 250+ switches including different models and versions? it will be challenging and time consuming. A simple humane error can cause troubleshooting of many hours. In this case, we were previously using manual upgrades for all of these 250+ switches. This post will explain the procedure and steps to automate this process using Ansible. At the end of this implementation, following benefits can be achieved,

Automated verifications

Reduced engineering hours and involvement

No need manual interventions

Automated configuration backup

Scalable inventory for future automation

Build your inventory first…

Before everything, an inventory supported for Ansible should be developed for the infrastructure. Here we use following YAML template to build our inventory.

all:
  children:
    switches:
      children: 
        Prod_Sites: 

    Prod_Sites:
      children:
        ABC:
        DEF:
        GHI:              
      vars:
        ansible_connection: network_cli
        ansible_network_os: ios
        ansible_user: XXXXXX 
        ansible_password: XXXXXX   

    ABC:
      hosts:
        SITE - ABC | SW - ABC-SW1 | IP - 10.10.10.1:
          ansible_host: '10.10.10.1'
        SITE - ABC | SW - ABC-SW2 | IP - 10.10.10.2:
          ansible_host: '10.10.10.2'

    DEF:
      hosts:
        SITE - DEF | SW - DEF-SW1 | IP - 10.10.20.1:
          ansible_host: '10.10.20.1'
        SITE - DEF | SW - DEF-SW2 | IP - 10.10.20.2:
          ansible_host: '10.10.20.2'

    GHI:
      hosts:
        SITE - GHI | SW - GHI-SW1 | IP - 10.10.30.1:
          ansible_host: '10.10.30.1'
        SITE - GHI | SW - GHI-SW2 | IP - 10.10.30.2:
          ansible_host: '10.10.30.2'

First we define a branch to define our production switches.

  children:
    switches:
      children: 
        Prod_Sites: 

Then credentials and variables common for all switches are defined under Prod_Sites. Prod_Sites includes many sites as shown here - ABC, DEF - these are site codes… In our case, we are having 35+ sites.

    Prod_Sites:
      children:
        ABC:
        DEF:
        GHI:              
      vars:
        ansible_connection: network_cli
        ansible_network_os: ios
        ansible_user: XXXXXX 
        ansible_password: XXXXXX  

Following variables are defined here in addition to the SSH credentials.

Connection Method - ansible_connection

OS of the connected host - ansible_network_os

Next we define host IPs of each site switch, site code, switch host name and IP have been added for easy identification.

    ABC:
      hosts:
        SITE - ABC | SW - ABC-SW1 | IP - 10.10.10.1:
          ansible_host: '10.10.10.1'
        SITE - ABC | SW - ABC-SW2 | IP - 10.10.10.2:
          ansible_host: '10.10.10.2'

If you only have 10-20 switches, yes you can easily create this inventory in YAML format. But what about if you have 250+ switches? or 1000+? this is not practical. There is an easy way to populate your inventory YAML file using another simple ansible script. Try out this blog post. It simply explain how you can convert your infrastructure inventory into YAML.

Then…

Now inventory has been populated for all sites with around 250+ switches. Let’s start understanding the part of Ansible playbook for real upgrade. Before that here is my folder structure, this is my upgrade folder. 

There is a sub-folder called “vars”. It has set of YAML files renamed by each switch model. In our case, there are around 15+ different models and there is a different YAML file in vars folder for each model. Content of each YAML file will be explained in next steps.

Next… upgrade script…

Now, let’s start the upgrade script - again written in Ansible.

1st Step - Create Backup Folder

These tasks will be run on localhost.

Task 01 - Get ansible date/time facts

- hosts: localhost

  tasks:
    - name: Get ansible date/time facts
      setup:
        filter: "ansible_date_time"
        gather_subset: "!all"

Here we collect date and time information from localhost. In next step, we save it as a variable.

Task 02 - Store DTG as fact

    - name: Store DTG as fact
      set_fact:
        DTG: "{{ ansible_date_time.date }}"

Task 02 - Store DTG as fact

Next we create a folder to backup the switch configurations. Folder will be named form current date.

    - name: Create a config backup folder
      file:
        path: /mnt/c/Users/Public/ios-upgrade-automation/backups/{{hostvars.localhost.DTG}}
        state: directory
      run_once: true

2nd Step - Configuration Backup

- name: Main Play for Cisco IOS Upgrade
  hosts: # Enter Site Code Here
  serial: 1 
  gather_facts: false
  connection: local

  vars_files:    
    - ["vars/{{ ansible_net_model }}.yml"]

These tasks will be run on each cisco switch. There are few key parameters that should be properly defined.

hosts:

Here site code should be defined. As per our inventory, it can be ABC, DEF or GHI. Script will only run in entered site.

serial:

This is used to serialize the code execution. If this is removed, script will be run on all switches in the site in parallels. If we define 1 here, automation will run on single switch first. Once it is successfully completed, it will run on next switch in the inventory.

In the same way, if we enter 2 or 3 here, it will run on 2 or 3 switches first, once all completed it will move to next 2 or 3.

  vars_files:    
    - ["vars/{{ ansible_net_model }}.yml"]

As mentioned, there are few different models in the infrastructure. IOS file, upgrade version and MD5 value of IOS file vary depend on switch model.

Here we get model specific variable values from the YAML files stored in “vars” folder mentioned previously. Content of each YAML file is as below, for different models, these values will vary.

---
ios_version: 15.XXXX
ios_file: c2960s-universalk9XXXXX.bin
ios_md5: ea604d030b378b6c5c3xxxxxxxxxxx
ios_size: 20000

Task 02 - Store DTG as fact.

Next set of tasks will run inside a task block. 

First subtask will collect cisco ios facts. 

In next step, it backup switch running configuration using ansible module and register it to a varible called ‘config’ - ios_command

Then the registered output is saved to a file in previously configured backup folder. File is named using IP address of the switch and date.

 

  tasks:

# Collect IOS facts
    
    - name: Collect IOS facts
      ios_facts:

# Backup switch running config

    - name: Backup switch running config  
      ios_command:
        commands: show run  
      register: config

    - name: Save output to /mnt/c/Users/Public/ios-upgrade-automation/backups/
      copy:
        content: "{{config.stdout[0]}}"
        dest: "/mnt/c/Users/Public/ios-upgrade-automation/backups/{{hostvars.localhost.DTG}}/{{ ansible_host }}-{{hostvars.localhost.DTG}}-runningconfig.txt"
      tags:
        - runconfig

Task 02 - Store DTG as fact

Next it saves startup config using the same steps.

Once it is done, config will be saved again using ansible module - ios_config

    - name: Backup switch startup config  
      ios_command:
        commands: show start  
      register: config

    - name: Save output to /mnt/c/Users/Public/ios-upgrade-automation/backups/
      copy:
        content: "{{config.stdout[0]}}"
        dest: "/mnt/c/Users/Public/ios-upgrade-automation/backups/{{hostvars.localhost.DTG}}/{{ ansible_host }}-{{hostvars.localhost.DTG}}-startupconfig.txt"
      tags:
        - startconfig

# Save the running config 

    - name: Save running config 
      ios_config:
        save_when: always
      vars:
        ansible_command_timeout: 120

Next we come to our first verification point, boot variable is checked using 'show boot | i BOOT' command and registered the output to a variable called “bootvar”. We later use this output to exclude switches where boot variable is already set to new ios file.

Task 02 - Store DTG as fact

    - name: Check boot path
      ios_command:
        commands: 'show boot | i BOOT'
      register: bootvar
      tags:
        - bootvar

X-XXX-01-SW1#show boot | i BOOT
BOOT path-list      : flash:cxxxx-universalk9-mz.xxxx.xx.bin;flash2:cxxxx-universalk9-mz.xxxx.xx.bin

Next we start main upgrade block which checks free space, copy ios, set boot variable and upgrade. In the first part, output of "show file systems | include flash" is saved to a variable called flash_values. This also will be used in next steps of the code to make some logics.

    - name: Main Block - Copy/Verify/Reboot when switch IOS version is non-compliant
      block:
      - name: Check availability of flashs
        ios_command:
          commands: "show file systems | include flash"
        register: flash_values

X-XXX-01-SW1#show file systems | include flash
*    122185728      52075008         flash     rw  flash: flash1:
     122185728      46614528         flash     rw  flash2:
     122185728      46624768         flash     rw  flash3:

In our enviroment, we have single switches, 2 switch stacks, 3 switch stacks and 4 switch stacks. Before we copy the IOS to each available flash, it should be verified that IOS is not already copied there. To assure that, first we run the command 'show flash: | include {{ ios_file }}' and register the output to a variable. There are 4 tasks here. Each task checks a specific flash. First one checks flash:, next flash1:, then flash2: …etc.

      - name: Check if IOS is already present on the flash
        ios_command:
          commands: 'show flash: | include {{ ios_file }}'
        register: ios_in_flash0
        when:   
          - '"{{ ios_file }}" not in bootvar.stdout[0]'
          - '"flash:"  in flash_values.stdout[0]'                  
        tags:
          - availabilityofios

      - name: Check if IOS is already present on the flash1
        ios_command:
          commands: 'show flash1: | include {{ ios_file }}'
        register: ios_in_flash1
        when:   
          - '"{{ ios_file }}" not in bootvar.stdout[0]'  
          - '"flash1:"  in flash_values.stdout[0]'                              
        tags:
          - availabilityofios

      - name: Check if IOS is already present on the flash2
        ios_command:
          commands: 'show flash2: | include {{ ios_file }}'
        register: ios_in_flash2
        when:   
          - '"{{ ios_file }}" not in bootvar.stdout[0]' 
          - '"flash2:"  in flash_values.stdout[0]'                                
        tags:
          - availabilityofios

      - name: Check if IOS is already present on the flash3
        ios_command:
          commands: 'show flash3: | include {{ ios_file }}'
        register: ios_in_flash3
        when:   
          - '"{{ ios_file }}" not in bootvar.stdout[0]'
          - '"flash3:"  in flash_values.stdout[0]'                              
        tags:
          - availabilityofios

      - name: Check if IOS is already present on the flash4
        ios_command:
          commands: 'show flash4: | include {{ ios_file }}'
        register: ios_in_flash4
        when:   
          - '"{{ ios_file }}" not in bootvar.stdout[0]' 
          - '"flash4:"  in flash_values.stdout[0]'                              
        tags:
          - availabilityofios

There is a when condition added under each task. Each task will only be executed when the conditions under when is satisfied.

        when:   
          - '"{{ ios_file }}" not in bootvar.stdout[0]'
          - '"flash:"  in flash_values.stdout[0]'

In a previous step, we registered the out put of 'show boot | i BOOT' to a variable.

'"{{ ios_file }}" not in bootvar.stdout[0]' - This condition make sure that task is run only when new ios file name is not in the output of above command. Therefore, if bootvar is already set to new IOS, this task will be skipped as it is unneccessary.

'"flash:"  in flash_values.stdout[0]' - This condition make sure that task is run only when “flash:” word appear in the out put of 'show flash: | include {{ ios_file }}' command. Therefore task is run only on available flashes. 

Next we start checking free space to copy the IOS,

      - name: Delete the current IOS if current space is not sufficient 
        cli_command: 
          command: "delete /force {{ ansible_net_image }}"
        when:   
          - ansible_net_filesystems_info['flash:']['spacefree_kb'] < {{ ios_size }}
          - '"{{ ios_file }}" not in ios_in_flash0.stdout[0]'

      - name: Assert that there is enough flash space for upload
        assert:
          that:
            - ansible_net_filesystems_info['flash:']['spacefree_kb'] > {{ ios_size }}
          msg: "There is not enough space left on the device's flash...stopping the upgrade!!!"
        when:
          - '"{{ ios_file }}" not in bootvar.stdout[0]' 
          - '"{{ ios_file }}" not in ios_in_flash0.stdout[0]'
        tags:
          - enoughflashspace   

In the first task, we delete the exisiting IOS if there is no sufficinet space to copy the IOS file. Here we use a parameter collected from ios facts collections - ansible_net_filesystems_info['flash:']['spacefree_kb'] - to compare it against ios_size. When condition assure that IOS is deleted only when sufficient space is not there.

Next, ansible moduele assert is used to assure that there is enough space to copy the new IOS. If this is not assured, scipt will break and upgrade will stop on current host and move to the next host.

Once space check is successfully completed, it starts copying IOS to all available flashes using FTP. 

      - name: Copy new IOS to target device flash - This could take up to 4 minutes 
        block:
        - name: Copy new IOS to target device flash - This could take up to 4 minutes 
          ios_command: 
            commands: 
              - command: copy ftp://10.X.X.X/{{ ios_file }} flash:/{{ ios_file }}
                prompt: '{{ ios_file }}'
                answer: "\r"
          vars:
              ansible_command_timeout: 3600
          when:
            - '"{{ ios_file }}" not in ios_in_flash0.stdout[0]'  
        when:
          - '"flash:"  in flash_values.stdout[0]'                                             

      - name: Copy new IOS to target device flash1 - This could take up to 4 minutes 
        block:
        - name: Copy new IOS to target device flash1 - This could take up to 4 minutes 
          ios_command: 
            commands: 
              - command: copy ftp://10.X.X.X/{{ ios_file }} flash1:/{{ ios_file }}
                prompt: '{{ ios_file }}'
                answer: "\r"
          vars:
              ansible_command_timeout: 3600
          when:
            - '"{{ ios_file }}" not in ios_in_flash1.stdout[0]'  
        when:
          - '"flash1:"  in flash_values.stdout[0]'                                             

      - name: Copy new IOS to target device flash2 - This could take up to 4 minutes 
        block:
        - name: Copy new IOS to target device flash2 - This could take up to 4 minutes 
          ios_command: 
            commands: 
              - command: copy ftp://10.X.X.X/{{ ios_file }} flash2:/{{ ios_file }}
                prompt: '{{ ios_file }}'
                answer: "\r"
          vars:
              ansible_command_timeout: 3600
          when:
            - '"{{ ios_file }}" not in ios_in_flash2.stdout[0]'  
        when:
          - '"flash2:"  in flash_values.stdout[0]'                                             

      - name: Copy new IOS to target device flash3 - This could take up to 4 minutes 
        block:
        - name: Copy new IOS to target device flash3 - This could take up to 4 minutes 
          ios_command: 
            commands: 
              - command: copy ftp://10.X.X.X/{{ ios_file }} flash3:/{{ ios_file }}
                prompt: '{{ ios_file }}'
                answer: "\r"
          vars:
              ansible_command_timeout: 3600
          when:
            - '"{{ ios_file }}" not in ios_in_flash3.stdout[0]'  
        when:
          - '"flash3:"  in flash_values.stdout[0]'                                             

      - name: Copy new IOS to target device flash4 - This could take up to 4 minutes 
        block:
        - name: Copy new IOS to target device flash4 - This could take up to 4 minutes 
          ios_command: 
            commands: 
              - command: copy ftp://10.X.X.X/{{ ios_file }} flash4:/{{ ios_file }}
                prompt: '{{ ios_file }}'
                answer: "\r"
          vars:
              ansible_command_timeout: 3600
          when:
            - '"{{ ios_file }}" not in ios_in_flash4.stdout[0]'  
        when:
          - '"flash4:"  in flash_values.stdout[0]'                                              

Once copying is completed, we again run the previously excuted 'show flash: | include {{ ios_file }}' command on each available flash. Then use ansible assure module to assure that new IOS is now available in each flash.

      - name: Check if IOS is now present on the flash
        ios_command:
          commands: 'show flash: | include {{ ios_file }}'
        register: ios_in_flash0
        when:   
          - '"{{ ios_file }}" not in bootvar.stdout[0]'
          - '"flash:"  in flash_values.stdout[0]'                  
        tags:
          - availabilityofios

      - name: Check if IOS is now present on the flash1
        ios_command:
          commands: 'show flash1: | include {{ ios_file }}'
        register: ios_in_flash1
        when:   
          - '"{{ ios_file }}" not in bootvar.stdout[0]'  
          - '"flash1:"  in flash_values.stdout[0]'                              
        tags:
          - availabilityofios

      - name: Check if IOS is now present on the flash2
        ios_command:
          commands: 'show flash2: | include {{ ios_file }}'
        register: ios_in_flash2
        when:   
          - '"{{ ios_file }}" not in bootvar.stdout[0]' 
          - '"flash2:"  in flash_values.stdout[0]'                                
        tags:
          - availabilityofios

      - name: Check if IOS is now present on the flash3
        ios_command:
          commands: 'show flash3: | include {{ ios_file }}'
        register: ios_in_flash3
        when:   
          - '"{{ ios_file }}" not in bootvar.stdout[0]'
          - '"flash3:"  in flash_values.stdout[0]'                              
        tags:
          - availabilityofios

      - name: Check if IOS is now present on the flash4
        ios_command:
          commands: 'show flash4: | include {{ ios_file }}'
        register: ios_in_flash4
        when:   
          - '"{{ ios_file }}" not in bootvar.stdout[0]'
          - '"flash4:"  in flash_values.stdout[0]'                              
        tags:
          - availabilityofios

# Assert that the new IOS is now available in flash

      - name: Assert that the new IOS is now available in flash
        assert:
          that:
            - '"{{ ios_file }}" in ios_in_flash0.stdout[0]'
          msg: "New ios is not available in flash...stopping the upgrade!!!"
        when:   
          - '"flash:"  in flash_values.stdout[0]'             
          - '"{{ ios_file }}" not in bootvar.stdout[0]'

      - name: Assert that the new IOS is now available in flash1
        assert:
          that:
            - '"{{ ios_file }}" in ios_in_flash1.stdout[0]'
          msg: "New ios is not available in flash1...stopping the upgrade!!!"
        when:   
          - '"flash1:"  in flash_values.stdout[0]'            
          - '"{{ ios_file }}" not in bootvar.stdout[0]'

      - name: Assert that the new IOS is now available in flash2
        assert:
          that:
            - '"{{ ios_file }}" in ios_in_flash2.stdout[0]'
          msg: "New ios is not available in flash2...stopping the upgrade!!!"
        when:   
          - '"flash2:"  in flash_values.stdout[0]'
          - '"{{ ios_file }}" not in bootvar.stdout[0]'

      - name: Assert that the new IOS is now available in flash3
        assert:
          that:
            - '"{{ ios_file }}" in ios_in_flash3.stdout[0]'
          msg: "New ios is not available in flash3...stopping the upgrade!!!"            
        when:   
          - '"flash3:"  in flash_values.stdout[0]'
          - '"{{ ios_file }}" not in bootvar.stdout[0]'

      - name: Assert that the new IOS is now available in flash4
        assert:
          that:
            - '"{{ ios_file }}" in ios_in_flash4.stdout[0]'
          msg: "New ios is not available in flash3...stopping the upgrade!!!"            
        when:   
          - '"flash4:"  in flash_values.stdout[0]'
          - '"{{ ios_file }}" not in bootvar.stdout[0]'


Next we move to the MD5 verification. First ios_command module is used to run "verify /md5 flash:{{ ios_file }}" command on each available flash and register the output to a different varable.

      - name: Check MD5 value of copied IOS in flash
        ios_command:
          commands: "verify /md5 flash:{{ ios_file }}"
        vars:
          ansible_command_timeout: 3600
        register: var_ios_md5_final0 
        when:
          - '"flash:"  in flash_values.stdout[0]'
          - '"{{ ios_file }}" not in bootvar.stdout[0]'

      - name: Check MD5 value of copied IOS in flash1
        ios_command:
          commands: "verify /md5 flash1:{{ ios_file }}"
        vars:
          ansible_command_timeout: 3600
        register: var_ios_md5_final1 
        when:
          - '"flash1:"  in flash_values.stdout[0]'
          - '"{{ ios_file }}" not in bootvar.stdout[0]'

      - name: Check MD5 value of copied IOS in flash2
        ios_command:
          commands: "verify /md5 flash2:{{ ios_file }}"
        vars:
          ansible_command_timeout: 3600
        register: var_ios_md5_final2 
        when:
          - '"flash2:"  in flash_values.stdout[0]'
          - '"{{ ios_file }}" not in bootvar.stdout[0]'

      - name: Check MD5 value of copied IOS in flash3
        ios_command:
          commands: "verify /md5 flash3:{{ ios_file }}"
        vars:
          ansible_command_timeout: 3600
        register: var_ios_md5_final3 
        when:
          - '"flash3:"  in flash_values.stdout[0]'
          - '"{{ ios_file }}" not in bootvar.stdout[0]'

      - name: Check MD5 value of copied IOS in flash4
        ios_command:
          commands: "verify /md5 flash4:{{ ios_file }}"
        vars:
          ansible_command_timeout: 3600
        register: var_ios_md5_final4 
        when:
          - '"flash4:"  in flash_values.stdout[0]'
          - '"{{ ios_file }}" not in bootvar.stdout[0]'


In next five steps, regitered MD5 values are compared against MD5 values mentioned in the variable file. If the MD5 value is not matching, script break and upgrade fail with "MD5 is not matching with original...stopping the upgrade!!!" error messege.


      - name: Verify MD5 is matching with original IOS - flash
        assert:
          that:
            - {{ ios_md5 }} == var_ios_md5_final0.stdout[0].split(' = ')[1]
          msg: "MD5 is not matching with original...stopping the upgrade!!!"
        when:
          - '"flash:"  in flash_values.stdout[0]'
          - '"{{ ios_file }}" not in bootvar.stdout[0]'

      - name: Verify MD5 is matching with original IOS - flash1
        assert:
          that:
            - {{ ios_md5 }} == var_ios_md5_final1.stdout[0].split(' = ')[1]
          msg: "MD5 is not matching with original...stopping the upgrade!!!"
        when:
          - '"flash1:"  in flash_values.stdout[0]'
          - '"{{ ios_file }}" not in bootvar.stdout[0]'

      - name: Verify MD5 is matching with original IOS - flash2
        assert:
          that:
            - {{ ios_md5 }} == var_ios_md5_final2.stdout[0].split(' = ')[1]
          msg: "MD5 is not matching with original...stopping the upgrade!!!"
        when:
          - '"flash2:"  in flash_values.stdout[0]'
          - '"{{ ios_file }}" not in bootvar.stdout[0]'

      - name: Verify MD5 is matching with original IOS - flash3
        assert:
          that:
            - {{ ios_md5 }} == var_ios_md5_final3.stdout[0].split(' = ')[1]
          msg: "MD5 is not matching with original...stopping the upgrade!!!"            
        when:
          - '"flash3:"  in flash_values.stdout[0]'
          - '"{{ ios_file }}" not in bootvar.stdout[0]'

      - name: Verify MD5 is matching with original IOS - flash3
        assert:
          that:
            - {{ ios_md5 }} == var_ios_md5_final4.stdout[0].split(' = ')[1]
          msg: "MD5 is not matching with original...stopping the upgrade!!!" 
        when:
          - '"flash4:"  in flash_values.stdout[0]' 
          - '"{{ ios_file }}" not in bootvar.stdout[0]'

Once all MD5 verifications are passed, switch boot variable is set to new bin file.

      - name: Change boot variable to new image 
        ios_config: 
          commands: 
            - "boot system flash:{{ ios_file }}"
          save_when: always 
        vars:
          ansible_command_timeout: 120
        when: 
          - ansible_net_version != "{{ ios_version }}"  
          - '"{{ ios_file }}" not in bootvar.stdout[0]'

Once it is configured, and saved, following steps are run to assure that we have accurately changed the bootvariable.

      - name: Check Boot path
        ios_command:
          commands: 'show boot | i BOOT'
        register: bootvar
        when: 
          - ansible_net_version != "{{ ios_version }}"  
        tags:
          - bootvar

# Verify the boot path after changing

      - name: Assert that the boot path is set to the new IOS
        assert:
          that:
            - '"{{ ios_file }}" in bootvar.stdout[0]'
          msg: "Boot path is not set to the new image...stopping the upgrade!!!"
        when: 
          - ansible_net_version != "{{ ios_version }}"          
        tags:
          - bootvar

Then we reboot the switch. We delegate localhost to wait and check until switch comes up again - it usually takes around 5-7 mins.

      - name: Reload the Device 
        cli_command: 
          command: reload
          prompt: 
            - confirm
          answer: 
            - 'y'
        when: 
          - ansible_net_version != "{{ ios_version }}" 

      - name: Reset connection to the switch
        meta: reset_connection

      - name: Wait for device to come back online
        wait_for:
          host: "{{ ansible_host }}"
          port: 22
          delay: 240
          timeout: 1800
        delegate_to: localhost
        connection: local

All done!!! Finally once switch is back live, again ios_facts are collected and verify the new version is expected version. For the documentation purpose, a debug output has been added to the script the print the details of upgraded switch on the console.

      - name: Collect IOS facts
        ios_facts:

      - name: Check image version after the reboot    
        ios_facts:

      - name: Assure that version is correct     
        assert:
          that:
            - ios_version == ansible_net_version

      - debug: 
          msg: 
          - "SOFTWARE UPGRADE IS SUCCESSFULLY COMPLETED!!!"
          - "HOSTNAME      - {{ ansible_net_hostname }}"
          - "IP ADDRESS    - {{ ansible_net_all_ipv4_addresses }}"
          - "MODEL         - {{ ansible_net_model }}"
          - "SERIAL NUMBER - {{ ansible_net_stacked_serialnums }}"
          - "IOS VERSION   - {{ ansible_net_version }}"
          - "IMAGE USED    - {{ ansible_net_image }}"

 

Related Documents

Working with Route Tables in AWS Transit Gateway
