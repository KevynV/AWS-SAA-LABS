# WordPress + MariaDB on EC2 Lab

## Goal
Deploy WordPress on an Amazon EC2 instance using:
- Apache (`httpd`)
- PHP
- MariaDB

This lab involved manually installing and configuring the web server, database server, and WordPress, then troubleshooting a database connection issue during the final browser setup.

---

## Step 1 - Configure Authentication Variables

These variables are used throughout the setup:

```bash
DBName='a4lwordpress'
DBUser='a4lwordpress'
DBPassword='your_secure_password_here'
DBRootPassword='your_secure_password_here'
```

To verify a variable was set:

```bash
echo $DBName
```

---

## Step 2 - Install Required Software

Install Apache, PHP, MariaDB, and supporting packages:

```bash
sudo dnf install wget php-mysqlnd httpd php-fpm php-mysqli mariadb105-server php-json php php-devel -y
```

---

## Step 3 - Enable and Start Web and Database Services

Enable both services to start automatically on boot, then start them now:

```bash
sudo systemctl enable httpd
sudo systemctl enable mariadb
sudo systemctl start httpd
sudo systemctl start mariadb
```

---

## Step 4 - Set the MariaDB Root Password

```bash
sudo mysqladmin -u root password $DBRootPassword
```

---

## Step 5 - Install WordPress

Download and extract WordPress into the web root:

```bash
sudo wget http://wordpress.org/latest.tar.gz -P /var/www/html
cd /var/www/html
sudo tar -zxvf latest.tar.gz
sudo cp -rvf wordpress/* .
```

Clean up the extracted directory and archive:

```bash
sudo rm -R wordpress
sudo rm latest.tar.gz
```

---

## Step 6 - Configure WordPress

Create the main WordPress configuration file from the sample:

```bash
sudo cp ./wp-config-sample.php ./wp-config.php
```

Replace the placeholders with the variables created earlier:

```bash
sudo sed -i "s/'database_name_here'/'$DBName'/g" wp-config.php
sudo sed -i "s/'username_here'/'$DBUser'/g" wp-config.php
sudo sed -i "s/'password_here'/'$DBPassword'/g" wp-config.php
```

Give Apache ownership of the web root:

```bash
sudo chown apache:apache * -R
```

Check that the config file was updated correctly:

```bash
sudo nano wp-config.php
```

Expected database section:

```php
define( 'DB_NAME', 'a4lwordpress' );
define( 'DB_USER', 'a4lwordpress' );
define( 'DB_PASSWORD', 'your_secure_password_here' );
define( 'DB_HOST', 'localhost' );
```

---

## Step 7 - Create the WordPress Database and User

Build a SQL setup script and run it:

```bash
echo "CREATE DATABASE $DBName;" >> /tmp/db.setup
echo "CREATE USER '$DBUser'@'localhost' IDENTIFIED BY '$DBPassword';" >> /tmp/db.setup
echo "GRANT ALL ON $DBName.* TO '$DBUser'@'localhost';" >> /tmp/db.setup
echo "FLUSH PRIVILEGES;" >> /tmp/db.setup

mysql -u root --password=$DBRootPassword < /tmp/db.setup
```

Delete the temporary SQL file after it runs:

```bash
sudo rm /tmp/db.setup
```

---

## MariaDB Shell Notes

While testing MariaDB manually, I entered the MariaDB monitor using:

```bash
mysql -u root --password=$DBRootPassword
```

Once inside the MariaDB shell:
- `deactivate` does **not** work there because it is a shell/Python virtual environment command, not a MariaDB command
- To cancel an unfinished SQL statement, use:

```sql
\c
```

- To exit MariaDB, use one of the following:

```sql
exit;
quit;
```

Or press:

```bash
Ctrl + D
```

---

## Step 8 - Browse to the EC2 Public IP

Open a browser and go to:

```text
http://<your-ec2-public-ip>
```

This should begin the WordPress setup page.

---

## Troubleshooting

### Issue: "Error establishing a database connection"

At the final step, browsing to the EC2 public IP returned:

```text
Error establishing a database connection
```

### Initial Checks

I confirmed MariaDB was running:

```bash
sudo systemctl status mariadb --no-pager
```

I also confirmed WordPress files were present:

```bash
sudo ls /var/www/html
```

And checked the database settings in `wp-config.php`:

```bash
sudo grep DB_ /var/www/html/wp-config.php
```

The configuration showed:
- `DB_NAME = a4lwordpress`
- `DB_USER = a4lwordpress`
- `DB_HOST = localhost`

### Root Cause

The issue was that the WordPress database credentials in `wp-config.php` did not match the actual MariaDB user password.

Even though:
- Apache was working
- MariaDB was running
- WordPress files were installed

WordPress could not connect because the database user authentication was failing.

### How I Verified the Problem

I tested the WordPress database login manually:

```bash
mysql -u a4lwordpress -p -h localhost a4lwordpress
```

This returned:

```text
ERROR 1045 (28000): Access denied for user 'a4lwordpress'@'localhost' (using password: YES)
```

That confirmed the issue was not the browser or Apache. It was the MariaDB user/password or permissions.

### Fix

I logged into MariaDB as root:

```bash
sudo mysql
```

Then I ran:

```sql
CREATE DATABASE IF NOT EXISTS a4lwordpress;
CREATE USER IF NOT EXISTS 'a4lwordpress'@'localhost' IDENTIFIED BY 'YOUR_PASSWORD_HERE';
ALTER USER 'a4lwordpress'@'localhost' IDENTIFIED BY 'a4leasy';
GRANT ALL PRIVILEGES ON a4lwordpress.* TO 'a4lwordpress'@'localhost';
FLUSH PRIVILEGES;
```

After that, I tested the login again:

```bash
mysql -u a4lwordpress -p -h localhost a4lwordpress
```

This time it worked.

### Final Step to Resolve WordPress

Even after MariaDB authentication worked manually, the browser still failed.

That meant `wp-config.php` was still using the old database password.

I updated this line in `/var/www/html/wp-config.php`:

```php
define( 'DB_PASSWORD', 'a4leasy' );
```

After that, WordPress loaded correctly in the browser and I was able to:
- complete the site setup
- create the admin account
- log into WordPress successfully

---

## What I Learned

- How to manually install WordPress on EC2
- How to install and manage Apache and MariaDB on Amazon Linux
- How to configure WordPress to connect to MariaDB
- How to troubleshoot database connection issues systematically
- The difference between the Linux shell and the MariaDB shell
- How to validate database access manually before assuming the web app is broken

---

## Validation

Successful validation included:

```bash
sudo systemctl status mariadb
mysql -u a4lwordpress -p -h localhost a4lwordpress
```

And finally:
- WordPress loaded in the browser
- account creation worked
- login to the WordPress dashboard succeeded
