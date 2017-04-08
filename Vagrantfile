# -*- mode: ruby -*-
# vi: set ft=ruby :

ANSIBLE_PATH = __dir__ # absolute path to Ansible directory on host machine
ANSIBLE_PATH_ON_VM = '/home/vagrant/trellis' # absolute path to Ansible directory on virtual machine

require File.join(ANSIBLE_PATH, 'lib', 'trellis', 'vagrant')
require 'yaml'

vconfig = YAML.load_file("#{ANSIBLE_PATH}/vagrant.default.yml")

if File.exist?("#{ANSIBLE_PATH}/vagrant.local.yml")
  local_config = YAML.load_file("#{ANSIBLE_PATH}/vagrant.local.yml")
  vconfig.merge!(local_config) if local_config
end

wordpress_sites = load_wordpress_sites
site_hosts = hosts(wordpress_sites)

Vagrant.require_version '>= 1.8.5'

Vagrant.configure('2') do |config|
  config.vm.box = vconfig.fetch('vagrant_box')
  config.vm.box_version = vconfig.fetch('vagrant_box_version')
  config.ssh.forward_agent = true
  config.vm.post_up_message = post_up_message

  # Fix for: "stdin: is not a tty"
  # https://github.com/mitchellh/vagrant/issues/1673#issuecomment-28288042
  config.ssh.shell = %{bash -c 'BASH_ENV=/etc/profile exec bash'}

  # Required for NFS to work
  config.vm.network :private_network, ip: vconfig.fetch('vagrant_ip'), hostsupdater: 'skip'

  main_hostname, *hostnames = site_hosts.map { |host| host['canonical'] }
  config.vm.hostname = main_hostname

  if Vagrant.has_plugin?('vagrant-hostmanager') && !multisite_subdomains?(wordpress_sites)
    redirects = site_hosts.flat_map { |host| host['redirects'] }.compact

    config.hostmanager.enabled = true
    config.hostmanager.manage_host = true
    config.hostmanager.aliases = hostnames + redirects
  elsif Vagrant.has_plugin?('landrush') && multisite_subdomains?(wordpress_sites)
    config.landrush.enabled = true
    config.landrush.tld = config.vm.hostname
    hostnames.each { |host| config.landrush.host host, vconfig.fetch('vagrant_ip') }
  else
    fail_with_message "vagrant-hostmanager missing, please install the plugin with this command:\nvagrant plugin install vagrant-hostmanager\n\nOr install landrush for multisite subdomains:\nvagrant plugin install landrush"
  end

  bin_path = File.join(ANSIBLE_PATH_ON_VM, 'bin')

  if Vagrant::Util::Platform.windows? and !Vagrant.has_plugin? 'vagrant-winnfsd'
    wordpress_sites.each_pair do |name, site|
      config.vm.synced_folder local_site_path(site), remote_site_path(name, site), owner: 'vagrant', group: 'www-data', mount_options: ['dmode=776', 'fmode=775']
    end

    config.vm.synced_folder ANSIBLE_PATH, ANSIBLE_PATH_ON_VM, mount_options: ['dmode=755', 'fmode=644']
    config.vm.synced_folder File.join(ANSIBLE_PATH, 'bin'), bin_path, mount_options: ['dmode=755', 'fmode=755']
  else
    if !Vagrant.has_plugin? 'vagrant-bindfs'
      fail_with_message "vagrant-bindfs missing, please install the plugin with this command:\nvagrant plugin install vagrant-bindfs"
    else
      wordpress_sites.each_pair do |name, site|
        config.vm.synced_folder local_site_path(site), nfs_path(name), type: 'nfs'
        config.bindfs.bind_folder nfs_path(name), remote_site_path(name, site), u: 'vagrant', g: 'www-data', o: 'nonempty'
      end

      config.vm.synced_folder ANSIBLE_PATH, '/ansible-nfs', type: 'nfs'
      config.bindfs.bind_folder '/ansible-nfs', ANSIBLE_PATH_ON_VM, o: 'nonempty', p: '0644,a+D'
      config.bindfs.bind_folder bin_path, bin_path, perms: '0755'
    end
  end

  vconfig.fetch('vagrant_synced_folders', []).each do |folder|
    options = {
      type: folder.fetch('type', 'nfs'),
      create: folder.fetch('create', false),
      mount_options: folder.fetch('mount_options', [])
    }

    destination_folder = folder.fetch('bindfs', true) ? nfs_path(folder['destination']) : folder['destination']

    config.vm.synced_folder folder['local_path'], destination_folder, options

    if folder.fetch('bindfs', true)
      config.bindfs.bind_folder destination_folder, folder['destination'], folder.fetch('bindfs_options', {})
    end
  end

  provisioner = local_provisioning? ? :ansible_local : :ansible
  provisioning_path = local_provisioning? ? ANSIBLE_PATH_ON_VM : ANSIBLE_PATH

  config.vm.provision provisioner do |ansible|
    if local_provisioning?
      ansible.install_mode = 'pip'
      ansible.provisioning_path = provisioning_path
      ansible.version = vconfig.fetch('vagrant_ansible_version')
    end

    ansible.playbook = File.join(provisioning_path, 'dev.yml')
    ansible.galaxy_role_file = File.join(provisioning_path, 'requirements.yml') unless vconfig.fetch('vagrant_skip_galaxy') || ENV['SKIP_GALAXY']
    ansible.galaxy_roles_path = File.join(provisioning_path, 'vendor/roles')

    ansible.groups = {
      'web' => ['default'],
      'development' => ['default']
    }

    ansible.tags = ENV['ANSIBLE_TAGS']
    ansible.extra_vars = { 'vagrant_version' => Vagrant::VERSION }

    if vars = ENV['ANSIBLE_VARS']
      extra_vars = Hash[vars.split(',').map { |pair| pair.split('=') }]
      ansible.extra_vars.merge(extra_vars)
    end
  end

  # Virtualbox settings
  config.vm.provider 'virtualbox' do |vb|
    vb.name = config.vm.hostname
    vb.customize ['modifyvm', :id, '--cpus', vconfig.fetch('vagrant_cpus')]
    vb.customize ['modifyvm', :id, '--memory', vconfig.fetch('vagrant_memory')]

    # Fix for slow external network connections
    vb.customize ['modifyvm', :id, '--natdnshostresolver1', 'on']
    vb.customize ['modifyvm', :id, '--natdnsproxy1', 'on']
  end

  # VMware Workstation/Fusion settings
  ['vmware_fusion', 'vmware_workstation'].each do |provider|
    config.vm.provider provider do |vmw, override|
      vmw.name = config.vm.hostname
      vmw.vmx['numvcpus'] = vconfig.fetch('vagrant_cpus')
      vmw.vmx['memsize'] = vconfig.fetch('vagrant_memory')
    end
  end

  # Parallels settings
  config.vm.provider 'parallels' do |prl, override|
    prl.name = config.vm.hostname
    prl.cpus = vconfig.fetch('vagrant_cpus')
    prl.memory = vconfig.fetch('vagrant_memory')
    prl.update_guest_tools = true
  end
end
