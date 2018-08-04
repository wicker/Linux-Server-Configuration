# Linux-Server-Configuration
Udacity FSND Project #5 

## Access Information

- IP address: 34.204.11.214
- URL: http://wallowawildlife.com

## Item Catalog App

The full app is called Wallowa Wildlife and can be found at its own [Github repo](https://github.com/wicker/Wallowa-Wildlife-Checklist-App).

## Software Installed

- finger
- nginx
- python3, python3-pip, python3-dev
- libpcre3, lipbcre3-dev

Inside the virtual environment,

- virtualenv
- uwsgi
- flask

## Configurations

#### Create a new user with appropriate permissions

Followed [these instructions](https://aws.amazon.com/premiumsupport/knowledge-center/new-user-accounts-linux-instance/) to add a user called `grader` in the sudoers group.

```
sudo adduser grader
sudo adduser grader sudo
sudo su grader
cd 
mkdir .ssh
touch .ssh/authorized_keys
chmod 600 .ssh/authorized_keys
```

Pasted the public key for the EC2 instance into `authorized_keys`.

Tested signing in from my local laptop.


#### Update all system packages

```
sudo apt update
sudo apt upgrade
```

#### Restrict Network Access

Adjusted the firewall rules on the server. Temporarily continue to allow ssh until ssh on 2200 has been tested.

Checked the current status.

```
sudo ufw status
```

Reset the default rules.

```
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

Set the particular services.

```
sudo ufw allow ssh
sudo ufw allow 2200/tcp
sudo ufw allow www
sudo ufw allow ntp
```

Applied the rules.

```
sudo ufw enable
```

Checked the rules.

```
sudo ufw status

Status: active

To                         Action      From
--                         ------      ----
Nginx Full                 ALLOW       Anywhere                  
22                         ALLOW       Anywhere                  
2200/tcp                   ALLOW       Anywhere                  
80/tcp                     ALLOW       Anywhere                  
123                        ALLOW       Anywhere                  
Nginx Full (v6)            ALLOW       Anywhere (v6)             
22 (v6)                    ALLOW       Anywhere (v6)             
2200/tcp (v6)              ALLOW       Anywhere (v6)             
80/tcp (v6)                ALLOW       Anywhere (v6)             
123 (v6)                   ALLOW       Anywhere (v6) 
```

Created a new security group under `Network & Security` > `Security Groups` in the EC2 console to allow the following, and applied that group to the instance.

|Protocol|Port|
|--------|----|
|SSH|22|
|SSH|2200|
|HTTP|80|
|NTP|123|

Finally, manually edit `/etc/init.d/ssh/sshd_config`.

```
# What ports, IPs and protocols we listen for
Port 2200
```

```
PermitRootLogin no
```

Restarted the SSH service.

```
sudo service ssh restart
```

Logged out and updated my local `.ssh/config` entry.

```
Host wallowagrader
Hostname 34.204.11.214
User grader
Port 2200
IdentityFile /home/me/.ssh/wallowawildlife.pem
```

Verified login with `ssh wallowagrader` and updated `ufw` and the EC2 instance to remove the SSH on port 22 rules. 

```
sudo ufw deny ssh
sudo ufw status

Status: active

To                         Action      From
--                         ------      ----
Nginx Full                 ALLOW       Anywhere                  
22                         DENY        Anywhere                  
2200/tcp                   ALLOW       Anywhere                  
80/tcp                     ALLOW       Anywhere                  
123                        ALLOW       Anywhere                  
Nginx Full (v6)            ALLOW       Anywhere (v6)             
22 (v6)                    DENY        Anywhere (v6)             
2200/tcp (v6)              ALLOW       Anywhere (v6)             
80/tcp (v6)                ALLOW       Anywhere (v6)             
123 (v6)                   ALLOW       Anywhere (v6)   
```

Verified the remote root login is disallowed for both port 22 and 2200. The attempt on port 22 eventually times out because the server isn't listening.

```
wicker@expis:~$ ssh root@34.204.11.214 -p 2200
Permission denied (publickey).

wicker@expis:~$ ssh root@34.204.11.214 -p 22
^C
```

Verified grader login is also disallowed without a key.

```
wicker@expis:~$ ssh grader@34.204.11.214 -p 2200
Permission denied (publickey).
```

#### Set up the Item Catalog App 

Set A records for `@` and `www` to point wallowawildlife.com the EC2 public IP `34.204.11.214`.  

Logged in with `grader` and started configuring the Wallowa Wildlife Flask application to be served by Nginx and uWSGI.

```
sudo apt install python3-pip python3-dev nginx
sudo pip3 install virtualenv 
```

Created the app folders and the virtual environment:

```
mkdir ~/public/wallowawildlife.com
cd ~/public/wallowawildlife.com
virtualenv wallowa-venv
source wallowa-venv/bin/activate
pip install uwsgi flask
deactivate
```

Used scp from my local machine to copy over the Flask application, which uses an app factory.

Created the WSGI entry point.

```
vim ~/public/wallowawildlife.com/wsgi.py
```

```
from wallowawildlife import create_app

app = create_app()

if __name__ == "__main__":
    app.run()
```

Created the uWSGI config file, which will set up the socket.

```
vim ~/public/wallowawildlife.com/wallowawildlife.ini
```

```
[uwsgi]
module = wsgi:app

master = true
processes = 5

socket = wallowawildlife.sock
chmod-socket = 660
vacuum = true

die-on-term = true
```

Next, set up the systemd service unit file so the server will start the app when it boots.

```
sudo vim /etc/systemd/system/wallowawildlife.service
```

```
[Unit]
Description=uWSGI instance to serve Wallowa Wildlife Checklists App
After=network.target

[Service]
User=grader
Group=www-data
WorkingDirectory=/home/grader/public/wallowawildlife.com
Environment="PATH=/home/grader/public/wallowawildlife.com/wallowa-venv/bin"
ExecStart=/home/grader/public/wallowawildlife.com/wallowa-venv/bin/uwsgi --ini wallowawildlife.ini

[Install]
WantedBy=multi-user.target
```

Then started the service.

```
sudo systemctl start wallowawildlife
sudo systemctl enable wallowawlidlife
```

Configured Nginx to proxy requests.

```
sudo vim /etc/nginx/sites-available/wallowawildlife.com
```

```
server {
	server_name wallowawildlife.com;

	location / {
	include uwsgi_params;
	uwsgi_pass unix:///home/grader/public/wallowawildlife.com/wallowawildlife.sock;
	}

	listen 80;

	access_log         /home/grader/public/wallowawildlife.com/logs/access.log;
	error_log          /home/grader/public/wallowawildlife.com/logs/error.log;
}
```

Then enabled and tested the Nginx server block configuration.

```
sudo ln -s /etc/nginx/sites-available/wallowawildlife.com /etc/nginx/sites-enabled
sudo nginx -t 
sudo systemctl restart nginx
```

#### Testing the App

Left the database as SQLite handled from inside Flask.

Added the site name `http://wallowawildlife.com` to Google's allowed list of referrers for the login process. 

The site had previously been set up to support Certbot (LetsEncrypt) and was forcing Port 80 HTTP -> Port 443 HTTPS redirect, so that had to be removed.  

#### Final Firewall Config

Removed the rule allowing port 22 from the EC2 security group.

Troubleshooting to bring up the app resulted in the ufw settings being a mess and needed to be reset.

```
sudo ufw reset

sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 2200/tcp
sudo ufw allow ntp
sudo ufw allow www
sudo ufw enable
sudo ufw status
```

```
Status: active

To                         Action      From
--                         ------      ----
2200/tcp                   ALLOW       Anywhere                  
123                        ALLOW       Anywhere                  
80/tcp                     ALLOW       Anywhere                  
2200/tcp (v6)              ALLOW       Anywhere (v6)             
123 (v6)                   ALLOW       Anywhere (v6)             
80/tcp (v6)                ALLOW       Anywhere (v6)
```

## Third Party Resources

- [AWS EC2 Docs](https://aws.amazon.com/ec2/)
- [DigitalOcean Tutorial](https://www.digitalocean.com/community/tutorials/how-to-serve-flask-applications-with-uwsgi-and-nginx-on-ubuntu-16-04), which is incorrect
- [StackOverflow](https://stackoverflow.com/) and [ServerFault](https://serverfault.com/) for Nginx, OAuth2, and Certbot
- [LetsEncrypt Community Support](https://community.letsencrypt.org) for Certbot
