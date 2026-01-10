# Ansible Tower Playbooks

This repository contains Ansible playbooks designed to run in Ansible Tower/AWX for Cisco network device automation.

## Playbooks

### `cisco_backup.yml`
Automated backup of Cisco device configurations with Git version control.
- Backs up running configs from IOS and NX-OS devices
- Commits backups to GitLab with date-stamped commits
- Stores configs in Git Repository in `backups/cisco/ios/` and `backups/cisco/nxos/`

### `Nexus_image_upload.yml`
Stages NX-OS and EPLD images on Nexus switches via SFTP/SCP.
- Supports Tower Survey variables for flexible deployment
- MD5 verification before and after transfer
- Skips upload if same or newer version already exists
- Optional EPLD image staging
- Processes 2 switches at a time (serial execution)
- Serial execution for 2 switches at a time (can change to a variable for survey)

**Survey Variables:**
- `survey_hosts` - Target hosts
- `nxos_image_filename` - NX-OS image filename
- `expected_md5` - Expected MD5 checksum
- `upload_epld` - Yes/No for EPLD upload
- `epld_filename` / `epld_md5` - EPLD details (if enabled)
- SFTP server credentials (Ansible Vault)

### `network_test.yml`
Basic connectivity validation and fact gathering for NX-OS devices.
- Gathers device facts
- Runs `show interface status`

### `Nexus_bootflash_cleanup.yml`
Deletes old NX-OS and EPLD images on Nexus switches.
- Checks current version of switch and keeps current and one version behind of NXOS image
- Only keeps the most current version of EPLD found
- Defaults to a "Dry Run" - shows what it would delete without deleting.

**Survey Variables:**
- `survey_hosts` - Target hosts
- `dry_run` - Yes/No for a dryrun
- `Keep only Current NXOS Bin?` - If you just want to keep the current version bin, else it will keep current and one version older

## Collections Required
- `cisco.ios`
- `cisco.nxos`
- `ansible.netcommon`

## Usage
These playbooks are intended to be run from Ansible Tower/AWX. Configure job templates with appropriate inventories and credentials.
