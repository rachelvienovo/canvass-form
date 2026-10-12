# Approver lock for the Google Sheet script

The dashboard only shows Approve / Disapprove after the approver passcode is entered. The page cannot enforce this on its own, so the Google Apps Script that stores the canvasses must check the passcode too. Without these changes the dashboard stays locked ("The Google Sheet script has not been updated with the approver lock yet").

## 1. Set the passcode

Apps Script editor > **Project Settings** > **Script properties** > **Add script property**

- Property: `APPROVER_KEY`
- Value: a private passcode that only the approver knows (not the same as the shared access code).

## 2. Add these functions to the script

```js
var APPROVAL_FIELDS_ = ['approved', 'approvedBy', 'approvedAt', 'disapproved', 'disapprovedBy', 'disapprovedAt', 'disapprovalReason'];

function approverOk_(k) {
  var p = PropertiesService.getScriptProperties().getProperty('APPROVER_KEY');
  return !!p && String(k || '') === p;
}

/**
 * Call with the incoming record, the stored record for the same canvass (or null) and whether
 * the sender gave the right approver passcode. Returns the record that is safe to store.
 * Only the approver can set or clear approval fields. If anyone else changes the price or quantity,
 * an existing decision is cleared (it no longer applies to the changed bid).
 */
function guardApproval_(rec, old, isApprover) {
  delete rec.approverKey;
  if (isApprover) return rec;
  var unchanged = old && ['price', 'finalPrice', 'quantity'].every(function (k) {
    return String(old[k] == null ? '' : old[k]) === String(rec[k] == null ? '' : rec[k]);
  });
  APPROVAL_FIELDS_.forEach(function (f) {
    if (unchanged && old[f] !== undefined) rec[f] = old[f]; else delete rec[f];
  });
  return rec;
}
```

## 3. Hook them in

**In `doGet`**, right after the shared access code has been accepted, add:

```js
if (e.parameter.approverCheck !== undefined) {
  return ContentService.createTextOutput(JSON.stringify({ ok: true, approver: approverOk_(e.parameter.approverCheck) }))
    .setMimeType(ContentService.MimeType.JSON);
}
```

**In `doPost`**, after the incoming record has been parsed and the shared access code accepted, and before the row is written:

```js
var isApprover = approverOk_(body.approverKey);          // 'body' = the parsed request JSON
rec = guardApproval_(rec, existingRecordOrNull, isApprover);
// 'rec' is the record about to be saved; 'existingRecordOrNull' is JSON.parse of the stored record for the same ref
```

Also make sure the access `key` is removed from the record before it is stored.

## 4. Redeploy

**Deploy** > **Manage deployments** > edit the existing deployment > **Version: New version** > **Deploy**. The web app URL stays the same.

## 5. Test

1. Open the dashboard, click **Unlock approvals** and enter the passcode. It should say "Approvals unlocked".
2. Approve a canvass, then press **Refresh data**. The approval should still be there.
3. Edit that canvass's final bid price from the form. The approval should clear (the bid changed).
