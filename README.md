# stuff

----------------------------------------------------------
--
-- 3.6 pre-requisites, host preparation
--
-- https://docs.openshift.com/container-platform/3.6/install_config/install/prerequisites.html
-- https://docs.openshift.com/container-platform/3.6/install_config/install/host_preparation.html

-- must be enforcing
sestatus

-- NTP - enable during deploy
# openshift_clock_enabled=true

-- DNS - properly functioning DNS environment
# openshift_use_dnsmasq=true

nmcli con mod <name> ipv4.dns “10.250.64.61 10.250.64.62 192.168.110.53”
dig <node_hostname> @<IP_address> +short

-- Wildcard

*.cloudapps.example.com. 300 IN  A 192.168.133.2

-- Network
-- shared network, NetworkManager
grep "^dns=none" /etc/NetworkManager/NetworkManager.conf

-- Ports
-- https://docs.openshift.com/container-platform/3.6/install_config/install/prerequisites.html#required-ports


-- subscriptions (all hosts)
subscription-manager config --server.proxy_hostname=proxy.sg.fap --server.proxy_port=3128

subscription-manager unsubscribe --all
subscription-manager unregister
subscription-manager subscribe --pool=8a85f98159f1d69a0159f206ba0b480e

subscription-manager register --username=<user_name> --password=<password>
subscription-manager list --available --matches '*OpenShift*'
subscription-manager attach --pool=<pool_id>

# subscription-manager config --server.proxy_hostname=proxy.example.com --server.proxy_port=8080 --server.proxy_user=admin --server.proxy_password=secret

subscription-manager repos --disable="*"
yum repolist
yum-config-manager --disable \*

subscription-manager repos \
    --enable="rhel-7-server-rpms" \
    --enable="rhel-7-server-extras-rpms" \
    --enable="rhel-7-server-ose-3.6-rpms" \
    --enable="rhel-7-fast-datapath-rpms"

-- base packages
yum install wget git net-tools bind-utils iptables-services bridge-utils bash-completion kexec-tools sos psacct
yum update

-- on host that is used to install
yum install atomic-openshift-utils

-- all (hosts) if u need to configure docker storage
yum install docker-1.12.6

-- Option A) Use an additional block device.

# cat <<EOF > /etc/sysconfig/docker-storage-setup
DEVS=/dev/vdc
VG=docker-vg
EOF

docker-storage-setup

-- Option B) Use an existing, specified volume group.

# cat <<EOF > /etc/sysconfig/docker-storage-setup
VG=docker-vg
EOF
docker-storage-setup

-- Option C) Use the remaining free space from the volume group where your root file system is located.

docker-storage-setup

systemctl is-active docker
systemctl enable docker
systemctl start docker
systemctl stop docker
rm -rf /var/lib/docker/*
systemctl restart docker

-- host access (install host)
ssh-keygen
for host in master.example.com \
    node1.example.com \
    node2.example.com; \
    do ssh-copy-id -i ~/.ssh/id_rsa.pub $host; \
done

-- check firewalld
yum -y install iptables
systemctl stop firewalld
systemctl disable firewalld
systemctl start iptables
systemctl enable iptables


