Google Sheets → Google Docs Automation Workflow
This solution creates an automation that triggers from a Google Sheet and generates new Google Docs from a Google Doc template, filling placeholders with values from the sheet’s columns. It supports both manual runs and automated triggers (installable on edit or on form submit), uses header-driven mapping, tracks status, and writes back the created Doc URL to the sheet.

Summary
Prepare one Google Doc template with placeholders like {{FieldName}} matching your sheet headers.
Add a Google Apps Script bound to your sheet that:
Listens for an installable trigger (on edit or on form submit) or runs on demand.
Copies the template into a target folder.
Replaces placeholders in the new doc using values from the edited/submitted row.
Names the file using a configurable pattern (e.g., {{Project}} - {{Name}}).
Writes back status, a timestamp, and the new Doc URL to the sheet.
Includes error handling, idempotency safeguards, and support for dates/booleans.
Prerequisites
A Google Doc template with placeholders in double curly braces (e.g., {{Name}}, {{Project}}, {{Date}}).
A Google Sheet with:
Data columns whose headers match the template placeholders.
Control columns:
Generate (checkbox or “Yes/TRUE” as trigger)
Status
Doc URL
Generated At (timestamp)
Access to Apps Script (Extensions → Apps Script) and permissions to use DriveApp and DocumentApp.
Recommended Sheet Structure
Row 1: Headers (e.g., Name, Email, Project, Date, Generate, Status, Doc URL, Generated At)
Rows 2+: Data rows
Triggering options:
Generate column (checkbox or text “Yes”/TRUE) to trigger per row
On form submit trigger if the sheet is linked to a Google Form
Manual menu item to generate documents for all pending rows
Template Guidelines
Use placeholders: {{HeaderName}} (case sensitive and whitespace tolerant between braces and key)
Example template text:
Title: Project Brief: {{Project}}
Body:
Client Name: {{Name}}
Email: {{Email}}
Start Date: {{Date}}
Notes: {{Notes}}
Setup Steps
Create your template Google Doc and note its file ID.
Create or choose a destination folder in Drive and note its folder ID.
Prepare your Google Sheet with recommended headers.
Open the Sheet → Extensions → Apps Script → paste the code below.
Configure constants (TEMPLATE_ID, OUTPUT_FOLDER_ID, FILENAME_PATTERN).
Save the project. Grant permissions on first run.
Set triggers:
Installable “On edit” → choose function onEditInstallable
OR installable “On form submit” → choose function onFormSubmit
Test:
Check the Generate box or submit a new form response.
Confirm Status, Doc URL, and Generated At populate.
Open the created document and verify replacements.
Apps Script Code
Paste this into the bound Apps Script project of your Google Sheet. Update TEMPLATE_ID, OUTPUT_FOLDER_ID, and optionally FILENAME_PATTERN to match your environment.

/***** CONFIG *****/
const CONFIG = {
  TEMPLATE_ID: 'REPLACE_WITH_TEMPLATE_DOC_ID',
  OUTPUT_FOLDER_ID: 'REPLACE_WITH_DESTINATION_FOLDER_ID',
  TRIGGER_COLUMN_NAME: 'Generate',
  STATUS_COLUMN_NAME: 'Status',
  URL_COLUMN_NAME: 'Doc URL',
  TIMESTAMP_COLUMN_NAME: 'Generated At',
  FILENAME_PATTERN: '{{Project}} - {{Name}} - {{Date}}', // customize as needed
  CLEAR_TRIGGER_AFTER: true, // uncheck "Generate" after success
  SKIP_IF_URL_EXISTS: true, // idempotency: skip if Doc URL already present
  LOG_SHEET_NAME: null // optional: set to a sheet name for a dedicated log
};

/***** MENU *****/
function onOpen() {
  SpreadsheetApp.getUi()
    .createMenu('Docs Automation')
    .addItem('Generate for all pending rows', 'generateDocsForAllPending')
    .addToUi();
}

/***** ENTRY POINTS *****/
// Installable trigger: choose "onEditInstallable" in Triggers for "On edit"
function onEditInstallable(e) {
  if (!e || !e.range) return;
  const sheet = e.range.getSheet();
  if (!isPrimarySheet_(sheet)) return;

  const headers = getHeaders_(sheet);
  const colName = headers[e.range.getColumn() - 1];
  if (!colName) return;

  // Only act if the trigger column changed
  if (normalize_(colName) === normalize_(CONFIG.TRIGGER_COLUMN_NAME)) {
    const rowIndex = e.range.getRow();
    if (rowIndex === 1) return; // ignore header
    processRow_(sheet, rowIndex, headers);
  }
}

// Installable trigger: choose "onFormSubmit" in Triggers for "On form submit"
function onFormSubmit(e) {
  const sheet = e.range ? e.range.getSheet() : SpreadsheetApp.getActiveSheet();
  if (!isPrimarySheet_(sheet)) return;

  const headers = getHeaders_(sheet);
  const rowIndex = e.range ? e.range.getRow() : sheet.getLastRow();
  processRow_(sheet, rowIndex, headers);
}

// Manual menu action
function generateDocsForAllPending() {
  const sheet = SpreadsheetApp.getActiveSheet();
  if (!isPrimarySheet_(sheet)) return;

  const headers = getHeaders_(sheet);
  const data = sheet.getDataRange().getValues();
  let processed = 0;

  for (let r = 2; r <= data.length; r++) {
    processed += processRow_(sheet, r, headers) ? 1 : 0;
  }
  log_(`Processed ${processed} row(s).`);
}

/***** CORE LOGIC *****/
function processRow_(sheet, rowIndex, headers) {
  try {
    const headerMap = indexHeaders_(headers);
    ensureControlColumnsExist_(sheet, headers, headerMap);

    // Refresh if columns were added
    const currentHeaders = getHeaders_(sheet);
    const curHeaderMap = indexHeaders_(currentHeaders);

    // Read row values
    const rowValues = sheet.getRange(rowIndex, 1, 1, currentHeaders.length).getValues()[0];
    const rowObj = rowToObject_(currentHeaders, rowValues);

    // Idempotency: skip if URL already exists
    const url = rowObj[CONFIG.URL_COLUMN_NAME];
    if (CONFIG.SKIP_IF_URL_EXISTS && url) {
      return false;
    }

    // Check trigger
    const triggerVal = rowObj[CONFIG.TRIGGER_COLUMN_NAME];
    if (!isTriggered_(triggerVal)) {
      return false;
    }

    // Generate the document
    const result = generateDocFromTemplate_(rowObj);

    // Write results back
    const updates = {};
    updates[CONFIG.STATUS_COLUMN_NAME] = 'Done';
    updates[CONFIG.URL_COLUMN_NAME] = result.url;
    updates[CONFIG.TIMESTAMP_COLUMN_NAME] = formatNow_();

    if (CONFIG.CLEAR_TRIGGER_AFTER) {
      updates[CONFIG.TRIGGER_COLUMN_NAME] = false;
    }

    writeBack_(sheet, rowIndex, currentHeaders, updates);
    log_(`Created doc "${result.name}" for row ${rowIndex}: ${result.url}`);
    return true;

  } catch (err) {
    const updates = {};
    updates[CONFIG.STATUS_COLUMN_NAME] = `Error: ${err.message}`;
    writeBack_(sheet, rowIndex, getHeaders_(sheet), updates);
    log_(`Error processing row ${rowIndex}: ${err.stack || err}`, true);
    return false;
  }
}

function generateDocFromTemplate_(rowObj) {
  // Prepare folder and template
  const templateFile = DriveApp.getFileById(CONFIG.TEMPLATE_ID);
  const folder = DriveApp.getFolderById(CONFIG.OUTPUT_FOLDER_ID);

  // Build file name
  const filename = interpolate_(CONFIG.FILENAME_PATTERN, rowObj);

  // Copy template
  const newFile = templateFile.makeCopy(filename, folder);

  // Open doc and perform replacements
  const doc = DocumentApp.openById(newFile.getId());
  const body = doc.getBody();

  // Replace placeholders for each key (header)
  Object.keys(rowObj).forEach(key => {
    const val = stringifyValue_(rowObj[key]);
    const regex = '\\{\\{\\s*' + escapeRegExp_(key) + '\\s*\\}\\}';
    body.replaceText(regex, val);
  });

  doc.saveAndClose();

  return {
    id: newFile.getId(),
    url: newFile.getUrl(),
    name: filename
  };
}

/***** HELPERS *****/
function isPrimarySheet_(sheet) {
  // Optionally limit to a named sheet; return true to allow all sheets
  return true;
}

function getHeaders_(sheet) {
  const lastCol = sheet.getLastColumn();
  if (lastCol === 0) return [];
  return sheet.getRange(1, 1, 1, lastCol).getValues()[0].map(h => (h || '').toString().trim());
}

function indexHeaders_(headers) {
  const map = {};
  headers.forEach((h, i) => map[h] = i);
  return map;
}

function rowToObject_(headers, rowValues) {
  const obj = {};
  headers.forEach((h, i) => {
    obj[h] = rowValues[i];
  });
  return obj;
}

function writeBack_(sheet, rowIndex, headers, updates) {
  const headerMap = indexHeaders_(headers);
  Object.keys(updates).forEach(k => {
    let col = headerMap[k];
    // If column is missing, append it to the end
    if (col === undefined) {
      col = headers.length;
      sheet.getRange(1, headers.length + 1).setValue(k);
      headers.push(k);
    }
    sheet.getRange(rowIndex, col + 1).setValue(updates[k]);
  });
}

function isTriggered_(val) {
  // Accept TRUE, "TRUE", "Yes", "yes", "Y", 1
  if (val === true || val === 1) return true;
  const s = (val || '').toString().trim().toLowerCase();
  return ['true', 'yes', 'y', '1'].includes(s);
}

function escapeRegExp_(s) {
  return (s || '').toString().replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
}

function interpolate_(pattern, rowObj) {
  if (!pattern) return 'Generated Document';
  return pattern.replace(/\{\{\s*([^}]+?)\s*\}\}/g, (_, key) => {
    const raw = rowObj[key] !== undefined ? rowObj[key] : '';
    return stringifyValue_(raw);
  });
}

function stringifyValue_(v) {
  if (v === null || v === undefined) return '';
  if (Object.prototype.toString.call(v) === '[object Date]' && !isNaN(v)) {
    return Utilities.formatDate(v, Session.getScriptTimeZone(), 'yyyy-MM-dd');
  }
  if (typeof v === 'boolean') return v ? 'Yes' : 'No';
  return String(v);
}

function formatNow_() {
  return Utilities.formatDate(new Date(), Session.getScriptTimeZone(), 'yyyy-MM-dd HH:mm:ss');
}

function ensureControlColumnsExist_(sheet, headers, headerMap) {
  const required = [
    CONFIG.TRIGGER_COLUMN_NAME,
    CONFIG.STATUS_COLUMN_NAME,
    CONFIG.URL_COLUMN_NAME,
    CONFIG.TIMESTAMP_COLUMN_NAME
  ];
  let changed = false;
  required.forEach(name => {
    if (headerMap[name] === undefined) {
      sheet.getRange(1, headers.length + 1).setValue(name);
      headers.push(name);
      headerMap[name] = headers.length - 1;
      changed = true;
    }
  });
  if (changed) {
    SpreadsheetApp.flush();
  }
}

function log_(message, isError) {
  if (CONFIG.LOG_SHEET_NAME) {
    const ss = SpreadsheetApp.getActiveSpreadsheet();
    const logSheet = ss.getSheetByName(CONFIG.LOG_SHEET_NAME) || ss.insertSheet(CONFIG.LOG_SHEET_NAME);
    logSheet.appendRow([formatNow_(), isError ? 'ERROR' : 'INFO', message]);
  }
  console[isError ? 'error' : 'log'](message);
}
Notes and Best Practices
Installable triggers are required. Simple triggers (onEdit) cannot access Drive/Docs services that need authorization.
Keep placeholder names in the template exactly matching the sheet headers for simpler mapping.
Use idempotency safeguards (Doc URL column) to prevent duplicate document creation.
Use a Generate column as a safe manual trigger when not using form submissions.
Consider alternative triggers (time-driven) for batching.
Enhancements (Optional)
Export the generated Doc as PDF and store the link.
Email the document to a recipient column from the sheet.
Create or select per-row folders using a Folder ID column.
Support multiple templates selected per row.
Add validation to ensure required fields are present before generation.
10 Key Points With Comments
#	Key Point	Comment
1	Installable triggers	Use installable on edit or on form submit; simple triggers can’t access Drive/Docs requiring auth.
2	Header-driven mapping	Template placeholders {{Header}} map directly to sheet headers, eliminating manual mapping.
3	Placeholder syntax	Double braces with whitespace tolerance; regex-safe replacement to avoid partial matches.
4	File naming pattern	Build names from data via a pattern like {{Project}} - {{Name}} - {{Date}}.
5	Output folder control	Set a destination folder ID; consider per-row folder logic if needed.
6	Idempotency	Skip generation if Doc URL already exists; prevents duplicates on repeated edits.
7	Error handling	Catch errors per row, write Status with error message, and log details.
8	Data formatting	Convert Dates to yyyy-MM-dd and booleans to Yes/No for cleaner output.
9	Performance and quotas	Batch runs via menu; respect Apps Script quotas for Drive/Docs operations.
10	Extensibility	Add PDF export, email notifications, multi-template selection, and validation.
JSON Based on the Table
[
  {
    "id": 1,
    "key_point": "Installable triggers",
    "comment": "Use installable on edit or on form submit; simple triggers can’t access Drive/Docs requiring auth."
  },
  {
    "id": 2,
    "key_point": "Header-driven mapping",
    "comment": "Template placeholders {{Header}} map directly to sheet headers, eliminating manual mapping."
  },
  {
    "id": 3,
    "key_point": "Placeholder syntax",
    "comment": "Double braces with whitespace tolerance; regex-safe replacement to avoid partial matches."
  },
  {
    "id": 4,
    "key_point": "File naming pattern",
    "comment": "Build names from data via a pattern like {{Project}} - {{Name}} - {{Date}}."
  },
  {
    "id": 5,
    "key_point": "Output folder control",
    "comment": "Set a destination folder ID; consider per-row folder logic if needed."
  },
  {
    "id": 6,
    "key_point": "Idempotency",
    "comment": "Skip generation if Doc URL already exists; prevents duplicates on repeated edits."
  },
  {
    "id": 7,
    "key_point": "Error handling",
    "comment": "Catch errors per row, write Status with error message, and log details."
  },
  {
    "id": 8,
    "key_point": "Data formatting",
    "comment": "Convert Dates to yyyy-MM-dd and booleans to Yes/No for cleaner output."
  },
  {
    "id": 9,
    "key_point": "Performance and quotas",
    "comment": "Batch runs via menu; respect Apps Script quotas for Drive/Docs operations."
  },
  {
    "id": 10,
    "key_point": "Extensibility",
    "comment": "Add PDF export, email notifications, multi-template selection, and validation."
  }
]
10 Follow-up Questions
What exact headers exist in your sheet, and do they already match the desired template placeholders?
Which trigger should we enable by default: installable on edit, on form submit, or both?
What is your preferred file naming convention, and should it ensure uniqueness (e.g., include a timestamp or unique ID)?
Should the destination folder be global, or do you need per-row destination folders based on a column?
Do you want the script to clear the Generate flag after success, or keep it for auditing?
Should the workflow also create a PDF copy and store an additional “PDF URL” column?
Do you want email notifications sent to a column (e.g., Email) with a link or attachment when a document is created?
How should date, currency, and other special fields be formatted in the generated document?
Do you need validation (required fields check) before generation, and how should missing data be handled?
What volume of rows per run do you expect, and should we add throttling/batching to respect Apps Script quotas?
