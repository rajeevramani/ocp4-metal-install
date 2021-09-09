# Credits

These instructions are based on the original `https://github.com/ryanhay/ocp4-metal-install`.

I have made a few modifications based on the following peices of documentation avaiable

[Redhat on AOS videao](https://www.youtube.com/watch?v=7616SPiOXi8)

[Nutnaix on AOS github](https://github.com/nutanix/openshift/tree/main/docs/install/manual)

[Nutanix CSI setup](https://github.com/nutanix/openshift/tree/main/operators/csi)

The following documentation was to set up everything from scracth and for me to do the install the hard way. A big thanks to @ryanhay for his original repo.

# OpenShift 4 Bare Metal Install - User Provisioned Infrastructure (UPI) on Nutanix AOS

- [OpenShift 4 Bare Metal Install - User Provisioned Infrastructure (UPI)](#openshift-4-bare-metal-install---user-provisioned-infrastructure-upi)
  - [Architecture Diagram](#architecture-diagram)
  - [Download Software](#download-software)
  - [Prepare the 'Bare Metal' environment](#prepare-the-bare-metal-environment)
  - [Configure Environmental Services](#configure-environmental-services)
  - [Generate and host install files](#generate-and-host-install-files)
  - [Deploy OpenShift](#deploy-openshift)
  - [Monitor the Bootstrap Process](#monitor-the-bootstrap-process)
  - [Remove the Bootstrap Node](#remove-the-bootstrap-node)
  - [Wait for installation to complete](#wait-for-installation-to-complete)
  - [Join Worker Nodes](#join-worker-nodes)
  - [Configure storage for the Image Registry](#configure-storage-for-the-image-registry)
  - [Create the first Admin user](#create-the-first-admin-user)
  - [Access the OpenShift Console](#access-the-openshift-console)
  - [Troubleshooting](#troubleshooting)

## Architecture Diagram

![Architecture Diagram](./diagram/Architecture.png)

## Download Software

1. Download [CentOS 8 x86_64 image](https://www.centos.org/centos-linux/)
1. Login to [RedHat OpenShift Cluster Manager](https://cloud.redhat.com/openshift)
1. Select 'Create Cluster' from the 'Clusters' navigation menu
1. Select 'RedHat OpenShift Container Platform'
1. Select 'Run on Bare Metal'
1. Download the following files:

   - Openshift Installer for Linux
   - Pull secret
   - Command Line Interface for Linux and your workstations OS
   - Red Hat Enterprise Linux CoreOS (RHCOS)
     - rhcos-X.X.X-x86_64-installer.x86_64.iso

## Prepare the 'Nutanix' environment

> Nutanix AHV used in this guide

1. Upload the CentOS 8 iso to an Nutanix image store
1. Create a new Port Group called 'OCP' under Networking
1. Create 3 Control Plane virtual machines with minimum settings:
   - Name: ocp-cp-# (Example ocp-cp-1)
   - 4vcpu
   - 8GB RAM
   - 50GB HDD
   - NIC connected to the OCP network
   - Load the rhcos-X.X.X-x86_64-installer.x86_64.iso image into the CD/DVD drive
1. Create 2 Worker virtual machines (or more if you want) with minimum settings:
   - Name: ocp-w-# (Example ocp-w-1)
   - 4vcpu
   - 8GB RAM
   - 50GB HDD
   - NIC connected to the OCP network
   - Load the rhcos-X.X.X-x86_64-installer.x86_64.iso image into the CD/DVD drive
1. Create a Bootstrap virtual machine (this vm will be deleted once installation completes) with minimum settings:
   - Name: ocp-boostrap
   - 4vcpu
   - 8GB RAM
   - 50GB HDD
   - NIC connected to the OCP network
   - Load the rhcos-X.X.X-x86_64-installer.x86_64.iso image into the CD/DVD drive
1. Create a Services virtual machine with minimum settings:
   - Name: ocp-svc
   - 4vcpu
   - 4GB RAM
   - 120GB HDD
   - NIC1 connected to the VM Network (LAN)
   - NIC2 connected to the OCP network
   - Load the CentOS_8.iso image into the CD/DVD drive
1. Boot all virtual machines so they each are assigned a MAC address
1. Shut down all virtual machines except for 'ocp-svc'
1. Use the Prism dashboard to record the MAC address of each vm, these will be used later to set static IPs

## Configure Environmental Services

1. Install CentOS8 on the ocp-svc host

   - Remove the home dir partition and assign all free storage to '/'
   - Optionally you can install the 'Guest Tools' package to have monitoring and reporting in the VMware ESXi dashboard
   - Enable the LAN NIC only to obtain a DHCP address from the LAN network and make note of the IP address (ocp-svc_IP_address) assigned to the vm

1. Boot the ocp-svc VM

1. Move the files downloaded from the RedHat Cluster Manager site to the ocp-svc node

   ```bash
   scp ~/Downloads/openshift-install-linux.tar.gz ~/Downloads/openshift-client-linux.tar.gz root@{ocp-svc_IP_address}:/root/
   ```

1. SSH to the ocp-svc vm

   ```bash
   ssh root@{ocp-svc_IP_address}
   ```

1. Extract Client tools and copy them to `/usr/local/bin`

   ```bash
   tar xvf openshift-client-linux.tar.gz
   mv oc kubectl /usr/local/bin
   ```

1. Confirm Client Tools are working

   ```bash
   kubectl version
   oc version
   ```

1. Extract the OpenShift Installer

   ```bash
   tar xvf openshift-install-linux.tar.gz
   ```

1. Update CentOS so we get the latest packages for each of the services we are about to install

   ```bash
   dnf update
   ```

1. Install Git

   ```bash
   dnf install git -y
   ```

1. Download [config files](https://github.com/rajeevramani/ocp4-metal-install) for each of the services

   ```bash
   git clone https://github.com/rajeevramani/ocp4-metal-install.git
   ```

1. OPTIONAL: Create a file '~/.vimrc' and paste the following (this helps with editing in vim, particularly yaml files):

   ```bash
   cat <<EOT >> ~/.vimrc
   syntax on
   set nu et ai sts=0 ts=2 sw=2 list hls
   EOT
   ```

   Update the preferred editor

   ```bash
   export OC_EDITOR="vim"
   export KUBE_EDITOR="vim"
   ```

1. Set a Static IP for OCP network interface `nmtui-edit ens4` or edit `/etc/sysconfig/network-scripts/ifcfg-ens4`

   - **Address**: 192.168.22.1
   - **DNS Server**: 127.0.0.1
   - **Search domain**: ocp.lan
   - Never use this network for default route
   - Automatically connect

   > If changes arent applied automatically you can bounce the NIC with `nmcli connection down ens224` and `nmcli connection up ens224`

1. Setup firewalld

   Create **internal** and **external** zones

   ```bash
   nmcli connection modify ens4 connection.zone internal
   nmcli connection modify ens3 connection.zone external
   ```

   View zones:

   ```bash
   firewall-cmd --get-active-zones
   ```

   Set masquerading (source-nat) on the both zones.

   So to give a quick example of source-nat - for packets leaving the external interface, which in this case is ens192 - after they have been routed they will have their source address altered to the interface address of ens192 so that return packets can find their way back to this interface where the reverse will happen.

   ```bash
   firewall-cmd --zone=external --add-masquerade --permanent
   firewall-cmd --zone=internal --add-masquerade --permanent
   ```

   Reload firewall config

   ```bash
   firewall-cmd --reload
   ```

   Check the current settings of each zone

   ```bash
   firewall-cmd --list-all --zone=internal
   firewall-cmd --list-all --zone=external
   ```

   When masquerading is enabled so is ip forwarding which basically makes this host a router. Check:

   ```bash
   cat /proc/sys/net/ipv4/ip_forward
   ```

1. Install and configure BIND DNS

   Install

   ```bash
   dnf install bind bind-utils -y
   ```

   Apply configuration

   ```bash
   cp ~/ocp4-metal-install/dns/named.conf /etc/named.conf
   cp -R ~/ocp4-metal-install/dns/zones /etc/named/
   ```

   Configure the firewall for DNS

   ```bash
   firewall-cmd --add-port=53/udp --zone=internal --permanent
   firewall-cmd --reload
   ```

   Enable and start the service

   ```bash
   systemctl enable named
   systemctl start named
   systemctl status named
   ```

   > At the moment DNS will still be pointing to the LAN DNS server. You can see this by testing with `dig ocp.lan`.

   Change the LAN nic (ens192) to use 127.0.0.1 for DNS AND ensure `Ignore automatically Obtained DNS parameters` is ticked

   ```bash
   nmtui-edit ens192
   ```

   Restart Network Manager

   ```bash
   systemctl restart NetworkManager
   ```

   Confirm dig now sees the correct DNS results by using the DNS Server running locally

   ```bash
   dig ocp.lan
   # The following should return the answer ocp-bootstrap.lab.ocp.lan from the local server
   dig -x 192.168.22.200
   ```

1. Install & configure DHCP

   Install the DHCP Server

   ```bash
   dnf install dhcp-server -y
   ```

   Edit dhcpd.conf from the cloned git repo to have the correct mac address for each host and copy the conf file to the correct location for the DHCP service to use

   ```bash
   cp ~/ocp4-metal-install/dhcpd.conf /etc/dhcp/dhcpd.conf
   ```

   Configure the Firewall

   ```bash
   firewall-cmd --add-service=dhcp --zone=internal --permanent
   firewall-cmd --reload
   ```

   Enable and start the service

   ```bash
   systemctl enable dhcpd
   systemctl start dhcpd
   systemctl status dhcpd
   ```

1. Install & configure Apache Web Server

   Install Apache

   ```bash
   dnf install httpd -y
   ```

   Change default listen port to 8080 in httpd.conf

   ```bash
   sed -i 's/Listen 80/Listen 0.0.0.0:8080/' /etc/httpd/conf/httpd.conf
   ```

   Configure the firewall for Web Server traffic

   ```bash
   firewall-cmd --add-port=8080/tcp --zone=internal --permanent
   firewall-cmd --reload
   ```

   Enable and start the service

   ```bash
   systemctl enable httpd
   systemctl start httpd
   systemctl status httpd
   ```

   Making a GET request to localhost on port 8080 should now return the default Apache webpage

   ```bash
   curl localhost:8080
   ```

1. Install & configure HAProxy

   Install HAProxy

   ```bash
   dnf install haproxy -y
   ```

   Copy HAProxy config

   ```bash
   \cp ~/ocp4-metal-install/haproxy.cfg /etc/haproxy/haproxy.cfg
   ```

   Configure the Firewall

   > Note: Opening port 9000 in the external zone allows access to HAProxy stats that are useful for monitoring and troubleshooting. The UI can be accessed at: `http://{ocp-svc_IP_address}:9000/stats`

   ```bash
   firewall-cmd --add-port=6443/tcp --zone=internal --permanent # kube-api-server on control plane nodes
   firewall-cmd --add-port=6443/tcp --zone=external --permanent # kube-api-server on control plane nodes
   firewall-cmd --add-port=22623/tcp --zone=internal --permanent # machine-config server
   firewall-cmd --add-service=http --zone=internal --permanent # web services hosted on worker nodes
   firewall-cmd --add-service=http --zone=external --permanent # web services hosted on worker nodes
   firewall-cmd --add-service=https --zone=internal --permanent # web services hosted on worker nodes
   firewall-cmd --add-service=https --zone=external --permanent # web services hosted on worker nodes
   firewall-cmd --add-port=9000/tcp --zone=external --permanent # HAProxy Stats
   firewall-cmd --reload
   ```

   Enable and start the service

   ```bash
   setsebool -P haproxy_connect_any 1 # SELinux name_bind access
   systemctl enable haproxy
   systemctl start haproxy
   systemctl status haproxy
   ```

## Generate and host install files

1. Generate an SSH key pair keeping all default options

   ```bash
   ssh-keygen
   ```

1. Create an install directory

   ```bash
   mkdir ~/ocp-install
   ```

1. Copy the install-config.yaml included in the clones repository to the install directory

   ```bash
   cp ~/ocp4-metal-install/install-config.yaml ~/ocp-install
   ```

1. Update the install-config.yaml with your own pull-secret and ssh key.

   - Line 23 should contain the contents of your pull-secret.txt
   - Line 24 should contain the contents of your '~/.ssh/id_rsa.pub'

   ```bash
   vim ~/ocp-install/install-config.yaml
   ```

1. Generate Kubernetes manifest files

   ```bash
   ~/openshift-install create manifests --dir ~/ocp-install
   ```

   A warning is shown about making the control plane nodes schedulable. It is up to you if you want to run workloads on the Control Plane nodes. If you dont want to you can disable this with:

   **I found that I needed to do this for UPI**
   `sed -i 's/mastersSchedulable: true/mastersSchedulable: false/' ~/ocp-install/manifests/cluster-scheduler-02-config.yml`.
   
   Make any other custom changes you like to the core Kubernetes manifest files.

   Generate the Ignition config and Kubernetes auth files

   ```bash
   ~/openshift-install create ignition-configs --dir ~/ocp-install/
   ```

1. Create a hosting directory to serve the configuration files for the OpenShift booting process

   ```bash
   mkdir /var/www/html/ocp4
   ```

1. Copy all generated install files to the new web server directory

   ```bash
   cp -R ~/ocp-install/* /var/www/html/ocp4
   ```

1. Change ownership and permissions of the web server directory

   ```bash
   chcon -R -t httpd_sys_content_t /var/www/html/ocp4/
   chown -R apache: /var/www/html/ocp4/
   chmod 755 /var/www/html/ocp4/
   ```

1. Confirm you can see all files added to the `/var/www/html/ocp4/` dir through Apache

   ```bash
   curl localhost:8080/ocp4/
   ```

## Deploy OpenShift

1. Power on the ocp-bootstrap host and ocp-cp-\# hosts (Do them one by one not together). Once the machine boots enter the following configuration:

   ```bash
   # Bootstrap Node - ocp-bootstrap
   sudo coreos-installer install /dev/sda --ignition-url http://192.168.22.1:8080/ocp4/bootstrap.ign --insecure ignition 
   ```
   After installation `shutdown -h now` 
   detach `CD-DOM`


   ```bash
   # Bootstrap Node - ocp-bootstrap
   sudo coreos-installer install /dev/sda --ignition-url http://192.168.22.1:8080/ocp4/master.ign --insecure ignition 
   ```
   After installation `shutdown -h now` 
   detach `CD-DOM`

1. Power on the ocp-w-\# hosts and select 'Tab' to enter boot configuration. Enter the following configuration:

   ```bash
   # Bootstrap Node - ocp-bootstrap
   sudo coreos-installer install /dev/sda --ignition-url http://192.168.22.1:8080/ocp4/worker.ign --insecure ignition 
   ```
   After installation `shutdown -h now` 
   detach `CD-DOM`

## Monitor the Bootstrap Process

1. You can monitor the bootstrap process from the ocp-svc host at different log levels (debug, error, info)

   ```bash
   ~/openshift-install --dir ~/ocp-install wait-for bootstrap-complete --log-level=debug
   ```

   Monitor Bootstrap server
   
   ```
   ssh core@192.168.22.200 'for pod in $(sudo podman ps -a -q); do sudo podman logs $pod; done'
   ```

1. Once bootstrapping is complete the ocp-boostrap node [can be removed](#remove-the-bootstrap-node)

## Remove the Bootstrap Node

1. Remove all references to the `ocp-bootstrap` host from the `/etc/haproxy/haproxy.cfg` file

   ```bash
   # Two entries
   vim /etc/haproxy/haproxy.cfg
   # Restart HAProxy - If you are still watching HAProxy stats console you will see that the ocp-boostrap host has been removed from the backends.
   systemctl reload haproxy
   ```

1. The ocp-bootstrap host can now be safely shutdown and deleted from the VMware ESXi Console, the host is no longer required

## Wait for installation to complete

> IMPORTANT: if you set mastersSchedulable to false the [worker nodes will need to be joined to the cluster](#join-worker-nodes) to complete the installation. This is because the OpenShift Router will need to be scheduled on the worker nodes and it is a dependency for cluster operators such as ingress, console and authentication.

1. Collect the OpenShift Console address and kubeadmin credentials from the output of the install-complete event

   ```bash
   ~/openshift-install --dir ~/ocp-install wait-for install-complete
   ```

   Monitoring install
   ```bash
   tail -f ~/ocp-install/.openshift_install.log 
   ```

1. Continue to join the worker nodes to the cluster in a new tab whilst waiting for the above command to complete

## Join Worker Nodes

1. Setup 'oc' and 'kubectl' clients on the ocp-svc machine

   ```bash
   export KUBECONFIG=~/ocp-install/auth/kubeconfig
   # Test auth by viewing cluster nodes
   oc get nodes
   ```

1. View and approve pending CSRs

   > Note: Once you approve the first set of CSRs additional 'kubelet-serving' CSRs will be created. These must be approved too.
   > If you do not see pending requests wait until you do.

   ```bash
   # View CSRs
   oc get csr
   # Approve all pending CSRs
   oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs oc adm certificate approve
   # Wait for kubelet-serving CSRs and approve them too with the same command
   oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs oc adm certificate approve
   ```

1. Watch and wait for the Worker Nodes to join the cluster and enter a 'Ready' status

   > This can take 5-10 minutes

   ```bash
   watch -n5 oc get nodes
   ```

## Configure storage for the Image Registry

> A Bare Metal cluster does not by default provide storage so the Image Registry Operator bootstraps itself as 'Removed' so the installer can complete. As the installation has now completed storage can be added for the Registry and the operator updated to a 'Managed' state.

### Install nutanix CSI operator

   Documentation [steps](https://github.com/nutanix/openshift/tree/main/operators/csi)

   For creating the secret
   `echo -n '<PRISMELEMENTIP>:9440:admin:<passwprd> | base64`

   #### Storage secret
   `oc -n ntnx-system apply -f ./csi-oc-registry-setup/storage-secret.yaml`

   #### Storage Class

   edit the `storage-class.yaml` and modify the following entries

   ```yaml
   dataServiceEndPoint: <PRISMELEMENT FLOATING IP>:3260 
   #maksure storageContainer is changed
   storageContainer: <default-container-NAME>
   ```

   `oc -n ntnx-system apply -f ./csi-oc-registry-setup/storage-class.yaml`

   #### Additional steps are required for CSI setup with AHV

   Follow the steps documented [here](https://portal.nutanix.com/page/documents/kbs/details?targetId=kA07V000000LWXJSA4)

   `oc -n ntnx-system apply -f ./csi-oc-registry-setup/99-master-custom-enable-iscsid.yaml`

   `oc -n ntnx-system apply -f ./csi-oc-registry-setup/99-worker-custom-enable-iscsid.yaml`

   **Run following comming on every node**

   `sudo systemctl enable --now iscsid`
   

   #### Create PVC

   `oc -n openshift-image-registry apply -f ./csi-oc-registry-setup/pvc-claim.yaml`

   Configure OpenShift registry storage similarly to OpenShift documentation):

   ```bash
   oc edit configs.imageregistry.operator.openshift.io
   ```

   Change entries to

   ```yaml
   managementState: Managed
   ```

   ```yaml
   rolloutStrategy: Recreate
   ```

   ```yaml
   storage:
     pvc:
       claim: image-registry-claim
   ```

   Confirm the 'image-registry-storage' pvc has been created and is currently in a 'Pending' state

   ```bash
   oc get pvc -n openshift-image-registry
   ```

   You can also watch the official [video](https://www.youtube.com/watch?v=7616SPiOXi8) or read the documentation [here](https://github.com/nutanix/openshift/tree/main/docs/install/manual)


## Access the OpenShift Console

1. Wait for the 'console' Cluster Operator to become available

   ```bash
   oc get co
   ```

1. Append the following to your local workstations `/etc/hosts` file:

   > From your local workstation
   > If you do not want to add an entry for each new service made available on OpenShift you can configure the ocp-svc DNS server to serve externally and create a wildcard entry for \*.apps.lab.ocp.lan

   ```bash
   # Open the hosts file
   sudo vi /etc/hosts

   # Append the following entries:
   <HELPER HOST IP> ocp-svc api.lab.ocp.lan console-openshift-console.apps.lab.ocp.lan oauth-openshift.apps.lab.ocp.lan downloads-openshift-console.apps.lab.ocp.lan alertmanager-main-openshift-monitoring.apps.lab.ocp.lan grafana-openshift-monitoring.apps.lab.ocp.lan prometheus-k8s-openshift-monitoring.apps.lab.ocp.lan thanos-querier-openshift-monitoring.apps.lab.ocp.lan
   ```

1. Navigate to the [OpenShift Console URL](https://console-openshift-console.apps.lab.ocp.lan) and log in as the 'admin' user

   > You will get self signed certificate warnings that you can ignore
   
## Troubleshooting

1. You can collect logs from all cluster hosts by running the following command from the 'ocp-svc' host:

   ```bash
   ./openshift-install gather bootstrap --dir ocp-install --bootstrap=192.168.22.200 --master=192.168.22.201 --master=192.168.22.202 --master=192.168.22.203
   ```

2. Modify the role of the Control Plane Nodes

   If you would like to schedule workloads on the Control Plane nodes apply the 'worker' role by changing the value of 'mastersSchedulable' to true.

   If you do not want to schedule workloads on the Control Plane nodes remove the 'worker' role by changing the value of 'mastersSchedulable' to false.

   > Remember depending on where you host your workloads you will have to update HAProxy to include or exclude the control plane nodes from the ingress backends.

   ```bash
   oc edit schedulers.config.openshift.io cluster
   ```
3. CSI Error
   If you see the following error
   ```
   MountVolume.SetUp failed for volume "pvc-XXX-YYY" : rpc error: code = Internal desc = iscsi/lvm failure, last err seen: iscsi: failed to sendtargets to portal 192.168.26.14:3260 output: Failed to connect to bus: No data available iscsiadm: can not connect to iSCSI daemon (111)! iscsiadm: Cannot perform discovery. Initiatorname required. iscsiadm: Could not perform SendTargets discovery: could not connect to iscsid , err exit status 20
   ```
   
   Follow the steps here `https://portal.nutanix.com/page/documents/kbs/details?targetId=kA07V000000LWXJSA4`
