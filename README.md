# flask-deployment-steps
Step by step on how to deploy a flask server on ubuntu 20.04

## Setting up Nginx
To install and cofigure Nginx on your server, follow the following instructions.

First, you need to update your server by running

```bash
sudo apt-get update
```

To install Nginx, run

```bash
sudo apt-get install nginx 
```
### Configuring Nginx
Create a new file for the server configuration.

```bash
sudo nano /etc/nginx/sites-available/test-server.conf
```

test-server.conf:

```bash
server {
	listen 80;
	real_ip_header X-Forwarded-For;
	set_real_ip_from 127.0.0.1;
	server_name localhost;

	location / {
		include uwsgi_params;
		uwsgi_pass unix:/var/www/html/test-server/socket.sock;
		uwsgi_modifier1 30;
	}

	error_page 404 /404.html;
	location = 404.html {
		root /usr/share/nginx/html;
	}

	error_page 500 502  503 504 50x.html;
	location = /50x.html {
		root /usr/share/nginx/html;
	}
}
``` 

After editing the test-server.conf

```bash
sudo ln -s /etc/nginx/sites-available/test-server.conf /etc/nginx/sites-enabled/
```

Got to the directory, clone the app and install dependencies. Run the following commands one by one in that order.

```bash
cd /var/www/html/test-server
git clone https://github.com/path/to/repo.
mkdir log
sudo apt-get install python-pip python3-dev libpq-dev uwsgi
pip3 install virtualenv
python3 -m venv venv
source venv/bin/activate
pip3 install -r requirements.txt
```

Test the server with the following command:

```bash
uwsgi --http-socket :5000 --plugin python3 --module wsgi:app
```

Now let's create an **app.ini** file:
```bash
[uwsgi]
chdir = /var/www/html/test-server/
module = wsgi:app

processes = 4
threads = 2
virtualenv = /var/www/html/test-server/venv

master = true
socket = socket.sock
chmod-socket = 666
vacuum = true

die-on-term = true

logto = /var/www/html/test-server/log/%n.log
```
Now we can start the server with the following command:
 ```bash
uwsgi app.ini
 ```




### Create Ubuntu service
After verifying that the app runs normally the next step is to create an Ubuntu service.
Run the following command to create a service file.

```bash
$ sudo nano /etc/systemd/system/test_server.service 
```
Then paste the following changing the values:
```bash
[Unit]
Description=My custom flask app Service
After=network.target

[Service]
User=root

WorkingDirectory=/var/www/html/test-server
Environment="PATH=/var/www/html/test-server/venv/bin"
ExecStart=/var/www/html/test-server/venv/bin/uwsgi --ini app.ini
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```
-   `User`: Tells our service to run our app as the Ubuntu root user.
-   `WorkingDirectory`: The working directory that we'll be serving our app from.
-   `Environment`: Our project's Python virtual environment.
-   `ExecStart`: This is the most important part of our configuration: a shell command to actually start our application. Each time we "start" or "restart" our service, this is the command being executed.
-   `Restart`  &  `RestartSec`: These two values are telling our service to check if our app is running every 10 seconds. If the app happens to be down, our service will restart the app, hence the  **on-failure**  value for  **Restart**.

Now we can check start check the status  of the service: 
```shell
service myapp start
service myapp status
```
