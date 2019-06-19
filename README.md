# Deploying CTFd with uWSGI, NGINX, and MySQL


CTFd is an open-source Capture The Flag scoring application that utilizes Python Flask. The purpose of this guide is to show how I implemented CTFd, so that my steps may be replicated. 

I didn't know much about any of the frameworks used, so I figured I'd create this guide for people in my position!

This guide assumes that you already have a working Ubuntu Server 18.04 set up with a superuser. 


##### Frameworks Used
  - CTFd v2.1.2 (https://github.com/CTFd/CTFd/)
  - Python Flask with Nginx and uWSGI (https://github.com/source-nerd/Python-flask-with-uwsgi-and-nginx)
  - Amazon EC2 Instance
  - Ubuntu Server v18.04
  - MySQL v5.7
  - NGINX v1.14.0
  - uWSGI v2.0.18

##### Guides followed + resources used 

https://github.com/CTFd/CTFd/wiki

https://docs.ctfd.io/en/latest/

https://github.com/source-nerd/Python-flask-with-uwsgi-and-nginx 

https://www.digitalocean.com/community/tutorials/how-to-serve-flask-applications-with-uswgi-and-nginx-on-ubuntu-18-04

https://gist.github.com/mplewis/6076082

#### Additional Notes

This was implemented on an AWS server that already had a base webpage. My task was to implement this on a subdirectory of this already existing website structure. I chose to deploy it to www.domain.com/ctf. 

Let's get into the terminal commands I used to implement this! I was working under the home directory of the user previously set up on the Ubuntu server. For this tutorial's sake, lets make the name of the user  Ubuntu. If there are any packages that I use that you don't have (i.e. git), just download them from apt.

##### Database Setup
The first thing you should do is ensure MariaDB isn't installed on your system, as it isn't compatible with CTFd's syntax (they use JSON as a datatype, which isn't supported by the default MariaDB, version 10.1.40, provided in Ubuntu 18.04). MariaDB 10.4 was recommended to me by ColdHeat (creator of CTFd), but since that version isn't stable as of the time I'm writing this, I decided to use MySQL. If you've already installed MariaDB, here's what I referenced to purge it from the system, followed by the commands I used. 
https://askubuntu.com/questions/806107/remove-mariadb-mysql-databases

```sh
$ sudo service mysql stop
$ sudo apt-get --purge remove "mysql*"
$ sudo rm -rf /etc/mysql/
#You can check to see if anything unwanted is still there using "locate mysql"
```
After removing any databases from your server, we're going to install MySQL 5.7. I followed the following guide: https://www.itzgeek.com/how-tos/linux/ubuntu-how-tos/how-to-install-mysql-8-0-on-ubuntu-18-04-lts.html#Install_MySQL_57_on_Ubuntu_1804
For some reaosn, when I chose mysql 8.0, it would still install mysql 5.7. I'm sure mysql 8.0 would work as well.
Here's my terminal commands:

```sh
$ sudo wget https://dev.mysql.com/get/mysql-apt-config_0.8.10-1_all.deb
$ sudo dpkg -i mysql-apt-config_0.8.10-1_all.deb
$ sudo apt-get update
$ sudo apt-get -y install mysql-server
```

Then I set a superuser in mysql so that CTFd would be using that, not root. You can view MySQL's syntax for this. https://dev.mysql.com/doc/refman/5.7/en/dynindex-statement.html. I only set the superuser for localhost.
Now you should be ready to move on!

#####  Installing and Configuring Everything
Here's the meat of the project. This is where you're downloading anything needed, and configuring all the files needed for the project to work.
```sh
$ cd
$ sudo apt-get update
$ sudo apt-get install python3 python3-pip
$ sudo apt-get install systemd nginx
$ sudo pip3 install virtualenv
$ git clone https://github.com/source-nerd/Python-flask-with-uwsgi-and-nginx #We will be using the uwsgi.ini and production_flask.service files from this. We will also isntall anything from requirements.txt.
$ cd 
$ sudo mv Python-flask-with-uwsgi #Renamed for readability
$ cd CTFd/app
$ git clone https://github.com/CTFd/CTFd.git #This is the CTFd repository
$ virtualenv -p python3 venv
$ source venv/bin/activate 
#These last two lines initialized a virtual Python environment. Your terminal should now look like (venv) $
(venv) $ pip3 install -r requirements.txt #This is source-nerd's requirements.txt file
(venv) $ cd CTFd/
(venv) $ pip3 install -r requirements.txt #This is CTFd's requirements.txt file
(venv) $ ./prepare.sh
(venv) $ deactivate #This ends the virtual environment.
#Navigate to the directory where uwsgi.ini is located (should be in source-nerd's directory)
$ sudo nano uwsgi.ini
```
I changed the following within uwsgi.ini:
```sh
my_app_folder = /home/ubuntu/CTFd/app/CTFd
my_user = ubuntu

file = wsgi.py #This is the name of the wsgi file provided by CTFd
```
Now let's create a log file for uWSGI and move uwsgi.ini to the main CTFd directory.
```sh
$ sudo mkdir -p /var/log/uwsgi
$ sudo chown -R ubuntu:ubuntu /var/log/uwsgi
$ sudo mv uwsgi.ini /home/ubuntu/CTFd/app/CTFd
```
Time to go to the NGINX config file.
```sh
$ cd /etc/nginx/sites-available
$ sudo nano default
```
Now we're going to configure NGINX so that CTFd will be available on www.domain.com/ctf/. I changed the server name to our domain, and I inserted a new location block (underneath the existing location / {} block) within the default NGINX configuration file: 
```sh
server_name domain.com www.domain.com

location /ctf/ {
    include uwsgi_params;
    uwsgi_pass unix:/home/ubuntu/CTFd/app/CTFd/production_flask.sock;
    uwsgi_param uwsgi.py /ctf;
}
```
Now let's navigate to CTFd's config.py file to specify the subdirectory path, and while we're there we can set the database to point to the MySQL superuser we created earlier.
```sh
$ cd ~/CTFd/app/CTFd/CTFd #I know, the directories could be named better. Feel free to do so.
$ sudo nano config.py
```
Here we're going to make changes to the lines specifying the DATABASE_URL and APPLICATION_ROOT.
```python
DATABASE_URL = os.getenv("DATABASE_URL") or "mysql+pymysql://<MySQL_Superuser_Username>:<MySQL_Superuser_Password>@localhost/ctfd".format(
    os.path.dirname(os.path.abspath(__file__))
)

#APPLICATION_ROOT is towards the bottom of the file
APPLICATION_ROOT = os.getenv("APPLICATION_ROOT") or "/ctf"
```
Before I forget, let's amend one line in the wsgi.py file so that when the app runs it doesn't try to on localhost.
```sh
$ cd ~/CTFd/app/CTFd
$ sudo nano wsgi.py
```
Inside wsgi.py, the have the app.run line look like this:
```python
    app.run(debug=True, threaded=True)
```
We're almost done! Let's move source-nerd's production_flask.service file to the correct place and amend some lines.
```sh
$ cd ~/CTFd
$ sudo mv production_flask.service /etc/systemd/system
$ cd /etc/systemd/system
$ sudo nano production_flask.service
```
Inside production_flask.service:
```sh
[Unit]
Description=uWSGI instance to serve CTFd service

[Service]
User=ubuntu
WorkingDirectory=/home/ubuntu/CTFd/app/CTFd
Environment="PATH=/home/ubuntu/CTFd/app/venv/bin"
ExecStart=/home/ubuntu/CTFd/app/venv/bin/uwsgi --ini /home/ubuntu/CTFd/app/CTFd/uwsgi.ini
Restart=on-failure

[Install]
WantedBy=multi-user.target
```
Now let's start our services (and, if you see enable, let it start when the server boots).
```sh
$ sudo systemctl start production_flask.service
$ sudo systemctl enable production_flask.service
$ sudo systemctl restart mysql
$ sudo systemctl restart nginx
```
Check the status and resolve any errors you see using the following:
```sh
$ sudo systemctl status production_flask.service
$ sudo systemctl status mysql
$ sudo systemctl status nginx
```
You can also check your logs. I found the uwsgi logs that we created earlier provided the most useful troubleshooting information.
> Note: if you already went through the setup using CTFd's SQLite, it may have created a database file for you in ~/CTFd/app/CTFd/CTFd. If you see errors to the extent of "database /ctfd already exists", simply navigate to said directory and sudo rm -r ctfd.db.

##### Conclusion & Reflection
DISCLAIMER: I did not go into securing anything in this guide. I didn't walk through SSL certifications, firewall optimization, or anything of the sort. Using this guide alone will leave you with a potentially vulnerable system. The goal of this guide was to show how to get it to work in a test environment. Secure everything before gonig live!

Also, I plan on updating this guide to include redis caching in the future. Keep an eye out!

If everything went well, you should be able to see your unique homepage at your main domain, and see the CTFd page at www.domain.com/ctf! Happy hacking!



This guide was created by DL02926

Last updated 6/21/2019
