# Assured AIOps for Claude Code

A Claude Code plugin that turns GitHub issues into reviewed, idempotent Ansible playbooks.

You open an issue describing what you want done. You invoke the skill in Claude Code. It reads the issue, drafts an Ansible playbook, runs it in dry-run mode, and posts the whole thing back to the issue as a comment. Nothing executes until you re-invoke with the literal argument `approved`. Then it runs, posts results back to the issue, and closes the issue on full success.

This implements the **AI suggests | Ansible enforces** pattern described in [Assured AI Ops](https://cyburdine.substack.com/p/assured-ai-ops).

## What you get

- **Auditable**: every action documented in a GitHub issue. Goal, drafted playbook, dry-run output, execution results, all in one timeline.
- **Reviewable**: the playbook lives in the issue comment as YAML. You can edit it on GitHub before approving and the skill will use your edited version.
- **Idempotent by default**: the skill drafts with built-in Ansible modules rather than shell commands, so re-running is safe.
- **Human gated**: nothing runs against your servers without a second deliberate invocation containing the literal word `approved`.
- **No surprises**: every drafted playbook is dry-run with `--check --diff` before you ever see it.

## Install

```
/plugin marketplace add cyburdine/assured-aiops
/plugin install aiops@cyburdine-aiops
```

(Replace `cyburdine` with whatever GitHub user/org hosts the marketplace if you've forked it.)

## Prerequisites

You need all of these on the machine running Claude Code:

- **[GitHub CLI (`gh`)](https://cli.github.com)**, authenticated with `gh auth login`. The skill uses `gh` to read and comment on issues.
- **Ansible**, installed and on your PATH. On macOS: `brew install ansible`.
- **An Ansible inventory** that the skill can find. By default it looks in `./inventory`, `./hosts`, `~/.ansible/hosts`, `/etc/ansible/hosts`, or anywhere `ansible.cfg` points it. If none of those exist, it will ask.
- **SSH access** to the hosts in your inventory, the way Ansible normally expects (keys loaded in your agent, etc.).

## Usage

1. **Open a GitHub issue** describing the goal. Be specific about which hosts or groups to target.

   Example issue body:
   ```
   Install htop on <my_hostname> so I can debug a process issue. Standard apt install.
   ```

2. **Invoke the skill in Claude Code (Draft phase):**

   ```
   /aiops:assured cyburdine/myrepo 11
   ```

   The skill will read the issue, discover your inventory, gather facts from the target hosts, draft a playbook, dry-run it with `--check --diff`, and post the playbook plus the dry-run output to the issue as a comment.

3. **Review on GitHub.** Open the issue. Read the drafted playbook. If you want changes, edit the playbook content directly inside the comment (the YAML block under the `### Playbook` header). Your edits become the source of truth.

4. **Approve and execute:**

   ```
   /aiops:assured cyburdine/myrepo 11 approved
   ```

   The skill reads the playbook back from the issue (your edits respected), shows it to you one more time in your terminal for a final confirmation, then runs it. It posts the results back to the issue and closes the issue on full success.

## What happens if execution fails

If any host fails or is unreachable during execution, the skill posts the full output to the issue but **does not close it**. The issue stays open so you can investigate.

## Notes on the approval gate

The skill uses `disable-model-invocation: true` in its frontmatter, meaning Claude will not start running this skill on its own based on conversational context. You have to type the slash command yourself. This is intentional. The skill is talking to real infrastructure.

## Limitations and known rough edges

This is v0.1. Things that work but could be sharper:

- The skill assumes a single-`gh` authentication. If you have multi-account setups it uses whichever account `gh auth status` shows as active.
- Vault detection is conservative. The skill flags potential vault usage in the comment so you can pass `--ask-vault-pass` at execution time, but it doesn't manage vault credentials itself.
- The Execute phase parses the playbook back out of the issue comment as the YAML inside the `### Playbook` code fence. Manual edits to that block work. Manual edits that change the comment's structure (deleting the header, changing the fence) will confuse it.
- The skill currently runs against the hosts directly. There's no integration with Ansible Automation Platform yet. That's a logical next step (PRs welcome).

## License

MIT. See [LICENSE](./LICENSE).

## Related

- [Assured AI Ops (original Substack post)](https://cyburdine.substack.com/p/assured-ai-ops): the pattern this skill implements.
- [Claude Code docs: skills](https://code.claude.com/docs/en/skills)
- [Claude Code docs: plugin marketplaces](https://code.claude.com/docs/en/plugin-marketplaces)
