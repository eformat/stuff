# stuff

----------------------------------------------------------
--
-- 3.6 pre-requisites, host preparation
--
-- https://docs.openshift.com/container-platform/3.6/install_config/install/prerequisites.html
-- https://docs.openshift.com/container-platform/3.6/install_config/install/host_preparation.html
-- http://playbooks-rhtconsulting.rhcloud.com/playbooks/installation

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


-----------------------------------------------------

$ htpasswd -nb admin adm-password
admin:$apr1$6CZ4noKr$IksMFMgsW5e5FL0ioBhkk/

$ htpasswd -nb developer devel-password
developer:$apr1$AvisAPTG$xrVnJ/J0a83hAYlZcxHVf1

------------------------------------------------------

-- DNS, bind

yum -y install bind bind-utils
systemctl enable named
systemctl start named

iptables -A INPUT -p tcp -m state --state NEW -m tcp --dport 53 -j ACCEPT
iptables -A INPUT -p udp -m state --state NEW -m udp --dport 53 -j ACCEPT
service iptables save


vi /var/named/dynamic/lab.com.zone 

$ORIGIN lab.com.
$TTL 86400
@ IN SOA dns1.lab.com. hostmaster.lab.com. (
 2001062501 ; serial
 21600 ; refresh after 6 hours
 3600 ; retry after 1 hour
 604800 ; expire after 1 week
 86400 ) ; minimum TTL of 1 day
;
;
 IN NS dns1.lab.com.
dns1 IN A 192.168.122.1
ose3-master IN A 192.168.122.60
*.cloudapps 300 IN A 192.168.122.60


# vi /etc/named.conf 
//
// named.conf
//
// Provided by Red Hat bind package to configure the ISC BIND named(8) DNS
// server as a caching only nameserver (as a localhost DNS resolver only).
//
// See /usr/share/doc/bind*/sample/ for example named configuration files.
//

options {
 listen-on port 53 { 127.0.0.1;192.168.122.1; };
 listen-on-v6 port 53 { ::1; };
 directory "/var/named";
 dump-file "/var/named/data/cache_dump.db";
 statistics-file "/var/named/data/named_stats.txt";
 memstatistics-file "/var/named/data/named_mem_stats.txt";
 allow-query { localhost;192.168.122.0/24;192.168.123.0/24; };

/* 
 - If you are building an AUTHORITATIVE DNS server, do NOT enable recursion.
 - If you are building a RECURSIVE (caching) DNS server, you need to enable 
 recursion. 
 - If your recursive DNS server has a public IP address, you MUST enable access 
 control to limit queries to your legitimate users. Failing to do so will
 cause your server to become part of large scale DNS amplification 
 attacks. Implementing BCP38 within your network would greatly
 reduce such attack surface 
 */
 recursion yes;

dnssec-enable yes;
 dnssec-validation yes;
 dnssec-lookaside auto;

/* Path to ISC DLV key */
 bindkeys-file "/etc/named.iscdlv.key";

managed-keys-directory "/var/named/dynamic";

pid-file "/run/named/named.pid";
 session-keyfile "/run/named/session.key";

//forward first;
 forwarders {
 //10.38.5.26;
 8.8.8.8;
 };
};

logging {
 channel default_debug {
 file "data/named.run";
 severity dynamic;
 };
};

zone "." IN {
 type hint;
 file "named.ca";
};

zone "lab.com" IN {
 type master;
 file "/var/named/dynamic/lab.com.zone";
 allow-update { none; };
};

//zone "122.168.192.in-addr.arpa" IN {
// type master;
// file "/var/named/dynamic/122.168.192.db";
// allow-update { none; };
//};

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";

# left out reverse DNS, PTR records. If you need this you can of course add
# zone file and set that up but it isn’t required for a lab configuration.

