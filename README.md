- Project: https://github.com/linuxcaffe/tw-daily-email
- Issues:  https://github.com/linuxcaffe/tw-daily-email/issues

# tw-daily-email

A daily Taskwarrior report delivered to your inbox every morning — one script,
one config file, one systemd timer. Plain text to start; designed to grow.

---

## TL;DR

- Sends any named Taskwarrior report by email each morning
- Plain text, zero dependencies beyond `msmtp`
- Scheduled via systemd user timer with `Persistent=true` (catches missed sends)
- Extensible: add weather, calendar, or any daily section as a shell function
- Installable via awesome-taskwarrior: `tw --install daily-email`

---

## Files installed

| File | Location |
|------|----------|
| `tw-daily-email` | `~/.task/scripts/tw-daily-email` |
| `daily-email.rc` | `~/.task/config/daily-email.rc` |
| `tw-daily-email.service` | `~/.config/systemd/user/` |
| `tw-daily-email.timer` | `~/.config/systemd/user/` |

---

## Prerequisites

### 1. Install msmtp

```bash
sudo apt install msmtp msmtp-mta
```

`msmtp-mta` provides a `sendmail`-compatible symlink — useful for other tools
that rely on it.

### 2. Gmail App Password

Gmail requires an App Password for SMTP access from scripts:

1. Enable 2-Step Verification on your Google Account (required)
2. Go to Google Account → Security → 2-Step Verification → App passwords
3. Create a password named `taskwarrior` (or similar)
4. Copy the 16-character password

If App passwords does not appear, use the search bar at
`myaccount.google.com` and search for "app passwords".

### 3. Configure msmtp

Create `~/.msmtprc` and set permissions to 600:

```
defaults
auth           on
tls            on
tls_trust_file /etc/ssl/certs/ca-certificates.crt
logfile        ~/.msmtp.log

account        gmail
host           smtp.gmail.com
port           587
from           your@gmail.com
user           your@gmail.com
password       xxxx xxxx xxxx xxxx

account default : gmail
```

```bash
chmod 600 ~/.msmtprc
```

Do not share or commit this file — it contains your App Password.

---

## Configuration

Edit `~/.task/config/daily-email.rc`:

```ini
# Task report to send — any named report, with optional rc. overrides
# Examples: next   list   rc.context=none next   due:today
daily.report=next

# Recipient email address (required)
daily.email=you@gmail.com

# Email subject prefix — date is appended automatically
daily.subject=Task Report
```

The `daily.report` value is passed directly to `task`, so any valid report
name or combination of `rc.` overrides works:

```ini
daily.report=rc.context=none next     # ignore active context
daily.report=list                     # full task list
daily.report=due:today                # only today's tasks
```

---

## Usage

```bash
tw-daily-email              # send using configured report
tw-daily-email list         # override report on the fly
tw-daily-email "rc.context=none next"
```

---

## Systemd timer

The installer places the service and timer units in `~/.config/systemd/user/`.
After installation, enable the timer:

```bash
systemctl --user daemon-reload
systemctl --user enable --now tw-daily-email.timer
```

Check status and history:

```bash
systemctl --user list-timers tw-daily-email.timer   # next scheduled send
journalctl --user -u tw-daily-email                 # full run history
systemctl --user status tw-daily-email              # last run result
```

### Adjusting the schedule

Edit `~/.config/systemd/user/tw-daily-email.timer`:

```ini
[Timer]
OnCalendar=Mon..Sun 07:30     # every day at 07:30
# OnCalendar=Mon..Fri 07:30   # weekdays only
Persistent=true               # catch up if machine was off at send time
RandomizedDelaySec=5min       # avoid hammering SMTP on exact minute
```

After editing:

```bash
systemctl --user daemon-reload
systemctl --user restart tw-daily-email.timer
```

---

## Extending with additional sections

The script is structured around composable `section_*` functions. To add a
new section, define a function that writes to stdout and call it from
`build_body()`:

```bash
section_weather() {
    echo "=== weather ==="
    echo ""
    curl -s "wttr.in/?format=3" 2>/dev/null || true
}

build_body() {
    date '+%A, %d %B %Y'
    echo ""
    section_tasks
    echo ""
    section_weather   # add here
}
```

Future sections might include calendar events, habit tracking, or a daily
note from `~/.task/notes/`.

---

## Installation via awesome-taskwarrior

```bash
tw --install daily-email
```

The installer:
- Downloads and installs all files
- Creates `~/.task/config/daily-email.rc` if not already present
- Installs systemd units and runs `daemon-reload`
- Prints post-install setup instructions

**After installation you still need to:**
1. Set up `~/.msmtprc` with your Gmail credentials (see above)
2. Edit `~/.task/config/daily-email.rc` and set `daily.email`
3. Enable the timer: `systemctl --user enable --now tw-daily-email.timer`

---

## Metadata

- Version: 0.1.0
- License: MIT
- Language: Bash
- Requires: msmtp, Taskwarrior 2.6.2
- Platforms: Linux (systemd)
