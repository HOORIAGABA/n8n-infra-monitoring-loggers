\# n8n Infrastructure Backup Monitoring



Webhook-driven n8n workflows that turn nightly backup jobs into an auditable, idempotent log in Google Sheets — so a silently failing backup is visible the next morning instead of at restore time.



!\[n8n](https://img.shields.io/badge/n8n-EA4B71?style=flat-square\&logo=n8n\&logoColor=white)

!\[Google Sheets](https://img.shields.io/badge/Google%20Sheets-34A853?style=flat-square\&logo=googlesheets\&logoColor=white)

!\[JavaScript](https://img.shields.io/badge/JavaScript-F7DF1E?style=flat-square\&logo=javascript\&logoColor=black)

!\[Webhook](https://img.shields.io/badge/trigger-webhook-blue?style=flat-square)

!\[License](https://img.shields.io/badge/license-MIT-green?style=flat-square)

!\[Last commit](https://img.shields.io/github/last-commit/HOORIAGABA/n8n-infra-monitoring-loggers?style=flat-square)



\---



\## The problem



Backup scripts are the classic silent failure. Cron runs them, they write to a log nobody reads, and the first time anyone checks is the day a restore is needed — the worst possible moment to discover last Tuesday's dump was empty.



The usual fix is an email on failure, which fails twice over: it says nothing when the script never ran at all, and after a month everyone filters it.



\*\*What this does instead:\*\* every backup run reports itself to a webhook. Each run becomes one row in a shared sheet, keyed so repeated reports never duplicate. A pivot over that sheet answers the only question that matters — \*which databases have an unbroken run of successful backups, and which have gaps?\*



\## Architecture



```mermaid

flowchart LR

&#x20;   A\["Backup script<br/>(cron)"] -->|"POST JSON"| B\["n8n Webhook"]

&#x20;   B --> C\["Capture<br/>timestamp"]

&#x20;   C --> D\["Format date<br/>dd/MM/yyyy"]

&#x20;   D --> E\["Build unique\_key<br/>name + date"]

&#x20;   E --> F{"Row exists<br/>for key?"}

&#x20;   F -->|No| G\["Append row"]

&#x20;   F -->|Yes| H\["Update row"]

&#x20;   G --> I\[("Google Sheet")]

&#x20;   H --> I

&#x20;   I --> J\["Pivot chart:<br/>backup coverage"]

```



Two workflows share this shape, writing to separate tabs of one spreadsheet:



| Workflow | Reports on | Target tab |

|---|---|---|

| `workflows/db-backup-logger.json` | Database dumps | `DbBackupSheet` |

| `workflows/media-backup-logger.json` | Media directory backups | `MediaBackupSheet` |



\## How it works



\*\*1. The backup script reports in.\*\* On completion it POSTs to the n8n webhook:



```json

{

&#x20; "database\_name": "clientdb\_prod",

&#x20; "successful\_backup": true,

&#x20; "has\_db": true,

&#x20; "has\_media\_dir": false

}

```



\*\*2. n8n timestamps the event\*\* rather than trusting a client-supplied date, then formats it as `dd/MM/yyyy` — one row per backup per day.



\*\*3. A Code node builds the idempotency key:\*\*



```js

unique\_key = `${database\_name}\_${formattedDate}`

```



with fallbacks to `UnknownBackup` and an ISO date, so a malformed payload still lands somewhere visible instead of being dropped.



\*\*4. Google Sheets `appendOrUpdate`\*\* matches on `unique\_key`. New key means a new row; an existing key updates in place.



\## Sheet schema



| Column | Source | Example |

|---|---|---|

| `unique\_key` | Generated | `clientdb\_prod\_24/09/2026` |

| `Backup Name` | `body.database\_name` | `clientdb\_prod` |

| `DB` / `Media` | `body.successful\_backup` | `TRUE` |

| `Date` | Server timestamp, formatted | `24/09/2026` |



A pivot over `Backup Name` × `Date` makes gaps obvious — an empty cell is a backup that never reported.



\## Design decisions



\*\*Idempotent upsert instead of append.\*\* A plain append means a retried script, a cron overlap, or a timeout-then-retry produces duplicate rows, and the sheet stops being trustworthy. Keying each row on `name + date` and using `appendOrUpdate` makes the write idempotent: reporting the same backup five times in one day leaves exactly one row. The reporting side can therefore retry freely without coordinating with this system — which matters, because the reporting side is a bash script with no state.



\*\*The date comes from the server, not the payload.\*\* A client-supplied timestamp can be wrong, missing, or in another timezone, and it's exactly the field a buggy script would corrupt. Generating it in n8n means the key is always well-formed and the log reflects when the report was \*received\*.



\*\*Absence is the signal, not failure.\*\* The system doesn't depend on a script successfully reporting an error — a script that dies before its reporting line sends nothing at all. Because every expected backup should produce a row every day, a \*missing\* row is the alert. This catches the failure mode that error-only alerting structurally cannot.



\*\*Google Sheets as the store.\*\* Deliberate, not a shortcut. The people who need to read this are ops and team leads, not engineers. A sheet gives them filtering, pivots and charts with no tooling, no access request and no dashboard to maintain. The tradeoff is row limits and no query language, which this volume doesn't reach.



\*\*Two workflows rather than one branched flow.\*\* They're near-identical, and merging them behind a switch would remove the duplication. Kept separate so a change to media-backup logic cannot break database-backup logging, and so either can be disabled independently during maintenance. At two workflows the duplication is cheaper than the coupling; at five it wouldn't be.



\## Screenshots



| | |

|---|---|

| !\[Database backup workflow](assets/db-backup-flow.png) | !\[Media backup workflow](assets/media-backup-flow.png) |

| Database backup workflow in n8n | Media backup workflow in n8n |

| !\[Backup log sheet](assets/db-backup-sheet.png) | !\[Coverage pivot chart](assets/backup-pivot-chart.png) |

| The resulting log | Coverage pivot — gaps visible at a glance |



\## Setup



\*\*Requirements:\*\* an n8n instance, a Google account, a Google Cloud project with the Sheets API enabled.



1\. \*\*Create the spreadsheet\*\* with tabs `DbBackupSheet` and `MediaBackupSheet`, each with headers: `unique\_key`, `Backup Name`, `DB` (or `Media`), `Date`.

2\. \*\*Import the workflows\*\* — n8n → \*Workflows → Import from File\* → each file in `workflows/`.

3\. \*\*Connect credentials.\*\* Create a Google Sheets OAuth2 credential and select it on the Sheets node of each workflow.

4\. \*\*Point each workflow at your sheet.\*\* Replace `YOUR\_SPREADSHEET\_ID` with your own spreadsheet ID (from its URL) and confirm the tab.

5\. \*\*Activate\*\* each workflow and copy its production webhook URL.

6\. \*\*Report from your backup script:\*\*



```bash

curl -X POST "$N8N\_WEBHOOK\_URL" \\

&#x20; -H "Content-Type: application/json" \\

&#x20; -d "{\\"database\_name\\":\\"$DB\_NAME\\",\\"successful\_backup\\":$SUCCESS,\\"has\_db\\":true,\\"has\_media\_dir\\":false}"

```



7\. \*\*Add the pivot:\*\* rows `Backup Name`, columns `Date`, values count of `unique\_key`.



> Document IDs and webhook paths in these files are placeholders. Real IDs are environment-specific and are intentionally not committed.



\## Limitations and next steps



\- \*\*No alerting.\*\* Gaps are visible in the sheet but nothing pushes a notification. Next step is a scheduled workflow that reads the sheet each morning and messages Slack about any expected backup with no row for yesterday — the natural completion of the "absence is the signal" design.

\- \*\*No integrity verification.\*\* The log records that a script \*claimed\* success, not that the dump is restorable. Reporting file size and a checksum would let the sheet flag a suspiciously small backup.

\- \*\*Unauthenticated webhooks.\*\* Fine on an internal network; a shared-secret header is needed before exposing these publicly.

\- \*\*Sheet row limits.\*\* Years of headroom at this frequency, but a long-lived deployment should archive by year or move to a database.

\- \*\*Duplicated logic across two workflows\*\* — a conscious tradeoff documented above, worth revisiting at a third logger.



\## Repository layout



```

├── workflows/

│   ├── db-backup-logger.json      # Database backup logger

│   └── media-backup-logger.json   # Media backup logger

├── assets/                        # Workflow and output screenshots

├── LICENSE

└── README.md

```



\## License



MIT — see \[LICENSE](LICENSE).

