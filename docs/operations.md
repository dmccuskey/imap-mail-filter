# IMAP Mail Filter Operations

## Running the Filter

The main program is:

```bash
python3 imap_mail_filter.py
```

The script loads `accounts.local.json5`, `folders.local.json5`, and `rules.local.json5` from the directory containing `imap_mail_filter.py`, regardless of the current working directory.

On the Raspberry Pi deployment, this is:

```text
/home/pi/imap-mail-filter
```

### Startup Rule Check

Before connecting to any account, the filter checks the rules of every enabled account (accounts with `"mail_enabled": false` are not checked) for unknown match fields, empty matches, and match fields with no values. If it finds any, it logs one `ERROR` line per problem and a summary line, processes no mail, and exits with status 1:

```text
[2026-09-23 16:09:44] [gmail-personal] ERROR: rule #1 'Store promotions to Archive' uses unknown match field 'from_contains'
[2026-09-23 16:09:44] ERROR: 1 rule problem(s) in rules.local.json5; no mail processed (see docs/rule-reference.md)
```

All problems in all accounts are reported in one run, so they can be fixed together. After editing `rules.local.json5`, run the filter once by hand (ideally with `"dry_run": true` on the account) to confirm the check passes. See [Invalid Rules](rule-reference.md#invalid-rules).

## Dry Run

To test an account's rules without changing its mail, set `dry_run` on the account in `accounts.local.json5`:

```json5
"gmail-personal": {
  "imap_host": "imap.gmail.com",
  "username": "your-address@gmail.com",
  "password": "...",
  "dry_run": true
}
```

The filter still connects, searches, fetches, and matches, and logs one line per message, but moves, deletes, and marks nothing and never expunges. Each message line ends in ` [DRY_RUN]`, and the account logs `dry_run is true; no changes will be made on the server` when it connects.

Dry run is set per account, so one account can be tried out while the others keep filtering. It is particularly useful for a new account and after changing rules. Remove the setting (or set it to `false`) to filter for real; see [`dry_run`](configuration.md#dry_run).

## Development Logging

The script also contains:

```python
DEV_LOGS = False
```

When enabled, additional diagnostic logging is emitted while matching messages.

This is intended for troubleshooting rather than normal operation.

## Folder Inspection

`list_folders.py` can inspect the mailboxes available on a configured account.

Use it when writing `folders.local.json5`. Each mapping value must be the mailbox's exact IMAP name. That name can differ from what a mail client displays, because it includes the server's hierarchy delimiter and any prefix, such as `INBOX.Services.Payments` or `[Gmail]/Promotions`. Copy those names from the "Extracted mailbox names" section of this tool's output.

It uses the account information from `accounts.local.json5`, loaded from the directory containing `list_folders.py`, regardless of the current working directory.

Usage:

```bash
python list_folders.py gmail-personal
```

The account ID must be a key in `accounts.local.json5`.

The utility prints both the raw IMAP folder listing and extracted mailbox names.

## Cron

The intended deployment model is a cron job that runs the filter periodically.

Example:

```cron
*/5 * * * * /usr/bin/python3 /home/pi/imap-mail-filter/imap_mail_filter.py >> /home/pi/.local/state/imap-mail-filter/imap_mail_filter.log 2>&1
```

This causes the filter to run every five minutes. The log goes in its own folder (see [Logging](#logging)), which must exist first.

Runs can be spread out at night, when mail matters less, with one line per schedule:

```cron
# every 5 minutes from 6:00 to 21:55
*/5 6-21 * * * /usr/bin/python3 /home/pi/imap-mail-filter/imap_mail_filter.py >> /home/pi/.local/state/imap-mail-filter/imap_mail_filter.log 2>&1

# every 30 minutes from 22:00 to 5:30
*/30 22-23,0-5 * * * /usr/bin/python3 /home/pi/imap-mail-filter/imap_mail_filter.py >> /home/pi/.local/state/imap-mail-filter/imap_mail_filter.log 2>&1
```

For macOS (cron or launchd) and Windows (Task Scheduler), see [Installation](installation.md).

If any account uses `password_env`, the cron line must also load the variables; see [Passwords from Environment Variables](#passwords-from-environment-variables).

If `rules.local.json5` has an invalid rule, every cron run logs the startup rule errors and exits with status 1 without processing mail, until the rules are fixed.

## Logging

The application logs to standard output.

A cron deployment redirects that output to a file, `imap_mail_filter.log`. On Linux and the Raspberry Pi, keep it in `~/.local/state/imap-mail-filter/`, the usual place for a program's own logs when it runs as your user: it needs no root, and it keeps the log out of the checkout, where a fresh clone or `git clean` would lose it. Create the folder once:

```bash
mkdir -p ~/.local/state/imap-mail-filter
```

If the folder is missing, the shell can't open the log, so the filter doesn't run, and the error goes to cron's mail, not to the log. The [Quick Start](../README.md#7-run-it-on-a-schedule) keeps the log in the checkout instead, which also works: `*.log` is ignored by Git.

Log entries include timestamps and account/rule context.

Examples of operational conditions that are logged include:

- connection
- invalid rules (unknown match fields, empty matches) found by the startup rule check
- folder validation warnings
- failed IMAP searches
- failed message fetches
- rule matches
- failed mailbox copies
- missing folder mappings
- move/trash/delete actions
- no matching rule

## Log Format

Every processed message produces exactly one line, written after its outcome is known, so a line never reports a destination the message did not reach.

```text
[timestamp] [account] MATCHED #uid RULE='rule name' SUBJECT='subject' <outcome><extras>
[timestamp] [account] NO_RULE #uid SUBJECT='subject'
[timestamp] [account] IGNORED #uid SUBJECT='subject'
```

- `[...]` is used only for the timestamp and the account ID.
- `MATCHED` means a rule, or `catch_all` (`RULE='<catch_all>'`), matched. `NO_RULE` means nothing matched and there is no catch-all. `IGNORED` means the message was made by the live IMAP tests (it has an `X-Imap-Mail-Filter-Test` header), so no rule was evaluated and the message is unchanged; see [Testing](development.md#testing).
- `#uid` is the message's IMAP UID. A UID is unique only within one account's mailbox, so combine it with the account when searching across accounts.
- `RULE` and `SUBJECT` are printed as-is; quotes inside them are not escaped.

The outcome is one of:

| Outcome | Meaning |
|---|---|
| `→ <mailbox>` | moved to that mailbox |
| `→ Trash (<mailbox>)` | trashed to that mailbox |
| `DELETED` | marked `\Deleted`; removed by the end-of-run `EXPUNGE` |
| `MARKED_READ` | `mark_read` was the only action; the message stays in `INBOX` |
| `NO_ACTION` | the rule has no actions; the message is unchanged |
| `SKIPPED: <reason>` | nothing was attempted; the message is unchanged |
| `FAILED: <reason>; <where the message was left>` | an IMAP step was refused |

The arrow `→` appears only when the message went somewhere. A `FAILED` outcome always ends by saying where the message was left, for example `left in INBOX` or `copied to 'GitHub', still in INBOX`, plus ` (marked read)` if `mark_read` had already been applied.

Extras follow the outcome:

- ` (mark_read)`: the message was also marked read. ` (mark_read FAILED: status=NO)` means the main action succeeded but `\Seen` could not be set.
- ` [DRY_RUN]`: the account has `"dry_run": true`, so nothing on the server was changed. This is added to every per-message line of a dry-run account.

Examples:

```text
[2026-09-23 12:21:50] [gmail-personal] MATCHED #16186 RULE='Store promotions' SUBJECT='Weekend sale: 20% off everything' → Trash ([Gmail]/Trash)
[2026-09-23 12:21:50] [gmail-personal] MATCHED #16201 RULE='GitHub mail' SUBJECT='New issue' → GitHub (mark_read)
[2026-09-23 12:21:50] [gmail-personal] NO_RULE #16159 SUBJECT='Lunch on Friday?'
[2026-09-23 12:21:50] [aerospace.biz] MATCHED #5120 RULE='Newsletters' SUBJECT='Spring newsletter' FAILED: copy to 'INBOX.Newsletters' refused (status=NO); left in INBOX
```

Lines that are not about a single matched message keep an account-level form, for example:

```text
[2026-09-23 12:21:50] [newsletters] mail_enabled is false; skipping account
[2026-09-23 12:21:50] [gmail-personal] dry_run is true; no changes will be made on the server
[2026-09-23 12:21:50] [gmail-personal] ERROR: capability query failed; move and trash actions will be skipped
[2026-09-23 12:21:50] [gmail-personal] Trash mailbox: '[Gmail]/Trash' (server \Trash)
[2026-09-23 12:21:50] [gmail-personal] ERROR: fetch #16159 failed (status=NO)
[2026-09-23 12:21:50] [gmail-personal] ERROR: rule #1 'GitHub mail' uses unknown match field 'from_contains'
```

The startup rule check ends with one line that has no account, because it covers the whole configuration:

```text
[2026-09-23 12:21:50] ERROR: 1 rule problem(s) in rules.local.json5; no mail processed (see docs/rule-reference.md)
```

Rule configuration warnings are written as `Warning: RULE='name' ...`.

Useful searches:

```bash
grep MATCHED imap_mail_filter.log              # every match
grep NO_RULE imap_mail_filter.log              # unmatched messages
grep IGNORED imap_mail_filter.log              # live-test messages left alone
grep -E 'FAILED|ERROR' imap_mail_filter.log    # anything that needs attention
grep 'SKIPPED' imap_mail_filter.log            # matches that could not be acted on
grep "RULE='GitHub mail'" imap_mail_filter.log # one rule
grep '#16186 ' imap_mail_filter.log            # one message
```

No other line contains the word `MATCHED`.

## Log Rotation

The historical deployment configuration uses `logrotate`.

Example:

```text
/home/pi/.local/state/imap-mail-filter/imap_mail_filter.log {
    su pi pi
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    copytruncate
}
```

This should be installed under:

```text
/etc/logrotate.d/imap-mail-filter
```

Keep the `su pi pi` line (use your own user and group if not `pi`). It makes
logrotate rotate the file as the owning user rather than root. It is required
when the log's folder is group- or world-writable (for example, created
under a `umask` of `002`): without it, logrotate silently skips the file with
"parent directory has insecure permissions". Check with
`sudo logrotate -d /etc/logrotate.d/imap-mail-filter`.

## Credentials

Credentials are stored in `accounts.local.json5`, either directly as `password` or indirectly as `password_env`, the name of an environment variable holding the password (see [`password_env`](configuration.md#password_env)).

`accounts.local.json5` should remain outside Git while it holds any literal `password`. Use `accounts.example.json5` for the shareable configuration structure.

### Passwords from Environment Variables

With `password_env`, keep the variables in a secrets file readable only by the user that runs the filter:

```bash
# /home/pi/.config/imap-mail-filter/secrets.env
export IMAP_MAIL_FILTER_PW_GMAIL_PERSONAL='app-password-here'
export IMAP_MAIL_FILTER_PW_AEROSPACE='app-password-here'
```

```bash
chmod 600 /home/pi/.config/imap-mail-filter/secrets.env
```

Cron does not read shell startup files, so source the secrets file in the cron line:

```cron
*/5 * * * * . /home/pi/.config/imap-mail-filter/secrets.env && /usr/bin/python3 /home/pi/imap-mail-filter/imap_mail_filter.py >> /home/pi/.local/state/imap-mail-filter/imap_mail_filter.log 2>&1
```

The variable of an account with `"mail_enabled": false` is not checked, so it need not be set.

Source the same file before running `imap_mail_filter.py` or `list_folders.py` by hand.

Or put both in a wrapper script, which cron and runs by hand share. Keep it outside the checkout, for example as `/home/pi/bin/imap-mail-filter.sh`, so `git status` in the checkout stays clean:

```bash
#!/bin/bash

cd /home/pi/imap-mail-filter || exit 1

# load the passwords for password_env; stop if the file is missing
. /home/pi/.config/imap-mail-filter/secrets.env || exit 1

exec /usr/bin/python3 imap_mail_filter.py
```

```bash
chmod +x /home/pi/bin/imap-mail-filter.sh
```

The `|| exit 1` stops the run with the shell's own error, such as `No such file or directory`, instead of the filter's errors for unset variables. The cron line then calls the script, and keeps the log redirection:

```cron
*/5 * * * * /home/pi/bin/imap-mail-filter.sh >> /home/pi/.local/state/imap-mail-filter/imap_mail_filter.log 2>&1
```

A process's environment is readable only by its own user and root, so this is as private as a `chmod 600` `accounts.local.json5`, not more. The gain is that the secrets sit in one small file outside the project, and the password manager remains their master copy. Prefer application-specific passwords, which can be revoked individually.

## Live-System Safety

This program operates on real mail.

Before changing filtering behavior:

1. Commit the current known-good implementation.
2. Make one logical change.
3. Test with `"dry_run": true` on the account.
4. Run against a controlled set of messages if possible.
5. Verify logs and resulting mail placement.
6. Commit the verified change.

Avoid combining unrelated IMAP behavior changes into a single checkpoint.

This is especially important for IMAP behavior changes such as the Gmail move and Trash handling in ADR 006.

## Recovery

The current deployed implementation should be treated as a known-good baseline.

Before behavior-changing work, create a Git checkpoint.

A useful development sequence is:

```text
known-good baseline
       ↓
commit
       ↓
UID migration
       ↓
test
       ↓
commit
       ↓
Gmail move and Trash behavior
       ↓
test
       ↓
commit
```

This keeps the live system reversible at each stage.
