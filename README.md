# README / Setup Instructions

1) Overview

This n8n workflow connects an inbound WhatsApp webhook (e.g., Twilio, 360dialog, or Meta Business API) to Google Drive and an LLM (OpenAI/Anthropic) for AI-powered summaries. It supports the commands you requested: LIST, DELETE, MOVE, SUMMARY, RENAME, UPLOAD.

2) High-level Flow
- WhatsApp message arrives at the Webhook node.\n- Function node parses command + media metadata.\n- Switch routes to the appropriate Google Drive node or OpenAI summary.\n- For uploads, HTTP Request downloads media; Google Drive uploads it.\n- Replies are built and sent back via the responding Webhook or an HTTP call to your WhatsApp provider's API.

3) Google Drive OAuth2
- In n8n, create Google Drive credentials using OAuth2. Scopes required: `https://www.googleapis.com/auth/drive` (full drive access) or narrower scopes like `https://www.googleapis.com/auth/drive.file` + `https://www.googleapis.com/auth/drive.metadata.readonly` depending on needs.
- Use the Drive credentials in all Google Drive nodes.

4) WhatsApp Provider
- Recommended: Twilio WhatsApp (easy webhook + media URLs) or 360dialog / Meta Business API.
- Configure the provider to POST incoming messages to the n8n webhook path: `https://<your-n8n-host>/webhook/whatsapp-drive-webhook`.
- Ensure media URLs are publicly accessible (Twilio gives temporary URLs you can download).

5) AI Summaries
- Use n8n OpenAI node (or HTTP Request to Anthropic). Provide your OpenAI API key in n8n credentials.
- The workflow downloads textual content from Drive files (you'll need an extra step to fetch file content for non-binary Drive files) and sends to the OpenAI node. For PDFs, add a PDF-to-text extraction step (e.g., call an external PDF parsing microservice or use an n8n function with a hosted PDF parser). The example assumes you will fetch the file content into `$json.text`.

6) Implementation Notes & Enhancements
- Folder lookup: The workflow uses a Search Folder node first (implement using Drive "search" by name) and stores `driveFolderId` to downstream nodes.
- File identification: For file-level commands, search the Drive for the exact path or file name and pass `fileId` to delete/move/rename nodes.
- Security: Use per-user Drive OAuth if multiple users; otherwise use a service account with a shared Drive.
- Replies to WhatsApp: Many providers require a separate API call (Twilio's Messages API) to send replies. Use an HTTP Request node to call Twilio with your message; or enable "respond" in the webhook if your provider supports immediate reply.

7) Deployment
- Export the workflow JSON (this file) and import into your n8n instance (Settings → Workflows → Import).\n- Configure credentials: Google Drive (OAuth2), OpenAI (API key), HTTP Request node credentials for WhatsApp provider if needed.

8) Limitations & To-do
- PDF text extraction is not fully wired here — you must add a PDF parsing node or external service.\n- Disambiguation when multiple folders share the same name; consider using full paths or present a numbered list to the user to choose.
- Rate-limiting and error-handling should be added (retry nodes, catch errors, and notify users).



# Example WhatsApp commands (user-facing)

- `LIST /Reports` → lists files in folder named Reports.
- `DELETE /Reports/old-file.pdf` → deletes the file.
- `MOVE /Reports/report.pdf /Archive` → moves file to Archive folder.
- `SUMMARY /Reports` → LLM summarizes text-bearing files in that folder.
- `RENAME report.pdf report_v2.pdf` → renames file in Drive.
- Upload flow: send the file in WhatsApp with message: `UPLOAD /Reports new_name.pdf` → file will be saved into Reports as new_name.pdf.


