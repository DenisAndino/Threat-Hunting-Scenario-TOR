# Official [Cyber Range](http://joshmadakor.tech/cyber-range) Project

<img width="400" src="https://github.com/user-attachments/assets/44bac428-01bb-4fe9-9d85-96cba7698bee" alt="Tor Logo with the onion and a crosshair on it"/>

# Threat Hunt Report: Unauthorized TOR Usage
- [Scenario Creation](https://github.com/DenisAndino/Threat-Hunting-Scenario-TOR/blob/main/threat-hunting-scenario-tor-event-creation.md)

## Platforms and Languages Leveraged
- Windows 10 Virtual Machines (Microsoft Azure)
- EDR Platform: Microsoft Defender for Endpoint
- Kusto Query Language (KQL)
- Tor Browser

##  Scenario

Management suspects that some employees may be using TOR browsers to bypass network security controls because recent network logs show unusual encrypted traffic patterns and connections to known TOR entry nodes. Additionally, there have been anonymous reports of employees discussing ways to access restricted sites during work hours. The goal is to detect any TOR usage and analyze related security incidents to mitigate potential risks. If any use of TOR is found, notify management.

### High-Level TOR-Related IoC Discovery Plan

- **Check `DeviceFileEvents`** for any `tor(.exe)` or `firefox(.exe)` file events.
- **Check `DeviceProcessEvents`** for any signs of installation or usage.
- **Check `DeviceNetworkEvents`** for any signs of outgoing connections over known TOR ports.

---

## Steps Taken

### 1. Searched the `DeviceFileEvents` Table

Searched for any file that had the string "tor" in it and discovered what looks like the user "employee" downloaded a TOR installer, did something that resulted in many TOR-related files being copied to the desktop, and the creation of a file called `tor-shopping-list.txt` on the desktop at `6/26/2025, 8:35:12.641 AM
            `. These events began at `6/26/2025, 7:25:38.500 AM`.

**Query used to locate events:**

```kql
DeviceFileEvents
| where DeviceName == "denis-threat-hu"
| where InitiatingProcessAccountName == "dluffyy"
| where FileName contains "tor"
| where Timestamp >= datetime('2025-06-26T07:25:38.5005684Z')
| order by Timestamp desc
| project Timestamp, DeviceName, ActionType, FileName, FolderPath, SHA256, Account = InitiatingProcessAccountName

```
<img width="1336" height="249" alt="image" src="https://github.com/user-attachments/assets/58b26ac6-9807-41d4-b5fc-cfd9e321c952" />


---

### 2. Searched the `DeviceProcessEvents` Table

Searched for any `ProcessCommandLine` that contained the string "tor-browser-windows-x86_64-portable-14.0.1.exe". Based on the logs returned, at '6/26/2025, 7:59:56.920 AM`, an employee on the "denis-threat-hu" device ran the file `tor-browser-windows-x86_64-portable-14.0.1.exe` from their Downloads folder, using a command that triggered a silent installation.

**Query used to locate event:**

```kql

DeviceProcessEvents
| where DeviceName == "denis-threat-hu"
| where ProcessCommandLine contains "tor-browser-windows-x86_64-portable-14.5.4.exe"
| project Timestamp, DeviceName, AccountName, ActionType, FileName, FolderPath, SHA256, ProcessCommandLine

```
<img width="2371" height="175" alt="image" src="https://github.com/user-attachments/assets/ee8cf05d-cd42-4416-818d-d17e912718d4" />

---

### 3. Searched the `DeviceProcessEvents` Table for TOR Browser Execution

Searched for any indication that user "employee" actually opened the TOR browser. There was evidence that they did open it at `6/26/2025, 8:01:06.706 AM`. There were several other instances of `firefox.exe` (TOR) as well as `tor.exe` spawned afterwards.

**Query used to locate events:**

```kql
DeviceProcessEvents
| where DeviceName == "denis-threat-hu"
| where FileName has_any ("tor.exe", "firefox.exe", "tor-browser.exe")
| project Timestamp, DeviceName, AccountName, ActionType, FileName, FolderPath, SHA256, ProcessCommandLine
| order by Timestamp desc

```
<img width="2162" height="368" alt="image" src="https://github.com/user-attachments/assets/0ef39b40-7d6a-4c0e-abd9-4bc642a2a393" />


---

### 4. Searched the `DeviceNetworkEvents` Table for TOR Network Connections

Searched for any indication the TOR browser was used to establish a connection using any of the known TOR ports. At `6/26/2025, 8:07:44.422 AM`, an employee on the "denis-threat-hu" device successfully established a connection to the remote IP address `103.251.167.20` on port `9001`. The connection was initiated by the process `tor.exe`, located in the folder 'c:\users\dluffyy\desktop\tor browser\browser\torbrowser\tor\tor.exe' There were a couple of other connections to sites over port `443`.

**Query used to locate events:**

```kql
DeviceNetworkEvents
| where DeviceName == "denis-threat-hu"
| where InitiatingProcessAccountName !="system"
| where InitiatingProcessFileName in (“tor.exe”, “firefox.exe”)
| where RemotePort in ("9001", "9030", "9050", "9150", "9051" “80”, “443”)
| project Timestamp, DeviceName, InitiatingProcessAccountName, ActionType, RemoteIP, RemotePort, RemoteUrl, InitiatingProcessParentFileName, InitiatingProcessFolderPath
| order by Timestamp desc

```
<img width="1105" height="710" alt="image" src="https://github.com/user-attachments/assets/811a2186-c6e6-4916-8ff6-f9e1f8bea24c" />


---

## Chronological Event Timeline 

### 1. File Download - TOR Installer

- **Timestamp:** `6/26/2025, 7:25:38.500 AM`
- **Event:** The user "dluffyy" downloaded a file named `tor-browser-windows-x86_64-portable-14.0.1.exe` to the Downloads folder.
- **Action:** File download detected.
- **File Path:** C:\Users\DLuffyy\Downloads\tor-browser-windows-x86_64-portable-14.5.4.exe

### 2. Process Execution - TOR Browser Installation

- **Timestamp:** `6/26/2025, 7:59:56.920 AM`
- **Event:** The user "dluffyy" executed the file `tor-browser-windows-x86_64-portable-14.5.4.exe  /S` in silent mode, initiating a background installation of the TOR Browser.
- **Action:** Process creation detected.
- **Command:** `tor-browser-windows-x86_64-portable-14.5.4.exe  /S`
- **File Path:** `tor-browser-windows-x86_64-portable-14.5.4.exe  /S`

### 3. Process Execution - TOR Browser Launch

- **Timestamp:** `6/26/2025, 8:01:06.706 AM`
- **Event:** User `dluffyy` opened the TOR browser. Subsequent processes associated with TOR browser, such as `firefox.exe` and `tor.exe`, were also created, indicating that the browser launched successfully.
- **Action:** Process creation of TOR browser-related executables detected.
- **File Path:** `C:\Users\DLuffyy\Desktop\Tor Browser\Browser\TorBrowser\Tor\tor.exe`

### 4. Network Connection - TOR Network

- **Timestamp:** `6/26/2025, 8:07:44.422 AM`
- **Event:** A network connection to IP `103.251.167.20` on port `9001` by user `dluffyy` was established using `tor.exe`, confirming TOR browser network activity.
- **Action:** Connection success.
- **Process:** `tor.exe`
- **File Path:** `c:\users\employee\desktop\tor browser\browser\torbrowser\tor\tor.exe`

### 5. Additional Network Connections - TOR Browser Activity

- **Timestamps:**
  - `6/26/2025, 8:07:46.446 AM` - Connected to `94.16.113.35` on port `443`.
  - `6/26/2025, 8:07:41.242 AM` - Local connection to `127.0.0.1` on port `9150`.
- **Event:** Additional TOR network connections were established, indicating ongoing activity by user "employee" through the TOR browser.
- **Action:** Multiple successful connections detected.

### 6. File Creation - TOR Shopping List

- **Timestamp:** `6/26/2025, 8:35:12.641 AM`
- **Event:** The user `dluffyy` created a file named `tor-shopping-list.txt` on the desktop, potentially indicating a list or notes related to their TOR browser activities.
- **Action:** File creation detected.
- **File Path:** `C:\Users\DLuffyy\Desktop\tor-shopping-list.txt`

---

## Summary

The user "dluffyy" on the "denis-threat-hu" device initiated and completed the installation of the TOR browser. They proceeded to launch the browser, establish connections within the TOR network, and created various files related to TOR on their desktop, including a file named `tor-shopping-list.txt`. This sequence of activities indicates that the user actively installed, configured, and used the TOR browser, likely for anonymous browsing purposes, with possible documentation in the form of the "shopping list" file.

---

## Response Taken

TOR usage was confirmed on the endpoint `denis-threat-hu` by the user `dluffyy`. The device was isolated, and the user's direct manager was notified.

---
