# Linux System Admin Projects

Hands-on Linux administration practice with RHCSA-oriented scenarios, commands, verification, and troubleshooting.

## Practical labs

| Lab | Scenario | Status |
| --- | --- | --- |
| [01 — User and Group Management](user_and_group_management/README.md) | Onboard engineers, share handover notes, test access, and offboard a contractor | Guide ready; execution evidence pending |

A published guide is not a claim of completed hands-on work. Each lab records actual results and sanitized evidence when executed.

## Project roadmap

The existing topic directories are retained:

- [Basic File System Management](basic_file_system_management/README.md)
- [User and Group Management](user_and_group_management/README.md)
- [Managing System Processes & Services](managing_system_processes_%26_services/README.md)
- [LVM, Stratis, and VDO Management](lvm,_stratis,_and_vdo_management/README.md)
- [Automating Tasks with Cron & Shell Scripting](automating_tasks_with_cron_%26_shell_scripting/README.md)
- [Configuring Networking and Firewalld](configuring_networking_and_firewalld/README.md)
- [Secure Web Server with Firewall](secure_web_server_with_firewall/README.md)
- [Managing Containers with Podman](managing_containers_with_podman/README.md)
- [Writing a Simple Ansible Playbook](writing_a_simple_ansible_playbook/README.md)

Topics other than Lab 01 currently contain placeholders. This roadmap is a personal learning plan, not a version-specific exam objective list.

## Adding a lab

Copy [LAB_TEMPLATE.md](LAB_TEMPLATE.md) into the relevant existing topic directory as its README. Keep the scenario, ordered tasks, expected results, verification, common mistakes, evidence, and cleanup sections. Update the practical-lab index when the guide is ready, and mark execution separately.

Use disposable VMs and publish only sanitized evidence. Never commit credentials or password hashes.
