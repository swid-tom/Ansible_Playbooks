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

---
# Nexus vPC Pair Upgrade Playbooks

This workspace contains Ansible playbooks to discover Nexus vPC pairs, validate pair/role integrity, run a primary-first upgrade workflow, and publish a detailed report.

## Files

- `Main_Nexus_Upgrader.yml`
  - Main orchestration playbook.
  - Discovers vPC pair relationships from `show vpc role`.
  - Validates reciprocal pairing and primary/secondary role mapping.
  - Saves running config before upgrades.
  - Runs per-pair workflow using `Upgrade_Workflow.yml`.
  - Generates final report and can optionally publish to Git and/or email.

- `Upgrade_Workflow.yml`
  - Per-pair workflow.
  - Captures PRE interface status snapshots.
  - Upgrades primary first, verifies recovery and vPC convergence.
  - Upgrades secondary, verifies recovery and vPC convergence.
  - Captures POST snapshots.
  - Computes connected-only diffs and asserts no regressions.

- `collections/requirements.yml`
  - Declares required collection:
    - `community.general` (used for SMTP mail task).

- `ansible.cfg`
  - Connection timeout tuning used by this workflow.

## High-level Workflow

1. Discover each device role and local/peer vPC MAC.
2. Build reciprocal pairs controller-side.
3. Validate each pair has exactly one primary and one secondary.
4. Save running config to startup config on all hosts.
5. Process each pair sequentially:
   - PRE snapshots
   - Primary upgrade + recovery checks
   - Secondary upgrade + recovery checks
   - POST snapshots
   - Diff/assert
6. Build a consolidated report.
7. Optionally push artifacts to Git.
8. Optionally email the report.

## Prerequisites

- Ansible control environment with network connectivity to Nexus devices.
- Inventory/credentials for `ansible.netcommon.network_cli` access.
- Cisco collection support for NX-OS modules used by playbooks.
- Community collection for mail support:

```bash
ansible-galaxy collection install -r collections/requirements.yml
```

If running in AAP/AWX execution environments, ensure project collection install is enabled or pre-bake required collections.

## Key Variables

Most important runtime inputs are defined in `Main_Nexus_Upgrader.yml` and should typically be overridden via AAP Survey or extra vars.

### Targeting and Images

- `survey_hosts`: host pattern to run against.
- `nxos_image_filename`: NX-OS image.
- `epld_value`: set true/1 to include EPLD install path.
- `epld_filename`: EPLD image (required when EPLD path is enabled).
- `issu_enabled`: ISSU toggle for standard install module path.

### Upgrade and Recovery Timing

- `wait_timeout_sec`: max time for SSH return after reboot.
- `settle_seconds`: extra pause after SSH return before health checks.
- `halt_on_failure`: stop remaining pairs after first failed pair.

### Report Options

- `report_dir`: output directory on controller/EE.
- `report_snapshot_view`: `table` or `json` representation in report.
- `save_snapshot_files`: save per-host PRE/POST snapshot files.

### Email Options

- `send_email_report`: enable/disable email send.
- `smtp_host`: required when email is enabled.
- `smtp_port`: default 25.
- `mail_from`, `mail_to`: required when email is enabled.
- `smtp_user`, `smtp_password`: optional.
  - Leave empty for unauthenticated relay (typical port 25).
  - Provide for authenticated SMTP.

### Git Publish Options

- `git_publish_report`: enable/disable Git publishing.
- `git_reports_repo_url`: target repo URL.
- `git_reports_repo_path`: local clone path.
- `git_reports_subdir`: subfolder for report files.
- `git_publish_snapshots`: include snapshot JSON files.
- `git_snapshots_subdir`: subfolder for snapshot files.
- `git_branch`: target branch.
- `gitlabuser`, `gitlabpassword`: credentials.

## Run Examples

### Run directly with ansible-playbook

```bash
ansible-playbook -i inventory Main_Nexus_Upgrader.yml \
  -e "survey_hosts=nexus_pair_group" \
  -e "nxos_image_filename=nxos.9.3.11.bin" \
  -e "epld_value=false" \
  -e "issu_enabled=false"
```

### Example: enable email without SMTP auth (port 25 relay)

```bash
ansible-playbook -i inventory Main_Nexus_Upgrader.yml \
  -e "send_email_report=true" \
  -e "smtp_host=mail.example.local" \
  -e "smtp_port=25" \
  -e "mail_from=netops@example.local" \
  -e "mail_to=team@example.local" \
  -e "smtp_user=" \
  -e "smtp_password="
```

## Health Check Behavior

After each device upgrade:

1. Waits for TCP/22 (SSH) to return.
2. Waits an additional settle period.
3. Retries `show vpc` convergence checks until:
   - `vPC status : up`
   - `Peer status : peer adjacency formed`

This avoids false success on simple port reachability and gates progress on actual control-plane convergence.

## Output Artifacts

- Main report:
  - `{{ report_dir }}/vpc_upgrade_report_<timestamp>.txt`
- Optional snapshots:
  - `{{ report_dir }}/<host>_pre_interfaces.json`
  - `{{ report_dir }}/<host>_post_interfaces.json`
- Optional set_stats artifacts for AAP job output:
  - Pair discovery data and per-host diff summaries.

## Troubleshooting

- Pair validation fails:
  - Confirm all target devices are true vPC peers.
  - Check `show vpc role` output consistency and inventory scope.

- SSH wait times out:
  - Increase `wait_timeout_sec` for slower reloads.
  - Confirm management reachability from control node/EE.

- vPC convergence check fails:
  - Increase settle or retry windows if control-plane takes longer.
  - Inspect `show vpc` output from failed host.

- Email task fails with module error:
  - Ensure `community.general` is installed from `collections/requirements.yml`.

- Git publish fails:
  - Validate credentials, branch permissions, and repository URL.

## Safety Notes

- This workflow is designed for sequential pair processing to reduce blast radius.
- Keep `halt_on_failure=true` for conservative change windows.
- Validate in lab/non-production before production maintenance windows.


## Usage
These playbooks are intended to be run from Ansible Tower/AWX. Configure job templates with appropriate inventories and credentials.
