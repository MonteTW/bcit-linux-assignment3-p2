This is an instruction to help you who has a droplet with Arch Linux operating system to implement the modules `nginx` as the server, and `ufw` as the firewall on your droplet with a script that will automatically generate a html file that displays some basic information of the droplet's system and hold it on the IP address of your droplet.

## Prerequisites
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

## Files
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
6. screenshot.png<br>
   The success screenshot of the index.html output by link to the IP address through a browser.

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
└── HTML
```

```bash
sudo mkdir /var/lib/webgen/bin
sudo mkdir /var/lib/webgen/HTML
```
### Step 3: `git clone` the Files from This Repository 
Clone the files from this repository to `/var/lib/webgen/bin`
```bash
cd /var/lib/webgen/bin
git clone <Repository URL>
```

### Step 4: Make the Files Executable and Change Ownership
Make the files in `/var/lib/webgen/bin` executable
```bash
sudo chmod 755 *
```

Change the ownership of `/var/lib/webgen` with all the subdirectories and files to user `webgen`
```bash
sudo chown -R webgen:webgen /var/lib/webgen
```

### Step 5: .service File
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

### Step 6: .timer File
Copy the timer file to `/etc/systemd/system/`
```bash
sudo cp /var/lib/webgen/bin/generate-index.timer /etc/systemd/system/
```

Reload the unix file
```bash
 sudo systemctl daemon-reload
```

Run and enable the .timer file
```bash
sudo systemctl start generate-index.timer
sudo systemctl enable generate-index.timer
```

Check the status of `generate-index.timer`
```bash
systemctl status generate-index.timer
```

Check the timer is counting at the background, and check the time left from now to 5 am is correct
```bash
systemctl list-timers
```

### Step 7: `nginx` Configuration

Replace the `/etc/nginx/nginx.conf` file by `nginx.conf` file
```bash
sudo rm /etc/nginx/nginx.conf
sudo cp /var/lib/webgen/bin/nginx.conf /etc/nginx/
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

Make a symbolic link to `/etc/nginx/sites-enabled`
```bash
ln -s /etc/nginx/sites-available/index.conf /etc/nginx/sites-enabled/index.conf
```

Test the config file, make sure no error occurs
```bash
sudo nginx -t
```

Start and enable
```bash
sudo systemctl start nginx
sudo systemctl enable nginx
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

Now, check the server is working as you want by put the IP address of your droplet to a browser to make sure the page that display your system information is implemented

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

Limit the connect attempt of `ssh`
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

## Extra: Script Enhancement
Adding additional system information
1. Ram usage
```bash
free -m
```
By using the command we can see the basic ram usage of the system, and use `>>` to append information after the original information section.<br>

Use `awd` to print the second line to the output file with positional parameters to specify the information we need.<br>
   
2. Disk usage
```bash
df -h | head -n 4
```
By using the command we can see the disk usage of the system, and use `>>` to append information after the ram usage section.<br>

Since only top 4 lines of the information contains disk usage from the system, we show only the top 4 lines by using `head -n 4` to reach the result.<br>

Use `awd` to print the lines after the title line to the output file with positional parameters to specify the information we need.<br>


load balancer
http://209.38.175.151/