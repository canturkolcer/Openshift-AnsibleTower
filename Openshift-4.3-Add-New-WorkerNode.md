# Openshift-4.3-Add-Remove-WorkerNode

- [Introduction](#introduction)
- [Add Worker Node](#add-worker-node)
- [Remove Worker Node](#remove-worker-node)

## Introduction
In this document, there are steps applied to add and remove worker nodes on Openshift 4.3

References:<br>
https://access.redhat.com/solutions/4799921<br>
https://github.ibm.com/Canturk-Olcer/Openshift-AnsibleTower/blob/master/OCP4.3_Airgap_Installation.md<br>
https://docs.openshift.com/container-platform/4.1/nodes/nodes/nodes-nodes-working.html#nodes-nodes-working-evacuating_nodes-nodes-working<br>


## Add Worker Node

1. Download openshift install and client setup with same version you have deployed. Upload to installation server.
  `https://mirror.openshift.com/pub/openshift-v4/clients/ocp/`

  ```
    Example for V4.3.33
    https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.3.33/openshift-install-linux-4.3.33.tar.gz
    https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.3.33/openshift-client-linux-4.3.33.tar.gz
  ```

2. Extract OpenShift client and copy following to binaries to path:
  ```
    $ sudo cp oc /usr/local/bin/
    $ sudo cp kubectl /usr/local/bin/
  ```

3. Grab your pull string from Redhat and store it on /ocp_depot (e.g. pull-secret.txt). Open the link below and click "Copy Pull String"
  ```
  https://cloud.redhat.com/openshift/install/metal/user-provisioned
  ```

4. Add node to DNS configuration and reload service 'named'
  ```
  $ cat /etc/bind/db.<first 3 octet of ip> 
  145       IN  PTR    <workerHostname>.<OCPDomainPrefix>.<BaseDomain>.

  $ cat /etc/bind/db.<OCPDomainPrefix>.<BaseDomain>
  <workerHostname> IN      A       <worker IP>

  $ systemctl reload named
  ```

  TEST:
  ```
  $ dig @<DNS Server IP> <workerHostname>.<OCPDomainPrefix>.<BaseDomain> +short
  <worker IP>
  $ dig @<DNS Server IP> -x <worker IP> +short
  <workerHostname>.<OCPDomainPrefix>.<BaseDomain>.
  ```

5. Add node to ingress loadbalancer. (skipped since we are building OCS worker nodes. No application will run on them)
  ```
  frontend ingress-http
    bind *:80
    default_backend ingress-http
    mode tcp
    option tcplog

  backend ingress-http
    balance source
    mode tcp
    ....  
    server <workerHostname> <worker IP>:80 check

  frontend ingress-https
    bind *:443
    default_backend ingress-https
    mode tcp
    option tcplog

  backend ingress-https
    balance source
    mode tcp
    ....  
    server <workerHostname> <worker IP>:80 check
  ```

6. Create install-config.yaml

  ```
  apiVersion: v1
  baseDomain: <BaseDomain>

  proxy:
    < Copy proxy information of your OCP deployment shown at oc edit proxy/cluster >

  compute:
  - hyperthreading: Enabled
    name: worker
    replicas: 0

  controlPlane:
    hyperthreading: Enabled
    name: master
    replicas: 3

  metadata:
    name: <OCPDomainPrefix>

  networking:

    clusterNetwork:
    - cidr: 10.128.0.0/14
      hostPrefix: 23
    networkType: OpenShiftSDN
    serviceNetwork:
    - 172.30.0.0/16

  platform:

    none: {}

  pullSecret: <Use pull-secret downloaded at step 3 and put inside single quotes here>

  sshKey: <Use root sshkey of installation server here>
  ```

7. Backup install-config.yaml and create Manifest files
  ```
  $ cp install-config_newworker.yml install-config.yaml
  $ ./openshift-install create manifests
  INFO Consuming Install Config from target directory
  WARNING Making control-plane schedulable by setting MastersSchedulable to true for Scheduler cluster settings
  ```

8. Create ignition files
  ```
  % ./openshift-install create ignition-configs
  INFO Consuming Openshift Manifests from target directory
  INFO Consuming Common Manifests from target directory
  INFO Consuming Master Machines from target directory
  INFO Consuming OpenShift Install (Manifests) from target directory
  INFO Consuming Worker Machines from target directory
  ```

9. Get current certificate to connect existing openshift. this certificate will be valid for 24 hours!<br>
   ** Important Note: Please Remember ignition files are valid for 24 hours. You need to recreate if you want to use them after 24 hours  **
  ```
  % export MCS=api-int.apps.<OCPDomainPrefix>.<BaseDomain>:22623

  % echo "q" | openssl s_client -connect $MCS  -showcerts | awk '/-----BEGIN CERTIFICATE-----/,/-----END CERTIFICATE-----/' | base64 --wrap=0 | tee ./api-int.base64 && sed --regexp-extended --in-place=.backup "s%base64,[^,]+%base64,$(cat ./api-int.base64)\"%" ./worker.ign
    depth=0 CN = api-int.<OCPDomainPrefix>.<BaseDomain>
    verify error:num=20:unable to get local issuer certificate
    verify return:1
    depth=0 CN = api-int.<OCPDomainPrefix>.<BaseDomain>
    verify error:num=21:unable to verify the first certificate
    verify return:1
    DONE
    <CERT>

  % ls -altr
  .......
  -rw-r-----. 1 root   root       1831 Jan 15 12:55 master.ign
  -rw-r-----. 1 root   root       1831 Jan 15 12:55 worker.ign.backup
  .......
  -rw-r-----. 1 root   root       1991 Jan 15 13:05 worker.ign
  ```

10. Start HTTP server for sharing ignition files and RAW disk file.
  ```
  % python -m SimpleHTTPServer  
  ```

11. Prepare virtual server without any operating system. And assign cpu, memory, disk and network cards on them.
    Mount ISO image as a CD drive to your worker node candidates.

12. Boot server and on boot screen press TAB and enter boot variables. Hit enter to start deployment.

  ```
  ip=<worker IPAddress>::<DefaultGatewayAddress>:<SUBNET Mask>:<worker hostname>.<OCPDomain>.<BaseDomain>:<Network Interface>:none nameserver=<DNS Server IP - Installation Server IP> coreos.inst.install_dev=<device name to install COREOS> coreos.inst.image_url=http://<Installation Server IP>:8000/<RAW image name> coreos.inst.ignition_url=http://<Installation Server IP>:8000/worker.ign
  ```

13. Approve CSR when needed
  ```
  % oc get csr -A
  NAME        AGE   REQUESTOR                                                                   CONDITION
  csr-cbxkc   63s   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Pending
  % oc adm certificate approve csr-cbxkc
  certificatesigningrequest.certificates.k8s.io/csr-cbxkc approved
  % oc get csr -A
  NAME        AGE     REQUESTOR                                                                   CONDITION
  csr-cbxkc   2m29s   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Approved,Issued
  ```

14. Check node status till see it Ready.

  ```
  % oc get nodes
  NAME                                         STATUS   ROLES    AGE     VERSION
  ...
  <workerHostname>.<OCPDomainPrefix>.<BaseDomain>   Ready    worker   7m13s   v1.16.2+295f6e6

  ```

15. [Optional] Unschedule the node if you do not want any pods will run on it.
  ```
  oc adm cordon <workerHostname>.<OCPDomainPrefix>.<BaseDomain>  
  ```


## Remove Worker Node  

1. Unschedule the node if you do not want any pods will run on it.
  ```
  oc adm cordon <workerHostname>.<OCPDomainPrefix>.<BaseDomain>  
  ```

2. Drain all pods on the node.

  ```
  % oc adm drain <workerHostname>.<OCPDomainPrefix>.<BaseDomain>   --force=true --ignore-daemonsets
  ```

3. Delete node from cluster.

  ```
  % oc delete node <workerHostname>.<OCPDomainPrefix>.<BaseDomain>
  ```

4. Shut down the node and do not start back. In case accidentally restart you may need to delete CSR's.

  ```
  % oc get csr -A
  NAME        AGE   REQUESTOR                                                                   CONDITION
  csr-hhgpk   20m   system:node:<workerHostname>.<OCPDomainPrefix>.<BaseDomain>                 Pending

  % oc delete csr csr-hhgpk
  certificatesigningrequest.certificates.k8s.io "csr-hhgpk" deleted
  ```
