# Encrypted USB Drive Backup with Ansible, and a tale of Pets and Cattle

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

Not so bad, with bare-minimum error handling. But is this something that we could do also with Ansible?

Before we do that, let's talk a little about _Pet versus Cattle servers_:

## The difference between Pet versus Cattle servers

There is an ongoing debate about [how you should treat your servers](https://www.markhneedham.com/blog/2013/04/07/treating-servers-as-cattle-not-as-pets/) (either as Pet or Cattle). There are many good analogies and [explanations](http://cloudscaling.com/blog/cloud-computing/the-history-of-pets-vs-cattle/) out there, but the following summarises pretty good this way of thinking:

**Pets**
> Servers or server pairs that are treated as indispensable or unique systems that can never be down. Typically, they are manually built, managed, and “hand fed”. Examples include mainframes, solitary servers, HA loadbalancers/firewalls (active/active or active/passive), database systems designed as master/slave (active/passive), and so on.

_Bash, Python_ are perfect tools to automate tasks on Pet servers as they are very expressive, and you are not very concerned about uniformity. This doesn
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
2. Define variables to make the playbook re-usable, but we need to prompt the user for the values as opposed to defined them on the command line. Pet servers have unique features, so it is better to ask the user for some choices interactively as the playbook runs.
3. Use tags. If we want to run just parts of this playbook we can skip to the desired target (```ansible-playbook --tag $mytag encrypted_us_backup.yaml```)

### Getting your facts straight

Every time you run an Ansible playbook, it collects facts about your target system, so it can operate properly. But for our task it is overkill and we only need information about the devices, while we can ignore other details like dns, python to mention a few.

First disable the general 'gather_facts', then override with a task below using the setup module, just enabling devices and mounts.

After that you can use the facts in any way you see fit:

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
    destination_dir: "/mnt"
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

Now that we know the size of the disk, we should also get how much disk our backup will take:

__TODO__

### Putting everything together
Here is how we can implement these requirements:

```yaml
- name: Take an empty USB disk and copy user home directory into an encrypted partition
  hosts: localhost
  connection: local
  become: true
  vars:
    volgrp: usbluks_vg
    logicalvol: backuplv
    cryptns: usbluks
    cryptdevice: "/dev/mapper/{{ cryptns }}"
    destination_dir: /mnt
  vars_prompt:
    - name: source_dir
      prompt: "Enter name of the directory to back up"
      private: false
      default: "/home/josevnz"
    - name: target_device
      prompt: "Enter name of the USB device"
      private: false
      default: "/dev/sda"
    - name: passphrase
      prompt: "Enter the passphrase to be used to protect the target device"
      private: true
```

Next step is to define the following actions:
1. Get [partition details](https://docs.ansible.com/ansible/latest/collections/community/general/parted_module.html) and destroy the destination if already exists. We use a [block](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_blocks.html) to make to group these operations together (like installing crysetup at the end if missing).
```yaml
  tasks:
    - name: Basic setup and verification for target system
      block:
        - name: Facts for {{ target_device }}
          community.general.parted:
            device: "{{ target_device }}"
            state: "info"
            unit: "GB"
          register: target_facts
          tags: parted_info

        - name: Print facts for {{ target_device }}
          ansible.builtin.debug:
            msg: "{{ target_facts }}"
          when: target_facts is defined
          tags: print_facts
        - name: Unmount USB drive
          ansible.posix.mount:
            path: "{{ destination_dir }}"
            state: absent
          when: target_facts is defined
          tags: unmount
        - name: Destroy existing partition if found, update target_facts
          community.general.parted:
            device: "{{ target_device }}"
            number: 1
            state: "absent"
            unit: "GB"
          when: target_facts is defined and destroy_partition is defined
          register: target_facts
          tags: destroy_partition

        - name: Abort with an error If there are any partitions for {{ target_device }}
          ansible.builtin.debug:
            msg: -|
              {{ target_device }} is already partitioned, {{ target_facts['partitions'] }}.
              Size is {{ target_facts['disk']['size'] }} GB
          failed_when: target_facts['partitions'] is defined and (target_facts['partitions'] | length > 0) and not ansible_check_mode
          tags: fail_on_existing_part
        - name: Install crysetup
          ansible.builtin.dnf:
            name: cryptsetup
            state: installed
```
2. Create the partition, with some information printout:
```yaml
    - name: Get destination device details
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

    - name: Create partition
      block:
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
3. Create the encrypted volume, format it and then mount it. Another block:

```yaml
    - name: LUKS and filesystem tasks
      block:
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

Note than we do not want to persist the status of the mounted filesystem across reboots, we will umount the drive as soon we are done with the backup. For that reason we use a temporary fstab (```fstab: /tmp/tmp.fstab```)

4. Last task is to create the backup on the new encrypted volume using [synchronize](https://docs.ansible.com/ansible/latest/collections/ansible/posix/synchronize_module.html) (frontend for rsync, we pass a few arguments to skip unwanted heavy directories):
```yaml
    - name: Backup stage
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
          tags: backup
```

Let's test this before running it on our USB drive

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

# If you have an existing partition, you can include --extra-vars destroy_partition=yes
# Nothing will get deleted if --check is also passed

ansible-playbook --check --extra-vars destroy_partition=yes  encrypted_usb_backup.yaml
```

## Making the backup, full run

Now after all this work we can make our backup:

```shell
ansible-playbook --check --extra-vars destroy_partition=yes --tags parted_info,print_facts  encrypted_usb_backup.yaml
```

But most likely you will run it like this, with the option to destroy existing partitions:

```shell
ansible-playbook --extra-vars destroy_partition=yes encrypted_usb_backup.yaml
```

### How does the backup session looks like?

Below you can see an example of how your backup session may look like:

```shell
(EncryptedUsbDriveBackup) [josevnz@dmaf5 EncryptedUsbDriveBackup]$ ansible-playbook --extra-vars destroy_partition=yes encrypted_usb_backup.yaml
Enter the passphrase to be used to protect /dev/sda: 
Enter name of the directory to back up [/home/josevnz]: 

PLAY [Take an empty USB disk and copy user home directory into an encrypted partition] ****************************************************************************

TASK [Gathering Facts] ********************************************************************************************************************************************
ok: [localhost]

TASK [Facts for /dev/sda] *****************************************************************************************************************************************
ok: [localhost]

TASK [Print facts for /dev/sda] ***********************************************************************************************************************************
ok: [localhost] => {
    "msg": {
        "changed": false,
        "disk": {
            "dev": "/dev/sda",
            "logical_block": 4096,
            "model": "JMicron Tech",
            "physical_block": 4096,
            "size": 128.0,
            "table": "msdos",
            "unit": "gb"
        },
        "failed": false,
        "partitions": [],
        "script": "unit 'GB' print"
    }
}

TASK [Unmount USB drive] ******************************************************************************************************************************************
ok: [localhost]

TASK [Destroy existing partition if found, update target_facts] ***************************************************************************************************
ok: [localhost]

TASK [If there are any partitions for /dev/sda, abort with an error] **********************************************************************************************
ok: [localhost] => {
    "msg": "-| /dev/sda is already partitioned ({'sda1': {'links': {'ids': ['usb-JMicron_Tech_DD56419883ECF-0:0-part1'], 'uuids': ['e6e287e7-c6ac-43ad-9a7e-efc883066b61'], 'labels': [], 'masters': ['dm-0']}, 'start': '524280', 'sectors': '249033000', 'sectorsize': 512, 'size': '118.75 GB', 'uuid': 'e6e287e7-c6ac-43ad-9a7e-efc883066b61', 'holders': ['usbluks']}}), []. Size is 128.0 GB"
}

TASK [Install crysetup] *******************************************************************************************************************************************
ok: [localhost]

TASK [Get info on destination partition] **************************************************************************************************************************
ok: [localhost]

TASK [Print info_output] ******************************************************************************************************************************************
ok: [localhost] => {
    "msg": {
        "changed": false,
        "disk": {
            "dev": "/dev/sda",
            "logical_block": 4096,
            "model": "JMicron Tech",
            "physical_block": 4096,
            "size": 124952576.0,
            "table": "msdos",
            "unit": "kib"
        },
        "failed": false,
        "partitions": [],
        "script": "unit 'KiB' print"
    }
}

TASK [Create new partition] ***************************************************************************************************************************************
changed: [localhost]

TASK [Create LUKS container with passphrase] **********************************************************************************************************************
ok: [localhost]

TASK [Open luks container] ****************************************************************************************************************************************
ok: [localhost]

TASK [Create usbluks_vg on /dev/mapper/usbluks] *******************************************************************************************************************
ok: [localhost]

TASK [Create a logvol in my new vg] *******************************************************************************************************************************
ok: [localhost]

TASK [Create a filesystem for the USB drive (man mkfs.btrfs for options explanation)] *****************************************************************************
changed: [localhost]

TASK [Mount USB drive] ********************************************************************************************************************************************
changed: [localhost]

TASK [Backup /home/josevnz to /mnt//mnt] **************************************************************************************************************************
changed: [localhost]

PLAY RECAP ********************************************************************************************************************************************************
localhost                  : ok=17   changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

## Next steps

* If you want to use a GUI for your encrypted backup then you should consider [VeraCrypt](https://www.veracrypt.fr/en/Home.html), which is Open Source and also has a nice wizard to let you through the process. There is a [nice tutorial](https://opensource.com/article/21/4/open-source-encryption) how to do that.
* Instead of local use the cloud (but encrypt first!): We used rsync to make a backup to a USB drive, but Ansible can also upload files to [an S3 cloud volume](https://docs.ansible.com/ansible/2.9/modules/s3_sync_module.html). Ideally you should make an archive and encrypt it then with [sops](https://docs.ansible.com/ansible/latest/collections/community/sops/sops_encrypt_module.html) before the upload.
* If you want to back up more than one user directory you could use [a loop](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_loops.html).
* This article borrows heavily from article written by [Peter Gervase](https://www.redhat.com/sysadmin/encrypt-single-filesystem), you should spend some time reading it.
* The source code for the complete [Ansible playbook](encrypted_usb_backup.yaml) is here, feel free to download and improve for your specific use case.