# Install-and-Understand-ELK-Stack


**Author:** Dashane James  
**Lab Environment:** [e.g. VMware Workstation | Kali Linux | Metasploitable 2]  
**Purpose:** [The goal of this repository is to use Nmap scripts for targeted vulnerability checks, bridging network scanning and vulnerability assesment.]
**Status:** 🔵 Completed

---

## 📋 Overview

[This repository documents combining the network scanning of Nmap with the task of vulnerabIlity assessments. This creates a simpler way to both scan a network and identify vulnerabilities simultaneously which is very useful for a cybersecurity analyst or penetration tester.]

---

## 🧪 Lab Environment

| Component | Details |
|---|---|
| Hypervisor | VMware Workstation (Host-Only Network) |
| Attacker Machine | Kali Linux 2026.1 — `192.168.79.129` |
| Target Machine | [Metasploitable 2] — `192.168.79.130` |
| Network Type | Host-Only (isolated, no internet exposure) |
| Host OS | Windows 11 — ASUS Vivobook 14 |

> ⚠️ **Note:** All activity was performed in a controlled, isolated lab environment against deliberately vulnerable machines. No unauthorized access to live networks was performed.

---

## 🛠️ Tools Used

- **Nmap** — Used for both standard scanning and running NSE (Nmap Scripting Engine) scripts to detect and, in one case, actively exploit vulnerabilities
- **NSE (Nmap Scripting Engine)** — Ran default (`-sC`), targeted (`ftp-anon`, `smb-vuln*`), and broad (`vuln`) script categories against the target
- **NVD (National Vulnerability Database)** — Referenced to research CVE details, CVSS scores, and attack vectors for confirmed vulnerabilities

---

## 📁 Repository Structure

```
[Repo-Name]/
├── README.md
├── [folder-1]/
│   ├── [file1.txt]        # Description of file
│   └── [file2.md]         # Description of file
├── [folder-2]/
│   ├── [file3.md]         # Description of file
│   └── [file4.txt]        # Description of file
└── reports/
    └── [final-report.md]  # Full assessment report
```

---

## 🔬 Tasks / Assessments Performed

### 1. [Run default scripts against key Metasploitable ports.]
For this task I ran a Nmap -sC script scan on the target. By doing this it runs a default set of scripts that go beyond basic port scanning to actually probe and interact with the services running on open ports, collecting much more valuable information than a simple open/closed" or "version number" result. 

# Command used
[sudo nmap -sC -Pn --disable-arp-ping -n -p 21,22,80,445 192.168.79.130]


# Output
-sC runs Nmap's default script set, which is a specific collection of scripts that go beyond basic sanning to actively interact with each service, testing things like default credentials and known weak configurations. Against Metasploitable, this revealed several notable results on ports 21,22,80, and 445.

<img width="325" height="260" alt="Screenshot 2026-07-25 165902" src="https://github.com/user-attachments/assets/d6d7347b-8354-4f74-aefc-4dd39ccf9d6f" />

<img width="326" height="257" alt="Screenshot 2026-07-25 165937" src="https://github.com/user-attachments/assets/b4b80872-4c9e-4b54-a036-fe37bfeb5d13" />

**Findings:**

Port 21 (FTP): Anonymous logins are allowed, meaning no credentials are required to access the FTP server. The scan also confirmed both control and data communications are transmitted in plain text.

Port 22 (SSH): Host key fingerprints(DSA/RSA) were retrieved, useful for host verification but not a vulnerability on its own. 

Port 445 (SMB): Identified as Samba 3.0.20-Debian, confirming this is a Linux/Unix based SMB rather than a Windows based SMB. The scan also flagged "message_signing: disabled (dangerous, but default), a known weakness that leaves SMB traffic vulnerable to tampering and showed the scan was able to authenticate using a guest account.

These results show that -sC surfaces real configuration weaknesses like anonymous access and disabled security features.


### 2. [Check FTP for anonymous login vulnerability]
For this task I use a different type of command to specifically display vulnerabilities for FTP. This allows me to have a more targeted approach for researching vulnerabilities on the target machine.

# Command used
[sudo nmap --script ftp-anon -Pn --disable-arp-ping -n -p 21 192.168.79.130]


# Output
The output displays almost identically the same vulnerabilities found when running the -sC command except it trims the info down to only the indicated port 21. Results of ftp-anon script confirmed: Anonymous FTP login allowed (FTP code 230).

<img width="314" height="142" alt="Screenshot 2026-07-25 191012" src="https://github.com/user-attachments/assets/77048d28-0911-41be-9c80-03ae549e72fb" />


**Findings:**
I found that this command again confirms port 21 accepts anonymous logins with no credentials required, which is obviously a critical misconfiguration.

### 3. [Run SMB vulnerability scripts against port 445]
[I ran a script to search for vulnerabilities in SMB. This is another example of using scripts in Nmap to uncover vulnerabilities on specific ports on the target device] 


# Command used
[sudo nmap --script smb-vuln* -Pn --disable-arp-ping -n -p 445 192.168.79.130]

# Output
The smb-vuln* wildcard ran every available SMB vulnerability-check script against port 445. Two of those checks returned false (MS10-061 and MS10-054). This means the target is not vulnerable to those specific issues. A third script(smb-vuln-regsvc-dos) failed to execute and returned an error rather than a definitive result. 

<img width="323" height="257" alt="Screenshot 2026-07-25 221859" src="https://github.com/user-attachments/assets/751e88a7-650a-44d9-821b-cdcd8fa342af" />


```

**Findings:**

I found that not every script scan turns up a positive or definitive result. This run confirmed the target is not vulnerable to two specific known SMB exploits (MS10-061 and MS10-054), and the other script failed to execute rather than returning a clear answer. This is an important distinction: a "false" result means the vulnerability was tested and ruled out, while this script error means the test was inconclusive and shouldn't be considered safe until further investigation. 



### 4. [Run the vuln category for a broad check]
In this task, I ran a script to identify a broader range of vulnerabilities across the target network. The purpose is to run a different type of script that shows a wider range of outputs unlike the last task which showed only port specific vulnerabilities. 

#Command
[sudo nmap --script vuln -Pn --disable-arp-ping -n 192.168.79.130]

# Output
After reviewing this output I immediately noticed that this output was way more massive than any of the other vulnerability scans that I've run so far. Scripts ran against nearly every open port instead of just one. The most significant result came from port 21 because I noticed that Nmap didn't just show me the vulnerability exists it actually exploited it live and confirmed root access.

<img width="320" height="257" alt="Screenshot 2026-07-25 234932" src="https://github.com/user-attachments/assets/10e2682f-50a8-4eb9-ad84-b1f56f23b7ea" />


**Findings:** 
The Vuln script category confirmed and actively exploited the vsftpd 2.3.4 backdoor (CVE-2011-2523) on port 21, achieving root-level command execution. The script ran the shell command id through the backdoor and received uid=0(root) gid=0(root), which is definitive proof of successful remote access. This was the strongest result of any scan in this project, since it moved beyond detection into actual proof of compromise. Notably, this was one of the only scripts in the scan that performed live exploitation. Most other scripts in the "vuln" category only detect and report whether a vulnerability exists, without attempting to exploit it. However, port 21's script is a special case because the vsftpd 2.3.4 backdoor is a well documented, simple command-injection vulnerability that Nmap's script can reliably trigger and verify with a single command. Most other vulnerability scripts test for more complex or riskier conditions, like crashing a service, so they are written to stop at detection rather than attempt exploitation.

### 5. [Document every vuln NSE finds with its severity.]
For this task I continued to review the vulnerability output that I received in task 4. However, this time I reviewed and  documented each vulnerability that was displayed. I then categorized them by port, service, vulnerability, NSE Script, severity, along with any other notes or important details.


### Output

Port / Service / Vulnerability / NSE Script / Severity / Notes

21/tcp (FTP) - ftp-vsftpd-backdoor (CVE-2011-2523)	🔴 Critical	Live exploitation confirmed. The vulnerable vsFTPd 2.3.4 backdoor allowed remote command execution and root shell access. (CVSS 9.8)

22/tcp (SSH) - Weak SSH cryptographic configuration - 🟡 Medium	Weak or outdated cryptographic algorithms were identified, potentially reducing the security of encrypted communications.

23/tcp (Telnet) - Telnet service detected	🟠 High	Telnet transmits credentials in plaintext, making usernames and passwords susceptible to interception.

80/tcp (HTTP) - Slowloris Denial-of-Service	🟡 Medium	The web server appears susceptible to the Slowloris DoS attack, which can exhaust server connections and deny service to legitimate users.

80/tcp (HTTP) - HTTP TRACE method enabled	🟢 Low	TRACE is enabled, which may aid reconnaissance or cross-site tracing attacks but does not directly compromise the server.

80/tcp (HTTP) - SQL Injection vulnerability	🟠 High	SQL injection was detected and could allow attackers to retrieve, modify, or delete database information.

111/tcp (rpcbind) - RPC information disclosure	🟢 Low	rpcbind exposes information about available RPC services, assisting attacker reconnaissance.

139/tcp (NetBIOS/SMB) - SMB enumeration	🟡 Medium	SMB information disclosure allows attackers to gather host and share information useful for later attacks.

445/tcp (SMB) - smb-vuln-ms10-061 🟢 Not Vulnerable	NSE script returned false, indicating the system is not vulnerable to MS10-061.

445/tcp (SMB) - smb-vuln-ms10-054 🟢 Not Vulnerable	NSE script returned false, indicating the target is not affected by MS10-054.

445/tcp (SMB) - smb-vuln-regsvc-dos	⚪ Inconclusive	The script failed to complete successfully, so the vulnerability status could not be determined.

3306/tcp (MySQL) - MySQL information disclosure	🟡 Medium	Database version and service information were exposed, which may help attackers identify known exploits.

443/tcp (HTTPS) - Logjam (Weak Diffie-Hellman Parameters) 🟡 Medium	Weak Diffie-Hellman parameters reduce TLS security and could allow encrypted communications to be weakened under certain attack scenarios

**Findings:** 

The table above consolidates all of the findings reported during the Nmap --script vuln scan and categorizes each result by severity. The most significant finding was the vsFTPd 2.3.4 backdoor (CVE-2011-2523) on port 21, which was successfully exploited to obtain root-level access, making it the only Critical vulnerability confirmed through live exploitation. Several additional weaknesses were identified, including SQL injection, Telnet running without encryption, Slowloris denial-of-service susceptibility, and weak SSH cryptographic settings, all of which increase the attack surface of the target system. The SMB vulnerability checks also demonstrated that vulnerability scans can produce different outcomes: some scripts confirmed the system was not vulnerable, while others returned inconclusive results because the checks could not be completed. Overall, the scan illustrates the importance of reviewing every NSE script result individually, as findings may represent confirmed vulnerabilities, informational issues, successful mitigations, or inconclusive tests requiring additional investigation.

---

## 📊 Key Findings Summary

| Port/Service | Tool Used | Risk Level | Notes |

| 21/tcp FTP | Nmap (`vuln`, `ftp-anon`) | 🔴 Critical | vsFTPd 2.3.4 backdoor (CVE-2011-2523) — live exploitation confirmed, root access achieved |
| 23/tcp Telnet | Nmap (`vuln`) | 🟠 High | Transmits credentials in plaintext, vulnerable to interception |
| 80/tcp HTTP | Nmap (`vuln`) | 🟠 High | SQL injection vulnerability detected |
| 80/tcp HTTP | Nmap (`vuln`) | 🟡 Medium | Susceptible to Slowloris denial-of-service attack |
| 445/tcp SMB | Nmap (`-sC`, `smb-vuln*`) | 🟡 Medium | Message signing disabled; guest authentication allowed |
| 22/tcp SSH | Nmap (`vuln`) | 🟡 Medium | Weak/outdated cryptographic configuration identified |
| 3306/tcp MySQL | Nmap (`vuln`) | 🟡 Medium | Database version and service info exposed |

**Risk Levels:** 🔴 Critical | 🟠 High | 🟡 Medium | 🟢 Low


---

## 🗺️ MITRE ATT&CK Mapping

| Action Performed | ATT&CK Tactic | Technique ID | Technique Name |
|---|---|---|---|
| Default and targeted script scanning (-sC, ftp-anon, smb-vuln*) | Reconnaissance | T1595 | Active Scanning |
| Anonymous FTP login confirmed | Initial Access | T1078 | Valid Accounts |
| vsFTPd backdoor exploitation (root shell via `id` command) | Execution | T1059 | Command and Scripting Interpreter |
| Root-level access confirmed post-exploitation | Privilege Escalation | T1068 | Exploitation for Privilege Escalation |

---

## 🛡️ Defensive Recommendations

Based on findings, the following remediations would be recommended in a real environment:

1. **vsFTPd 2.3.4 backdoor (Critical)** — Immediately upgrade or replace the FTP service; this version is known to contain an intentional backdoor and should never be used in production.
2. **Anonymous FTP login allowed** — Disable anonymous access entirely; require authenticated accounts for any FTP access.
3. **Telnet enabled (plaintext credentials)** — Disable Telnet and replace with SSH for all remote administration.
4. **SMB message signing disabled** — Enable SMB message signing to prevent tampering and man-in-the-middle attacks on SMB traffic.
5. **SQL injection on port 80** — Implement input validation/parameterized queries on the web application, and consider a WAF as an additional layer of defense.

---

## 📚 CySA+ Exam Relevance

This lab directly maps to the following CompTIA CySA+ (CS0-003) exam domains:

| Domain | Coverage |

| Security Operations (33%) | Hands-on use of Nmap and NSE scripts to perform reconnaissance and identify live services and misconfigurations |
| Vulnerability Management (30%) | Running vulnerability-check scripts (`smb-vuln*`, `vuln`), interpreting results, and cross-referencing CVE/CVSS data from NVD |
| Incident Response (20%) | Recognizing confirmed exploitation (root access via vsFTPd backdoor) as evidence of active compromise |
| Reporting & Communication (17%) | Documenting findings by severity in a structured, readable format for technical and non-technical audiences |

---

## 🔑 Technical Notes

> # Note: not all NSE vulnerability scripts behave the same way — most only detect and report (true/false), while a small number (like ftp-vsftpd-backdoor) actually attempt live exploitation. Always check script documentation or output carefully rather than assuming uniform behavior.
>
> "Always add the -n flag to Nmap scans in this VMware environment to prevent DNS resolution hangs."]
> 
> -sP and -sT contradict eachother (Can't be used together because -sP means just do a ping/host-discovery sweep, skip ports entirely but -sT tells Nmap to do a full TCP connect port scan. These commands contradict eachother.)

```bash

# Any important commands or workarounds
[-sC]
[-Pn]

# Any important commands or workarounds

# Task 1: Run default scripts (-sC) against key ports
sudo nmap -sC -Pn --disable-arp-ping -n -p 21,22,80,445 192.168.79.130

# Task 2: Check FTP for anonymous login specifically
sudo nmap --script ftp-anon -Pn --disable-arp-ping -n -p 21 192.168.79.130

# Task 3: Run all SMB vulnerability scripts against port 445
sudo nmap --script smb-vuln* -Pn --disable-arp-ping -n -p 445 192.168.79.130

# Task 4: Run the full vuln script category across all open ports
sudo nmap --script vuln -Pn --disable-arp-ping -n 192.168.79.130

# Workaround: -p requires a value directly after it — omitting the port number
# (e.g. "-p 192.168.79.130") causes Nmap to misinterpret the target IP as a
# port specification. To scan all ports instead of a specific one, remove
# the -p flag entirely rather than leaving it empty.

# Workaround: always use -n in this lab to prevent DNS resolution hangs,
# since the isolated Host-Only network has no real DNS server.

# Note: --script <name>* (wildcard) runs every NSE script matching that
# name pattern — useful for running a whole category (e.g. smb-vuln*)
# in a single command instead of specifying each script individually.
```

---

## 📌 About This Project

[1-2 sentences about how this fits into your overall portfolio and career goals.]

This repository is part of my broader cybersecurity portfolio demonstrating practical, hands-on vulnerability assessment skills from initial scanning through confirmed exploitation as I work toward a career as a cybersecurity analyst.

This repository is important because here we have built up to the point of having live proof of the actual exploitation by an automated Nmap script for the CVE-2011-2523. In a past NVD task I researched this exact vulnerability with a CVSS score of 9.8 now this has come full circle and we found out how this exact vulnerability is exploited and root access was acheived. 


**Related repositories:**
- Nmap-Host-Discovery-and-Lab-Baseline — Established a baseline of the lab network using ping sweeps and host discovery, documenting the target environment before deeper scanning began
- Nmap-Scan-Types-SYN-vs-TCP-vs-UDP — Compared SYN, TCP connect, and UDP scan types against the target, examining differences in speed, stealth, and reliability
- Service-Version-Detection-and-OS-Fingerprinting — Identified exact software versions on Metasploitable and researched real CVEs tied to them, including the vsFTPd backdoor later exploited in this repository
- Wireshark-Capture-and-Analyze-Traffic — Captured and analyzed live packet traffic, including plaintext credential exposure over Telnet
- TCPDump-CLI-Packet-Capture — Used TCPDump from the command line to capture traffic (including a live Telnet session showing plaintext credential exposure), and demonstrated saving captures to a `.pcap` file for later analysis in Wireshark

---

## 👤 Author

**Dashane James**  
Senior Field Service Technician → Cybersecurity Analyst  
📍 Yonkers, NY  
🎓 B.S. Information Technology — SUNY Canton  
🏆 CompTIA Security+ | CySA+ (In Progress)  
🔗 [GitHub](https://github.com/Dashanejames1) | [Zero Trust Cyber Security Brand](https://www.instagram.com/zerotrust_cybersecurity)

---

*This repository is part of an active portfolio demonstrating hands-on cybersecurity skills. All lab work performed in isolated environments for educational purposes.*
