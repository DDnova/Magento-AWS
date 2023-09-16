# Setting up Magento on AWS EC2 t2.micro Instance

In this tutorial, we will walk through the steps to set up Magento on an AWS EC2 t2.micro instance. Magento is a popular open-source e-commerce platform, and AWS EC2 is a cloud computing service that provides resizable compute capacity in the cloud.

## Prerequisites
Before starting the installation process, make sure you have the following:
- An AWS account with an EC2 t2.micro instance running Ubuntu Server.
- Access to the instance via SSH.

## Step 1: Update the system and create a Swap File
1. Connect to your EC2 instance using SSH.
2. Update the package list and upgrade existing packages:
   ```
   sudo apt update && sudo apt upgrade
   ```
3. Create a Swap File as t2.micro instances often have limited RAM:
   ```
   sudo fallocate -l 5G /swapfile
   sudo chmod 600 /swapfile
   sudo mkswap /swapfile
   sudo swapon /swapfile
   ```

## Step 2: Install Nginx
4. Install Nginx:
   ```
   sudo apt install nginx-full
   ```

## Step 3: Install PHP and MySQL
5. Install PHP and required PHP extensions for Magento:
   ```
   sudo apt install php php-{bcmath,intl,mbstring,mysql,soap,xml,xsl,zip,cli,common,curl,fpm,gd}
   ```

6. Install MySQL Server 8.0:
   ```
   sudo apt install mysql-server-8.0
   ```

## Step 4: Install Elasticsearch and Redis
7. Install Elasticsearch for Magento's search functionality:
   ```
   wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg
   echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-7.x.list
   sudo apt update
   sudo apt install elasticsearch
   sudo systemctl daemon-reload
   sudo systemctl start elasticsearch.service
   ```

8. Install Redis as a caching backend:
   ```
   curl -fsSL https://packages.redis.io/gpg | sudo gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg
   echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] https://packages.redis.io/deb $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/redis.list
   sudo apt update
   sudo apt install redis
   ```

    Configure Redis as the session and cache backend for improved performance:
    ```
    sudo php bin/magento setup:store-config:set --session-save=redis --session-save-redis-db=0 \
    --session-save-redis-password=YOUR_REDIS_PASSWORD --cache-backend=redis \
    --cache-backend-redis-db=2 --cache-backend-redis-password=YOUR_REDIS_PASSWORD \
    --page-cache=redis --page-cache-redis-db=4 --page-cache-redis-password=YOUR_REDIS_PASSWORD
    ```

## Step 5: Install Magento
9. Install Composer (if not already installed):
   ```
   sudo apt-get install apt-transport-https curl
   sudo php composer-setup.php --install-dir=/usr/local/bin --filename=composer
   ```

10. Create Magento directory and set appropriate permissions:
    ```
    sudo mkdir -p /var/www/html/magento/
    sudo chown www-data:www-data /var/www/html/ -R
    ```

11. Change to the Magento directory:
    ```
    cd /var/www/html/
    ```

12. Download and install Magento using Composer:
    ```
    sudo -u www-data composer create-project --repository-url=https://repo.magento.com/ magento/project-community-edition magento
    ```

## Step 6: Configure Nginx
13. Remove the default Nginx configuration:
    ```
    sudo rm /etc/nginx/sites-enabled/default
    ```

14. Create a new Nginx configuration for Magento:
    ```
    sudo mv /var/www/html/magento/nginx.conf.sample /etc/nginx/conf.d/magento.sample
    ```

15. Edit the Nginx configuration file:
    ```
    sudo nano /etc/nginx/conf.d/magento.conf
    ```

16. Add the following content to the file:
      ```
      nginx
      Copy code
      server {
          listen 80;
          server_name YOUR_DOMAIN_OR_IP;
          root /var/www/html/magento/pub;
      
          index index.php;
          autoindex off;
          charset UTF-8;
          error_page 404 403 = /errors/404.php;
          # ...
          # Other Nginx configuration directives specific to your setup
          # ...
      
          location / {
              try_files $uri $uri/ /index.php?$args;
          }
      
          location /pub/ {
              location ~ ^/pub/media/(downloadable|customer|import|theme_customization/.*\.xml) {
                  deny all;
              }
              alias /var/www/html/magento/pub/;
              add_header X-Frame-Options "SAMEORIGIN";
          }
      
          location /static/ {
              # ...
              # Static content caching directives (optional)
              # ...
          }
      
          location /media/ {
              # ...
              # Media files caching directives (optional)
              # ...
          }
      
          location ~ ^/index\.php {
              fastcgi_pass unix:/run/php/php8.1-fpm.sock;  # Change version if different
              fastcgi_index index.php;
              fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
              include fastcgi_params;
          }
      
          location ~ (index|get|static|report|404|503|health_check)\.php$ {
              fastcgi_pass unix:/run/php/php8.1-fpm.sock;  # Change version if different
              fastcgi_index index.php;
              fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
              include fastcgi_params;
          }
      
          location ~* \.(ico|jpg|jpeg|png|gif|svg|js|css|swf|eot|ttf|otf|woff|woff2)$ {
              expires 1y;
              add_header Cache-Control "public";
          }
      
          location ~* \.(zip|gz|gzip|bz2|csv|xml)$ {
              expires 1h;
              add_header Cache-Control "no-store";
          }
      }
      ```
17. Test the Nginx configuration for any syntax errors:
    ```
    sudo nginx -t
    ```

18. Restart Nginx to apply the changes:
    ```
    sudo service nginx restart
    ```

## Step 7: Set Up Magento
19. Configure Magento with your specific settings using the following command:
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

20. Set the correct ownership and permissions for Magento files:
    ```
    sudo chown -R www-data:www-data .
    ```

## Step 8: Final Steps and Debugging

In case you get errors on the website such as 503 or the php files are automatically downloading from the browser
For the 503 error you need to check the permissions and ownership of the files as well as nginx and php fpm
If the files are automatically downloading it means the php fpm is not configured correctly

21. Set the Magento deployment mode to "developer":
    ```
    sudo php bin/magento deploy:mode:set developer
    ```

22. Flush the Magento cache:
    ```
    sudo php bin/magento cache:flush
    ``` 

## Conclusion
Congratulations! You have successfully set up Magento on an AWS EC2 t2.micro instance, including the creation of a swap file to accommodate Elasticsearch. Now you can access your Magento store via the provided domain or IP. Remember to follow security best practices and regularly update your system to keep it secure.

Please note that this tutorial provides a basic setup for Magento, and depending on your specific requirements, you may need to configure additional settings or install additional modules. Always refer to the official Magento documentation for detailed information. Happy e-commerce selling with Magento! 🛍️
