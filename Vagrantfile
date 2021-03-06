# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.require_version ">= 1.6.0"

# Library to pass in parameters
require 'getoptlong'

MOUNT_OPT='--mount'
NETWORK_OPT='--network'
DOCKER_USERNAME_OPT='--docker-username'
DOCKER_PASSWORD_OPT='--docker-password'

cmd_opts = GetoptLong.new(
    # The path on the host that will be mounted on the nodes under /data
    [ MOUNT_OPT, GetoptLong::OPTIONAL_ARGUMENT ],
    # The network driver to user (weave or flannel)
    [ NETWORK_OPT, GetoptLong::OPTIONAL_ARGUMENT ],
    # The dockerhub credentials
    [ DOCKER_USERNAME_OPT, GetoptLong::OPTIONAL_ARGUMENT ],
    [ DOCKER_PASSWORD_OPT, GetoptLong::OPTIONAL_ARGUMENT ]
)

options = {
  #:kubernetes => "latest",
  :kubernetes => "1.15.6",
  :pod_network_cidr => "10.244.0.0/16",
  :kubeadm_token => "54c315.78a320e33baaf27d",
  :host_mount => nil,  
  :guest_mount => "/data",  
  :docker_username => nil,  
  :docker_password => nil,  
  :network => "flannel" # or "flannel" weave
}

cmd_opts.each do |opt, arg|
  case opt
    when MOUNT_OPT
      options[:host_mount]=arg
      puts "Mount local folder #{arg} --> /data"
      when NETWORK_OPT
        options[:network]=arg
        puts "Use Network driver #{arg}"
      when DOCKER_USERNAME_OPT
      options[:docker_username]=arg
    when DOCKER_PASSWORD_OPT
      options[:docker_password]=arg
  end
end

boxes = [
    {
        :name => "k8smaster",
        :eth1 => "192.168.8.10",
        :mem => "2048",
        :cpu => "2",
        :is_master => true
    },
    {
        :name => "k8sworker1",
        :eth1 => "192.168.8.11",
        :mem => "4096",
        :cpu => "2"
    },
    {
       :name => "k8sworker2",
       :eth1 => "192.168.8.12",
       :mem => "2048",
       :cpu => "2"
    }
]

Vagrant.configure("2") do |config|

  config.vm.box = "ubuntu/bionic64"

  # Validate the nodes
  masterCount = boxes.select { |box| box[:is_master] }.count 
  raise "No master defined in the boxes" if masterCount == 0
  raise "You must only have one master" if masterCount > 1

  # Configure the nodes
  boxes.each do |opts|
    config.vm.define opts[:name] do |config|
      config.vm.hostname = opts[:name]

      config.vm.provider "virtualbox" do |v|
        v.customize ["modifyvm", :id, "--memory", opts[:mem]]
        v.customize ["modifyvm", :id, "--cpus", opts[:cpu]]
      end

      config.vm.network :private_network, ip: opts[:eth1]
    end
  end

  # Install the nodes
  boxes.each do |box|
    config.vm.define box[:name], primary: box[:is_master] == true do |node|
      node.ssh.forward_agent = true
      # Mount an additional shared folder if specified as a command-line argument
      if options[:host_mount]
        node.vm.synced_folder "#{options[:host_mount]}", "#{options[:guest_mount]}"
      end
      # setup the node with kubernetes requirements
      node.vm.provision "shell", path: "./scripts/setup-node.sh", args: [options[:kubernetes], box[:name], box[:eth1]]

      # setup the node depending on its role: master or worker
      if box[:is_master]
        node.vm.provision "shell", args: [options[:kubernetes]], inline: <<-SHELL
          set -e -x
          KUBE_VERSION=$1
          KUBEADM_VERSION=
          # Create the master node
          #kubeadm init --apiserver-advertise-address #{box[:eth1]} --pod-network-cidr #{options[:pod_network_cidr]} --kubernetes-version #{options[:kubernetes]} --token #{options[:kubeadm_token]}
          if [ "$KUBE_VERSION" != "latest" ]; then
            KUBEADM_VERSION="--kubernetes-version v$KUBE_VERSION"
          fi
          ## kubeadm init --pod-network-cidr #{options[:pod_network_cidr]} --apiserver-advertise-address #{box[:eth1]} 
          kubeadm init --apiserver-advertise-address #{box[:eth1]} --pod-network-cidr #{options[:pod_network_cidr]} --token #{options[:kubeadm_token]} $KUBEADM_VERSION
          # Copy Kube config into our shared Vagrant folder
          cp -rf  /etc/kubernetes/admin.conf /vagrant/kubeconfig/      
          apt-get install -y nfs-common nfs-kernel-server
        SHELL
      else # it is a worker
        master = boxes.select { |box| box[:is_master] }.first
        raise "Could not find master box" if master == nil

        node.vm.provision "shell", inline: <<-SHELL
          set -e -x
          # Add a worker node to the cluster
          apt-get install -y nfs-common
          kubeadm join  --discovery-token-unsafe-skip-ca-verification --token #{options[:kubeadm_token]} #{master[:eth1]}:6443 
        SHELL
      end

      # if the user provided its credentials for his DockerHub account, then do the login for each node.
      if options[:docker_username] && options[:docker_password] then
        node.vm.provision "shell", env: {"USERNAME" => options[:docker_username], "PASSWORD" => options[:docker_password]}, inline: <<-SHELL
          set -e
          echo "Log into Dockerhub with user $USERNAME"
          docker login -u $USERNAME -p $PASSWORD
        SHELL
      end
       
      # Run post install script only in the last box
      isLastBox = boxes.last[:name] == box[:name]
      if isLastBox
        node.vm.provision "shell", path: "./scripts/post-install.sh",  args: [options[:network]]
      end
    end
  end  
end
