# Encrypted USB Drive Backup with Ansible, and a tale of Pets and Cattle

![](mazinger-z.png)

I like to make backups. Yes, there is nothing wrong with me, I think having a failsafe before doing something that cannot be rolled back easily, like installing a new version of your favorite operating system, should be compulsory.

Also, *I don't like to repeat myself*. For that reason I like to write scripts so a successful experience can be repeated, like the following one:

```shell
#!/bin/bash
# Abort after the first error!
set -e
# Partition the disk, single partition on /dev/sda1 (type Linux)
sudo fdisk /dev/sda
sudo partprobe /dev/sda
# Encrypt the partition, https://www.redhat.com/sysadmin/disk-encryption-luks
sudo cryptsetup luksFormat /dev/sda1
sudo cryptsetup luksOpen /dev/sda1 usbluks
# Create a logical volume on the encrypted disk
sudo vgcreate usbluks_vg /dev/mapper/usbluks
sudo lvcreate -n usbluks_logvol -L 118G+ usbluks_vg
# Format the disk with btrfs, https://btrfs.readthedocs.io/en/latest/mkfs.btrfs.html
sudo mkfs.btrfs --mixed --data SINGLE --metadata SINGLE /dev/usbluks_vg/usbluks_logvol
# Mount our new volume
sudo mount /dev/usbluks_vg/usbluks_logvol /mnt/
# Backup my files
sudo tar --create --directory /home --file -| tar --directory /mnt --extract --file -
```

Not so bad, with bare-minimum error handling. Bash is probably the first choice as is simple to use yet expressive. But we also know that Bash stops being the ideal choice when you want to do something like this as compared with other tool that I really like, Ansible:

1. Provide [more complete debugging](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_debugger.html). I'm not just talking about ```printf``` statements but also more rich [error handling tools](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_error_handling.html), providing a detailed [stacktrace](https://en.wikipedia.org/wiki/Stack_trace).
2. [idempotence](https://www.ansible.com/blog/whats-new-in-the-ansible-content-collection-for-kubernetes-1.2), the ability to group your operations in [blocks](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_blocks.html).
3. Control how your script runs: [tags](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_tags.html), limit where it runs, do a dry run
4. Code smell detection tools. Bash [has great tools](https://www.shellcheck.net/) out there, but Ansible has [ansible-lint](https://www.redhat.com/sysadmin/ansible-lint-YAML), [check](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_checkmode.html) flag.
5. Remote execution: While my script doesn't need it if I ever decide to run this on a [different group of machines](https://docs.ansible.com/ansible/latest/inventory_guide/intro_inventory.html) I know I only need to make a few changes, [support for remote task execution](https://docs.ansible.com/ansible/latest/inventory_guide/connection_details.html) is unmatched.

All these capabilities made me re-think the script above and realize than an improved version would be probably too verbose and more difficult to maintain than one written in Ansible.

Before we get started, let's talk a little about _Pet versus Cattle servers_:

## The difference between Pet versus Cattle servers

There is an ongoing debate about [how you should treat your servers](https://www.markhneedham.com/blog/2013/04/07/treating-servers-as-cattle-not-as-pets/) (either as Pet or Cattle). There are many good analogies and [explanations](http://cloudscaling.com/blog/cloud-computing/the-history-of-pets-vs-cattle/) out there, but the following summarises pretty good this way of thinking:

**Pets**
> Servers or server pairs that are treated as indispensable or unique systems that can never be down. Typically, they are manually built, managed, and “hand fed”. Examples include mainframes, solitary servers, HA loadbalancers/firewalls (active/active or active/passive), database systems designed as master/slave (active/passive), and so on.

_Bash, Python_ are perfect tools to automate tasks on Pet servers as they are very expressive, and you are not very concerned about uniformity. This doesn't
t mean you cannot have an Ansible playbook for such special server, in fact it may be a very good reason to have a dedicated playbook for it/ them.

**Cattle**
> Arrays of more than two servers, that are built using automated tools, and are designed for failure, where no one, two, or even three servers are irreplaceable. Typically, during failure events no human intervention is required as the array exhibits attributes of “routing around failures” by restarting failed servers or replicating data through strategies like triple replication or erasure coding. Examples include web server arrays, multi-master datastores such as Cassandra clusters, multiple racks of gear put together in clusters, and just about anything that is load-balanced and multi-master.

_Ansible and other specialized provisioning tools_ ensure than the state of the servers is uniform, and doesn't change by accident.

_It is Ansible then a bad tool to handle pet servers_?. It is actually quite good. I want to show you on this article how you can also use an Ansible playbook to make an encrypted backup of your home directory, so you can keep your workstation, properly backed up.

### Requirements
* SUDO (to run programs that require elevated privileges like creating a filesystem, mounting a disk)
* Python 3 and PIP
* Ansible 2
* A USB drive (don't worry if it comes formatted, we will remove all the files inside)
* Curiosity

Also, will show you a few changes you need to make as Ansible is normally suited to deal with cattle servers, for pets we need to make a few changes to the playbook.

## Installation

If you don't have Ansible installed you can do something like this:

```shell
python3 -m venv ~/virtualenv/ansible
. ~/virtualenv/ansible/bin/activate
python -m pip install --upgrade pip
pip install -r requirements.txt
```

And if the ```community.general``` Galaxy module is not present:

```shell
ansible-galaxy collection list 2>&1|rg general # Comes empty
ansible-galaxy collection install community.general
```

## How does the Ansible playbook look like?

You probably have seen Ansible playbooks before, but this one has a few special features:

1. It doesn't use SSH to connect to a remote host. The 'remote' host is localhost (the same machine where the playbook runs), so we use a special time of connection called 'local'
2. It uses special fact gathering to speed up the device feature detection. Will elaborate more in the next section.
3. Define variables to make the playbook re-usable, but we need to prompt the user for the values as opposed to defined them on the command line. Pet servers have unique features, so it is better to ask the user for some choices interactively as the playbook runs.
4. Use tags. If we want to run just parts of this playbook we can skip to the desired target (```ansible-playbook --tag $mytag encrypted_us_backup.yaml```)

### Getting your facts straight

Every time you run an Ansible playbook, it collects facts about your target system, so it can operate properly. But for our task it is overkill, and we only need information about the devices, while we can ignore other details like dns, python to mention a few.

First disable the general 'gather_facts', then override with a task below using the setup module, just enabling devices and mounts.

After that you can use the facts in any way you see fit (check [fact_filtering.yaml](fact_filtering.yaml) to see how is done):

```yaml
---
- name: Restricted fact gathering example
  hosts: localhost
  connection: local
  become: true
  gather_facts: false
  vars_prompt:
    - name: device
      prompt: "Enter name of the USB device"
      private: false
      default: "sda"
  vars:
    target_device: "/dev/{{ device }}"
  tasks:
    - name: Get only 'devices, mount' facts
      ansible.builtin.setup:
        gather_subset:
          - '!all'
          - '!min'
          - devices
          - mounts
    - name: Basic setup and verification for target system
      block:
        - name: Facts for {{ target_device }}
          community.general.parted:
            device: "{{ target_device }}"
            state: "info"
            unit: "GB"
          register: target_parted_data
        - name: Calculate disk size
          debug:
            msg: "{{ ansible_devices[device] }}"
        - name: Calculate available space on USB device and save it as a fact
          ansible.builtin.set_fact:
            total_usb_disk_space: "{{ (ansible_devices[device]['sectorsize']| int) * (ansible_devices[device]['sectors']|int) }}"
            cacheable: yes
          when: target_parted_data is defined
        - name: Print facts for {{ target_device }}
          ansible.builtin.debug:
            msg: "{{ ansible_devices[device].size }}, {{ total_usb_disk_space }} bytes"
          when: target_parted_data is defined
```

See it in action:

[![asciicast](https://asciinema.org/a/553183.svg)](https://asciinema.org/a/553183)

Now that we know the size of the disk, we should also get how much disk our backup will take. Ansible doesn't have a du task, so we wrap our own.

### How much disk space we will have to back up

We do a few things here:

1. Capture the [output of the du command](disk_usage_dir.yaml), and [report if it changes](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_error_handling.html#defining-changed) only the return code
2. Filter the output of the [du](https://www.man7.org/linux/man-pages/man1/du.1.html) command, so we can use it later.

```yaml
---
- name: Disk utilization capture
  hosts: localhost
  connection: local
  become: true
  gather_facts: false
  vars_prompt:
    - name: source_dir
      prompt: "Enter name of the directory to back up"
      private: false
      default: "/home/josevnz"
  tasks:
    - name: Capture disk utilization on {{ source_dir }}
      block:
        - name: Get disk utilization from {{ source_dir }}
          ansible.builtin.command:
            argv:
              - /bin/du
              - --max-depth=0
              - --block-size=1
              - --exclude='*/.cache'
              - --exclude='*/.gradle/caches'
              - --exclude='*/Downloads'
              - "{{ source_dir }}"
          register: du_capture
          changed_when: "du_capture.rc != 0"
        - name: Process DU output
          ansible.builtin.set_fact:
            du: "{{ du_capture.stdout | regex_replace('\\D+') | int }}"
        - name: Print facts for {{ target_device }}
          ansible.builtin.debug:
            msg: "{{ source_dir }} -> {{ du }} bytes"
          when: du_capture is defined
```

Let's see it running:

[![asciicast](https://asciinema.org/a/553200.svg)](https://asciinema.org/a/553200)

With this information we can test beforehand if the USB drive has enough space to save our files.

```yaml
        - name: Check if destination USB drive has enough space to store our backup
          ansible.builtin.assert:
            that:
              - ( total_usb_disk_space | int ) > ( du | int)
            fail_msg: "Not enough disk space on USB drive! {{ du | int | human_readable() }} > {{ total_usb_disk_space | int | human_readable() }}"
            success_msg: "We have enough space to make the backup!"
          tags: disk_space_check
```

If the destination is too small we can get abort the whole operation:

[![asciicast](https://asciinema.org/a/553214.svg)](https://asciinema.org/a/553214)

So we are good to go, right? Well, we live on an imperfect world where [counterfit USB drives are sold](https://datarecovery.com/2022/03/the-2tb-flash-drive-scam-why-high-capacity-flash-drives-are-fakes/), they do tricks to advertise higher capacity that is really supported.

Using [Open Source tool f3](https://fight-flash-fraud.readthedocs.io/en/latest/introduction.html) we can run with brand-new media to make sure the capacity is indeed what we think we purchased, will cover that next

### Trust but verify (than the disk is indeed legitimate)

Or [puting it in another way](https://en.wikipedia.org/wiki/Trust,_but_verify):

> To test men and verify what has actually been done— this, this again this alone is now the main feature of all our activities, of our whole policy

Depending on the size of the disk this task can be quick or take a long time, for this part

```yaml
  tasks:
    - name: Verify with f3 {{ device }}
      block:
        - name: Testing with f3 {{ device }}
          ansible.builtin.command:
            argv:
              - /usr/bin/f3probe
              - "{{ target_device }}"
          register: f3_capture
          changed_when: "f3_capture.rc != 0"
          when: paranoid is defined and paranoid == "y"
        - name: Print facts for {{ target_device }}
          ansible.builtin.debug:
            msg: "{{ target_device }} -> {{ f3_capture }}"
          when: f3_capture is defined
```

I don't want to force this task every time I run my encrypted backup with media I know is good, so unless is enabled on the prompt it will not run.

How does it look ([running_f3.yaml](running_f3.yaml))?:

```shell
[josevnz@dmaf5 EncryptedUsbBackup]$ ansible-playbook running_f3.yaml
[WARNING]: No inventory was parsed, only implicit localhost is available
[WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost does not match 'all'
Enter name of the USB device [sda]: 
Check the USB drive real capacity with f3 (y/n) [n]: y

PLAY [USB drive verification] ***************************************************************************************************************************************************************

TASK [Testing with f3 sda] ******************************************************************************************************************************************************************
ok: [localhost]

TASK [Print facts for /dev/sda] *************************************************************************************************************************************************************
ok: [localhost] => {
    "msg": "/dev/sda -> {'changed': False, 'stdout': 'F3 probe 8.0\\nCopyright (C) 2010 Digirati Internet LTDA.\\nThis is free software; see the source for copying conditions.\\n\\nWARNING: Probing normally takes from a few seconds to 15 minutes, but\\n         it can take longer. Please be patient.\\n\\nProbe finished, recovering blocks... Done\\n\\nGood news: The device `/dev/sda\\' is the real thing\\n\\nDevice geometry:\\n\\t         *Usable* size: 960.00 MB (1966080 blocks)\\n\\t        Announced size: 960.00 MB (1966080 blocks)\\n\\t                Module: 1.00 GB (2^30 Bytes)\\n\\tApproximate cache size: 0.00 Byte (0 blocks), need-reset=no\\n\\t   Physical block size: 512.00 Byte (2^9 Bytes)\\n\\nProbe time: 4\\'57\"', 'stderr': '', 'rc': 0, 'cmd': ['/usr/bin/f3probe', '/dev/sda'], 'start': '2023-01-20 15:57:24.886985', 'end': '2023-01-20 16:04:10.578318', 'delta': '0:06:45.691333', 'msg': '', 'stdout_lines': ['F3 probe 8.0', 'Copyright (C) 2010 Digirati Internet LTDA.', 'This is free software; see the source for copying conditions.', '', 'WARNING: Probing normally takes from a few seconds to 15 minutes, but', '         it can take longer. Please be patient.', '', 'Probe finished, recovering blocks... Done', '', \"Good news: The device `/dev/sda' is the real thing\", '', 'Device geometry:', '\\t         *Usable* size: 960.00 MB (1966080 blocks)', '\\t        Announced size: 960.00 MB (1966080 blocks)', '\\t                Module: 1.00 GB (2^30 Bytes)', '\\tApproximate cache size: 0.00 Byte (0 blocks), need-reset=no', '\\t   Physical block size: 512.00 Byte (2^9 Bytes)', '', 'Probe time: 4\\'57\"'], 'stderr_lines': [], 'failed': False}"
}

PLAY RECAP **********************************************************************************************************************************************************************************
localhost                  : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

Took 5+ minutes on a small but slow USB drive.

### Putting everything together, final playbook

Here is how we can implement these requirements, first capture user choices interactively:

```yaml
---
- name: Take an empty USB disk and copy user home directory into an encrypted partition
  hosts: localhost
  connection: local
  become: true
  gather_facts: false
  any_errors_fatal: true
  vars_prompt:
    - name: source_dir
      prompt: "Enter name of the directory to back up"
      private: false
      default: "/home/josevnz"
    - name: device
      prompt: "Enter name of the USB device"
      private: false
      default: "sda"
    - name: passphrase
      prompt: "Enter the passphrase to be used to protect the target device"
      private: true
    - name: destroy_partition
      prompt: "Destroy any existing partitions (y/n)?"
      default: "y"
      private: false
    - name: cryptns
      prompt: "Name of the luks device. Strongly suggested you pick one"
      default: "{{ query('community.general.random_string', upper=false, numbers=false, special=false, length=4)[0] }}"
      private: false
    - name: paranoid
      prompt: "Check the USB drive real capacity with f3 (y/n)"
      default: "n"
      private: false
  vars:
    volgrp: "{{ cryptns }}_vg"
    logicalvol: "backuplv"
    cryptns: "usbluks"
    cryptdevice: "/dev/mapper/{{ cryptns }}"
    destination_dir: "/mnt"
    target_device: "/dev/{{ device }}"
```

Next step is to define the following actions:

Get [partition details](https://docs.ansible.com/ansible/latest/collections/community/general/parted_module.html), validate devices and destroy the destination if already exists. We use a [block](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_blocks.html) to make to group these operations together (like installing crysetup at the end if missing).
```yaml
  tasks:
    - name: Basic setup and verification for target system
      block:
        - name: Get only 'devices, mount' facts
          ansible.builtin.setup:
            gather_subset:
              - '!all'
              - '!min'
              - devices
              - mounts
          tags: facts_subset
        - name: Facts for {{ target_device }}
          community.general.parted:
            device: "{{ target_device }}"
            state: "info"
            unit: "GB"
          register: target_parted_data
          tags: parted_data
        - name: Calculate available space on USB device and save it as a fact
          ansible.builtin.set_fact:
            total_usb_disk_space: "{{ (ansible_devices[device]['sectorsize'] | int) * (ansible_devices[device]['sectors'] | int) }}"
            cacheable: true
          when: target_parted_data is defined
          tags: space_usb
        - name: Get disk utilization from {{ source_dir }}
          ansible.builtin.command:
            argv:
              - /bin/du
              - --max-depth=0
              - --block-size=1
              - --exclude='*/.cache'
              - --exclude='*/.gradle/caches'
              - --exclude='*/Downloads'
              - "{{ source_dir }}"
          register: du_capture
          changed_when: "du_capture.rc != 0"
          tags: du_capture
        - name: Process DU output
          ansible.builtin.set_fact:
            du: "{{ du_capture.stdout | regex_replace('\\D+') | int }}"
            type: integer
          tags: du_filter
        - name: Check if destination USB drive has enough space to store our backup
          ansible.builtin.assert:
            that:
              - ( total_usb_disk_space | int ) > ( du | int)
            fail_msg: "Not enough disk space on USB drive! {{ du | int | human_readable() }} > {{ total_usb_disk_space | int | human_readable() }}"
            success_msg: "We have enough space to make the backup!"
          tags: disk_space_check
        - name: Facts for {{ target_device }}
          community.general.parted:  # Note this task doesn't use facts
            device: "{{ target_device }}"
            state: "info"
            unit: "GB"
          register: target_parted_data
          tags: parted_info
        - name: Print facts for {{ target_device }}
          ansible.builtin.debug:
            msg: "{{ target_parted_data }}"
          when: target_parted_data is defined
          tags: print_facts
        - name: Unmount USB drive
          ansible.posix.mount:
            path: "{{ destination_dir }}"
            state: absent
          when: target_parted_data is defined
          tags: unmount
        - name: Install f3 (test for fake flash drives and cards)
          ansible.builtin.dnf:
            name: f3
            state: installed
          tags: f3install
        - name: Check if USB device is a fake one with f3
          ansible.builtin.command: "/usr/bin/f3probe {{ target_device }}"
          register: f3_run
          when: target_parted_data is defined and paranoid is defined and paranoid == "y"
          tags: f3_fake
        - name: Destroy existing partition if found, update target_parted_data
          community.general.parted:
            device: "{{ target_device }}"
            number: 1
            state: "absent"
            unit: "GB"
          when: target_parted_data is defined and destroy_partition == "y"
          register: target_parted_data
          tags: destroy_partition
        - name: "Abort with an error If there are still any partitions for {{ target_device }}"
          ansible.builtin.debug:
            msg: -|
              {{ target_device }} is already partitioned, {{ target_parted_data['partitions'] }}.
              Size is {{ target_parted_data['disk']['size'] }} GB
          failed_when: target_parted_data['partitions'] is defined and (target_parted_data['partitions'] | length > 0) and not ansible_check_mode
          tags: fail_on_existing_part
    - name: Get destination device details
```
Create the partition, with some information printout
```yaml
    - name: Create partition
      block:
        - name: Get info on destination partition
          community.general.parted:
            device: "{{ target_device }}"
            number: 1
            state: info
          register: info_output
          tags: parted_info
        - name: Print info_output
          ansible.builtin.debug:
            msg: "{{ info_output }}"
          tags: parted_print
        - name: Create new partition
          community.general.parted:
            device: "{{ target_device }}"
            number: 1
            state: present
            part_end: "100%"
          register: parted_output
          tags: parted_create
      rescue:
        - name: Parted failed
          ansible.builtin.fail:
            msg: 'Parted failed:  {{ parted_output }}'
```
Then create the encrypted volume, format it and then mount it. Another block:

```yaml
    - name: LUKS and filesystem tasks
      block:
        - name: Install crysetup
          ansible.builtin.dnf:
            name: cryptsetup
            state: installed
          tags: crysetup
        - name: Create LUKS container with passphrase
          community.crypto.luks_device:
            device: "{{ target_device }}1"
            state: present
            name: "{{ cryptns }}"
            passphrase: "{{ passphrase }}"
          tags: luks_create
        - name: Open luks container
          community.crypto.luks_device:
            device: "{{ target_device }}1"
            state: opened
            name: "{{ cryptns }}"
            passphrase: "{{ passphrase }}"
          tags: luks_open
        - name: Create {{ cryptdevice }}
          community.general.system.lvg:
            vg: "{{ volgrp }}"
            pvs: "{{ cryptdevice }}"
            force: true
            state: "present"
          when: not ansible_check_mode
          tags: lvm_volgrp
        - name: Create a logvol in my new vg
          community.general.system.lvol:
            vg: "{{ volgrp }}"
            lv: "{{ logicalvol }}"
            size: "+100%FREE"
          when: not ansible_check_mode
          tags: lvm_logvol
        - name: Create a filesystem for the USB drive (man mkfs.btrfs for options explanation)
          community.general.system.filesystem:
            fstype: btrfs
            dev: "/dev/mapper/{{ volgrp }}-{{ logicalvol }}"
            force: true
            opts: --data SINGLE --metadata SINGLE --mixed --label backup --features skinny-metadata,no-holes
          when: not ansible_check_mode
          tags: mkfs
        - name: Mount USB drive (use a dummy fstab to avoid changing real /etc/fstab)
          ansible.posix.mount:
            path: "{{ destination_dir }}"
            src: "/dev/mapper/{{ volgrp }}-{{ logicalvol }}"
            state: mounted
            fstype: btrfs
            fstab: /tmp/tmp.fstab
          when: not ansible_check_mode
          tags: mount
```

Note than _we do not want_ to persist the status of the mounted filesystem across reboots, we will umount the drive as soon we are done with the backup. For that reason we use a temporary fstab (```fstab: /tmp/tmp.fstab```)

Last task is to create the backup on the new encrypted volume using [synchronize](https://docs.ansible.com/ansible/latest/collections/ansible/posix/synchronize_module.html) (frontend for rsync, we pass a few arguments to skip unwanted heavy directories):
```yaml
    - name: Backup stage
      tags: backup
      block:
        - name: Backup using rsync
          ansible.posix.synchronize:
            archive: true
            compress: false
            dest: "{{ destination_dir }}"
            owner: true
            partial: true
            recursive: true
            src: "{{ source_dir }}"
            rsync_opts:
              - "--exclude Downloads"
              - "--exclude .cache"
              - "--exclude .gradle/caches"
```

Now it is time to dry-run this before running it on our USB drive.

## Fixing style issues and mistakes, doing a dry-run

You can test this playbook before running it!. To see if ansible-lint has any complains:

```shell
[josevnz@dmaf5 EncryptedUsbDriveBackup]$ ~/virtualenv/EncryptedUsbDriveBackup/bin/ansible-lint encrypted_usb_backup.yaml 
WARNING  Overriding detected file kind 'yaml' with 'playbook' for given positional argument: encrypted_usb_backup.yaml

Passed with production profile: 0 failure(s), 0 warning(s) on 1 files.
```

Then doing a dry-run:

```shell
ansible-playbook --check encrypted_usb_backup.yaml
```

## Making the backup, full run

Below you can see an example of how your backup session may look like (without the f3 and backup verification):

```shell
[josevnz@dmaf5 EncryptedUsbBackup]$ ansible-playbook encrypted_usb_backup.yaml 
[WARNING]: No inventory was parsed, only implicit localhost is available
[WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost does not match 'all'
Enter name of the directory to back up [/home/josevnz]: /home/josevnz/Documents/Domino/
Enter name of the USB device [sda]: sdc
Enter the passphrase to be used to protect the target device: 
Destroy any existing partitions (y/n)? [y]: 
Name of the luks device. Strongly suggested you pick one [rfjz]: 
Check the USB drive real capacity with f3 (y/n) [n]: 

PLAY [Take an empty USB disk and copy user home directory into an encrypted partition] *****************************************************************************************************************************************************

TASK [Get only 'devices, mount' facts] *****************************************************************************************************************************************************************************************************
ok: [localhost]

TASK [Facts for /dev/sdc] ******************************************************************************************************************************************************************************************************************
ok: [localhost]

TASK [Calculate available space on USB device and save it as a fact] ***********************************************************************************************************************************************************************
ok: [localhost]

TASK [Get disk utilization from /home/josevnz/Documents/Domino/] ***************************************************************************************************************************************************************************
ok: [localhost]

TASK [Process DU output] *******************************************************************************************************************************************************************************************************************
ok: [localhost]

TASK [Check if destination USB drive has enough space to store our backup] *****************************************************************************************************************************************************************
ok: [localhost] => {
    "changed": false,
    "msg": "We have enough space to make the backup!"
}

TASK [Facts for /dev/sdc] ******************************************************************************************************************************************************************************************************************
ok: [localhost]

TASK [Print facts for /dev/sdc] ************************************************************************************************************************************************************************************************************
ok: [localhost] => {
    "msg": {
        "changed": false,
        "disk": {
            "dev": "/dev/sdc",
            "logical_block": 512,
            "model": "General UDisk",
            "physical_block": 512,
            "size": 1.01,
            "table": "msdos",
            "unit": "gb"
        },
        "failed": false,
        "partitions": [
            {
                "begin": 0.0,
                "end": 1.01,
                "flags": [],
                "fstype": "",
                "name": "",
                "num": 1,
                "size": 1.01,
                "unit": "gb"
            }
        ],
        "script": "unit 'GB' print"
    }
}

TASK [Unmount USB drive] *******************************************************************************************************************************************************************************************************************
ok: [localhost]

TASK [Install f3 (test for fake flash drives and cards)] ***********************************************************************************************************************************************************************************
ok: [localhost]

TASK [Check if USB device is a fake one with f3] *******************************************************************************************************************************************************************************************
skipping: [localhost]

TASK [Destroy existing partition if found, update target_parted_data] **********************************************************************************************************************************************************************
changed: [localhost]

TASK [Abort with an error If there are still any partitions for /dev/sdc] ******************************************************************************************************************************************************************
ok: [localhost] => {
    "msg": "-| /dev/sdc is already partitioned, []. Size is 1.01 GB"
}

TASK [Get info on destination partition] ***************************************************************************************************************************************************************************************************
ok: [localhost]

TASK [Print info_output] *******************************************************************************************************************************************************************************************************************
ok: [localhost] => {
    "msg": {
        "changed": false,
        "disk": {
            "dev": "/dev/sdc",
            "logical_block": 512,
            "model": "General UDisk",
            "physical_block": 512,
            "size": 983040.0,
            "table": "msdos",
            "unit": "kib"
        },
        "failed": false,
        "partitions": [],
        "script": "unit 'KiB' print"
    }
}

TASK [Create new partition] ****************************************************************************************************************************************************************************************************************
changed: [localhost]

TASK [Install crysetup] ********************************************************************************************************************************************************************************************************************
ok: [localhost]

TASK [Create LUKS container with passphrase] ***********************************************************************************************************************************************************************************************
ok: [localhost]

TASK [Open luks container] *****************************************************************************************************************************************************************************************************************
changed: [localhost]

TASK [Create /dev/mapper/rfjz] *************************************************************************************************************************************************************************************************************
changed: [localhost]

TASK [Create a logvol in my new vg] ********************************************************************************************************************************************************************************************************
changed: [localhost]

TASK [Create a filesystem for the USB drive (man mkfs.btrfs for options explanation)] ******************************************************************************************************************************************************
changed: [localhost]

TASK [Mount USB drive (use a dummy fstab to avoid changing real /etc/fstab)] ***************************************************************************************************************************************************************
changed: [localhost]

TASK [Backup using rsync] ******************************************************************************************************************************************************************************************************************
changed: [localhost]

TASK [Facts for /dev/sdc] ******************************************************************************************************************************************************************************************************************
skipping: [localhost]

PLAY RECAP *********************************************************************************************************************************************************************************************************************************
localhost                  : ok=19   changed=8    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0   
```

## Next steps

* If you want to use a GUI for your encrypted backup then you should consider [VeraCrypt](https://www.veracrypt.fr/en/Home.html), which is Open Source and also has a nice wizard to let you through the process. There is a [nice tutorial](https://opensource.com/article/21/4/open-source-encryption) how to do that.
* Instead of local use the cloud (but encrypt first!): We used rsync to make a backup to a USB drive, but Ansible can also upload files to [an S3 cloud volume](https://docs.ansible.com/ansible/2.9/modules/s3_sync_module.html). Ideally you should make an archive and encrypt it then with [sops](https://docs.ansible.com/ansible/latest/collections/community/sops/sops_encrypt_module.html) before the upload.
* If you want to back up more than one user directory you could use [a loop](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_loops.html).
* This article borrows heavily from article written by [Peter Gervaise](https://www.redhat.com/sysadmin/encrypt-single-filesystem), you should spend some time reading it.
* It is possible to check if the USB has bad blocks, before or after making the backup. I wrote for you a small ansible playbook you can run to see how the program [badblocks](verify_usb.yaml) works.
* The source code for the complete [Ansible playbook](encrypted_usb_backup.yaml) is here, feel free to download and improve for your specific use case.
