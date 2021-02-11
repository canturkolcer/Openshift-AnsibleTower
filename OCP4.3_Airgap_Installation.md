# Openshift-4.3-Offline-StaticIP-Installation

- [Introduction](#introduction)
- [Create and Setup the Installation Server](#create-and-setup-the-installation-server)
- [Configure DNS Server](#configure-dns-server)
- [Configure Load Balancers](#configure-load-balancers)
- [Create Local Mirror for Offline installation](#create-local-mirror-for-offline-installation)
- [Create Ignition Files](#create-ignition-files)
- [Start Installation](#start-installation)
- [Monitor Installation](#monitor-installation)
- [Remove Bootstrap Server](#remove-bootstrap-server)
- [Accessing to WEB UI](#accessing-to-web-ui)
- [Appendix A - HTTP Proxy over SSH Session](#appendix-a---http-proxy-over-ssh-session)

## Introduction
Installing OpenShift (OCP) 4.x on an offline environment.

In this document, there are steps applied to install OCP 4.3 on offline environment with a local DNS configration. All servers will be built manually on VMware since we do not have user to automate server build. That's why, Bare metal installation on restricted network was applied from Redhat.

References:<br>
https://docs.openshift.com/container-platform/4.3/installing/installing_bare_metal/installing-restricted-networks-bare-metal.html<br>
https://docs.openshift.com/container-platform/4.3/installing/install_config/installing-restricted-networks-preparations.html#installing-restricted-networks-preparations<br>

Topology:

* Installation server with following components
  - DNS server for OCP 4.3
  - Local Mirror for offline installation
* 1 load balancer for the control control plane (master nodes)
* 1 load balancer for the compute nodes (worker nodes)
* 1 Bootstrap node
* 3 control plane nodes (master nodes)
* 5 compute nodes (worker nodes)

<br>

# Create and Setup the Installation Server

1. Create a new virtual machine with RHEL which is in same network with Openshift cluster. In this server, following packages should be installed:
  - Min. Phyton 2.7.5
  - bind (for dns setup on this server)
  - podman
  - httpd-tools
  - jq
  - 30 Gb disk space for installation files and mirror
  - internet access if possible. If not You can find ssh proxy setting below as a workaround.
  - accessible from the location where your Openshift cluster will run.

2. Either register with subscription manager or Redhat satellite so needed packages can be installed.

3. Disable the firewall and selinux (or open a hole for port 80)

  ```
  # Stop the firewall and disable it
  systemctl stop firewalld
  systemctl disable firewalld
  ```
  set selinux to passive make changes persist over a reboot
  ```
  # setenforce 0
  # sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config

  ```

4. Create a directory for installation files.

  ```
  # mkdir /ocp_depot
  ```
  It needs at least 3Gb free space. So I created it as a seperate filesystem.

  ```
  # lsblk
  ```
  Find suitable place to create filesystem. I picked sdb to keep this filesystems seperated from OS related filesystems.

  ```
  # lsblk
    NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
    sda                         8:0    0  100G  0 disk
    ├─sda1                      8:1    0  750M  0 part /boot
    └─sda2                      8:2    0 59.3G  0 part
      ├─rootvg-vloglv         253:2    0    1G  0 lvm  /var/log
      ├─rootvg-vlauditlv      253:3    0  512M  0 lvm  /var/log/audit
      ├─rootvg-varlv          253:4    0    4G  0 lvm  /var
      ├─rootvg-usrlv          253:5    0    4G  0 lvm  /usr
      ├─rootvg-tmplv          253:6    0    4G  0 lvm  /tmp
      ├─rootvg-paging00       253:7    0   16G  0 lvm  [SWAP]
      ├─rootvg-optlv          253:8    0    4G  0 lvm  /opt
      ├─rootvg-homelv         253:9    0  512M  0 lvm  /home
      ├─rootvg-auditlv        253:10   0  512M  0 lvm  /audit
      ├─rootvg-rootlv         253:11   0    2G  0 lvm  /
      ├─rootvg-corefileslv    253:12   0    1G  0 lvm  /corefiles
      ├─rootvg-factorylv      253:13   0    5G  0 lvm  /var/opt/data/swv
      ├─rootvg-hpudlv         253:14   0  256M  0 lvm  /opt/HP
      ├─rootvg-opt_mcafeelv   253:15   0    1G  0 lvm  /opt/McAfee
      ├─rootvg-cacheiemlv     253:16   0   10G  0 lvm  /var/opt/BESClient
    sdb                       8:16   0  150G  0 disk
    sr0                      11:0    1 1024M  0 rom

  # vgcreate ocpvg /dev/sdb
  Physical volume "/dev/sdb" successfully created.
  Volume group "ocpvg" successfully created

  # vgs
  VG     #PV #LV #SN Attr   VSize    VFree
  ocpvg    1   2   0 wz--n- <150.00g <139.50g
  rootvg   1  17   0 wz--n-  <59.27g  528.00m


  # pvs
  PV         VG     Fmt  Attr PSize    PFree
  /dev/sda2  rootvg lvm2 a--   <59.27g  528.00m
  /dev/sdb   ocpvg  lvm2 a--  <150.00g <139.50g

  # lvcreate -L 3G -n lv_ocp_depot ocpvg
  Logical volume "lv_ocp_depot" created.

  # lvs
  LV           VG     Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  lv_ocp_depot ocpvg  -wi-a-----   3.00g
  auditlv      rootvg -wi-ao---- 512.00m
  cacheiemlv   rootvg -wi-ao----  10.00g
  corefileslv  rootvg -wi-ao----   1.00g
  factorylv    rootvg -wi-ao----   5.00g
  homelv       rootvg -wi-ao---- 512.00m
  hpudlv       rootvg -wi-ao---- 256.00m
  opt_mcafeelv rootvg -wi-ao----   1.00g
  optlv        rootvg -wi-ao----   4.00g
  paging00     rootvg -wi-ao----  16.00g
  rootlv       rootvg -wi-ao----   2.00g
  tmplv        rootvg -wi-ao----   4.00g
  usrlv        rootvg -wi-ao----   4.00g
  varlv        rootvg -wi-ao----   4.00g
  vlauditlv    rootvg -wi-ao---- 512.00m
  vloglv       rootvg -wi-ao----   1.00g


  # mkfs.ext4 /dev/mapper/ocpvg-lv_ocp_depot

  ```
  Append following line for ocp_depot to fstab to make it available
  ```
  /dev/mapper/ocpvg-lv_ocp_depot /ocp_depot ext4 user_xattr,acl 1 2
  ```
  Finally, mount filesystem
  ```
  # mount /ocp_depot
  ```
5. Create filesystem for local mirror. Follow the same steps above and create following filesystems on same volume group 'ocpvg'.
  ```
  /var/lib/containers 512 Mb
  /opt/registry 10 Gb
  ```

6.  Download the OpenShift client, installer, ISO image and RAW disk file. Upload them to /ocp_depot directory.

  - ISO Image: https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.3/4.3.8/rhcos-4.3.8-x86_64-installer.x86_64.iso
  - RAW Disk Image: https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.3/4.3.8/rhcos-4.3.8-x86_64-metal.x86_64.raw.gz
  - Openshift Installer:https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.3.18/openshift-install-linux.tar.gz
  - Openshift Client:https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.3.18/openshift-client-linux-4.3.18.tar.gz



7. Extract OpenShift client and copy following to binaries to path:  
  ```
  # sudo cp oc /usr/local/bin/
  # sudo cp kubectl /usr/local/bin/
  ```

8. Make sure ISO is uploaded and mounted to vSphere server.

9. Create ssh keys for root user which will be used to connect Redhat CoreOS servers.

  ```
  # ssh-keygen
  ```

10. Grab your pull string from Redhat and store it on /ocp_depot (e.g. pull-secret.txt). Open the link below and click "Copy Pull String"

    https://cloud.redhat.com/openshift/install/metal/user-provisioned


## Configure DNS Server
In our setup Installation server will be our Internal DNS server for OCP 4.3 infrastructure.

We did not use customer DNS because OCP got confuses when there are multiple entries on reverse DNS lookup. Instead of removing default entries from customer's global DNS we created internal DNS server only for OCP infrastructure.

1. Install bind to the server.
  ```
  # yum install bind
  ```

2. Create local configuration file for DNS Lookup.

  /etc/bind/db.<OCPDomainPrefix>.<BaseDomain>
  * Please replace the values between < and > .
  * DO NOT remove any leading or following "."
  ```
  $TTL    604800
  @       IN      SOA     <OCPDomainPrefix>.<BaseDomain>. root.<OCPDomainPrefix>.<BaseDomain>. (
                               3         ; Serial
                           604800         ; Refresh
                            86400         ; Retry
                          2419200         ; Expire
                           604800 )       ; Negative Cache TTL
          IN      A       <DNSServer IP Address>
  ;
  @       IN      NS      ns1
  @       IN      A       <DNSServerIPAddress>
  @       IN      AAAA    ::1
  ns1     IN      A       <DNSServerIPAddress>
  gwy     IN      A       <DefaultGatewayAddress>
  <master1> IN      A       <master1 IP Address>
  <master2> IN      A       <master2 IP Address>
  <master3> IN      A       <master3 IP Address>
  <worker1> IN      A       <worker1 IP Address>
  <worker2> IN      A       <worker2 IP Address>
  <worker3> IN      A       <worker3 IP Address>
  <worker4> IN      A       <worker4 IP Address>
  <worker5> IN      A       <worker5 IP Address>
  <loadbalancer1>      IN      CNAME   api
  <loadbalancer2>        IN      A       <compute Node LoadBalancer IP Address>
  <bootstrap>       IN      A       <bootstrap Ip Address>
  <installationServer>       IN      A       <installationServer IP Address>
  api     IN      A       <loadbalancer1 IP Address>
  api-int IN      A       <loadbalancer1 IP Address>
  *.apps  IN      A       <loadbalancer2 IP Address>
  etcd-0  IN      A       <master1 IP Address>
  etcd-1  IN      A       <master2 IP Address>
  etcd-2  IN      A       <master3 IP Address>
  ; _service._proto.name.                         TTL   class SRV priority weight port target.
  _etcd-server-ssl._tcp  86400 IN    SRV 0        10     2380 etcd-0.<OCPDomainPrefix>.<BaseDomain>.
  _etcd-server-ssl._tcp  86400 IN    SRV 0        10     2380 etcd-1.<OCPDomainPrefix>.<BaseDomain>.
  _etcd-server-ssl._tcp  86400 IN    SRV 0        10     2380 etcd-2.<OCPDomainPrefix>.<BaseDomain>.

  ```
3. Create local configuration file for Reverse DNS Lookup.

  /etc/bind/db.<Ip Address' first 3 octets>
  * Please replace the values between < and > .
  * DO NOT remove any leading or following "."
  * Ip Address' first 3 octets in reverse order look like:
    for IP 192.168.1
    it is 1.168.192
  ```
  $TTL    86400 ; 24 hours, could have been written as 24h or 1d
  ; $ORIGIN <Ip Address' first 3 octets in reverse order>.IN-ADDR.ARPA.
  @    IN  SOA ns1.<OCPDomainPrefix>.<BaseDomain>.      root.<OCPDomainPrefix>.<BaseDomain>. (
                                11 ; serial
                                3H ; refresh
                                15 ; retry
                                1w ; expire
                                3h ; minimum
                               )
  ; Name servers for the zone - both out-of-zone - no A RRs required
         IN  NS ns1.<OCPDomainPrefix>.<BaseDomain>.
  ; Infrastructure
  $ORIGIN <Ip Address' first 3 octets in reverse order>.IN-ADDR.ARPA.
  <Last octet of GW>       IN  PTR    <Default Gateway FQDN>.
  <Last octet of master 1>       IN  PTR    <master 1 FQDN with OCP Domain>.
  <Last octet of master 2>       IN  PTR    <master 2 FQDN with OCP Domain>.
  <Last octet of master 3>       IN  PTR    <master 3 FQDN with OCP Domain>.
  <Last octet of worker 1>       IN  PTR    <worker 1 FQDN with OCP Domain>.
  <Last octet of worker 2>       IN  PTR    <worker 2 FQDN with OCP Domain>.
  <Last octet of worker 3>       IN  PTR    <worker 3 FQDN with OCP Domain>.
  <Last octet of worker 4>       IN  PTR    <worker 4 FQDN with OCP Domain>.
  <Last octet of worker 5>       IN  PTR    <worker 5 FQDN with OCP Domain>.
  <Last octet of loadbalancer1>        IN  PTR    <OCPDomainPrefix>.<BaseDomain>.
  <Last octet of loadbalancer1>       IN  PTR    <OCPDomainPrefix>.<BaseDomain>.
  <Last octet of loadbalancer2>       IN  PTR    <OCPDomainPrefix>.<BaseDomain>.
  <Last octet of bootstrap>       IN  PTR    <bootstrap FQDN with OCP Domain>.
  <Last octet of installation server>       IN  PTR    <Installation Server FQDN with OCP Domain>.  
  ```

4. Create Local configuration file and include your configuration files in this file.

  /etc/named.conf.local
  * Please replace the values between < and > .
  * DO NOT remove any leading or following "."
  ```
  //
  // Do any local configuration here
  //


  zone "<OCPDomainPrefix>.<BaseDomain>" { type master; file "/etc/bind/db.<OCPDomainPrefix>.<BaseDomain>"; };
  zone "<Ip Address' first 3 octets in reverse order>.in-addr.arpa" { type master; file "/etc/bind/db.<Ip Address' first 3 octets>"; };
  ```

5. Edit /etc/named.conf
  - add server ip to 'listen-on' ip list
  ```
  FROM:
  listen-on port 53 { 127.0.0.1; };
  TO:
  listen-on port 53 { 127.0.0.1; <DNSServerIPAddress>; };

  FROM:
  allow-query     { localhost; }; or allow-query     { any; };
  TO:
  allow-query     { localhost; <OCPDomain Subnet>; };
  ```
  - include local configuration with appending below line to file:
  ```
  include "/etc/named.conf.local";
  ```

6. Restart named services (restart if already started)
  ```
  # systemctl start named
  ```

7. Test DNS settings:
  * Please replace the values between < and > .
  * DO NOT remove any leading or following "."

  ```
  # dig @<DNS Server IP Address> <serverName> +short
  ```
  Test Reverse DNS settings:
  ```
  # dig -x <IP Address> +short
  ```
  Each should return only one result except SRV records. Any multiple line may create issues during installations.

  Example queries:
  ```
  dig @<DNS Server IP Address> bootstrap.<OCPDomainPrefix>.<BaseDomain>  +short
  dig @<DNS Server IP Address> <master1>.<OCPDomainPrefix>.<BaseDomain> +short
  dig @<DNS Server IP Address> <master2>.<OCPDomainPrefix>.<BaseDomain> +short
  dig @<DNS Server IP Address> <master3>.<OCPDomainPrefix>.<BaseDomain> +short
  dig @<DNS Server IP Address> <worker1>.<OCPDomainPrefix>.<BaseDomain> +short
  dig @<DNS Server IP Address> <worker2>.<OCPDomainPrefix>.<BaseDomain> +short
  dig @<DNS Server IP Address> <worker3>.<OCPDomainPrefix>.<BaseDomain> +short
  dig @<DNS Server IP Address> <worker4>.<OCPDomainPrefix>.<BaseDomain> +short
  dig @<DNS Server IP Address> <worker5>.<OCPDomainPrefix>.<BaseDomain> +short
  dig @<DNS Server IP Address> api.<OCPDomainPrefix>.<BaseDomain> +short
  dig @<DNS Server IP Address> api-int.<OCPDomainPrefix>.<BaseDomain> +short
  dig @<DNS Server IP Address> *.apps.<OCPDomainPrefix>.<BaseDomain> +short
  dig @<DNS Server IP Address> etcd-0.<OCPDomainPrefix>.<BaseDomain> +short
  dig @<DNS Server IP Address> etcd-1.<OCPDomainPrefix>.<BaseDomain> +short
  dig @<DNS Server IP Address> etcd-2.<OCPDomainPrefix>.<BaseDomain> +short
  dig @<DNS Server IP Address> _etcd-server-ssl._tcp.<OCPDomainPrefix>.<BaseDomain> SRV +short

  dig @<DNS Server IP Address> -x <bootstrap IP Address> +short
  dig @<DNS Server IP Address> -x <installation server IP Address> +short
  dig @<DNS Server IP Address> -x <master1 IP Address> +short
  dig @<DNS Server IP Address> -x <master2 IP Address> +short
  dig @<DNS Server IP Address> -x <master3 IP Address> +short
  dig @<DNS Server IP Address> -x <worker1 IP Address> +short
  dig @<DNS Server IP Address> -x <worker2 IP Address> +short
  dig @<DNS Server IP Address> -x <worker3 IP Address> +short
  dig @<DNS Server IP Address> -x <worker4 IP Address> +short
  dig @<DNS Server IP Address> -x <worker5 IP Address> +short
  ```

## Configure Load Balancers

1. Install haproxy on both loadbalancer servers.
  ```
  # yum install haproxy
  ```

2. Create user for haproxy without any home directory.
  ```
  # useradd -M haproxy
  ```

3. On first load balancer server which will be used as master node's haproxy; edit haproxy config and add following lines

  /etc/haproxy/haproxy.cfg
  * Please replace the values between < and > .
  ```
  frontend machine-config-server
    bind *:22623
    default_backend machine-config-server
    mode tcp
    option tcplog

  backend machine-config-server
    balance source
    mode tcp
    server bootstrap <bootstrap IP Address>:22623 check
    server master1 <master1 IP Address>:22623 check
    server master2 <master2 IP Address>:22623 check
    server master3 <master3 IP Address>:22623 check

  frontend openshift-api-server
    bind *:6443
    default_backend openshift-api-server
    mode tcp
    option tcplog

  backend openshift-api-server
    balance source
    mode tcp
    server bootstrap <bootstrap IP Address>:6443 check
    server master1 <master1 IP Address>:6443 check
    server master2 <master2 IP Address>:6443 check
    server master3 <master3 IP Address>:6443 check
  ```

4. On second load balancer server which will be used as worker node's haproxy; edit haproxy config and add following lines

  /etc/haproxy/haproxy.cfg
  * Please replace the values between < and > .
  ```
  frontend ingress-http
    bind *:80
    default_backend ingress-http
    mode tcp
    option tcplog

  backend ingress-http
    balance source
    mode tcp
    server worker1 <worker1 IP Address>:80 check
    server worker2 <worker2 IP Address>:80 check
    server worker3 <worker3 IP Address>:80 check
    server worker4 <worker4 IP Address>:80 check
    server worker5 <worker5 IP Address>:80 check

  frontend ingress-https
    bind *:443
    default_backend ingress-https
    mode tcp
    option tcplog

  backend ingress-https
   balance source
   mode tcp
    server worker1 <worker1 IP Address>:443 check
    server worker2 <worker2 IP Address>:443 check
    server worker3 <worker3 IP Address>:443 check
    server worker4 <worker4 IP Address>:443 check
    server worker5 <worker5 IP Address>:443 check
  ```  

5. Restart haproxy service on both servers and check the status.

  ```
  # systemctl restart haproxy
  # systemctl status haproxy
  ```  
## Create Local Mirror for Offline installation

1. Install necessary packages with using yum
  ```
  # yum -y install podman httpd-tools jq
  ```

2. /opt/registry and /var/lib/containers directories we created before will be used here.

3. Create key pair for registry
  ```
  # cd /opt/registry/certs
  # openssl req -newkey rsa:4096 -nodes -sha256 -keyout domain.key -x509 -days 365 -out domain.crt
    Generating a 4096 bit RSA private key
    .....................................
    writing new private key to 'domain.key'
    -----
    You are about to be asked to enter information that will be incorporated
    into your certificate request.
    What you are about to enter is what is called a Distinguished Name or a DN.
    There are quite a few fields but you can leave some blank
    For some fields there will be a default value,
    If you enter '.', the field will be left blank.
    -----
    Country Name (2 letter code) [XX]:
    State or Province Name (full name) []:
    Locality Name (eg, city) [Default City]:
    Organization Name (eg, company) [Default Company Ltd]:
    Organizational Unit Name (eg, section) []:
    Common Name (eg, your name or your server's hostname) []:
    Email Address []:
  ```
  - **Country Name (2 letter code):** Specify the two-letter ISO country code for your location. See the ISO 3166 country codes standard.
  - **State or Province Name (full name):** Enter the full name of your state or province.
  - **Locality Name (eg, city):** Enter the name of your city.
  - **Organization Name (eg, company):** Enter your company name.
  - **Organizational Unit Name (eg, section):** Enter your department name.
  - **Common Name (eg, your name or your server’s hostname):**  Enter the host name for -the registry host. Ensure that your hostname is in DNS and that it resolves to the expected IP address.
  - **Email Address:**  Enter your email address. For more information, see the req description in the OpenSSL documentation.

4. Generate username and password for registry and create base64 encrypted string
  ```
  # htpasswd -bBc /opt/registry/auth/htpasswd <registry username> <registry password>
  # echo -n "<registry username>:<registry password>" | base64 -w0
  <This string will be used later in yaml creation>
  ```

5. Create the mirror-registry container to host your registry
  ```
  # podman run --name mirror-registry -p 5000:5000 -v /opt/registry/data:/var/lib/registry:z -v /opt/registry/auth:/auth:z -e "REGISTRY_AUTH=htpasswd" -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd -v /opt/registry/certs:/certs:z -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key -d docker.io/library/registry:2
  ```

6. Add the self-signed certificate to your list of trusted certificates
  ```
  # cp /opt/registry/certs/domain.crt /etc/pki/ca-trust/source/anchors/
  # update-ca-trust
  ```
7. Test if the repository is reachable or not.
  ```
  # curl -u oc4registry:0c4registry -k https://<InstallationServerFQDN>:5000/v2/_catalog
    {"repositories":[]}
  ```
8. Create a copy of your pull-secret file for mirror.
  ```
  # cat /ocp_depot/pull-secret | jq . > /ocp_depot/pull-secret-mirror
  ```

9. Add following line before "registry.redhat.io" entry
  ```
  "<InstallationServerFQDN>:5000": {
    "auth": "<base64 encoded string for registry created at step 4>",
    "email": "<your email>"
  },  
  ```
10. Export needed variables.

  **For OCP release you can find your version from quay.io:**

  https://quay.io/repository/openshift-release-dev/ocp-release?tab=tags

  ```
  # export OCP_RELEASE=4.3.18-x86_64
  # export LOCAL_REGISTRY='<InstallationServerFQDN>:5000'
  # export LOCAL_REPOSITORY='ocp4/openshift4'
  # export PRODUCT_REPO='openshift-release-dev'
  # export LOCAL_SECRET_JSON='/ocp_depot/pull-secret-mirror'
  # export RELEASE_NAME="ocp-release"
  ```
11. [Optional] Check your proxy settings if installation server does not have internet access.
  ```
  # env|grep -i proxy


  # export HTTP_PROXY=http://<ProxyIP>:8080
  # export HTTPS_PROXY=http://<ProxyIP>:8080
  # export NO_PROXY=..<OCPDomainPrefix>.<BaseDomain>
  ```

12. Start downloading content. This will take long depending to your bandwidth.
  ```
  # oc adm -a ${LOCAL_SECRET_JSON} release mirror --from=quay.io/${PRODUCT_REPO}/${RELEASE_NAME}:${OCP_RELEASE} --to=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY} --to-release-image=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY}:${OCP_RELEASE}

  ..........
  ................
  Update image:  <InstallationServerFQDN>:5000/ocp4/openshift4:4.3.18-x86_64
  Mirror prefix: <InstallationServerFQDN>:5000/ocp4/openshift4

  To use the new mirrored repository to install, add the following section to the install-config.yaml:

  imageContentSources:
  - mirrors:
    - <InstallationServerFQDN>:5000/ocp4/openshift4
    source: quay.io/openshift-release-dev/ocp-release
  - mirrors:
    - <InstallationServerFQDN>:5000/ocp4/openshift4
    source: quay.io/openshift-release-dev/ocp-v4.0-art-dev


  To use the new mirrored repository for upgrades, use the following to create an ImageContentSourcePolicy:

  apiVersion: operator.openshift.io/v1alpha1
  kind: ImageContentSourcePolicy
  metadata:
    name: example
  spec:
    repositoryDigestMirrors:
    - mirrors:
      - <InstallationServerFQDN>:5000/ocp4/openshift4
      source: quay.io/openshift-release-dev/ocp-release
    - mirrors:
      - <InstallationServerFQDN>:5000/ocp4/openshift4
      source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
  ```

13. Create new openshift-installer for the mirror you created:  
  ```
  # cd /ocp_depot
  # oc adm -a ${LOCAL_SECRET_JSON} release extract --command=openshift-install "${LOCAL_REGISTRY}/${LOCAL_REPOSITORY}:${OCP_RELEASE}"
  ```
## Create Ignition Files
1. Prepare install-config.yaml file under /ocp_depot directory.

  For detailed information about yaml creation please refer to Redhat's official documentation:

  https://docs.openshift.com/container-platform/4.3/installing/installing_bare_metal/installing-restricted-networks-bare-metal.html#installation-bare-metal-config-yaml_installing-restricted-networks-bare-metal

  * Please replace the values between < and > .
  ```
  apiVersion: v1
  baseDomain: <BaseDomain>

  compute:
  - hyperthreading: Enabled
    name: worker
    replicas: <Number of workers will be deployed>

  controlPlane:
    hyperthreading: Enabled
    name: master
    replicas: <Number of masters will be deployed>

  metadata:
    name: <OCPDomain>

  networking:

    clusterNetwork:
    - cidr: 10.128.0.0/14
      hostPrefix: 23
    networkType: OpenShiftSDN
    serviceNetwork:
    - 172.30.0.0/16

  platform:

    none: {}

  pullSecret: '<Enter mirror pull-secret created during mirror creation. Also stored in /ocp_depot/pull-secret-mirror file >'

  additionalTrustBundle: |
    <Paste content of /opt/registry/certs/domain.crt>


  sshKey: '<Paste content of ~/.ssh/id_rsa.pub>'

  imageContentSources: <Paste output of mirror downloading command result>


  ```
2. Run openshift-installer created at the end of mirror creation to validate. If you skip this step you may have faulty ignition files which will affect OS creation.

  ```
  # openshift-install create install-config
  ```

3. Create manifest files.
  ```
  # openshift-install create manifests
  ```

4. Change masterschedulables to false
  ```
  # sed -i 's/mastersSchedulable: true/mastersSchedulable: false/g' manifests/cluster-scheduler-02-config.yml
  ```

5. Create ignition files.
  ```
  # openshift-install create ignition-configs
  ```

  ***Important Note: Please Remember ignition files are valid for 24 hours. You need to recreate if you want to use them after 24 hours***

6. Start HTTP server for sharing ignition files and RAW disk file.
  ```
  # cd /ocp_depot
  # python -m SimpleHTTPServer
  ```

## Start Installation
1. Prepare virtual server without any operating system. And assign cpu, memory, disk and network cards on them.

  | Server Role | CPU | RAM | OS Space | Application Space | Count |
  | ----------- | --- | --- | -------- | ----------------- | ----- |
  | bootstrap   | 16  | 32  |  100 GB  |         -         |   1   |  
  | master node | 12  | 64  |  100 GB  |       150 GB      |   3   |
  | worker node | 16  | 32  |  100 GB  |       150 GB      |   5   |

2. Mount ISO image as a CD drive to all 9 servers.
3. Boot servers and on boot screen press **TAB** and enter boot variables. Hit enter to start deployment. For Worker nodes, it is better to wait until all masters are ready.

  ***Important! All parameters should have in one line***

  **Bootstrap Server**
  ```
  ip=<bootstrap IPAddress>::<DefaultGatewayAddress>:<SUBNET Mask>:<bootstrap hostname>.<OCPDomain>.<BaseDomain>:<Network Interface>:none nameserver=<DNS Server IP - Installation Server IP> coreos.inst.install_dev=<device name to install COREOS> coreos.inst.image_url=http://<Installation Server IP>:8000/<RAW image name> coreos.inst.ignition_url=http://<Installation Server IP>:8000/bootstrap.ign
  ```
  **Master Server**
  ```
  ip=<master IPAddress>::<DefaultGatewayAddress>:<SUBNET Mask>:<master hostname>.<OCPDomain>.<BaseDomain>:<Network Interface>:none nameserver=<DNS Server IP - Installation Server IP> coreos.inst.install_dev=<device name to install COREOS> coreos.inst.image_url=http://<Installation Server IP>:8000/<RAW image name> coreos.inst.ignition_url=http://<Installation Server IP>:8000/master.ign
  ```
  **Worker Server**
  ```
  ip=<worker IPAddress>::<DefaultGatewayAddress>:<SUBNET Mask>:<worker hostname>.<OCPDomain>.<BaseDomain>:<Network Interface>:none nameserver=<DNS Server IP - Installation Server IP> coreos.inst.install_dev=<device name to install COREOS> coreos.inst.image_url=http://<Installation Server IP>:8000/<RAW image name> coreos.inst.ignition_url=http://<Installation Server IP>:8000/worker.ign
  ```
## Monitor installation

1. Openshift installer on isntallation server can be used till bootstrapping is completed.
  ```
  openshift-install --dir=/ocp_depot wait-for bootstrap-complete --log-level=debug
  ```
2. On bootstrap server, Folowing command can be run to track installation and error. Please be aware, some error can be seen until master nodes are up.
  ```
  journalctl -b -f -u bootkube.service
  ```
  *You can also run this command without "-u" so you will see all services on COREOS server. It is useful on master and worker nodes.*

3. After bootstrap server is ready and install api-int master nodes will be booted up.Worker nodes will come later. Here is a example timeline from my installation to give an idea.
  ```
  - bootstrap booted up (started running). Let's say 't'
  - master1 booted after t+2 minutes (waiting for api-int). Up after t+15 minutes.
  - master2 booted after t+4 minutes (waiting for api-int). Up after t+15 minutes.
  - master3 booted after t+6 minutes (waiting for api-int). Up after t+15 minutes.
  - worker1 booted after t+30 minutes (waiting for api-int). Up after t+35 minutes.
  - worker2 booted after t+30 minutes (waiting for api-int). Up after t+35 minutes.
  - worker3 booted after t+30 minutes (waiting for api-int). Up after t+35 minutes.
  - worker4 booted after t+30 minutes (waiting for api-int). Up after t+35 minutes.
  - worker5 booted after t+30 minutes (waiting for api-int). Up after t+35 minutes.
  - API started after t+15 minutes.
  - Bootstrap completed after t+25 minutes.
  - Cluster was ready after t+45 minutes.
  ```
4. During the installation i also recommend to run following command to see how the cluster looks like at that moment.
  ```
  watch -n5 'oc get clusteroperators ; oc get nodes ; oc get clusterversion'
  ```

5. In case of any issues you can collect all logs from bootstrap and master nodes with following command.
  ```
  openshift-install gather bootstrap --bootstrap <Bootstrap IP> --master <master1 IP> --master <master2 IP> --master <master3 IP>
  ```

6. I also checked CSR's if they are approved or not. Sometimes it stucks and you can approve them manually. It will not affect installation than speeding it up.
  - To see CSR records and their statues:
  ```
  oc get csr
  ```
  - To approve all
  ```
  oc get csr -ojson | jq -r '.items[] | select(.status == {} ) | .metadata.name' | xargs oc adm certificate approve
  ```

7. If you have any operator not started you can check its logs.
  ```
  oc describe clusteroperators <operatorName>
  ```

8. You can also check all pods and where they are running.
  ```
  oc get pods -A -o wide
  ```
## Remove Bootstrap Server

1. Edit haproxy configuration (/etc/haproxy/haproxy.cfg) on master node's load balancer and comment out or delete bootstrap entries .
  ```
  backend machine-config-server
    balance source
    mode tcp
    #server bootstrap <bootstrap IP Address>:22623 check
    server master1 <master1 IP Address>:22623 check
    server master2 <master2 IP Address>:22623 check
    server master3 <master3 IP Address>:22623 check

  frontend openshift-api-server
    bind *:6443
    default_backend openshift-api-server
    mode tcp
    option tcplog

  backend openshift-api-server
    balance source
    mode tcp
    #server bootstrap <bootstrap IP Address>:6443 check
    server master1 <master1 IP Address>:6443 check
    server master2 <master2 IP Address>:6443 check
    server master3 <master3 IP Address>:6443 check  
  ```

2. Check if everything works fine or not. I recommend to keep bootstrap running for at least 24 hours -till the certificates renewed- if you do not need the resources.

## Accessing to WEB UI

1. Run following command to see all routes.
  ```
  $ oc get routes -A | grep -iE 'console-openshift|oauth-openshift|prometheus' | awk '{print $3}'
  oauth-openshift.apps.<OCPDomainPrefix>.<BaseDomain>
  console-openshift-console.<OCPDomainPrefix>.<BaseDomain>
  prometheus-k8s-openshift-monitoring.<OCPDomainPrefix>.<BaseDomain>
  ```

2. Edit /etc/hosts and add following lines on workstations you want to reach OCP UI's.
  ```
  <worker node's haproxy IP> oauth-openshift.apps.<OCPDomainPrefix>.<BaseDomain>
  <worker node's haproxy IP> console-openshift-console.<OCPDomainPrefix>.<BaseDomain>
  <worker node's haproxy IP> prometheus-k8s-openshift-monitoring.<OCPDomainPrefix>.<BaseDomain>  
  ```

3. You can access now with using *kubeadmin* user. Password can be found on installation server at "/ocp_depot/auth/kubeadmin-password"

----
# Appendix
## Appendix A - HTTP Proxy over SSH Session

*This is a workaround to provide internet over SSH to installation server to create mirror.*

1. Install and start squid application to the server where you will establish SSH session.
  ```
  yum install -y squid
  systemctl start squid
  ```

2. Create config file under ssh directory of the user you will establish connection.

  ```
  vi ~/.ssh/config
  ```
3. Add following lines to config file.
  ```
  Host <Server hostname or ip which will be used as http proxy>
  User <user will be used to connect http proxy server>
  RemoteForward 0.0.0.0:8080 127.0.0.1:3128
  ServerAliveInterval 60
  ```

4. Change permissions of config file to "600".
  ```
  chmod 600 ~/.ssh/config
  ```
4. Connect to your http proxy server. Check sshd_config to be sure it can share internet with OCP environment.
  ```
  # cat /etc/ssh/sshd_config  | grep AllowTcpForwarding
  AllowTcpForwarding yes
  ```

  If not; change it and reload sshd service.
  ```
  systemctl reload sshd
  ```

5. Create proxy.sh file under profile.d to make proxy available for all sessions.
  ```
  # cat /etc/profile.d/proxy.sh
  export http_proxy=http://127.0.0.1:8080/
  export https_proxy=http://127.0.0.1:8080/
  export no_proxy=.<BaseDomain>,10.0.0.0/8,127.0.0.1,localhost,.cluster.local,.svc,172.30.0.1,
  ```

6. Logout and login and check if environment has proxy defined or not.
  ```
  env | grep -i proxy
  ```
7. Test connection.
  - working proxy
  ```
  # curl https://registry.redhat.io/v2/
  {"errors":[{"code":"UNAUTHORIZED","message":"Access to the requested resource is not authorized"}]}
  ```
  - Not working Proxy
  ```
  # curl https://registry.redhat.io/v2/
  curl: (6) Could not resolve host: registry.redhat.io; Unknown error

  ```
