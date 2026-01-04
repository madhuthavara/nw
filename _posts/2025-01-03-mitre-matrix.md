---
layout: post
title: "MITRE ATT&CK Matrix - Enterprise Overview"
date: 2025-01-03
categories: [security, mitre, threat-modeling]
---
References:
Mitre ATT&CK Framework - https://attack.mitre.org/ 

## ATT&CK Matrix for Enterprise

The MITRE ATT&CK framework categorizes adversary behavior into tactics and techniques.

## Reconnaissance

**Focus:**  
Who is the target? What systems, people, and defenses exist?

**Common Techniques:**
- Active Scanning
- Gather Victim Identity Information
- Gather Victim Network Information
- Gather Victim Organization Information
- Search Open Websites/Domains
- Phishing for Information

## Resource Development

**Focus:**  
Prepare infrastructure and capabilities for the attack.

**Common Techniques:**
- Acquire Infrastructure
- Develop Capabilities
- Establish Accounts
- Obtain Capabilities

## Initial Access

**Focus:**  
Gain an initial foothold in the target environment.

**Common Techniques:**
- Phishing
- Drive-by Compromise
- Exploit Public-Facing Application
- Valid Accounts
- Supply Chain Compromise

## Execution

**Focus:**  
Run malicious code on a system.

**Common Techniques:**
- Command and Scripting Interpreter
- PowerShell
- Windows Management Instrumentation (WMI)
- User Execution
- Scheduled Task / Job

## Persistence

**Focus:**  
Maintain access across reboots, logouts, or system changes.

**Common Techniques:**
- Boot or Logon Autostart Execution
- Account Manipulation
- Create or Modify System Process
- Registry Run Keys / Startup Folder
- Scheduled Task

## Privilege Escalation

**Focus:**  
Gain higher-level permissions on a system.

**Common Techniques:**
- Exploitation for Privilege Escalation
- Abuse Elevation Control Mechanism
- Access Token Manipulation
- Sudo and Sudo Caching

## Defense Evasion

**Focus:**  
Avoid detection and bypass security controls.

**Common Techniques:**
- Obfuscated Files or Information
- Disable or Modify Security Tools
- Masquerading
- Rootkit
- Modify Registry

## Credential Access

**Focus:**  
Steal credentials such as usernames, passwords, and tokens.

**Common Techniques:**
- Brute Force
- Credential Dumping
- OS Credential Dumping
- Input Capture (Keylogging)
- Man-in-the-Middle
  
## Discovery

**Focus:**  
Understand the internal environment and available resources.

**Common Techniques:**
- Account Discovery
- Network Service Discovery
- System Information Discovery
- Process Discovery
- Permission Groups Discovery

## Lateral Movement

**Focus:**  
Move from one system to another within the network.

**Common Techniques:**
- Remote Services
- Pass the Hash
- Pass the Ticket
- Exploitation of Remote Services

## Collection

**Focus:**  
Gather data of interest from compromised systems.

**Common Techniques:**
- Data from Information Repositories
- Screen Capture
- Clipboard Data
- Email Collection
- Archive Collected Data

## Command and Control

**Focus:**  
Communicate with attacker-controlled infrastructure.

**Common Techniques:**
- Application Layer Protocol
- Web Protocols
- Encrypted Channel
- Proxy
- Domain Generation Algorithms (DGA)

## Exfiltration

**Focus:**  
Transfer stolen data out of the victim environment.

**Common Techniques:**
- Exfiltration Over Command and Control Channel
- Exfiltration Over Web Services
- Automated Exfiltration
- Data Transfer Size Limits

## Impact

**Focus:**  
Disrupt operations, destroy data, or achieve final objectives.

**Common Techniques:**
- Data Destruction
- Ransomware
- Network Denial of Service
- Defacement
- Service Stop

## Why ATT&CK Matters

- Common language for defenders
- Maps detection gaps
- Improves threat modeling
- Used in red/blue team operations
