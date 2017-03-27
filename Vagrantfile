# -*- mode: ruby -*-
# vi: set ft=ruby :
require 'securerandom'

$NO_NODES = 2
$TOKEN_FILE = ".cluster_token"

def get_cluster_token()
  if File.exist?($TOKEN_FILE) then
    token = File.read($TOKEN_FILE)
  else
    token = "#{SecureRandom.hex(3)}.#{SecureRandom.hex(8)}"
    File.write($TOKEN_FILE, token)
  end
  token
end

def common_setup_script()
  script = <<SCRIPT
export LANGUAGE=en_US.UTF-8
export LANG=en_US.UTF-8
export LC_ALL=en_US.UTF-8
export DEBIAN_FRONTEND=noninteractive
locale-gen en_US.UTF-8
dpkg-reconfigure locales
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF1 > /etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF1
apt-get update
apt-get install -y ntp
systemctl enable ntp
systemctl start ntp
apt-get install -y docker.io
apt-get install -y kubelet kubeadm kubectl kubernetes-cni
cat <<EOF2 > /etc/docker/daemon.json
{
  "insecure-registries": ["192.168.33.1:5000"]
}
EOF2
systemctl enable docker && systemctl start docker
systemctl enable kubelet && systemctl start kubelet
unset DEBIAN_FRONTEND
SCRIPT
end

def master_setup_script(cluster_token)
  script = <<SCRIPT
wget -q http://stedolan.github.io/jq/download/linux64/jq
chmod +x ./jq
mv jq /usr/local/bin
sed 's|127.0.0.1.*host0|#{ip_from_num(0)}\thost0\thost0|' -i /etc/hosts
kubeadm init --token=#{cluster_token} --api-advertise-addresses=#{ip_from_num(0)}
kubectl -n kube-system get ds -l 'component=kube-proxy' -o json \
  | jq '.items[0].spec.template.spec.containers[0].command |= .+ ["--proxy-mode=userspace"]' \
  |   kubectl apply -f - && kubectl -n kube-system delete pods -l 'component=kube-proxy'
kubectl apply -f https://git.io/weave-kube
kubectl create -f https://rawgit.com/kubernetes/dashboard/master/src/deploy/kubernetes-dashboard.yaml
cp /etc/kubernetes/admin.conf /vagrant
SCRIPT
end

def node_setup_script(host_no, cluster_token)
  script = <<SCRIPT
sed 's|127.0.0.1.*host#{host_no}|#{ip_from_num(host_no)}\thost#{host_no}\thost#{host_no}|' -i /etc/hosts
kubeadm join --token=#{cluster_token} #{ip_from_num(0)}
SCRIPT
end

def ip_from_num(i)
  "192.168.33.#{10+i}"
end

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
Vagrant.configure("2") do |config|
  # The most common configuration options are documented and commented below.
  # For a complete reference, please see the online documentation at
  # https://docs.vagrantup.com.

  # Every Vagrant development environment requires a box. You can search for
  # boxes at https://atlas.hashicorp.com/search.
  config.vm.box = "ubuntu/xenial64"
  config.vm.box_check_update = true

  # Provider-specific configuration so you can fine-tune various
  # backing providers for Vagrant. These expose provider-specific options.
  config.vm.provider "virtualbox" do |vb|
    # Display the VirtualBox GUI when booting the machine
    vb.gui = false
  
    # Customize the amount of memory on the VM:
    vb.memory = "2048"
  end

  # Enable provisioning with a shell script. Additional provisioners such as
  # Puppet, Chef, Ansible, Salt, and Docker are also available. Please see the
  # documentation for more information about their specific syntax and use.
  config.vm.provision "shell", inline: common_setup_script()

  # Generate a token to authenticate hosts joining the cluster
  cluster_token = get_cluster_token()

  # Kubernetes hosts (host 0 is master)
  (0..$NO_NODES).each do |i|
    config.vm.define "host#{i}" do |host|
      host.vm.network "forwarded_port", guest: 8001, host: (8001 + i)
      host.vm.network "private_network", ip: ip_from_num(i)
      host.vm.hostname = "host#{i}"

      if (i == 0) then
        host.vm.provision "shell", inline: master_setup_script(cluster_token)
      else
        host.vm.provision "shell", inline: node_setup_script(i, cluster_token)
      end
    end
  end

end
