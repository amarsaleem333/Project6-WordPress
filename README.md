Markdown
# Project 6: Three-Tier Web Solution with WordPress and LVM Storage Management

This repository documents the successful implementation of a scalable, resilient **Three-Tier Web Architecture** hosting a WordPress website connected to a remote MySQL database. The infrastructure is built on AWS EC2 instances running Red Hat Enterprise Linux (RHEL), utilizing Advanced Linux Storage Management via **Logical Volume Manager (LVM)**.

---

## Architecture Overview

The project implements a classic **Three-Tier Architecture**:
1. **Presentation Layer (Client):** A web browser on a laptop/PC.
2. **Business Logic Layer (Web Server):** An AWS EC2 RHEL instance running Apache HTTP Server, PHP, and WordPress.
3. **Data Management Layer (Database Server):** An AWS EC2 RHEL instance running MySQL Server.

---

## Part 1: Storage Infrastructure & LVM Configuration (Web Server)

### 1. Inspecting Attached Block Devices
Three 10 GiB Elastic Block Store (EBS) volumes were attached to the Red Hat EC2 instance. The devices were inspected using `lsblk`:

```bash
lsblk
Output:

Plaintext
NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
nvme2n1     259:0    0   10G  0 disk
nvme0n1     259:1    0   10G  0 disk
├─nvme0n1p1 259:3    0    1M  0 part
├─nvme0n1p2 259:4    0  200M  0 part /boot/efi
└─nvme0n1p3 259:5    0  9.8G  0 part /
nvme1n1     259:2    0   10G  0 disk
nvme3n1     259:6    0   10G  0 disk
Engineering Note: The newly attached 10 GB storage drives were recognized by the OS kernel as nvme1n1, nvme2n1, and nvme3n1.

2. Overcoming the RHEL Subscription Trap
Problem Solving: Since the un-registered RHEL instance lacked access to standard subscription repositories where gdisk resides, fdisk was utilized instead. fdisk comes pre-installed and natively supports creating GPT partition tables and setting modern LVM hex codes (30 for Linux LVM).

The partitioning sequence executed for each disk (nvme1n1, nvme2n1, nvme3n1) using sudo fdisk /dev/<device>:

g (Create a new empty GPT partition table)

n (Create new partition)

Defaults accepted for partition number, first sector, and last sector

t (Change partition type)

30 (Hex code for Linux LVM in modern fdisk GPT mode)

w (Write changes and exit)

3. Creating Physical Volumes (PVs) and Volume Group (VG)
The newly created LVM partitions were marked as Physical Volumes and pooled together into a single, cohesive storage pool named webdata-vg:

Bash
sudo pvcreate /dev/nvme1n1p1 /dev/nvme2n1p1 /dev/nvme3n1p1
sudo vgcreate webdata-vg /dev/nvme1n1p1 /dev/nvme2n1p1 /dev/nvme3n1p1
Verification of the storage pool using sudo vgs confirmed a total pooled space of approximately 30 GB.

4. Allocating Logical Volumes (LVs)
The pooled space was carved out into two dedicated Logical Volumes:

apps-lv: Allocated 14 GB to store the WordPress web files.

logs-lv: Allocated the remaining free space for system and application logging.

Bash
sudo lvcreate -L 14G -n apps-lv webdata-vg
sudo lvcreate -l 100%FREE -n logs-lv webdata-vg
5. Formatting, Mounting, and Log Data Preservation
The logical blocks were formatted using the ext4 filesystem:

Bash
sudo mkfs -t ext4 /dev/webdata-vg/apps-lv
sudo mkfs -t ext4 /dev/webdata-vg/logs-lv
Crucial Log Backup Step
To prevent losing existing system log data during initial volume mounting over /var/log, rsync was leveraged to safeguard the data:

Bash
sudo mkdir -p /var/www/html
sudo mkdir -p /home/recovery/logs

# Backup existing logs
sudo rsync -av /var/log/ /home/recovery/logs/

# Mount the fresh logical volumes
sudo mount /dev/webdata-vg/apps-lv /var/www/html/
sudo mount /dev/webdata-vg/logs-lv /var/log

# Restore log files into the newly mounted storage
sudo rsync -av /home/recovery/logs/ /var/log/
6. Establishing Storage Persistence (/etc/fstab)
To guarantee that mounts persist across system reboots, unique block identifiers (UUIDs) were obtained via sudo blkid. The entries were safely appended to /etc/fstab:

Plaintext
UUID=11111111-2222-3333-4444-555555555555  /var/www/html  ext4  defaults  0  0
UUID=abcdefab-cdef-abcd-efab-cdefabcdefab  /var/log       ext4  defaults  0  0
The setup was tested and reloaded with sudo mount -a and sudo systemctl daemon-reload to verify syntactic stability.

Part 2: Database Server Storage Deployment
A second RHEL EC2 instance was launched to act as the dedicated Database Tier. The LVM lifecycle steps were mirrored exactly, but instead of mapping an app directory, the data-centric logical volume (db-lv) was safely mounted to the /db directory to host the database engines.

Part 3: Application Installation and Database Integration
1. Web Layer Configuration (Apache & PHP)
On the Web Server, package mirrors were updated, and the Apache Web Server along with modern PHP 7.4 runtimes (via Remi repositories) were deployed to satisfy WordPress technical requirements:

Bash
sudo yum -y update
sudo yum install [https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm](https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm)
sudo yum install yum-utils [http://rpms.remirepo.net/enterprise/remi-release-8.rpms](http://rpms.remirepo.net/enterprise/remi-release-8.rpms)
sudo yum module reset php && sudo yum module enable php:remi-7.4
sudo yum install wget httpd php php-opcache php-gd php-curl php-mysqlnd php-fpm php-json -y
sudo systemctl enable --now httpd php-fpm
2. WordPress Deployment & Security Remediation (SELinux)
The latest WordPress binary distribution package was fetched, extracted, and placed directly in the web root. Crucially, native Linux kernel access controls were preserved by configuring explicit SELinux policies to allow Apache to safely write data files and communicate over the network:

Bash
# Configure ownership and SELinux context
sudo chown -R apache:apache /var/www/html/wordpress
sudo chcon -t httpd_sys_rw_content_t /var/www/html/wordpress -R
sudo setsebool -P httpd_can_network_connect=1
3. Database Layer Setup (MySQL)
On the isolated Database Server, MySQL Server was installed and initialized:

Bash
sudo yum install mysql-server -y
sudo systemctl enable --now mysqld
A dedicated database instance was created, and explicit permission grants were restricted strictly to the Web Server's internal private IP block for enhanced backend protection:

SQL
CREATE DATABASE wordpress;
CREATE USER 'myuser'@'<Web-Server-Private-IP-Address>' IDENTIFIED BY 'mypass';
GRANT ALL ON wordpress.* TO 'myuser'@'<Web-Server-Private-IP-Address>';
FLUSH PRIVILEGES;
Part 4: Frontend Verification & Successful Installation
With TCP port 80 opened on the Web Server's Security Group and port 3306 opened on the DB Server's Security Group, the setup wizard was accessed through a web browser.

Phase 1: Localization Selection
The web configuration routine successfully parsed PHP code, presenting the language initialization selector screen:

Phase 2: Administrative Provisioning
Administrative account metadata, including site heading variables (amar project 6 Devop) and security credentials, were mapped into the application layer:

Phase 3: Successful Deployment Database Handshake
The database configuration files successfully performed the internal connection sequence over the private network subnet:

Phase 4: Full Administration Control Panel
The installation concluded with full access to the backend admin dashboard, operating on the distributed LVM infrastructure:
