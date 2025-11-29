# linux-sandbox
**Advanced Containerized Sandbox using systemd-nspawn**

This document provides a solution for creating a secure, resource-controlled execution environment by combining the requirements of a classic chroot environment, kernel isolation (`unshare`), and resource limiting (`isolate`). The single tool used is `systemd-nspawn`, which integrates all these functions seamlessly using **Linux Namespaces** and **Cgroups**.

---

## Solution Overview: Synthesis of Isolation Tools

The `systemd-nspawn` utility acts as a lightweight container runtime, enabling deep isolation and resource governance without the overhead of Docker or full virtual machines.

| Feature                     | Simple chroot | unshare (Raw Namespaces) | systemd-nspawn (This Solution) |
|-----------------------------|---------------|--------------------------|--------------------------------|
| **Filesystem Isolation**    | ✅ Basic chroot | ❌ Requires manual setup  | ✅ Enhanced chroot (Handles `/proc`, `/sys`, etc.) |
| **Kernel Isolation (Namespaces)** | ❌ No | ✅ Manual and complex | ✅ Automatic, deep isolation (PID, UTS, IPC, Mount) |
| **Resource Limits (Cgroups)** | ❌ No | ❌ No | ✅ Integrated via systemd properties (CPU, RAM, IO) |
| **Network Security**        | ❌ No          | ❌ No                    | ✅ Easy to disable (`--private-network`) |

---

## Step 1: Sandbox Setup and Dependency Check (`setup_advanced_jail.sh`)

This script installs the necessary `systemd-container` package (which provides `systemd-nspawn`) and uses `debootstrap` to create a minimal Debian root filesystem for the sandbox.

- `setup_advanced_jail.sh`
```bash
#!/bin/bash
# --- Configuration ---
JAIL_DIR="/var/chroot/worker_jail"

# --- Dependency Check and Installation ("Man" Install) ---
echo "--- 1. Checking for systemd-container (systemd-nspawn) ---"
if ! command -v systemd-nspawn &> /dev/null; then
    echo "systemd-nspawn not found. Attempting to install 'systemd-container'..."
    if command -v apt &> /dev/null; then
        sudo apt update
        sudo apt install -y systemd-container
    elif command -v yum &> /dev/null; then
        sudo yum install -y systemd-container
    elif command -v dnf &> /dev/null; then
        sudo dnf install -y systemd-container
    else
        echo "🚨 ERROR: Package manager not found. Please manually install 'systemd-container'."
        exit 1
    fi
    if command -v systemd-nspawn &> /dev/null; then
        echo "✅ systemd-nspawn installed successfully."
    else
        echo "🚨 ERROR: Failed to install systemd-nspawn."
        exit 1
    fi
else
    echo "✅ systemd-nspawn is already installed."
fi

# --- Minimal Jail Filesystem Provisioning ---
echo ""
echo "--- 2. Setting up minimal jail directory: $JAIL_DIR ---"
if [ -d "$JAIL_DIR" ]; then
    echo "Directory $JAIL_DIR already exists. Skipping debootstrap setup."
else
    echo "Creating directory $JAIL_DIR..."
    sudo mkdir -p "$JAIL_DIR"
    echo "Running debootstrap to create a minimal Debian system..."
    # Note: Using 'stable' and 'minimal' for a small, functional base.
    sudo debootstrap --arch=amd64 stable "$JAIL_DIR" http://deb.debian.org/debian/

    if [ $? -eq 0 ]; then
        echo "✅ Minimal Debian jail provisioned successfully."
    else
        echo "🚨 ERROR: debootstrap failed. Check if 'debootstrap' package is installed."
        sudo rm -rf "$JAIL_DIR"
        exit 1
    fi
fi

echo ""
echo "--- Setup Complete ---"
echo "You can now use the execution script to run commands securely."
```

---

## Step 2: Advanced Isolation and Resource Control Script (`nspawn_jail.sh`)

This is the main executor script. It takes the command as an argument and launches it inside the container with strict Cgroup limits (CPU, RAM, IO) and deep kernel isolation (namespaces, private network).

- `nspawn_jail.sh`
```bash
#!/bin/bash
# --- Configuration ---
JAIL_DIR="/var/chroot/worker_jail"

# Resource Limits (Cgroups - like the 'isolate' tool)
# These limits are applied to the temporary slice systemd-nspawn creates.
CPU_QUOTA="20%"       # Restrict CPU usage to 20% of one core
MEMORY_LIMIT="128M"   # 128 Megabyte RAM limit
IO_WEIGHT="10"        # BlockIOWeight = 10 (very low disk I/O priority, range 10-1000)

# Command to execute inside the container
COMMAND_TO_RUN="$*"

# --- Main Execution ---
if [ $# -eq 0 ]; then
    echo "Usage: $0 <command_to_run_inside_jail> [args...]"
    echo "Example: $0 'cat /etc/os-release'"
    exit 1
fi

if [ ! -d "$JAIL_DIR/bin" ]; then
    echo "🚨 ERROR: Jail directory $JAIL_DIR is not fully set up."
    echo "Please run the setup script first."
    exit 1
fi

echo "--- Running Isolated Worker ---"
echo "Command: $COMMAND_TO_RUN"
echo "Resources: CPU=$CPU_QUOTA, RAM=$MEMORY_LIMIT, IO_Weight=$IO_WEIGHT"
echo "-----------------------------------"

# systemd-nspawn Execution Breakdown:
# --private-network: Disables network access (Crucial for security).
# --capability=CAP_AUDIT: Restrict all capabilities except the minimal audit capability.
# --property=...: Applying Cgroup resource controls (the isolate part).
sudo systemd-nspawn \
    --quiet \
    --private-network \
    --no-sync \
    --link-journal=no \
    --capability=CAP_AUDIT \
    --property=CPUQuota=$CPU_QUOTA \
    --property=MemoryLimit=$MEMORY_LIMIT \
    --property=BlockIOWeight=$IO_WEIGHT \
    --directory="$JAIL_DIR" \
    /usr/bin/env bash -c "$COMMAND_TO_RUN"

EXIT_CODE=$?
echo "-----------------------------------"
echo "🚪 Isolated process exited with code $EXIT_CODE."
return $EXIT_CODE
```

---

## Usage

1. Save the content of **Step 1** into a file named `setup_advanced_jail.sh`.
2. Save the content of **Step 2** into a file named `nspawn_jail.sh`.
3. Make both files executable:
   ```bash
   chmod +x setup_advanced_jail.sh nspawn_jail.sh
   ```
4. Run the setup:
   ```bash
   ./setup_advanced_jail.sh
   ```
5. Run a command:
   ```bash
   ./nspawn_jail.sh "ps aux"
   ```
   This will show only processes inside the container.

---

## man `nspawn_jail.sh`
**man page** for the `nspawn_jail.sh` command, formatted as a traditional Unix manual page:

- `man nspawn_jail`: install_man_jail.sh
```bash
#!/bin/bash

# Define the man page content
MAN_CONTENT='.TH NSPAWN_JAIL 1 "November 2025" "nspawn_jail 1.0" "User Commands"
.SH NAME
nspawn_jail \- Execute commands in an isolated, resource-controlled container using systemd-nspawn

.SH SYNOPSIS
.B nspawn_jail.sh
[COMMAND [ARGS...]]

.SH DESCRIPTION
.B nspawn_jail
is a script that launches a command inside a secure, resource-limited container using
.B systemd-nspawn.
It provides deep kernel isolation (namespaces), filesystem isolation (chroot), and resource limits (Cgroups) for enhanced security and control.

.SH CONFIGURATION
The following variables can be edited in the script:
.TP
.B JAIL_DIR="/var/chroot/worker_jail"
Directory where the container\'s root filesystem is stored.
.TP
.B CPU_QUOTA="20%"
Limit CPU usage to 20% of one core.
.TP
.B MEMORY_LIMIT="128M"
Limit memory usage to 128 MB.
.TP
.B IO_WEIGHT="10"
Set disk I/O priority (range: 10-1000).

.SH EXAMPLES
Run a command inside the container:
.RS
.B ./nspawn_jail.sh "ps aux"
.RE

Check the OS release:
.RS
.B ./nspawn_jail.sh "cat /etc/os-release"
.RE

.SH SETUP
Before using
.B nspawn_jail,
you must run the setup script:
.RS
.B ./setup_advanced_jail.sh
.RE

.SH SECURITY
The container runs with the following security features:
.IP • 3
Private network (no network access)
.IP • 3
Restricted capabilities (only CAP_AUDIT)
.IP • 3
Resource limits (CPU, RAM, I/O)
.IP • 3
Isolated filesystem

.SH FILES
.TP
.B /var/chroot/worker_jail
Default directory for the container\'s root filesystem.
.TP
.B setup_advanced_jail.sh
Script to set up the container environment.
.TP
.B nspawn_jail.sh
Main script to execute commands in the container.

.SH SEE ALSO
.BR systemd-nspawn (1),
.BR debootstrap (8),
.BR chroot (1)

.SH AUTHOR
Written by mosi pvp'

# Install the man page
INSTALL_DIR="/usr/local/share/man/man1"
MAN_PAGE="$INSTALL_DIR/nspawn_jail.1"

echo "Installing nspawn_jail man page to $MAN_PAGE..."
sudo mkdir -p "$INSTALL_DIR"
echo "$MAN_CONTENT" | sudo tee "$MAN_PAGE" > /dev/null
sudo mandb -q

echo "Done. You can now view the man page with: man nspawn_jail"

```

### Instructions:
1. Save this script as `install_man_jail.sh`.
2. Make it executable:
   ```bash
   chmod +x install_man_jail.sh
   ```
3. Run it as root or with sudo:
   ```bash
   sudo ./install_man_jail.sh
   ```
4. After installation, you can view the man page with:
   ```bash
   man nspawn_jail
   ```
---

### Notes:
- Replace placeholders like `[Your Name or Organization]`, `[Year]`, and `[Specify License]` with actual information.
- If you want to distribute this as a real man page, save it as `nspawn_jail.1` in the appropriate man page directory (e.g., `/usr/local/man/man1/`), and run `mandb` to update the man database. Users can then view it with `man nspawn_jail`.
