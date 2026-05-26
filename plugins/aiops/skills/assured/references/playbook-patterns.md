# Playbook patterns

Reference for drafting idempotent Ansible playbooks in the assured-aiops workflow. Load this when drafting a playbook in the Draft phase.

## Idempotency first

The whole point of the approval gate is wasted if the playbook isn't safe to run more than once. Every task must be idempotent.

**Prefer built-in modules over `command` or `shell`.** Built-in modules detect existing state and only act when needed. `command`/`shell` always run unless you add `creates:`, `removes:`, or a `changed_when:` clause.

| Goal | Use | Not |
|---|---|---|
| Install a package | `ansible.builtin.package` (or `apt`, `dnf`, `yum`) | `command: apt install foo` |
| Manage a file | `ansible.builtin.copy` or `template` | `shell: echo ... > /etc/foo` |
| Manage a service | `ansible.builtin.service` or `systemd` | `command: systemctl restart foo` |
| Manage a user | `ansible.builtin.user` | `command: useradd foo` |
| Manage a directory | `ansible.builtin.file` with `state: directory` | `command: mkdir -p` |
| Line in a config file | `ansible.builtin.lineinfile` or `blockinfile` | `shell: echo ... >>` |
| Replace text in a file | `ansible.builtin.replace` | `sed -i` |

If you must use `command` or `shell`, add `creates:` or `removes:` to make it idempotent, or `changed_when:` to make change reporting accurate.

## Become semantics

The default at the play level is `become: true` per assured-aiops convention. Specific overrides:

- A task that runs as a non-root service user: add `become_user: <user>` at the task level. `become: true` is still required so Ansible can `sudo` to that user.
- A task that genuinely shouldn't escalate (e.g., reading the invoking user's `~/.ssh`): add `become: false` at the task level.

Do not put `become: true` on individual tasks if it's already set at the play level. Redundant and noisy.

## Handlers

Use handlers for restart-on-change patterns. Notify them from the task that changes config.

```yaml
- name: Deploy nginx config
  ansible.builtin.template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
  notify: Restart nginx

handlers:
  - name: Restart nginx
    ansible.builtin.service:
      name: nginx
      state: restarted
```

Handlers only run if the notifying task reports a change. They run at the end of the play, not immediately. If you need immediate restart, use `meta: flush_handlers` or call the service task directly.

## Variables

- Define variables in the playbook's `vars:` block for playbook-specific values.
- Reference inventory variables (host_vars, group_vars) by name. Don't hardcode IPs, FQDNs, or paths that vary per host.
- If a value is sensitive, reference a variable name like `{{ db_password }}` and add a note in the issue comment that the user must provide it via vault or `--extra-vars`. Never embed the value.

## Fact dependencies

You already gathered facts in Draft Step 3 before drafting. If your playbook depends on facts at runtime (e.g., `ansible_os_family`), keep `gather_facts: true`. If you've already pinned every value from the gathered facts and don't need fresh data, you can set `gather_facts: false` to speed up the run. When in doubt, leave it on. Facts are cheap.

## Tags

Don't add tags unless the playbook is long enough that selective execution matters. For a typical issue-driven playbook (one goal, a handful of tasks), tags are noise.

## Error handling

- Use `failed_when:` to refine what counts as a failure if the module's default is too strict or too loose.
- Use `ignore_errors: true` very rarely, and only with `register:` plus a subsequent check.
- For batch operations across hosts where one host failing shouldn't stop the rest, set `any_errors_fatal: false` (default) and the play continues. If the work is interdependent across hosts, set `any_errors_fatal: true`.

## Don'ts

- Don't write playbooks that print sensitive values to stdout (`debug:` on a password var). The dry-run output gets posted to GitHub.
- Don't use `delegate_to: localhost` to run commands on the Mac unless the issue specifically asked for that. The skill targets remote servers.
- Don't introduce new roles or `include_tasks:` references to files that don't exist. The playbook drafted must be self-contained or reference only modules that ship with Ansible.
- Don't add `become_method: su` unless the issue requires it. Default `sudo` is what was configured.

## Minimal skeleton

```yaml
---
- name: <one-line summary of the goal>
  hosts: <group or host pattern from inventory>
  become: true
  gather_facts: true

  vars:
    # only if needed

  tasks:
    - name: <descriptive name of the change>
      ansible.builtin.<module>:
        <args>

  handlers:
    # only if needed
```
