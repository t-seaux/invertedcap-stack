---
name: shared-references
description: Shared reference files used by multiple skills. Not a standalone skill — do not trigger directly. Skills reference files in this directory via explicit read instructions in their own SKILL.md.
---

# Shared References

This directory contains shared reference files that are read by other skills at runtime. Files here define constants, conventions, and reusable instructions that would otherwise be duplicated across multiple skill files.

## Contents

- `add-link-to-diligence-materials.md` — Source of truth for programmatically adding external URLs to the Diligence Materials Files property on Notion Opportunity pages via Notion's internal API (`/api/v3/saveTransactions`) through Chrome `javascript_tool`. Includes the full function, usage examples, and fallback behavior.
- `claude-note-icon.md` — Defines the custom Claude logo emoji ID and the `notion-update-page` call pattern for setting it as the icon on Claude-generated Notes DB entries.
- `drive-upload.md` — Apps Script endpoint for uploading files directly to Google Drive via base64-encoded POST requests. Works in any environment with no Chrome, filesystem mount, or Zapier dependency. Used by `investor-update`, `materials-handler`, and any skill that needs to save files to Drive. Supports any Drive folder via the `folderId` parameter.
- `gmail-attachment-saver.md` — Apps Script endpoint for saving Gmail attachments directly to Google Drive. Works in any environment with no Chrome or filesystem mount dependency. Used by `materials-handler` and `investor-update`.
- `long-form-pdf-spec.md` — Canonical reportlab format spec for authored long-form PDFs (diligence memos, pre-mortems, LP letters, investor reports). Covers page setup, typography, spacing, tables, bullets, hyperlinks, and header structure. Used by `first-pass-diligence`, `update-diligence-priors`, and any skill producing an authored long-form PDF. Not for source-document preservation renders.
- `pdf_builder_template.py` — Copy-and-customize reportlab builder that implements `long-form-pdf-spec.md`. Handles Notion API encoding artifacts, enhanced-markdown HTML tables, pipe tables, and markdown-to-reportlab XML conversion. Four CUSTOMIZE blocks at the top cover output path, canonical URL, title, date, and input content.

## Usage

Skills that need a shared reference will include an explicit instruction like:

> Read the shared reference at `/Users/tomseo/.claude/skills/shared-references/claude-note-icon.md` and follow its instructions.

Do not trigger this skill directly.
