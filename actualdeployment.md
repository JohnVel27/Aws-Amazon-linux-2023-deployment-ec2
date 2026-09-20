 # Django Deployment on Amazon Linux 2023

This guide deploys the practice project from [django-ec2-deploy-practice](https://github.com/JohnVel27/django-ec2-deploy-practice) to an Amazon EC2 instance using Gunicorn and Nginx.

Run the commands below on the EC2 instance unless a step says **AWS Console** or **local computer**.

## 1. Connect to EC2

From your local computer, replace the key file and public DNS name with your values:

```bash
chmod 400 my-key.pem
ssh -i my-key.pem ec2-user@YOUR_EC2_PUBLIC_DNS
```

Amazon Linux 2023 uses `ec2-user` by default.

## 2. Update Amazon Linux

```bash
sudo dnf update -y
sudo dnf check-update
```

`dnf check-update` returns exit code `100` when updates are available. That is expected and is not an installation failure.

## 3. Install Git, Python, and Nginx

```bash
sudo dnf install git python3 python3-pip nginx -y

git --version
python3 --version
pip3 --version
nginx -v
```

## 4. Clone the Django project

```bash
git clone https://github.com/JohnVel27/django-ec2-deploy-practice.git
ls
```

Enter the cloned directory. The repository is expected to create `django-on-ec2`:

```bash
cd django-on-ec2
ls
```

You should see `manage.py` and, if the project uses dependencies, `requirements.txt`.

If the directory name differs, use the directory printed by `ls`.

## 5. Create and activate a virtual environment

```bash
python3 -m venv venv
source venv/bin/activate
```

The prompt should now begin with `(venv)`. Activate it again after every new SSH session:

```bash
cd ~/django-on-ec2
source venv/bin/activate
```

## 6. Install Django dependencies

If `requirements.txt` exists, run:

```bash
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
python -m pip list
```

If there is no requirements file, install Django directly:

```bash
python -m pip install Django
```

Install Gunicorn in the same virtual environment:

```bash
python -m pip install gunicorn
```

## 7. Check and prepare Django

```bash
python manage.py check
python manage.py makemigrations
python manage.py migrate
```

Create an administrator account if the application needs the Django admin site:

```bash
python manage.py createsuperuser
```

## 8. Test Django locally

For a temporary development test, run:

```bash
python manage.py runserver 0.0.0.0:8000
```

In another terminal, open `http://YOUR_EC2_PUBLIC_IPV4:8000/`. Stop the development server with `Ctrl+C`.

In the EC2 security group, allow TCP port `8000` from **My IP** for testing. Do not expose port `8000` publicly when Nginx will be the public entry point.

## 9. Find the WSGI module

From the directory containing `manage.py`, run:

```bash
find . -name wsgi.py -print
```

If the result is `./your_project/wsgi.py`, the Gunicorn application name is:

```text
your_project.wsgi:application
```

Replace `your_project` in the commands below with the actual Django project package name.

Test Gunicorn manually before creating the service:

```bash
gunicorn --bind 127.0.0.1:8000 your_project.wsgi:application
```

Stop it with `Ctrl+C` after confirming it starts without an import error.

## 10. Create the Gunicorn systemd service

Create the service file:

```bash
sudo nano /etc/systemd/system/gunicorn.service
```

Paste the following and replace `your_project` if necessary:

```ini
[Unit]
Description=Gunicorn daemon for Django
After=network.target

[Service]
User=ec2-user
Group=ec2-user
WorkingDirectory=/home/ec2-user/django-on-ec2
ExecStart=/home/ec2-user/django-on-ec2/venv/bin/gunicorn \
	--workers 3 \
	--bind 127.0.0.1:8000 \
	your_project.wsgi:application

[Install]
WantedBy=multi-user.target
```

Save and exit Nano with `Ctrl+O`, `Enter`, then `Ctrl+X`.

Reload systemd, enable Gunicorn at boot, and start it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable gunicorn
sudo systemctl start gunicorn
sudo systemctl status gunicorn
```

The status should show `Active: active (running)`. If it fails, inspect the logs:

```bash
sudo journalctl -u gunicorn -n 100 --no-pager
```

## 11. Configure Nginx

Create an Nginx server configuration:

```bash
sudo nano /etc/nginx/conf.d/django.conf
```

Start with this configuration. Replace `YOUR_EC2_PUBLIC_IPV4` with the instance's public IPv4 address:

```nginx
server {
	listen 80;
	server_name YOUR_EC2_PUBLIC_IPV4;

	location / {
		proxy_pass http://127.0.0.1:8000;
		proxy_set_header Host $host;
		proxy_set_header X-Real-IP $remote_addr;
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		proxy_set_header X-Forwarded-Proto $scheme;
	}
}
```

Test the configuration before starting Nginx:

```bash
sudo nginx -t
```

The output should include `syntax is ok` and `test is successful`.

Start Nginx and enable it at boot:

```bash
sudo systemctl start nginx
sudo systemctl enable nginx
sudo systemctl status nginx
```

Open `http://YOUR_EC2_PUBLIC_IPV4/` in a browser. Port `80` is now the public application port.

## 12. Configure Django static files

In `settings.py`, make sure the project has settings similar to:

```python
STATIC_URL = "/static/"
STATIC_ROOT = BASE_DIR / "staticfiles"
```

Collect static files from the activated virtual environment:

```bash
python manage.py collectstatic
```

Edit the Nginx configuration:

```bash
sudo nano /etc/nginx/conf.d/django.conf
```

Add the `/static/` location before the general `/` location:

```nginx
server {
	listen 80;
	server_name YOUR_EC2_PUBLIC_IPV4;

	location /static/ {
		alias /home/ec2-user/django-on-ec2/staticfiles/;
	}

	location / {
		proxy_pass http://127.0.0.1:8000;
		proxy_set_header Host $host;
		proxy_set_header X-Real-IP $remote_addr;
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		proxy_set_header X-Forwarded-Proto $scheme;
	}
}
```

Test and reload Nginx:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

## 13. Configure the EC2 security group

In the AWS Console, add these inbound rules to the instance's security group:

| Type | Port | Source |
| --- | ---: | --- |
| SSH | 22 | Your IP address |
| HTTP | 80 | `0.0.0.0/0` |

Only add port `8000` temporarily for direct development-server testing. Remove it when using Nginx.

## 14. Final checks

```bash
sudo systemctl status gunicorn
sudo systemctl status nginx
curl http://127.0.0.1:8000/
curl http://YOUR_EC2_PUBLIC_IPV4/
```

Both services should be running, and the final `curl` should return the Django response.

## Troubleshooting

### Gunicorn cannot import the application

Confirm the WSGI path and service directory:

```bash
find . -name wsgi.py -print
pwd
sudo journalctl -u gunicorn -n 100 --no-pager
```

The `WorkingDirectory`, virtual-environment path, and `your_project.wsgi:application` value in the service must match the project.

### Django reports `DisallowedHost`

Add the EC2 public IP address to `ALLOWED_HOSTS` in `settings.py`:

```python
ALLOWED_HOSTS = ["YOUR_EC2_PUBLIC_IPV4", "localhost", "127.0.0.1"]
```

Avoid `ALLOWED_HOSTS = ["*"]` in production.

### The browser cannot connect

Check that the instance is running, the public IPv4 address is current, the security group allows HTTP on port `80`, and both services are active:

```bash
sudo systemctl status gunicorn
sudo systemctl status nginx
sudo nginx -t
```

### Nginx returns `502 Bad Gateway`

Gunicorn is usually stopped or listening on a different address. Check its logs and confirm that it binds to `127.0.0.1:8000`:

```bash
sudo journalctl -u gunicorn -n 100 --no-pager
sudo ss -ltnp | grep 8000
```
