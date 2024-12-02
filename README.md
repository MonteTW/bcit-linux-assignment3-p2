This is an instruction to help you who want to build a website with a basic file system that consisted by multiple droplets and a load balancer with Arch Linux operating system implementing the modules `nginx` as the server, and `ufw` as the firewall on your droplets. <br>
The script will automatically generate a html file that displays some basic information of each droplet's system, including IP address, and  can be reached by linking to the IP address of the load balancer.

## Prerequisites

### Download and Upload the Latest Arch Linux Image
1. Go to https://gitlab.archlinux.org/archlinux/arch-boxes/-/packages
2. Download the file that has the `qcow2` extension
3. Upload the image to [DigitalOcean](https://cloud.digitalocean.com/) at `Backups & Snapshots` under `Manage`
### Create Droplets
Create two new droplets with settings below:<br>
1. Region: SFO3 (closest to you)
2. Select the custom image we just uploaded
3. Size and CPU: Basic Regular $4/month
4. SSH key: Select the one you want to connect to it (or select all to make it easy)
5. Tags: web
6. Project: depend on you
### Create Load Balancer
A load balancer can distribute traffic across multiple servers to enhance the availability, reliability, and scalability of applications by balancing the load, which helps prevent downtime and improves performance. 
#### Create the Load Balancer
Go to the `Network` under `Manage` in DigitalOcean and click `Create Load Balancer`. Setting as below:<br>
1. Type: Regional
2. Datacenter: San Francisco 3 (same as your droplets)
3. Network: External
4. Connect Droplets: web (by using the tag)
5. Name: Give it a unique name
6. Project: Same as your droplets
7. Others: stay default
### For Each Droplet
1. Update your operating system by running
```bash
sudo pacman -Syu
```

2. Reboot the system
```bash
sudo reboot 
```
Please wait a few minutes to wait for the droplet is done the reboot, then connect to the droplet again and continue the steps below.

3. Install `nginx` module
```bash
sudo pacman -S nginx
```

4. Install `ufw` module
```bash
sudo pacman -S ufw
```

4. Install `git` module
```bash
sudo pacman -S git
```

## Files
The files should be cloned to your droplets:<br>
1. generate_index<br>
   The script that automatically generates index.html.
2. generate-index.service<br>
   The .service file that can execute the script that should be copied to `/etc/systemd/system/`
3. generate-index.timer<br>
   The .timer file that activates the .service file everyday 5 am that should be copied to `/etc/systemd/system/`
4. nginx.conf<br>
   The `nginx` config file that should be copied to `/etc/nginx/nginx.conf`   
5. index.conf<br>
   The server block file that should be copied to `/etc/nginx/sites-available` and make a symbolic link to `/etc/nginx/sites-enabled`

6. file-one<br>
   An example file that you should put inside the `documents` folder and see it when link to the file system
7. file-two<br>
   An example file that you should put inside the `documents` folder and see it when link to the file system

## Tutorial
### Step 1: Create a System User

Create a user `webgen` with the command:
```bash
sudo useradd -r -m -d /var/lib/webgen -s /usr/bin/nologin webgen
```
- `-s /usr/bin/nologin` is a non-login user
- `-r` make the new user as a system user 
- `-m -d /var/lib/webgen` is making a home directory at the path of `/var/lib/webgen`

>The reasons why we are creating a system user with `nologin` shell to contain and execute the script and files for the server is due to security reasons:
>
>1. Setting the user with nologin shell to make sure that no interactive login is allowed, and no commands from user webgen is allowed, so that it's more difficult for hackers to make damages by brute forcing into the account
>2. Isolating the system running from regular users or root can prevent accidental adjustments breaking the system, the server, or the `nginx` 
>
>Reference: https://www.paloaltonetworks.ca/cyberpedia/what-is-the-principle-of-least-privilege

### Step 2: Create Certain Directories 
Use `mkdir` to create the directories `bin` and `HTML` inside `/var/lib/webgen` to make the tree as below:
```
/var/lib/webgen/
├── bin
├── documents
└── HTML
```

```bash
sudo mkdir /var/lib/webgen/bin
sudo mkdir /var/lib/webgen/HTML
sudo mkdir /var/lib/webgen/documents
```
### Step 3: `git clone` the Files from This Repository 
Make a `bin` directory in home if it doesn't exist
```bash
mkdir bin
```

Clone the files from git repository to make things easier
```bash
cd ~/bin
git clone <Repository URL>
```

Move the files to `/var/lib/webgen/bin`
```bash
cd bcit-linux-assignment3-p2
sudo mv ./* /var/lib/webgen/bin
```
### Step 4: Make the Files Executable and Change Ownership
Make the files in `/var/lib/webgen/bin` executable
```bash
sudo chmod -R 755 /var/lib/webgen/bin
```

Move `file-one` and `file-two` to `/var/lib/webgen/documents`
```bash
sudo mv /var/lib/webgen/file-one /var/lib/webgen/documents/
sudo mv /var/lib/webgen/file-two /var/lib/webgen/documents/
```

Change the ownership of `/var/lib/webgen` with all the subdirectories and files to user `webgen`
```bash
sudo chown -R webgen:webgen /var/lib/webgen
```

Script file `generate_index` at `/var/lib/webgen/bin` will generate `index.html` to `/var/lib/webgen/HTML` 
```bash
#!/bin/bash

set -euo pipefail

# this is the generate_index script
# you shouldn't have to edit this script

# Variables
KERNEL_RELEASE=$(uname -r)
OS_NAME=$(grep '^PRETTY_NAME' /etc/os-release | cut -d '=' -f2 | tr -d '"')
DATE=$(date +"%d/%m/%Y")
PACKAGE_COUNT=$(pacman -Q | wc -l)
OUTPUT_DIR="/var/lib/webgen/HTML"
OUTPUT_FILE="$OUTPUT_DIR/index.html"

MEMORY_INFO=$(free -m)
DISK_INFO=$(df -h | head -n 4)

# Ensure the target directory exists
if [[ ! -d "$OUTPUT_DIR" ]]; then
        echo "Error: Failed to create or access directory $OUTPUT_DIR." >&2
        exit 1
fi

# Create the index.html file
cat <<EOF > "$OUTPUT_FILE"
<!DOCTYPE html>
<html lang="en">
<head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>System Information</title>
</head>
<body>
        <h1>System Information</h1>
        <p><strong>Kernel Release:</strong> $KERNEL_RELEASE</p>
        <p><strong>Operating System:</strong> $OS_NAME</p>
        <p><strong>Date:</strong> $DATE</p>
        <p><strong>Number of Installed Packages:</strong> $PACKAGE_COUNT</p>
EOF

# add MEMORY_INFO table
cat <<EOF >> "$OUTPUT_FILE"
        <h2>Memory Information</h2>
        <table border="1" style="border-collapse: collapse">
        <tr>
            <th>Total</th>
            <th>Used</th>
            <th>Free</th>
            <th>Shared</th>
            <th>Buff/Cache</th>
            <th>Available</th>
        </tr>
EOF

echo "$MEMORY_INFO" | awk 'NR==2 {
        print "         <tr>"
        print "                 <td>" $2 " MB</td>"
        print "                 <td>" $3 " MB</td>"
        print "                 <td>" $4 " MB</td>"
        print "                 <td>" $5 " MB</td>"
        print "                 <td>" $6 " MB</td>"
        print "                 <td>" $7 " MB</td>"
        print "         </tr>"
        print " </table>"
}' >> "$OUTPUT_FILE"

# add DISK_INFO table
cat <<EOF >> "$OUTPUT_FILE"
        <h2>Disk Usage</h2>
        <table border="1" style="border-collapse: collapse">
        <tr>
                <th>Filesystem</th>
                <th>Size</th>
                <th>Used</th>
                <th>Available</th>
                <th>Use%</th>
                <th>Mounted On</th>
        </tr>
EOF

echo "$DISK_INFO" | awk 'NR>1 {
        print " <tr>"
        print "                 <td>" $1 "</td>"
        print "                 <td>" $2 "</td>"
        print "                 <td>" $3 "</td>"
        print "                 <td>" $4 "</td>"
        print "                 <td>" $5 "</td>"
        print "                 <td>" $6 "</td>"
        print " </tr>"
}' >> "$OUTPUT_FILE"

# add the closing tag of bod and html
cat <<EOF >> "$OUTPUT_FILE"
        </table>
</body>
</html>
EOF

# Check if the file was created successfully
if [ $? -eq 0 ]; then
        echo "Success: File created at $OUTPUT_FILE."
else
        echo "Error: Failed to create the file at $OUTPUT_FILE." >&2
        exit 1
fi

```
### Step 5: .service File
Service file is to run the script at `/var/lib/webgen/bin`
```bash
[Unit]
Description=Run generate_index script
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot
ExecStart=/var/lib/webgen/bin/generate_index
User=webgen
Group=webgen

[Install]
WantedBy=multi-user.target
```

Copy the service file to `/etc/systemd/system/`
```bash
sudo cp /var/lib/webgen/bin/generate-index.service /etc/systemd/system/
```

Run the file and test it
```bash
sudo systemctl start generate-index
```

Check the status of generate-index
```bash
systemctl status generate-index
```

Check if there's an `index.html` generated inside `/var/lib/webgen/HTML`
```bash
sudo ls -l /var/lib/webgen/HTML
```
### Step 6: .timer File
Timer file is to run the script at `/var/lib/webgen/bin`
```bash
[Unit]
Description=Run the service at 5 am

[Timer]
OnCalendar=*-*-* 05:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

Copy the timer file to `/etc/systemd/system/`
```bash
sudo cp /var/lib/webgen/bin/generate-index.timer /etc/systemd/system/
```

Since we put new files into `/etc/systemd/system/`, reload the files
```bash
 sudo systemctl daemon-reload
```

Run and enable the .timer file
```bash
sudo systemctl enable --now generate-index.timer
```

Check the status of `generate-index.timer`
```bash
systemctl status generate-index.timer
```

Change time zone
```bash
sudo timedatectl set-timezone America/Vancouver
```

Check whether the timer is counting at the background, and check the time left from now to 5 am is correct
```bash
systemctl list-timers
```

### Step 7: `nginx` Configuration

Replace the `/etc/nginx/nginx.conf` file by `nginx.conf` file
```bash
sudo cp /var/lib/webgen/bin/nginx.conf /etc/nginx/
```

nginx config file`/etc/nginx/nginx.conf`
```bash
user webgen;
worker_processes auto;
worker_cpu_affinity auto;

events {
    multi_accept on;
    worker_connections 1024;
}

http {
    charset utf-8;
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    server_tokens off;
    log_not_found off;
    types_hash_max_size 4096;
    client_max_body_size 16M;

    # MIME
    include mime.types;
    default_type application/octet-stream;

    # logging
    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log warn;

    # load configs
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

Create two specific directories for server block
```bash
sudo mkdir /etc/nginx/sites-available
sudo mkdir /etc/nginx/sites-enabled
```

Copy `index.conf` to `/etc/nginx/sites-available`
```bash
sudo cp /var/lib/webgen/bin/index.conf /etc/nginx/sites-available/
```

The server block config file
```bash
server {
    listen 80;
    listen [::]:80;

    server_name _;

    root  /var/lib/webgen/HTML;
    index index.html;
        location / {
        try_files $uri $uri/ =404;
    }
    location /documents {
        alias /var/lib/webgen/documents/;
        autoindex on;                # Enables the directory listing
        autoindex_exact_size off;    # Shows file sizes, human-readable
        autoindex_localtime on;      # Displays file timestamps
    }

}
```

Make a symbolic link to `/etc/nginx/sites-enabled`
```bash
sudo ln -s /etc/nginx/sites-available/index.conf /etc/nginx/sites-enabled/index.conf
```

Test the config file, make sure no error occurs
```bash
sudo nginx -t
```

Start and enable
```bash
sudo systemctl enable --now nginx
```

Check the status of `nginx`
```bash
systemctl status nginx
```

>**Why is it important to use a separate server block file instead of modifying the main `nginx.conf` file directly?**
>
>By putting different `server` blocks in different files. This allows you to easily enable or disable certain sites.
>
>we can disable a site by unlink the active symbolic link:
>`unlink /etc/nginx/sites-enabled/index.conf`
>
>Reference: https://wiki.archlinux.org/title/Nginx

Now, check the server is working as you want by put the IP address of your droplet to a browser to make sure the page that display your system information is implemented<br>
Then put the IP address of your load balancer to a browser to make sure the balancer is automatically switching the server between two droplets.<br>
Then put in `<IPAddress/documents>` to make sure you are able to see the file system with the two files and is able to download them to your local device.

### Step 8: `ufw` configuration
>**==IMPORTANT==**
>make sure you allow `ssh` and `http` before you run `sudo ufw enable` to enable the firewall

Enable and start the service of `ufw`
```bash
sudo systemctl enable --now ufw.service
```

Allowing `ssh` and `http`
```bash
sudo ufw allow ssh
sudo ufw allow http
```

Limit the connect attempt of `ssh` to enhance the security
```bash
sudo ufw limit ssh
```

Turn on the Firewall
```bash
sudo ufw enable
```

Check the firewall is activated
```bash
sudo ufw status verbose
```

Lastly, put the IP address of your load balancer to a browser to make sure everything is still working fine.