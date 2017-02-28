# -*- mode: ruby -*-
# vi: set ft=ruby :

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
Vagrant.configure(2) do |config|
  # The most common configuration options are documented and commented below.
  # For a complete reference, please see the online documentation at
  # https://docs.vagrantup.com.

  # Every Vagrant development environment requires a box. You can search for
  # boxes at https://atlas.hashicorp.com/search.
  # UCP 2.1 node for DDC
    config.vm.define "ucp-nfs-node1" do |ucp_vancouver_node1|
      ucp_vancouver_node1.vm.box = "ubuntu/xenial64"
      ucp_vancouver_node1.vm.network "private_network", ip: "172.28.128.21"
      ucp_vancouver_node1.vm.hostname = "ucp.local"
      config.vm.provider :virtualbox do |vb|
         vb.customize ["modifyvm", :id, "--memory", "2048"]
         vb.customize ["modifyvm", :id, "--cpus", "2"]
         vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
         vb.name = "ucp-nfs-node1"
      end
      ucp_vancouver_node1.vm.provision "shell", inline: <<-SHELL
       sudo apt-get update
       sudo apt-get install -y apt-transport-https ca-certificates ntpdate
       sudo ntpdate -s time.nist.gov
       sudo curl -fsSL https://packages.docker.com/1.13/install.sh | sh
       sudo usermod -aG docker ubuntu
       ifconfig enp0s8 | grep 'inet addr:' | cut -d: -f2 | awk '{ print $1}' > /vagrant/ucp-vancouver-node1-ipaddr
       export UCP_IPADDR=$(cat /vagrant/ucp-vancouver-node1-ipaddr)
       export UCP_PASSWORD=$(cat /vagrant/ucp_password)
       export HUB_USERNAME=$(cat /vagrant/hub_username)
       export HUB_PASSWORD=$(cat /vagrant/hub_password)
       sudo sh -c "echo '${UCP_IPADDR} ucp.local' >> /etc/hosts"
       sudo sh -c "echo '172.28.128.11 dtr.local' >> /etc/hosts"
       docker login -u ${HUB_USERNAME} -p ${HUB_PASSWORD}
       docker pull docker/ucp:2.1.0
       docker run --rm --name ucp -v /var/run/docker.sock:/var/run/docker.sock -v /vagrant/docker_subscription.lic:/docker_subscription.lic docker/ucp:2.1.0 install --host-address ${UCP_IPADDR} --admin-password ${UCP_PASSWORD} --san ucp.local
       docker swarm join-token manager | awk -F " " '/token/ {print $2}' > /vagrant/swarm-join-token-mgr
       docker swarm join-token worker | awk -F " " '/token/ {print $2}' > /vagrant/swarm-join-token-worker
       docker run --rm --name ucp -v /var/run/docker.sock:/var/run/docker.sock docker/ucp:2.1.0 id | awk '{ print $1}' > /vagrant/ucp-vancouver-id
       export UCP_ID=$(cat /vagrant/ucp-vancouver-id)
       docker run --rm -i --name ucp -v /var/run/docker.sock:/var/run/docker.sock docker/ucp:2.1.0 backup --id ${UCP_ID} --root-ca-only --passphrase "secret" > /vagrant/backup.tar
     SHELL
    end

    # DTR Node 1 for DDC setup
    config.vm.define "dtr-nfs-node1" do |dtr_vancouver_node1|
      dtr_vancouver_node1.vm.box = "ubuntu/xenial64"
      dtr_vancouver_node1.vm.network "private_network", ip: "172.28.128.22"
      dtr_vancouver_node1.vm.hostname = "dtr.local"
      config.vm.provider :virtualbox do |vb|
         vb.customize ["modifyvm", :id, "--memory", "2048"]
         vb.customize ["modifyvm", :id, "--cpus", "2"]
         vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
         vb.name = "dtr-nfs-node1"
      end
      dtr_vancouver_node1.vm.provision "shell", inline: <<-SHELL
        sudo apt-get update
        sudo apt-get install -y apt-transport-https ca-certificates ntpdate
        sudo ntpdate -s time.nist.gov
        sudo curl -fsSL https://packages.docker.com/1.13/install.sh | sh
        sudo usermod -aG docker ubuntu
        # Login to Hub
        export HUB_USERNAME=$(cat /vagrant/hub_username)
        export HUB_PASSWORD=$(cat /vagrant/hub_password)
        docker login -u ${HUB_USERNAME} -p ${HUB_PASSWORD}
        # Join UCP Swarm
        ifconfig enp0s8 | grep 'inet addr:' | cut -d: -f2 | awk '{ print $1}' > /vagrant/dtr-vancouver-node1-ipaddr
        cat /dev/urandom | tr -dc 'a-f0-9' | fold -w 12 | head -n 1 > /vagrant/dtr-replica-id
        export UCP_PASSWORD=$(cat /vagrant/ucp_password)
        export UCP_IPADDR=$(cat /vagrant/ucp-vancouver-node1-ipaddr)
        export UCP_URL=https://ucp.local
        export DTR_URL=https://dtr.local
        export DTR_IPADDR=$(cat /vagrant/dtr-vancouver-node1-ipaddr)
        export SWARM_JOIN_TOKEN_WORKER=$(cat /vagrant/swarm-join-token-worker)
        export DTR_REPLICA_ID=$(cat /vagrant/dtr-replica-id)
        sudo sh -c "echo '${UCP_IPADDR} ucp.local' >> /etc/hosts"
        sudo sh -c "echo '${DTR_IPADDR} dtr.local' >> /etc/hosts"
        docker pull docker/ucp:2.1.0
        docker swarm join --token ${SWARM_JOIN_TOKEN_WORKER} ${UCP_IPADDR}:2377
        # Wait for Join to complete
        sleep 30
        # Install DTR
        curl -k https://${UCP_IPADDR}/ca > ucp-ca.pem
        docker run --rm docker/dtr:2.2.1 install --hub-username ${HUB_USERNAME} --hub-password ${HUB_PASSWORD} --ucp-url https://${UCP_IPADDR} --ucp-node dtr --replica-id ${DTR_REPLICA_ID} --dtr-external-url https://${DTR_IPADDR} --ucp-username admin --ucp-password ${UCP_PASSWORD} --ucp-ca "$(cat ucp-ca.pem)"
        # Run backup of DTR
        docker run --rm docker/dtr:2.2.1 backup --ucp-url https://${UCP_IPADDR} --existing-replica-id ${DTR_REPLICA_ID} --ucp-username admin --ucp-password ${UCP_PASSWORD} --ucp-ca "$(cat ucp-ca.pem)" > /tmp/backup.tar
        # Trust self-signed DTR CA
        openssl s_client -connect ${DTR_IPADDR}:443 -showcerts </dev/null 2>/dev/null | openssl x509 -outform PEM | sudo tee /usr/local/share/ca-certificates/${DTR_IPADDR}.crt
        sudo update-ca-certificates
        sudo service docker restart
        # Copy convenience scripts
        sudo cp -r /vagrant/scripts /home/ubuntu/scripts
      SHELL
    end

    # Application Worker Node 1
    config.vm.define "dtr-nfs-node2" do |worker_node1|
      worker_node1.vm.box = "ubuntu/xenial64"
      worker_node1.vm.network "private_network", ip: "172.28.128.23"
      worker_node1.vm.hostname = "dtr-nfs-node2"
      config.vm.provider :virtualbox do |vb|
         vb.customize ["modifyvm", :id, "--memory", "2048"]
         vb.customize ["modifyvm", :id, "--cpus", "2"]
         vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
         vb.name = "dtr-nfs-node2"
      end
      worker_node1.vm.provision "shell", inline: <<-SHELL
       sudo apt-get update
       sudo apt-get install -y apt-transport-https ca-certificates ntpdate
       sudo ntpdate -s time.nist.gov
       sudo curl -fsSL https://packages.docker.com/1.13/install.sh | sh
       sudo usermod -aG docker ubuntu
       ifconfig enp0s8 | grep 'inet addr:' | cut -d: -f2 | awk '{ print $1}' > /vagrant/worker-node1-ipaddr
       export HUB_USERNAME=$(cat /vagrant/hub_username)
       export HUB_PASSWORD=$(cat /vagrant/hub_password)
       docker login -u ${HUB_USERNAME} -p ${HUB_PASSWORD}
       docker pull docker/ucp:2.1.0
       # Join Swarm as worker
       export UCP_IPADDR=$(cat /vagrant/ucp-vancouver-node1-ipaddr)
       export DTR_IPADDR=$(cat /vagrant/dtr-vancouver-node1-ipaddr)
       export SWARM_JOIN_TOKEN_WORKER=$(cat /vagrant/swarm-join-token-worker)
       docker swarm join --token ${SWARM_JOIN_TOKEN_WORKER} ${UCP_IPADDR}:2377
       # Trust self-signed DTR CA
       openssl s_client -connect ${DTR_IPADDR}:443 -showcerts </dev/null 2>/dev/null | openssl x509 -outform PEM | sudo tee /usr/local/share/ca-certificates/${DTR_IPADDR}.crt
       sudo update-ca-certificates
       sudo service docker restart
     SHELL
    end

    # Application Worker Node 2
    config.vm.define "dtr-nfs-node3" do |worker_node2|
      worker_node2.vm.box = "ubuntu/xenial64"
      worker_node2.vm.network "private_network", ip: "172.28.128.24"
      worker_node2.vm.hostname = "dtr-nfs-node3"
      config.vm.provider :virtualbox do |vb|
         vb.customize ["modifyvm", :id, "--memory", "2048"]
         vb.customize ["modifyvm", :id, "--cpus", "2"]
         vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
         vb.name = "dtr-nfs-node3"
      end
      worker_node2.vm.provision "shell", inline: <<-SHELL
       sudo apt-get update
       sudo apt-get install -y apt-transport-https ca-certificates ntpdate
       sudo ntpdate -s time.nist.gov
       sudo curl -fsSL https://packages.docker.com/1.13/install.sh | sh
       sudo usermod -aG docker ubuntu
       ifconfig enp0s8 | grep 'inet addr:' | cut -d: -f2 | awk '{ print $1}' > /vagrant/worker-node1-ipaddr
       export HUB_USERNAME=$(cat /vagrant/hub_username)
       export HUB_PASSWORD=$(cat /vagrant/hub_password)
       docker login -u ${HUB_USERNAME} -p ${HUB_PASSWORD}
       docker pull docker/ucp:2.1.0
       # Join Swarm as worker
       export UCP_IPADDR=$(cat /vagrant/ucp-vancouver-node1-ipaddr)
       export DTR_IPADDR=$(cat /vagrant/dtr-vancouver-node1-ipaddr)
       export SWARM_JOIN_TOKEN_WORKER=$(cat /vagrant/swarm-join-token-worker)
       export WORKER_NODE_NAME=$(hostname)
       docker swarm join --token ${SWARM_JOIN_TOKEN_WORKER} ${UCP_IPADDR}:2377
       # Trust self-signed DTR CA
       openssl s_client -connect ${DTR_IPADDR}:443 -showcerts </dev/null 2>/dev/null | openssl x509 -outform PEM | sudo tee /usr/local/share/ca-certificates/${DTR_IPADDR}.crt
       sudo update-ca-certificates
       sudo service docker restart
       # Install Notary
       curl -L https://github.com/docker/notary/releases/download/v0.4.3/notary-Linux-amd64 > /home/ubuntu/notary
       chmod +x /home/ubuntu/notary
       # Create jenkins folder to store Jenkins container config
       sudo mkdir /home/ubuntu/jenkins
       # Create notary foldoer to store trust config
       sudo mkdir -p /home/ubuntu/notary-config/.docker/trust
     SHELL
    end

    # NFS node for DDC
      config.vm.define "nfs-server-node" do |ucp_vancouver_node1|
        ucp_vancouver_node1.vm.box = "ubuntu/xenial64"
        ucp_vancouver_node1.vm.network "private_network", ip: "172.28.128.20"
        ucp_vancouver_node1.vm.hostname = "ucp.local"
        config.vm.provider :virtualbox do |vb|
           vb.customize ["modifyvm", :id, "--memory", "2048"]
           vb.customize ["modifyvm", :id, "--cpus", "2"]
           vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
           vb.name = "nfs-server-node"
        end
        ucp_vancouver_node1.vm.provision "shell", inline: <<-SHELL
         sudo apt-get update
         sudo apt-get install -y apt-transport-https ca-certificates ntpdate nfs-kernel-server
         sudo ntpdate -s time.nist.gov
         ifconfig enp0s8 | grep 'inet addr:' | cut -d: -f2 | awk '{ print $1}' > /vagrant/ucp-vancouver-node1-ipaddr
         export UCP_IPADDR=$(cat /vagrant/ucp-vancouver-node1-ipaddr)
         export UCP_PASSWORD=$(cat /vagrant/ucp_password)
         export HUB_USERNAME=$(cat /vagrant/hub_username)
         export HUB_PASSWORD=$(cat /vagrant/hub_password)
         sudo sh -c "echo '${UCP_IPADDR} ucp.local' >> /etc/hosts"
         sudo sh -c "echo '172.28.128.11 dtr.local' >> /etc/hosts"
         sudo mkdir /var/nfs/dtr -p
         sudo chown nobody:nogroup /var/nfs/dtr
         sudo sh -c "echo '/var/nfs/jenkins    10.10.4.110(rw,sync,no_subtree_check)' >> /etc/exports"
         sudo service nfs-kernel-server restart
       SHELL
      end

end
