# Enterprise VPN Appliance - Testing Guide & Concepts

This guide is designed for testing the appliance on a fresh Linux machine using the **Docker Image** method. It also simplifies key concepts so you understand exactly what is happening during setup and policy creation.

---

## 🧠 Part 1: Concepts Made Simple (Read First!)

Before deploying, it helps to understand the terminology using a **"Secure Building"** analogy.

### 1. The Endpoint (The Front Door)
*   **What is it?** This is the **Public Address** of your VPN server. You enter this during the "Initial Setup".
*   **Analogy:** The **street address** of the building (e.g., `200 Main St` or `vpn.mycompany.com`).
*   **Why it matters:** If you don't give your users the right address, they can't find the building to knock on the door.

### 2. The User Connection (Entering the Building)
*   **What is it?** When a user scans the QR code or uses a config file.
*   **Analogy:** The user walking up to the specific address (Endpoint) and showing their **ID Badge** (Private Key).
*   **Result:** If the ID matches, the server opens the door and lets them into the "Lobby" (The VPN Tunnel). They are assigned an internal IP (like `10.0.0.5`) which is their badge number inside the building.

### 3. IPs in Policies (The Office Rooms)
*   **What are they?** The internal IP addresses of the resources you want to access or protect (e.g., a Database at `192.168.1.55`).
*   **Analogy:** Specific **locked rooms** inside the building.
*   **The Policy:** The security guard. If you write a policy "Allow access to `192.168.1.55`", you are telling the guard: *"If a user tries to enter Room 55, let them in."*

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

## ⚙️ Part 4: Initial Setup

1.  Open your browser on your PC.
2.  Go to: `http://<YOUR_LINUX_IP>:8000` (Use the IP you found in Part 2).
3.  **Setup Wizard Steps**:
    *   **Admin Account**: Set a username/password.
    *   **Endpoint Address**: The wizard usually auto-detects this. It should be the same **Public IP** or **DNS** that external users will use to reach this server.
        *   *If testing locally on LAN, use the IP from `hostname -I`.*
    *   **Finish**: The dashboard will load.

---

## 🛡️ Part 5: Testing Policies

Now you act as the "Security Guard".

### Scenario A: The "Blacklist" (Block a Bad Server)
**Concept**: "Let everyone go anywhere, EXCEPT this one dangerous room."

1.  Go to **Policies** > **Create Policy**.
2.  **Name**: `Block Bad Server`
3.  **Applies To**: `All Users`
4.  **Destination**: `1.2.3.4/32` (This is the specific "Bad Room").
5.  **Action**: `Deny`
6.  **Click Create**.
    *   *Test it*: Connect to VPN, try `ping 1.2.3.4`. It should fail.

### Scenario B: The "Whitelist" (Allow Specific Access)
**Concept**: "Only Engineering staff can enter the Server Room."

1.  **Prerequisite**: Create a Group called "Engineering" and add a User to it.
2.  Go to **Policies** > **Create Policy**.
3.  **Name**: `Access Server Room`
4.  **Applies To**: `Engineering` (The Group).
5.  **Destination**: `10.5.0.0/24` (The whole hallway of servers).
6.  **Action**: `Allow`
7.  **Click Create**.

### Scenario C: Validating the Connection
1.  Create a User.
2.  Download their WireGuard config or scan QR code.
3.  Connect.
4.  Try to reach the Internet (e.g., `google.com`).
    *   *Success?* Your **Endpoint** and **NAT** (IP Forwarding) are working.
5.  Try to reach a blocked IP.
    *   *Failure?* Your **Policies** are working.
