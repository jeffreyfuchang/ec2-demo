web2relate-stg server is 10.4.0.89
web2relate production server is 10.4.0.88


ssh webdev@10.4.0.89

sudo vim autossh-tunnel.conf

sudo service autossh-tunnel stop
sudo service autossh-tunnel start

# This is an upstart script.
# Should be installed at /etc/init/
# This allows us to call `sudo service autossh-tunnel start/stop/restart`
# Also allows autossh to start this tunnel during boot
description "autossh daemon for ssh tunnel to oapi aws"

start on net-device-up IFACE=eth0
stop on runlevel [016]

setuid webdev

respawn
respawn limit unlimited

script
export AUTOSSH_FIRST_POLL=30
export AUTOSSH_GATETIME=0
export AUTOSSH_LOGFILE="/var/log/autossh.log"
export DBC_IP="labrelate.deseretbook.com"
export AWS_IP="35.163.203.75"
autossh -M 0 -N -R "8443:$DBC_IP:8443" -o ExitOnForwardFailure=yes -o "ServerAliveInterval 60" -o "ServerAliveCountMax 3" -o "StrictHostKeyChecking=no" -o "BatchMode=yes" "ubuntu@$AWS_IP"

autossh -M 0 -N -R 8443:$DBC_IP:8443 -o ExitOnForwardFailure=yes -o "ServerAliveInterval 60" -o "ServerAliveCountMax 3" -o "StrictHostKeyChecking=no" -o "BatchMode=yes" ubuntu@$AWS_IP

end script





http://blog.siphos.be/2015/08/switching-openssh-to-ed25519-keys/
ssh-keygen -t ed25519




vim ~/.ssh/config


Host aws-relate-api
  HostName 35.163.203.75
  User ubuntu
#  LocalForward 8443 localhost:8443
  RemoteForward 8443 localhost:8443
