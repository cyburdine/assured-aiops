---
name: assured
description: Translate a GitHub issue into an idempotent Ansible playbook, document the draft on the issue for review, and execute it only after explicit approval. Use when the user asks to implement or fulfill a GitHub issue against managed servers, or invokes /aiops:assured with a repo and issue number.
when_to_use: User says things like "implement issue 42", "run the work in this issue against my servers", "use assured-aiops to handle this ticket", or directly invokes /aiops:assured owner/repo NN.
argument-hint: <owner/repo> <issue-number> [approved]
disable-model-invocation: true
allowed-tools: Bash(gh issue view *) Bash(gh issue comment *) Bash(gh issue close *) Bash(gh auth status) Bash(gh --version) Bash(ansible *) Bash(ansible-inventory *) Bash(ansible-playbook *) Bash(cat *) Bash(ls *) Bash(test *) Read Write Edit Grep Glob
---

# assured (assured-aiops)

Implement a GitHub issue against managed infrastructure by drafting an Ansible playbook, posting it to the issue for human review, and executing it idempotently only after explicit approval.

This skill runs in two phases. The first phase drafts and documents. The second phase executes. The user controls the transition between them by re-invoking the skill with the literal argument `approved`.

## Prerequisites (verify first, fail fast)

Before doing anything else, verify the environment. If any check fails, stop and tell the user exactly what to fix.

1. `gh --version` must succeed. If not: tell the user to install GitHub CLI from https://cli.github.com.
2. `gh auth status` must show authenticated. If not: tell the user to run `gh auth login`.
3. `ansible --version` must succeed. If not: tell the user to install Ansible.
4. `ansible-playbook --version` must succeed.

Do these as a single combined check and report all failures at once. Do not proceed if any check fails.

## Parse arguments

The skill is invoked as `/aiops:assured <owner/repo> <issue-number> [approved]`.

- `$0` is `owner/repo` (e.g., `octocat/hello-world`)
- `$1` is the issue number (e.g., `42`)
- `$2` if present and equal to the literal string `approved` means execute the previously-drafted playbook

If `$0` or `$1` is missing, stop and tell the user the correct invocation.

If `$2` is present but is not the literal string `approved`, stop and ask the user whether they meant to approve.

## Phase routing

- If `$2` is absent: run the **Draft phase**.
- If `$2` is `approved`: run the **Execute phase**.

---

## Draft phase

Goal: read the issue, understand the goal, draft an idempotent playbook, dry-run it, and post everything to the issue.

### Step 1. Read the issue

Run `gh issue view $1 --repo $0 --json title,body,labels,state`.

If the issue is closed, stop and tell the user.

Read the issue body carefully. Identify:
- The concrete goal (what state should the servers end up in?)
- The target hosts or groups (if specified). If not specified, ask the user before continuing.
- Any constraints (maintenance windows, exclusions, ordering)

If the goal is ambiguous, ask the user one clarifying question. Do not guess.

### Step 2. Discover the Ansible inventory

Find the inventory in this order. Stop at the first one that resolves.

1. If `ansible.cfg` exists in cwd, read its `[defaults]` section for an `inventory` key.
2. Look for `./inventory`, `./inventory.yml`, `./inventory.yaml`, `./hosts`, `./hosts.yml`, `./hosts.yaml` in the current directory.
3. Look for `~/.ansible/hosts`.
4. Look for `/etc/ansible/hosts`.
5. If none found, ask the user for the inventory path. Do not invent one.

Once an inventory is identified, run `ansible-inventory -i <path> --list` to validate it and see the available hosts and groups. If this command fails, surface the error to the user and stop.

### Step 3. Gather facts from target hosts

For the hosts or groups identified in Step 1, run:

```
ansible -i <inventory> <target> -m setup --tree /tmp/assured-aiops-facts-$1
```

If this fails on some hosts, note which ones in your draft and let the user decide whether to proceed without them.

Use the gathered facts to make informed playbook decisions (OS family, package manager, init system, existing service state, etc.). Do not draft a playbook that assumes facts you have not gathered.

### Step 4. Draft the playbook

Write the playbook to `/tmp/assured-aiops-playbook-$1.yml`.

Follow the patterns in `references/playbook-patterns.md`. Load that file before drafting.

Key rules:
- Idempotent by default. Prefer built-in modules over `command`/`shell`.
- `become: true` at the play level unless the task explicitly should not escalate.
- Use `hosts:` matching what was identified in Step 1.
- Add `gather_facts: true` only if you actually need fresh facts (you already gathered them in Step 3, but the playbook may need them at run time too).
- No hardcoded secrets. If the issue implies secrets are needed, reference variables and note in the issue comment that the user must provide them via vault or `--extra-vars`.

### Step 5. Detect vault usage

Scan the drafted playbook for any references to encrypted vars (e.g., `!vault` tags, `vars_files:` pointing at files with `vault` in the name, or any `{{ var }}` reference where the var name suggests a secret). If vault is involved, flag it in the issue comment with a note: "This playbook references vault-encrypted variables. When executing, you will need to provide vault credentials (e.g., `--ask-vault-pass`)."

### Step 6. Dry-run the playbook

Run:

```
ansible-playbook -i <inventory> /tmp/assured-aiops-playbook-$1.yml --check --diff
```

Capture stdout and stderr.

If dry-run fails entirely (syntax error, unreachable hosts, etc.), do not post a broken draft. Surface the error to the user and offer to revise the playbook.

If dry-run succeeds but shows unexpected changes (changes outside what the issue asked for), call that out in the issue comment as a warning. Do not silently include them.

### Step 7. Post the draft to the issue

Build a comment with this structure:

```
## assured-aiops draft

### Goal (as I understand it)
<one or two sentences summarizing what you understood from the issue>

### Target hosts
<list of hosts or groups, from the inventory>

### Playbook
\`\`\`yaml
<the playbook content>
\`\`\`

### Dry-run output (--check --diff)
\`\`\`
<the captured output>
\`\`\`

### Notes
<vault warning if applicable, host gather failures if any, unexpected changes if any>

### To approve and execute
Re-invoke with: \`/aiops:assured $0 $1 approved\`

The playbook posted above will be used as-is. If you want changes, edit the playbook content in this comment before approving, and assured-aiops will use the edited version.
```

Post it with `gh issue comment $1 --repo $0 --body-file <path-to-comment-file>`.

### Step 8. Stop

Tell the user the draft is posted, give them the URL to the issue (you can get it from the `gh issue view` JSON output), and remind them how to approve. Do not execute the playbook. The Draft phase ends here.

---

## Execute phase

Goal: re-read the most recent draft comment on the issue, run that exact playbook, post results back, and close the issue.

### Step 1. Re-verify prerequisites

Same as Draft phase Step 1. Fail fast if anything is missing.

### Step 2. Read the latest assured-aiops comment

Run `gh issue view $1 --repo $0 --comments --json comments,title,body,state`.

Find the most recent comment that starts with `## assured-aiops draft`. If none exists, stop and tell the user: "No assured-aiops draft found on issue $1. Run the draft phase first."

If the issue is closed, stop and tell the user.

### Step 3. Extract the playbook from the comment

The playbook is in the yaml code fence after the "### Playbook" heading. Parse it out.

**Important:** The user may have edited the playbook in the comment before approving. That is allowed and expected. Use the playbook as it currently appears in the comment, not any local file.

Write the extracted playbook to `/tmp/assured-aiops-execute-$1.yml`.

### Step 4. Re-discover inventory

Same logic as Draft phase Step 2. The inventory may have moved between phases, so do not assume.

### Step 5. Final confirmation in chat

Show the user:
- The playbook that will run (in chat, not just the file path)
- The inventory and target hosts
- Any vault flags they need to add (e.g., `--ask-vault-pass`)

Ask: "Run this now?" Wait for affirmative response (`yes`, `y`, `run`, `go`). If anything else, stop.

This is a soft gate, intentional. The user already approved on GitHub, but a final terminal-side check guards against the case where they re-invoke from the wrong directory or against the wrong inventory.

### Step 6. Execute

Run:

```
ansible-playbook -i <inventory> /tmp/assured-aiops-execute-$1.yml --diff
```

Stream output to the user as it runs. Capture full stdout and stderr.

If vault was flagged in the draft, the user is responsible for passing `--ask-vault-pass` or `--vault-password-file` themselves. If the run fails because of missing vault credentials, surface that clearly.

### Step 7. Post results to the issue

Build a results comment:

```
## assured-aiops execution results

### Status
<succeeded | failed | partial>

### Output
\`\`\`
<full ansible-playbook output>
\`\`\`

### Summary
- Hosts changed: <list>
- Hosts ok (no change): <list>
- Hosts unreachable: <list>
- Hosts failed: <list>
```

Post with `gh issue comment $1 --repo $0 --body-file <path>`.

### Step 8. Close the issue (only on full success)

If every host succeeded with no failures and no unreachable hosts, run:

```
gh issue close $1 --repo $0 --comment "Closed by assured-aiops after successful execution."
```

If there were any failures or unreachable hosts, do not close. Tell the user the issue stays open for follow-up.

---

## When things go wrong

Read `references/troubleshooting.md` if you encounter:
- `gh auth status` failures
- `ansible-inventory` errors
- Hosts unreachable during fact gathering
- Dry-run shows changes that look wrong
- Vault detection edge cases
- The user wants to abort mid-flow

## Cleanup

The skill writes to `/tmp/assured-aiops-*-$1*`. These are intentionally left in place so the user can inspect them after the fact. Do not delete them automatically. If the user asks for cleanup, remove them then.
