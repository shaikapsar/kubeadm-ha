# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'ipaddr'
require 'yaml'
require 'fileutils'

if File.exists?(File.join(File.dirname(__FILE__), "local_config.yaml")) then
  puts "Using local configuration file"
  config_data = YAML.load_file(File.join(File.dirname(__FILE__), "local_config.yaml"))
else
  config_data = YAML.load_file(File.join(File.dirname(__FILE__), 'config.yaml'))
end

# Create Directory for ansible if the directory doesn't exist.
ansible_dir = File.dirname('ansible')
unless File.directory?(ansible_dir)
  FileUtils.mkdir_p(ansible_dir)
end

inventory_content = "# DONT EDIT INVENTORY FILE MANUALLY\n\n"

inventory = Hash.new []
inventory["hosts"] = Hash.new []
inventory["groups"] = Hash.new []

etcd_initial_cluster = ""
etcd_servers = ""

config_data.fetch('servers').each do |server|

  if server.fetch('enable') then
    hostentry = ""
    server_id = server.fetch('id')
    inventory["hosts"][server_id]=Hash.new []
    ansible_host = server.fetch('base_address')
    ansible_user = server.fetch('ansible_user')
    inventory["hosts"][server_id]["ansible_host"]= ansible_host
    inventory["hosts"][server_id]["ansible_user"] = ansible_user

    server.fetch('roles').each do |role|
      group_name=role
      if !inventory["groups"].has_key?(group_name) then
        inventory["groups"][group_name] = Array.new
      end
      inventory["groups"][group_name].push server_id
    end

    if server.fetch('roles').include? 'deployer' then
      hostentry = "#{server_id} ansible_connection=local\n"
      inventory["hosts"][server_id]["ansible_connection"]="local"
    else
      hostentry = "#{server_id} ansible_connection=ssh ansible_host=#{ansible_host}\n"
      inventory["hosts"][server_id]["ansible_connection"]="ssh"
      #ansible_ssh_private_key_file=/vagrant/.vagrant/machines/#{server_id}/virtualbox/private_key
    end

    inventory_content = inventory_content + hostentry 

    if server.fetch('roles').include? 'etcd' then
      etcd_initial_cluster = etcd_initial_cluster + "#{server_id}=https://#{ansible_host}:2380,"
      etcd_servers = etcd_servers + + "https://#{ansible_host}:2379,"
    end

  end 

end


etcd_initial_cluster = etcd_initial_cluster.chop
etcd_servers = etcd_servers.chop

cfssl_version = config_data.fetch('cfssl_version')

domain = config_data.fetch('domain')
kubernetes_vip = config_data.fetch('kubernetes_vip')
vrrp_nic = config_data.fetch('vrrp_nic')
kubernetes_vip_port = config_data.fetch('kubernetes_vip_port')
kubernetes_release = config_data.fetch('kubernetes_release')
kubernetes_api_port = config_data.fetch('kubernetes_api_port')


deployer_certificate_directory = config_data.fetch('deployer_certificate_directory')
deployer_configuration_directory = config_data.fetch('deployer_configuration_directory')

File.open("inventory", "w+") {
  |f|
  f.write("#Customized Inventory file \n\n")

  inventory["hosts"].each do |host_key, host_value|
    ansible_connection = host_value["ansible_connection"]
    ansible_host = host_value["ansible_host"]
    ansible_user = host_value["ansible_user"]
    if ansible_connection == "local" then
      f.write("#{host_key} ansible_connection=#{ansible_connection} ansible_host=#{ansible_host} ansible_user=#{ansible_user}\n")
    else
      f.write("#{host_key} ansible_connection=#{ansible_connection} ansible_host=#{ansible_host} ansible_user=#{ansible_user}\n")
      #f.write("#{host_key} ansible_connection=#{ansible_connection} ansible_host=#{ansible_host} ansible_user=#{ansible_user} ansible_ssh_private_key_file=/vagrant/.vagrant/machines/#{host_key}/virtualbox/private_key\n")
    end
  end

  inventory["groups"].each do | group_key, group_value |
    f.write("\n")
    f.write("[#{group_key}]\n")
    for i in 0...inventory["groups"][group_key].size do
      group_entity=inventory["groups"][group_key][i]
      f.write("#{group_entity} \n")
    end
  end

  f.write("\n[all:vars]\nansible_ssh_pass=vagrant\ndomain=#{domain}\netcd_initial_cluster=#{etcd_initial_cluster}\nkubernetes_api_port=#{kubernetes_api_port}\nkubernetes_vip=#{kubernetes_vip}\netcd_servers=#{etcd_servers}\nvrrp_nic=#{vrrp_nic}\nkubernetes_vip_port=#{kubernetes_vip_port}\nkubernetes_release=#{kubernetes_release}\ncfssl_version=#{cfssl_version}\ndeployer_certificate_directory=#{deployer_certificate_directory}\ndeployer_configuration_directory=#{deployer_configuration_directory}") 

}



infra_provider = config_data.fetch('infra_provider')

Vagrant.configure("2") do |config|

  config_data.fetch('servers').each do |server|

    if server.fetch('enable') then
      #hostname = server.fetch('id') + "." + $domain
      hostname = server.fetch('id')

      config.vm.define hostname do |node|

        if Vagrant.has_plugin?("vagrant-hostmanager")
          config.hostmanager.enabled = true
          config.hostmanager.manage_host = true
          config.hostmanager.manage_guest = true
          config.hostmanager.ignore_private_ip = false
          config.hostmanager.include_offline = true
          #node.hostmanager.manage_guest = server.fetch('hostmanager').fetch('manage_guest')
        end

        node.vm.box = server.fetch('box')
        node.vm.hostname = hostname
        node.vm.box_version = server.fetch('box_version')

        node.vm.provider infra_provider do |v|
          v.memory = server.fetch('resources').fetch('memory')
          v.cpus = server.fetch('resources').fetch('cpus')
          v.name = hostname
        end

        node.vm.network "private_network", ip: server.fetch('base_address')
        
        if server.fetch('roles').include? 'deployer'  then
          #puts "s iThis deployer.\n\n"
          node.vm.provision "ansible_local" do |ansible|
            ansible.playbook = "ansible/site.yaml"
            ansible.inventory_path = "inventory"
            ansible.config_file = "ansible/ansible.cfg"
            ansible.galaxy_role_file = "ansible/roles-requirements.yaml"
            ansible.galaxy_command = "sudo ansible-galaxy install -r %{role_file} -p /usr/share/ansible/roles --force && sudo ansible-galaxy collection install -r ansible/collections-requirements.yaml -p /usr/share/ansible/collections " #--force
            ansible.galaxy_roles_path = "/usr/share/ansible/roles" #"/etc/ansible/roles"
            ansible.verbose = "-vvv"
            ansible.limit = "all" 
            ansible.become = true
            ansible.extra_vars = {
                domain: domain,
                etcd_initial_cluster: etcd_initial_cluster,
                kubernetes_vip: kubernetes_vip,
                kubernetes_vip_port: kubernetes_vip_port,
                kubernetes_api_port: kubernetes_api_port,
                etcd_servers: etcd_servers,
                vrrp_nic: vrrp_nic,
                kubernetes_release: kubernetes_release,
                cfssl_version: cfssl_version,
                deployer_certificate_directory: deployer_certificate_directory,
                deployer_configuration_directory: deployer_configuration_directory,
                
            }
          end

          node.trigger.before :destroy do |trigger|
            trigger.name = "Impossible trigger, Pre-Destroy"
            trigger.run_remote = { inline: "rm -rf /vagrant/.pki /vagrant/.configuration /vagrant/ansible/vagrant-inventory" }
            trigger.on_error = :continue
          end
        end
      end
    end
  end
end