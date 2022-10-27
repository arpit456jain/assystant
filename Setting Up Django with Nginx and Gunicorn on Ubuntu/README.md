First intsall the required modules

sudo apt update
sudo apt install python3-venv nginx curl

mkdir ~/myprojectdir
cd ~/myprojectdir

Within the project directory, create a Python virtual environment by typing:

python3 -m venv env

Activate

source env/bin/activate

pip install django gunicorn psycopg2-binary

Creating and Configuring a New Django Project
With your Python components installed, you can now create the actual Django project files.

Creating the Django Project
django-admin startproject myproject ~/myprojectdir


At this point, your project directory (~/myprojectdir in this example case) should have the following content:

~/myprojectdir/manage.py: A Django project management script.
~/myprojectdir/myproject/: The Django project package. This should contain the __init__.py, settings.py, urls.py, asgi.py, and wsgi.py files.
~/myprojectdir/myprojectenv/: The virtual environment directory you created earlier.

Adjusting the Project Settings
The first thing you should do with your newly created project files is adjust the settings. Open the settings file in your text editor:

nano ~/myprojectdir/myproject/settings.py

ALLOWED_HOSTS = ['your_server_domain_or_IP', 'second_domain_or_IP', . . ., 'localhost']
import os
STATIC_ROOT = os.path.join(BASE_DIR, 'static/')

Completing Initial Project Setup
Now, you can migrate the initial database schema to our PostgreSQL database using the management script:

~/myprojectdir/manage.py makemigrations
~/myprojectdir/manage.py migrate
~/myprojectdir/manage.py createsuperuser
~/myprojectdir/manage.py collectstatic

Create an exception for port 8000 by typing:
sudo ufw allow 8000

Finally, you can test out your project by starting up the Django development server with this command:

~/myprojectdir/manage.py runserver 0.0.0.0:8000
In your web browser, visit your server’s domain name or IP address followed by :8000:

http://server_domain_or_IP:8000


Testing Gunicorn’s Ability to Serve the Project
The last thing you need to do before leaving your virtual environment is test Gunicorn to make sure that it can serve the application. You can do this by entering the project directory and using gunicorn to load the project’s WSGI module:

cd ~/myprojectdir
gunicorn --bind 0.0.0.0:8000 myproject.wsgi
This will start Gunicorn on the same interface that the Django development server was running on. You can go back and test the app again in your browser.

You’re now finished configuring your Django application. You can back out of our virtual environment by typing:

deactivate

Creating systemd Socket and Service Files for Gunicorn
sudo nano /etc/systemd/system/gunicorn.socket

in the file  : /etc/systemd/system/gunicorn.socket write this .

[Unit]
Description=gunicorn socket

[Socket]
ListenStream=/run/gunicorn.sock

[Install]
WantedBy=sockets.target

Save and close the file when you are finished.

Next, create and open a systemd service file for Gunicorn with sudo privileges in your text editor. The service filename should match the socket filename with the exception of the extension:
sudo nano /etc/systemd/system/gunicorn.service

in the file  : /etc/systemd/system/gunicorn.service write

[Unit]
Description=gunicorn daemon
Requires=gunicorn.socket
After=network.target

[Service]
User=sammy
Group=www-data
WorkingDirectory=/home/sammy/myprojectdir
ExecStart=/home/sammy/myprojectdir/myprojectenv/bin/gunicorn \
          --access-logfile - \
          --workers 3 \
          --bind unix:/run/gunicorn.sock \
          myproject.wsgi:application

[Install]
WantedBy=multi-user.target

With that, your systemd service file is complete. Save and close it now.

sudo systemctl start gunicorn.socket
sudo systemctl enable gunicorn.socket

if you got any error like this : o systemctl enable gunicorn.socketSystem has not been booted with systemd as init system (PID 1). Can't operate.
Failed to connect to bus: Host is down

Try this : 
1 – Update the system
sudo apt-get update

2 – Install below package
sudo apt-get install -yqq daemonize dbus-user-session fontconfig

3 – execute below command to fix the issue
sudo daemonize /usr/bin/unshare --fork --pid --mount-proc /lib/systemd/systemd --system-unit=basic.target

exec sudo nsenter -t $(pidof systemd) -a su - $LOGNAME

4 – check the version
snap version

Now the error will be resolved

Checking for the Gunicorn Socket File
Check the status of the process to find out whether it was able to start:

sudo systemctl status gunicorn.socket

Next, check for the existence of the gunicorn.sock file within the /run directory:

file /run/gunicorn.sock
Output
/run/gunicorn.sock: socket

If the systemctl status command indicated that an error occurred or if you do not find the gunicorn.sock file in the directory, it’s an indication that the Gunicorn socket was not able to be created correctly. Check the Gunicorn socket’s logs by typing:

sudo journalctl -u gunicorn.socket


Testing Socket Activation
Currently, if you’ve only started the gunicorn.socket unit, the gunicorn.service will not be active yet since the socket has not yet received any connections. You can check this by typing:

sudo systemctl status gunicorn
Output
○ gunicorn.service - gunicorn daemon
     Loaded: loaded (/etc/systemd/system/gunicorn.service; disabled; vendor preset: enabled)
     Active: inactive (dead)
TriggeredBy: ● gunicorn.socket


To test the socket activation mechanism, you can send a connection to the socket through curl by typing:

curl --unix-socket /run/gunicorn.sock localhost

You should receive the HTML output from your application in the terminal. This indicates that Gunicorn was started and was able to serve your Django application. You can verify that the Gunicorn service is running by typing:

sudo systemctl status gunicorn
Output
● gunicorn.service - gunicorn daemon
     Loaded: loaded (/etc/systemd/system/gunicorn.service; disabled; vendor preset: enabled)
     Active: active (running) since Mon 2022-04-18 17:54:49 UTC; 5s ago
TriggeredBy: ● gunicorn.socket

Check your /etc/systemd/system/gunicorn.service file for problems. If you make changes to the /etc/systemd/system/gunicorn.service file, reload the daemon to reread the service definition and restart the Gunicorn process by typing:

sudo systemctl daemon-reload
sudo systemctl restart gunicorn

Configure Nginx to Proxy Pass to Gunicorn
Now that Gunicorn is set up, you need to configure Nginx to pass traffic to the process.

Start by creating and opening a new server block in Nginx’s sites-available directory:

sudo nano /etc/nginx/sites-available/myproject

/etc/nginx/sites-available/myproject
server {
    listen 80;
    server_name server_domain_or_IP;

    location = /favicon.ico { access_log off; log_not_found off; }
    location /static/ {
        root /home/sammy/myprojectdir;
    }

    location / {
        include proxy_params;
        proxy_pass http://unix:/run/gunicorn.sock;
    }
}


Save and close the file when you are finished. Now, you can enable the file by linking it to the sites-enabled directory:

sudo ln -s /etc/nginx/sites-available/myproject /etc/nginx/sites-enabled
Test your Nginx configuration for syntax errors by typing:

sudo nginx -t
If no errors are reported, go ahead and restart Nginx by typing:

sudo systemctl restart nginx

Finally, you need to open up your firewall to normal traffic on port 80. Since you no longer need access to the development server, you can remove the rule to open port 8000 as well:

sudo ufw delete allow 8000
sudo ufw allow 'Nginx Full'
You should now be able to go to your server’s domain or IP address to view your application.

