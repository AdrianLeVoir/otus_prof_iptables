# -*- mode: ruby -*-
# ДЗ: Сценарии iptables
# Топология наследуется от сетевой лаборатории (hw17):
#   office1/office2 -> centralRouter -> inetRouter -> internet
# Добавлены: inetRouter2 (виден с хоста) и knocking на inetRouter.

VAGRANTFILE_API_VERSION = "2"

MACHINES = {
  :inetRouter => {
    :box_name => "generic/ubuntu2204",
    :vm_name => "inetRouter",
    :net => [
      # NAT-интерфейс vagrant (по умолчанию eth0) + сеть к centralRouter
      ["192.168.255.1", 2, "255.255.255.252", "router-net"],
    ]
  },
  :inetRouter2 => {
    :box_name => "generic/ubuntu2204",
    :vm_name => "inetRouter2",
    :net => [
      # host-only сеть: виден с хоста
      ["192.168.50.2", 2, "255.255.255.0", "inet2-hostonly"],
      # сеть к centralRouter (транзит до centralServer)
      ["192.168.0.3",  3, "255.255.255.240", "dir-net"],
    ]
  },
  :centralRouter => {
    :box_name => "generic/ubuntu2204",
    :vm_name => "centralRouter",
    :net => [
      ["192.168.255.2", 2, "255.255.255.252",  "router-net"],
      ["192.168.0.1",   3, "255.255.255.240",  "dir-net"],
    ]
  },
  :centralServer => {
    :box_name => "generic/ubuntu2204",
    :vm_name => "centralServer",
    :net => [
      ["192.168.0.2",   2, "255.255.255.240",  "dir-net"],
    ]
  },
}

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  MACHINES.each do |boxname, boxconfig|
    config.vm.define boxname do |box|
      box.vm.box = boxconfig[:box_name]
      box.vm.host_name = boxconfig[:vm_name]

      box.vm.provider "virtualbox" do |v|
        v.memory = 1024
        v.cpus = 1
      end

      boxconfig[:net].each do |ipconf|
        box.vm.network("private_network", ip: ipconf[0], adapter: ipconf[1],
                       netmask: ipconf[2], virtualbox__intnet: ipconf[3])
      end

      box.vm.provision "shell", inline: <<-SHELL
        mkdir -p ~root/.ssh
        cp ~vagrant/.ssh/auth* ~root/.ssh
      SHELL
    end
  end

  # Ansible запускается после последней ВМ
  config.vm.define "centralServer" do |box|
    box.vm.provision "ansible" do |ansible|
      ansible.playbook = "ansible/provision.yml"
      ansible.inventory_path = "ansible/hosts"
      ansible.host_key_checking = "false"
      ansible.limit = "all"
    end
  end
end