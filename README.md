# TryHackMe — ABC Company (b3dr0ck) Write‑Up

Difficulty: Medium

Category: CTF / Linux / TLS / Privilege Escalation

Skills Used: Scanning, TLS connections, certificate abuse, decoding, hash cracking

Scenario Overview

Barney is setting up a web server for Abbadabba Broadcasting Company (ABC) and tries to secure services using TLS certificates. Unfortunately, the setup is badly configured, exposing sensitive services.

We must:

Gain initial access

Move from Barney ➜ Fred

Escalate privileges

Capture all flags

This challenge shows how misconfigured security tools can be more dangerous than no security at all.

#Step 1 — Reconnaissance

First, we scan the machine to discover open services.

nmap -sC -sV -p- <TARGET_IP>

| Port  | Service Description        |
| ----- | -------------------------- |
| 80    | Web server (redirects)     |
| 4040  | TLS web service            |
| 9009  | Certificate helper service |
| 54321 | Secure TLS login service   |

Ports 9009 and 54321 are unusual and worth investigating.

#Step 2 — Interacting with the Certificate Helper

socat TCP:<TARGET_IP>:9009 -
The service responds interactively.

Typing:

help
certificate
key

returns:

A client certificate

A private key

These files act like digital credentials.

Step 3 — Using TLS Credentials

We use the certificate and key to connect to the TLS login service:

socat stdio ssl:<TARGET_IP>:54321,cert=certificate,key=key,verify=0


Inside the service:

help


We receive:

Password hint: d1ad7c0a3805955a35eb260dab4180dd
(user = Barney Rubble)


This string is actually Barney’s password.

👤 Step 4 — Accessing Barney
ssh barney@<TARGET_IP>


After login:

cat barney.txt

🏁 Barney Flag
THM{f05780f08f0eb1de65023069d0e4c90c}

🔺 Step 5 — Checking Sudo Privileges
sudo -l


Barney can run:

/usr/bin/certutil


This tool manages certificates and runs with elevated privileges — a major weakness.

👤 Step 6 — Discovering Fred

We find another user:

/home/fred


But we don’t have access.

Using certutil, we list available certificates:

sudo certutil ls


We find certificate files for Fred.

We extract them:

sudo certutil -a fred.csr.pem


Now we possess Fred’s TLS credentials.

🔑 Step 7 — Logging in as Fred

We reuse the TLS login service:

socat stdio ssl:<TARGET_IP>:54321,cert=fred_cert,key=fred_key,verify=0


We receive:

Password hint: YabbaDabbaD0000!


Switch users:

su fred


Then:

cat fred.txt

🏁 Fred Flag
THM{08da34e619da839b154521da7323559d}

👑 Step 8 — Fred’s Privileges
sudo -l


Fred can run:

base32 /root/pass.txt
base64 /root/pass.txt


This allows us to read a root file in encoded form.

🔓 Step 9 — Decoding the Root Secret
sudo base32 /root/pass.txt


Then decode:

echo "<output>" | base32 -d
echo "<result>" | base64 -d


We obtain:

a00a12aad6b7c16bf07032bd05a31d56


This is an MD5 hash.

Cracking it reveals:

flintstonesvitamins

🧨 Step 10 — Root Access
su root


Enter the cracked password.

cat /root/root.txt

🏁 Root Flag
THM{de4043c009214b56279982bf10a661b7}

💥 Key Lessons
Weakness	Impact
Exposed certificate helper	Anyone could get login credentials
TLS login revealed passwords	Credential leakage
certutil allowed via sudo	Cross-user compromise
Encoded root password	Easily reversible
Weak hash	Quickly cracked

This machine was compromised due to poor security configuration, not advanced exploits.

🎯 Conclusion

Attack chain:

Scanned services

Retrieved TLS certificates

Logged in as Barney

Abused certutil to access Fred

Decoded root secret

Cracked hash

Gained root

Full system compromise achieved.
