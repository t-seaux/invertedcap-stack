---
name: uhc-superbill-filer
description: >
  File an out-of-network mental health superbill on the UnitedHealthcare member portal.
  Parses Dr. Jain's monthly invoice PDF, drives Chrome through the full claim submission
  flow, verifies OCR-extracted data, fills provider details manually, and submits.
  Trigger whenever Tom uploads a superbill or invoice from Dr. Jain / Sargam Jain and says
  anything like "file this", "submit this to UHC", "file my superbill", "submit my
  therapy invoice", "insurance claim", "file with united healthcare", "UHC claim",
  "submit to insurance", or provides a therapy/mental health invoice with intent to file.
  Also trigger if Tom says "superbill" in any context related to filing or reimbursement.
  Even if Tom just drops an invoice PDF without further instruction, if the filename or
  content matches the known superbill format (Sargam Jain, CPT 99214+90836), trigger
  this skill automatically.
---

# UHC Superbill Filer

This skill automates filing out-of-network mental health superbills on the UnitedHealthcare
member portal (member.uhc.com). The monthly invoices come from Dr. Sargam Jain and follow a
consistent format. The portal requires navigating a 6-step wizard, and UHC's OCR consistently
fails to extract provider information, making manual entry necessary every time.

## Prerequisites

- **Claude in Chrome** must be available and connected
- Tom must be **logged into member.uhc.com** in the active Chrome browser
- The invoice PDF must be accessible (uploaded to the conversation or at a known path)

If Chrome is not available or Tom is not logged in, tell him what's needed before proceeding.

## Step 0: Parse the Invoice

Before touching Chrome, read the uploaded invoice PDF and extract:

1. **Month and year** of service (e.g., "March 2026")
2. **Session dates** – the day-of-month numbers listed on the left side (e.g., 2, 9, 16, 30)
3. **Number of sessions** – count of session dates
4. **CPT codes per session** – should be 99214 and 90836
5. **Rate per code** – should be $225.00 each
6. **Total amount** – should equal (number of sessions × 2 × $225)
7. **Provider info** – Sargam Jain, MD; NPI 1114184173; Tax ID 15-8668032
8. **Diagnosis code** – F43.20

Cross-check the total against the computed amount (sessions × $450). If there is any
discrepancy, stop and flag it to Tom before proceeding.

Read `references/static-data.md` for the full set of static values used throughout the flow.

## Step 1: Navigate to Submit a Claim

Use Claude in Chrome to:

1. Navigate to `https://member.uhc.com/claims-and-accounts/submit-claim`
2. Wait for the page to load
3. Select **"Service"** (Medical and mental health treatments and therapies)
4. After Step 2 appears, select **"Mental Health and Substance use"**
5. Click **"Begin request"**

## Step 2: Click Through Intro Page

The intro page at `/claims-assistant/dmr/Intro` lists documentation requirements.
No action needed except clicking **"Continue"**.

## Step 3: Upload the Invoice

On the Upload Documents page (`/claims-assistant/dmr/UploadDocuments/`):

1. Upload the invoice PDF using the file upload area
2. Confirm the file appears in "Attached files (1 of 10)"
3. Click **"Next step"**

On the Supplemental Documents page (`/claims-assistant/dmr/UploadSupplemental`):

1. Do NOT check any boxes (no proof of timely filing, no provider reports needed)
2. Click **"Next step"**

## Step 4: Fill Patient Information

On the Patient Info page (`/claims-assistant/dmr/PatientInfo`):

1. **Email address** – should be pre-populated as `thomas.seo@outlook.com`. Verify, don't change.
2. **Is UnitedHealthcare your primary insurance provider?** – select **"Yes"**
3. **Where were the services rendered?** – select **"Office"**
4. Click **"Next step"**

## Step 5: Wait for OCR Scan

The portal runs an OCR scan on the uploaded invoice (`/claims-assistant/dmr/scanning`).
This takes 30–90 seconds. Wait for "Scan Complete" to appear, then click **"Review results"**.

The scan will jump you to Step 6 (Review) with auto-populated fields. The key thing to know
is that UHC's OCR consistently extracts the diagnosis and service codes correctly but **always
fails to extract the provider information**. This is expected behavior – the skill must handle it.

## Step 6: Verify OCR Results and Fix Provider

On the Review page (`/claims-assistant/dmr/ReviewPage`), verify every section against the
parsed invoice data:

### 6a: Verify Claim Info and Member Details

Check that these match (they should be pre-populated correctly):
- Claim category: Service
- Claim type: Mental Health and Substance use
- Email: thomas.seo@outlook.com
- Member who received care: THOMAS SEO
- Primary Insurance: UnitedHealthcare
- Place of service: Office

### 6b: Add Provider Information

The Provider Details section will show a red error: "Provider Information is required."
Click **"Add provider information"**.

This goes to the Provider Search page (`/claims-assistant/dmr/ProviderSearch`).
The search will either be blank or pre-filled with the TIN – either way, it won't find
results because Dr. Jain is out-of-network. Click **"Enter provider details manually"**.

On the manual entry page (`/claims-assistant/dmr/ProviderDetails`), fill in:

| Field | Value |
|-------|-------|
| Provider last name | Jain |
| Provider first name | Sargam |
| Provider Tax Identification Number (TIN) | 158668032 |
| Street address | 11 5th Avenue, Office 7 |
| City | New York |
| State | NY |
| Zip code | 10003 |
| Phone number | (leave blank) |

Click **"Save changes"**, then click **"Yes, confirm"** on the confirmation dialog.

### 6c: Verify Diagnosis

The diagnosis should show:
- Diagnosis #1: **F4320** – ADJUSTMENT DISORDER UNSPECIFIED

This maps to F43.20 from the invoice (the portal strips the dot). Confirm it matches.

### 6d: Verify Service Codes

The portal should show **(number of sessions × 2)** service entries, alternating between
99214 and 90836. For a typical 4-session month, that is 8 service entries.

Expand each service entry and verify:
- The CPT code alternates correctly (99214, 90836, 99214, 90836, ...)
- Each entry has **Quantity: 1**
- Each entry has **Amount you paid: $225.00**
- The **dates of service** pair correctly:
  - Services #1 and #2 → first session date
  - Services #3 and #4 → second session date
  - Services #5 and #6 → third session date
  - And so on...
- All dates use the correct month and year from the invoice

If any field is wrong, use the "Edit details" link to correct it.

### 6e: Report Verification Results

Before submitting, briefly tell Tom what you verified and flag any corrections made.
Format: "Verified N service entries across M dates, diagnosis F4320, provider Jain.
[All correct / Corrected: <what was wrong>]."

### 6f: Check Agreement and Submit

1. Check the **Electronic Agreement** checkbox
2. Click **"Submit claim"**

## Step 7: Confirm Submission

On the confirmation page (`/claims-assistant/dmr/web-confirmation`), capture:

1. **Reference number** (the UUID shown in the Submission reference section)
2. **Submission date**
3. **Claim category and type**

Report these to Tom as confirmation that the claim was filed successfully. Remind him
that UHC says claims are typically processed in 15–30 days.

## Error Handling

- **Not logged in**: If any page redirects to a login screen, tell Tom to log in and
  then say "continue" to resume.
- **OCR misreads**: If the scanner extracts wrong dates, amounts, or codes, use the
  "Edit details" link on the review page to correct them. Always cross-check against
  the parsed invoice, not just the static data.
- **Upload fails**: If the file upload doesn't register, try the upload again. Accepted
  formats: PNG, PDF, JPEG, JPG, TIF, TIFF, HEIF. Max 25MB.
- **Provider search returns results**: If UHC ever starts returning results for the
  provider search (unlikely), verify the match is correct before selecting it. If the
  details match Dr. Jain, select it instead of entering manually.
- **Confirmation page not reached**: If submission fails or shows an error, capture the
  error message and report it to Tom. Do NOT retry automatically – let Tom decide.
