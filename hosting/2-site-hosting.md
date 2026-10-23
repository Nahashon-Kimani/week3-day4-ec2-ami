# 2. Hosting the DGcom Website on Apache

This guide installs the Apache web server on the Amazon Linux 2023 instance from [launching-ec2.md](launching-ec2.md), deploys the DGcom static site from this repository, and tests it.

> **Previous:** [launching-ec2.md](launching-ec2.md) · **Next:** [create-ami.md](create-ami.md)

> **Apache2 vs httpd:** On Debian/Ubuntu the package and service are called `apache2`. On Amazon Linux (and RHEL/Fedora) the same Apache HTTP Server is called **`httpd`**. Every `httpd` command below is "Apache2".

---

## What gets deployed

This repo is a pure static site (HTML, CSS, JS and images – no build step, no database):

```
index.html  about.html  service.html  project.html  feature.html
team.html   testimonial.html  contact.html  404.html
css/  js/  img/  lib/
```

Apache will serve these files from its document root: **`/var/www/html`**.

---

## Step 1 – Connect and update the instance

```bash
ssh -i dgcom-key.pem ec2-user@<PUBLIC_IP>
sudo dnf update -y
```

> Amazon Linux 2023 uses **`dnf`** (not `yum` or `apt`).

## Step 2 – Install Apache (httpd) and Git

```bash
sudo dnf install -y httpd git
httpd -v
```

## Step 3 – Start Apache and enable it on boot

```bash
sudo systemctl start httpd
sudo systemctl enable httpd
sudo systemctl status httpd --no-pager
```

You should see `Active: active (running)`. Press `q` if the output pauses.

> Enabling the service on boot matters: the AMI you create later will start Apache automatically on any new instance launched from it.

## Step 4 – Quick test with the default page

Open `http://<PUBLIC_IP>` in your browser. You should see the Apache **"It works!"** test page.

> Use **`http://`**, not `https://`. Port 443 is open in the security group, but no TLS certificate is configured yet, so HTTPS will fail.

## Step 5 – Deploy the DGcom site

Clone the repository and copy the site files into the document root:

```bash
cd /tmp
git clone https://github.com/Nahashon-Kimani/week3-day4-ec2-ami.git dgcom
sudo cp -r /tmp/dgcom/* /var/www/html/
```

The `hosting/` folder (these guides) and the repo's text files will also be copied. That's harmless, but you can remove them if you like:

```bash
sudo rm -rf /var/www/html/hosting /var/www/html/READ-ME.txt /var/www/html/LICENSE.txt
```

### Alternative – upload from your own machine (no Git)

Run this **on your laptop**, from the repo root:

```bash
scp -i dgcom-key.pem -r index.html about.html contact.html feature.html project.html \
  service.html team.html testimonial.html 404.html DGcom.jpg css js img lib \
  ec2-user@<PUBLIC_IP>:/tmp/
```

Then **on the instance**:

```bash
cd /tmp && sudo cp -r *.html DGcom.jpg css js img lib /var/www/html/
```

## Step 6 – Set ownership and permissions

```bash
sudo chown -R apache:apache /var/www/html
sudo find /var/www/html -type d -exec chmod 755 {} \;
sudo find /var/www/html -type f -exec chmod 644 {} \;
```

## Step 7 – (Optional) Use the site's custom 404 page

The repo ships a `404.html`. Tell Apache to use it:

```bash
echo 'ErrorDocument 404 /404.html' | sudo tee /etc/httpd/conf.d/dgcom.conf
sudo apachectl configtest      # should print: Syntax OK
sudo systemctl restart httpd
```

---

## Step 8 – Test the website

### From the instance

```bash
curl -I http://localhost
# Expect: HTTP/1.1 200 OK

curl -s http://localhost | grep -o '<title>.*</title>'
# Expect: <title>DGcom - Web Design Agency HTML Template</title>

curl -s -o /dev/null -w '%{http_code}\n' http://localhost/does-not-exist
# Expect: 404
```

### From your laptop

```bash
curl -I http://<PUBLIC_IP>
```

### In a browser

1. Open `http://<PUBLIC_IP>` – the DGcom home page with the image carousel should load.
2. Click through **About, Services, Projects, Contact** in the navbar – every page should load with styling and images.
3. Open `http://<PUBLIC_IP>/anything` – the DGcom 404 page should appear (if you did Step 7).

### Check the logs

```bash
sudo tail -n 20 /var/log/httpd/access_log
sudo tail -n 20 /var/log/httpd/error_log
```

You should see your own browser requests in `access_log`.

---

## Troubleshooting

| Symptom | Likely cause / fix |
|---------|--------------------|
| Browser spins then times out | Port 80 not open in the security group, or you used `https://`. |
| "It works!" page still shows | Files not copied to `/var/www/html`, or browser cache – hard refresh (`Ctrl+F5`). |
| Page loads without styling/images | `css/`, `js/`, `img/` or `lib/` folders not copied, or wrong permissions (re-run Step 6). |
| `403 Forbidden` | Permissions/ownership on `/var/www/html` – re-run Step 6. |
| `curl: (7) Failed to connect` on the instance | Apache not running: `sudo systemctl start httpd` and check `error_log`. |

---

## ✅ Check your progress

Share your answers and screenshots with your instructor/team.

1. Paste the output of:
   ```bash
   sudo systemctl is-active httpd && sudo systemctl is-enabled httpd
   ```
   *(Expected: `active` and `enabled`)*
2. Screenshot the DGcom home page loaded at `http://<PUBLIC_IP>` with the browser address bar visible.
3. What HTTP status code does `curl -I http://<PUBLIC_IP>` return? Paste the first line.
4. **Think about it:** On Ubuntu you would run `sudo apt install apache2`. What is the equivalent package name and install command on Amazon Linux 2023, and where is the document root?
5. **Think about it:** Port 443 is open, but `https://<PUBLIC_IP>` doesn't work. What is missing?
