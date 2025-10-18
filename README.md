# Django Project Deployment on Ubuntu VPS with Gunicorn and Nginx (IP: 72.60.219.2)

This guide covers deploying your Django project `prop` from GitHub to an Ubuntu VPS.

---

## 1. SSH into the VPS

```bash
ssh root@72.60.219.2
```

## 2. Update Server Packages

```bash
sudo apt update && sudo apt upgrade -y
```

## 3. Install Dependencies

```bash
sudo apt install python3 python3-venv python3-pip nginx git build-essential pkg-config python3-dev default-libmysqlclient-dev -y
```

## 4. Navigate to Web Directory

```bash
cd /var/www
sudo mkdir prop
cd prop
```

## 5. Clone Your GitHub Repository

```bash
git clone https://github.com/Anurag7828/prop.git .
```

## 6. Create Virtual Environment

```bash
python3 -m venv venv
source venv/bin/activate
```

## 7. Install Python Dependencies

```bash
pip install --upgrade pip
pip install -r requirements.txt
pip install gunicorn mysqlclient
```

## 8. Update Django `settings.py`

* Set `ALLOWED_HOSTS = ['72.60.219.2']`
* Set database credentials as per server
* Configure `STATIC_ROOT = os.path.join(BASE_DIR, 'staticfiles')`
* Configure `MEDIA_ROOT = os.path.join(BASE_DIR, 'uploads')`

## 9. Apply Migrations and Collect Static Files

```bash
python manage.py migrate
python manage.py collectstatic --noinput
```

## 10. Test Gunicorn

```bash
gunicorn property.wsgi:application --chdir /var/www/prop --bind 0.0.0.0:8000
```

* Open `http://72.60.219.2:8000` to verify
* Stop server with `Ctrl+C`

## 11. Create Gunicorn Systemd Service

```bash
sudo vim /etc/systemd/system/property.service
```

Paste:

```
[Unit]
Description=Gunicorn daemon for Django prop project
After=network.target

[Service]
User=root
Group=www-data
WorkingDirectory=/var/www/prop
ExecStart=/var/www/prop/venv/bin/gunicorn --workers 3 --chdir /var/www/prop --bind unix:/var/www/prop/prop.sock property.wsgi:application

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl start property
sudo systemctl enable property
sudo systemctl status property
```

## 12. Configure Nginx

```bash
sudo vim /etc/nginx/sites-available/prop
```

Paste:

```
server {
    listen 80;
    server_name 72.60.219.2;

    location /static/ {
        root /var/www/prop;
    }

    location /uploads/ {
        root /var/www/prop;
    }

    location / {
        include proxy_params;
        proxy_pass http://unix:/var/www/prop/prop.sock;
    }
}
```

Enable site and restart:

```bash
sudo ln -s /etc/nginx/sites-available/prop /etc/nginx/sites-enabled
sudo nginx -t
sudo systemctl restart nginx
sudo systemctl enable nginx
```

## 13. Firewall

* Open HTTP port if needed: `sudo ufw allow 80`

## 14. Pull Latest Code (after pushing to GitHub)

```bash
cd /var/www/prop
source venv/bin/activate
git pull origin main
pip install -r requirements.txt
python manage.py migrate
python manage.py collectstatic --noinput
sudo systemctl restart property
sudo systemctl restart nginx
```

Now your Django project is live at `http://72.60.219.2/` via Nginx → Gunicorn → Django.
