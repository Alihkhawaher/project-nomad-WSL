# Project Nomad - WSL Installation Guide

This guide will walk you through installing Project Nomad on Windows Subsystem for Linux (WSL).

## Prerequisites

- Windows 10 or Windows 11
- Administrator access on your Windows machine

---

## Step 1: Install WSL on Windows

Open PowerShell as Administrator and run:

```powershell
wsl --install
```

This will install WSL with Ubuntu as the default distribution.

---

## Step 2: Enable Docker in WSL

After WSL is installed, open your WSL terminal and configure it:

```bash
cat > /etc/wsl.conf <<EOF
[boot]
systemd=true
EOF
```

### Why Enable systemd?

Systemd is the init system used by most Linux distributions to manage services and the system boot process. Enabling systemd in WSL provides several benefits:

- **Service Management**: Allows you to use `systemctl` commands to manage services like Docker, SSH, and other daemons
- **Snap Support**: Many snap packages (including Docker) require systemd to function properly
- **Compatibility**: Brings WSL behavior closer to traditional Linux systems, improving compatibility with Linux-native software
- **Background Services**: Enables services to start automatically and run in the background properly

> **Note**: Without systemd enabled, some services may fail to start or behave unexpectedly in WSL.

Then enable and start the snapd service:

```bash
sudo systemctl unmask snapd.service
sudo systemctl enable snapd.service
sudo systemctl start snapd.service
```

Update your system packages:

```bash
apt update && apt upgrade -y
```

Install Docker:

```bash
snap install docker
```

---

## Step 3: Restart WSL

Exit WSL and restart it from Windows:

```bash
exit
```

Then from PowerShell or Command Prompt:

```powershell
wsl --list
wsl -d Ubuntu
```

> **Note:** Replace `Ubuntu` with your actual WSL distribution name if different.

---

## Step 4: Re-enter WSL

Open a new WSL terminal:

```bash
wsl
```

---

## Step 5: Install the Project

Run the following command in your WSL terminal:

```bash
sudo apt-get update && sudo apt-get install -y curl && curl -fsSL https://raw.githubusercontent.com/Alihkhawaher/project-nomad-WSL/refs/heads/main/install_nomad.sh -o install_nomad.sh && sudo bash install_nomad.sh
```

This will:
1. Update package lists
2. Install curl
3. Download the installation script
4. Execute the installation

---

## Troubleshooting

### If WSL command not found
Make sure you're running Windows 10 version 2004 or later, or Windows 11.

### If Docker doesn't start
Ensure Docker is properly installed and the service is running:
```bash
sudo systemctl start snapd.service
sudo snap start docker
```

### If installation script fails
Check that you have internet connectivity in WSL and that GitHub is accessible.

---

## Support

For issues or questions, please refer to the project repository or open an issue.
