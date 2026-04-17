# Bootstrapping WordPress on EC2

**AWS Solutions Architect — Lab Documentation**

---

## Overview

This lab demonstrates how to bootstrap a full WordPress installation on an EC2 instance using User Data scripts. It covers:

- Manual EC2 launch with inline User Data
- Verifying the bootstrap via the EC2 Instance Metadata Service (IMDSv2)
- Automating the same setup with CloudFormation

---

## Step 1 — Launch EC2 with User Data

Launch a new EC2 instance with the following configuration:

- **VPC:** a4l-vpc1
- **Subnet:** sn-web-A (public subnet in AZ A)
- Under **Advanced Details**, paste the User Data script below

> ⚠️ **Common Mistake:** The `#!/bin/bash -xe` shebang line at the top is required. Without it, EC2 doesn't know to execute the script as bash and the bootstrap silently fails.

### User Data Script

```bash
#!/bin/bash -xe

# STEP 1 - Set password & DB Variables
DBName='a4lwordpress'
DBUser='a4lwordpress'
DBPassword='4n1m4l$4L1f3'
DBRootPassword='4n1m4l$4L1f3'

# STEP 2 - Install system software - including Web and DB
dnf install wget php-mysqlnd httpd php-fpm php-mysqli mariadb105-server php-json php php-devel cowsay -y

# STEP 3 - Web and DB Servers Online - and set to startup
systemctl enable httpd
systemctl enable mariadb
systemctl start httpd
systemctl start mariadb

# STEP 4 - Set Mariadb Root Password
mysqladmin -u root password $DBRootPassword

# STEP 5 - Install Wordpress
wget http://wordpress.org/latest.tar.gz -P /var/www/html
cd /var/www/html
tar -zxvf latest.tar.gz
cp -rvf wordpress/* .
rm -R wordpress
rm latest.tar.gz

# STEP 6 - Configure Wordpress
cp ./wp-config-sample.php ./wp-config.php
sed -i "s/'database_name_here'/'$DBName'/g" wp-config.php
sed -i "s/'username_here'/'$DBUser'/g" wp-config.php
sed -i "s/'password_here'/'$DBPassword'/g" wp-config.php

# Step 6a - Permissions
usermod -a -G apache ec2-user
chown -R ec2-user:apache /var/www
chmod 2775 /var/www
find /var/www -type d -exec chmod 2775 {} \;
find /var/www -type f -exec chmod 0664 {} \;

# STEP 7 - Create Wordpress DB
echo "CREATE DATABASE $DBName;" >> /tmp/db.setup
echo "CREATE USER '$DBUser'@'localhost' IDENTIFIED BY '$DBPassword';" >> /tmp/db.setup
echo "GRANT ALL ON $DBName.* TO '$DBUser'@'localhost';" >> /tmp/db.setup
echo "FLUSH PRIVILEGES;" >> /tmp/db.setup
mysql -u root --password=$DBRootPassword < /tmp/db.setup
sudo rm /tmp/db.setup

# STEP 8 - COWSAY
echo "#!/bin/sh" > /etc/update-motd.d/40-cow
echo 'cowsay "Amazon Linux 2023 AMI - Animals4Life"' >> /etc/update-motd.d/40-cow
chmod 755 /etc/update-motd.d/40-cow
update-motd
```

---

## Step 2 — Verify Bootstrap via Instance Metadata (IMDSv2)

After the instance launches, SSH in and use the EC2 Instance Metadata Service to confirm the User Data was received. EC2 enforces IMDSv2, which requires a session token before any metadata can be accessed.

### 2a — Get a Session Token

```bash
TOKEN=`curl -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600"`
```

> ⚠️ **Common Mistake:** Running the metadata `curl` command before setting `$TOKEN` will return `401 Unauthorized` because the variable is empty. Always fetch the token first.

### 2b — Browse Available Metadata

```bash
curl -H "X-aws-ec2-metadata-token: $TOKEN" -v http://169.254.169.254/latest/meta-data/
```

### 2c — Confirm User Data Was Received

```bash
curl -H "X-aws-ec2-metadata-token: $TOKEN" -v http://169.254.169.254/latest/user-data/
```

A successful response prints the full User Data script back, confirming EC2 received and stored it at launch.

---

## Step 3 — Review Bootstrap Logs

To see what ran during bootstrap and check for errors:

```bash
cd /var/log
ls -la
sudo cat cloud-init-output.log
```

This log captures every command executed by the User Data script along with stdout/stderr. It's the primary debugging tool when a bootstrap doesn't behave as expected.

---

## CloudFormation Equivalent

The same bootstrap can be automated in a CloudFormation template using `Fn::Base64` and `!Sub` so that parameter values like `${DBName}` are substituted at stack creation time.

```yaml
UserData:
  Fn::Base64: !Sub |
    #!/bin/bash -xe
    # STEP 2 - Install system software - including Web and DB
    dnf install wget php-mysqlnd httpd php-fpm php-mysqli mariadb105-server php-json php php-devel cowsay -y
    # STEP 3 - Web and DB Servers Online - and set to startup
    systemctl enable httpd
    systemctl enable mariadb
    systemctl start httpd
    systemctl start mariadb
    # STEP 4 - Set Mariadb Root Password
    mysqladmin -u root password ${DBRootPassword}
    # STEP 5 - Install Wordpress
    wget http://wordpress.org/latest.tar.gz -P /var/www/html
    cd /var/www/html
    tar -zxvf latest.tar.gz
    cp -rvf wordpress/* .
    rm -R wordpress
    rm latest.tar.gz
    # STEP 6 - Configure Wordpress
    cp ./wp-config-sample.php ./wp-config.php
    sed -i "s/'database_name_here'/'${DBName}'/g" wp-config.php
    sed -i "s/'username_here'/'${DBUser}'/g" wp-config.php
    sed -i "s/'password_here'/'${DBPassword}'/g" wp-config.php
    # Step 6a - Permissions
    usermod -a -G apache ec2-user
    chown -R ec2-user:apache /var/www
    chmod 2775 /var/www
    find /var/www -type d -exec chmod 2775 {} \;
    find /var/www -type f -exec chmod 0664 {} \;
    # STEP 7 - Create Wordpress DB
    echo "CREATE DATABASE ${DBName};" >> /tmp/db.setup
    echo "CREATE USER '${DBUser}'@'localhost' IDENTIFIED BY '${DBPassword}';" >> /tmp/db.setup
    echo "GRANT ALL ON ${DBName}.* TO '${DBUser}'@'localhost';" >> /tmp/db.setup
    echo "FLUSH PRIVILEGES;" >> /tmp/db.setup
    mysql -u root --password=${DBRootPassword} < /tmp/db.setup
    sudo rm /tmp/db.setup
    # STEP 8 - COWSAY
    echo "#!/bin/sh" > /etc/update-motd.d/40-cow
    echo 'cowsay "Amazon Linux 2023 AMI - Animals4Life"' >> /etc/update-motd.d/40-cow
    chmod 755 /etc/update-motd.d/40-cow
    update-motd
```

With CloudFormation, database credentials are passed in as Parameters rather than hardcoded, making the template reusable.

---

## Step 4 — Cleanup

- Terminate the EC2 instance that was manually created
- Delete the CloudFormation stack used to create the a4l VPC environment

---

## Key Takeaways

- Always start User Data scripts with `#!/bin/bash -xe` — without it the script won't execute
- IMDSv2 requires a session token (PUT request) before any metadata can be read — always fetch the token first
- `cloud-init-output.log` is the go-to log for debugging bootstrap failures
- CloudFormation's `Fn::Base64` + `!Sub` pattern is the infrastructure-as-code equivalent of manual User Data
- IMDSv2's session-oriented auth is a security improvement over IMDSv1 — worth knowing for the SAA exam
