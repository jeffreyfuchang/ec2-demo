### Start Puma with Systemd

```
vim
```

```
eval $(cat /etc/environment | sed 's/^/export /')

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

  cd /home/ubuntu/relate_api
  logger -t puma "Starting server: $app"

  exec bundle exec puma -C /home/ubuntu/relate_api/config/puma.rb
EOT
```

```
# based on https://gist.github.com/twtw/5494223
# create systemd service file for rails/puma startup
# 0. [if required: rvm use ruby@default]
# 1. rvm wrapper default systemd rails
# 2. put this file in /etc/systemd/system/rails-puma.service
# 3. sudo systemctl enable rails-puma
# 4. sudo systemctl start rails-puma
[Unit]
Description=Rails-Puma Webserver

[Service]
Type=simple
User=app
WorkingDirectory=/home/ubuntu/relate_api
ExecStart=/home/ubuntu/relate_api/config/relate.sh
ExecStart=/home/ubuntu/.rvm/gems/ruby-2.3.1@relate_api/bin/bundle exec puma -C /home/ubuntu/relate_api/config/puma.rb
TimeoutSec=15
Restart=always

[Install]
WantedBy=multi-user.target
```

---------------------------

```
sudo vim /usr/bin/puma-init
sudo chmod +x /usr/bin/puma-init
```

```
#!/bin/sh

# Set the Puma server configuration file.
APP_DIR=/home/ubuntu/relate_api
PUMA_CONFIG_FILE="$APP_DIR/config/puma.rb"

export HOME="/home/ubuntu"
source "$HOME/.rvm/scripts/rvm"

status() {
    echo "Reporting status of the Puma HTTP server."
    cd $APP_DIR
    bundle exec pumactl -F "$PUMA_CONFIG_FILE" status
}

start() {
    echo "Starting the Puma HTTP server."
    cd $APP_DIR
    bundle exec pumactl -F "$PUMA_CONFIG_FILE" start
}

stop() {
    echo "Stopping the Puma HTTP server."
    cd $APP_DIR
    bundle exec pumactl -F "$PUMA_CONFIG_FILE" stop
}

restart() {
    echo "Restarting the Puma HTTP server."
    cd $APP_DIR
    bundle exec pumactl -F "$PUMA_CONFIG_FILE" restart
}

case $1 in
  status|start|stop|restart) "$1" ;;
esac
```

```
sudo vim /etc/systemd/system/rails-puma.service

sudo systemctl enable rails-puma.service

sudo systemctl daemon-reload

sudo systemctl start rails-puma.service

sudo journalctl -u rails-puma.service

sudo systemctl status rails-puma.service

```

```
REF: https://rvm.io/rubies/alias

cd relate_api/
rvm current

rvm alias create relate_api ruby-2.3.1@relate_api
rvm alias list


/home/ubuntu/.rvm/wrappers/relate_api

ERROR:
  Could not locate Gemfile or .bundle/ directory

gem update --system
```

```
[Unit]
Description=Puma (Ruby HTTP server)

[Service]
Type=oneshot
WorkingDirectory=/home/ubuntu/relate_api
User=ubuntu
Environment=RAILS_ENV=stage
ExecStart=/bin/bash -lc "cd /home/ubuntu/relate_api && /home/ubuntu/.rvm/wrappers/relate_api/bundle exec pumactl -F /home/ubuntu/relate_api/config/puma.rb start &"
ExecStop=/bin/bash -lc "cd /home/ubuntu/relate_api && /home/ubuntu/.rvm/wrappers/relate_api/bundle exec pumactl -F /home/ubuntu/relate_api/config/puma.rb stop"
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

```
environment variables
set in application.yml

also can do do .ruby-env in the project folder
```
