# -*- mode: ruby -*-
# vi: set ft=ruby :
#
# Copyright 2017 Liu Hongyu
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#

OPENSHIFT_RELEASE = "3.11"
OPENSHIFT_ANSIBLE_BRANCH = "release-#{OPENSHIFT_RELEASE}"
NETWORK_BASE = "10.16.35"
NETWORK_NETMASK = "255.255.252.0"
INTEGRATION_START_SEGMENT = 155

VAGRANT_PROVIDER_NAME = "vmware_desktop"

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
Vagrant.configure("2") do |config|
  # The most common configuration options are documented and commented below.
  # For a complete reference, please see the online documentation at
  # https://docs.vagrantup.com.

  # Every Vagrant development environment requires a box. You can search for
  # boxes at https://vagrantcloud.com/search.
  config.vm.box = "centos/7"
  config.vm.box_version = "1812.01"
  config.vm.box_check_update = false

  # if Vagrant.has_plugin?('landrush')
  #   config.landrush.enabled = true
  #   config.landrush.tld = 'okd.lab'
  #   config.landrush.guest_redirect_dns = false
  # end

  config.hostmanager.enabled = true
  config.hostmanager.manage_host = true
  config.hostmanager.ignore_private_ip = false
  config.hostmanager.ip_resolver = proc do |machine|
    result = ""
    machine.communicate.execute("ip address show eth1") do |type, data|
        result << data if type == :stdout
    end
    (ip = /inet (\d+\.\d+\.\d+\.\d+)/.match(result)) && ip[1]
  end

  config.vm.provision "shell", inline: <<-SHELL
    bash -c 'echo "export TZ=UTC0" > /etc/profile.d/tz.sh'

    yum -y install docker
    usermod -aG dockerroot vagrant
    cat > /etc/docker/daemon.json <<EOF
{
    "group": "dockerroot"
}
EOF
    systemctl enable docker
    systemctl start docker

    # Sourcing common functions
    . /vagrant/common.sh
    # Fix missing packages for openshift origin 3.11.0
    # https://lists.openshift.redhat.com/openshift-archives/dev/2018-November/msg00005.html
    if [ "$(version #{OPENSHIFT_RELEASE})" -eq "$(version 3.11)" ]; then
      yum install -y centos-release-openshift-origin311
    fi
  SHELL

  config.vm.provider "virtualbox" do |vb|
    vb.memory = "2048"
    vb.cpus   = "1"
  end

  config.vm.provider "vmware_desktop" do |v|
    v.vmx["memsize"] = "2048"
    v.vmx["numvcpus"] = "1"
  end

  # Define nodes
  (1..2).each do |i|
    config.vm.define "node0#{i}" do |node|
      node.vm.network "public_network", ip: "#{NETWORK_BASE}.#{INTEGRATION_START_SEGMENT + i}", netmask: NETWORK_NETMASK
      node.vm.hostname = "node0#{i}.okd.lab"

      if "#{i}" == "1"
        node.hostmanager.aliases = %w(lb.okd.lab)
      end
    end
  end

  # Define master
  config.vm.define "master", primary: true do |node|
    node.vm.network "public_network", ip: "#{NETWORK_BASE}.#{INTEGRATION_START_SEGMENT}", netmask: NETWORK_NETMASK
    node.vm.hostname = "master.okd.lab"
    node.hostmanager.aliases = %w(
      openshift.okd.lab
      master-internal.okd.lab
      etcd.okd.lab
      nfs.okd.lab
      kibana.okd.lab
      console.app.okd.lab
    )
    
    # 
    # Memory of the master node must be allocated at least 2GB in order to
    # prevent kubernetes crashed-down due to 'out of memory' and you'll end
    # up with 
    # "Unable to restart service origin-master: Job for origin-master.service 
    #  failed because a timeout was exceeded. See "systemctl status 
    #  origin-master.service" and "journalctl -xe" for details."
    #
    # See https://github.com/kubernetes/kubernetes/issues/13382#issuecomment-154891888
    # for mor details.
    #
    node.vm.provider "virtualbox" do |vb|
      vb.memory = "4096"
    end

    node.vm.provider "vmware_desktop" do |v|
      v.vmx["memsize"] = "4096"
      v.vmx["numvcpus"] = "4"
    end

    node.vm.provision "shell", inline: <<-SHELL
      yum -y install git net-tools bind-utils iptables-services bridge-utils bash-completion kexec-tools sos psacct

      # required when using own certificates
      yum -y install pyOpenSSL
      
      # Sourcing common functions
      . /vagrant/common.sh
      
      if [ "$(version #{OPENSHIFT_RELEASE})" -gt "$(version 3.7)" ]; then
        yum -y install https://releases.ansible.com/ansible/rpm/release/epel-7-x86_64/ansible-2.6.9-1.el7.ans.noarch.rpm
      else
        yum -y install https://releases.ansible.com/ansible/rpm/release/epel-7-x86_64/ansible-2.5.9-1.el7.ans.noarch.rpm
      fi
      
      git clone -b #{OPENSHIFT_ANSIBLE_BRANCH} https://github.com/openshift/openshift-ansible.git /home/vagrant/openshift-ansible
      
      mkdir -p /home/vagrant/.ssh
      bash -c 'echo "Host *" >> /home/vagrant/.ssh/config'
      bash -c 'echo "StrictHostKeyChecking no" >> /home/vagrant/.ssh/config'
      chmod 600 /home/vagrant/.ssh/config
      chown -R vagrant:vagrant /home/vagrant
    SHELL

    node.vm.provision "ansible-hosts", type: "shell", inline: <<-SHELL
      # Sourcing common functions
      . /vagrant/common.sh

      mv -f /etc/ansible/hosts /etc/ansible/hosts.bak
      
      # Pre-define all possible openshift node groups
      NODE_GROUP_MASTER="openshift_node_group_name='node-config-master'"
      NODE_GROUP_INFRA="openshift_node_group_name='node-config-infra'"
      NODE_GROUP_COMPUTE="openshift_node_group_name='node-config-compute'"
      NODE_GROUP_MASTER_INFRA="openshift_node_group_name='node-config-master-infra'"
      NODE_GROUP_ALLINONE="openshift_node_group_name='node-config-all-in-one'"
      HTPASSWORD_FILENAME=", 'filename': '/etc/origin/master/htpasswd'"

      # Prevent error "provider HTPasswdPasswordIdentityProvider contains unknown keys filename"
      # when openshift version is 3.10 or above.
      if [ "$(version #{OPENSHIFT_RELEASE})" -ge "$(version 3.10)" ]; then
        unset HTPASSWORD_FILENAME
      fi

      cat /vagrant/ansible-hosts \
        | sed "s/{{OPENSHIFT_RELEASE}}/#{OPENSHIFT_RELEASE}/g" \
        | sed "s/{{NETWORK_BASE}}/#{NETWORK_BASE}/g" \
        | sed "s/{{NETWORK_MASTER_IP}}/#{NETWORK_BASE}.#{INTEGRATION_START_SEGMENT}/g" \
        | sed "s/{{NETWORK_NODE01_IP}}/#{NETWORK_BASE}.#{INTEGRATION_START_SEGMENT + 1}/g" \
        | sed "s/{{NETWORK_NODE02_IP}}/#{NETWORK_BASE}.#{INTEGRATION_START_SEGMENT + 2}/g" \
        | sed "s/{{NODE_GROUP_MASTER}}/${NODE_GROUP_MASTER}/g" \
        | sed "s/{{NODE_GROUP_INFRA}}/${NODE_GROUP_INFRA}/g" \
        | sed "s/{{NODE_GROUP_COMPUTE}}/${NODE_GROUP_COMPUTE}/g" \
        | sed "s/{{NODE_GROUP_MASTER_INFRA}}/${NODE_GROUP_MASTER_INFRA}/g" \
        | sed "s/{{NODE_GROUP_ALLINONE}}/${NODE_GROUP_ALLINONE}/g" \
        | sed "s~{{HTPASSWORD_FILENAME}}~${HTPASSWORD_FILENAME}~g" \
        > /etc/ansible/hosts
    SHELL

    node.vm.provision "ca-key", type: "shell", inline: <<-SHELL
      cp -f /vagrant/okd_root_ca.key /etc/ansible/okd_root_ca.key
      cp -f /vagrant/okd_root_ca.crt /etc/ansible/okd_root_ca.crt
    SHELL

    # Deploy private keys of each node to master
    if File.exist?(".vagrant/machines/master/#{VAGRANT_PROVIDER_NAME}/private_key")
      node.vm.provision "master-key", type: "file", run: "never", source: ".vagrant/machines/master/#{VAGRANT_PROVIDER_NAME}/private_key", destination: "/home/vagrant/.ssh/master.key"
    end

    if File.exist?(".vagrant/machines/node01/#{VAGRANT_PROVIDER_NAME}/private_key")
      node.vm.provision "node01-key", type: "file", run: "never", source: ".vagrant/machines/node01/#{VAGRANT_PROVIDER_NAME}/private_key", destination: "/home/vagrant/.ssh/node01.key"
    end

    if File.exist?(".vagrant/machines/node02/#{VAGRANT_PROVIDER_NAME}/private_key")
      node.vm.provision "node02-key", type: "file", run: "never", source: ".vagrant/machines/node02/#{VAGRANT_PROVIDER_NAME}/private_key", destination: "/home/vagrant/.ssh/node02.key"
    end

    node.vm.provision "fix-keys", type: "shell", run: "never", privileged: false, inline: "chmod 600 ~/.ssh/*.key"
  end
end
