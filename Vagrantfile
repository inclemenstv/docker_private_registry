require 'yaml'
config = YAML.load_file 'config_vm.yaml'

IMAGE_NAME       = config['Docker_registry']['IMAGE_NAME']
REGISTRY_IP      = config['Docker_registry']['REGISTRY_IP']
USER_NAME        = config['Docker_registry']['USER_NAME']
USER_PASSWORD    = config['Docker_registry']['USER_PASSWORD']
REGISTRY_HOST    = config['Docker_registry']['HOST']

Vagrant.configure("2") do |config|


    config.vm.define "docker_registry" do |docker|
       docker.vm.box = IMAGE_NAME
       docker.vm.network "private_network", ip: REGISTRY_IP
       docker.vm.hostname = "registry"
       docker.vm.provider "virtualbox" do |v|
        v.memory = 1024
        v.cpus = 2
    end
       docker.vm.provision :shell, privileged: true, inline: $install_docker
       docker.vm.provision :shell, env: {"REGISTRY_IP" => REGISTRY_IP, "USER_NAME" => USER_NAME, "USER_PASSWORD" => USER_PASSWORD, "REGISTRY_HOST" => REGISTRY_HOST }, privileged: true, inline: $config_registry
end


end



$install_docker = <<-SCRIPT
echo "Updating apt-get"
sudo apt-get -qq update
echo "Installing packages"
sudo apt-get -y install \
apt-transport-https \
ca-certificates \
url \
gnupg \
lsb-release > /dev/null 2>&1
echo "Add Docker’s official GPG key"
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg > /dev/null
echo "Set up the stable repository."
echo \
"deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
$(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
echo "Installing Docker"
sudo apt-get update > /dev/null 2>&1
sudo apt-get -y install docker-ce docker-ce-cli containerd.io > /dev/null 2>&1

echo "Start docker service"
sudo service docker start

SCRIPT

$config_registry = <<-SCRIPT
#!/bin/bash

sudo sed -i "/v3_ca/a subjectAltName = IP:$REGISTRY_IP"  /etc/ssl/openssl.cnf

echo "Creating working dir"
mkdir /home/vagrant/registry
cd /home/vagrant/registry

echo "genereting certificate"
sudo openssl req -x509 -nodes -sha256 -newkey rsa:4096 -keyout registry.key -out registry.crt -days 14 -subj "/CN=$REGISTRY_IP" > /dev/null 2>&1

echo "copying certs"
sudo mkdir -p "/etc/docker/certs.d/$REGISTRY_IP:5000"
sudo cp /home/vagrant/registry/registry.crt "/etc/docker/certs.d/$REGISTRY_IP:5000"

echo "restarting docker"
sudo systemctl restart docker
sudo sleep 10


echo "creating credentials"
cd /home/vagrant/registry
sudo docker run --rm --entrypoint htpasswd registry:2.7.0 -Bbn $USER_NAME $USER_PASSWORD > htpasswd

echo "copying and config docker registry config"
sudo cp /vagrant/registry/config.yml /home/vagrant/registry
sudo sed -i "s/host:/host: $REGISTRY_HOST/" /home/vagrant/registry/config.yml

echo "copying Dockerfile"
sudo cp /vagrant/registry/Dockerfile /home/vagrant/registry

echo "Building registry"
cd /home/vagrant/registry
sudo docker build -t local-registry .

echo "run registry"
sudo docker run -d -p 5000:5000 --name registry local-registry
SCRIPT
