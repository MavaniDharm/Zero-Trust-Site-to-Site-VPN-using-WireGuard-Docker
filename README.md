🔐 Zero Trust Site-to-Site VPN using WireGuard & Docker

<p align="center">
<img src="https://img.shields.io/badge/Platform-Kali%20Linux-blue?logo=kalilinux" />
<img src="https://img.shields.io/badge/VPN-WireGuard-green?logo=wireguard" />
<img src="https://img.shields.io/badge/Containers-Docker-blue?logo=docker" />
<img src="https://img.shields.io/badge/Status-Lab%20Project-success" />
</p>

📌 Project Overview

This project demonstrates how to build a Site-to-Site VPN using WireGuard and Docker.

Two separate private networks (Site A and Site B) are connected securely through WireGuard VPN tunnels.

The goal is to allow devices in both networks to communicate securely without exposing their private services directly to the internet.

🎯 Project Objectives
🔐 Establish a secure WireGuard VPN tunnel.
🐳 Deploy VPN gateways using Docker containers.
🌐 Connect two isolated Docker networks.
🔄 Enable communication between Site A and Site B.
🛡️ Configure IP forwarding and firewall rules.
📡 Test private network connectivity.
🏗️ Network Architecture
text
                 SITE-TO-SITE VPN
                       
       SITE A                    SITE B
   172.28.1.0/24             172.28.2.0/24
         |                         |
         |                         |
  VPN Gateway A              VPN Gateway B
  172.28.1.254               172.28.2.254
  10.13.13.1                 10.14.14.1
         |                         |
         +------ WireGuard --------+
                 VPN Tunnel
              172.18.0.0/16
                        
         |                         |
   Docker Client A           Docker Client B
   172.28.1.10                172.28.2.10


Note: The diagram represents the intended architecture. Client A is optional and Client B is used in the current lab configuration.

🛠️ Technologies Used
Technology	Purpose
Kali Linux	Host operating system
Docker	Containerization
Docker Compose	Container orchestration
WireGuard	VPN tunnel
Linux iptables	Firewall and NAT
Alpine Linux	Test client
📋 IP Address Configuration
Component	IP Address
Site A Network	172.28.1.0/24
Site B Network	172.28.2.0/24
VPN Transport Network	172.18.0.0/16
Gateway A	172.28.1.254
Gateway B	172.28.2.254
Gateway A VPN IP	10.13.13.1
Gateway B VPN IP	10.14.14.1
Site B Client	172.28.2.10
📦 Step 1: Install Required Packages

Update Kali Linux:

bash
sudo apt update
sudo apt upgrade -y


Install Docker and Docker Compose:

bash
sudo apt install docker.io docker-compose-plugin wireguard-tools iptables -y


Enable Docker:

bash
sudo systemctl enable docker
sudo systemctl start docker


Check Docker:

bash
sudo docker --version
sudo docker compose version

📁 Step 2: Create Project Directory
bash
mkdir -p ~/zero-trust-vpn
cd ~/zero-trust-vpn


Create configuration directories:

bash
mkdir -p wg_config_a/wg_confs
mkdir -p wg_config_b/wg_confs


Project structure:

text
zero-trust-vpn/
│
├── docker-compose.yml
│
├── wg_config_a/
│   └── wg_confs/
│       └── wg0.conf
│
└── wg_config_b/
    └── wg_confs/
        └── wg0.conf

⚙️ Step 3: Configure Docker Compose

Create the Docker Compose file:

bash
nano docker-compose.yml


Paste the following configuration:

yaml
services:

  vpn_gateway_a:
    image: lscr.io/linuxserver/wireguard:latest
    container_name: vpn-gateway-a

    cap_add:
      - NET_ADMIN

    sysctls:
      net.ipv4.ip_forward: "1"

    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Asia/Kolkata
      - SERVERURL=auto
      - SERVERPORT=51820
      - PEERS=1
      - PEERDNS=auto
      - INTERNAL_SUBNET=10.13.13.0

    volumes:
      - ./wg_config_a:/config

    networks:
      site_a_net:
        ipv4_address: 172.28.1.254
      vpn_transport_net:
        ipv4_address: 172.18.0.2

    restart: unless-stopped

  vpn_gateway_b:
    image: lscr.io/linuxserver/wireguard:latest
    container_name: vpn-gateway-b

    cap_add:
      - NET_ADMIN

    sysctls:
      net.ipv4.ip_forward: "1"

    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Asia/Kolkata
      - SERVERURL=auto
      - SERVERPORT=51820
      - PEERS=1
      - PEERDNS=auto
      - INTERNAL_SUBNET=10.14.14.0

    volumes:
      - ./wg_config_b:/config

    networks:
      site_b_net:
        ipv4_address: 172.28.2.254
      vpn_transport_net:
        ipv4_address: 172.18.0.3

    restart: unless-stopped

  site_b_client:
    image: alpine
    container_name: site-b-client
    command: sleep infinity

    networks:
      site_b_net:
        ipv4_address: 172.28.2.10

networks:

  site_a_net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.28.1.0/24

  site_b_net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.28.2.0/24

  vpn_transport_net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.18.0.0/16


Save the file.

🔑 Step 4: Generate WireGuard Configurations

Start the containers initially so LinuxServer WireGuard can generate its configuration files:

bash
sudo docker compose up -d


Generated configurations:

text
wg_config_a/wg_confs/wg0.conf
wg_config_b/wg_confs/wg0.conf


The two gateways initially generate their own VPN keys.

Important: Do not overwrite the generated private keys or client peer configurations.

🔐 Step 5: Configure Site-to-Site Peers

Obtain Gateway A's public key:

bash
sudo docker exec vpn-gateway-a wg show


Obtain Gateway B's public key:

bash
sudo docker exec vpn-gateway-b wg show


Edit Gateway A:

bash
sudo nano wg_config_a/wg_confs/wg0.conf


Add the following peer entry, replacing the public key with Gateway B's actual public key:

ini
[Peer]
PublicKey = GATEWAY_B_PUBLIC_KEY
AllowedIPs = 10.14.14.1/32, 172.28.2.0/24
Endpoint = 172.18.0.3:51820
PersistentKeepalive = 25


Edit Gateway B:

bash
sudo nano wg_config_b/wg_confs/wg0.conf


Add Gateway A:

ini
[Peer]
PublicKey = GATEWAY_A_PUBLIC_KEY
AllowedIPs = 10.13.13.1/32, 172.28.1.0/24
Endpoint = 172.18.0.2:51820
PersistentKeepalive = 25


⚠️ Replace both placeholders with actual public keys. Never share private keys or preshared keys publicly.

🔄 Step 6: Restart VPN Gateways

Apply the configuration:

bash
sudo docker compose restart vpn_gateway_a vpn_gateway_b

🌐 Step 7: Configure IP Forwarding

The Docker Compose configuration enables IPv4 forwarding inside both gateway containers.

The WireGuard configurations also contain firewall rules:

ini
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -A FORWARD -o %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth+ -j MASQUERADE

PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -D FORWARD -o %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth+ -j MASQUERADE


These rules allow forwarding through the VPN interface and provide NAT for outgoing traffic.

📡 Step 8: Test VPN Connectivity

From Gateway A, test Gateway B:

bash
sudo docker exec vpn-gateway-a ping -c 4 10.14.14.1


Test Site B client:

bash
sudo docker exec vpn-gateway-a ping -c 4 172.28.2.10


From Gateway B, test Gateway A:

bash
sudo docker exec vpn-gateway-b ping -c 4 10.13.13.1


Test Site A client, if configured:

bash
sudo docker exec vpn-gateway-b ping -c 4 172.28.1.10

📊 Step 9: Monitor WireGuard Tunnel

Display WireGuard status:

bash
sudo docker exec vpn-gateway-a wg show

bash
sudo docker exec vpn-gateway-b wg show


Check tunnel traffic counters:

bash
sudo docker exec vpn-gateway-a wg show


A successful handshake and increasing transfer counters indicate that encrypted traffic is passing through the tunnel.

📸 Project Screenshots

Add your screenshots here after completing each step.

Screenshot	Description
01	Docker containers running
02	Docker network configuration
03	Gateway A WireGuard status
04	Gateway B WireGuard status
05	Gateway A to Gateway B ping
06	Site A to Site B client ping
07	UDP traffic captured using tcpdump

Example:

markdown
![Docker Containers](screenshots/docker-containers.png)

![WireGuard Status](screenshots/wireguard-status.png)

![VPN Connectivity](screenshots/vpn-connectivity.png)

🔒 Security Best Practices
Never upload private keys to GitHub.
Never upload preshared keys.
Do not expose VPN configuration secrets.
Use firewall rules to restrict unnecessary traffic.
Use dedicated VPN subnets.
Regularly update Docker images.
Restrict access to management interfaces.
🎓 Learning Outcomes

Through this project, I learned:

WireGuard VPN configuration.
Docker networking and bridge networks.
Site-to-Site VPN architecture.
Public and private key authentication.
Linux IP forwarding.
Firewall and NAT configuration.
Network troubleshooting using ping, wg and tcpdump.
🚀 Future Improvements
Add a Site A test client.
Implement firewall-based access restrictions.
Add centralized VPN monitoring.
Implement logging and traffic monitoring.
Integrate additional remote sites.
👨‍💻 Author

Cybersecurity / Network Security Lab Project

Built using Kali Linux, Docker and WireGuard.

⭐ If you find this project useful, consider giving this repository a star!

#CyberSecurity #WireGuard #Docker #VPN #ZeroTrust #NetworkSecurity #KaliLinux

Cybersecurity / Network Security Lab Project

Built using Kali Linux, Docker and WireGuard.

⭐ If you find this project useful, consider giving this repository a star!

`#CyberSecurity` `#WireGuard` `#Docker` `#VPN` `#ZeroTrust` `#NetworkSecurity` `#KaliLinux`
