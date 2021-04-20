## Setup your own Nextcloud instance

This repository is a 'tutorial' of how to setup your own instance of [Nextcloud](https://nextcloud.com) with a PostgreSQL database.

## To Do

- [x] Backup on AWS S3;
- [ ] Allow synchronization with multiple devices;

## Tutorial

### Steps to install Nextcloud with PostgreSQL

###### Create a virtual machine with Ubuntu 20.04;

###### Install PostgreSQL:

```
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
sudo apt-get update
sudo apt-get -y install postgresql
sudo service postgresql restart
sudo systemctl enable postgresql.service
sudo systemctl status postgresql.service
```

###### Change default user `postgres` password:

```
sudo -u postgres psql
\password postgres
\q
```

###### Create database and user for Nextcloud:

```
sudo -u postgres psql
CREATE DATABASE nextcloud;
CREATE USER nextcloud WITH PASSWORD 'password';
GRANT CONNECT ON DATABASE nextcloud TO nextcloud;
GRANT USAGE ON SCHEMA public TO nextcloud;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO nextcloud;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO nextcloud;
\q
```

###### Install PHP:

```
sudo apt install php php-mbstring php-pgsql php-bz2 php-intl php-gmp php-apcu php-memcached php-redis php-imagick php-zip php-dompdf php-xml php-curl
```

###### Download Nextcloud:

```
wget https://download.nextcloud.com/server/releases/nextcloud-21.0.1.zip
unzip nextcloud-21.0.1.zip
sudo mv nextcloud /var/www
sudo chown -R www-data:www-data /var/www/nextcloud/
```

###### Install and configure Apache:

```
sudo apt install apache2
sudo vim /etc/apache2/sites-available/nextcloud.conf
```

The file `nextcloud.conf` must have the following:

```
ServerName localhost

<VirtualHost *:80>
    RewriteEngine On
    RewriteCond %{HTTPS} off
    RewriteRule ^ https://%{HTTP_HOST}%{REQUEST_URI}
</VirtualHost>

<VirtualHost *:443>
    DocumentRoot /var/www/nextcloud/

    SSLEngine On
    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
    SSLCertificateFile /etc/ssl/certs/ssl-cert-snakeoil.pem
    SSLCertificateKeyFile /etc/ssl/private/ssl-cert-snakeoil.key

    <Directory /var/www/nextcloud/>
      Require all granted
      AllowOverride All
      Options FollowSymLinks MultiViews

      <IfModule mod_dav.c>
        Dav off
      </IfModule>
    </Directory>
</VirtualHost>
```

Save the file and then run

```
sudo rm -rf /etc/apache2/sites-enabled/000-default.conf
sudo rm -rf /etc/apache2/sites-enabled/default-ssl.conf
sudo a2ensite nextcloud.conf
sudo a2enmod rewrite headers env dir mime setenvif ssl default-ssl
sudo systemctl reload apache2
sudo systemctl restart apache2
sudo systemctl enable apache2
```

###### Nextcloud should be accessible on `localhost`;

## Auto backup on S3

- Create a S3 bucket;
- Create a policy with the following content:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": "s3:ListBucket",
            "Resource": "arn:aws:s3:::<s3-bucket>"
        },
        {
            "Sid": "VisualEditor1",
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:DeleteObject"
            ],
            "Resource": "arn:aws:s3:::<s3-bucket>/*"
        }
    ]
}
```

- Create an user and attach the policy just created;
- Create an access key for the new user;
- Install AWS CLI:

```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

- Configure credentials:

```
aws configure
sudo aws configure
```

Configure also as `sudo` to be able to run aws cli as root.

- Create a script with the following content:

```
sudo aws s3 sync /var/www/nextcloud/data s3://<s3-bucket>/nextcloud
sudo -u postgres pg_dump nextcloud -F t > $HOME/nextcloud_database.tar
sudo aws s3 cp $HOME/nextcloud_database.tar s3://<s3-bucket>/database/
sudo rm $HOME/nextcloud_database.tar
```

- Run `sudo crontab -e` and schedule the script on `crontab`:

```
0 22 * * * sh <path>.sh
```

Here I'm running the script every day at 10 PM.
