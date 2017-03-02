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
    config.vm.define "ucp-nfs-node1" do |ucp_nfs_node1|
      ucp_nfs_node1.vm.box = "ubuntu/xenial64"
      ucp_nfs_node1.vm.network "private_network", ip: "172.28.128.21"
      ucp_nfs_node1.vm.hostname = "ucp-nfs-node1"
      config.vm.provider :virtualbox do |vb|
         vb.customize ["modifyvm", :id, "--memory", "2048"]
         vb.customize ["modifyvm", :id, "--cpus", "2"]
         vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
         vb.name = "ucp-nfs-node1"
      end
      ucp_nfs_node1.vm.provision "shell", inline: <<-SHELL
       sudo apt-get update
       sudo apt-get install -y apt-transport-https ca-certificates ntpdate nfs-common
       sudo ntpdate -s time.nist.gov
       sudo curl -fsSL https://packages.docker.com/1.13/install.sh | sh
       sudo usermod -aG docker ubuntu
       ifconfig enp0s8 | grep 'inet addr:' | cut -d: -f2 | awk '{ print $1}' > /vagrant/ucp-nfs-node1
       export UCP_IPADDR=$(cat /vagrant/ucp-nfs-node1)
       export UCP_PASSWORD=$(cat /vagrant/ucp_password)
       export HUB_USERNAME=$(cat /vagrant/hub_username)
       export HUB_PASSWORD=$(cat /vagrant/hub_password)
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

    # DTR Node 1 for DDC
    config.vm.define "dtr-nfs-node1" do |dtr_nfs_node1|
      dtr_nfs_node1.vm.box = "ubuntu/xenial64"
      dtr_nfs_node1.vm.network "private_network", ip: "172.28.128.22"
      dtr_nfs_node1.vm.hostname = "dtr-nfs-node1"
      config.vm.provider :virtualbox do |vb|
         vb.customize ["modifyvm", :id, "--memory", "2048"]
         vb.customize ["modifyvm", :id, "--cpus", "2"]
         vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
         vb.name = "dtr-nfs-node1"
      end
      dtr_nfs_node1.vm.provision "shell", inline: <<-SHELL
        sudo apt-get update
        sudo apt-get install -y apt-transport-https ca-certificates ntpdate nfs-common
        sudo ntpdate -s time.nist.gov
        sudo curl -fsSL https://packages.docker.com/1.13/install.sh | sh
        sudo usermod -aG docker ubuntu
        # Login to Hub
        export HUB_USERNAME=$(cat /vagrant/hub_username)
        export HUB_PASSWORD=$(cat /vagrant/hub_password)
        docker login -u ${HUB_USERNAME} -p ${HUB_PASSWORD}
        # Join UCP Swarm
        ifconfig enp0s8 | grep 'inet addr:' | cut -d: -f2 | awk '{ print $1}' > /vagrant/dtr-nfs-node1
        cat /dev/urandom | tr -dc 'a-f0-9' | fold -w 12 | head -n 1 > /vagrant/dtr-node1-replica-id
        export UCP_PASSWORD=$(cat /vagrant/ucp_password)
        export UCP_IPADDR=$(cat /vagrant/ucp-nfs-node1)
        export UCP_URL=https://ucp.local
        export DTR_URL=https://dtr.local
        export DTR_IPADDR=$(cat /vagrant/dtr-nfs-node1)
        export SWARM_JOIN_TOKEN_WORKER=$(cat /vagrant/swarm-join-token-worker)
        export WORKER_NODE_NAME=$(hostname)
        export DTR_REPLICA_ID=$(cat /vagrant/dtr-node1-replica-id)
        export NFS_IPADDR=172.28.128.20
        docker pull docker/ucp:2.1.0
        docker swarm join --token ${SWARM_JOIN_TOKEN_WORKER} ${UCP_IPADDR}:2377
        # Wait for Join to complete
        sleep 30
        # Install DTR
        curl -k https://${UCP_IPADDR}/ca > ucp-ca.pem
        docker run --rm docker/dtr:2.2.2 install --hub-username ${HUB_USERNAME} --hub-password ${HUB_PASSWORD} --ucp-url https://${UCP_IPADDR} --ucp-node ${WORKER_NODE_NAME} --replica-id ${DTR_REPLICA_ID} --dtr-external-url https://${DTR_IPADDR} --nfs-storage-url nfs://${NFS_IPADDR}/var/nfs/dtr --ucp-username admin --ucp-password ${UCP_PASSWORD} --ucp-ca "$(cat ucp-ca.pem)"
        # Run backup of DTR
        docker run --rm docker/dtr:2.2.2 backup --ucp-url https://${UCP_IPADDR} --existing-replica-id ${DTR_REPLICA_ID} --ucp-username admin --ucp-password ${UCP_PASSWORD} --ucp-ca "$(cat ucp-ca.pem)" > /tmp/backup.tar
        # Trust self-signed DTR CA
        openssl s_client -connect ${DTR_IPADDR}:443 -showcerts </dev/null 2>/dev/null | openssl x509 -outform PEM | sudo tee /usr/local/share/ca-certificates/${DTR_IPADDR}.crt
        sudo update-ca-certificates
        sudo service docker restart
        # Copy convenience scripts
        sudo cp -r /vagrant/scripts /home/ubuntu/scripts
      SHELL
    end

    # DTR Node 2 for DDC
    config.vm.define "dtr-nfs-node2" do |dtr_nfs_node2|
      dtr_nfs_node2.vm.box = "ubuntu/xenial64"
      dtr_nfs_node2.vm.network "private_network", ip: "172.28.128.23"
      dtr_nfs_node2.vm.hostname = "dtr-nfs-node2"
      config.vm.provider :virtualbox do |vb|
         vb.customize ["modifyvm", :id, "--memory", "2048"]
         vb.customize ["modifyvm", :id, "--cpus", "2"]
         vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
         vb.name = "dtr-nfs-node2"
      end
      dtr_nfs_node2.vm.provision "shell", inline: <<-SHELL
       sudo apt-get update
       sudo apt-get install -y apt-transport-https ca-certificates ntpdate nfs-common
       sudo ntpdate -s time.nist.gov
       sudo curl -fsSL https://packages.docker.com/1.13/install.sh | sh
       sudo usermod -aG docker ubuntu
       ifconfig enp0s8 | grep 'inet addr:' | cut -d: -f2 | awk '{ print $1}' > /vagrant/dtr-nfs-node2
       cat /dev/urandom | tr -dc 'a-f0-9' | fold -w 12 | head -n 1 > /vagrant/dtr-node2-replica-id
       export HUB_USERNAME=$(cat /vagrant/hub_username)
       export HUB_PASSWORD=$(cat /vagrant/hub_password)
       docker login -u ${HUB_USERNAME} -p ${HUB_PASSWORD}
       docker pull docker/ucp:2.1.0
       # Join Swarm as worker
       export UCP_IPADDR=$(cat /vagrant/ucp-nfs-node1)
       export DTR_IPADDR=$(cat /vagrant/dtr-nfs-node2)
       export UCP_PASSWORD=$(cat /vagrant/ucp_password)
       export SWARM_JOIN_TOKEN_WORKER=$(cat /vagrant/swarm-join-token-worker)
       export WORKER_NODE_NAME=$(hostname)
       export DTR_REPLICA_ID=$(cat /vagrant/dtr-node2-replica-id)
       docker swarm join --token ${SWARM_JOIN_TOKEN_WORKER} ${UCP_IPADDR}:2377
       # Join DTR as a replica
       curl -k https://${UCP_IPADDR}/ca > ucp-ca.pem
       docker run -it --rm docker/dtr:2.2.2 join --replica-id ${DTR_REPLICA_ID} --ucp-url https://${UCP_IPADDR} --ucp-node ${WORKER_NODE_NAME} --ucp-username admin --ucp-password ${UCP_PASSWORD} --ucp-ca "$(cat ucp-ca.pem)"
       # Run backup of DTR
       docker run --rm docker/dtr:2.2.2 backup --ucp-url https://${UCP_IPADDR} --existing-replica-id ${DTR_REPLICA_ID} --ucp-username admin --ucp-password ${UCP_PASSWORD} --ucp-ca "$(cat ucp-ca.pem)" > /tmp/backup.tar
       # Trust self-signed DTR CA
       openssl s_client -connect ${DTR_IPADDR}:443 -showcerts </dev/null 2>/dev/null | openssl x509 -outform PEM | sudo tee /usr/local/share/ca-certificates/${DTR_IPADDR}.crt
       sudo update-ca-certificates
       sudo service docker restart
     SHELL
    end

    # DTR Node 3 for DDC
    config.vm.define "dtr-nfs-node3" do |dtr_nfs_node3|
      dtr_nfs_node3.vm.box = "ubuntu/xenial64"
      dtr_nfs_node3.vm.network "private_network", ip: "172.28.128.24"
      dtr_nfs_node3.vm.hostname = "dtr-nfs-node3"
      config.vm.provider :virtualbox do |vb|
         vb.customize ["modifyvm", :id, "--memory", "2048"]
         vb.customize ["modifyvm", :id, "--cpus", "2"]
         vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
         vb.name = "dtr-nfs-node3"
      end
      dtr_nfs_node3.vm.provision "shell", inline: <<-SHELL
       sudo apt-get update
       sudo apt-get install -y apt-transport-https ca-certificates ntpdate nfs-common
       sudo ntpdate -s time.nist.gov
       sudo curl -fsSL https://packages.docker.com/1.13/install.sh | sh
       sudo usermod -aG docker ubuntu
       ifconfig enp0s8 | grep 'inet addr:' | cut -d: -f2 | awk '{ print $1}' > /vagrant/dtr-nfs-node3
       cat /dev/urandom | tr -dc 'a-f0-9' | fold -w 12 | head -n 1 > /vagrant/dtr-node3-replica-id
       export HUB_USERNAME=$(cat /vagrant/hub_username)
       export HUB_PASSWORD=$(cat /vagrant/hub_password)
       docker login -u ${HUB_USERNAME} -p ${HUB_PASSWORD}
       docker pull docker/ucp:2.1.0
       # Join Swarm as worker
       export UCP_IPADDR=$(cat /vagrant/ucp-nfs-node1)
       export DTR_IPADDR=$(cat /vagrant/dtr-nfs-node3)
       export SWARM_JOIN_TOKEN_WORKER=$(cat /vagrant/swarm-join-token-worker)
       export WORKER_NODE_NAME=$(hostname)
       export DTR_REPLICA_ID=$(cat /vagrant/dtr-node3-replica-id)
       docker swarm join --token ${SWARM_JOIN_TOKEN_WORKER} ${UCP_IPADDR}:2377
       # Join DTR as a replica
       curl -k https://${UCP_IPADDR}/ca > ucp-ca.pem
       docker run -it --rm docker/dtr:2.2.2 join --replica-id ${DTR_REPLICA_ID} --ucp-url https://${UCP_IPADDR} --ucp-node ${WORKER_NODE_NAME} --ucp-username admin --ucp-password ${UCP_PASSWORD} --ucp-ca "$(cat ucp-ca.pem)"
       # Trust self-signed DTR CA
       openssl s_client -connect ${DTR_IPADDR}:443 -showcerts </dev/null 2>/dev/null | openssl x509 -outform PEM | sudo tee /usr/local/share/ca-certificates/${DTR_IPADDR}.crt
       sudo update-ca-certificates
       sudo service docker restart
     SHELL
    end

    # NFS node for DDC
      config.vm.define "nfs-server-node" do |nfs_server_node1|
        nfs_server_node1.vm.box = "ubuntu/xenial64"
        nfs_server_node1.vm.network "private_network", ip: "172.28.128.20"
        nfs_server_node1.vm.hostname = "ucp.local"
        config.vm.provider :virtualbox do |vb|
           vb.customize ["modifyvm", :id, "--memory", "2048"]
           vb.customize ["modifyvm", :id, "--cpus", "2"]
           vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
           vb.name = "nfs-server-node"
        end
        nfs_server_node1.vm.provision "shell", inline: <<-SHELL
         sudo apt-get update
         sudo apt-get install -y apt-transport-https ca-certificates ntpdate nfs-kernel-server
         sudo ntpdate -s time.nist.gov
         ifconfig enp0s8 | grep 'inet addr:' | cut -d: -f2 | awk '{ print $1}' > /vagrant/nfs-server-node
         export UCP_IPADDR=$(cat /vagrant/ucp-nfs-node1)
         export UCP_PASSWORD=$(cat /vagrant/ucp_password)
         export HUB_USERNAME=$(cat /vagrant/hub_username)
         export HUB_PASSWORD=$(cat /vagrant/hub_password)
         export DTR_NODE1_IPADDR=172.28.128.22
         export DTR_NODE2_IPADDR=172.28.128.23
         export DTR_NODE3_IPADDR=172.28.128.24
         sudo mkdir /var/nfs/dtr -p
         sudo chown nobody:nogroup /var/nfs/dtr
         sudo sh -c "echo '/var/nfs/dtr    ${DTR_NODE1_IPADDR}(rw,sync,no_subtree_check)  ${DTR_NODE2_IPADDR}(rw,sync,no_subtree_check) ${DTR_NODE3_IPADDR}(rw,sync,no_subtree_check)' >> /etc/exports"
         sudo service nfs-kernel-server restart
       SHELL
      end

end
