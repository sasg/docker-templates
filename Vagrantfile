# -*- mode: ruby -*-
# # vi: set ft=ruby :

# Commands required to setup working docker enviro, link containers etc.
$setup = <<SCRIPT

# Stop and remove any existing containers
#docker stop $(docker ps -a -q)
#docker rm $(docker ps -a -q)

# Build containers from Dockerfiles
#docker pull launchd/postgres
#docker pull launchd/rails
#docker pull launchd/phppgadmin
#docker pull launchd/ngircd
#docker pull launchd/kiwiirc

# run Postgress
#docker run -d --name postgres launchd/postgres
#docker run -d --name phppgadmin --link postgres:db -p 80:80 launchd/phppgadmin

# run IRC
#docker run -d --name ngircd -p 6667:6667 -p 6697:6697 launchd/ngircd
#docker run -d --name kiwiirc --link ngircd:irc  -p 7778:7778 launchd/kiwiirc

#docker run -d --name rails --link postgres:db -v /app:/app -p 3000:3000 launchd/rails
# to run the rails container with an interactive shell, use the following instead
#docker run -i -t --name rails --link postgres:db -v /app:/app -p 3000:3000 launchd/rails /bin/bash -l


SCRIPT

# Commands required to ensure correct docker containers are started when the vm is rebooted.
$start = <<SCRIPT

#docker start postgres
#docker start rails
#docker start phppgadmin
#docker start ngircd
#docker start kiwiirc

SCRIPT


# See https://coreos.com/docs/running-coreos/platforms/vagrant/#single-machine

require 'fileutils'
Vagrant.require_version ">= 1.6.0"

CLOUD_CONFIG_PATH = File.join(File.dirname(__FILE__), "user-data")
CONFIG = File.join(File.dirname(__FILE__), "config.rb")

# Defaults for config options defined in CONFIG
$num_instances = 1
$update_channel = "alpha"
$enable_serial_logging = false
$vb_gui = false
$vb_memory = 1024
$vb_cpus = 1

# Attempt to apply the deprecated environment variable NUM_INSTANCES to
# $num_instances while allowing config.rb to override it
if ENV["NUM_INSTANCES"].to_i > 0 && ENV["NUM_INSTANCES"]
  $num_instances = ENV["NUM_INSTANCES"].to_i
end

if File.exist?(CONFIG)
  require CONFIG
end

Vagrant.configure("2") do |config|
  config.vm.box = "coreos-%s" % $update_channel
  config.vm.box_version = ">= 308.0.1"
  config.vm.box_url = "http://%s.release.core-os.net/amd64-usr/current/coreos_production_vagrant.json" % $update_channel

  config.vm.provider :vmware_fusion do |vb, override|
    override.vm.box_url = "http://%s.release.core-os.net/amd64-usr/current/coreos_production_vagrant_vmware_fusion.json" % $update_channel
  end

  config.vm.provider :virtualbox do |v|
    # On VirtualBox, we don't have guest additions or a functional vboxsf
    # in CoreOS, so tell Vagrant that so it can be smarter.
    v.check_guest_additions = false
    v.functional_vboxsf     = false
  end

  # plugin conflict
  if Vagrant.has_plugin?("vagrant-vbguest") then
    config.vbguest.auto_update = false
  end

  (1..$num_instances).each do |i|
    config.vm.define vm_name = "core-%02d" % i do |config|
      config.vm.hostname = vm_name

      if $enable_serial_logging
        logdir = File.join(File.dirname(__FILE__), "log")
        FileUtils.mkdir_p(logdir)

        serialFile = File.join(logdir, "%s-serial.txt" % vm_name)
        FileUtils.touch(serialFile)

        config.vm.provider :vmware_fusion do |v, override|
          v.vmx["serial0.present"] = "TRUE"
          v.vmx["serial0.fileType"] = "file"
          v.vmx["serial0.fileName"] = serialFile
          v.vmx["serial0.tryNoRxLoss"] = "FALSE"
        end

        config.vm.provider :virtualbox do |vb, override|
          vb.customize ["modifyvm", :id, "--uart1", "0x3F8", "4"]
          vb.customize ["modifyvm", :id, "--uartmode1", serialFile]
        end
      end

      if $expose_docker_tcp
        config.vm.network "forwarded_port", guest: 2375, host: ($expose_docker_tcp + i - 1), auto_correct: true
      end

      config.vm.provider :vmware_fusion do |vb|
        vb.gui = $vb_gui
      end

      config.vm.provider :virtualbox do |vb|
        vb.gui = $vb_gui
        vb.memory = $vb_memory
        vb.cpus = $vb_cpus
      end

      ip = "10.1.0.#{i+100}"
      config.vm.network :private_network, ip: ip

      # Sharing the app dir into the coreos-vagrant VM.
      config.vm.synced_folder ".", "/app", id: "core", :nfs => true, :mount_options => ['nolock,vers=3,udp']

      if File.exist?(CLOUD_CONFIG_PATH)
        config.vm.provision :file, :source => "#{CLOUD_CONFIG_PATH}", :destination => "/tmp/vagrantfile-user-data"
        config.vm.provision :shell, :inline => "mv /tmp/vagrantfile-user-data /var/lib/coreos-vagrant/", :privileged => true
      end

      # See https://github.com/TalkingQuickly/docker_rails_dev_env/blob/master/Vagrantfile

      # [rails] Port Forwarding
      config.vm.network "forwarded_port", guest: 3000, host: 3000

      # [phppgadmin] Port Forwarding
      config.vm.network "forwarded_port", guest: 80, host: 3001

      # [ngircd] Port Forwarding (only SSL)
      config.vm.network "forwarded_port", guest: 6697, host: 6697

      # [kiwiirc] Port Forwarding (only SSL)
      config.vm.network "forwarded_port", guest: 7778, host: 7778

      # Setup the containers when the VM is first created
      config.vm.provision "shell", inline: $setup

      # Make sure the correct containers are running every time we start the VM.
      # Note: commented out since autostart was not working reliably
      config.vm.provision "shell", run: "always", inline: $start

    end
  end
end
