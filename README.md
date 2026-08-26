# Local SIEM Lab: Wazuh Deployment & Automated Threat Mitigation

This repository documents the step-by-step installation, configuration, and validation of a centralized security monitoring and automated incident response lab. The setup utilizes **Wazuh (SIEM/XDR)** deployed via **Docker** on a main host machine to monitor and defend a separate **Debian-based Virtual Machine (VM)** endpoint against live network attacks.

---

## 🛠 Lab Architecture & Components

*   **SIEM Server (Main Host)**: Runs the Wazuh stack (Indexer, Manager, Dashboard) containerized via Docker Compose.
*   **Monitored Endpoint (Target VM)**: A virtual machine running a Debian-based Linux distribution equipped with the native Wazuh Agent.
*   **Attack Simulation Engine**: A separate terminal workspace executing network brute-force vectors to test system defenses.

---

## 🚀 Step 1: Deploying the Wazuh Single-Node Container Stack

Wazuh components are deployed on the main host machine via Docker. Because the indexer engine relies heavily on virtual memory system mappings, the default host kernel parameters must be adjusted prior to initialization to prevent deployment crashes.

### 1. Adjust Host Virtual Memory Allocations
Run the following commands on the main host machine to scale up the allocation limits:
```bash
sudo sysctl -w vm.max_map_count=262144
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
```
<img width="1006" height="90" alt="Screenshot From 2026-08-26 13-53-46" src="https://github.com/user-attachments/assets/56cbdcd5-e3d8-488a-aae3-6178be07715a" />

*   **Why**: The underlying search architecture uses memory-mapped files. Insufficient map counts cause Out-Of-Memory boot loops on standard system architectures.

### 2. Pull and Deploy the Containers
Install [Docker CLI](https://docs.docker.com/engine/install/ubuntu/) from the official guide. 

Download the deployment templates, generate cluster-specific encryption certificates, and orchestrate the runtime environment:
```bash
# Clone the verified stable release configuration branch
cd ~/Desktop
git clone https://github.com/wazuh/wazuh-docker.git
cd wazuh-docker/single-node

# Start the docker service
sudo systemctl start docker

# Run the localized standalone utility to generate unique system certificates
sudo docker compose -f generate-indexer-certs.yml run --rm generator

# Pull image payloads and initialize the entire single-node SIEM infrastructure
sudo docker compose up -d
```
*   **Why**: Generating independent system certificates secures communication protocols between components. Running `docker compose up -d` detaches the running container stack to run continuously in the background.

---

## 📥 Step 2: Provisioning the Debian-Based Agent VM

With the main node functional, the target endpoint must be linked to feed monitoring telemetry back into the analytics engine.

### 1. Extract the Agent Setup Script
1. Navigate to the web user interface via your browser at `https://localhost` (on your host computer's local IP address).
2. Authenticate using the administrative system credentials.
   
         Username: admin
         Password: SecretPassword

   
4. Access **Agents** > **Deploy new agent** through the navigation matrix.
5. Select **DEB** matching your Debian-based distribution.
6. Provide your host machine's interface network IP to map the target destination.

### 2. Install the Package on the Target VM
Execute the target network installation payload directly in the terminal workspace of your Debian-based VM:
```bash
# Download package dependencies and assign the target management address variable
# Generated in the Wazuh-dashboard

# Force a reload of the system service management engine configurations
sudo systemctl daemon-reload

# Configure the agent binary daemon to initiate automatically on system power up
sudo systemctl enable wazuh-agent

# Turn on the running agent instance to start reporting data
sudo systemctl start wazuh-agent
```
*   **Why**: The configuration variable injections embed the structural destination properties into the agent framework, setting up the secure transport channel to route raw host system events.

---

## 🔍 Step 3: Configuring File Integrity Monitoring (FIM)

File Integrity Monitoring uses cryptographic checksum tracking to capture system alterations or unapproved directory changes on the target endpoint.

On the **Debian-based VM**, open the main client configuration template:
```bash
sudo nano /var/ossec/etc/ossec.conf
```

Locate the `<syscheck>` XML configuration block and append your specific asset paths, forcing real-time engine processing hooks:
```xml
<syscheck>
  <directories realtime="yes"><"_YOUR_ABSOLUTE_FOLDER_PATH_"></directories>
</syscheck>
```
Example for the absolute path "/home/ben/test-files/"

Apply your adjustments cleanly by recycling the agent background process loop:
```bash
sudo systemctl restart wazuh-agent
```
---

## 🛡 Step 4: Activating Automated Threat Mitigations (Active Response)

When repeated authentication drops happen, the system matches events against signature Rule ID `5763` (`sshd: brute force trying to get access`). We can wire up the manager to instruct the endpoint agent to execute an automated iptables drop rule instantly.

On the **Main Host Machine**, edit the structural application parameters map inside your storage volumes:
```bash
sudo nano wazuh-docker/single-node/config/wazuh_manager/wazuh_app.conf
```
*(Alternatively, you can edit this directly in the browser dashboard via **Management** > **Configuration** > **Edit Configuration**).*

Insert the defining execution and active rule parameters inside the core root XML configuration structures:

```xml
<!-- Define the response script command target -->
<command>
  <name>firewall-drop</name>
  <executable>firewall-drop</executable>
  <timeout_allowed>yes</timeout_allowed>
</command>

<!-- Trigger the firewall-drop on the local agent whenever SSH brute force triggers -->
<active-response>
  <command>firewall-drop</command>
  <location>local</location>
  <rules_id>5763</rules_id>
  <timeout>60</timeout>
</active-response>
```
*   **Why**: Setting the location parameter structure to `local` makes the defense script execute directly on the compromised endpoint VM. The `timeout` element sets a strict 1-minute temporary ban block, allowing normal traffic to safely resume after the timer clears.

Restart the management container service node on your host to bring your new configuration rules online:
```bash
cd ~/Desktop
cd wazuh-docker/single-node
sudo docker compose restart wazuh.manager
```

---

## ⚡ Step 5: Verification and Attack Validation Testing

To prove the security system functions under real attack conditions, an aggressive SSH dictionary-based attack is directed straight at the Debian-based target system interface.
### 1. Install the SSH server and Hydra by running this command:
```bash
sudo apt install openssh-server hydra
```
### 2. Build an Attack Dictionary
Construct a 15-entry mock credential file named `passwords.txt` within your testing environment terminal:
```bash
cat << EOF > passwords.txt
password123
admin
root
qwerty
123456
login
system
security
password
server
backup
user1
testpass
invalidlogin
wrongsecret
EOF
```

### 2. Launch the Attack
Execute the network authentication exploit target routing parameters via `hydra`:
```bash
hydra -l root -P passwords.txt ssh://<DEBIAN_VM_IP_ADDRESS>
```

### 3. Review Security Lab Validation Metrics
During active script runtime execution, the attacking framework terminal logs connection timeouts and abruptly drops off the stream. Inspecting the management console records the explicit incident alert sequence chain:

| Timestamp | Component Name | Rule Description | Severity Level | Rule ID |
| :--- | :--- | :--- | :---: | :---: |
| `13:06:56` | `Wazuh-Debian-VM` | `sshd: authentication failed.` | 5 | **5760** |
| `13:06:58` | `Wazuh-Debian-VM` | `sshd: brute force trying to get access...` | 10 | **5763** |
| `13:06:59` | `Wazuh-Debian-VM` | `Host Blocked by firewall-drop Active Response` | 3 | **651** |

