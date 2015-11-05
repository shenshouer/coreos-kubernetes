# -*- mode: ruby -*-
# # vi: set ft=ruby :

require 'fileutils'
require 'open-uri'
require 'tempfile'
require 'yaml'

Vagrant.require_version ">= 1.6.0"

$update_channel = "alpha"
$controller_count = 1
$controller_vm_memory = 1024
$worker_count = 1
$worker_vm_memory = 1024
$etcd_count = 1
$etcd_vm_memory = 512
$shared_folders = {'./binary' => '/kubernetes'}

CONFIG = File.expand_path("config.rb")
if File.exist?(CONFIG)
  require CONFIG
end

ETCD_CLOUD_CONFIG_PATH = File.expand_path("etcd-cloud-config.yaml")

CONTROLLER_CLOUD_CONFIG_PATH = File.expand_path("./controller-install.sh")
WORKER_CLOUD_CONFIG_PATH = File.expand_path("./worker-install.sh")

SSL_CONTROLLER_PATH = File.expand_path("ssl/controller.tar")
SSL_WORKER_PATH = File.expand_path("ssl/worker.tar")

def etcdIP(num)
  return "172.17.4.#{num+50}"
end

def controllerIP(num)
  return "172.17.4.#{num+100}"
end

def workerIP(num)
  return "172.17.4.#{num+200}"
end

controllerIPs = [*1..$controller_count].map{ |i| controllerIP(i) }
workerIPs = [*1..$worker_count].map{ |i| workerIP(i) }
etcdIPs = [*1..$etcd_count].map{ |i| etcdIP(i) }
initial_etcd_cluster = etcdIPs.map.with_index{ |ip, i| "etcd#{i+1}=http://#{ip}:2380" }.join(",")
etcd_endpoints = etcdIPs.map.with_index{ |ip, i| "http://#{ip}:2379" }.join(",")

# Configure SSL certificates
if !File.exist?(SSL_CONTROLLER_PATH) || !File.exists?(SSL_WORKER_PATH) then
    sans = "IP.1=10.3.0.1," + controllerIPs.map.with_index { |ip, i| "IP.#{i+2}=#{ip}"}.join(",")
    system("mkdir -p ssl && ./init-ssl ssl #{sans}") or abort ("failed generating SSL artifacts")
end

Vagrant.configure("2") do |config|
  # always use Vagrant's insecure key
  config.ssh.insert_key = false

  config.vm.box = "coreos-%s" % $update_channel
  config.vm.box_version = ">= 766.0.0"
  config.vm.box_url = "http://%s.release.core-os.net/amd64-usr/current/coreos_production_vagrant.json" % $update_channel

  ["vmware_fusion", "vmware_workstation"].each do |vmware|
    config.vm.provider vmware do |v, override|
      override.vm.box_url = "http://%s.release.core-os.net/amd64-usr/current/coreos_production_vagrant_vmware_fusion.json" % $update_channel
    end
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

  ["vmware_fusion", "vmware_workstation"].each do |vmware|
    config.vm.provider vmware do |v|
      v.vmx['numvcpus'] = 1
      v.gui = false
    end
  end

  config.vm.provider :virtualbox do |vb|
    vb.cpus = 1
    vb.gui = false
  end

  (1..$etcd_count).each do |i|
    config.vm.define vm_name = "etcd%d" % i do |etcd|

      data = YAML.load(IO.readlines(ETCD_CLOUD_CONFIG_PATH)[1..-1].join)
      data['coreos']['etcd2']['initial-cluster'] = initial_etcd_cluster
      data['coreos']['etcd2']['name'] = vm_name
      etcd_config_file = Tempfile.new('etcd_config')
      etcd_config_file.write("#cloud-config\n#{data.to_yaml}")
      etcd_config_file.close

      etcd.vm.hostname = vm_name

      ["vmware_fusion", "vmware_workstation"].each do |vmware|
        etcd.vm.provider vmware do |v|
          v.vmx['memsize'] = $etcd_vm_memory
        end
      end

      etcd.vm.provider :virtualbox do |vb|
        vb.memory = $etcd_vm_memory
      end

      etcd.vm.network :private_network, ip: etcdIP(i)

      etcd.vm.provision :file, :source => etcd_config_file.path, :destination => "/tmp/vagrantfile-user-data"
      etcd.vm.provision :shell, :inline => "mv /tmp/vagrantfile-user-data /var/lib/coreos-vagrant/", :privileged => true
    end
  end


  (1..$controller_count).each do |i|
    config.vm.define vm_name = "master%d" % i do |controller|

      env_file = Tempfile.new('env_file')
      env_file.write("ETCD_ENDPOINTS=#{etcd_endpoints}\n")
      env_file.close

      controller.vm.hostname = vm_name

      ["vmware_fusion", "vmware_workstation"].each do |vmware|
        controller.vm.provider vmware do |v|
          v.vmx['memsize'] = $controller_vm_memory
        end
      end

      controller.vm.provider :virtualbox do |vb|
        vb.memory = $controller_vm_memory
      end

      controller.vm.network :private_network, ip: controllerIP(i)

      controller.vm.provision :file, :source => SSL_CONTROLLER_PATH, :destination => "/tmp/ssl.tar"
      controller.vm.provision :shell, :inline => "mkdir -p /etc/kubernetes/ssl && tar -C /etc/kubernetes/ssl -xf /tmp/ssl.tar", :privileged => true

      controller.vm.provision :file, :source => env_file, :destination => "/tmp/coreos-kube-options.env"
      controller.vm.provision :shell, :inline => "mkdir -p /run/coreos-kubernetes && mv /tmp/coreos-kube-options.env /run/coreos-kubernetes/options.env", :privileged => true

      controller.vm.provision :file, :source => CONTROLLER_CLOUD_CONFIG_PATH, :destination => "/tmp/vagrantfile-user-data"
      controller.vm.provision :shell, :inline => "mv /tmp/vagrantfile-user-data /var/lib/coreos-vagrant/", :privileged => true

      $shared_folders.each_with_index do |(host_folder, guest_folder), index|
        config.vm.synced_folder host_folder.to_s, guest_folder.to_s, id: "core-share%02d" % index, nfs: true, mount_options: ['nolock,vers=3,udp']
      end
    end
  end

  (1..$worker_count).each do |i|
    config.vm.define vm_name = "node%d" % i do |worker|
      worker.vm.hostname = vm_name

      env_file = Tempfile.new('env_file')
      env_file.write("ETCD_ENDPOINTS=#{etcd_endpoints}\n")
      env_file.write("CONTROLLER_ENDPOINT=https://#{controllerIPs[0]}\n") #TODO(aaron): LB or DNS across control nodes
      env_file.close

      ["vmware_fusion", "vmware_workstation"].each do |vmware|
        worker.vm.provider vmware do |v|
          v.vmx['memsize'] = $worker_vm_memory
        end
      end

      worker.vm.provider :virtualbox do |vb|
        vb.memory = $worker_vm_memory
      end

      worker.vm.network :private_network, ip: workerIP(i)

      worker.vm.provision :file, :source => SSL_WORKER_PATH, :destination => "/tmp/ssl.tar"
      worker.vm.provision :shell, :inline => "mkdir -p /etc/kubernetes/ssl && tar -C /etc/kubernetes/ssl -xf /tmp/ssl.tar", :privileged => true

      worker.vm.provision :file, :source => env_file, :destination => "/tmp/coreos-kube-options.env"
      worker.vm.provision :shell, :inline => "mkdir -p /run/coreos-kubernetes && mv /tmp/coreos-kube-options.env /run/coreos-kubernetes/options.env", :privileged => true

      worker.vm.provision :file, :source => WORKER_CLOUD_CONFIG_PATH, :destination => "/tmp/vagrantfile-user-data"
      worker.vm.provision :shell, :inline => "mv /tmp/vagrantfile-user-data /var/lib/coreos-vagrant/", :privileged => true

      $shared_folders.each_with_index do |(host_folder, guest_folder), index|
        config.vm.synced_folder host_folder.to_s, guest_folder.to_s, id: "core-share%02d" % index, nfs: true, mount_options: ['nolock,vers=3,udp']
      end
    end
  end

end