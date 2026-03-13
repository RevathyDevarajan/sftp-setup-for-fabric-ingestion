# Microsoft Fabric — SFTP to Lakehouse Pipeline

![Microsoft Fabric](https://img.shields.io/badge/Microsoft_Fabric-Trial-0078D4?style=flat&logo=microsoft)
![Status](https://img.shields.io/badge/Status-Complete-2E7D32?style=flat)
![Platform](https://img.shields.io/badge/Platform-Windows_11-0078D4?style=flat&logo=windows)
![Architecture](https://img.shields.io/badge/Architecture-Medallion-B45309?style=flat)
![SFTP](https://img.shields.io/badge/SFTP-Bitvise_SSH_Server-2B547E?style=flat)

A portfolio project demonstrating an **end-to-end data ingestion pipeline** built on Microsoft Fabric. The pipeline extracts a daily CSV sales file from a local SFTP server and lands it into a **Bronze Lakehouse** — following the Medallion Architecture pattern (Bronze → Silver → Gold).

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  DATA SOURCE  (Local Windows 11 PC)                             │
│                                                                 │
│  Bitvise SSH Server  (port 22)                                  │
│  D:\SFTPShare\DailySales\Upload\sales_20260301.csv              │
└──────────────────────────┬──────────────────────────────────────┘
                           │  SSH / SFTP (port 22)
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  CONNECTIVITY LAYER                                             │
│  On-premises Data Gateway  (Gateway-LocalPC)                    │
│  Bridges local Windows PC  ←→  Microsoft Fabric cloud          │
└──────────────────────────┬──────────────────────────────────────┘
                           │  Secure encrypted tunnel
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  MICROSOFT FABRIC  (cloud)                                      │
│  Workspace: Daily_Sales_Analytics                               │
│                                                                 │
│  Data Pipeline → Copy Data Activity                             │
│                           │                                     │
│                           ▼                                     │
│  Pulse_Sales_Bronze Lakehouse / Files  ✓                        │
└─────────────────────────────────────────────────────────────────┘
```

---

## Medallion Architecture

| Layer | Lakehouse | Description |
|-------|-----------|-------------|
| **Bronze** ← *this project* | `Pulse_Sales_Bronze` | Raw data landed from source — no transformations |
| **Silver** | `Silver data` | Cleaned, validated, correctly typed data |
| **Gold** | `Golden data` | Aggregated, business-ready tables and reports |

---

## Tech Stack

| Component | Technology |
|-----------|------------|
| Cloud Platform | Microsoft Fabric (Trial) |
| Pipeline | Fabric Data Factory — Copy Data activity |
| SFTP Server | Bitvise SSH Server (free, Windows-native, port 22) |
| Connectivity | On-premises Data Gateway (Gateway-LocalPC) |
| Destination | Microsoft Fabric Lakehouse (Pulse_Sales_Bronze) |
| OS | Windows 11 |
| Source Format | CSV (daily sales file) |

---

## Prerequisites

- Microsoft Fabric workspace (free trial at [app.fabric.microsoft.com](https://app.fabric.microsoft.com))
- Windows 10 or 11 PC
- [Bitvise SSH Server](https://bitvise.com/ssh-server-download) — free personal licence
- [On-premises Data Gateway](https://www.microsoft.com/en-us/download/details.aspx?id=53127) — free

> All components used in this project are **free**. No paid licences are required.

---

## Setup Guide

### Step 1 — Set Up the SFTP Server (Bitvise)

1. Download and install Bitvise SSH Server from `bitvise.com/ssh-server-download`
2. Open the control panel → **Open easy settings** → **Accounts** tab → **Add**
3. Configure the virtual account:

   | Field | Value |
   |-------|-------|
   | Virtual account name | `fabricuser` |
   | Password | Set a secure password |
   | Login allowed | Ticked |
   | Allow file transfer | Ticked |
   | Virtual filesystem layout | Limit to root directory |
   | Root directory | `D:\SFTPShare\DailySales\Upload` |

4. Click **OK** → then click **Save changes** in the main panel (both steps required)

> ⚠️ **Important:** The root directory must be on a **local drive** (D: or C:) — not an OneDrive-synced folder. OneDrive files are cloud placeholders that SFTP servers cannot read.

Create the folder and add your data file:
```
D:\
  SFTPShare\
    DailySales\
      Upload\
        sales_20260301.csv    ← your data file
```

Verify the server is running:
```powershell
Test-NetConnection -ComputerName 192.168.1.18 -Port 22
# TcpTestSucceeded : True
```

---

### Step 2 — Install the On-premises Data Gateway

1. Download and install the On-premises Data Gateway from Microsoft
2. Sign in using the **same Microsoft account** as your Fabric workspace
3. Register a new gateway named `Gateway-LocalPC`
4. Set a recovery key and save it securely
5. Confirm the gateway shows **Connected** status

---

### Step 3 — Find Your Local IP Address

The gateway service account cannot reach `localhost`. Use your actual network IP:

```powershell
ipconfig
# Note the IPv4 Address under your active adapter (e.g. 192.168.1.18)
```

---

### Step 4 — Create the SFTP Connection in Fabric

1. Open Fabric → your pipeline → **Copy data** activity → **Source** tab
2. Connection dropdown → **More** → search **SFTP**
3. Fill in:

   | Field | Value |
   |-------|-------|
   | Server | `192.168.1.18` (your local IPv4) |
   | Port | `22` |
   | Connection name | `SFTP_Bitvise` |
   | Data gateway | `[On-premises] Gateway-LocalPC` |
   | Authentication | Basic |
   | Username | `fabricuser` |
   | Password | *(your Bitvise password)* |

4. Click **Connect** → you should see **Connection successful** ✅

---

### Step 5 — Configure Source, Destination & Run

**Source:**
- File path type: `File path`
- Directory: `/`
- File name: `sales_20260301.csv`
- Format: `DelimitedText`, first row as header

**Destination:**
- Select `Pulse_Sales_Bronze` Lakehouse
- Root folder: `Files`
- Format: `DelimitedText`, first row as header

**Run:** Click **Run** → monitor in the **Output** tab → verify file appears in Lakehouse Files ✅

---

## Sample Data Schema

| Column | Type | Description |
|--------|------|-------------|
| order_id | Integer | Unique order identifier |
| customer_name | String | Full name of the customer |
| product | String | Product name |
| quantity | Integer | Number of units ordered |
| unit_price | Decimal | Price per individual unit |
| total_price | Decimal | quantity × unit_price |
| order_date | Date | YYYY-MM-DD format |
| status | String | Completed / Shipped / Pending |

File naming convention: `sales_YYYYMMDD.csv`

---

## Key Learnings & Real-World Challenges

This project surfaced several infrastructure challenges rarely covered in tutorials:

### 1. Docker WSL2 Networking on Windows
Docker Desktop uses WSL2 virtualisation. The gateway service account (`NT SERVICE\PBIEgwService`) cannot reliably reach Docker container ports through WSL2 port forwarding — even though `Test-NetConnection` succeeds from regular user sessions.

**Fix:** Use Bitvise SSH Server — a native Windows service that binds directly to the Windows network adapter.

### 2. OneDrive Cloud-Only File Placeholders
Files in OneDrive-synced folders appear locally available but are cloud placeholders (`-a---l` flag in PowerShell). SFTP servers cannot serve them.

**Fix:** Move source data to a local drive path outside OneDrive.

### 3. Gateway Service Account Network Restrictions
The gateway runs as `NT SERVICE\PBIEgwService` with restricted access. Changing the service account fixes networking but breaks cloud connectivity (`GatewayUnreachable`).

**Fix:** Use a native Windows server (Bitvise) that the gateway service account can reach without any workarounds.

### 4. localhost vs IP Address
The gateway service account cannot reach `localhost` or `127.0.0.1` the same way a regular user session can.

**Fix:** Always use the machine's actual IPv4 address in Fabric connection settings.

---

## Troubleshooting

| Error | Resolution |
|-------|------------|
| `SocketErrorCode: ConnectionRefused` | Use your IPv4 address (not localhost). Confirm Bitvise is running on port 22. Confirm gateway is Connected. |
| No results found in file browser | Move files off OneDrive to a local drive. Check Bitvise Root directory is correct. Restart Bitvise after saving config. |
| Gateway shows Offline | Revert gateway service Log On account to `NT SERVICE\PBIEgwService` in services.msc. |
| `GatewayUnreachable` | Gateway service account was changed. Revert and restart the service. |
| Gateway not in Fabric dropdown | Ensure gateway was registered with the same Microsoft account as Fabric. |
| Bitvise config not saving | Both steps required: click **OK** on editor AND **Save changes** in main panel. |

---

## Next Steps

- [ ] Load Bronze CSV into a **Delta Table** using a Fabric Notebook
- [ ] Build a **Silver layer** — clean, validate, and type the data
- [ ] Build a **Gold layer** — aggregated sales metrics by product, date, region
- [ ] Connect a **Power BI** semantic model to the Gold tables
- [ ] Schedule the pipeline to run **daily** with a Fabric trigger
- [ ] Add **email alerts** on pipeline failure

---

## Repository Structure

```
fabric-sftp-pipeline/
├── README.md                              ← this file
├── docs/
│   └── SFTP_Pipeline_Documentation.docx  ← full Word document guide
└── sample-data/
    └── sales_20260301.csv                 ← sample source data
```

---

*Built with Microsoft Fabric | Bitvise SSH Server | On-premises Data Gateway | Windows 11*
