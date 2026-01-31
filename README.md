# Enterprise VPN Appliance - Testing Guide & Concepts

This guide is designed for testing the appliance on a fresh Linux machine using the **Docker Image** method. It simplifies key concepts and explains **every input field** so you know exactly what to select.

---

## 🧠 Part 1: Concepts Made Simple (Read First!)

Before deploying, it helps to understand the terminology using a **"Secure Building"** analogy.

### 1. The Endpoint (The Front Door)
*   **What is it?** This is the **Public Address** of your VPN server. You enter this during the "Initial Setup".
*   **Analogy:** The **street address** of the building (e.g., `203.0.113.50` or `vpn.mycompany.com`).
*   **Why it matters:** If you don't give your users the right address, they can't find the building to knock on the door.

### 2. The User Connection (Entering the Building)
*   **What is it?** When a user scans the QR code or uses a config file.
*   **Analogy:** The user walking up to the specific address (Endpoint) and showing their **ID Badge** (Private Key).
*   **Result:** If the ID matches, the server opens the door and lets them into the "Lobby" (The VPN Tunnel). They are assigned an internal IP (like `10.0.0.5`) which is their badge number inside the building.

### 3. IPs in Policies (The Office Rooms)
*   **What are they?** The internal IP addresses of the resources you want to access or protect (e.g., a Database at `192.168.1.55`).
*   **Analogy:** Specific **locked rooms** inside the building.
*   **The Policy:** The security guard. If you write a policy "Allow access to `192.168.1.55`", you are telling the guard: *"If a user tries to enter Room 55, let them in."*

### 4. Admin vs. User (Different Roles)
*   **The Admin (YOU):**
    *   **Runs Docker** on the Linux Server.
    *   Creates Users and Policies in the Web Dashboard.
    *   Sends the Config File (QR Code) to the user.
*   **The User (Employee):**
    *   **Does NOT run Docker.**
    *   Installs the **WireGuard Client App** on their laptop/phone.
    *   Scans the QR code provided by the Admin.
    *   Clicks "Connect".

---

## 📋 Part 2: Prepare the Linux Machine

### 1. Update the System
Open your terminal and run:
```bash
sudo apt-get update && sudo apt-get upgrade -y
```

### 2. Get Your Server IP
You need to know your Linux machine's IP address to access the dashboard later. Run:
```bash
hostname -I
```
*   **Write this down.** (It will look something like `192.168.1.50` or `10.0.0.14`).
*   You will access the server at `http://<YOUR_IP>:8000`.

### 3. Install Docker
```bash
curl -fsSL https://get.docker.com | sh
```

### 4. Enable IP Forwarding (Crucial!)
The VPN needs to route traffic between "rooms". Access will fail without this.
```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

---

## 🚀 Part 3: Deploy via Docker Image

We will use the pre-built image file (`enterprise-vpn-docker.tar.gz`). **Do not use source code.**

### 1. Transfer the File
Copy the `enterprise-vpn-docker.tar.gz` file to your Linux machine (e.g., your Desktop or Home folder).

### 2. Load the Image
Tell Docker to read the file:
```bash
sudo docker load -i enterprise-vpn-docker.tar.gz
```
*Wait for it to say: `Loaded image: networkershome/enterprise-vpn:latest`*

### 3. Run the Container
Copy and run this entire block to start the server:

```bash
sudo docker run -d \
  --name vpn-appliance \
  --restart unless-stopped \
  --cap-add NET_ADMIN \
  --network host \
  -v vpn-data:/app/data \
  -v vpn-wireguard:/etc/wireguard \
  networkershome/enterprise-vpn:latest
```

---

## ⚙️ Part 4: Initial Setup Wizard - Detailed Explanation

1.  Open your browser on your PC.
2.  Go to: `http://<YOUR_LINUX_IP>:8000` (The IP you found in Part 2).
3.  **Step 1: Initialize System**
    *   Click **Initialize System**. This generates the secret encryption keys for the server.
4.  **Step 2: Network Configuration** (Crucial Step)
    *   **Endpoint (Public IP or Domain)**:
        *   *What is it?* The public address users will connect to.
        *   *What to enter?* 
            *   If testing on **LAN/Home**: Enter the IP from Part 2 (e.g., `192.168.1.50:51820`).
            *   If using a **Cloud VPS** (AWS/DigitalOcean): Enter the Public IP (e.g., `203.0.113.5:51820`).
    *   **VPN Subnet**:
        *   *Default*: `10.0.0.0/24`
        *   *Meaning*: The "virtual" IP addresses assigned to VPN users.
        *   *Advice*: Leave as default unless it conflicts with your home network.
    *   **Client DNS**:
        *   *Default*: Empty.
        *   *Meaning*: Which DNS server VPN users will use (e.g., `8.8.8.8`).
        *   *Advice*: Leave empty to let the appliance handle it automatically.
5.  **Step 3: Admin Account**
    *   Create your login credentials for the dashboard.

---

## 🛡️ Part 5: Creating Policies - Detailed Explanation

Go to **Policies** > **Create Policy** on the dashboard. Here is what every field implies:

### 1. Name
*   **What is it?** A label for you to remember.
*   **Example**: `Allow SSH to Database`

### 2. Applies To (Dropdown)
*   **Options**:
    *   **All Users**: This rule applies to everyone (default).
    *   **Specific Group** (e.g., "Engineering"): This rule ONLY applies to users in that group.
*   **Why use it?** To segregate access. e.g., "Only Finance Group can access the Payroll Server".

### 3. Action (Dropdown)
*   **Options**:
    *   **Allow**: Traffic matches? Let it through.
    *   **Deny**: Traffic matches? Block it immediately.
*   **Logic**: Policies are checked in order (Priority). If a packet matches a "Deny" rule, it stops there.

### 4. Destination (CIDR)
*   **What is it?** The IP address of the **Server/Resource** inside your network.
*   **Format**: CIDR notation.
    *   **Specific IP**: `192.168.1.55/32` (Only this one single computer).
    *   **Whole Network**: `192.168.1.0/24` (Any computer from 192.168.1.1 to 192.168.1.254).
    *   **Everywhere**: `0.0.0.0/0` (The entire internet).

### 5. Protocol (Dropdown)
*   **Options**:
    *   **All**: Any type of traffic (Web, Pings, SSH, Games). **(Most Common)**
    *   **TCP**: Reliable connections (Websites, SSH, File Transfers).
    *   **UDP**: Fast, connectionless (Video streaming, DNS, VOIP).
    *   **ICMP**: Diagnostics (Ping).
*   **Why use it?** If you only want to allow `Ping` (ICMP) but not `SSH` (TCP) to a server.

### 6. Port (Optional)
*   **What is it?** The specific "door" number on the destination server.
*   **Examples**: `80` (Web), `443` (Secure Web), `22` (SSH), `3306` (MySQL).
*   **Advice**: Leave empty to match ALL ports. Fill it in to be super strict (e.g., "Only allow Port 80").

### 7. Priority
*   **What is it?** A number (e.g., 10, 100).
*   **How it works**: Lower numbers are checked **FIRST**.
*   **Strategy**: Put your "Deny" rules with low numbers (e.g., 10) so they block bad stuff immediately. Put generic "Allow" rules with higher numbers (e.g., 100).

---

## ✅ Example: The "Whitelist" Strategy

**Goal**: "Block everything by default, but allow Engineering to access the Server."

1.  **Rule 1 (The Block)**
    *   **Applies To**: All Users
    *   **Destination**: `0.0.0.0/0` (Everything)
    *   **Action**: **Deny**
    *   **Priority**: `999` (Last resort)
    *   *Result*: If no other rule matches, you get blocked.

2.  **Rule 2 (The Access)**
    *   **Applies To**: Engineering Group
    *   **Destination**: `192.168.1.50/32` (The Server)
    *   **Action**: **Allow**
    *   **Priority**: `10` (Check first!)
    *   *Result*: Engineering users match this rule first -> **Allowed**. Everyone else fails this rule, falls through to Rule 1 -> **Blocked**.

---
**Summary Checklist:**
- [ ] Use `hostname -I` to find your IP.
- [ ] In Setup, "Endpoint" is simply `YOUR_IP:51820`.
- [ ] In Policies, "Destination /32" = Single IP, "/24" = Whole Network.

---

## 📱 Part 6: How "Real Users" Connect

Since you are testing, **you** will act as the User too.

1.  **Create a User**: In the Dashboard, go to **Users > Add User**.
2.  **Download Config**: Click the user/device to see a **QR Code** or **Download Config** button.
3.  **Install Client**:
    *   On Windows/Mac: Install the official **WireGuard** app.
    *   On Phone: Install **WireGuard** from App Store/Play Store.
4.  **Import**: Scan the QR code or drag the config file into the WireGuard app.
5.  **Connect**: Click "Activate". You are now inside the "Building".

---

## 🏢 Part 7: Handling Large Teams (Self-Registration)

If you have **100,000 employees**, you do NOT create accounts manually!

1.  **Send the URL**: Give your users the link to the dashboard (e.g., `http://vpn.mycompany.com`).
2.  **User Sign Up**: They click **"Sign Up"** and create their own account.
3.  **Admin Approval**:
    *   You (Admin) go to **Users > Pending Approvals**.
    *   You see a list of all 1,000 requests.
    *   Click **"Select All"** -> **"Approve"**.
4.  **Done**: The system automatically generates their keys and lets them download the config.

---

## 🔄 Part 8: How to Reset / Start Over

If you messed up or want to test from scratch (to see the "Initialize System" screen again), you must **delete the container AND the data**.

Run these commands on your Linux machine:

```bash
# 1. Stop and remove the current container
sudo docker rm -f vpn-appliance

# 2. Delete the saved data (Database and WireGuard keys)
sudo docker volume rm vpn-data vpn-wireguard

# 3. Now verify they are gone (Should list nothing or not show usage)
sudo docker volume ls
```

After running these, run the **Start Command** from Part 3 again. You will see the "Initialize System" screen like it's a brand new install.

---

## 🛠️ Appendix: How to Build the New Image (Since you changed code)

Since we fixed the "Failed to connect" bug, you need to use the **New Code**, not the old `tar.gz` file.

### Option A: Build on Linux (Recommended)
1.  **Transfer the Code**: Copy the entire `enterprise_vpn` folder to your Linux machine.
2.  **Build**: Run this command inside the folder:
    ```bash
    sudo docker build -t networkershome/enterprise-vpn:latest .
    ```
    *(Note: This might take 5-10 minutes as it builds the frontend).*
3.  **Run**: Now proceed with **Part 3, Step 3** (Run the Container).

### Option B: Build on Windows (If you have Docker Desktop)
1.  Open PowerShell in the folder.
2.  Run: `docker build -t networkershome/enterprise-vpn:latest .`
3.  Save it: `docker save -o enterprise-vpn-docker.tar.gz networkershome/enterprise-vpn:latest`
4.  Transfer the `tar.gz` to Linux and use it as before.
