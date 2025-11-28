# 8793PartBot

Discord bot + Google Apps Script integration for FRC Team 8793 “Pumpkin Bots”.

## Features

- Discord slash commands for part requests (`/requestpart`)
- Inventory lookup (`/inventory`)
- Open orders & ETA (`/openorders`)
- Google Sheets–based pipeline (Requests, Orders, Inventory)
- AI enrichment for Part Name, SKU, Price, Vendor stock

## Local Setup

1. Clone this repo
2. Create a `.env` file with:

   - DISCORD_TOKEN=
   - CLIENT_ID=
   - GUILD_ID=
   - APPS_SCRIPT_URL=
   - (optional) OPENAI_API_KEY=

3. Install dependencies:

   ```bash
   npm install

