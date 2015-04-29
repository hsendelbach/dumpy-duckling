Vagrant.configure('2') do |config|
  config.vm.box = "ubuntu/trusty64"

  config.vm.define :devstack do |devstack|
    devstack.vm.hostname = "devstack"
    devstack.vm.network :private_network, ip: "192.168.123.10"
    devstack.vm.network :public_network, dev: 'br0', mode: 'bridge', ip: "192.168.11.197"

    devstack.vm.synced_folder ".", "/vagrant", type: "nfs"

    devstack.vm.provider :libvirt do |libvirt, override|
      libvirt.memory = 8192
      libvirt.nested = true
    end
    
    # Add some post install here.
    #devstack.vm.provision "shell", path: "./install.sh"
  end

end
