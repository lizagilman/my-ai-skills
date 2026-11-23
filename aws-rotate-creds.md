# AWS Credential Rotation Runbook

> **For AI assistants:** Follow the steps below to rotate AWS IAM access keys for the user, update their local AWS CLI profile, verify credentials, and optionally re-authenticate ROSA CLI. Use the AWS CLI throughout. Ask the user only when a decision requires their input.

## Step 1 — Discover current state

Run: `aws configure list-profiles` to list all local profiles.

**If the user specified an AWS account ID upfront** (e.g. "rotate creds for account 123456789012"):
- Run `aws sts get-caller-identity --profile <profile>` for each profile in parallel to find which one matches the specified account ID.
- Use that profile automatically and confirm it to the user. If no profile matches, proceed to **Step 1a** to set up the profile first, then continue with Step 2.

**If no account was specified:**
- If there is only one profile, use it.
- If multiple profiles exist, ask the user which one to rotate.

Show the user: account ID, IAM username, and which profile will be updated.

## Step 1a — Set up a missing profile (only if no matching profile was found in Step 1)

Explain to the user that no local AWS profile was found for the specified account, and that one needs to be configured before rotation can proceed.

Ask the user for:
- A profile name to use (suggest a meaningful default like the account ID or a descriptive name)
- Their preferred region (default: `us-east-1`)

**For credentials:** recommend the user enters them interactively to avoid exposing secrets in chat history:
```
aws configure --profile <chosen-profile-name>
```
Ask them to run this themselves and confirm when done. If they explicitly prefer to provide credentials in chat, that is their choice — accept them and configure the profile using `aws configure set`.

Once confirmed, verify the profile works:
```
aws sts get-caller-identity --profile <chosen-profile-name>
```

Confirm the returned account ID matches the one the user specified. If it doesn't match, the user provided credentials for the wrong account — tell them and ask them to re-run `aws configure --profile <chosen-profile-name>` with the correct credentials.

Once verified, continue with Step 2 using this new profile.

## Step 2 — List existing access keys

```
aws iam list-access-keys --profile <profile>
```

IAM allows a maximum of 2 access keys per user. Assess the situation:

**Case A — 1 key exists:**
Proceed directly to Step 3.

**Case B — 2 keys exist:**
Identify which key the local profile is currently using by matching the last 4 chars of the key ID from `aws configure list --profile <profile>` against the list.

- Always ask the user before deleting any key — even if one is clearly older. The key may be in use on another machine, in CI, or in other tools. Show both keys (ID, creation date, status) and which one you recommend deleting, then wait for confirmation.
- If it's unclear which key is safe to delete, explain the ambiguity and let the user decide.

## Step 3 — Create new access key

```
aws iam create-access-key --profile <profile>
```

**Important:** The secret access key is shown only once. Capture both `AccessKeyId` and `SecretAccessKey` from the response immediately — they are needed in Step 4.

## Step 4 — Update local profile

```
aws configure set aws_access_key_id <new-key-id> --profile <profile>
aws configure set aws_secret_access_key <new-secret> --profile <profile>
```

Do NOT change the region or output format — preserve existing values.

## Step 4a — Check for AWS env vars

Check the following env vars in the current shell session and in common shell config files (`~/.bashrc`, `~/.zshrc`, `~/.bash_profile`, `~/.profile`):

- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
- `AWS_PROFILE`

```
echo "AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}"
echo "AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}"
echo "AWS_PROFILE=${AWS_PROFILE}"
```

```
grep -n "AWS_ACCESS_KEY_ID\|AWS_SECRET_ACCESS_KEY\|AWS_PROFILE" ~/.bashrc ~/.zshrc ~/.bash_profile ~/.profile 2>/dev/null
```

**If none are set anywhere:** skip this step silently.

---

**`AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`:**

- If set in the current shell only: inform the user they will override the profile until the terminal is restarted. Ask if they want to unset them now (`unset AWS_ACCESS_KEY_ID && unset AWS_SECRET_ACCESS_KEY`).
- If set in a config file: show the exact matching lines and ask if the user wants the skill to update them to the new credentials. If confirmed, show the exact change (old line → new line) before writing. Remind the user to `source <file>` or open a new terminal after.
- If the user declines: note in the Step 8 summary that env vars still hold the old key.

---

**`AWS_PROFILE`:**

- If set and matches the profile being rotated: no action needed — tools like ROSA CLI will correctly pick up the updated credentials from the profile.
- If set but points to a different profile: inform the user — tools like ROSA CLI will use that other profile, not the one just rotated. Ask if they want to update it.
- If not set at all: suggest the user add `export AWS_PROFILE=<profile-name>` to their shell config so tools like ROSA CLI consistently use the right AWS profile. Ask if they want the skill to add it.
- If the user confirms adding or updating `AWS_PROFILE`: show the exact line before writing, then update the config file. Remind the user to `source <file>` or open a new terminal after.

## Step 5 — Verify new credentials

```
aws sts get-caller-identity --profile <profile>
```

Confirm the returned account ID and ARN match what was shown in Step 1.

**If verification fails:** stop immediately and tell the user — do NOT delete the old key.

## Step 6 — Delete old key

Only after Step 5 succeeds, delete the key that was previously active in this profile:

```
aws iam delete-access-key --access-key-id <old-key-id> --profile <profile>
```

Confirm by listing keys again:

```
aws iam list-access-keys --profile <profile>
```

## Step 7 — ROSA CLI (optional)

Check if `rosa` is installed:

```
which rosa
```

If not installed: skip this step silently.

If installed, check whether ROSA is currently using the same AWS account that was just rotated:

```
rosa whoami
```

Compare the `AWS Account ID` field in the output against the account ID from Step 1.

- **Account IDs match:** the rotated credentials affect ROSA's AWS access. Check if the ROSA OCM token is still valid. If expired or missing, ask the user which ROSA environment they use (staging or production), then ask them to run `rosa login --use-auth-code --env <staging|production>` interactively, wait for them to confirm it completed, then run `rosa whoami` to verify.
- **Account IDs differ:** ROSA is using a different AWS account — skip re-auth, no action needed. Mention this to the user so they're aware.

## Step 8 — Summary

Report:
- Which profile was updated
- New key ID (last 4 chars only)
- Old key(s) deleted
- ROSA auth status (if applicable)
- Env var status: whether `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY`/`AWS_PROFILE` were found, and whether they were updated or left unchanged (and why)
