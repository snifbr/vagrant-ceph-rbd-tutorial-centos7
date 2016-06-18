# -*- mode: ruby -*-
# vi: set ft=ruby :

RELEASE = 'infernalis'
USER = 'ceph'

hosts = {
  'server0' => {'hostname' => 'server0', 'ip' => '192.168.10.10', 'mac' => '080027001010'},
  'server1' => {'hostname' => 'server1', 'ip' => '192.168.10.11', 'mac' => '080027001011'},
  'server2' => {'hostname' => 'server2', 'ip' => '192.168.10.12', 'mac' => '080027001012'},
  'client0' => {'hostname' => 'client0', 'ip' => '192.168.10.100', 'mac' => '080027000100'}
}

Vagrant.configure(2) do |config|
  hosts.keys.sort.each do |host|
    if host.start_with?("server")
      config.vm.define hosts[host]['hostname'] do |server|
        server.vm.box = 'centos/7'
        server.vm.box_url = 'centos/7'
        server.vm.synced_folder '.', '/home/vagrant/sync', disabled: true
        server.vm.network 'private_network', ip: hosts[host]['ip'], mac: hosts[host]['mac'], auto_config: false
        server.vm.provider 'virtualbox' do |v|
          v.memory = 512
          v.cpus = 1
          # disable VBox time synchronization and use ntp
          v.customize ['setextradata', :id, 'VBoxInternal/Devices/VMMDev/0/Config/GetHostTimeDisabled', 1]
          # sdb and sdc block devices belonging to the servers
          disk = hosts[host]['hostname'] + 'sdb.vdi'
          if !File.exist?(disk)
            v.customize ['createhd', '--filename', disk, '--size', 128, '--variant', 'Fixed']
            v.customize ['modifyhd', disk, '--type', 'writethrough']
          end
          v.customize ['storageattach', :id, '--storagectl', 'IDE Controller', '--port', 0, '--device', 1, '--type', 'hdd', '--medium', disk]
          disk = hosts[host]['hostname'] + 'sdc.vdi'
          if !File.exist?(disk)
            v.customize ['createhd', '--filename', disk, '--size', 16, '--variant', 'Fixed']
            v.customize ['modifyhd', disk, '--type', 'writethrough']
          end
          v.customize ['storageattach', :id, '--storagectl', 'IDE Controller', '--port', 1, '--device', 0, '--type', 'hdd', '--medium', disk]
        end
      end
    end
  end
  hosts.keys.sort.each do |host|
    if host.start_with?("client")
      config.vm.define hosts[host]['hostname'] do |client|
        client.vm.box = 'centos/7'
        client.vm.box_url = 'centos/7'
        client.vm.synced_folder '.', '/home/vagrant/sync', disabled: true
        client.vm.network 'private_network', ip: hosts[host]['ip'], mac: hosts[host]['mac'], auto_config: false
        client.vm.provider 'virtualbox' do |v|
          v.memory = 256
          v.cpus = 1
          # disable VBox time synchronization and use ntp
          v.customize ['setextradata', :id, 'VBoxInternal/Devices/VMMDev/0/Config/GetHostTimeDisabled', 1]
        end
      end
    end
  end
  # disable IPv6 on Linux
  $linux_disable_ipv6 = <<SCRIPT
sysctl -w net.ipv6.conf.default.disable_ipv6=1
sysctl -w net.ipv6.conf.all.disable_ipv6=1
sysctl -w net.ipv6.conf.lo.disable_ipv6=1
SCRIPT
  # setenforce 0
  $setenforce_0 = <<SCRIPT
if test `getenforce` = 'Enforcing'; then setenforce 0; fi
#sed -Ei 's/^SELINUX=.*/SELINUX=Permissive/' /etc/selinux/config
SCRIPT
  # stop firewalld
  $systemctl_stop_firewalld = <<SCRIPT
systemctl stop firewalld.service
SCRIPT
  # common settings on all machines
  $etc_hosts = <<SCRIPT
echo "$*" >> /etc/hosts
SCRIPT
  $ceph_noarch_el = <<SCRIPT
cat <<END > /etc/yum.repos.d/ceph-noarch.repo
[ceph-noarch]
name=CentOS-\\$releasever - ceph noarch
baseurl=https://download.ceph.com/rpm/el\\$releasever/noarch/
enabled=1
#key retrieval often fails
gpgcheck=0
#gpgkey='https://ceph.com/git/?p=ceph.git;a=blob_plain;f=keys/release.asc'
END
SCRIPT
  # key-based ssh using vagrant keys
  $key_based_ssh = <<SCRIPT
home=`getent passwd $1 | cut -d: -f6`
rm -rf ${home}/.ssh
ls -al ~vagrant ${home}
cp -rp ~vagrant/.ssh ${home}
yum -y install wget
wget --retry-connrefused --waitretry=5 --no-check-certificate https://raw.github.com/mitchellh/vagrant/master/keys/vagrant.pub -O ${home}/.ssh/authorized_keys
SCRIPT
  # .ssh config
  $dotssh_config = <<SCRIPT
home=`getent passwd $1 | cut -d: -f6`
user=$1
sudo su - -c "cat << EOF > ${home}/.ssh/config
Host *
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null
    User $user
EOF"
sudo su - -c "chmod 600 ${home}/.ssh/config"
SCRIPT
  # .ssh permission
  $dotssh_chmod_600 = <<SCRIPT
home=`getent passwd $1 | cut -d: -f6`
sudo su - -c "chmod -R 600 ${home}/.ssh"
sudo su - -c "chmod 700 ${home}/.ssh"
SCRIPT
  # .ssh ownership
  $dotssh_chown = <<SCRIPT
home=`getent passwd $1 | cut -d: -f6`
sudo su - -c "chown -R $1:$2 ${home}/.ssh"
SCRIPT
  # give user sudo root privileges
  $user_sudo = <<SCRIPT
user=$1
sudo su - -c "echo '$user ALL = (root) NOPASSWD:ALL' > /etc/sudoers.d/$user"
sudo su - -c "chmod 0440 /etc/sudoers.d/$user"
SCRIPT
  # configure the second vagrant eth interface
  $ifcfg = <<SCRIPT
IPADDR=$1
NETMASK=$2
DEVICE=$3
TYPE=$4
cat <<END >> /etc/sysconfig/network-scripts/ifcfg-$DEVICE
NM_CONTROLLED=no
BOOTPROTO=none
ONBOOT=yes
IPADDR=$IPADDR
NETMASK=$NETMASK
DEVICE=$DEVICE
PEERDNS=no
TYPE=$TYPE
END
ARPCHECK=no /sbin/ifup $DEVICE 2> /dev/null
SCRIPT
  hosts.keys.sort.each do |host|
    if host.start_with?("server")
      config.vm.define hosts[host]['hostname'] do |server|
        server.vm.provision :shell, :inline => 'hostname ' + hosts[host]['hostname'], run: 'always'
        hosts.keys.sort.each do |k|
          server.vm.provision 'shell' do |s|
            s.inline = $etc_hosts
            s.args   = [hosts[k]['ip'], hosts[k]['hostname']]
          end
        end
        server.vm.provision :shell, :inline => $linux_disable_ipv6
        server.vm.provision :shell, :inline => $setenforce_0
        server.vm.provision :shell, :inline => $ceph_noarch_el
        # configure key-based ssh for ceph user using vagrant's keys
        server.vm.provision :file, source: '~/.vagrant.d/insecure_private_key', destination: '~vagrant/.ssh/id_rsa'
        server.vm.provision :shell, :inline => 'useradd -m ' + USER
        server.vm.provision 'shell' do |s|
          s.inline = $user_sudo
          s.args   = [USER]
        end
        server.vm.provision 'shell' do |s|
          s.inline = $key_based_ssh
          s.args   = [USER]
        end
        server.vm.provision 'shell' do |s|
          s.inline = $dotssh_chmod_600
          s.args   = [USER]
        end
        server.vm.provision 'shell' do |s|
          s.inline = $dotssh_config
          s.args   = [USER]
        end
        server.vm.provision 'shell' do |s|
          s.inline = $dotssh_chown
          s.args   = [USER, USER]
        end
        server.vm.provision 'shell' do |s|
          s.inline = $ifcfg
          s.args   = [hosts[host]['ip'], '255.255.255.0', 'eth1', 'Ethernet']
        end
        server.vm.provision :shell, :inline => 'ifup eth1', run: 'always'
        # restarting network fixes RTNETLINK answers: File exists
        server.vm.provision :shell, :inline => 'systemctl restart network', run: 'always'
        server.vm.provision :shell, :inline => 'yum -y install ceph-deploy'
        # install Ceph packages on all servers
        server.vm.provision :shell, :inline => 'ceph-deploy install --release ' + RELEASE + ' ' + hosts[host]['hostname']
        # install and enable ntp
        server.vm.provision :shell, :inline => 'yum -y install ntp'
        server.vm.provision :shell, :inline => 'systemctl enable ntpd'
        server.vm.provision :shell, :inline => 'systemctl start ntpd'
      end
    end
  end
  hosts.keys.sort.each do |host|
    if host.start_with?("client")
      config.vm.define hosts[host]['hostname'] do |client|
        client.vm.provision :shell, :inline => 'hostname ' + hosts[host]['hostname'], run: 'always'
        hosts.keys.sort.each do |k|
          client.vm.provision 'shell' do |s|
            s.inline = $etc_hosts
            s.args   = [hosts[k]['ip'], hosts[k]['hostname']]
          end
        end
        client.vm.provision :shell, :inline => $linux_disable_ipv6
        client.vm.provision :shell, :inline => $setenforce_0
        # configure key-based ssh for ceph user using vagrant's keys
        client.vm.provision :file, source: '~/.vagrant.d/insecure_private_key', destination: '~vagrant/.ssh/id_rsa'
        client.vm.provision :shell, :inline => 'useradd -m ' + USER
        client.vm.provision 'shell' do |s|
          s.inline = $user_sudo
          s.args   = [USER]
        end
        client.vm.provision 'shell' do |s|
          s.inline = $key_based_ssh
          s.args   = [USER]
        end
        client.vm.provision 'shell' do |s|
          s.inline = $dotssh_chmod_600
          s.args   = [USER]
        end
        client.vm.provision 'shell' do |s|
          s.inline = $dotssh_config
          s.args   = [USER]
        end
        client.vm.provision 'shell' do |s|
          s.inline = $dotssh_chown
          s.args   = [USER, USER]
        end
        client.vm.provision 'shell' do |s|
          s.inline = $ifcfg
          s.args   = [hosts[host]['ip'], '255.255.255.0', 'eth1', 'Ethernet']
        end
        client.vm.provision :shell, :inline => 'ifup eth1', run: 'always'
        # restarting network fixes RTNETLINK answers: File exists
        client.vm.provision :shell, :inline => 'systemctl restart network', run: 'always'
        # install and enable ntp
        client.vm.provision :shell, :inline => 'yum -y install ntp'
        client.vm.provision :shell, :inline => 'systemctl enable ntpd'
        client.vm.provision :shell, :inline => 'systemctl start ntpd'
      end
    end
  end
end
