# Setting up Magento on AWS EC2 t2.micro Instance

In this tutorial, we will walk through the steps to set up Magento on an AWS EC2 t2.micro instance. Magento is a popular open-source e-commerce platform, and AWS EC2 is a cloud computing service that provides resizable compute capacity in the cloud.

## Prerequisites
Before starting the installation process, make sure you have the following:
- An AWS account with an EC2 t2.micro instance running Ubuntu Server.
- Access to the instance via SSH.

## Step 1: Update the system and create a Swap File
 Connect to your EC2 instance using SSH.
 Update the package list and upgrade existing packages:
   ```bash
    sudo apt update && sudo apt upgrade
   ```
 Create a Swap File as t2.micro instances often have limited RAM:
   ```bash
   sudo fallocate -l 5G /swapfile
   sudo chmod 600 /swapfile
   sudo mkswap /swapfile
   sudo swapon /swapfile
   ```

## Step 2: Install Nginx
 Install Nginx:
   ```bash
   sudo apt install nginx-full
   ```

## Step 3: Install PHP and MySQL
 Install PHP and required PHP extensions for Magento:
   ```bash
   sudo apt install php php-{bcmath,intl,mbstring,mysql,soap,xml,xsl,zip,cli,common,curl,fpm,gd}
   ```

 Install MySQL Server 8.0:
   ```bash
   sudo apt install mysql-server-8.0
   ```

## Step 4: Install Elasticsearch and Redis
 Install Elasticsearch for Magento's search functionality:
   ```bash
   wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg
   echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-7.x.list
   sudo apt update
   sudo apt install elasticsearch
   sudo systemctl daemon-reload
   sudo systemctl start elasticsearch.service
   ```

 Install Redis as a caching backend:
   ```bash
   curl -fsSL https://packages.redis.io/gpg | sudo gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg
   echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] https://packages.redis.io/deb $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/redis.list
   sudo apt update
   sudo apt install redis
   ```

    Configure Redis as the session and cache backend for improved performance:
    ```bash
    sudo php bin/magento setup:store-config:set --session-save=redis --session-save-redis-db=0 \
    --session-save-redis-password=YOUR_REDIS_PASSWORD --cache-backend=redis \
    --cache-backend-redis-db=2 --cache-backend-redis-password=YOUR_REDIS_PASSWORD \
    --page-cache=redis --page-cache-redis-db=4 --page-cache-redis-password=YOUR_REDIS_PASSWORD
    ```

## Step 5: Install Magento

 Install Composer (if not already installed):
 
   ```bash
    sudo apt-get install apt-transport-https curl
    sudo php composer-setup.php --install-dir=/usr/local/bin --filename=composer
   ```

Create Magento directory and set appropriate permissions:
 
    ```bash
    sudo mkdir -p /var/www/html/magento/
    sudo chown www-data:www-data /var/www/html/ -R
    ```

Change to the Magento directory:
 
    ```bash
    cd /var/www/html/
    ```
    
 Download and install Magento using Composer:
 
    ```bash
    sudo -u www-data composer create-project --repository-url=https://repo.magento.com/ magento/project-community-edition magento
    ```
    
## Step 6: Configure SSL with a Self-Signed Certificate
   Install OpenSSL (if not already installed):
   ```
   sudo apt-get install openssl
   ```bash

   Generate a self-signed SSL certificate and private key:
   ```bash
   sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/nginx.key -out /etc/ssl/certs/nginx.crt
   ```

   Generate a Diffie-Hellman parameter for enhanced security:
   ```bash
   sudo openssl dhparam -out /etc/nginx/dhparam.pem 2048
   ```

   Create a configuration file for self-signed certificates:
   ```bash
   sudo nano /etc/nginx/snippets/self-signed.conf
   ```

## Step 7: Configure Nginx
 Remove the default Nginx configuration:
    ```bash
    sudo rm /etc/nginx/sites-enabled/default
    ```

 Create a new Nginx configuration for Magento:
    ```bash
    sudo mv /var/www/html/magento/nginx.conf.sample /etc/nginx/conf.d/magento.sample
    ```

 Edit the Nginx configuration file:
    ```bash
    sudo nano /etc/nginx/conf.d/magento.conf
    ```

 Add the following content to the file:
      ```
    upstream fastcgi_backend {
            server unix:/var/run/php/php8.1-fpm.sock;
         }
         server {
                 listen 8080;
         #        listen [::]:80;
                 server_name test.mgt.com;
                 return 302 https://$server_name$request_uri;
         }
         server {
           listen 443 ssl;
         #  listen [::]:443 ssl;
         #  server_name localhost;
           include snippets/self-signed.conf;
           include snippets/phpmyadmin.conf;
         
           set $MAGE_ROOT /var/www/html/magento/;
           include /etc/nginx/conf.d/magento.sample;
           client_max_body_size 2M;
         
           access_log /var/log/nginx/magento.access;
           error_log /var/log/nginx/magento.error;
         }
      ```
 Test the Nginx configuration for any syntax errors:
    ```bash
    sudo nginx -t
    ```

 Restart Nginx to apply the changes:
    ```bash
    sudo service nginx restart
    ```

## Step 8: Set Up Magento
 Configure Magento with your specific settings using the following command:
    ```bash
    cd /var/www/html/magento/
    sudo php bin/magento setup:install --base-url=http://YOUR_DOMAIN_OR_IP \
    --db-host=localhost --db-name=YOUR_DB_NAME --db-user=YOUR_DB_USER --db-password=YOUR_DB_PASSWORD \
    --admin-firstname=admin --admin-lastname=admin --admin-email=admin@admin.com \
    --admin-user=admin --admin-password=admin123 --language=en_US --currency=USD \
    --timezone=America/Chicago --use-rewrites=1 --search-engine=elasticsearch7 \
    --elasticsearch-host=http://localhost --elasticsearch-port=9200 \
    --elasticsearch-index-prefix=magento2 --elasticsearch-timeout=15 --elasticsearch-enable-auth=0
    ```

 Set the correct ownership and permissions for Magento files:
    ```bash
    sudo chown -R www-data:www-data .
    ```

## Step 9: Final Steps and Debugging

In case you get errors on the website such as 503 or the php files are automatically downloading from the browser
For the 503 error you need to check the permissions and ownership of the files as well as nginx and php fpm
If the files are automatically downloading it means the php fpm is not configured correctly

 Set the Magento deployment mode to "developer":
    ```bash
    sudo php bin/magento deploy:mode:set developer
    ```

 Flush the Magento cache:
    ```bash
    sudo php bin/magento cache:flush
    ``` 

## Conclusion
Congratulations! You have successfully set up Magento on an AWS EC2 t2.micro instance, including the creation of a swap file to accommodate Elasticsearch. Now you can access your Magento store via the provided domain or IP. Remember to follow security best practices and regularly update your system to keep it secure.

Please note that this tutorial provides a basic setup for Magento, and depending on your specific requirements, you may need to configure additional settings or install additional modules. Always refer to the official Magento documentation for detailed information. Happy e-commerce selling with Magento! 🛍️
