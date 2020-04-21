# -*- mode: ruby -*-
# vi:set ft=ruby sw=2 ts=2 sts=2:

# Define the number of machines
NUM_NODES = 3

# Node network
IP_NW = "172.16.0."
NODES_IP_START = 10

Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/bionic64"

  # Provision Nodes
  (1..NUM_NODES).each do |i|
    config.vm.define "ubuntu-#{i}" do |node|
      # Name shown in the GUI
      node.vm.provider "virtualbox" do |vb|
        vb.name = "ubuntu-lab-#{i}"
        vb.memory = 512
        vb.cpus = 1
      end
      node.vm.hostname = "ubuntu-#{i}"
      node.vm.network :private_network, ip: IP_NW + "#{NODES_IP_START + i}"
      #node.vm.network "forwarded_port", guest: 22, host: "#{2750 + i}"

      node.vm.provision "environment-file", type: "file", source: "ubuntu-lab.env", destination: "/tmp/ubuntu-lab.sh"
      node.vm.provision "setup-environment", type: "shell", inline: "mv /tmp/ubuntu-lab.sh /etc/profile.d/"
      
      node.vm.provision "setup-ssh", type: "shell", inline: $setup_ssh, privileged: false
      node.vm.provision "setup-hosts", type: "shell", inline: $setup_hosts

    end
  end

end


$setup_ssh = <<SCRIPT
set -x
if [ -r /vagrant/ssh/id_ed25519 ]; then
  cp /vagrant/ssh/id_* ~/.ssh/
  cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys
else  
  ssh-keygen -t ed25519 -a 100 -q -N "" -f ~/.ssh/id_ed25519
  cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys
  mkdir -p /vagrant/ssh
  cp ~/.ssh/id_* /vagrant/ssh/
fi
SCRIPT


$setup_hosts = <<SCRIPT
set -euxo pipefail
# remove 127.0.1.1 and ubuntu-bionic entry
sed -e '/^127.0.1.1.*/d' -i /etc/hosts
sed -e '/^.*ubuntu-bionic.*/d' -i /etc/hosts

# Update /etc/hosts about other hosts
for i in {1..#{NUM_NODES}}; do
  NR=$(expr #{NODES_IP_START} + ${i})
  echo "#{IP_NW}${NR} ubuntu-${i}" >> /etc/hosts
done
SCRIPT


$install_docker = <<SCRIPT
set -x
# Install docker
curl -fsSL https://get.docker.com | bash

# Give vagrant user access to docker socket
usermod -aG docker vagrant

# Setup daemon
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

mkdir -p /etc/systemd/system/docker.service.d

# Restart docker
systemctl daemon-reload
systemctl restart docker
SCRIPT


$allow_bridge_nf_traffic = <<SCRIPT
set -euxo pipefail

lsmod | grep br_netfilter || modprobe br_netfilter

cat <<EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sysctl --system
SCRIPT
