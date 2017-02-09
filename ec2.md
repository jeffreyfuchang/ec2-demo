https://deseretbook.signin.aws.amazon.com/console

Launch ec2 instance.

Select `Ubuntu Server 14.04 LTS (HVM), SSD Volume Type - ami-d732f0b7`

Select `t2.nano`

On `Configure Security Group` select `Create a new security group` and name it.

On `Tags` name it. Add `UserName` value `ubuntu`

After you click `Launch` on the first drop down select `Create a new key pair` and name it.

```
mv ~/Downloads/demo.pem ~/.ssh
chmod 600 ~/.ssh/demo.pem
ssh -i ~/.ssh/demo.pem ubuntu@52.39.77.213
vim ~/.ssh/authorized_keys
```

Add everyone public key.


### nginx
```
sudo apt-get install nginx
sudo mv /etc/nginx/nginx.conf /etc/nginx/nginx.conf.orig
```


```
sudo vim /etc/nginx/nginx.conf
```

Paste below
```
user www-data;
worker_processes auto;
pid /run/nginx.pid;

events {
    worker_connections 768;
    # multi_accept on;
}


http {

  access_log /var/log/nginx/access.log;
  error_log /var/log/nginx/error.log;

  server {
      listen 80;
      server_name localhost;

      location / {
        return 200 'gangnam style!';
      }
  }

}
```
```
sudo nginx -t -c /etc/nginx/nginx.conf
sudo service nginx restart
```

Find your instance and under description select the security group.

Go to `Inbound` and click `Edit`

Click `Add Rule`

Select `HTTP`

Click `Add Rule`

Select `HTTPS`

Click `Save`

### DNS Route53

Click `Hosted Zones`

Click `deseretbook.net`

Click `Create Record Set`

Enter a name

Enter the ip address of the ec2 instance in `Value`

Click `Create`

### letsencrypt

```
sudo apt-get install git
sudo git clone https://github.com/letsencrypt/letsencrypt /opt/letsencrypt
```
```
sudo mkdir -p /var/www/letsencrypt
sudo chgrp www-data /var/www/letsencrypt

sudo mkdir -p /etc/letsencrypt/configs
sudo vim /etc/letsencrypt/configs/demo.deseretbook.net.conf
```

```
# the domain we want to get the cert for;
# technically it's possible to have multiple of this lines, but it only worked
# with one domain for me, another one only got one cert, so I would recommend
# separate config files per domain.
domains = demo.deseretbook.net

# increase key size
rsa-key-size = 2048 # Or 4096

# the current closed beta (as of 2015-Nov-07) is using this server
server = https://acme-v01.api.letsencrypt.org/directory

# this address will receive renewal reminders
email = webdev@deseretbook.com

# turn off the ncurses UI, we want this to be run as a cronjob
text = True

# authenticate by placing a file in the webroot (under .well-known/acme-challenge/)
# and then letting LE fetch it
authenticator = webroot
webroot-path = /var/www/letsencrypt/
```

```
sudo vim /etc/nginx/nginx.conf
```
Add below after the first location block.
```
     location /.well-known/acme-challenge {
         root /var/www/letsencrypt;
     }
```
```
sudo nginx -t && sudo nginx -s reload
```
Request the cert
```
sudo /opt/letsencrypt//letsencrypt-auto --config /etc/letsencrypt/configs/demo.deseretbook.net.conf certonly
```

Should see, and press a to agree
```
-------------------------------------------------------------------------------
Please read the Terms of Service at
https://letsencrypt.org/documents/LE-SA-v1.1.1-August-1-2016.pdf. You must agree
in order to register with the ACME server at
https://acme-v01.api.letsencrypt.org/directory
-------------------------------------------------------------------------------
(A)gree/(C)ancel:
```

If successful you should see
```
IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at
   /etc/letsencrypt/live/demo.deseretbook.net/fullchain.pem. Your cert
   will expire on 2016-11-30. To obtain a new or tweaked version of
   this certificate in the future, simply run letsencrypt-auto again.
   To non-interactively renew *all* of your certificates, run
   "letsencrypt-auto renew"
 - If you lose your account credentials, you can recover through
   e-mails sent to webdev@deseretbook.com.
 - Your account credentials have been saved in your Certbot
   configuration directory at /etc/letsencrypt. You should make a
   secure backup of this folder now. This configuration directory will
   also contain certificates and private keys obtained by Certbot so
   making regular backups of this folder is ideal.
 - If you like Certbot, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le
```



Request new cert every other month. Need to run as super user
```
sudo crontab -e
```

```
0 0 1 JAN,MAR,MAY,JUL,SEP,NOV * /home/ubuntu/renew-letsencrypt.sh
```

```
vim /opt/letsencrypt/renew-letsencrypt.sh
```
Insert
```
#!/bin/sh

cd /opt/letsencrypt/
./letsencrypt-auto --config /etc/letsencrypt/configs/demo.deseretbook.net.conf certonly

if [ $? -ne 0 ]
 then
        ERRORLOG=`tail /var/log/letsencrypt/letsencrypt.log`
        echo -e "The Let's Encrypt cert has not been renewed! \n \n" \
                 $ERRORLOG
 else
        sudo service nginx restart
fi

exit 0
```

```
chmod 655 /etc/log/letsencrypt
chmod +x /opt/letsencrypt/renew-letsencrypt.sh
```

```
sudo vim /etc/nginx/nginx.conf
```
Add
```
  server {
      listen 443 ssl default_server;
      server_name demo.deseretbook.net;

      ssl_certificate /etc/letsencrypt/live/demo.deseretbook.net/fullchain.pem;
      ssl_certificate_key /etc/letsencrypt/live/demo.deseretbook.net/privkey.pem;

       location / {
        return 200 'gangnam style!';
      }
  }
```

```
sudo nginx -t && sudo nginx -s reload
```

### postgresql

```
sudo apt-get install libpq-dev
```

Install postgres repo
```
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ `lsb_release -cs`-pgdg main" >> /etc/apt/sources.list.d/pgdg.list'
wget -q https://www.postgresql.org/media/keys/ACCC4CF8.asc -O - | sudo apt-key add -
sudo apt-get update
sudo apt-get install postgresql-9.6
```

Create postgres user and choose a password.
```
sudo -u postgres createuser -P ubuntu
```

```
sudo -u postgres psql postgres
```
Give access table create access to user `ubuntu`
```
alter user ubuntu createdb;
\q
```
Create database
```
psql -U ubuntu postgres
```
```
create database demo;
\q
```
```
sudo vim /etc/postgresql/9.6/main/postgresql.conf
```
Uncomment line
```
#listen_addresses = 'localhost'         # what IP address(es) to listen on;
```

sudo service postgresql restart

Test
```
psql -U ubuntu -h localhost postgres
```

### RVM

```
gpg --keyserver hkp://keys.gnupg.net --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3
\curl -sSL https://get.rvm.io | bash -s stable

source /home/ubuntu/.rvm/scripts/rvm

rvm install ruby-2.3.2
rvm use ruby-2.3.2
rvm gemset create demo
rvm gemset use demo

gem install bundle
gem install rails
```

```
cd ~/demo
vim .ruby-version
vim .ruby-gemset
```


```
cd ~/demo
vim .ruby-env
```
Add

```
RAILS_ENV=development
```

RVM should be run as a function
```
type rvm | head -n 1

# should be
rvm is a function
```

### rails project

```
rails new demo --database=postgresql
bundle exec rake db:create
```

```
vim ~/demo/config/database.yml
```
```
production:
  <<: *default
  database: demo
  username: ubuntu
  password: <%= ENV['DEMO_DATABASE_PASSWORD'] || 'ubuntu' %>
```

### puma
```
sudo apt-get install nodejs
mkdir ~/demo/shared
```
```
vim ~/demo/config/puma.rb
```
Add
```
app_dir = File.expand_path("../..", __FILE__)
shared_dir = "#{app_dir}/shared"

# Set master PID and state locations
pidfile "#{shared_dir}/puma.pid"
state_path "#{shared_dir}/puma.state"

# Logging
stdout_redirect "#{app_dir}/log/puma.stdout.#{ENV.fetch("RAILS_ENV") { "development" }}.log", "#{app_dir}/log/puma.stderr.#{ENV.fetch("RAILS_ENV") { "development" }}.log", true
```


manual start
```
bundle exec puma -C config/puma.rb -e stage
```




### Final nginx conf

```
user www-data;
worker_processes auto;
pid /run/nginx.pid;

events {
    worker_connections 768;
    # multi_accept on;
}


http {

  access_log /var/log/nginx/access.log;
  error_log /var/log/nginx/error.log;

  upstream app {
      server unix:/home/ubuntu/demo/shared/puma.sock fail_timeout=0;
      #server localhost:3000;
  }

  server {
      listen 80;
      server_name _;

      location / {
        return 301 https://$host$request_uri;
      }

      location /.well-known/acme-challenge {
          root /var/www/letsencrypt;
      }
  }

  # HTTPS server
  #
  server {
      listen 443 ssl default_server;
      server_name demo.deseretbook.net;

      ssl_certificate /etc/letsencrypt/live/demo.deseretbook.net/fullchain.pem;
      ssl_certificate_key /etc/letsencrypt/live/demo.deseretbook.net/privkey.pem;

      #error_log /var/log/nginx/error.log warn;

      root /home/ubuntu/demo/public;

      try_files $uri/index.html $uri @app;

      location @app {
          proxy_pass http://app;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header Host $http_host;
          proxy_redirect off;
      }

      error_page 500 502 503 504 /500.html;
      client_max_body_size 10M;
      keepalive_timeout 10;

  }

}
```
### screen and rvm

`https://rvm.io/workflow/screen`

Edit
```
 ~/.screenrc
```
Add
```
shell -${SHELL}
```
