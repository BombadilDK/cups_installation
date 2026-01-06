# Guide: Setting up CUPS Print Server on Ubuntu

**Date:** December 31, 2025

**System:** Ubuntu Server + Windows 11 & Ubuntu Desktop Clients + HP DeskJet 2630

**Purpose**

To allow the central Ubuntu server to manage a network printer (HP DeskJet 2600 Series), so that Windows and Linux clients print through the server (queue management) instead of directly to the printer. This ensures uniform settings (e.g., forced black and white) and logging.

## Part 1: Server Setup (The Brain)

Application: CUPS (Common Unix Printing System).

### 1. Software Installation

Update and install CUPS along with HP’s driver package (HPLIP).

`sudo apt update`

`sudo apt install cups hplip`

### 2. Find Server IP

Find the server's IP address, which is needed for client configuration and the web interface.

It is assumed that the IP is static.

`hostname -I`

### 3. Access and Permissions (Network Sharing)

Share printers on the local network, make it possible to log on to the webinterface from a remote computer <br>
and make it possible to delete print jobs (the last is convinient): 

`sudo cupsctl --share-printers --remote-admin --user-cancel-any`

Make user (you) a print administrator (may require logging out/in):

`sudo usermod -aG lpadmin $USER`

### 4. Firewall (UFW)

Allow traffic on port 631 (the IPP/CUPS port).

`sudo ufw allow 631/tcp`

### 5. Configuration via Web Interface (Securely)

The CUPS web interface is now accessible only from the server itself or via a secure tunnel. The setup is completed using an SSH tunnel from your client PC:

1. Run the tunnel from your client PC (Windows/Linux):

`ssh -L 6310:localhost:631 <user name>@<server-ip>`

2. Access in a browser (while the tunnel is open):

`http://localhost:6310`

(Accept the security warning).
3. Select **Administration** -> **Add Printer**.
4. Choose the HP printer under **"Discovered Network Printers"** (preferably the one with hp:/net/... in the name).
5. **IMPORTANT: Naming Tip:** When prompted, choose a simple printer name (e.g., *HP_2600*) without spaces or special characters to simplify future terminal commands.
6. **IMPORTANT:** Check the box for **"Share This Printer"**.
7. **Driver:** Select **HP DeskJet 2600 Series, hpcups (recommended)**.
8. **Set Default Options:**

- Media Size: A4
- Output Mode: Grayscale (or Black Only) to save color ink by default.

Part 2: Connecting a Windows 11 Client

Windows typically only discovers the printer directly. We must manually force it through the server.

1. Go to **Settings** -> **Bluetooth & devices** -> **Printers & scanners**.
2. Click **Add device** -> Wait for it to scan -> Click **Add manually**.
3. Select the option: **"Select a shared printer by name"**.
4. Enter the server's address (HTTP):
`http://<server-ip>:631/printers/HP_DeskJet_2600_series`
*(Replace IP and name to match the server's queue name).*
5. Select driver: **HP DeskJet 2600 series** (or **Microsoft IPP Class Driver**).
6. Optionally name the printer **"HP Server Print"** to distinguish it.

Part 3: Connecting an Ubuntu Client

The graphical menu might fail with authentication. The terminal is used to create an IPP connection.

1. Open the terminal on the client PC.
2. Run the following command to add the printer (driverless):

`sudo lpadmin -p "HP_Server_Print" -E -v ipp://192.168.8.50:631/printers/HP_DeskJet_2600_series -m everywhere`

1. The printer is now visible in the system settings.

Part 4: CUPS SysAdmin: Command Overview

Use these commands via SSH directly on the server for daily operation.

1. Status and Overview

See if the printer is available and what's in the queue.

- **Check everything (Printer status + Queue):**
`lpstat -t`
- **Show only the queue (Active jobs):**
`lpstat -o`

2. Managing the Queue (Cleanup)

When a print job is stuck or needs to be deleted.

- **Delete a specific job:**
*(Find the ID with `lpstat -o` first, e.g., `HP_DeskJet-15`)*
`cancel <JOB-ID>`
**Example:**
`cancel HP_DeskJet-15`

- **Delete EVERYTHING in the queue (Emergency Stop):**
`cancel -a`

3. Managing the Printer (On/Off)

If the printer is "Paused" (e.g., after a paper jam), it must be manually restarted.

- **Resume / Turn on the printer:**
`cupsenable <PRINTER-NAME>`
**Example:**
`cupsenable HP_DeskJet_2600_series`

- **Pause / Temporarily stop the printer:**
`cupsdisable <PRINTER-NAME>`

4. System Troubleshooting

If the CUPS service itself is acting up.

- **Restart the entire print system:**
`sudo systemctl restart cups`
- **Check if CUPS is running:**
`sudo systemctl status cups`

5. Finding the Printer Name

If you've forgotten the exact name of the printer in the system (necessary for start/stop commands).

- **Show all installed printers (and their connection):**
`lpstat -v`
- **Show printers and their current status:**
`lpstat -p`
*(The name is the first word after "printer" or "device for", e.g., `HP_DeskJet_2600_series`)*

Part 5: Glossary and Commands

Here is an explanation of the concepts used for future reference.

**Key Commands**

- **sudo**: (SuperUser DO). Run command as administrator/root.
- **apt install**: Installs packages/programs from Ubuntu's repository.
- **usermod -aG [group] [user]**: Adds a user to an existing group (here, 'lpadmin').
- **ufw allow [port]**: (Uncomplicated Firewall). Opens a hole in the firewall for traffic.
- **cupsctl**: Tool to change CUPS settings (e.g., remote access) without editing text files.

**Parameters (Client-side)**

**lpadmin**

- **-p**: Specifies the name the printer should have on the PC.
- **-E**: (Enable). Activates the printer and accepts jobs immediately.
- **-v**: (Device URI). The address where the printer is found (here ipp://...).
- **-m everywhere**: Tells the system to use the "IPP Everywhere" (driverless) standard, so it fetches PPD/info directly from the server.

**Protocols**

- **CUPS**: The print server software (used by Linux and macOS).
- **IPP**: (Internet Printing Protocol). Modern network standard for printing.
- **HPLIP**: HP's Linux driver package, which provides advanced control over HP printers.
