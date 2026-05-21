# Employee Time-Off Management System

An automated leave request and approval system built entirely on Google Workspace — no third-party tools, no paid subscriptions.

---

## What it does

- Employees submit leave requests via a Google Form
- Managers receive an email with one-click Approve / Reject buttons
- Working days are auto-calculated, excluding weekends and Nigerian public holidays
- Approved requests automatically deduct from the employee's leave balance
- Employees are notified of the outcome by email
- All balances reset to 22 days automatically every January 1st

---

## Tech stack

| Tool | Role |
|---|---|
| Google Sheets | Database — employees, requests, config |
| Google Forms | Employee-facing submission form |
| Google Apps Script | Automation — emails, approvals, balance updates |
| Gmail | Notifications to managers and employees |

---

## Project files

| File | Purpose | Run |
|---|---|---|
| `Setup.gs` | Creates and formats the Employees, Requests, and Config sheet tabs | Once |
| `FormSetup.gs` | Creates the Google Form and links it to the spreadsheet | Once |
| `Code.gs` | Main application logic — deploy this as a Web App | Always running |

---

## Sheet structure

### Employees
| Column | Description |
|---|---|
| Email | Employee's work email — used as unique identifier |
| Manager Email | Who receives and actions their leave requests |
| Leave Balance | Remaining PTO days (starts at 22, resets annually) |

### Requests
| Column | Description |
|---|---|
| Request ID | Auto-generated unique ID (e.g. REQ-1748765432100) |
| Employee Email | Submitted email |
| Employee Name | Submitted name |
| Start Date | First day of leave |
| End Date | Last day of leave |
| Days | Auto-calculated working days |
| Notes | Optional employee message |
| Status | PENDING → APPROVED or REJECTED |
| Manager Comment | Optional comment added during approval |
| Submitted At | Timestamp of form submission |
| Decided At | Timestamp of manager decision |

### Config
| Key | Value |
|---|---|
| Annual PTO Days | 22 |
| Form URL | Published form link shared with employees |
| Form Edit URL | Link to edit the form |
| Last Reset | Date of last annual balance reset |

---

## Form fields

Fields in submission order — order matters and must match `Code.gs`:

```
values[0]  Timestamp       (auto)
values[1]  Employee Email
values[2]  Employee Name
values[3]  Start Date
values[4]  End Date
values[5]  Notes           (optional)
```

> Number of Days is intentionally absent — it is calculated automatically by the script.

---

## Setup instructions

### Step 1 — Create a Google Sheet
1. Go to [sheets.google.com](https://sheets.google.com) and create a new blank spreadsheet
2. Copy the Sheet ID from the URL (the string between `/d/` and `/edit`)

### Step 2 — Run Setup.gs
1. Open **Extensions → Apps Script**
2. Create a new script file named `Setup`
3. Paste the contents of `Setup.gs` and save
4. Run `runFullSetup()`
5. Grant permissions when prompted
6. Three tabs will be created: Employees, Requests, Config

### Step 3 — Add your employees
Go to the Employees tab and replace the sample rows with real staff data.

### Step 4 — Run FormSetup.gs
1. Create another script file named `FormSetup`
2. Paste the contents of `FormSetup.gs` and save
3. Run `runFormSetup()`
4. A popup will display the shareable form URL — share this with your employees
5. Both URLs are saved to the Config tab

### Step 5 — Add Code.gs
1. Paste the contents of `Code.gs` into the default `Code.gs` file
2. Replace the two placeholders at the top:

```javascript
const SPREADSHEET_ID = 'YOUR_SPREADSHEET_ID_HERE'; // from Step 1
function getWebAppUrl() {
  return 'YOUR_WEB_APP_URL_HERE'; // from Step 6
}
```

### Step 6 — Deploy as Web App
1. Click **Deploy → New deployment**
2. Type: **Web app**
3. Execute as: **Me**
4. Who has access: **Anyone**
5. Click **Deploy** and copy the Web App URL
6. Paste the URL into the `getWebAppUrl()` function in `Code.gs`
7. Redeploy: **Deploy → Manage deployments → edit → New version → Deploy**

### Step 7 — Set up triggers
Go to **Apps Script → Triggers** and add two:

| Function | Event source | Event type |
|---|---|---|
| `onFormSubmit` | From spreadsheet | On form submit |
| `resetAnnualLeaveBalances` | Time-driven | Year timer → January 1 |

> ⚠️ The `onFormSubmit` trigger **must** be set to **From spreadsheet** — not From form. Using "From form" breaks the `e.values` array and the script will fail silently.

---

## QA validation

Every form submission is validated before anything is written to the sheet:

| Check | Error returned if |
|---|---|
| Start date not in the past | Start date is before today |
| End date not before start | End date is earlier than start date |
| Max 30 calendar days | Date range spans more than 30 days |
| No zero working days | All days in range are weekends or holidays |
| Sufficient balance | Days requested exceeds remaining balance |

If any check fails, the employee receives an email with a clear explanation and a prompt to resubmit. Nothing is written to the Requests sheet.

---

## Public holidays

Nigerian public holidays are defined in the `calculateWorkingDays()` function inside `Code.gs`. The list currently covers **2026 and 2027**. 

Update the list each December by adding the next year's dates:

```javascript
const publicHolidays = [
  '2026-01-01', // New Year's Day
  '2026-05-01', // Workers' Day
  // ... add new year below
  '2028-01-01', // New Year's Day 2028
];
```

---

## Updating the code

After any edit to `Code.gs`, you must redeploy for changes to take effect:

1. **Deploy → Manage deployments**
2. Click the **edit (pencil)** icon
3. Change version to **New version**
4. Click **Deploy**

The Web App URL stays the same after redeployment.

---

## Test functions

Run these manually in Apps Script to verify each part works:

| Function | What it tests |
|---|---|
| `TEST_sendManagerEmail()` | Sends a sample manager notification email |
| `TEST_resetBalances()` | Resets all employee balances to 22 |
| `TEST_workingDays()` | Calculates working days for a test date range |
| `TEST_validation()` | Runs all four QA checks and logs pass/fail |
| `printFormFields()` | Logs current form field order to the console |
| `printFormUrls()` | Logs the published and edit URLs of the linked form |

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Requests sheet not populating | Email mismatch between form and Employees tab | Ensure emails match exactly |
| Manager email not arriving | Wrong manager email in Employees tab | Update Manager Email column with real address |
| Approve/Reject link goes to Google Drive error | Web App not deployed | Deploy as Web App (Step 6) |
| Approve/Reject link shows "refused to connect" | `getWebAppUrl()` returning wrong URL | Hardcode the correct Web App URL |
| Script runs but no email sent | Trigger set to "From form" instead of "From spreadsheet" | Delete and recreate trigger correctly |
| Employee gets "email not recognised" error | Email in form doesn't match Employees tab | Fix the email in the Employees tab |
| Balance not deducting after approval | Employee email lookup failing silently | Check Executions log in Apps Script for errors |

---

## Annual maintenance checklist

Do this each December:

- [ ] Add next year's public holidays to `calculateWorkingDays()` in `Code.gs`
- [ ] Redeploy `Code.gs` as a new version
- [ ] Verify the annual reset trigger is still active in Apps Script → Triggers
- [ ] Confirm the `resetAnnualLeaveBalances` trigger is set to January 1

---

## License

Internal use only.
