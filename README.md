# VSCode Tunnel Setup Guide for NOAA HPC Systems

This guide walks through setting up a Visual Studio Code tunnel on a NOAA HPC system so you can develop remotely using VSCode on your local machine or browser.

---

## Important: Accessing the HPC via Parallel Works

This guide assumes you are accessing your HPC system through **[Parallel Works](https://noaa.parallel.works/)**. You do **not** connect directly to the HPC via a terminal client such as PuTTY. Instead:

1. Log into the Parallel Works platform
2. Open a terminal session on your HPC system through Parallel Works
3. Follow the steps below from within that terminal session

---

## Prerequisites

- A NOAA HPC account with access to your target system (`{MACHINE}`) via Parallel Works
- [Visual Studio Code](https://code.visualstudio.com/) installed on your local machine
- A GitHub account (used to authenticate the tunnel)

---

## Overview

A VSCode tunnel allows you to run a VSCode server on a remote HPC system and connect to it from your local VSCode or a browser. The tunnel is authenticated through GitHub.

---

## Step 1: Set Up `~/bin`

Log into your HPC system via Parallel Works and check if `~/bin` exists:

```bash
ls ~/bin
```

If it does not exist, create it:

```bash
mkdir ~/bin
```

Make sure `~/bin` is in your `PATH` by checking your `.bashrc`:

```bash
grep PATH ~/.bashrc
```

If it is not present, add the following to your `.bashrc`:

```bash
if ! [[ "$PATH" =~ "$HOME/.local/bin:$HOME/bin:" ]]
then
    PATH="$HOME/.local/bin:$HOME/bin:$PATH"
fi
export PATH
```

Then source your `.bashrc`:

```bash
source ~/.bashrc
```

---

## Step 2: Download VSCode CLI

Navigate to `~/bin` and download the VSCode CLI binary:

```bash
cd ~/bin
wget -O VSCode-linux-x64.tar.gz "https://update.code.visualstudio.com/latest/linux-x64/stable"
```

Extract it:

```bash
tar -xzf VSCode-linux-x64.tar.gz
```

You should now see a `VSCode-linux-x64` directory in `~/bin`.

**Remove the tar.gz archive** to save disk space:

```bash
rm ~/bin/VSCode-linux-x64.tar.gz
```

---

## Step 3: Create Symlinks

Create symlinks in `~/bin` pointing to the `code` and `code-tunnel` binaries:

```bash
cd ~/bin
ln -s VSCode-linux-x64/bin/code code
ln -s VSCode-linux-x64/bin/code-tunnel code-tunnel
```

Verify the symlinks:

```bash
ls -la ~/bin | grep code
```

You should see something like:

```
lrwxrwxrwx  code -> VSCode-linux-x64/bin/code
lrwxrwxrwx  code-tunnel -> VSCode-linux-x64/bin/code-tunnel
```

---

## Step 4: Create the Tunnel Script

Create a shell script to launch the tunnel in the background:

```bash
nano ~/bin/{MACHINE}code.sh
```

Paste the following contents, replacing `{MACHINE}code` with whatever name you want to give your tunnel (e.g., `mycode`):

```bash
#!/bin/bash
server_name=${1:-"{MACHINE}code"}
rm -f "${HOME}/${server_name}.out"
nohup code tunnel --name "${server_name}" --accept-server-license-terms > "${HOME}/${server_name}.out" 2>&1 &
exit
```

Make the script executable:

```bash
chmod +x ~/bin/{MACHINE}code.sh
```

> **Tip:** The tunnel name (e.g., `{MACHINE}code`) is how it will appear in your local VSCode. Choose something descriptive but not sensitive.

---

## Step 5: Start the Tunnel

Run your tunnel script:

```bash
{MACHINE}code.sh
```

Monitor the output:

```bash
tail -f ~/{MACHINE}code.out
```

You should see output similar to:

```
* Visual Studio Code Server
*
* By using the software, you agree to
* the Visual Studio Code Server License Terms (https://aka.ms/vscode-server-license) and
* the Microsoft Privacy Statement (https://privacy.microsoft.com/en-US/privacystatement).
*
[TIMESTAMP] info Using GitHub for authentication
To grant access to the server, please log into https://github.com/login/device and use code XXXX-XXXX

  Visual Studio Code Tunnel v1.x.x

  ➜  Tunnel:   {MACHINE}code
  ➜  Open:  https://vscode.dev/tunnel/{MACHINE}code/...
```

---

## Step 6: Authenticate with GitHub

1. Open **https://github.com/login/device** in your browser
2. Enter the code shown in your tunnel output (e.g., `XXXX-XXXX`)
3. Approve access with your GitHub account

> **Note:** You may see a warning like `Failed to update keyring with new credentials`. This is harmless and can be ignored.

---

## Step 7: Connect VSCode to the Tunnel

### Option A: Local VSCode (recommended)
1. Open VSCode on your local machine
2. Click the **blue button** in the bottom-left corner
3. Select **"Connect to Tunnel..."**
4. Select your tunnel name (e.g., `{MACHINE}code`) from the list

### Option B: Browser
Navigate directly to the URL shown in your tunnel output:
```
https://vscode.dev/tunnel/{MACHINE}code/...
```

---

## Managing Disk Quota

The VSCode binary (`VSCode-linux-x64`) is approximately 1 GB. If your home directory has a tight quota, consider moving it to your scratch space and updating the symlinks:

```bash
mv ~/bin/VSCode-linux-x64 $SCRATCH/
cd ~/bin
rm code code-tunnel
ln -s $SCRATCH/VSCode-linux-x64/bin/code code
ln -s $SCRATCH/VSCode-linux-x64/bin/code-tunnel code-tunnel
```

To check your current home directory quota:

```bash
quota -vs
```

---

## Stopping the Tunnel

To stop a running tunnel:

```bash
pkill -u $USER -f tunnel
```

To stop a tunnel running on a different login node from your current session:

```bash
ssh {OTHER_NODE} "pkill -u $USER -f tunnel"
```

> **Note:** If the remote node is unresponsive, this command may fail. In that case, start a fresh tunnel on a healthy node instead.

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|--------|--------------|-----|
| `exec request failed on channel 0` | Remote node is too degraded to run commands | Start tunnel on a different node |
| Tunnel keeps reconnecting (`warn Tunnel exited unexpectedly`) | High node load or network instability | Check node load with `uptime`; try a different node |
| `WebSocket protocol error: Connection reset` | Network interruption between HPC and GitHub | Wait and allow auto-reconnect, or restart tunnel |
| `code: command not found` | Symlink missing or `~/bin` not in PATH | Re-check symlinks and `.bashrc` PATH setup |
| Home directory quota warning email | Large files in `$HOME` | Move `VSCode-linux-x64` to scratch; remove `.tar.gz` archive |

---

## Notes

- Tunnel scripts run in the background via `nohup`, so you can close your Parallel Works terminal session and the tunnel will persist.
- Output logs are saved to `~/{MACHINE}code.out` for debugging.
- Each HPC system requires its own tunnel setup and tunnel name.
- Avoid running tunnels on login nodes for extended periods if system policy discourages it; check your HPC's user guidelines.

---

# 🌟 Acknowledgements 🌟

A huge, heartfelt **thank you** to **Brian Curtis** for making all of this possible.

Brian's expertise, patience, and generosity in sharing his knowledge were instrumental in getting VSCode tunnels working across NOAA HPC systems. He is not only exceptionally skilled, but also one of the kindest and most helpful people you will ever have the pleasure of working with. This guide would not exist without him.

**Thank you, Brian!** 🙏
