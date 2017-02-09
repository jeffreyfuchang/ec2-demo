Ubuntu 16.04 does't install upstart anymore
```
https://wiki.ubuntu.com/SystemdForUpstartUsers
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
#port        ENV.fetch("PORT") { 3000 }

# Specifies the `environment` that Puma will run in.
#
environment ENV.fetch("RAILS_ENV") { "development" }

app_dir = File.expand_path("../..", __FILE__)
shared_dir = "#{app_dir}/shared"

# Set up socket location
bind "unix://#{shared_dir}/puma.sock"

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
