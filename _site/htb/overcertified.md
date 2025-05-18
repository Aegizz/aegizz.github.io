# Overcertified

![alt text](image.png)

Overcertified is an easy windows box that is accessible through HTB Enterprise.

It's actually pretty interesting as some of the concepts outlined here are pretty realistic.

Let's dive in.

### Recon & Enumeration

Every good HTB Write up starts with very, very good recon. Mine does not. I'll miss some things here and will come back to some more recon later.

First, lets get some eyes.

#### Scan

```bash

┌──(kali㉿kali)-[~]
└─$ nmap 10.129.229.25 -T4 -A
Nmap scan report for 10.129.229.25
Host is up (0.36s latency).
Not shown: 987 filtered tcp ports (no-response)
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2025-05-18 17:11:48Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows 
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ssl/ldap      Microsoft Windows 
1433/tcp open  ms-sql-s      Microsoft SQL Server 2022 16.00.1000.00; RTM
| ms-sql-info: 
|   10.129.229.25:1433: 
|     Version: 
|       name: Microsoft SQL Server 2022 RTM
|       number: 16.00.1000.00
|       Product: Microsoft SQL Server 2022
|       Service pack level: RTM
|       Post-SP patches applied: false
|_    TCP port: 1433
| ms-sql-ntlm-info: 
|   10.129.229.25:1433: 
|     Target_Name: CERTIFIEDDC
|     NetBIOS_Domain_Name: CERTIFIEDDC
|     NetBIOS_Computer_Name: CERTIFIED
|     DNS_Domain_Name: certified.htb
|     DNS_Computer_Name: CERTIFIED.certified.htb
|     DNS_Tree_Name: certified.htb
|_    Product_Version: 10.0.20348
| ssl-cert: Subject: commonName=SSL_Self_Signed_Fallback
| Not valid before: 2025-05-18T16:53:23
|_Not valid after:  2055-05-18T16:53:23
|_ssl-date: 2025-05-18T17:13:22+00:00; +6h59m13s from scanner time.
3268/tcp open  ldap          Microsoft Windows 
3269/tcp open  ssl/ldap      Microsoft Windows 
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0

Host script results:
| smb2-time: 
|   date: 2025-05-18T17:12:41
|_  start_date: N/A
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
|_clock-skew: mean: 6h59m12s, deviation: 0s, median: 6h59m11s

TRACEROUTE (using port 53/tcp)
HOP RTT       ADDRESS
1   385.98 ms 10.10.14.1
2   386.33 ms 10.129.229.25

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 122.83 seconds
                                                
```

Alright, we've got:

- DNS
- **Kerberos Auth (!)**
- RPC (!)
- **SMB (!)**
- **LDAP (!)**
- Kerberos Password Change
- More rpc
- LDAPS
- **MSSQL (!)**
- **WINRM (!)**

#### Enumerating

Now lets enumerate these services. First I set up responder, just in case.

```bash
┌──(kali㉿kali)-[~]
└─$ sudo responder -I tun0 -dw 
                                         __
  .----.-----.-----.-----.-----.-----.--|  |.-----.----.
  |   _|  -__|__ --|  _  |  _  |     |  _  ||  -__|   _|
  |__| |_____|_____|   __|_____|__|__|_____||_____|__|
                   |__|

           NBT-NS, LLMNR & MDNS Responder 3.1.5.0

  To support this project:
  Github -> https://github.com/sponsors/lgandx
  Paypal  -> https://paypal.me/PythonResponder

  Author: Laurent Gaffie (laurent.gaffie@gmail.com)
  To kill this script hit CTRL-C


<Insert yap here>

[+] Listening for events...                                                                                                                                 

```
Alright lets get cracking with some netexec.
```bash
┌──(kali㉿kali)-[~]
└─$ nxc smb 10.129.229.25 -u 'guest' -p '' --shares                                                                
SMB         10.129.229.25   445    CERTIFIED        [*] Windows Server 2022 Build 20348 x64 (name:CERTIFIED) (domain:certified.htb) (signing:True) (SMBv1:False) 
SMB         10.129.229.25   445    CERTIFIED        [-] certified.htb\guest: STATUS_ACCOUNT_DISABLED 
                                                                                                                                                                                                       
┌──(kali㉿kali)-[~]
└─$ nxc smb 10.129.229.25 -u 'anonymous' -p '' --shares 
SMB         10.129.229.25   445    CERTIFIED        [*] Windows Server 2022 Build 20348 x64 (name:CERTIFIED) (domain:certified.htb) (signing:True) (SMBv1:False) 
SMB         10.129.229.25   445    CERTIFIED        [-] certified.htb\anonymous: STATUS_LOGON_FAILURE 
                                                                                                                                                                                                       
┌──(kali㉿kali)-[~]
└─$ nxc ldap 10.129.229.25 -u 'anonymous' -p ''      

                    [-] Error reflecting tables for the LDAP protocol - this means there is a DB schema mismatch
                    [-] This is probably because a newer version of nxc is being run on an old DB schema
                    [-] Optionally save the old DB data (`cp /home/kali/.nxc/workspaces/default/ldap.db ~/nxc_ldap.bak`)
                    [-] Then remove the nxc LDAP DB (`rm -f /home/kali/.nxc/workspaces/default/ldap.db`) and run nxc to initialize the new DB
                                                                                                                                                                                                       
┌──(kali㉿kali)-[~]
└─$ rm -f /home/kali/.nxc/workspaces/default/ldap.db
                                                                                                                                                                                                       
┌──(kali㉿kali)-[~]
└─$ nxc ldap 10.129.229.25 -u 'anonymous' -p ''     
[*] Initializing LDAP protocol database
LDAP        10.129.229.25   389    CERTIFIED        [*] Windows Server 2022 Build 20348 (name:CERTIFIED) (domain:certified.htb)
LDAP        10.129.229.25   389    CERTIFIED        [-] certified.htb\anonymous: 
                                                                                                                                                                                                       
┌──(kali㉿kali)-[~]
└─$ nxc ldap 10.129.229.25 -u 'guest' -p ''     
LDAP        10.129.229.25   389    CERTIFIED        [*] Windows Server 2022 Build 20348 (name:CERTIFIED) (domain:certified.htb)
LDAP        10.129.229.25   389    CERTIFIED        [-] certified.htb\guest: STATUS_ACCOUNT_DISABLED
                                                                                                                                                                                                       
┌──(kali㉿kali)-[~]
└─$ nxc ldap 10.129.229.25 -u '' -p ''      
LDAP        10.129.229.25   389    CERTIFIED        [*] Windows Server 2022 Build 20348 (name:CERTIFIED) (domain:certified.htb)
LDAP        10.129.229.25   389    CERTIFIED        [+] certified.htb\: 
                                                                                                                                                                                                       
┌──(kali㉿kali)-[~]
└─$ nxc smb 10.129.229.25 -u '' -p '' --shares 
SMB         10.129.229.25   445    CERTIFIED        [*] Windows Server 2022 Build 20348 x64 (name:CERTIFIED) (domain:certified.htb) (signing:True) (SMBv1:False) 
SMB         10.129.229.25   445    CERTIFIED        [+] certified.htb\: 
SMB         10.129.229.25   445    CERTIFIED        [-] Error enumerating shares: STATUS_ACCESS_DENIED


```
Hmmm, okay seems that we can do some stuff with LDAP.
```bash
┌──(kali㉿kali)-[~]
└─$ nxc ldap 10.129.229.25 -u '' -p '' -M user-desc
LDAP        10.129.229.25   389    CERTIFIED        [*] Windows Server 2022 Build 20348 (name:CERTIFIED) (domain:certified.htb)
LDAP        10.129.229.25   389    CERTIFIED        [+] certified.htb\: 
USER-DESC   10.129.229.25   389    CERTIFIED        User: ldapusr - Description: Ldap Service Account Password: ldapisfun
USER-DESC   10.129.229.25   389    CERTIFIED        Saved 2 user descriptions to /home/kali/.nxc/logs/UserDesc-CERTIFIED.certified.htb-20250518_150831.log
                                                                                                                                                                                                       
┌──(kali㉿kali)-[~]
└─$ nxc ldap 10.129.229.25 -u '' -p '' -M get-userPassword
LDAP        10.129.229.25   389    CERTIFIED        [*] Windows Server 2022 Build 20348 (name:CERTIFIED) (domain:certified.htb)
LDAP        10.129.229.25   389    CERTIFIED        [+] certified.htb\: 
GET-USER... 10.129.229.25   389    CERTIFIED        [+] Found following users: 
                                                                                                                                                                                                       
┌──(kali㉿kali)-[~]
└─$ nxc ldap 10.129.229.25 -u '' -p '' -M adcs            
LDAP        10.129.229.25   389    CERTIFIED        [*] Windows Server 2022 Build 20348 (name:CERTIFIED) (domain:certified.htb)
LDAP        10.129.229.25   389    CERTIFIED        [+] certified.htb\: 
ADCS        10.129.229.25   389    CERTIFIED        [*] Starting LDAP search with search filter '(objectClass=pKIEnrollmentService)'
                                                                                                                                                                                                                                                                 
┌──(kali㉿kali)-[~]
└─$ nxc ldap 10.129.229.25 -u '' -p '' -M daclread
DACLREAD                                            [-] Select an option, example: -M daclread -o TARGET=Administrator ACTION=read
                                                                                                                                                                                                       
┌──(kali㉿kali)-[~]
└─$ nxc ldap 10.129.229.25 -u '' -p '' -M maq     
LDAP        10.129.229.25   389    CERTIFIED        [*] Windows Server 2022 Build 20348 (name:CERTIFIED) (domain:certified.htb)
LDAP        10.129.229.25   389    CERTIFIED        [+] certified.htb\: 
MAQ         10.129.229.25   389    CERTIFIED        [*] Getting the MachineAccountQuota
MAQ         10.129.229.25   389    CERTIFIED        MachineAccountQuota: 10
                                                                                                                                                                                                       
┌──(kali㉿kali)-[~]
└─$ nxc ldap 10.129.229.25 -u '' -p '' -M pre2k
LDAP        10.129.229.25   389    CERTIFIED        [*] Windows Server 2022 Build 20348 (name:CERTIFIED) (domain:certified.htb)
LDAP        10.129.229.25   389    CERTIFIED        [+] certified.htb\: 
PRE2K       10.129.229.25   389    CERTIFIED        [+] Successfully obtained TGT for 0 pre-created computer accounts. Saved to /home/kali/.nxc/modules/pre2k/ccache
                                                                                                                                        
```

Okay, so not thats pretty lame. But... after checking that user descriptions we did find something. 

![alt text](<Screenshot from 2025-05-18 21-55-56.png>)

Heyyyy, there we go! We got a user account.

ldapusr:ldapisfun

Alright, I'm going to save you a bit of scrolling time. None of our other commands above were helped by this user account. Lets try some privilege escalation.

Lets start with my go-to Certipy-ad. Certipy is like 

