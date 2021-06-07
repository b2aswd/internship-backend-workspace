# -*- mode: ruby -*-
# vi: set ft=ruby :

# ######################################################### #
#
# Includes:
# - Basic tools
# - Fish shell
# - Composer
# - Docker & docker-compose
#
# Author: Jan Kotrba <jan.kotrba@b2a.cz>
#
# Note for Windows : vagrant plugin install vagrant-winnfsd
#
# ######################################################### #

["vagrant-vbguest", "vagrant-disksize"].each do |plugin|
  unless Vagrant.has_plugin?(plugin)
    raise plugin + ' plugin is not installed. Hint: vagrant plugin install ' + plugin
  end
end

Vagrant.configure("2") do |config|

  # Specify debian/buster64 as OS
  config.vm.box = "generic/debian10"

  # Configure VM name in VirtualBox provider
  config.vm.provider "virtualbox" do |vb|
    vb.name = "b2a_staze_workspace"
  end

  # Configure private network, VM will be accessible at this ip address
  config.vm.network "private_network", ip: "172.32.32.32"

  # Disable default synced folder
  config.vm.synced_folder ".", "/vagrant", disabled: true

  config.vm.synced_folder 'projects/', "/var/www/projects",
    create: true,
    type: "nfs",
    mount_options: %w{rw,async,fsc,nolock,vers=3,udp,rsize=32768,wsize=32768,hard,noatime,actimeo=2}

  # Configure VM settings in VirtualBox provider
  config.vm.provider "virtualbox" do |vb|

    # Do not display the VirtualBox GUI when booting the machine
    vb.gui = false

    # Customize the amount of RAM
    vb.memory = "2048"

    # Customize the amount of CPU cores
    vb.cpus = "2"

  end

  # Use 64 GB disk
  config.disksize.size = '64GB'

  # Provision
  config.vm.provision "shell", inline: <<-SHELL

    export DEBIAN_FRONTEND=noninteractive

    apt-get update && apt-get upgrade -y

    # Configure timezone
    ln -fs /usr/share/zoneinfo/Europe/Prague /etc/localtime
    dpkg-reconfigure --frontend noninteractive tzdata

    # Install and configure locales
    apt-get install -y locales locales-all

    # Generate locales
    echo "en_US.UTF-8 UTF-8" > /etc/locale.gen && locale-gen

    # Install basic tools
    apt-get install -y \
      wget \
      curl \
      git \
      htop \
      zip \
      unzip \
      ssh \
      fish \
      nano

    # Install tools required for installing packages from different sources
    apt-get install -y \
      apt-transport-https \
      ca-certificates \
      software-properties-common \
      lsb-release

    # Install package sources for docker and install
    if [ ! -f /etc/apt/sources.list.d/docker.list ]; then
      echo "deb [arch=amd64] https://download.docker.com/linux/debian $(lsb_release -cs) stable" | sudo tee -a /etc/apt/sources.list.d/docker.list
      wget -nv https://download.docker.com/linux/debian/gpg -O /tmp/Docker-Release.key
      apt-key add - < /tmp/Docker-Release.key
      apt-get update
      apt-get install -y docker-ce

      # Setup groups
      usermod -aG docker vagrant
      usermod -aG www-data vagrant

      systemctl disable docker
      systemctl stop docker

      # Expose Docker Daemon API over TCP
      mkdir -p /etc/systemd/system/docker.service.d
      echo "[Service]" > /etc/systemd/system/docker.service.d/override.conf
      echo "ExecStart=" >> /etc/systemd/system/docker.service.d/override.conf
      echo "ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2375 -H fd:// --containerd=/run/containerd/containerd.sock" >> /etc/systemd/system/docker.service.d/override.conf

      systemctl daemon-reload
      systemctl enable docker
      systemctl restart docker

    fi

    # Install docker-compose
    if [ ! -f /usr/local/bin/docker-compose ]; then
      curl -s -L "https://github.com/docker/compose/releases/download/1.24.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
      chmod +x /usr/local/bin/docker-compose
    fi

    if [[ ! $(grep "XDEBUG_CONFIG" /etc/environment) ]] ; then
      echo 'XDEBUG_CONFIG=remote_host=10.0.2.2' >> /etc/environment
    fi

    # Create vagrant home directory
    mkdir -p /home/vagrant
    chown vagrant:vagrant -R /home/vagrant

    apt-get update && apt-get upgrade -y && apt-get autoremove -y

    # Force git to refresh index
    git config --global core.preloadIndex true

    # Force git to use fast-forward merge by defualt
    git config --global pull.ff only

  SHELL

  # Provision
  config.vm.provision "shell", privileged: false, inline: <<-SHELL

    export DEBIAN_FRONTEND=noninteractive

    # Create core home directories
    mkdir -p /home/vagrant/.ssh
    mkdir -p /home/vagrant/.ssh/docker
    mkdir -p /home/vagrant/.composer

    # Touch known_hosts
    touch /home/vagrant/.ssh/known_hosts

    # Generate id_rsa if not already generated
    if [ ! -f /home/vagrant/.ssh/id_rsa ]; then
      ssh-keygen -b 2048 -t rsa -f /home/vagrant/.ssh/id_rsa -q -N "" -C "vagrant@debian10"
      ln /home/vagrant/.ssh/id_rsa /home/vagrant/.ssh/docker/id_rsa
      ln /home/vagrant/.ssh/id_rsa.pub /home/vagrant/.ssh/docker/id_rsa.pub
      ln /home/vagrant/.ssh/known_hosts /home/vagrant/.ssh/docker/known_hosts
    fi

    # Force git to refresh index
    git config --global core.preloadIndex true

  SHELL

end