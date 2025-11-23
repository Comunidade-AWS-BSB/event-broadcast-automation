# Legacy n8n + Evolution Automation

Before we built the Amplify platform, the user group automated reminders with n8n workflows that read Google Sheets, matched the upcoming events, and sent WhatsApp messages via the same Evolution API instance that we self-host today. This section summarizes the steps from the [original repository](https://github.com/Comunidade-AWS-BSB/Solu-o-Auotma-o-para-Eventos).

## 1. Google Sheets integration
1. Create a project in Google Cloud Console and enable the **Google Sheets API**.
2. Configure the OAuth consent screen and create OAuth client credentials.
3. Point the redirect URI to the n8n callback, e.g. `https://n8n.seudominio.com.br/rest/oauth2-credential/callback`.
4. Store the client ID/secret in n8n credentials.

## 2. Provision AWS resources
- Launch an **EC2 t3.small** (free tier-compatible) in the target region.
- Install Docker Engine, Docker Compose, and Nginx.
- Allocate an Elastic IP and update DNS records for `n8n.*` and `evolution.*` hosts.

## 3. Reverse proxy & HTTPS
See `evolution-self-hosting.md` for the exact Nginx and Certbot commands. The important idea: n8n listens on port 5678 and Evolution on 8081, both hidden behind HTTPS virtual hosts.

## 4. Docker Compose deployment
Clone the legacy repo and run `docker compose up -d` to bring up n8n + Evolution.

## 5. Evolution API pairing
1. Go to `https://evolution.seudominio.com.br`.
2. Pair the WhatsApp number inside Evolution Manager.
3. Copy the API key for use inside n8n (and now inside Amplify Lambdas).

## 6. n8n Google Sheets credential
Within `https://n8n.seudominio.com.br`, create an OAuth2 credential referencing the Google project created earlier.

## 7. Install the Evolution node in n8n
In **Settings → Nodes**, add the Evolution node package and configure it with the API key above.

## 8. Build the workflow
1. **Trigger** – Recurrence node, e.g., daily at 06:00.
2. **Fetch events** – Google Sheets node reading the events spreadsheet with a filter `data_evento - 7 dias == hoje`.
3. **Fetch contacts** – Google Sheets node for participant list.
4. **Send reminders** – HTTP Request node hitting `POST /message/sendText` with templated message per contact.

## Why migrate?
This flow worked well but required manual upkeep of spreadsheets and n8n nodes. The current Amplify platform keeps the same Evolution API foundation while providing a first-party UI, DynamoDB storage, EventBridge scheduling, and Cognito-sourced audiences. The historical instructions remain valuable for UG leaders who want to replicate the bare-bones setup quickly.
