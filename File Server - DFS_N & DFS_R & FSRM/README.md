# TechCorp File Server Lab — DFS Namespace, Replication & Quota Management

A continuation of the `techcorp.local` Active Directory lab environment.
This phase replaces a single, server-bound file share with a
multi-branch style shared folder: one consistent path
(`\\techcorp.local\Files\CompanyPolicies`) backed by two physical file
servers, kept in sync automatically, with a storage quota enforced on
each copy — so the company can survive a server outage without users
noticing, and without one department filling up the disk.

## Starting Environment (Pre-Existing)

Before this phase, the domain already had the following structure in
place from earlier lab builds:

| Machine | Role | OS | IP |
|---|---|---|---|
| WinServer-2022 | Domain Controller (AD DS, DNS) | Windows Server 2022 | 192.168.10.1 |
| WinServer-2019 | File Server #1 | Windows Server 2019 | 192.168.10.2 |
| Windows10 | Client workstation | Windows 10 22H2 | 192.168.10.10 |

- A third VM, **WinServer2022-FS2**, was built specifically for this
  scenario (DFS Namespace/Replication require Windows Server, so this
  couldn't reuse an existing Linux VM)
- Installed as **Server Core** (no GUI) due to laptop resource
  constraints — all configuration on this server done via PowerShell
  instead of Server Manager
- Windows auto-truncated the intended 18-character rename to fit the
  15-character NetBIOS limit, so the server's actual computer name ended
  up as `WINSERVER2022-2`





![DFS Roles Installed — WinServer-2019](screenshots/DFS-N_DFS-R_2019.png)









![DFS Roles Installed — FS2](screenshots/DFS-N_DFS-R_FS2.png)





## What Was Built

### 1. File Shares
Created a `CompanyPolicies` SMB share on both servers
(`C:\Shares\CompanyPolicies`) — via the Server Manager GUI on
WinServer-2019, and via `New-SmbShare` on FS2.





![Share Created — WinServer-2019](screenshots/Created-Share_CompanyPolicies-2019.png)









![Share Created — FS2](screenshots/Created-Share_CompanyPolicies-FS2.png)





### 2. DFS Namespace
Created a domain-based namespace `\\techcorp.local\Files` on
WinServer-2019 (root share at `C:\DFSRoots\Files`).





![Namespace Created — WinServer-2019](screenshots/create-Namespace-2019.png)





### 3. Folder Targets
Added a `CompanyPolicies` folder inside the namespace, with folder
targets pointing to the share on each server.





![Folder Target — WinServer-2019](screenshots/FolderTarget-2019.png)









![Folder Target — FS2](screenshots/FolderTarget-FS2.png)





### 4. Replication Group
Created a DFS Replication group (Full mesh topology, continuous
replication, WinServer-2019 as primary member) linking the two folder
targets.





![Replication Group Created](screenshots/ReplicationGroup-2019.png)





### 5. Namespace Redundancy Fix
An initial failover test failed: the namespace root was only hosted on
WinServer-2019, so shutting that server down made
`\\techcorp.local\Files` unreachable entirely — a single point of
failure that folder targets alone didn't solve. Added FS2 as a second
namespace server (`New-DfsnRootTarget`) so both servers host the
namespace root, not just the shared folder underneath it.





![Namespace Server Added — FS2](screenshots/create-Namespace-FS2.png)





### 6. FSRM Quota
Installed the File Server Resource Manager role on both servers and
configured a 5GB hard quota on `CompanyPolicies` on each — via the GUI
on WinServer-2019, via `New-FsrmQuota` on FS2.





![FSRM Installed — FS2](screenshots/FSRM-Installed-FS2.png)









![Quota Configured — WinServer-2019](screenshots/Qouta-Configuration-2019.png)









![Quota Configured — FS2](screenshots/Qouta-Configuration-FS2.png)





### 7. Client Drive Mapping (Group Policy)
Deployed a GPO on WinServer-2022 (User Configuration → Preferences →
Windows Settings → Drive Maps) mapping the namespace path to `Z:`,
labeled "Shared Files", with Reconnect enabled — so end users get the
share automatically without needing to know or type the UNC path.





![Drive Mapping GPO](screenshots/DriveMapping-2022.png)





## Verification

### Replication Test
Created a test file on one server's share and confirmed it appeared on
the other server's share shortly after, with no manual intervention.

### Failover Test
With WinServer-2019 fully shut down, `\\techcorp.local\Files\CompanyPolicies`
remained accessible from a client and the test file was still visible —
served transparently by FS2 instead.





![Share Accessible During Failover — Client](screenshots/Share-Access_ClientWin10.png)





### Quota Enforcement Test
Attempted to copy a 6GB test file into the share from a client. Copy was
correctly rejected with a "not enough space" error, confirming the 5GB
quota is enforced by FSRM independently of the underlying disk's actual
free space (59GB).





![Quota Enforcement — Client](screenshots/Qouta-Test-ClientWin10.png)





### Drive Mapping Test
Logged in as a client user and confirmed `Z:` appears under Network
locations, labeled "Shared Files", correctly showing the 5GB quota as
the drive's total capacity (4.99 GB free of 5.00 GB).





![Drive Mapping Confirmed — Client](screenshots/Confirm-DriveMapping-ClientWin10.png)





## Best Practices & Security Notes

- **A DFS namespace root needs its own redundancy.** Adding a folder
  target on a second server is not enough on its own — if the namespace
  root itself is only hosted on one server, the whole namespace becomes
  unreachable when that server goes down, even though the underlying
  data is safely replicated elsewhere.
- **FSRM quotas are enforced above the file system, not the disk.**
  Clients see the quota limit as if it were the drive's actual capacity,
  even when the physical disk has far more free space — this makes the
  restriction transparent and easy for end users to understand without
  needing to explain the underlying mechanism.
- **Server Core forces PowerShell fluency.** Managing a role entirely
  without a GUI (FS2 in this lab) surfaces gaps that GUI-based management
  hides — worth doing deliberately on at least one server in any
  multi-server scenario, even when a GUI option is available elsewhere.
- **NetBIOS computer names are capped at 15 characters.** Longer names
  are silently truncated by Windows rather than rejected — worth checking
  the actual resulting name after any rename, rather than assuming it
  matches what was typed.
- **Drive mapping via Group Policy scales better than manual setup.** A
  GPO targeted at the right OU/group applies automatically to every
  current and future member, instead of requiring each user to be walked
  through mapping the drive manually.

## Key Takeaways

- DFS Namespace, DFS Replication, and FSRM Quota are three separate,
  independently-configured components — getting real fault tolerance out
  of them requires deliberately checking each layer (namespace hosting,
  replication, and quota) rather than assuming one working piece means
  the whole scenario is resilient.
- The failed initial failover test was itself a useful part of the
  scenario: it surfaced a real single-point-of-failure gap and the fix
  (adding a second namespace server) reflects a genuine production
  consideration, not just a happy-path setup.
- This is a snapshot of the current state of the lab and reflects the
  decisions and trade-offs made at this stage — not a finished or final
  configuration.