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
sudo chgrp www-data /etc/letsencrypt

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
/opt/letsencrypt//letsencrypt-auto --config /etc/letsencrypt/configs/demo.deseretbook.net.conf certonly
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



Request new cert every other month
```
crontab -e
```

```
0 0 1 JAN,MAR,MAY,JUL,SEP,NOV * /home/ubuntu/renew-letsencrypt.sh
```

```
vim ~/renew-letsencrypt.sh
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
sudo apt-get install postgresql-9.5
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
sudo vim /etc/postgresql/9.5/main/postgresql.conf
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

rvm install ruby-2.3.1
rvm use ruby-2.3.1
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




### upstart scripts
Edit
```
sudo vim /etc/environment
```
Restarting the system will load the environment file when you login. That file is only read on login by pam_env.
```
. /etc/environment
```
```
while read -r env; do export "$env"; done
```
Add
```
RAILS_ENV=production
```
Edit
```
sudo vim /etc/puma.conf
```
Add
```
/home/ubuntu/demo
```
```
sudo vim /etc/init/puma-manager.conf
```
Add
```
# /etc/init/puma-manager.conf - manage a set of Pumas

# This example config should work with Ubuntu 12.04+.  It
# allows you to manage multiple Puma instances with
# Upstart, Ubuntu's native service management tool.
#
# See puma.conf for how to manage a single Puma instance.
#
# Use "stop puma-manager" to stop all Puma instances.
# Use "start puma-manager" to start all instances.
# Use "restart puma-manager" to restart all instances.
# Crazy, right?
#

description "Manages the set of puma processes"

# This starts upon bootup and stops on shutdown
start on runlevel [2345]
stop on runlevel [06]

# Set this to the number of Puma processes you want
# to run on this machine
env PUMA_CONF="/etc/puma.conf"

pre-start script
  for i in `cat $PUMA_CONF`; do
    app=`echo $i | cut -d , -f 1`
    logger -t "puma-manager" "Starting $app"
    start puma app=$app
  done
end script
```
Edit
```
sudo vim /etc/init/puma.conf
```
Add
```
# /etc/init/puma.conf - Puma config

# This example config should work with Ubuntu 12.04+.  It
# allows you to manage multiple Puma instances with
# Upstart, Ubuntu's native service management tool.
#
# See workers.conf for how to manage all Puma instances at once.
#
# Save this config as /etc/init/puma.conf then manage puma with:
#   sudo start puma app=PATH_TO_APP
#   sudo stop puma app=PATH_TO_APP
#   sudo status puma app=PATH_TO_APP
#
# or use the service command:
#   sudo service puma {start,stop,restart,status}
#

description "Puma Background Worker"

# no "start on", we don't want to automatically start
stop on (stopping puma-manager or runlevel [06])

# change apps to match your deployment user if you want to use this as a less privileged user (recommended!)
setuid ubuntu
setgid ubuntu

respawn
respawn limit 3 30

instance ${app}

script
# this script runs in /bin/sh by default
# respawn as bash so we can source in rbenv/rvm
# quoted heredoc to tell /bin/sh not to interpret
# variables

# source ENV variables manually as Upstart doesn't, eg:
#. /etc/environment

exec /bin/bash <<'EOT'
  # set HOME to the setuid user's home, there doesn't seem to be a better, portable way
  #export HOME="$(eval echo ~$(id -un))"
  export HOME="/home/ubuntu"

  if [ -d "/usr/local/rbenv/bin" ]; then
    export PATH="/usr/local/rbenv/bin:/usr/local/rbenv/shims:$PATH"
  elif [ -d "$HOME/.rbenv/bin" ]; then
    export PATH="$HOME/.rbenv/bin:$HOME/.rbenv/shims:$PATH"
  elif [ -f  /etc/profile.d/rvm.sh ]; then
    source /etc/profile.d/rvm.sh
  elif [ -f /usr/local/rvm/scripts/rvm ]; then
    source /etc/profile.d/rvm.sh
  elif [ -f "$HOME/.rvm/scripts/rvm" ]; then
    source "$HOME/.rvm/scripts/rvm"
  elif [ -f /usr/local/share/chruby/chruby.sh ]; then
    source /usr/local/share/chruby/chruby.sh
    if [ -f /usr/local/share/chruby/auto.sh ]; then
      source /usr/local/share/chruby/auto.sh
    fi
    # if you aren't using auto, set your version here
    # chruby 2.0.0
  fi

  cd $app
  logger -t puma "Starting server: $app"

  exec bundle exec puma -C config/puma.rb
EOT
end script
```

Start the demo app via the upstart scripts
```
sudo start puma-manager
```







### Final puma rb
```
# Puma can serve each request in a thread from an internal thread pool.
# The `threads` method setting takes two numbers a minimum and maximum.
# Any libraries that use thread pools should be configured to match
# the maximum value specified for Puma. Default is set to 5 threads for minimum
# and maximum, this matches the default thread size of Active Record.
#
threads_count = ENV.fetch("RAILS_MAX_THREADS") { 5 }.to_i
threads threads_count, threads_count

# Specifies the `port` that Puma will listen on to receive requests, default is 3000.
#
port        ENV.fetch("PORT") { 3000 }

# Specifies the `environment` that Puma will run in.
#
environment ENV.fetch("RAILS_ENV") { "development" }

app_dir = File.expand_path("../..", __FILE__)
shared_dir = "#{app_dir}/shared"

# Set up socket location
#bind "unix://#{shared_dir}/puma.sock"

# Set master PID and state locations
pidfile "#{shared_dir}/puma.pid"
state_path "#{shared_dir}/puma.state"

# Logging
stdout_redirect "#{app_dir}/log/puma.stdout.#{ENV.fetch("RAILS_ENV") { "development" }}.log", "#{app_dir}/log/puma.stderr.#{ENV.fetch("RAILS_ENV") { "development" }}.log", true

# Specifies the number of `workers` to boot in clustered mode.
# Workers are forked webserver processes. If using threads and workers together
# the concurrency of the application would be max `threads` * `workers`.
# Workers do not work on JRuby or Windows (both of which do not support
# processes).
#
# workers ENV.fetch("WEB_CONCURRENCY") { 2 }

# Use the `preload_app!` method when specifying a `workers` number.
# This directive tells Puma to first boot the application and load code
# before forking the application. This takes advantage of Copy On Write
# process behavior so workers use less memory. If you use this option
# you need to make sure to reconnect any threads in the `on_worker_boot`
# block.
#
# preload_app!

# The code in the `on_worker_boot` will be called if you are using
# clustered mode by specifying a number of `workers`. After each worker
# process is booted this block will be run, if you are using `preload_app!`
# option you will want to use this block to reconnect to any threads
# or connections that may have been created at application boot, Ruby
# cannot share connections between processes.
#
# on_worker_boot do
#   ActiveRecord::Base.establish_connection if defined?(ActiveRecord)
# end

# Allow puma to be restarted by `rails restart` command.
plugin :tmp_restart
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

  upstream app {
      #server unix:/home/ubuntu/demo/shared/puma.sock fail_timeout=0;
      server localhost:3000;
  }

  server {
      listen 80;
      server_name localhost;

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

      error_log /var/log/nginx/error.log warn;

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
