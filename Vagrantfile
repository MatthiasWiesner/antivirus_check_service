$script = <<SCRIPT
apt update
apt -y upgrade

# install all needed packages
apt install -y python3 rabbitmq-server amqp-tools clamav-daemon clamav-freshclam clamav-unofficial-sigs make git

# configure tcp network socket
echo 'TCPSocket 3310' >> /etc/clamav/clamd.conf
echo 'TCPAddr 0.0.0.0' >> /etc/clamav/clamd.conf
systemctl restart clamav-daemon.service

# enable rabbitmq cli tools
rabbitmq-plugins enable rabbitmq_management

# set rabbitmq user, vhost and queue
cp /vagrant/secrets/rabbitmq.config /etc/rabbitmq/rabbitmq.config
cp /vagrant/secrets/rabbitmq-definitions.json /etc/rabbitmq/rabbitmq-definitions.json
systemctl restart rabbitmq-server

# set service config
cp /vagrant/secrets/config.yml /vagrant/antivirus_service/config.yml

# install antivirus service
cd /vagrant
make install

# The clamav-daemon has to wait until the freshclam-daemon has finished loading the signatures.
# You eventually have to restart clamav-daemon.
# If clamav-daemon is running properly, you can start the services by:

# systemctl start antivirus-webserver.service
# systemctl start antivirus-scanurl.service
# systemctl start antivirus-scanfile.service
SCRIPT

Vagrant.configure("2") do |config|
  config.vm.box = "debian/stretch64"

  config.vm.define "servicehost" do |master|
    master.vm.network :private_network, ip: "192.168.2.1"
    master.vm.network "forwarded_port", guest: 15672, host: 15672
  end

  config.vm.synced_folder ".", "/vagrant", type: "virtualbox"

  config.vm.provision "shell", inline: $script
end