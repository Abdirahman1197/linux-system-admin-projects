# Lab 01 — User and Group Management

## Objective
Provision local accounts for a small operations team, control shared-directory access, and revoke a departing contractor's access. Practice account creation, primary and supplementary groups, password aging, permissions, and account expiry.

**Status:** Lab guide prepared; execution and evidence pending. Expected output below is illustrative, not proof of a completed lab.

## Scenario and acceptance criteria
A fictional company needs two engineers to share handover notes on a Linux server. A contractor needs temporary access, while an unrelated account must have none. No account requires sudo.

| Account | Primary group | Supplementary group | Initial state |
| --- | --- | --- | --- |
| lab_alice | lab_alice | lab_ops | Engineer, interactive Bash login |
| lab_bilal | lab_bilal | lab_ops | Engineer, interactive Bash login |
| lab_temp | lab_temp | lab_ops | Contractor, expires in 7 days |
| lab_guest | lab_guest | None | Negative access-test account |

Success means engineers can create and edit each other's notes, the guest cannot access the shared directory, and the contractor loses access after offboarding while their files remain available to the team.

## Environment and tools
- Disposable RHEL 9 VM, local accounts, Bash, and an administrator with sudo.
- Snapshot the VM first. Use only these lab-prefixed identities; do not run on a production server.
- Keep SELinux enforcing. This is a local filesystem exercise, not a network service.
- Tools: `useradd`, `usermod`, `groupadd`, `gpasswd`, `passwd`, `chage`, `getent`, `id`, `runuser`, `stat`.
- Commands below run from the administrator's Bash shell. Run blocks in order; stop and investigate unexpected failures.
- Estimated time: 45–60 minutes. Record the actual OS version and date in your evidence.

## Step-by-step tasks

### 1. Check the starting state
```bash
cat /etc/redhat-release
date -I
getenforce
for name in lab_alice lab_bilal lab_temp lab_guest; do
    getent passwd "$name"
    getent group "$name"
done
getent group lab_ops
sudo test ! -e /srv/lab_ops
echo "directory check exit=$?"
```
Expected: no matching account or group records; directory check exits 0. Missing `getent` records return nonzero, which is expected here. If any identity or directory exists, stop and use a clean snapshot; do not overwrite it.

### 2. Create the team and accounts
```bash
sudo groupadd lab_ops
sudo useradd -m -U -s /bin/bash -c "Lab engineer Alice" lab_alice
sudo useradd -m -U -s /bin/bash -c "Lab engineer Bilal" lab_bilal
sudo useradd -m -U -s /bin/bash -c "Lab contractor" lab_temp
sudo useradd -m -U -s /bin/bash -c "Lab access-test account" lab_guest
sudo usermod -aG lab_ops lab_alice
sudo usermod -aG lab_ops lab_bilal
sudo usermod -aG lab_ops lab_temp
id lab_alice
id lab_bilal
id lab_temp
id lab_guest
getent group lab_ops
```
Expected: private primary groups match usernames; only the first three accounts have `lab_ops` as a supplementary group. Numeric UIDs/GIDs vary. `-U` explicitly creates private groups; `-aG` preserves existing supplementary memberships.

### 3. Set passwords and aging
Set unique temporary passwords interactively; never place passwords or hashes in Git.
```bash
sudo passwd lab_alice
sudo passwd lab_bilal
sudo passwd lab_temp
sudo chage -m 0 -M 90 -W 7 lab_alice
sudo chage -m 0 -M 90 -W 7 lab_bilal
sudo chage -m 0 -M 30 -W 7 lab_temp
sudo chage -d 0 lab_alice
sudo chage -d 0 lab_bilal
sudo chage -d 0 lab_temp
sudo chage -E "$(date -d '+7 days' +%F)" lab_temp
sudo passwd -S lab_alice
sudo chage -l lab_alice
sudo chage -l lab_temp
```
Expected: engineer maximum password age 90 days, contractor maximum 30 days, warning 7 days, minimum 0, and password change required at next login. Contractor account expiry is 7 days from today. Account expiry and password expiry are separate controls. The guest remains password-locked.

Using the VM console, log in as each of the three team accounts and complete the forced password change, then log out. Return to the administrator shell before continuing. This verifies authentication; root-run `runuser` checks later do not verify passwords.

### 4. Create a shared handover directory
```bash
sudo install -d -o root -g lab_ops -m 2770 /srv/lab_ops
stat -c '%a %U %G %n' /srv/lab_ops
sudo runuser -u lab_alice -- bash -c 'umask 0007; printf "Alice: monitoring checks complete\n" > /srv/lab_ops/handover.txt'
sudo runuser -u lab_bilal -- bash -c 'printf "Bilal: backup checked\n" >> /srv/lab_ops/handover.txt'
sudo runuser -u lab_temp -- bash -c 'umask 0007; printf "Contractor: inventory complete\n" > /srv/lab_ops/contractor.txt'
stat -c '%a %U %G %n' /srv/lab_ops/handover.txt /srv/lab_ops/contractor.txt
sudo runuser -u lab_bilal -- cat /srv/lab_ops/handover.txt
```
Expected:
- Directory: `2770 root lab_ops /srv/lab_ops`.
- Files: `660 lab_alice lab_ops` and `660 lab_temp lab_ops`.
- Handover contains Alice's and Bilal's lines.

Setgid on the directory makes new files inherit its group. The explicit umask makes these new files group-writable; setgid alone does not do that. A fresh `runuser` process picks up current group membership.

### 5. Prove access is denied to an unrelated user
```bash
sudo runuser -u lab_guest -- cat /srv/lab_ops/handover.txt
echo "read exit=$?"
sudo runuser -u lab_guest -- touch /srv/lab_ops/guest.txt
echo "write exit=$?"
sudo test ! -e /srv/lab_ops/guest.txt
echo "no guest file exit=$?"
```
Expected: both access attempts report permission denied and exit nonzero; the final check exits 0. Run each `echo` immediately after its associated command. Root reading the file would not prove isolation.

### 6. Offboard the contractor without deleting their work
In this isolated VM, make sure the contractor has logged out. Check for remaining processes before continuing:
```bash
pgrep -a -u lab_temp
```
Expected: no output (exit 1). If processes exist, end the lab sessions first. Membership changes and account locks do not revoke credentials already held by running processes.

```bash
sudo usermod -L lab_temp
sudo usermod -s /sbin/nologin lab_temp
sudo chage -E "$(date -d yesterday +%F)" lab_temp
sudo gpasswd -d lab_temp lab_ops
sudo passwd -S lab_temp
sudo chage -l lab_temp
getent passwd lab_temp
id lab_temp
sudo runuser -u lab_temp -- cat /srv/lab_ops/contractor.txt
echo "contractor read exit=$?"
sudo runuser -u lab_bilal -- cat /srv/lab_ops/contractor.txt
stat -c '%a %U %G %n' /srv/lab_ops/contractor.txt
```
Expected: password status `LK`, account expiry in the past, shell `/sbin/nologin`, no `lab_ops` membership, contractor read denied, and Bilal can still read the retained file. Some PAM configurations reject the expired account before filesystem access; either rejection is expected. Verify that a normal console login as the contractor also fails.

Password locking alone does not block SSH keys or terminate sessions. This exercise combines account expiry, a non-login shell, group removal, and checking for existing processes. A real offboarding procedure also covers keys, tokens, scheduled jobs, and other access paths.

## Verification summary
Run after step 6. Compare observations against the criteria rather than copying illustrative values.

| Check | Command | Expected result |
| --- | --- | --- |
| Engineer membership | `id lab_alice; id lab_bilal` | Both include lab_ops |
| Contractor membership | `id lab_temp` | No lab_ops |
| Guest membership | `id lab_guest` | No lab_ops |
| Directory permissions | `stat -c '%a %U %G' /srv/lab_ops` | 2770 root lab_ops |
| Shared file | `stat -c '%a %U %G' /srv/lab_ops/handover.txt` | 660 lab_alice lab_ops |
| Account policy | `sudo chage -l lab_alice` | Maximum 90, warning 7 |
| Contractor lock | `sudo passwd -S lab_temp` | LK |
| Contractor expiry | `sudo chage -l lab_temp` | Date in the past |
| Retained work | `sudo runuser -u lab_bilal -- cat /srv/lab_ops/contractor.txt` | Contractor's note readable |

Repeat the engineer write and guest deny checks after a VM reboot to demonstrate persistence. Record any differences.

## Common mistakes and troubleshooting

| Symptom or mistake | Explanation and correction |
| --- | --- |
| Existing memberships disappear | `usermod -G` replaces supplementary groups. Use `-aG` when appending. |
| A terminal still has old access | Existing processes keep their group credentials. Log out fully and create a fresh session. |
| Shared file cannot be edited | Setgid controls group inheritance, not write bits. Inspect `stat`, `id`, and the creating process's umask. |
| Guest can read the handover | Check directory ownership/mode and unexpected ACLs with `getfacl /srv/lab_ops` if installed. Do not fix this with chmod 777. |
| User has no home or wrong shell | Inspect `getent passwd USER`; creation explicitly uses `-m -s /bin/bash`. |
| New user cannot log in | Check password status, account expiry, and whether first-login password change was completed. |
| Locked user still has activity | A password lock does not kill processes or disable all authentication methods. Inspect sessions and account expiry. |
| Permission denied despite expected Unix modes | Inspect parent directory traversal with `namei -l`, ACLs, and SELinux audit messages. Keep SELinux enabled and diagnose the cause. |
| UID/GID differs from example | IDs are allocated locally. Verify names and membership instead of assuming numbers. |

## Outcome and GitHub evidence
**Actual outcome:** Not yet recorded. Complete this section after running the lab.

- [ ] Record date, OS version, VM platform, and SELinux mode.
- [ ] Capture sanitized `id`, `stat`, `passwd -S`, and `chage -l` output.
- [ ] Record successful engineer write and failed guest access with exit codes.
- [ ] Record successful first login and password change without recording passwords.
- [ ] Record contractor lock, expiry, group removal, failed login, and retained work.
- [ ] Reboot and repeat access checks.
- [ ] Add one issue encountered, its cause, and how it was resolved.
- [ ] Explain primary vs supplementary groups, setgid vs umask, and lock vs expiry.
- [ ] Store sanitized evidence in an `evidence/` subdirectory here and link it below.
- [ ] Review `git diff --check` and `git diff` before committing.

Never commit passwords, password hashes, `/etc/shadow`, SSH keys, tokens, or unredacted personal/production details. Screenshots are optional; text transcripts are easier to review. Label all examples separately from actual results.

## Cleanup
Preferred: restore the disposable VM snapshot after saving sanitized evidence. This also resets passwords, identities, and file ownership consistently. Do not manually delete the contractor or their work until retention checks are documented.

## References
Use the installed RHEL man pages for exact options:
```bash
man useradd
man usermod
man chage
man passwd
man runuser
man chmod
man umask
```
- [Red Hat Enterprise Linux 9 — Managing users and groups](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/configuring_basic_system_settings/managing-users-and-groups_configuring-basic-system-settings)
- [Reusable lab README template](../LAB_TEMPLATE.md)
