# stuff

```

----------------------------------------------------------
--
-- 3.6 pre-requisites, host preparation
--
-- https://docs.openshift.com/container-platform/3.6/install_config/install/prerequisites.html
-- https://docs.openshift.com/container-platform/3.6/install_config/install/host_preparation.html
-- http://playbooks-rhtconsulting.rhcloud.com/playbooks/installation
-- https://keithtenzer.com/2016/08/04/openshift-enterprise-3-2-all-in-one-lab-environment/

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
dns1        IN  A  192.168.122.1
ose3-master IN  A  192.168.122.60
*.cloudapps IN  A  192.168.122.60


-- may have multiple A records, client side dns round robin
*      IN      A       192.168.137.100
       IN      A       192.168.137.101

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

-----------

-- BIND DNS Setup

--
-- example bind files. adjust paths to files for your environment
-- please setup dns1, hostmaster, SOA details as approriate for your environment
--

vi /var/named/dynamic/openshift.lab.com.zone 

$ORIGIN openshift.lab.com.
$TTL 86400
@ IN SOA dns1.openshift.lab.com. hostmaster.openshift.lab.com. (
 2001062501 ; serial
 21600 ; refresh after 6 hours
 3600 ; retry after 1 hour
 604800 ; expire after 1 week
 86400 ) ; minimum TTL of 1 day
;
;
 IN NS dns1.openshift.lab.com.
dns1        IN  A  XXX.XXX.XXX.XXX
master      IN  A  10.250.90.29
*.apps      IN  A  10.250.90.24


vi /etc/named.conf

zone "lab.com" IN {
   type master;
   file "/var/named/dynamic/openshift.lab.com.zone";
   allow-update { none; };
};


# left out reverse DNS, PTR records. If you need this you can of course add
# zone file and set that up but it isn’t required for a lab configuration


----------------------------------------------

$ for dc in $(oc get deploymentconfig --selector logging-infra=elasticsearch -o name); do
    oc set volume $dc \
          --add --overwrite --name=elasticsearch-storage \
          --type=hostPath --path=/usr/local/es-storage
    oc rollout latest $dc
    oc scale $dc --replicas=1
  done


--------------------------------------------------

-- nfs from bastion 

-- via console, add disk to bastion

-- on bastion
iptables -I INPUT -p tcp -m state --state NEW -m tcp --dport 111 -j ACCEPT
iptables -I INPUT -p tcp -m state --state NEW -m tcp --dport 2049 -j ACCEPT
iptables -I INPUT -p tcp -m state --state NEW -m tcp --dport 20048 -j ACCEPT
iptables -I INPUT -p tcp -m state --state NEW -m tcp --dport 50825 -j ACCEPT
iptables -I INPUT -p tcp -m state --state NEW -m tcp --dport 53248 -j ACCEPT
iptables-save > /etc/sysconfig/iptables

sed -i -e 's/^RPCMOUNTDOPTS.*/RPCMOUNTDOPTS="-p 20048"/' -e 's/^STATDARG.*/STATDARG="-p 50825"/' /etc/sysconfig/nfs

echo "Setting sysctl params..."
grep -q fs.nlm /etc/sysctl.conf
if [ $? -eq 1 ]
then
  sed -i -e '$afs.nfs.nlm_tcpport=53248' -e '$afs.nfs.nlm_udpport=53248' /etc/sysctl.conf
fi
systemctl enable rpcbind nfs-server
sysctl -p
systemctl start rpcbind nfs-server nfs-lock nfs-idmap
systemctl stop firewalld
systemctl disable firewalld

-- nodes/selinux can mount nfs
ansible "ocp-node-*" -m shell -a 'setsebool -P virt_use_nfs=true'
ansible "ocp-infra-*" -m shell -a 'setsebool -P virt_use_nfs=true'

-- create thin lvm for eack vol
yum -y install lvm2*
pvcreate /dev/disk/by-id/google-ocp-nfs-disk-1
vgcreate ocp-nfs-disk-1 /dev/sdb
lvcreate -l 100%FREE --type thin-pool --thinpool thin_pool ocp-nfs-disk-1
vgchange -ay -K ocp-nfs-disk-1

for x in {1..400}; do
  #echo $x;
  lvcreate -V1G -T ocp-nfs-disk-1/thin_pool --name pv$x
  mkdir -p /mnt/ocp-nfs-disk-1/pv$x
  chmod -R 777 /mnt/ocp-nfs-disk-1/pv$x
  mke2fs -t ext4 /dev/ocp-nfs-disk-1/pv$x  
  mount /dev/ocp-nfs-disk-1/pv$x /mnt/ocp-nfs-disk-1/pv$x
  echo "/mnt/ocp-nfs-disk-1/pv$x *(rw,no_root_squash,no_wdelay,sync)" >> /etc/exports  
  echo "/dev/ocp-nfs-disk-1/pv$x /mnt/ocp-nfs-disk-1/pv$x  ext4     defaults,nofail        0 0" >> /etc/fstab
done

vgchange -ay -K ocp-nfs-disk-1
mount -a
exportfs -va
chmod -R 777 /mnt/ocp-nfs-disk-1

-- as cluster admin
oc project default

for x in {1..400}; do
oc create -f - <<EOF
 {
      "apiVersion": "v1",
      "kind": "PersistentVolume",
      "metadata": {
        "name": "pv$x"
      },
      "spec": {
        "capacity": {
            "storage": "1Gi"
           },
        "accessModes": [ "ReadWriteOnce","ReadWriteMany","ReadOnlyMany" ],
        "nfs": {
            "path": "/mnt/ocp-nfs-disk-1/pv$x",
            "server": "ocp-bastion"
        },
        "persistentVolumeReclaimPolicy": "Retain"
      }
    }
EOF
done

-- test it out
oc new-project nexus --display-name="Nexus" --description="Nexus"
oc new-app -f https://raw.githubusercontent.com/eformat/openshift-nexus/master/nexus.yaml -p VOLUME_CAPACITY=5Gi


--- install playbook

ansible-playbook -i static-inventory /usr/share/ansible/openshift-ansible/playbooks/byo/config.yml -e openshift_disable_check=disk_availability,docker_storage,memory_availability'

-- set admin user as cluster admin
oc adm policy add-cluster-role-to-group sudoer system:authenticated \
      --config="/root/openshift-3.6.local.config/master/admin.kubeconfig" \
      --context="default/192-168-137-2:8443/system:admin"

oc adm policy add-cluster-role-to-user cluster-admin admin --as=system:admin


-- general ansible commands
ansible -m ping "all"
ansible "master*" -m shell -a 'systemctl restart etcd.service'
ansible "master*" -m shell -a 'systemctl restart atomic-openshift-master-api.service'
ansible "master*" -m shell -a 'systemctl restart atomic-openshift-master-controllers.service'
ansible "all" -m shell -a 'systemctl restart atomic-openshift-node.service'


-- add users
ansible "master*" -m shell -a 'for x in {0..40}; do y=`printf "%03d" $x`; htpasswd -b /etc/origin/master/htpasswd user$y user$y; done'
FIRST_MASTER=master1
ansible "${FIRST_MASTER}" -m shell -a 'for x in {0..40}; do y=`printf "%03d" $x`; oadm policy add-role-to-user system:registry user$y; done'

-- managment token

oc serviceaccounts get-token -n management-infra management-admin

oc adm new-project management-infra --description="Management Infrastructure"
oc create -n management-infra -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: management-admin
EOF
oc create -f - <<EOF
apiVersion: v1
kind: ClusterRole
metadata:
  name: management-infra-admin
rules:
- resources:
  - pods/proxy
  verbs:
  - '*'
EOF
oc adm policy add-role-to-user -n management-infra admin -z management-admin
oc adm policy add-role-to-user -n management-infra management-infra-admin -z management-admin
oc adm policy add-cluster-role-to-user cluster-reader system:serviceaccount:management-infra:management-admin
oc adm policy add-scc-to-user privileged system:serviceaccount:management-infra:management-admin


--
oc create serviceaccount 'management-admin' -n 'management-infra'
oc adm policy add-role-to-user -n management-infra admin -z management-admin
oc adm policy add-role-to-user -n management-infra management-infra-admin -z management-admin
oc adm policy add-cluster-role-to-user cluster-reader system:serviceaccount:management-infra:management-admin
oc adm policy add-cluster-role-to-user system:image-auditor system:serviceaccount:management-infra:management-admin
oc adm policy add-scc-to-user privileged system:serviceaccount:management-infra:management-admin

oc create serviceaccount 'inspector-admin' -n 'management-infra'
oc adm policy add-cluster-role-to-user system:image-puller system:serviceaccount:management-infra:inspector-admin
oc adm policy add-scc-to-user privileged system:serviceaccount:management-infra:inspector-admin
oc adm policy add-cluster-role-to-user self-provisioner system:serviceaccount:management-infra:management-admin


-- Env.Preparation
ansible "node*" -m shell -a 'docker pull docker.io/openshift/jenkins-2-centos7 &'
ansible "node*" -m shell -a 'docker pull docker.io/openshift/jenkins-slave-nodejs-centos7 &'
ansible "node*" -m shell -a 'docker pull docker.io/openshift/jenkins-slave-maven-centos7 &'
ansible "node*" -m shell -a 'docker pull registry.access.redhat.com/rhscl/mongodb-32-rhel7 &'
ansible "node*" -m shell -a 'docker pull registry.access.redhat.com/jboss-fuse-6/fis-java-openshift &'
ansible "node*" -m shell -a 'docker pull registry.access.redhat.com/rhscl/nodejs-6-rhel7 &'
ansible "node*" -m shell -a 'docker pull registry.access.redhat.com/rhscl/nodejs-4-rhel7 &'
ansible "node*" -m shell -a 'docker pull registry.access.redhat.com/rhscl/php-70-rhel7 &'
ansible "node*" -m shell -a 'docker pull registry.access.redhat.com/rhscl/php-56-rhel7 &'
ansible "node*" -m shell -a 'docker pull registry.access.redhat.com/redhat-openjdk-18/openjdk18-openshift &'

ansible "node*" -m shell -a 'docker pull registry.access.redhat.com/openshift3/jenkins-slave-maven-rhel7 &'
ansible "node*" -m shell -a 'docker pull registry.access.redhat.com/openshift3/jenkins-slave-nodejs-rhel7 &'

ansible "node*" -m shell -a 'docker pull registry.access.redhat.com/openshift3/ose-sti-builder:v3.6.173.0.21 &'

ansible "node*" -m shell -a 'docker pull registry.access.redhat.com/dotnet/dotnet-20-rhel7 &'

ansible "node*" -m shell -a 'docker pull docker.io/mattf/workshop &'
ansible "node*" -m shell -a 'docker pull docker.io/radanalyticsio/base-notebook &'

ansible "node*" -m shell -a 'docker pull docker.io/radanalyticsio/radanalytics-java-spark:stable &'


-- jenkins image streams
oc import-image --all --insecure=true --confirm -n openshift docker.io/openshift/jenkins-2-centos7
oc import-image --all --insecure=true --confirm -n openshift registry.access.redhat.com/openshift3/jenkins-2-rhel7
oc import-image --all --insecure=true --confirm -n openshift docker.io/openshift/jenkins-slave-nodejs-centos7
oc import-image --all --insecure=true --confirm -n openshift docker.io/openshift/jenkins-slave-maven-centos7

-- jenkins templates
oc apply -f https://raw.githubusercontent.com/openshift/openshift-ansible/master/roles/openshift_examples/files/examples/v3.6/quickstart-templates/jenkins-ephemeral-template.json -n openshift
oc apply -f https://raw.githubusercontent.com/openshift/openshift-ansible/master/roles/openshift_examples/files/examples/v3.6/quickstart-templates/jenkins-persistent-template.json -n openshift

-- mongo db for application
oc apply -f https://raw.githubusercontent.com/openshift/openshift-ansible/master/roles/openshift_examples/files/examples/v3.6/db-templates/mongodb-persistent-template.json -n openshift
oc apply -f https://raw.githubusercontent.com/openshift/origin/master/examples/jenkins/pipeline/samplepipeline.yaml -n openshift


```
