# Django Project Deployment Guide - Ubuntu VPS par Gunicorn aur Nginx ke saath

**Server IP:** 72.60.219.2  
**Domain:** propmatez.shop

---

## 1. VPS mein SSH se Login Karo

```bash
ssh root@72.60.219.2
```

---

## 2. Server Packages Update Karo

```bash
sudo apt update && sudo apt upgrade -y
```

---

## 3. Dependencies Install Karo

```bash
sudo apt install python3 python3-venv python3-pip nginx git build-essential pkg-config python3-dev default-libmysqlclient-dev certbot python3-certbot-nginx -y
```

---

## 4. Web Directory mein Jao

```bash
cd /var/www
sudo mkdir prop
cd prop
```

---

## 5. GitHub Repository Clone Karo

```bash
git clone https://github.com/Anurag7828/prop.git .
```

---

## 6. Virtual Environment Banao

```bash
python3 -m venv venv
source venv/bin/activate
```

---

## 7. Python Dependencies Install Karo

```bash
pip install --upgrade pip
pip install -r requirements.txt
pip install gunicorn mysqlclient
```

---

## 8. Django `settings.py` Update Karo

* `ALLOWED_HOSTS = ['72.60.219.2', 'propmatez.shop', 'www.propmatez.shop']`
* Database credentials server ke hisaab se set karo
* `STATIC_ROOT = os.path.join(BASE_DIR, 'staticfiles')` configure karo
* `MEDIA_ROOT = os.path.join(BASE_DIR, 'uploads')` configure karo

---

## 9. Migrations Apply Karo aur Static Files Collect Karo

```bash
python manage.py migrate
python manage.py collectstatic --noinput
```

---

## 10. Gunicorn Test Karo

```bash
gunicorn property.wsgi:application --chdir /var/www/prop --bind 0.0.0.0:8000
```

* `http://72.60.219.2:8000` browser mein kholo aur verify karo
* `Ctrl+C` se server band karo

---

## 11. Gunicorn Systemd Service Banao

```bash
sudo vim /etc/systemd/system/property.service
```

Yeh paste karo:

```ini
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

Enable aur start karo:

```bash
sudo systemctl daemon-reload
sudo systemctl start property
sudo systemctl enable property
sudo systemctl status property
```

---

## 12. Nginx Configure Karo (HTTP - Temporary)

```bash
sudo vim /etc/nginx/sites-available/prop
```

Yeh paste karo:

```nginx
server {
    listen 80;
    server_name 72.60.219.2 propmatez.shop www.propmatez.shop;
    client_max_body_size 50M;

    location = /favicon.ico { 
        access_log off; 
        log_not_found off; 
    }

    # ACME challenge for SSL
    location /.well-known/acme-challenge/ {
        root /var/www/letsencrypt;
        try_files $uri =404;
    }

    location /static/ {
        alias /var/www/prop/staticfiles/;
    }

    location /uploads/ {
        alias /var/www/prop/uploads/;
        autoindex on;
    }

    location / {
        include proxy_params;
        proxy_pass http://unix:/var/www/prop/prop.sock;
    }
}
```

Site enable karo aur restart karo:

```bash
sudo ln -s /etc/nginx/sites-available/prop /etc/nginx/sites-enabled
sudo nginx -t
sudo systemctl restart nginx
sudo systemctl enable nginx
```

---

## 13. SSL Certificate Install Karo (HTTPS ke liye)

### Directory banao SSL challenge ke liye:

```bash
sudo mkdir -p /var/www/letsencrypt/.well-known/acme-challenge
sudo chmod -R 755 /var/www/letsencrypt
sudo chown -R www-data:www-data /var/www/letsencrypt
```

### Certbot chalao:

```bash
sudo certbot --nginx -d propmatez.shop -d www.propmatez.shop
```

**Instructions follow karo:**
- Email address dalo
- Terms of Service accept karo (Y)
- HTTPS redirect chahiye? Yes (2)

Certificate install ho jayega aur Nginx automatically configure ho jayega!

---

## 14. Final Nginx Configuration (SSL ke baad - Auto ho jayega)

Certbot automatically yeh configuration bana dega:

```nginx
# HTTP to HTTPS redirect
server {
    listen 80;
    server_name propmatez.shop www.propmatez.shop;
    return 301 https://$server_name$request_uri;
}

# HTTPS server
server {
    listen 443 ssl http2;
    server_name propmatez.shop www.propmatez.shop;
    client_max_body_size 50M;

    ssl_certificate /etc/letsencrypt/live/propmatez.shop/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/propmatez.shop/privkey.pem;

    location = /favicon.ico { 
        access_log off; 
        log_not_found off; 
    }

    location /static/ {
        alias /var/www/prop/staticfiles/;
    }

    location /uploads/ {
        alias /var/www/prop/uploads/;
        autoindex on;
    }

    location / {
        include proxy_params;
        proxy_pass http://unix:/var/www/prop/prop.sock;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

---

## 15. Firewall Setup (Agar zaroorat ho)

```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 22/tcp
sudo ufw enable
```

---

## 16. Latest Code Pull Karne Ka Tarika (GitHub se update ke baad)

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

---

## 17. SSL Certificate Auto-Renewal Check

Certificate 90 din mein expire hota hai. Auto-renewal test karo:

```bash
sudo certbot renew --dry-run
```

Certbot automatically renew kar dega. Check karne ke liye:

```bash
sudo systemctl status certbot.timer
```

---

## 18. Useful Commands

### Service Status Check:
```bash
sudo systemctl status property
sudo systemctl status nginx
```

### Logs Dekhne ke liye:
```bash
sudo journalctl -u property -f
sudo tail -f /var/log/nginx/error.log
```

### Service Restart:
```bash
sudo systemctl restart property
sudo systemctl restart nginx
```

### Certificate Info:
```bash
sudo certbot certificates
```

---

## ✅ Ab Aapka Django Project Live Hai!

- **HTTP:** `http://propmatez.shop` (automatically HTTPS par redirect ho jayega)
- **HTTPS:** `https://propmatez.shop` 🔒

---

**Done! Aapki site ab secure HTTPS ke saath live hai! 🎉**

### Troubleshooting Tips

**Agar site nahi chal rahi:**
1. Check karo ki DNS properly set hai: `dig propmatez.shop +short`
2. Gunicorn service check karo: `sudo systemctl status property`
3. Nginx error logs dekho: `sudo tail -f /var/log/nginx/error.log`
4. Socket file check karo: `ls -l /var/www/prop/prop.sock`

**Agar SSL error aa rahi hai:**
1. Certificate check karo: `sudo certbot certificates`
2. Nginx configuration test karo: `sudo nginx -t`
3. Certbot logs dekho: `sudo cat /var/log/letsencrypt/letsencrypt.log`
