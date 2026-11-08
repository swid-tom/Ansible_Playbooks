# Cisco Ansible Playbooks

Playbooks in this repository automate common tasks on Cisco IOS and NX-OS
devices. They are designed to run from Ansible Automation Platform (AAP),
Ansible Tower, or AWX.

## Repository layout

```text
collections/
  requirements.yml
playbooks/
  cisco/
    Cisco_ACL_Update.yml
    cisco_backup.yml
    network_test.yml
  nexus/
    Main_Nexus_Upgrader.yml
    Upgrade_Workflow.yml
    Nexus_image_upload.yml
    Nexus_bootflash_cleanup.yml
```

In AAP, add this repository as a Project and select a playbook from its path
under `playbooks/`. `Upgrade_Workflow.yml` is a helper used by
`Main_Nexus_Upgrader.yml`; run the main playbook, not the helper, for an
upgrade.

## Before you run a job

- Add the required questions as an AAP **Survey** on the job template. Playbook
  variables do not automatically show up as launch prompts.
- Configure inventory, device credentials, and network access in AAP. Set
  `ansible_network_os` for network devices, such as `cisco.nxos.nxos` or
  `cisco.ios.ios`.
- Keep passwords and tokens in AAP credentials or secret survey fields, not in
  the repository.
- AAP can install collections listed in `collections/requirements.yml` when it
  syncs the Project. The execution environment also needs the required network
  connection dependencies.

## Choose a playbook

### Nexus pair upgrade — `playbooks/nexus/Main_Nexus_Upgrader.yml`

Upgrades vPC pairs one pair at a time: the primary first, then the secondary.
It checks recovery and vPC health, creates a report, and can email or publish
the report to Git.

Add these Survey questions:

| Variable | What to enter |
|---|---|
| `survey_hosts` | Inventory pattern containing both switches in each pair |
| `nxos_image_filename` | NX-OS image to install |
| `epld_value` | `true` or `false` |
| `epld_filename` | EPLD image; needed when `epld_value` is true |
| `issu_enabled` | `true` or `false` |

Email and Git publishing are enabled by the playbook defaults. Before running,
either set `send_email_report` and `git_publish_report` to `false`, or provide
valid SMTP settings (`smtp_host`, `mail_from`, `mail_to`) and Git settings
(`git_reports_repo_url`, `git_user_name`, `git_user_password`). Store the Git
password as a secret. The checked-in SMTP and Git destinations are examples
and must be replaced with your environment's values. To publish snapshot
files, enable both `git_publish_snapshots` and `save_snapshot_files`.

### Stage Nexus images — `playbooks/nexus/Nexus_image_upload.yml`

Copies an NX-OS image from an SFTP/SCP server to bootflash and checks its MD5.
It can also stage an EPLD image. It does not install either image.

Survey questions:

- `survey_hosts` — Nexus devices to target
- `nxos_image_filename`, `expected_md5` — NX-OS image and its MD5
- `image_dir_on_sftp`, `sftp_server` — image location and server
- `sftp_username`, `sftp_password` — server credentials; make the password secret
- `upload_epld` — `yes` or `no`
- `epld_filename`, `epld_md5`, `epld_dir_on_sftp` — needed when EPLD is enabled

The image directory is joined directly to the filename, so include a trailing
slash. The playbook works on two switches at a time.

### Clean Nexus bootflash — `playbooks/nexus/Nexus_bootflash_cleanup.yml`

Lists old NX-OS/EPLD images and, when asked, deletes them. It keeps the current
NX-OS image and one older version by default, plus the newest EPLD image.

Survey questions:

- `survey_hosts` — Nexus devices to target
- `dry_run` — `yes` to preview; set to `no` to delete
- `keep_only_current` — `yes` to keep only the current NX-OS image

Review the deletion candidates before setting `dry_run` to `no`.

### Update an ACL — `playbooks/cisco/Cisco_ACL_Update.yml`

Updates or deletes a named IPv4 ACL on IOS or NX-OS. It previews changes by
default; live replacement and deletion each require a separate confirmation.

Survey questions:

- `survey_hosts` — IOS or NX-OS devices to target
- `acl_config_text` — ACL header and numbered rules, entered as a Textarea
- `acl_delete` — `true` to delete the ACL; otherwise update it
- `acl_dry_run` — defaults to `true`; set to `false` for a live update
- `acl_state` — `merged` (default) or `replaced`
- `acl_replace_confirmed` — set to `true` only after reviewing a Check run
- `acl_delete_confirmed` — set to `true` to allow a live deletion

For deletion, enter only the ACL header. For replacement, include the complete
desired ACL. The playbook does not back up or roll back changes. It saves the
running configuration by default; `acl_save` can be set to `false`.

### Back up Cisco configs — `playbooks/cisco/cisco_backup.yml`

Backs up IOS and NX-OS configurations and pushes them to Git. The playbook
still contains a `<<GIT_URL>>` placeholder, which must be replaced before use.
It targets `all`, so limit the AAP inventory to the devices you intend to back
up. Configure credentials in AAP.

### Test NX-OS connectivity — `playbooks/cisco/network_test.yml`

Runs fact gathering and `show interface status` on the `nxos` inventory group.
It has no Survey inputs.

## Collections

Install the project's collections with:

```bash
ansible-galaxy collection install -r collections/requirements.yml
```
