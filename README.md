# Automated Linux User Login Audit & CSV Email Reporter

## Project Overview
A lightweight Bash automation tool designed for Linux servers to monitor daily user login activity, parse raw system logs into an Excel-compatible `.csv` report, and automatically email the report to system administrators on a scheduled basis via `cron`.

## Key Technologies & Tools
* **OS / Shell:** Linux (Ubuntu/Debian), Bash
* **Log Parsing & Text Processing:** `last`, `date`, `grep`, `awk`
* **Automation & Scheduling:** `cron` (`crontab`)
* **Email & SMTP Relay:** `mailutils`, `msmtp` (TLS / Gmail SMTP)

## Architecture & Workflow
1. **Log Extraction:** Queries `/var/log/wtmp` via the `last` command, filters out system reboot entries, and isolates sessions matching the current system date (`date +"%a %b %e"`).
2. **Data Transformation:** Uses `awk` to normalize space-delimited log fields (username, terminal, source IP, timestamp, and session duration) into comma-separated values (`.csv`) with column headers.
3. **Conditional Alerting:** Verifies that login records exist (`-n`) before generating a file, preventing empty spam reports.
4. **Automated Delivery & Cleanup:** Attaches the dated `.csv` file (`-A`) via `mail` routed through `msmtp`, then removes temporary files from `/tmp` (`rm -f`).

## Step-by-Step Setup Guide

### 1. Install Mail & SMTP Utilities
```bash
sudo apt update && sudo apt install -y msmtp msmtp-mta mailutils
```

### 2. Configure SMTP Relay (`~/.msmtprc`)
Create `~/.msmtprc` with restricted permissions (`chmod 600 ~/.msmtprc`):
```text
defaults
auth           on
tls            on
tls_trust_file /etc/ssl/certs/ca-certificates.crt
logfile        ~/.msmtp.log

account        gmail
host           ://gmail.com
port           587
from           your-email@gmail.com
user           your-email@gmail.com
password       YOUR_16_CHAR_APP_PASSWORD

account default : gmail
```
Lock down permissions and link `mailutils` to `msmtp`:
```bash
chmod 600 ~/.msmtprc
echo 'set sendmail="/usr/bin/msmtp -t"' > ~/.mailrc
```

### 3. Deploy the Audit Script (`~/listusers`)
```bash
#!/bin/bash
# Author: Isaac Medugu
# Description: Exports daily user logins to a CSV file and emails it as an attachment

RECIPIENT="admin@example.com"
Today=\$(date +"%a %b %e")
REPORT_FILE="/tmp/user_logins_\$(date +%Y-%m-%d).csv"

MATCHES=\((last \vert{} grep -v "^reboot" \vert{} grep "\)Today")

if [ -n "\$MATCHES" ]; then
    echo "Username,Terminal,IP_Address,Day,Month,Date,Login_Time,Details" > "\$REPORT_FILE"
    echo "\$MATCHES" | awk '{
        user=\$1; tty=\$2; ip=\$3; day=\$4; mon=\$5; date=\$6; time=\$7;
        details="";
        for(i=8; i<=NF; i++) details=details \$i " ";
        gsub(/^[ \t]+|[ \t]+\$/, "", details);
        printf "%s,%s,%s,%s,%s,%s,%s,\"%s\"\n", user, tty, ip, day, mon, date, time, details
    }' >> "\$REPORT_FILE"

    echo "Please find attached today's user login audit report." | \
        mail -s "User Logins Report - \$Today" -A "\(REPORT_FILE" "\)RECIPIENT"
fi

rm -f "\$REPORT_FILE"
```
Make the script executable:
```bash
chmod +x ~/listusers
```

### 4. Schedule with Cron
Open `crontab -e` and schedule the script to run daily at 11:55 PM:
```cron
55 23 * * * /home/isaac/listusers
```

## Sample CSV Output

| Username | Terminal | IP_Address | Day | Month | Date | Login_Time | Details |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| isaac | pts/0 | 192.168.1.10 | Sat | Nov | 21 | 12:45 | still logged in |

## Troubleshooting & Lessons Learned
* **Date Padding Alignment:** Standard `date` output zero-pads single-digit days (`Nov 05`), whereas `last` uses space-padding (`Nov  5`). Resolved by formatting with `date +"%a %b %e"`.
* **Non-Interactive Execution:** Replaced interactive `read` prompts with dynamic system date evaluation so the script runs unattended inside `cron`.
* **Credential Security:** Secured SMTP credentials using `chmod 600` on `~/.msmtprc` and Google App Passwords.
