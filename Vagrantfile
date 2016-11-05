# -*- mode: ruby -*-
# vi: set ft=ruby :

# A dummy plugin for Barge to set hostname and network correctly at the very first `vagrant up`
module VagrantPlugins
  module GuestLinux
    class Plugin < Vagrant.plugin("2")
      guest_capability("linux", "change_host_name") { Cap::ChangeHostName }
      guest_capability("linux", "configure_networks") { Cap::ConfigureNetworks }
    end
  end
end

VERSION="2.2.7"

Vagrant.configure(2) do |config|
  config.vm.define "yapc-barge"

  config.vm.box = "ailispaw/barge"
  config.vm.box_version = "#{VERSION}.2"

  config.vm.synced_folder ".", "/vagrant"

  config.vm.provision "shell", inline: <<-SHELL
    mkdir -p /opt/pkg/#{VERSION}/
    cp /vagrant/pkg/barge-pkg-*-#{VERSION}.tar.gz /opt/pkg/#{VERSION}/

    pkg install iproute2
    pkg install libcgroup -e BR2_PACKAGE_LIBCGROUP_TOOLS=y
    pkg install libcap -e BR2_PACKAGE_LIBCAP_TOOLS=y

    mkdir -p /home/bargee/centos
    docker export $(docker create centos) | tar xf - -C /home/bargee/centos
  SHELL
end
