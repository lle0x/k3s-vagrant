# to make sure the nodes are created in order, we
# have to force a --no-parallel execution.
ENV['VAGRANT_NO_PARALLEL'] = 'yes'

require 'ipaddr'

def get_or_generate_k3s_token
  # TODO generate an unique random an cache it.
  # generated with openssl rand -hex 32
  '7e982a7bbac5f385ecbb988f800787bc9bb617552813a63c4469521c53d83b6e'
end

# see https://get.k3s.io/
# see https://update.k3s.io/v1-release/channels
# see https://github.com/k3s-io/k3s/releases
k3s_channel = 'latest'
k3s_version = 'v1.22.8-rc1+k3s1'
# see https://github.com/helm/helm/releases
helm_version = 'v3.8.1'
# see https://github.com/roboll/helmfile/releases
helmfile_version = 'v0.143.1'
# see https://github.com/kubernetes/dashboard/releases
k8s_dashboard_version = 'v2.5.1'
# see https://github.com/derailed/k9s/releases
k9s_version = 'v0.25.18'
# see https://github.com/kubernetes-sigs/krew/releases
krew_version = 'v0.4.3'
# see https://github.com/etcd-io/etcd/releases
# NB make sure you use the same version as k3s.
etcdctl_version = 'v3.5.0'
# see https://gitlab.com/gitlab-org/charts/gitlab-runner/-/tags
gitlab_runner_chart_version = '0.39.0'
# link to the gitlab-vagrant environment (https://github.com/rgl/gitlab-vagrant running at ../gitlab-vagrant).
gitlab_fqdn = 'gitlab.example.com'
gitlab_ip = '10.10.9.99'

# set the flannel backend. use one of:
# * host-gw:   non-secure network (needs ethernet (L2) connectivity between nodes).
# * wireguard:     secure network (needs UDP (L3) connectivity between nodes).
flannel_backend = 'host-gw'

number_of_server_nodes  = 1
number_of_agent_nodes   = 2
first_server_node_ip    = '10.11.0.101'
first_agent_node_ip     = '10.11.0.201'

server_node_ip_address  = IPAddr.new first_server_node_ip
agent_node_ip_address   = IPAddr.new first_agent_node_ip
k3s_token               = get_or_generate_k3s_token

Vagrant.configure(2) do |config|
  config.vm.box = 'debian-11-amd64'

  config.vm.provider 'libvirt' do |lv, config|
    lv.cpus = 2
    lv.cpu_mode = 'host-passthrough'
    lv.nested = true
    lv.keymap = 'pt'
    config.vm.synced_folder '.', '/vagrant', type: 'nfs', nfs_version: '4.2', nfs_udp: false
  end

  config.vm.provider 'virtualbox' do |vb|
    vb.linked_clone = true
    vb.cpus = 2
  end

  (1..number_of_server_nodes).each do |n|
    name = "s#{n}"
    fqdn = "#{name}.example.test"
    ip_address = server_node_ip_address.to_s; server_node_ip_address = server_node_ip_address.succ

    config.vm.define name do |config|
      config.vm.provider 'libvirt' do |lv, config|
        lv.memory = 1024
      end
      config.vm.provider 'virtualbox' do |vb|
        vb.memory = 1024
      end
      config.vm.hostname = fqdn
      config.vm.network :private_network, ip: ip_address, libvirt__forward_mode: 'none', libvirt__dhcp_enabled: false
      config.vm.provision 'hosts' do |hosts|
        hosts.autoconfigure = true
        hosts.sync_hosts = true
        hosts.add_localhost_hostnames = false
        hosts.add_host gitlab_ip, [gitlab_fqdn]
      end
      config.vm.provision 'shell', path: 'provision-base.sh'
      config.vm.provision 'shell', path: 'provision-wireguard.sh'
      config.vm.provision 'shell', path: 'provision-etcdctl.sh', args: [etcdctl_version]
      config.vm.provision 'shell', path: 'provision-k3s-server.sh', args: [
        n == 1 ? "cluster-init" : "cluster-join",
        k3s_channel,
        k3s_version,
        k3s_token,
        flannel_backend,
        ip_address,
        krew_version
      ]
      config.vm.provision 'shell', path: 'provision-helm.sh', args: [helm_version] # NB this might not really be needed, as rancher has a HelmChart CRD.
      config.vm.provision 'shell', path: 'provision-helmfile.sh', args: [helmfile_version]
      config.vm.provision 'shell', path: 'provision-k9s.sh', args: [k9s_version]
      if n == 1
        config.vm.provision 'shell', path: 'provision-k8s-dashboard.sh', args: [k8s_dashboard_version]
        config.vm.provision 'shell', path: 'provision-gitlab-runner.sh', args: [gitlab_runner_chart_version, gitlab_fqdn, gitlab_ip]
      end
    end
  end

  (1..number_of_agent_nodes).each do |n|
    name = "a#{n}"
    fqdn = "#{name}.example.test"
    ip_address = agent_node_ip_address.to_s; agent_node_ip_address = agent_node_ip_address.succ

    config.vm.define name do |config|
      config.vm.provider 'libvirt' do |lv, config|
        lv.memory = 1*1024
      end
      config.vm.provider 'virtualbox' do |vb|
        vb.memory = 1*1024
      end
      config.vm.hostname = fqdn
      config.vm.network :private_network, ip: ip_address, libvirt__forward_mode: 'none', libvirt__dhcp_enabled: false
      config.vm.provision 'hosts' do |hosts|
        hosts.autoconfigure = true
        hosts.sync_hosts = true
        hosts.add_localhost_hostnames = false
        hosts.add_host gitlab_ip, [gitlab_fqdn]
      end
      config.vm.provision 'shell', path: 'provision-base.sh'
      config.vm.provision 'shell', path: 'provision-wireguard.sh'
      config.vm.provision 'shell', path: 'provision-k3s-agent.sh', args: [
        k3s_channel,
        k3s_version,
        k3s_token,
        ip_address
      ]
    end
  end

  config.trigger.before :up do |trigger|
    trigger.only_on = 's1'
    trigger.run = {
      inline: '''bash -euc \'
mkdir -p tmp
artifacts=(
  ../gitlab-vagrant/tmp/gitlab.example.com-crt.pem
  ../gitlab-vagrant/tmp/gitlab.example.com-crt.der
  ../gitlab-vagrant/tmp/gitlab-runners-registration-token.txt
)
for artifact in "${artifacts[@]}"; do
  if [ -f $artifact ]; then
    cp $artifact tmp
  fi
done
\'
'''
    }
  end
end
