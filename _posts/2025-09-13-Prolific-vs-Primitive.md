---
title: "Prolific vs Primitive: INC Ransomware Battles a Shell Script Malware"
date: 2025-09-13
categories:
  - blog
tags:
  - ransomware
  - comparative analysis
  - cybersecurity
---


**Objective**

I will execute a linux-based ransomware known as INC Ransomware in an isolated virtual environment and then analyze its effects — a process known as Dynamic Malware Analysis. I will then compare it with a script-based malware by analyzing the code, which is referred to as Static Malware Analysis.

The project is divided into three phases:

* Phase 1: Dynamic Analysis of INC Ransomware  
* Phase 2: Static Analysis of a Script-Based Ransomware  
* Phase 3: Comparative Analysis of Both Ransomwares

**Environment Setup**

I set up a virtual machine with Kali Linux installed on it. I ensured that the network adapter was set to Host-Only, meaning it was isolated from both the host machine and the internet.

**Phase 1: Dynamic Analysis of INC Ransomware**

The malicious file is in ELF binary format. I used the srch\_strings utility to scan the entire file and discovered that the binary can accept arguments.

![binary_arguments](/assets/2/1-binary_arguments.png)

I then executed the file with the arguments discovered in the previous step. One of these arguments was \--esxi, which is specifically intended for VMware ESXi environments. Since my test environment was not ESXi, this argument did not work.

![esxi_argument_did_not_execute](/assets/2/2-esxi_argument_did_not_execute.png)

The script encrypted all files in the directory /home/farhan/documents/ and appended .INC to each filename. Additionally, it left INC-README.html and INC\_README.txt files in the directory.

![file_encryption_done](/assets/2/3-file_encryption_done.png)

The contents of /etc/motd were replaced with a ransomware note.

![ransom_note](/assets/2/4-ranson_note.png)

Contents of INC-README.html file:

![INC_readme](/assets/2/5-INC_readme.png)

I attempted to access the onion links provided in the ransom note, but they were no longer reachable.

![onion_links_not_accessible](/assets/2/6-onion_links_not_accessible.png)

However, I was able to locate the INC Ransomware group’s known disclosure page via [ransomware.live](https://www.ransomware.live/group/incransom), as shown in the image below: 

![onion_links_not_accessible](/assets/2/7-INC_disclosure_site.png)

**Phase 2: Static Analysis of a Script-Based Ransomware**

I analyzed a shell script–based malware sample. The exact target environment of this malware is unknown. For the sake of this documentation’s readability, I will refer to this malware as **quicksilver**. 

![quicksilver_file_type](/assets/2/8-quicksilver_file_type.png)

To simplify further, I have divided the code into steps to highlight its gradual spread and operation.

**Step 1:**   
quicksilver searches all volumes to check if the ransom note already exists. If it does, the malware exits. This is done to avoid re-encrypting volumes, which can make decryption more difficult and increase the ransom's leverage.

![check_if_readme_file_exists](/assets/2/9-check_if_readme_file_exists.png)

**Step 2:**   
It connects with the Command and Control (C2) server to retrieve:

* The AES-256 encryption key (later used to encrypt files)  
* A cryptocurrency wallet address to which the ransom must be paid  
* The ransom note content

![connects_to_C2_Server](/assets/2/10-connects_to_C2_Server.png)

**Step 3:**   
If the AES key is not retrieved from the server for any reason, the program exits without performing any encryption. Since my test environment was isolated from the internet, the command-and-control server was not accessible, meaning the malware could not continue to full execution. 

![AES_key_retrieval](/assets/2/11-AES_key_retrieval.png)

**Step 4:** This part of the code ensures that the command executes even when the terminal is closed, guaranteeing that the ransom note is placed in every volume. 

![leave_note_in_all_volumes](/assets/2/12-leave_note_in_all_volumes.png)

The malware is hardcoded to target a wide range of file types. It uses OpenSSL — a local tool — to encrypt the files with AES-256 encryption. In cybersecurity terms, this is known as the LotL (Living off the Land) technique, which refers to using built-in system tools for malicious purposes to avoid suspicion or detection.   
A well-known resource for such techniques is [GTFOBins](https://gtfobins.github.io/), a public list of Linux tools that attackers can misuse for things like privilege escalation, file access, or encryption

After execution, quicksilver deletes itself, leaving no signs of its existence.

![deletes_after_execution](/assets/2/13-deletes_after_execution.png)

**Phase 3: Comparative Analysis of Both Ransomwares**

![comparative_table](/assets/2/14-comparative_table.png)

The workings of both malware samples are largely similar because both are ransomware; however, their techniques and target systems differ. INC Ransomware is packed in a binary file, so its code and methods cannot be easily revealed without performing reverse engineering. 

quicksilver, on the other hand, is a script, so its code can be analyzed to fully understand its behavior without execution.

INC Ransomware not only targets Linux systems but also extends its functionality to ESXi environments (VMware virtual machines). This highlights its focus on corporate and enterprise infrastructure, which is a common trend among modern ransomware operators, including INC. By targeting ESXi, attackers can disrupt multiple virtual machines at once, causing maximum impact across an organization. 

quicksilver, on the other hand, has been found in the wild but is not attributed to any known ransomware group, and its specific target environment remains unknown.

**Recommendations**

* Restrict the use of built-in system tools like openssl and curl, reducing the risk of LotL (Living off the Land) attacks.  
* Pay special attention to patching and mitigating Local Privilege Escalation (LPE) vulnerabilities on Linux hosts. (LPE refers to flaws that allow attackers to gain higher-level privileges, such as root, from a normal user account.)  
* Continuously monitor Linux systems, especially those running ESXi, for unusual activity or early signs of compromise.  
* Regularly apply security patches and updates across Linux and ESXi environments to close known vulnerabilities.  
* Enforce strong authentication practices (strong passwords and MFA) to limit the risk of attackers moving laterally once inside a network.