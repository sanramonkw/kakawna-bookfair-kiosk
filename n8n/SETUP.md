# Live tracking: kiosk → n8n → Zoho Sheet

Every new entry on the stand tablet is sent to n8n within a few seconds and saved in a Zoho Sheet.
If the internet drops, the kiosk keeps working and sends everything it missed once it is back online.
If Zoho is down or rejects a save, n8n does not confirm the batch, so the kiosk keeps it and retries.

## 1. Zoho Sheet
Quickest: in Zoho Sheet, upload `kakawna-kiosk-zoho-template.xlsx` (it already has the Dashboard,
Entries and Draws worksheets with the right headers and formulas). Or create it by hand with
two worksheets and these headers in row 1, spelled exactly as shown:

**Entries:** `id | ts | name | phone | quote | language | hidden | disqualified | deviceId | receivedAt`

**Draws:** `id | ts | name | phone | quote | entryId | poolSize | deviceId | receivedAt`

Copy the workbook's resource ID from its URL: `https://sheet.zoho.com/sheet/open/<RESOURCE-ID>`.

Optional **Dashboard** worksheet (live numbers):

| Cell | Formula |
|---|---|
| Total entries | `=COUNTA(Entries!A2:A5000)` |
| Entries today | `=COUNTIF(Entries!B2:B5000; TEXT(TODAY(); "yyyy-mm-dd") & "*")` |
| Disqualified entries | `=COUNTIF(Entries!H2:H5000; "yes")` |

Times are in UTC (Kuwait time is UTC+3). The kiosk's own staff panel still shows the exact number of people in the draw.

## 2. n8n
1. Go to **Workflows → Import from file** and choose `kakawna-kiosk-live-sync.json`.
2. In **Parse and check token**, change `CHANGE-ME-long-random-token` to a long random secret.
3. In **Zoho: update row by id** and **Zoho: add new rows**, replace `PASTE-ZOHO-WORKBOOK-RESOURCE-ID` in the URL with the resource ID.
4. Both Zoho nodes use the same Zoho OAuth2 credential as the Karak Tea flow. It needs permission to **read and update** sheet data, not only to add rows. If saves fail with a scope error, add `ZohoSheet.dataAPI.UPDATE` and `ZohoSheet.dataAPI.READ` to the credential's scope and reconnect it.
5. Save the workflow and switch it to **Active**.
6. Open **Kiosk webhook** and copy the **Production URL** (it ends in `/webhook/kakawna-kiosk`).

## 3. Kiosk (stand tablet only)
Open the staff panel (tap the logo 5 times, then enter the PIN). Under **Backup → Cloud backup**:
- Address: the Production URL from n8n
- Secret token: the same secret as in step 2.2
- Tap **Save and send now**. The status line should read **Everything is backed up.**

Set this only on the stand tablet. The address and token are saved on that device, not in the HTML file,
so test copies (for example the GitHub link) never send anything to the sheet.

## How it saves
- Each row is first **updated by `id`**, so when staff hide or disqualify an entry, its existing row changes and no duplicate appears.
- Rows that matched nothing are new and are **added** in one request per worksheet.
- A quote that starts like a formula (`=`, `+`, `@`) gets a leading `'`, so it is stored as text and never runs as a formula.

## Checking it
- **Backup problem (HTTP 401)**: the secret token does not match.
- **Backup problem (HTTP 404)**: the workflow is not active, or the URL is the *Test* URL instead of the *Production* URL.
- **Backup problem (HTTP 500)**: Zoho rejected the save. Open the failed execution in n8n to see Zoho's message (usually the credential scope or a worksheet name).
- **no internet**: entries are safe on the tablet and will be sent automatically later.
