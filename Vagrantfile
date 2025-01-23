# Local three-node lab: one control-plane node and two workers on a
# host-only network.
#
#   vagrant up
#   ansible-playbook -i inventory/vagrant.yml site.yml verify.yml

NODES = {
  "cp1"     => "192.168.56.10",
  "worker1" => "192.168.56.21",
  "worker2" => "192.168.56.22",
}

Vagrant.configure("2") do |config|
  config.vm.box = "bento/ubuntu-24.04"

  NODES.each do |name, ip|
    config.vm.define name do |node|
      node.vm.hostname = name
      node.vm.network "private_network", ip: ip

      node.vm.provider "virtualbox" do |vb|
        vb.name = "kubeadm-#{name}"
        vb.cpus = 2
        vb.memory = name.start_with?("cp") ? 2048 : 1536
      end
    end
  end
end
