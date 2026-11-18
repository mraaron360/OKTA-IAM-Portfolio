
# Okta + PowerShell 


| Script | What it does |
|---|---|
| `scripts/Update-OktaUsersBulk.ps1` | Bulk profile / group / suspend changes from a CSV. `-WhatIf` dry run, only real diffs are sent, `-MaxChanges` guard, automatic JSON backup, `-RollbackFrom`. |
| `scripts/Get-OktaStaleAccounts.ps1` | Finds users inactive for N days or never logged in. CSV + HTML report. Optional `-Action Suspend\|Deactivate` with per-user confirmation, `-ExcludeLogin` / `-ExcludeGroup` protection. |
| `scripts/Invoke-OktaGroupAudit.ps1` | Read-only group inventory and ranked findings: admin-role groups, empty groups, invalid group rules, non-active members, no description, naming, stale membership. |
| `scripts/Invoke-OktaTokenRotation.ps1` | `-Mode Audit` (token age/idle/network), `-Mode SwapSsws` (verify new token, revoke old), `-Mode RotateJwk` (automatic key rotation for an OAuth service app). |

`module/OktaLab.psm1` handles auth, Link-header paging, HTTP 429 (`X-Rate-Limit-Reset`) waits, 5xx retries, and refuses to send your credential to any host other than the org.

**New to the Okta console side?** Jump to [Okta console screenshots](#okta-console-screenshots-setup-order) below (ten numbered, annotated screenshots) or follow the step-by-step [`docs/setup-guide.md`](docs/setup-guide.md).

## Requirements
* PowerShell 7.2+ (`pwsh`). The scripts will not run in Windows PowerShell 5.1.
* Python 3.9+ for the local mock. `pip install cryptography` is optional for the mock but required for `lab/selftest_mock.py`.

## Lab A: the local mock (do this first, risk-free)
The mock imitates the parts of the Okta API these scripts use, with 65 users (stale, never-logged-in, suspended, locked out), 11 groups (an empty one, admin-role groups, a rule-managed one, an app-imported one), 4 API tokens of different ages, 3 group rules (one INVALID), and an OAuth service app with a registered key.

```powershell
# terminal 1
python lab/mock_okta.py                       # add --rate-limit-every 9 to rehearse 429 handling
                                              # add --no-group-stats to test the audit fallback path
# terminal 2
./lab/Invoke-LabWalkthrough.ps1               # runs every script end to end, loopback only
```

Or drive it by hand:

```powershell
$env:OKTA_ORG_URL   = 'http://127.0.0.1:8777'
$env:OKTA_API_TOKEN = 'lab-token-old'
./scripts/Get-OktaStaleAccounts.ps1 -DaysInactive 90 -ExcludeLogin 'svc-*' -ExcludeGroup 'Break-Glass'
./scripts/Invoke-OktaGroupAudit.ps1 -IncludeMembers
./scripts/Invoke-OktaTokenRotation.ps1 -Mode Audit -ResolveOwners
```
Reports land in `reports/` (or `$env:OKTA_REPORT_DIR`). `POST /_lab/reset` restores the seed data (and regenerates the lab service key).

Check the mock itself any time: `python lab/mock_okta.py &` then `python lab/selftest_mock.py`.

## Lab B: your real Okta org (Integrator Free plan)
Your org `integrator-5832766.okta.com` allows **10 active users** and already uses some. The seed script therefore creates at most **4 STAGED users** (no emails, no passwords) and 4 `LAB-` groups, all tagged so teardown removes only them.

1. Create an API token yourself: Admin Console > Security > API > Tokens > Create token. Name it `okta-ps-lab`. The token acts with *your* admin rights, so treat it like a password. Restrict it to a network zone if you can.
2. In PowerShell (use the **org** URL, not the `-admin` URL):
   ```powershell
   $env:OKTA_ORG_URL   = 'https://integrator-5832766.okta.com'
   $env:OKTA_API_TOKEN = Read-Host 'Okta token'     # or: Connect-Okta -Token (Read-Host -AsSecureString)
   ./lab/New-OktaLabEnvironment.ps1 -WhatIf          # preview
   ./lab/New-OktaLabEnvironment.ps1
   ./scripts/Update-OktaUsersBulk.ps1 -CsvPath ./data/sample-updates.csv -WhatIf
   ./scripts/Invoke-OktaGroupAudit.ps1
   ./lab/Remove-OktaLabEnvironment.ps1               # removes only objects it created
   ```
3. Real-org caveat: STAGED users never sign in, so the stale report shows them as "never logged in". Use the mock for realistic stale data.
4. Keep real-tenant dry runs on: always run a bulk update or stale action with `-WhatIf` first.

## Okta console screenshots (setup order)
Captured from org `integrator-5832766` on 2026-11-18. Names and emails are blurred; red boxes mark where to click. Nothing was created while taking them. Full walkthrough: [`docs/setup-guide.md`](docs/setup-guide.md).

### Screenshot 1: Confirm you are signed in
Use this **org** URL for the scripts, not the `-admin` URL.

<img width="1352" height="819" alt="01-signed-in-account-chooser" src="https://github.com/user-attachments/assets/d2d3edfd-baa6-4ffc-a408-c76aa681eb53" />

### Screenshot 2: Admin Console dashboard
The plan box (bottom left) shows the Integrator Free limit of 10 active users.



<img width="1467" height="882" alt="02-admin-dashboard" src="https://github.com/user-attachments/assets/360c1203-cc0b-48e6-acfb-2bc50db35c4a" />



### Screenshot 3: Directory > People
Where the lab users appear. Emails are blurred.


<img width="1530" height="854" alt="03-directory-people" src="https://github.com/user-attachments/assets/833de656-0018-424b-a2cb-c2a0841cde71" />



### Screenshot 4: Directory > Groups
Lab groups are prefixed `LAB-`; existing groups are never touched by the teardown script.


<img width="1530" height="854" alt="04-directory-groups" src="https://github.com/user-attachments/assets/1e942942-25ee-4a97-886e-fec914801cb2" />




### Screenshot 5: Security > Networks
Optional but recommended: create an IP zone and restrict the API token to it.




<img width="1530" height="854" alt="05-security-networks" src="https://github.com/user-attachments/assets/c09ef9a3-6f2d-47db-a173-0f0d1f8ab3c2" />


### Screenshot 6: Security > API > Tokens
Click **Create token**.


<img width="1530" height="854" alt="06-security-api-tokens" src="https://github.com/user-attachments/assets/086be57f-9c00-43b2-8b87-d45106e9cd4b" />


### Screenshot 7: Create token dialog
Name it `okta-ps-lab` and choose your network zone. The token is shown once, so paste it straight into PowerShell.


<img width="1530" height="854" alt="07-create-token-dialog" src="https://github.com/user-attachments/assets/bed1e6c9-f954-478c-83cf-9e21f54bdd27" />



### Screenshot 8: Applications
For key rotation, start a service app with **Create App Integration**.



<img width="1530" height="854" alt="08-applications" src="https://github.com/user-attachments/assets/7cfeea36-68a9-4c67-b8c5-1cefc1e3002b" />


### Screenshot 9: Create app integration > API Services
Choose **API Services**, then **Next**. Name it `okta-ps-lab-svc`.




<img width="1530" height="854" alt="09-create-app-api-services" src="https://github.com/user-attachments/assets/6fa00379-dd17-487b-b921-da67fe925824" />


### Screenshot 10: Security > Administrators
Assign the service app an admin role on the **Admins** tab.


<img width="1530" height="854" alt="10-security-administrators" src="https://github.com/user-attachments/assets/824d51f3-b5e6-4218-8c88-df1b12142154" />


## Token rotation, honestly
* Okta cannot create SSWS tokens through the API. `SwapSsws` therefore expects *you* to create the new token in the console, then it proves the new token works, finds the old one, refuses to revoke the token that is in use, revokes the old one, and (optionally) stores the new one with `Microsoft.PowerShell.SecretManagement`.
* `RotateJwk` is the fully automated path and is what Okta recommends for automation: an OAuth 2.0 service app using `private_key_jwt`. Setup once in the console: Applications > Create App Integration > API Services; client authentication **Public key / Private key**; grant the scopes you need (`okta.apps.manage` for rotation, `okta.users.read`, `okta.groups.read`, ...) and assign an admin role to the service app. Then set:
  ```powershell
  $env:OKTA_CLIENT_ID = '<client id>'; $env:OKTA_PRIVATE_KEY_PATH = './keys/svc.pem'
  $env:OKTA_KEY_ID = '<kid>'; $env:OKTA_SCOPES = 'okta.users.read okta.groups.read'
  Remove-Item Env:OKTA_API_TOKEN      # OAuth is used when no SSWS token is set
  ```
  The rotation writes the new private key with owner-only permissions, registers and activates the new public key, proves it by minting a token with it, then deactivates and deletes the old key. If the new key fails verification, it is removed and the old key stays.

## Safety defaults
* Anything that changes data supports `-WhatIf`. Bulk updates abort before any write if more than `-MaxChanges` users would change. Stale actions stop above `-MaxActions` and never touch protected users.
* The module only calls the connected org's host (including paging links) and only allows plain HTTP for loopback.
* No secret is written to a log or report. Private keys are created with owner-only permissions before the key material is written.

## Known limits
* `lastLogin` is judged client-side; very large orgs (100k+ users) should narrow with `-Status` or use System Log based reporting.
* The group audit uses `expand=stats`; if your org does not return it the script falls back to per-group calls (slower, more rate limit).
* The mock follows the public API docs, not live behaviour; validate against a real org before relying on edge cases (for example how Okta orders `Link` headers, or whether `lastUpdated` on an API token reflects use).
