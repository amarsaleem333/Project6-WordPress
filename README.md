# Project6-WordPress
# Project 6: Three-Tier WordPress Web Solution with LVM Storage Infrastructure

This repository contains the complete documentation and deployment files for **Project 6: Web Solution with WordPress**, demonstrating a highly available, decoupled architecture utilizing a **Three-Tier Architecture pattern** on Red Hat Enterprise Linux (RHEL) EC2 instances

---

## 🏗️ System Architecture Overview

The project showcases a classic production-ready design split into three distinct management layers:
1. **Presentation Layer (PL):** Client laptop/browser accessing the application via HTTP Port 80
2. **Business/Application Layer (BL):** Apache (HTTPD) and PHP-FPM running on a dedicated Web Server instance (`172.31.43.63`)
3. **Data Access Layer (DAL):** A remote MySQL Database server (`172.31.35.138`) managing relational storage

---

## 🛠️ Step 1: Storage Subsystem & LVM Configuration (Web Server)

To establish dynamic storage bounds and isolate logical failures, three 10 GiB Amazon EBS volumes were mounted and pooled using Logical Volume Management (LVM2)[cite: 25, 42, 77].

### 1. Disk Partitioning
Raw NVMe devices (`/dev/nvme1n1`, `/dev/nvme2n1`, and `/dev/nvme3n1`) were partitioned via `fdisk`, initializing a modern GPT table and assigning the modern Linux LVM identifier 
```bash
sudo fdisk /dev/nvme1n1
# Interactive sequence used: g ➔ n ➔ Enter ➔ Enter ➔ Enter ➔ t ➔ 30 ➔ w
2. LVM Volume Creation & Safe Log MigrationBash# Create Physical Volumes
sudo pvcreate /dev/nvme1n1p1 /dev/nvme2n1p1 /dev/nvme3n1p1

# Pool into a Volume Group
sudo vgcreate webdata-vg /dev/nvme1n1p1 /dev/nvme2n1p1 /dev/nvme3n1p1

# Allocate Logical Volumes
sudo lvcreate -L 14G -n apps-lv webdata-vg
sudo lvcreate -l 100%FREE -n logs-lv webdata-vg
⚠️ Critical Step Avoidance of Data Loss: Pre-existing system logs inside /var/log were safely synchronized to a recovery buffer before mounting the new block device to avoid breaking core OS monitoring hooks.  Bashsudo mkdir -p /var/www/html /home/recovery/logs
sudo rsync -av /var/log/ /home/recovery/logs/
sudo mount /dev/webdata-vg/apps-lv /var/www/html/
sudo mount /dev/webdata-vg/logs-lv /var/log
sudo rsync -av /home/recovery/logs/ /var/log/
3. FSTAB Persistence RegistryBlock UUID identifiers (blkid) were harvested to ensure that the partition table persists gracefully across infrastructure restarts.  Plaintext# Configuration appended to /etc/fstab
UUID=your-apps-lv-uuid  /var/www/html  ext4  defaults  0  0
UUID=your-logs-lv-uuid  /var/log       ext4  defaults  0  0
Verify setup layout with df -h and lsblk.  🗄️ Step 2: Remote Database Infrastructure ProvisioningThe storage provisioning sequence was mirrored on the second RHEL instance (DB Server), creating a dedicated logical volume db-lv mapped to the database data directory /db.  1. MySQL Server Engine SetupBashsudo yum update -y
sudo yum install mysql-server -y
sudo systemctl enable --now mysqld
2. Relational Schema Blueprint & Target Security BoundsSQLCREATE DATABASE wordpress;
-- Security Hardening: Restricting authentication strictly to the Web Server internal IP
CREATE USER 'myuser'@'172.31.43.63' IDENTIFIED BY '********';
GRANT ALL PRIVILEGES ON wordpress.* TO 'myuser'@'172.31.43.63';
FLUSH PRIVILEGES;
📝 Step 3: Comprehensive Deployment Error Log & ResolutionsDuring deployment, a series of complex infrastructural errors were encountered and systematically resolved:🚨 Case 1: Read-Only File System ErrorSymptom: Execution scripts returned sed: couldn't open temporary file: Read-only file system.Root Cause: The underlying AWS storage block layer tripped into protective safety mode due to early disk resource saturation or allocation lockouts.Resolution: Performed a hardware reset/reboot cycle via the EC2 console dashboard to trigger automated background check arrays (fsck) to restore read-write capability (rw).🚨 Case 2: Continuous 504 Gateway Timeout ErrorsSymptom: The public browser endpoint hung indefinitely before printing a Gateway Timeout loop. Apache logged error code AH01075: Error dispatching request to : (polling).Root Cause 1 (Password Parsing Loop): The initial database password contained a trailing $ character (113355pasS$). The underlying PHP compiler interpreted this special character as an unset environment variable pointer, throwing an infinite parsing loop.Root Cause 2 (Apache to PHP-FPM Disconnect): PHP-FPM was bound natively to a local file stream socket (/run/php-fpm/www.sock), while Apache was searching over a TCP network boundary (127.0.0.1:9000).Resolution: 1. Updated the password inside MySQL and wp-config.php to a web-safe, policy-compliant format: '********'.2. Modified the handler definition inside Apache's PHP proxy interface configuration /etc/httpd/conf.d/php.conf:Apache<FilesMatch \.php$>
    SetHandler "proxy:unix:/run/php-fpm/www.sock|fcgi://localhost"
</FilesMatch>
Aligned file permissions across the communication socket: sudo chown apache:apache /run/php-fpm/www.sock.🚨 Case 3: Error Establishing a Database ConnectionSymptom: The gateway loop resolved, but the app dropped instantly into a database validation block.Root Cause: A tiny single-digit typo existed in the database host config string (173.31.35.138), routing queries outside the internal AWS ecosystem.Resolution: Corrected the line item in /var/www/html/wp-config.php to target the accurate internal network path without extra ports:PHPdefine( 'DB_HOST', '172.31.35.138' );
📸 Step-by-Step Step-by-Step Implementation ProgressAll active confirmation verification checkpoints have been packaged and hyperlinked from our core directory repository tree:1. Storage Infrastructure LayoutVisual verification displaying LVM allocation trees and ext4 mapping registries.2. Remote Client Validation HandshakeManual MySQL terminal proxy authentication connection test verification.3. Production Launch ScreenLive capture displaying the accessible WordPress deployment environment.(Note: Full set of step images are zipped inside the asset bundle file project6_images.zip in this root folder).🚀 Recommended Verification Steps Before Project SubmissionBefore submitting the solution files to your technical reviewer, execute these final operational assertions on your instances to verify complete structural compliance:  Verify Live Status Rules: Confirm Apache and PHP-FPM engine layers are stable on the web host:Bashsudo systemctl status httpd php-fpm
Verify Security Group Isolation Constraints: Ensure port 3306 on your Database instance strictly utilizes a /32 CIDR limit bounded solely to your Web Server's Private IP for optimal surface exposure reduction.  Validate Persistence Routines: Execute sudo mount -a to guarantee that your /etc/fstab configuration file possesses completely clean syntax parameters to avoid startup system stalls on boot.  Clean Active Session Cache Elements: When validating in a web browser, consistently maintain an active Incognito session to bypass local cookie storage locks.
