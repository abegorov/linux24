# -*- mode: ruby -*-
# vi: set ft=ruby :

BOX = 'debian/bookworm64/v12.20240905.1'
DOMAIN = 'internal'
MACHINES = {
  :'web' => { :cpus => 1, :memory => 1024, :networks => [
    [:private_network, {:ip => '192.168.56.10'}]
  ] },
  :'log' => { :cpus => 1, :memory => 1024, :networks => [
    [:private_network, {:ip => '192.168.56.15'}]
  ] },
  :'elk' => { :cpus => 2, :memory => 2048, :networks => [
    [:private_network, {:ip => '192.168.56.20'}]
  ] }
}

Vagrant.configure("2") do |config|
  MACHINES.each do |host_name, host_config|
    config.vm.define host_name do |host|
      host.vm.box = BOX
      host.vm.host_name = host_name.to_s + '.' + DOMAIN

      host.vm.provider :virtualbox do |vb|
        vb.cpus = host_config[:cpus]
        vb.memory = host_config[:memory]
      end

      host_config[:networks].each do |network|
        host.vm.network(network[0], **network[1])
      end

      if MACHINES.keys.last == host_name
        host.vm.provision :ansible do |ansible|
          ansible.playbook = 'playbook.yml'
          ansible.limit = 'all'
          ansible.compatibility_mode = '2.0'
          ansible.raw_arguments = ['--diff']
        end
      end
    end
  end
end
