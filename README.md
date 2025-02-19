# Linux Hacking Workshop Documentation

## Overview
This workshop is designed to give students hands-on experience in ethical hacking by providing a controlled environment. Each student is assigned two containers:

1. **Attacker Machine** (`containerX`) - This serves as the attacker's system.
2. **Target Machine** (`targetmachineX`) - A vulnerable machine for exploitation.

A central **Ubuntu VM** hosts a web portal where students can log in and access their assigned containers. The setup utilizes **Shellinabox** for web-based terminal access, **Flask** for user authentication, and **Nginx** as a reverse proxy.

## System Architecture
The system consists of:

- A **publicly accessible Ubuntu VM** that hosts a web application for login and container access.
- Each student is assigned two **containers on Proxmox**, running in a private network.
- **Nginx** acts as a reverse proxy, allowing students to connect to their respective containers.
- **Shellinabox** provides a web-based terminal interface.
- **Flask** manages user sessions and authentication.
- **A Linux bridge in Proxmox PVE** is created to allow the VM to connect to the private network. Once created, a new network interface is added to the VM through the hardware tab. After this, running `ifconfig` will show a new network adapter, which must be configured in Netplan with a private IP.

### **Creating a Linux Bridge in Proxmox**
1. Navigate to **Proxmox PVE > Network**.
2. Click **Create** and select **Linux Bridge**.
3. Assign the appropriate **CIDR and Gateway** (e.g., `172.16.230.1/24`).
4. Apply the configuration and restart networking.

### **Assigning the Network Interface to a VM**
1. Go to the **Hardware tab** of the target VM in Proxmox.
2. Click **Add** > **Network Device**.
3. Select the newly created bridge (e.g., `vmbr410`).
4. Configure the **static IP** and **gateway** within the VM.

### **Verifying Network Setup**
Once the network interface is added to the VM:
- Run `ifconfig` to see the newly assigned adapter.
- Configure Netplan to assign it a **private IP**.
- Apply the changes with:
  ```bash
  sudo netplan apply
  ```

## Setup Instructions

### 1. Install Required Packages on the Ubuntu VM
```bash
sudo apt update && sudo apt install -y python3 python3-pip python3-venv curl nginx
```

### 2. Set Up Python Virtual Environment and Flask
```bash
mkdir -p /var/www/container-access
cd /var/www/container-access
python3 -m venv myenv
source myenv/bin/activate
pip install flask flask-session
```

### 3. Configure Flask Application
A Flask application manages student authentication and container access.

#### **Flask Features**:
- Each student logs in with their own credentials.
- The system ensures only assigned containers are accessible.

Run the application with:
```bash
python3 app.py
```
Ensure Flask is bound to the public IP:
```python
app.run(host='PUBLIC_IP', port=5000)
```

### 4. Configure Nginx Reverse Proxy
Nginx routes traffic to the appropriate student containers.

Example `/etc/nginx/sites-available/reverse-proxy`:
```
server {
    listen 8080;
    server_name PUBLIC_IP;

    location /container1/ {
        proxy_pass http://172.16.230.2:4200/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
    location /targetmachine1/ {
        proxy_pass http://172.16.230.3:4200/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### 5. Enable the Site and Restart Nginx
These commands ensure Nginx is configured correctly and active:
```bash
sudo ln -s /etc/nginx/sites-available/reverse-proxy /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```
- `ln -s /etc/nginx/sites-available/reverse-proxy /etc/nginx/sites-enabled/` creates a symbolic link to enable the reverse proxy configuration.
- `rm /etc/nginx/sites-enabled/default` removes the default configuration to avoid conflicts.
- `nginx -t` tests the configuration for errors.
- `systemctl reload nginx` applies the changes without restarting the service.

### 6. Verify Running Services
Check if **Flask and Nginx** are running correctly:
```bash
sudo netstat -plnt
```
Expected output:
```
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address    State    PID/Program name
tcp   0      0 0.0.0.0:8080         LISTEN   nginx: master
tcp   0      0 PUBLIC_IP:5000       LISTEN   python3 app.py
```
Ensure Flask is bound to the public IP and Nginx is listening on port **8080**.

## Tools Installed on the Attacking Machine
The following tools are pre-installed on the attacking container:
- **steghide** - Hides data in images and audio files.
- **nmap** - Network scanning tool.
- **crunch** - Wordlist generator.
- **hydra** - Password-cracking tool.
- **metasploit** - Exploitation framework.
- **hping3** - Network packet crafting tool.

## Tools Installed on the Target Machine
The target machine includes:
- **snort** - Intrusion detection and prevention system.
- **OpenSSH** - SSH service enabled for remote access.

## Student Access Process
1. Students log in at `your-workshop-domain.com`
2. The dashboard lists assigned containers.
3. Clicking a container opens **Shellinabox** for web-based terminal access.

## Security Considerations
- **Limit external access:** Only the **Ubuntu VM** is exposed to the internet.
- **Restrict access to port 8080**:
  ```bash
  sudo ufw allow 8080/tcp
  sudo ufw enable
  ```
- **Harden Nginx** to prevent unauthorized access.
- **Regularly rotate passwords** for security.

## License
This project is released under the MIT License.

