require "net/ssh"

module Net::SSH
  class << self
    alias_method :old_start, :start
    
    def start(host, username, opts)
      opts[:keys_only] = false
      self.old_start(host, username, opts)
    end
  end
end 

Vagrant.configure("2") do |config|
  shome=File.expand_path("..", __FILE__)

  nm_box=ENV['BOX_NAME']

  cibuild_script = %x{which block-cibuild 2>/dev/null}.strip
  cibuild_args = [ ENV['BASEBOX_HOME_URL'] ]

  brbuild_script = "#{shome}/script/docker-bootstrap"
  brbuild_args = [ ENV['BASEBOX_DOCKER_NETWORK_PREFIX'] ]

  %w(http_proxy ssh_gateway ssh_gateway_user).each {|ele|
    unless ENV[ele].nil? || ENV[ele].empty?
      cibuild_args << ENV[ele]
    end
  }

  config.ssh.username = "ubuntu"
  config.ssh.forward_agent = true
  config.ssh.insert_key = false

  config.vm.synced_folder ENV['BASEBOX_CACHE'], '/vagrant'
  config.vm.synced_folder "#{ENV['BASEBOX_CACHE']}/tmp/packer", '/vagrant/tmp/packer'

  ssh_keys = [
    "#{ENV['BASEBOX_CACHE']}/.ssh/ssh-vagrant",
    "#{shome}/.ssh/ssh-vagrant-insecure"
  ]
  
  config.vm.define "osx" do |region|
    region.vm.box = ENV['BASEBOX_NAME']
    region.vm.provision "shell", path: cibuild_script, args: cibuild_args, privileged: false

    region.vm.provider "vmware_fusion" do |v|
      v.gui = false
      v.linked_clone = true
      v.verify_vmnet = true
      v.vmx["memsize"] = "4096"
      v.vmx["numvcpus"] = "2"

      v.vmx["ethernet0.present"] = "TRUE"
      v.vmx["ethernet0.connectionType"] = "nat"
    end
  end # end vmware config

  config.vm.define ENV['BASEBOX_NAME'] do |region|
    region.vm.box = ENV['BASEBOX_NAME']
    region.vm.provision "shell", path: cibuild_script, args: cibuild_args, privileged: false

    region.ssh.private_key_path = ssh_keys

    region.vm.network "private_network", ip: ENV['BASEBOX_IP']

    region.vm.provider "virtualbox" do |v, override|
      override.vm.provision "shell", path: brbuild_script, args: brbuild_args, privileged: false
      v.linked_clone = true
      v.memory = 4096
      v.cpus = 2

      unless File.exists?("#{ENV['LIMBO_HOME']}/cidata.iso")
        Dir.chdir(ENV['LIMBO_HOME']) do
          system("make")
        end
      end

      v.customize [ 
        'storageattach', :id, 
        '--storagectl', 'SATA Controller', 
        '--port', 1, 
        '--device', 0, 
        '--type', 'dvddrive', 
        '--medium', "#{ENV['LIMBO_HOME']}/cidata.iso"
      ]
    end
  end # end virtualbox config

  config.vm.define "#{ENV['BASEBOX_NAME']}-aws" do |region|
    region.vm.box = "#{ENV['BASEBOX_NAME']}"
    region.vm.provision "shell", path: cibuild_script, args: cibuild_args, privileged: false

    region.ssh.private_key_path = ssh_keys

    region.vm.provider "aws" do |v, override|
      override.vm.provision "shell", path: brbuild_script, args: brbuild_args, privileged: false

      v.keypair_name = "vagrant-#{Digest::MD5.file(ssh_keys[0]).hexdigest}"
      v.instance_type = 't2.medium'
      v.region = %x{configure get region}.chomp
      v.block_device_mapping = [
        { 'DeviceName' => '/dev/sda1', 'Ebs.VolumeSize' => 100 },
        { 'DeviceName' => '/dev/sdb', 'VirtualName' => 'ephemeral0', },
        { 'DeviceName' => '/dev/sdc', 'VirtualName' => 'ephemeral1', },
        { 'DeviceName' => '/dev/sdd', 'VirtualName' => 'ephemeral2', },
        { 'DeviceName' => '/dev/sde', 'VirtualName' => 'ephemeral3', }
      ]
    end
  end # end aws config

  ([''] + (0..99).to_a).each do |nm_region|
  config.vm.define "#{nm_box}#{nm_region}" do |region|
    region.ssh.private_key_path = ssh_keys

    region.vm.provider "docker" do |v, override|
      v.create_args = []
      v.volumes = []
      v.cmd = [ "/usr/sbin/sshd", "-D", "-o", "VersionAddendum=#{nm_box}#{nm_region}" ]

      if nm_region == ''
        region.vm.provision "shell", path: cibuild_script, args: cibuild_args, privileged: false
        v.volumes = [ "/var/run/sshd" ]
        v.image = ENV['BASEBOX_SOURCE'] || "#{ENV['BASEBOX_NAME']}:packer"
      else
        v.image = ENV['BASEBOX_SOURCE'] || "#{ENV['BASEBOX_NAME']}:vagrant"
      end
      
      v.has_ssh = true
      
      module VagrantPlugins
        module DockerProvider
          class Provider < Vagrant.plugin("2", :provider)
            def host_vm?
              false
            end
          end
          module Action
            class Create
              def forwarded_ports(include_ssh=false)
                return []
              end
            end
          end
        end
      end # end docker monkey patching
    end
  end # end docker config
  end # end docker configs
end
