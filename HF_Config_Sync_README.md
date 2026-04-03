# HF Config Sync v1.0

## Splunk App — Sync Heavy Forwarder Configurations to Deployment Server

---

## What This App Does

HF Config Sync solves a common Splunk operational problem: when you configure add-ons locally on a Heavy Forwarder (HF), those configurations live only on that HF. If the Deployment Server (DS) pushes the app again or you add new HFs to the server class, your local configs are either overwritten or missing from the new HFs.

This app runs on your **primary HF** and syncs local configuration changes back to the DS's `deployment-apps/` directory via SCP. Once synced, the DS automatically pushes those configs to all HFs in the server class.

### Workflow

```
You configure add-on on Primary HF
        │
        ▼
HF Config Sync detects the change
        │
        ▼
App SCPs configs to DS deployment-apps/<app>/local/
        │
        ▼
DS reloads and pushes configs to ALL HFs in server class
```

---

## Prerequisites

Before installing, ensure the following:

1. **Splunk Enterprise 9.0+** on both HF and DS (tested on Splunk 10.0).
2. **SSH key-based authentication** from the primary HF to the DS (the `splunk` OS user on the HF must be able to SSH into the DS without a password).
3. The HF must be connected to the DS as a deployment client.
4. The apps you want to sync must already exist in `deployment-apps/` on the DS (i.e., they were pushed by the DS to the HF originally).

---

## Installation

### Step 1: Set Up SSH Keys (HF → DS)

On the **primary HF**, run:

```bash
ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa
ssh-copy-id splunk@<DS_IP_ADDRESS>
```

Enter the DS `splunk` user's password when prompted. Then verify passwordless access:

```bash
ssh splunk@<DS_IP_ADDRESS> "echo connected"
```

This should print `connected` without asking for a password.

### Step 2: Install the App on the Primary HF

Copy the `hf_config_sync` app to the HF's apps directory:

```bash
tar -xzf hf_config_sync-2.1.0.tar.gz -C /opt/splunk/etc/apps/
```

### Step 3: Configure the App

Create the local configuration file:

```bash
mkdir -p /opt/splunk/etc/apps/hf_config_sync/local
```

Edit `/opt/splunk/etc/apps/hf_config_sync/local/hf_config_sync.conf`:

```ini
[general]
# Deployment Server IP or hostname
ds_host = 192.168.88.96
# DS management port
ds_port = 8089
# Splunk admin credentials on the DS (used for DS reload)
ds_username = admin
ds_password = YourDSPassword
ds_use_ssl = true
ds_verify_ssl = false
# SSH user on the DS (used for SCP file transfer)
ds_ssh_user = splunk
# Splunk install path on the DS
ds_splunk_home = /opt/splunk

[sync]
# Leave empty — apps are selected from the dashboard or CLI
monitored_apps =
# Path to apps directory on this HF
hf_apps_path = /opt/splunk/etc/apps
# Automatically reload DS after pushing configs
auto_reload_ds = true

[logging]
log_level = INFO
```

### Step 4: Add the Required commands.conf Override

Create `/opt/splunk/etc/apps/hf_config_sync/local/commands.conf`:

```ini
[hfconfigstatus]
filename = cmd_status.py
type = python
enableheader = false
outputheader = false
python.version = python3

[hfconfigapps]
filename = cmd_apps.py
type = python
enableheader = false
outputheader = false
python.version = python3

[hfconfigsync]
filename = cmd_sync.py
type = python
enableheader = false
outputheader = false
python.version = python3

[hfconfigtoggle]
filename = cmd_toggle.py
type = python
enableheader = false
outputheader = false
python.version = python3
```

### Step 5: Restart Splunk

```bash
/opt/splunk/bin/splunk restart
```

---

## Usage

All operations are performed via custom search commands in the Splunk search bar, or viewed from the **HF Config Sync** dashboard.

### View All Apps on the HF

```
| hfconfigstatus
```

Returns a table showing every app on the HF with columns: app name, whether it has local configs, whether it's monitored, last sync time, and whether it has pending changes.

### Filter for a Specific App

```
| hfconfigstatus | search app_name="*paloalto*"
```

### Add an App to Monitoring

```
| hfconfigtoggle app=Splunk_TA_paloalto action=add
```

Replace `Splunk_TA_paloalto` with the exact folder name of the app as it appears in `/opt/splunk/etc/apps/`.

### Remove an App from Monitoring

```
| hfconfigtoggle app=Splunk_TA_paloalto action=remove
```

### Sync Changed Apps Only

```
| hfconfigsync mode=changed
```

This detects which monitored apps have new local config changes since the last sync and pushes only those.

### Force Sync All Monitored Apps

```
| hfconfigsync mode=force
```

Pushes all monitored apps' local configs to the DS regardless of whether changes were detected.

### View Only Monitored Apps

```
| hfconfigstatus | where is_monitored="True"
```

### View Apps with Pending Changes

```
| hfconfigstatus | where has_pending_changes="True"
```

---

## Dashboard

Open the **HF Config Sync** app in Splunk Web to see the dashboard with:

- **HF Connection** — confirms the HF is connected.
- **Monitored Apps** — count of apps being monitored.
- **Pending Changes** — count of apps with unsynchronized changes.
- **Last Sync** — timestamp of the most recent sync operation.
- **All Apps on this HF** — full table of all apps with their status.
- **Currently Monitored Apps** — list of apps currently selected for monitoring.

The **Sync History** tab shows a log of all past sync operations.

---

## Enable Automatic Scheduled Sync

To sync automatically every 5 minutes without manual intervention, create `/opt/splunk/etc/apps/hf_config_sync/local/inputs.conf`:

```ini
[script://./bin/scheduled_sync.py]
disabled = false
interval = 300
python.version = python3
```

Then restart Splunk:

```bash
/opt/splunk/bin/splunk restart
```

The app will now check for config changes every 5 minutes and push them to the DS automatically.

---

## Example: Full Walkthrough

This example demonstrates the complete workflow from start to finish.

**Scenario:** The DS pushes `Splunk_TA_nix` to your primary HF. You configure it locally. Later, you add a second HF and want it to receive the same config.

**1. Configure the add-on on the primary HF:**

```bash
mkdir -p /opt/splunk/etc/apps/Splunk_TA_nix/local
cat > /opt/splunk/etc/apps/Splunk_TA_nix/local/inputs.conf << 'EOF'
[script://./bin/cpu.sh]
disabled = false
interval = 60
sourcetype = cpu
index = os

[script://./bin/df.sh]
disabled = false
interval = 300
sourcetype = df
index = os
EOF
```

**2. Add the app to monitoring (Splunk search bar):**

```
| hfconfigtoggle app=Splunk_TA_nix action=add
```

**3. Sync to the DS:**

```
| hfconfigsync mode=force
```

**4. Verify on the DS:**

```bash
ssh splunk@192.168.88.96 "cat /opt/splunk/etc/deployment-apps/Splunk_TA_nix/local/inputs.conf"
```

You should see your `inputs.conf` content on the DS.

**5. Add a second HF to the server class on the DS.** The DS will automatically push `Splunk_TA_nix` with your configs to the new HF.

---

## How It Works (Technical Details)

The app uses four custom search commands backed by a Python sync engine:

- **hfconfigstatus** — scans `/opt/splunk/etc/apps/` for all apps, checks their `local/` directories for `.conf` files, and compares hashes against the last-synced state.
- **hfconfigtoggle** — writes the selected app list to `local/selected_apps.json`.
- **hfconfigsync** — for each monitored app with changes, SCPs the `.conf` files from the HF's `<app>/local/` to the DS's `deployment-apps/<app>/local/`, then triggers a DS reload via REST API.
- **scheduled_sync.py** — a scripted input that runs `hfconfigsync mode=changed` on a timer.

Config changes are detected by hashing the contents of each app's `local/` directory and comparing against the stored hash from the last successful sync.

---

## Troubleshooting

**"No apps selected for monitoring"**
Run `| hfconfigtoggle app=YourAppName action=add` to add an app.

**"Push to DS failed"**
Check SSH connectivity: `ssh splunk@<DS_IP> "echo test"`. If it asks for a password, SSH keys are not set up correctly. Re-run `ssh-copy-id`.

**"HTTP Error 401: Unauthorized"**
The `ds_username` and `ds_password` in `hf_config_sync.conf` don't match the DS Splunk admin credentials. Fix them and restart Splunk.

**Configs not appearing on other HFs after sync**
Verify `auto_reload_ds = true` in the config. Or manually reload: `ssh splunk@<DS_IP> "/opt/splunk/bin/splunk reload deploy-server"`. Also confirm the other HFs are in the same server class.

**Dashboard shows errors about command type**
Ensure `local/commands.conf` exists with `type = python` for all four commands (see Step 4 above).

**App doesn't appear in `| hfconfigstatus`**
The app must have either a `default/` or `local/` directory. Internal Splunk apps (search, launcher, etc.) are excluded by default.

---

## File Structure

```
hf_config_sync/
├── bin/
│   ├── sync_engine.py        # Core sync engine
│   ├── cmd_status.py          # | hfconfigstatus command
│   ├── cmd_apps.py            # | hfconfigapps command
│   ├── cmd_sync.py            # | hfconfigsync command
│   ├── cmd_toggle.py          # | hfconfigtoggle command
│   └── scheduled_sync.py     # Scripted input for auto-sync
├── default/
│   ├── app.conf
│   ├── commands.conf
│   ├── hf_config_sync.conf
│   ├── inputs.conf
│   └── data/ui/
│       ├── nav/default.xml
│       └── views/
│           ├── hf_config_sync_dashboard.xml
│           └── sync_history.xml
├── local/                     # Your config overrides go here
│   ├── hf_config_sync.conf
│   ├── commands.conf
│   ├── selected_apps.json
│   └── sync_state.json       # Auto-generated state tracking
└── metadata/
    └── default.meta
```

---

## Configuration Reference

### hf_config_sync.conf

| Setting | Section | Description | Default |
|---------|---------|-------------|---------|
| ds_host | general | DS IP or hostname | — |
| ds_port | general | DS management port | 8089 |
| ds_username | general | Splunk admin user on DS | admin |
| ds_password | general | Splunk admin password on DS | — |
| ds_use_ssl | general | Use HTTPS for DS REST calls | true |
| ds_verify_ssl | general | Verify SSL certificate | false |
| ds_ssh_user | general | OS user for SSH/SCP to DS | splunk |
| ds_splunk_home | general | Splunk install path on DS | /opt/splunk |
| monitored_apps | sync | Comma-separated app list (or use dashboard) | — |
| hf_apps_path | sync | Apps directory on HF | /opt/splunk/etc/apps |
| auto_reload_ds | sync | Auto-reload DS after sync | true |
| log_level | logging | DEBUG, INFO, WARNING, ERROR | INFO |

### Search Commands

| Command | Description | Example |
|---------|-------------|---------|
| hfconfigstatus | List all apps and their sync status | `\| hfconfigstatus` |
| hfconfigtoggle | Add/remove app from monitoring | `\| hfconfigtoggle app=MyApp action=add` |
| hfconfigsync | Trigger sync to DS | `\| hfconfigsync mode=force` |
| hfconfigapps | List managed apps (simplified) | `\| hfconfigapps` |

---

## Security Notes

- DS Splunk credentials are stored in plain text in `hf_config_sync.conf`. Restrict file permissions: `chmod 600 /opt/splunk/etc/apps/hf_config_sync/local/hf_config_sync.conf`
- SSH keys should be restricted to the `splunk` user only.
- Only `.conf` files are synced — binary files and non-conf files are ignored.
- The app only pushes TO the DS — it never modifies or deletes files on the HF.
