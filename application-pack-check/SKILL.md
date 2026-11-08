---
name: application-pack-check
description: Private readiness check and copy-only document pack for a visa, university admission, scholarship or job application, built from the personal papers, scans and phone photos already stored on the NAS. Use when the user says things like "Are my documents ready for my student visa?", "Prepare my admission document pack" or "Check my papers before I apply".
---

# Goal

Help one applicant find out, before they apply, whether every document on their checklist is on the NAS, readable, still valid on the submission date, and issued in the same name. The result is a readiness report (found, missing, expired or expiring, name mismatch, unclear) and, only after the user confirms, a new pack folder that holds copies of the chosen files with clear names. Original files are never moved, renamed or deleted.

Everything stays on the NAS. This Skill does not upload, email or share any document, and it is not legal or immigration advice: the user decides what the checklist is and what to submit.

A plain-language guide for users is in [references/user-guide.md](references/user-guide.md).

Supported scope: one applicant and one application per run, a checklist of up to 25 items, and only the folders and photo library the user allows.

# Required input

Collect these from the user before the first delegation. Ask once, in one message.

1. Application name, for example "UK Student Visa 2027". It is used for the pack folder name, the tag and the report title.
2. Applicant full name exactly as it appears on the passport or national ID.
3. Planned submission date. Expiry checks use this date.
4. The checklist. The user may type it or pick a starter list from [references/checklist-starters.md](references/checklist-starters.md) and then edit it. Always show the final checklist and wait for the user to confirm it.
5. Where to search: the folders the user allows, and whether to include photos of documents from the photo library.
6. Where to create the pack folder. If the user has no preference, propose a folder named after the application inside the first allowed folder and ask them to confirm.

Do not ask for passwords, full document numbers, bank logins or any other secrets. Do not pre-collect parameters that the child experts resolve themselves.

# Generation-time capability evidence

- `ug-filesystem` → `file-agent`
- `ug-filemgr` → `file-agent`
- `ug-photo` → `photo-agent`
- `ug-document` → `document-agent`
- `ug-knowledge` → `knowledge-agent`

# Workflow

Run the steps in order. Every step uses only the real result returned by the previous delegation, never a guessed path, ID or value. If a step that later steps depend on does not return `completed`, stop those later steps and report what is missing.

## Step 1: Find candidate documents (file-agent)

delegate --target file-agent --instruction 'Search only inside the folders the user allowed for this run, including subfolders. Return every PDF, Word document and image file that could be a personal or official document, such as a passport, national ID, birth certificate, degree, transcript, bank statement, offer letter, reference letter, photo ID or certificate. For each file return its real path, file name, type, size and modified date. Do not open, move, rename or change any file. If a folder is not accessible, report it as not accessible and continue with the others. If there are no results, say that no matching files were found.'

- `completed` with files: keep the real list of paths for Step 2 and Step 3.
- `completed` with no results: tell the user that no documents were found in the allowed folders, suggest checking the folder choice, and stop.

## Step 2: Read photographed and scanned documents (photo-agent)

Only run this step if the user allowed photos, or if Step 1 returned image files.

delegate --target photo-agent --instruction 'For the image file paths returned by the file search in this run, run OCR on each image and return the recognised text per path. If the user allowed the photo library, also search it for photos recognised as documents (category file-image-recognition) taken in the last ten years and return their real paths and recognised text. Do not create albums, edit photos or share anything. Mark an image as unreadable if the text is blurred, cropped or too short to identify the document. If there are no results, say so.'

- `completed`: pass the real OCR text per path to Step 3.
- `completed` with no readable images: continue with Step 3 using only the documents from Step 1, and note that no photographed documents could be read.

## Step 3: Identify documents and extract key facts (document-agent)

delegate --target document-agent --instruction 'Use only the real file paths from the file search in this run and the OCR text returned by the photo step in this run. For each document, identify which checklist item it matches from the confirmed checklist, and extract: document type, holder full name, date of birth if shown, issuing authority, issue date, expiry date, and only the last four characters of any document number. Then compare the holder name and date of birth across all documents against the applicant full name the user gave, and list every difference, including spelling, order of names, missing middle names and initials. Give a confidence of high, medium or low for each match. If a value is not printed on the document, return not found instead of guessing. Do not translate names. If there are no results for a checklist item, report that item as missing.'

- `completed`: build the readiness report in Step 4 from this real result only.
- Partial result: keep the documents that were read, list the documents that could not be read, and continue.

## Step 4: Show the readiness report and ask what to include (no delegation)

Build the report with the layout in [references/report-format.md](references/report-format.md). For every checklist item show one status:

- Ready: a matching document was found, it is readable, the name matches and it is valid on the submission date.
- Expiring: valid on the submission date but expires within six months after it.
- Expired: the expiry date is before the submission date.
- Name mismatch: the holder name or date of birth differs from the applicant details or from another document.
- Unclear: low confidence, unreadable scan or more than one candidate file.
- Missing: no matching document was found.

Then ask the user to choose which files go into the pack (one file per checklist item), and whether to tag the originals. This is a `waiting_user` point. Do not create, copy or tag anything until the user replies.

## Step 5: Create the pack folder and copy the chosen files (file-agent)

delegate --target file-agent --instruction 'Create the pack folder the user confirmed for this application if it does not exist. Copy, never move, each file the user chose in this run into that folder. Name each copy with its checklist number and item, for example 01-passport.pdf, keeping the original file extension. Do not overwrite an existing file: if a name is taken, add a short number to the new name. Leave every original file unchanged. Return the real path of the pack folder and the real path of each copy, and list any copy that failed with its reason.'

This step changes the NAS, so the platform asks the user for approval. Report `waiting_approval` to the user and wait. If the user declines, stop and keep the report on screen only.

## Step 6: Tag the original files (file-agent, optional)

Only if the user asked for tags.

delegate --target file-agent --instruction 'Add one tag named after the application the user confirmed in this run to each original file the user chose. Create the tag if it does not exist. Do not change, move or rename the files. Return which files were tagged and any that failed.'

## Step 7: Save the report inside the pack folder (knowledge-agent)

delegate --target knowledge-agent --instruction 'Save the readiness report text built in this run as a Markdown file named readiness-report in the real pack folder path returned by the copy step in this run. Save the text exactly as provided. Do not add document numbers beyond the last four characters. Return the real saved file path and confirm it was verified.'

Finally, tell the user where the pack folder and the report are, how many items are Ready, and the most urgent action, for example "Your passport expires before the submission date, renew it first".

# Completion and failure

- `completed`: report only what the expert actually returned. Never state that a document exists, is valid or was copied unless the delegation result says so.
- `failed`: say which step failed and why, keep any results from earlier steps, and stop the steps that depend on it. For example, if Step 3 fails, still show the list of files found in Step 1, and do not create the pack.
- `cancelled`: stop at once, change nothing else, and tell the user what was already done, for example "2 of 6 files were copied before you cancelled".
- `waiting_user`: ask the user the open question in plain words and wait. Do not choose for them.
- `waiting_approval`: explain what will change (folder created, files copied, tags added) and wait for the user to approve. Continue only after the platform confirms the approval.
- Empty or partial result: if no matching files are found, say so and stop instead of widening the search or using unrelated files. If some documents cannot be read, mark them Unclear and continue with the rest.
- Never invent document details, dates or names. If an expiry date is not printed, show "not found" and tell the user to check the paper document.
- Privacy: keep all document content on the NAS, show only the last four characters of document numbers, and never send, upload or share files.
