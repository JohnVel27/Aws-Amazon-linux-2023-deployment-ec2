# Aws-Amazon-linux-2023-deployment-ec2

# Deploy a Django application on Amazon Linux 2023

These steps deploy the practice Django project to an Amazon EC2 instance for development and testing.

## 1. Connect to the EC2 instance

From your local computer, connect with SSH. Replace the key path, public DNS name, and username as needed:

```bash
chmod 400 my-key.pem
ssh -i my-key.pem ec2-user@YOUR_EC2_PUBLIC_DNS
```

Amazon Linux 2023 uses `ec2-user` by default.

## 2. Update the operating system

Run these commands on the EC2 instance:

```bash
sudo dnf update -y
sudo dnf check-update
```

`dnf check-update` may return exit code `100` when updates are available. That is normal; it does not mean the command failed.

## 3. Install Git and verify it

```bash
sudo dnf install git -y
git --version
```

## 4. Clone the project

```bash
git clone https://github.com/JohnVel27/django-ec2-deploy-practice.git
ls
```

If the clone created a directory named `django-on-ec2`, enter it:

```bash
cd django-on-ec2
```

If the directory has a different name, use the name shown by `ls` instead.

Confirm that the Django project files are present:

```bash
ls
```

You should see `manage.py`. If the repository contains a `requirements.txt` file, install the dependencies in the next step.

## 5. Install Python and create a virtual environment

```bash
sudo dnf install python3 python3-pip -y
python3 --version
python3 -m venv venv
source venv/bin/activate
```

After activation, the shell prompt should begin with `(venv)`. Activate the environment again whenever you reconnect to the instance:

```bash
cd django-on-ec2
source venv/bin/activate
```

## 6. Install project requirements

Run this only after confirming that `requirements.txt` exists:

```bash
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
python -m pip list
```

If the project does not include `requirements.txt`, install Django directly instead:

```bash
python -m pip install Django
```

## 7. Prepare the database

```bash
python3 manage.py makemigrations
python3 manage.py migrate
```

`makemigrations` creates migration files from model changes. Normally, those files should be committed to the project; `migrate` applies them to the database.

## 8. Create an administrator account

```bash
python3 manage.py createsuperuser
```

Follow the prompts to enter a username, email address, and password.

## 9. Allow traffic to port 8000

In the AWS console, open the EC2 instance's **Security Group** and add an inbound rule:

- **Type:** Custom TCP
- **Port:** `8000`
- **Source:** Your IP address for safer testing, or `0.0.0.0/0` only when public access is intentional

Do not use `0.0.0.0/0` for an application that should remain private.

## 10. Start Django for testing

```bash
python3 manage.py runserver 0.0.0.0:8000
```

Open the application in a browser:

```text
http://YOUR_EC2_PUBLIC_IPV4:8000/
```

The Django development server stays attached to the terminal. Press `Ctrl+C` to stop it.

## 11. Optional: run the server in the background

For a temporary testing session, use `nohup`:

```bash
nohup python3 manage.py runserver 0.0.0.0:8000 > django.log 2>&1 &
```

View the log with:

```bash
tail -f django.log
```

This is suitable for development and testing only. For production, use a production WSGI server such as Gunicorn behind a reverse proxy such as Nginx, with HTTPS and a proper process manager.

## Common fixes

### `git: command not found`

```bash
sudo dnf install git -y
git --version
```

### `python3 -m venv` fails

```bash
sudo dnf install python3 python3-pip -y
python3 -m venv venv
```

### `No such file or directory: manage.py`

You are not in the directory containing the Django project. Find it and change into it:

```bash
find . -name manage.py -print
cd PATH_TO_DIRECTORY_CONTAINING_MANAGE_PY
```

### The browser cannot connect

Check all of the following:

1. The server is running with `0.0.0.0:8000`, not only `127.0.0.1:8000`.
2. The EC2 security group allows inbound TCP traffic on port `8000`.
3. You are using the instance's current public IPv4 address.
4. The instance is running and has a reachable public address.

### Django rejects the host

For testing, add the EC2 public IP address to Django's `ALLOWED_HOSTS` setting. For example:

```python
ALLOWED_HOSTS = ["YOUR_EC2_PUBLIC_IPV4", "localhost", "127.0.0.1"]
```

Do not use `ALLOWED_HOSTS = ["*"]` as a production security configuration.
