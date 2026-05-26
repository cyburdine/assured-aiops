# Troubleshooting

Reference for assured-aiops failure modes. Load this when something goes wrong during the Draft or Execute phase.

## `gh auth status` fails

Symptom: `gh auth status` returns non-zero, or the JSON output shows the user is not logged in.

Action: stop the skill and tell the user:
> GitHub CLI is not authenticated. Run `gh auth login` and choose your preferred method (HTTPS with browser is simplest). Re-invoke assured-aiops once authenticated.

Do not attempt to authenticate on the user's behalf. Do not suggest setting a token via environment variable unless the user explicitly asks for a non-interactive setup.

## `ansible-inventory --list` fails

Symptom: command returns non-zero with a parse error, missing-file error, or "no inventory" error.

Action:
- If the error mentions a parse error in the inventory file, surface the exact error and tell the user to fix the file. Do not attempt to "auto-repair" inventory.
- If the error is "no inventory was parsed", verify the path actually exists with `test -f <path>`. If it doesn't, fall back to the next inventory discovery step (see SKILL.md Draft Step 2).
- If a dynamic inventory script is being used and it fails, surface the script's stderr verbatim.

## Hosts unreachable during fact gathering

Symptom: `ansible -m setup` reports UNREACHABLE for some hosts.

Action:
- List the unreachable hosts to the user.
- Ask: "Proceed with drafting against the reachable hosts only, retry all, or abort?"
- Do not silently proceed with partial fact data. The playbook quality depends on accurate facts.

If the user chooses to proceed with reachable hosts only, note this prominently in the issue comment under "### Notes". The drafted playbook should target only the hosts that responded.

## Dry-run shows changes that look wrong

Symptom: `--check --diff` output shows changes the user did not ask for, or shows zero changes when changes were expected.

If unexpected changes are shown:
- Read the diff carefully. Sometimes the playbook is correct but the current state is unexpected (e.g., a file is currently in a different state than the playbook target).
- Call out the unexpected changes explicitly in the issue comment under "### Notes". Do not bury them.
- Do not edit the playbook to "hide" the changes. If they're real, the user needs to see them.

If zero changes are shown when changes were expected:
- Verify the playbook actually targets the intended hosts.
- Check whether the desired state already exists (the playbook is correct and the work is already done).
- If you cannot explain the lack of changes, surface this to the user and ask them to inspect.

## Vault detection: ambiguous case

The skill scans for vault encryption. Edge cases:
- A variable named `{{ admin_password }}` referenced from `vars:` with a plain string value. This is not vault, just a plaintext secret in the playbook. **Do not draft this.** Refuse and tell the user secrets must go into vault.
- A `vars_files:` entry pointing to a path that doesn't exist. Note this in the comment so the user knows the file must exist at execution time.
- A `!vault` tag in `vars:`. This is vault. Flag for `--ask-vault-pass`.

## User wants to abort mid-flow

If the user says stop, abort, cancel, never mind:
- Acknowledge.
- Do not post anything to the GitHub issue if you haven't already.
- If a draft comment was already posted, ask whether they want it removed (`gh issue comment delete` is not available via gh CLI directly, so it would require the API). Default to leaving it and adding a follow-up comment: "assured-aiops draft was cancelled by user. Do not approve."
- Clean up temp files in `/tmp/assured-aiops-*-$1*` if the user asks.

## Execute phase: playbook in comment doesn't parse

Symptom: extracting the yaml from the comment yields invalid YAML.

Action:
- Show the user the exact extracted content.
- Do not attempt to fix or guess.
- Suggest the user edit the issue comment to fix the YAML and re-invoke.

## Execute phase: hosts in playbook don't match current inventory

Symptom: the playbook targets `webservers` but the current inventory has no such group, or it targets a host that no longer exists.

Action:
- Stop before running.
- Tell the user the playbook in the issue references hosts/groups not in the current inventory.
- Ask whether they want to re-run the Draft phase (which will rebuild against the current inventory) or fix the inventory.

## Execute phase: partial failure

Symptom: ansible-playbook completes but some hosts failed or were unreachable.

Action:
- Do not close the issue.
- Post the full results comment as described in SKILL.md Execute Step 7.
- Tell the user: "Issue stays open. Review the failed/unreachable hosts before deciding next steps."

## General principle

When in doubt, stop and ask the user. The skill should fail loud and clear, never silent and clever. The whole architecture exists to keep a human in the loop. Don't undermine that by guessing.
