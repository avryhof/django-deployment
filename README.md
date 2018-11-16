# How to Deploy a Django Site
Adapted from the [DigitalOcean docs](https://www.digitalocean.com/community/tutorials/how-to-set-up-django-with-postgres-nginx-and-gunicorn-on-ubuntu-16-04).

* Replace nano with the editor of your choice, this is not the place for a text editor holy war.

## Setting up Python

If your application uses Python2

```bash
sudo apt-get install python-pip python-dev libpq-dev
sudo -H pip install --upgrade pip
sudo -H pip install virtualenv
```

If your application uses Python3
```bash
sudo apt-get install python3-pip python3-dev libpq-dev
sudo -H pip3 install --upgrade pip
sudo -H pip3 install virtualenv
```

## Setting up Postgresql and creating a database
Installing Postgresql
```bash
sudo apt-get install postgresql postgresql-contrib
sudo -u postgres psql
```
Creating your Database and user
```postgresql
CREATE USER myprojectuser WITH PASSWORD 'password';
ALTER ROLE myprojectuser SET client_encoding TO 'utf8';
ALTER ROLE myprojectuser SET default_transaction_isolation TO 'read committed';
ALTER ROLE myprojectuser SET timezone TO 'UTC';
CREATE DATABASE myprojectdb;
GRANT ALL PRIVILEGES ON DATABASE myprojectdb TO myprojectuser;
```

## Creating your VirtualEnv
```bash
sudo su
mkdir -p /var/www/virtualenvs
cd /var/www/virtualenvs
virtualenv myprojectenv
```
* This configuration is for more secure websites that store sensitive data outside of the website root, as it should be. 
```bash
cd myprojectenv/bin
nano activate
```
Scroll to the bottom of your activate file, and add in environmental variables you want to be available to your project.
* I like to have a different prefix for each project that might be hosted on the server. (PRJ_ in the example below)
```bash
export PRJ_SECRET_KEY='<your secret key>'
export PRJ_DB_ENGINE=django.db.backends.postgresql_psycopg2
export PRJ_DB_NAME='myprojectdb'
export PRJ_DB_USER='myprojectuser'
export PRJ_DB_PASS='<database password>'
export PRJ_DB_HOST='<database host: localhost>'
export PRJ_DB_PORT='<database port: 5432>'
```

## Install nginx
We aren't configuring it just yet, but we want to make sure the www-data user exists.
```bash
sudo apt-get install nginx
```

## Your Django Project
If you have already developed a Django project, congratulations!!! Let's get this puppy hosted.

* I personally like to build my project, push it to source control, then clone it down to the web server.
  * Be aware of sensitive files, and make sure you don't check them into source control, or someone will go steal them from your Git repository and hack your site!  
* If this isn't your thing, that's fine.  You can put it up with scp, sftp, ftp, or copy/paste each file one by one.  It's not my job to judge how you do it.
  * The sensitive files warning applies here, too.

First, I build a basic directory structure for my project.  Remember that we are making sure it's possible to host multiple sites on your server should you need to.

Another handy thing to use is build directories, that way you can deploy complete builds when you make major updates, and easily roll back to a previous version just by replacing a symlink.
```bash
sudo su
mkdir -p /var/www/myproject
mkdir -p /var/www/myproject/logs
mkdir -p /var/www/myproject/media
mkdir -p /var/www/myproject/static
mkdir -p /var/www/myproject/run
mkdir -p /var/www/myproject/build/YYYY-MM-DD
cd /var/www/myproject
chown www-data:www-data media
cd ~/var/www/myproject/build
# This step is just putting your project into the YYYY-MM-DD folder. This would be the folder containing manage.py
git clone https://github.com/mycooldevname/myproject YYYY-MM-DD
source /var/www/virtualenvs/myprojectenv/bin/activate
# For your existing Django project.
pip install -r requirements.txt
./manage.py migrate
./manage.py makemigrations
./manage.py migrate
# END Parts for your existing Django project
pip install gunicorn
```

## Setup Gunicorn
In the previous section, you installed gunicorn into your VirtualEnv.  Now, we will configure it.

First, create a script to start gunicorn with your project specific settings
```bash
sudo nano /var/www/virtualenvs/myprojectenv/bin/gunicorn_start.sh
```
Make the file look something like this.   
```bash
#!/bin/bash

NAME="myproject"
PRJ_VENV=/var/www/virtualenvs/myprojectenv
PRJ_HOME=/var/www/myproject

RUNDIR=${PRJ_HOME}/run
SOCKFILE=${RUNDIR}/myproject.sock
DJANGO_WSGI_MODULE=myproject.wsgi
TIMEOUT=60
LOG_LEVEL=info
CERTFILE=/etc/ssl/mycert.crt;
KEYFILE=/etc/ssl/mycert.key;
USER=www-data
GROUP=www-data
NUM_WORKERS=3
export PYTHONPATH=${PRJ_HOME}/htdocs:${PYTHONPATH}

cd ${PRJ_HOME}
# This activates your virtual environment, and brings in all the environmental variables you exported in your activate file.
source ${PRJ_VENV}/bin/activate

exec gunicorn ${DJANGO_WSGI_MODULE}:application --name $NAME --workers $NUM_WORKERS -t $TIMEOUT  --user=$USER --group=$GROUP --bind unix:${SOCKFILE} --log-level=$LOG_LEVEL --log-file ${PRJ_HOME}/logs/myproject.err --access-logfile ${PRJ_HOME}/logs/myproject.access
```

Don't forget to make it executable
```bash
chmod +x /var/www/virtualenvs/myprojectenv/bin/gunicorn_start.sh
```

## Create a systemd Service File

```bash
sudo nano /etc/systemd/system/myproject_gunicorn.service
```

Make it look like this.

You are calling your gunicorn_start script that you created in the section above because it activates your environment and exports all of your environmental variable before executing your highly customized gunicorn command.
```bash
[Unit]
Description=myproject gunicorn daemon
After=network.target

[Service]
User=www-data
Group=www-data
WorkingDirectory=/var/www/myproject
ExecStart=/var/www/virtualenvs/myprojectenv/bin/gunicorn_start.sh

[Install]
WantedBy=multi-user.target
```

Now you can control your gunicorn instance with systemctl
```bash
sudo systemctl start myproject_gunicorn
sudo systemctl enable myproject_gunicorn

sudo systemctl restart myproject_gunicorn

sudo systemctl stop myproject_gunicorn
```

## Configure nginx to proxy-pass to your gunicorn
First, create your nginx configuration file
```bash
sudo nano /etc/nginx/sites-available/myproject
```

Now, make it look like this -

* Make sure you specify your server name (or names, separated by a space) because it will make adding SSL easier in the next section, and there is no excuse not to.
```
server {
    listen 80;
    server_name myproject.com www.myproject.com;

    location = /favicon.ico { access_log off; log_not_found off; }
    
    location /static/ {
        root /var/www/myproject;
    }
    
    location /media/ {
        root /var/www/myproject;
    }

    location / {
        include proxy_params;
        proxy_pass http://unix:/var/www/myproject/run/myproject.sock;
    }
}
```

Symlink it into the sites-enabled folder

```bash
sudo ln -s /etc/nginx/sites-available/myproject /etc/nginx/sites-enabled
```

Test your nginx configuration
```bash
sudo nginx -t
```

Restart nginx
```bash
sudo systemctl restart nginx
```

## Allowing your application through your firewall
Ubuntu uses UFW for a firewall.

* We'll allow nginx full, since we will also want SSL. (you do, and it won't cost you!)
```bash
sudo ufw allow 'Nginx Full'
```

## Adding SSL To your website (just do it!)
First, install Certbot
```bash
sudo add-apt-repository ppa:certbot/certbot
sudo apt-get update
sudo apt-get install python-certbot-nginx
```

Generate your certificate... and install it (yeah, it's that easy!)
* You can add additional domains or subdomains to your cert with additional -d parameters.
```bash
sudo certbot --nginx -d myproject.com -d www.myproject.com
```

When you see the message below, choose "2"
```
Please choose whether or not to redirect HTTP traffic to HTTPS, removing HTTP access.
-------------------------------------------------------------------------------
1: No redirect - Make no further changes to the webserver configuration.
2: Redirect - Make all requests redirect to secure HTTPS access. Choose this for
new sites, or if you're confident your site works on HTTPS. You can undo this
change by editing your web server's configuration.
-------------------------------------------------------------------------------
Select the appropriate number [1-2] then [enter] (press 'c' to cancel):
```

This will give you an A grade on the [SSL Labs Server Test](https://www.ssllabs.com/ssltest/).  Not too shabby for 4 lines.

## Final Nginx configuration
If you are happy with an A on the SSL Labs Server Test, you can skip this section.

If not, read on!

Looking at [Getting a Qualys SSL Labs A+ rating with Nginx](https://sanderknape.com/2016/06/getting-ssl-labs-rating-nginx/) he has the following items in his Table of Contents

* Setting up the server (done above!)
* Getting a SSL Certificate (Certbot did this for us!)
* Installing nginx (yeah, we got it!)
* Forward Secrecy and Diffie-Hellman
  * Fixing the weak keys (Certbot did this, too!)
  * Fixing “Forward Secrecy not available for reference browsers”
* Getting the A+ Rating (forcing SSL!)
* Taking it even further
  * HSTS preloading
  * Session resumption
* OCSP stapling

Looks like we've done several of the steps already.
  
Open your site's nginx configuration
```bash
sudo nano /etc/nginx/sites-available/myproject
```

You will notice that there is now a server block with
```
server {
  listen 443 ssl;
```

That was put in there by certbot, as well as the directive to redirect all traffic there.

This is the server block that all of the lines below go in.  If the line is already there... replace it.

### Fixing pretty much all of the rest....
```
ssl_ciphers 'ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA:ECDHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA256:DHE-RSA-AES256-SHA:ECDHE-ECDSA-DES-CBC3-SHA:ECDHE-RSA-DES-CBC3-SHA:EDH-RSA-DES-CBC3-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:DES-CBC3-SHA:!DSS';
ssl_prefer_server_ciphers on;
ssl_session_timeout 1d;
ssl_session_cache shared:SSL:50m;
ssl_session_tickets off;
ssl_stapling on;
ssl_stapling_verify on;
resolver 8.8.8.8 8.8.4.4 valid=300s;
```

### Submitting your site to the HSTS Pre-Load list

Doing this tells browsers to never visit your website through http.

[HSTS Preload List](https://hstspreload.org/)
