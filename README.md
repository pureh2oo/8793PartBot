# 8793PartBot — Automated Parts Purchasing System for FRC Robotics

## Overview
8793PartBot is an automation system built for **FRC Team 8793 – Pumpkin Bots** to streamline:
- Part requests from students via Discord slash commands
- **User-specified SKU override** for multi-variant product pages (NEW!)
- Automatic SKU/name extraction via AI (Google Gemini 2.5 Flash)
- Real-time inventory lookup
- Student self-service request cancellation
- Purchasing approvals and workflow management
- Order tracking with ETA notifications
- Discord ↔ Google Sheets integration

It replaces DM chaos, ad‑hoc spreadsheets, and manual vendor lookups with a structured, automated workflow.

---

## Features

### 🎯 User SKU Override (NEW!)
Students can now specify exact SKUs when requesting parts from multi-variant product pages:
```
/requestpart subsystem:Drive link:https://wcproducts.com/products/ball-bearings sku:WCP-0785 qty:4
```

**Why this matters:**
- Product pages like WCP ball bearings have multiple SKUs (WCP-0783, WCP-0784, WCP-0785, etc.)
- AI might guess the wrong variant based on notes alone
- User-specified SKU **overrides AI detection** and guarantees the correct part
- Still optional - AI enrichment works normally if SKU not provided

**Example use case:**
```
Without SKU field:
/requestpart subsystem:Drive link:https://wcproducts.com/products/ball-bearings qty:4 notes:"1/2 inch flanged"
→ AI might extract WCP-0783 (wrong variant) ❌

With SKU field:
/requestpart subsystem:Drive link:https://wcproducts.com/products/ball-bearings sku:WCP-0785 qty:4
→ Guaranteed WCP-0785 (correct variant) ✅
```

### 🔧 AI-Powered Part Enrichment (Gemini 2.5 Flash)
Automatically extracts from a vendor URL:
- Part name
- SKU / product code (unless user-specified)
- Estimated price

**Supported vendors:**
- ✅ West Coast Products (WCP)
- ✅ REV Robotics
- ✅ VexPro
- ✅ AndyMark
- ✅ Amazon (URL-based extraction)
- ✅ McMaster-Carr (SKU-based extraction)
- ✅ CTRE
- ✅ Studica
- ✅ Redux Robotics
- ✅ Thrifty Bot

**Fallback strategies:**
- **Amazon:** Extracts product name from URL slug + ASIN
- **McMaster-Carr:** Extracts SKU from URL pattern
- **Other vendors:** Intelligent pattern matching

**Smart SKU handling:**
- If user provides SKU → AI only enriches Part Name and Price
- If user omits SKU → AI enriches Part Name, SKU, and Price
- User SKU always takes precedence over AI detection

### 💬 Discord Slash Commands
Students request parts using intuitive slash commands:
```
/requestpart subsystem:Drive link:<URL> sku:WCP-0785 qty:2 priority:High notes:"For shooter prototype"
```

**All commands:**
- `/requestpart` - Submit a new part request (now with optional `sku` field!)
- `/cancelrequest` - Cancel your own request
- `/inventory` - Search inventory by SKU or keyword
- `/openorders` - View all pending orders and denied requests
- `/orderstatus` - Check status of a specific request or order

### 🚫 Request Cancellation
Students can cancel their own requests:
```
/cancelrequest requestid:REQ-12345678 reason:No longer needed
```

**Security features:**
- ✅ Can only cancel your own requests
- ✅ Cannot cancel if already ordered/received/complete
- ✅ Timestamps and audit trail in mentor notes
- ✅ Row turns gray when cancelled
- ✅ Status set to "🚫 Cancelled"

### 📦 Order Tracking
- `/openorders` – displays all non-received orders
- `/orderstatus requestid:REQ-xxxx` – detailed status for a request
- `/orderstatus orderid:ORD-xxxx` – detailed status for an order
- ETAs shown in human-readable format (e.g., "Dec 15, 2024")
- Tracking numbers displayed when available

### 📚 Inventory Lookup
```
/inventory sku:WCP-0783
/inventory search:bearing
/inventory search:BIN-001
```
Returns:
- Stock on hand (aggregated across multiple locations)
- Location(s)
- Vendor
- Part name

Supports:
- Exact SKU matching
- Fuzzy keyword search
- Location-based lookup (BIN-xxx, RACK-xxx)

### 🛠 Automated Workflow
1. Student submits `/requestpart` in Discord (optionally with specific SKU)
2. Apps Script creates request in Google Sheets
3. **User-provided SKU written immediately** (if specified)
4. **AI enrichment (Gemini)** extracts part details from URL
   - If user provided SKU → AI only fills Part Name and Price
   - If no user SKU → AI fills Part Name, SKU, and Price
5. System checks inventory for existing stock
6. Mentor reviews and approves in Google Sheets
7. **Student can cancel** using `/cancelrequest` (if not yet ordered)
8. Approved requests automatically moved to Orders sheet
9. Order status tracked until received
10. Denied requests flagged in `/openorders` for visibility

**Status workflow:**
- **📥 Submitted** → Initial request state
- **👀 Under Review** → Pending mentor review
- **✅ Approved** → Auto-creates order in Orders sheet
- **🛒 Ordered** → Tracking and ETA management
- **📦 Received** → Auto-adds to inventory with location prompt
- **✔️ Complete** → Marks request as done, grays out row
- **❌ Denied** → Prompts for reason, highlights row red
- **🚫 Cancelled** → Student-initiated cancellation, grays out row

---

## Architecture
```
Discord Slash Commands
        ↓
Node.js Discord Bot (bot.js)
  - Command handling (/requestpart with SKU field, /cancelrequest, etc.)
  - User interaction
  - HTTP requests to Apps Script
  - SKU parameter extraction and validation
        ↓
Google Apps Script Web App (Code.gs)
  - Request routing (doPost)
  - Database operations
  - User SKU handling (priority over AI)
  - Gemini API integration
  - Security checks (cancellation permissions)
        ↓
Google Sheets (Database)
  - Part Requests (pending items with user/AI SKUs)
  - Orders (approved/ordered items)
  - Inventory (on-hand stock)
        ↓
Google Gemini API (gemini-2.5-flash)
  - Part name extraction from URLs
  - SKU extraction (only if user didn't specify)
  - Price extraction
  - Intelligent fallbacks for Amazon/McMaster
```

---

## Repository Structure
```
8793PartBot/
│
├── bot.js                        # Main Discord bot (with SKU field support)
├── package.json
├── package-lock.json
├── .env                          # Environment variables (not in git)
├── .env.example                  # Template for .env
│
├── apps-script/
│   ├── Code.gs                   # Main Apps Script (with user SKU override)
│   ├── SharedConstants.gs        # Shared constants (optional)
│   └── appsscript.json           # Apps Script manifest
│
├── LICENSE
└── README.md                     # This file
```

---

## Setup Instructions

### Prerequisites
- Google Account (for Google Sheets and Apps Script)
- Discord Developer Account
- Google Gemini API Key (free tier available at https://aistudio.google.com/apikey)
- Google Cloud account for hosting the bot (see VM Setup below)

---

### 1. Google Sheets Setup

#### Create Spreadsheet
1. Create a new Google Sheet named "8793 FullInventory and PartPurchasing"
2. Create three tabs with exact names:
   - `Part Requests`
   - `Orders`
   - `Inventory`

#### Part Requests Sheet (Columns A-T)
```
Request ID | Timestamp | Requester | Subsystem | Part Name | SKU | Part Link | Quantity | 
Priority | Needed By | Inventory On-Hand | Vendor Stock | Est Unit Price | 
Total Est Cost | Max Budget | Budget Status | Request Status | Mentor Notes | 
Expedited Shipping | In Inventory
```

#### Orders Sheet (Columns A-O)
```
Order ID | Included Request IDs | Vendor | Part Name | SKU | Qty Ordered | 
Final Unit Price | Total Cost | Order Date | Shipping Method | Tracking | 
ETA | Received Date | Order Status | Mentor Notes
```

#### Inventory Sheet (Columns A-I)
```
SKU | Vendor | Part Name | Location | Qty On-Hand | Reorder Threshold | 
Usage Rate | Last Count Date | Notes
```

---

### 2. Apps Script Setup

#### Create Apps Script Project
1. Open your Google Sheet
2. Go to **Extensions → Apps Script**
3. Delete any default code
4. Copy the complete `Code.gs` from this repository
5. Paste into the Apps Script editor
6. **Save** (Ctrl+S or ⌘+S)

#### Configure Gemini API Key
1. In Apps Script, go to **Project Settings** (⚙️ on left)
2. Scroll to **Script Properties**
3. Click **Add script property**
   - Property: `GEMINI_API_KEY`
   - Value: your Gemini API key (get from https://aistudio.google.com/apikey)
4. Click **Save script properties**

#### Configure Discord Webhook (Optional)
5. Click **Add script property**
   - Property: `DISCORD_PROCUREMENT_WEBHOOK_URL`
   - Value: your Discord webhook URL

#### Deploy as Web App
1. Click **Deploy → New deployment**
2. Click gear icon ⚙️ → **Web app**
3. Configure:
   ```
   Execute as: Me
   Who has access: Anyone
   ```
4. Click **Deploy → Authorize access**
5. **Copy the Web app URL** (ends with `/exec`)

#### Test the Deployment
Open the Web app URL in browser. You should see:
```
OK FROM FRC PURCHASING WEB APP
```

#### Setup Dropdown Workflow
In Google Sheets: **🎃 PartBot menu → ⚙️ Setup Dropdown Workflow**

---

### 3. Discord Bot Setup

#### Create Discord Application
1. Go to https://discord.com/developers/applications
2. Click **New Application** → name it "8793PartBot"

#### Create Bot User
1. Click **Bot** in left sidebar
2. Click **Reset Token** → copy the token (only shown once!)

#### Get Application ID
1. Click **General Information**
2. Copy the **Application ID**

#### Invite Bot to Server
1. Click **OAuth2 → URL Generator**
2. Select scopes: `bot` and `applications.commands`
3. Select permissions: Send Messages, Use Slash Commands, Embed Links
4. Open the generated URL and invite to your server

#### Get Guild ID
1. Open Discord → User Settings → Advanced → Enable **Developer Mode**
2. Right-click your server icon → **Copy Server ID**

---

### 4. VM Setup (Google Cloud)

> **⚠️ Recovery Note:** This section documents how to rebuild the bot's VM from scratch.
> The original VM was lost and functionality was restored using these exact steps
> (August 2026). Follow this guide if the VM needs to be recreated.

#### 4a. Create Google Cloud Project

1. Go to [console.cloud.google.com](https://console.cloud.google.com)
2. Click the project dropdown → **New Project**
3. Configure:
   ```
   Project name: 8793PartBot
   Organization: HMB Robotics (or No organization for personal account)
   ```
4. Click **Create**
5. Enable billing (credit card required; e2-micro is free tier eligible)

#### 4b. Configure Network Access (CRITICAL)

> **⚠️ HMB Robotics Org Policy Note:** The HMB Robotics Google org has a policy that
> blocks external IPs on VM instances by default. The workaround is **Cloud NAT**, which
> provides outbound internet access without requiring an external IP.

**Step 1: Create a Cloud Router**
```
☰ Menu → VPC Network → Cloud Routers → Create Router

Name:    partbot-router
Network: default
Region:  us-west1
```

**Step 2: Create Cloud NAT Gateway**
```
☰ Menu → VPC Network → Cloud NAT → Create NAT Gateway

Gateway name: partbot-nat
Network:      default
Region:       us-west1
Cloud Router: partbot-router
NAT type:     Public
Source:       All primary and secondary IP ranges for all subnets
```

Wait ~60 seconds for NAT to provision before proceeding.

#### 4c. Create VM Instance

```
☰ Menu → Compute Engine → VM Instances → Create Instance

Name:    discord-bot-vm
Region:  us-west1 (Oregon)
Zone:    us-west1-a
Series:  E2
Type:    e2-micro (free tier eligible)

Boot disk:
  OS:   Ubuntu 22.04 LTS
  Type: Balanced persistent disk
  Size: 10 GB

Firewall:
  ✅ Allow HTTP traffic
  ✅ Allow HTTPS traffic
```

Click **Create** and wait ~60 seconds for the VM to start.

#### 4d. Install Dependencies

SSH into the VM (click **SSH** button next to VM in the console), then run each
command block separately and wait for completion before proceeding:

**Update system:**
```bash
sudo apt-get update && sudo apt-get upgrade -y
```

**Install Node.js 18.x:**
```bash
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt-get install -y nodejs
```

> Note: Node.js 18.x shows a deprecation warning — this is expected and harmless.
> Migrate to Node.js 20.x when convenient.

**Verify Node.js:**
```bash
node --version   # Should show v18.x.x
npm --version    # Should show 10.x.x
```

**Install Git:**
```bash
sudo apt-get install -y git
```

**Install PM2:**
```bash
sudo npm install -g pm2
```

#### 4e. Deploy PartBot

**Clone the repository:**
```bash
cd ~
git clone https://YOUR_TOKEN@github.com/pureh2oo/8793PartBot.git
```

> **GitHub Authentication Note:** GitHub no longer accepts passwords. Use a
> Personal Access Token (PAT) instead:
> - github.com → Settings → Developer settings → Personal access tokens → Tokens (classic)
> - Generate token with `repo` scope
> - Use format: `https://YOUR_TOKEN@github.com/pureh2oo/8793PartBot.git`

**Install dependencies:**
```bash
cd ~/8793PartBot
npm install
npm install dotenv
```

> **dotenv Note:** The bot requires dotenv for loading environment variables.
> If not already in package.json, install it manually as shown above.

**Create environment variables file:**
```bash
nano ~/8793PartBot/.env
```

Paste the following with your actual values:
```env
DISCORD_TOKEN=your-discord-bot-token-here
CLIENT_ID=your-application-id-here
GUILD_ID=your-guild-id-here
APPS_SCRIPT_URL=https://script.google.com/macros/s/YOUR_DEPLOYMENT_ID/exec
```

Save with **Ctrl+X → Y → Enter**

**Verify .env was created:**
```bash
cat ~/8793PartBot/.env
```

**Test bot (one-time run):**
```bash
cd ~/8793PartBot
node bot.js
```

Expected output:
```
[Bot] Registering slash commands...
[Bot] ✅ Slash commands registered
[Bot] ✅ Logged in as 8793PartBot#9090
```

Press **Ctrl+C** to stop.

#### 4f. Configure PM2 for 24/7 Operation

```bash
cd ~/8793PartBot

# Start with PM2
pm2 start bot.js --name discord-bot

# Configure auto-start on reboot
pm2 startup
```

Copy and run the command PM2 outputs (starts with `sudo env PATH=...`), then:

```bash
pm2 save
```

**Verify:**
```bash
pm2 status
pm2 logs discord-bot --lines 20
```

You should see `discord-bot` with status **online** 🟢

#### 4g. Verify Discord Commands

In your Discord server, type `/` and confirm these appear:
- `/requestpart` (with `sku` field between `link` and `qty`)
- `/cancelrequest`
- `/inventory`
- `/openorders`
- `/orderstatus`

**Verify SKU registration:**
```bash
cd ~/8793PartBot
node check-commands.js
```

Expected:
```
✅ SKU IS REGISTERED
```

---

## Useful Commands

### Managing the Bot (PM2)
```bash
pm2 status                      # View status
pm2 logs discord-bot            # View real-time logs
pm2 restart discord-bot         # Restart bot
pm2 stop discord-bot            # Stop bot
pm2 monit                       # Dashboard view
```

### Updating from GitHub
```bash
cd ~/8793PartBot
pm2 stop discord-bot
git pull origin main
npm install                     # If dependencies changed
pm2 restart discord-bot
pm2 logs discord-bot
```

### Updating Apps Script
1. Edit Code.gs in Apps Script editor
2. Click Save (💾)
3. **Deploy → Manage deployments → Edit → New version → Deploy**
4. URL stays the same — no bot restart needed

---

## Command Reference

### `/requestpart` - Submit New Request
```
/requestpart 
  subsystem:Drive           # Required
  link:https://...          # Optional: Vendor URL (triggers AI enrichment)
  sku:WCP-0785              # Optional: Exact SKU (overrides AI detection) ⭐
  qty:2                     # Optional: Quantity (default: 1)
  maxbudget:50              # Optional: Maximum budget in USD
  priority:High             # Optional: Critical, High, Medium, Low
  notes:"For prototype"     # Optional: Additional context
```

**Examples:**
```bash
# Exact SKU for multi-variant page
/requestpart subsystem:Drive link:https://wcproducts.com/products/ball-bearings sku:WCP-0785 qty:4

# Let AI detect SKU (single-product pages)
/requestpart subsystem:Intake link:https://www.revrobotics.com/rev-21-1650/ qty:2

# No link (manual entry by mentor)
/requestpart subsystem:Electrical notes:"5mm LED, red" qty:10
```

### `/cancelrequest` - Cancel Your Request
```
/cancelrequest 
  requestid:REQ-12345678    # Required
  reason:No longer needed   # Optional
```

### `/inventory` - Search Inventory
```
/inventory sku:WCP-0783              # Exact SKU lookup
/inventory search:bearing            # Keyword search
/inventory search:BIN-001            # Location lookup
```

### `/openorders` - View Pending Orders
```
/openorders
```

### `/orderstatus` - Check Status
```
/orderstatus requestid:REQ-1fd8b811
/orderstatus orderid:ORD-a2b3c4d5
```

---

## Troubleshooting

### VM Has No Internet Access
**Symptom:** `curl` hangs or times out during Node.js install

**Cause:** HMB Robotics org policy blocks external IPs on VMs

**Fix:** Set up Cloud NAT (see VM Setup → 4b above)

```bash
# Verify connectivity after Cloud NAT setup
ping -c 3 google.com
```

### Bot Won't Start — Missing Environment Variables
```bash
# Verify .env exists and has all 4 variables
cat ~/8793PartBot/.env

# Verify dotenv is installed
ls ~/8793PartBot/node_modules/dotenv

# If missing, install it
cd ~/8793PartBot && npm install dotenv
```

### SKU Field Missing from `/requestpart`
```bash
cd ~/8793PartBot

# Verify SKU is in bot.js (should show 2 lines)
grep -n "setName('sku')" bot.js

# Re-register commands
pm2 restart discord-bot
sleep 3
node check-commands.js
```

### Bot Shows Offline in Discord
```bash
pm2 status
pm2 logs discord-bot --lines 50
pm2 restart discord-bot
```

### "Error from Sheets" Message
1. Check Apps Script **Executions** log
2. Verify deployment is set to "Anyone" can access
3. Test Apps Script URL in browser

### GitHub Clone Fails — Authentication Error
- Password authentication is not supported
- Use a Personal Access Token (PAT) with `repo` scope
- Format: `https://YOUR_TOKEN@github.com/pureh2oo/8793PartBot.git`

### Dropdown Validation Error in Google Sheets
Run **🎃 PartBot → ⚙️ Setup Dropdown Workflow** to reset all dropdowns and validation rules.

---

## Best Practices

### When to Use the SKU Field
- ✅ **Always use** for multi-variant pages (WCP ball bearings, McMaster hardware)
- ✅ **Always use** when you know the exact part number
- ⚠️ **Optional** for single-product vendor pages (REV, AndyMark)
- ❌ **Skip** for generic requests without a vendor link

---

## Roadmap

### Recently Added ✅
- [x] User SKU override field ✅
- [x] Student self-service cancellation ✅
- [x] AI enrichment with Gemini API ✅
- [x] Location-based inventory search ✅
- [x] Formula-based "In Inventory" indicator (Column T) ✅
- [x] Cloud NAT for HMB Robotics org network compliance ✅

### Planned Features
- [ ] OnShape BOM integration (batch part requests from CAD)
- [ ] Check request / reimbursement form generation
- [ ] Vendor API integrations (REV, AndyMark direct ordering)
- [ ] Inventory QR code scanning
- [ ] Budget dashboards and spending analytics
- [ ] Web dashboard for mentors
- [ ] Automated reorder points for consumables

### Known Issues
- Node.js 18.x deprecation warning on install (harmless; migrate to 20.x when convenient)
- Gemini occasionally returns incomplete data for complex product pages
  - **Workaround:** Use `sku` parameter for multi-variant pages
- Rate limiting on Gemini API (15 requests/minute on free tier)

---

## Contributing

1. Fork this repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

---

## Maintainers

**FRC Team 8793 – Pumpkin Bots**  
Contact: pumpkinbots@hmbrobotics.org

---

## 📜 License

**8793PartBot – Automated Parts Management System**  
Copyright (c) 2025 FRC Team 8793 – Pumpkin Bots

Licensed under the **MIT License with a Use Notification Requirement**.

If you use, copy, modify, or distribute this software, please notify FRC Team 8793:
- Open an Issue on this GitHub repository
- Email: pumpkinbots@hmbrobotics.org

---

## Acknowledgments

- **FIRST Robotics Competition** for inspiring innovative solutions
- **Anthropic** for Claude AI assistance in development
- **Google** for Gemini 2.5 Flash API and Apps Script infrastructure
- **Discord** for the platform and excellent API
- **FRC vendor community** (REV, AndyMark, WCP, VEX) for supporting robotics education
- **FRC Team 8793 students** (especially Tyler H. for the SKU field feature request!)

---

**Built with ❤️ by FRC Team 8793 – Pumpkin Bots**  
*Automating the boring parts so we can focus on building robots!* 🤖🎃
