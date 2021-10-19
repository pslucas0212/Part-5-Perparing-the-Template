# Part 5: Perparing the VMWare VM Template

[Tutorial Menu](https://github.com/pslucas0212/RedHat-Satellite-VM-Provisioning-to-vSphere-Tutorial)    

In this section we will prepare the VMWare template...

We will start our work on the VMWare side.

### Create VM Template on VMWare

We will now create VM template on VMware.  We will create a RHEL 8.3 teamplate to have some practice later updating the RHEL VM. Note: The following instruction steps come from the Template section of [What user permissions/roles are required for the VMware vCenter user account to provision from Satellite 6.x?](https://access.redhat.com/solutions/1339483) Red Hat Knowledge Center.  See this article for more details.

First you will need to upload any RHEL ISO files to the VMware environment you will need for the RHEL VM we will be creating.  

Create and start the RHEL VM.  Chose 1 CPU with 2 GB of RAM and 20GB disk space.  You will use the web console to interact with the VM and configure it.  You will need to set the language, define the disk and set the root password.  Start the installation and reboot the system per the installation instruction.  After the system has rebooted, login as root and start a terminal session for the remainder of the configuration.

Check to see if the the ethernet connection is active and if not active, run the connect command
```
# nmcli device status
# nmcli device connect ens192
# nmcli device show ens192
```
We will temporarily configure the network device as we will need a network connection during the setup of the template
```
[root@localhost ~]# nmcli con modify ens192 connection.id ens192
[root@localhost ~]# nmcli con mod ens192 ipv4.method auto
[root@localhost ~]# nmcli con edit ens192
nmcli> remove ipv4.dns
nmcli> set ipv4.ignore-auto-dns yes
nmcli> set ipv4.dns 10.1.10.254
nmcli> save
nmcli> quit
[root@localhost ~]# nmcli device reapply ens192
[root@localhost ~]# nmcli device show ens192
[root@localhost ~]# hostnamectl set-hostname "localhost.localdomain"
```
We will now need to temporarily subscribe to Satellite to access template packages.  Note: Here for --org parameter we use the Operations Department label.
```
# rpm -ivh http://sat01.example.com/pub/katello-ca-consumer-latest.noarch.rpm
# subscription-manager register --org=operations --activationkey=ak-ops-rhel8-prem-server 
```

Next we will install open-vm-tools
```
yum -y install open-vm-tools.x86_64
```

Install Perl
```
yum -y install perl
```

Let's make a VM snapshot of the image now in case we need any other iterations.  Shutdown the server and make a VM snapshot from the vCenter console.

Next we will install cloud-init
```
# yum -y install cloud-init
```

Configure cloud-init to skip networking
```
# cat << EOF > /etc/cloud/cloud.cfg.d/01_network.cfg
network:
  config: disabled
EOF
```
We setup cloud-init to call backk to Satellite
```
# cat << EOF > /etc/cloud/cloud.cfg.d/10_foreman.cfg
datasource_list: [NoCloud]
datasource:
  NoCloud:
    seedfrom: http://sat01.example.com/userdata/
EOF
```

Make up a backup of the default cloud-init and relace the default cloud-init
```
# cp /etc/cloud/cloud.cfg ~/cloud.cfg.`date -I`
# cat << EOF > /etc/cloud/cloud.cfg
cloud_init_modules:
 - bootcmd

cloud_config_modules:
 - runcmd

cloud_final_modules:
 - scripts-per-once
 - scripts-per-boot
 - scripts-per-instance
 - scripts-user
 - phone-home

system_info:
  distro: rhel
  paths:
    cloud_dir: /var/lib/cloud
    templates_dir: /etc/cloud/templates
  ssh_svcname: sshd

# vim:syntax=yaml
EOF
```
We will now unregister the server from Satellite
```
# subscription-manager unregister
Unregistering from: satellite.example.com:443/rhsm
System has been unregistered.
# subscription-manager clean
All local data removed
```

We will creat a clean up script.
```
cat > ~/clean.sh <<EOF
#!/bin/bash

# stop logging services
/usr/bin/systemctl stop rsyslog
/usr/bin/systemctl stop auditd

# remove old kernels
# /bin/package-cleanup -oldkernels -count=1

#clean yum cache
/usr/bin/yum clean all

#force logrotate to shrink logspace and remove old logs as well as truncate logs
/usr/sbin/logrotate -f /etc/logrotate.conf
/bin/rm -f /var/log/*-???????? /var/log/*.gz
/bin/rm -f /var/log/dmesg.old
/bin/rm -rf /var/log/anaconda
/bin/cat /dev/null > /var/log/audit/audit.log
/bin/cat /dev/null > /var/log/wtmp
/bin/cat /dev/null > /var/log/lastlog
/bin/cat /dev/null > /var/log/grubby

#remove udev hardware rules
/bin/rm -f /etc/udev/rules.d/70*

#remove uuid from ifcfg scripts
/bin/cat > /etc/sysconfig/network-scripts/ifcfg-ens192 <<EOM
DEVICE=ens192
ONBOOT=yes
EOM

#remove SSH host keys
/bin/rm -f /etc/ssh/*key*

#remove root users shell history
/bin/rm -f ~root/.bash_history
unset HISTFILE

#remove root users SSH history
/bin/rm -rf ~root/.ssh/known_hosts
EOF
```

Run the clean script.
```
sh ~/clean.sh
```

Finally we will power off the system to make the VMWare template
```
# systemctl poweroff
```
No we will return the the vCenter console and convert the VM to a template.  Name the template ‘template-rhel8-cloudinit’

### Satellite side...

First we will create a cloud-init template.  Login into Satellite and switch to roor user.  In the root user's home direct create the cloud-init teamplate.  

```
# cat > ~/vmware-cloud-init-template.erb <<EOF
#cloud-config
hostname: <%= @host.name %>
fqdn: <%= @host %>
manage_etc_hosts: true
users: {}
runcmd:
- touch ~/cloud-init

phone_home:
  url: <%= foreman_url('built') %>
  post: []
tries: 10
EOF
```

No we will do the same to create the vmware-userdata-template

```
cat > ~/vmware-userdata-template.erb <<EOF
# Template for VMWare customization via open-vm-tools
identity:
  LinuxPrep:
    domain: <%= @host.domain %>
    hostName: <%= @host.shortname %>
    hwClockUTC: true
    timeZone: <%= @host.params['time-zone'] || 'UTC' %>

globalIPSettings:
  dnsSuffixList: [<%= @host.domain %>]
  <%- @host.interfaces.each do |interface| -%>
  <%- next unless interface.subnet -%>
  dnsServerList: [<%= interface.subnet.dns_primary %>, <%= interface.subnet.dns_secondary %>]
  <%- end -%>

nicSettingMap:
<%- @host.interfaces.each do |interface| -%>
<%- next unless interface.subnet -%>
  - adapter:
      dnsDomain: <%= interface.domain %>
      dnsServerList: [<%= interface.subnet.dns_primary %>, <%= interface.subnet.dns_secondary %>]
      gateway: [<%= interface.subnet.gateway %>]
      ip: <%= interface.ip %>
      subnetMask: <%= interface.subnet.mask %>
<%- end -%>
EOF
```

From Hammer, let's get the ids for the operating system, architecture, and compute resources.
```
$ hammer os list; hammer architecture list; hammer compute-resource list
---|------------|--------------|-------
ID | TITLE      | RELEASE NAME | FAMILY
---|------------|--------------|-------
1  | RedHat 7.9 |              | Redhat
2  | RedHat 8.3 |              | Redhat
---|------------|--------------|-------
---|-------
ID | NAME  
---|-------
1  | x86_64
2  | i386  
---|-------
---|------------|---------
ID | NAME       | PROVIDER
---|------------|---------
4  | cr-vcenter | VMware  
---|------------|---------
```

Now we will install our cloud-init and vmware-userdata-template templates

```
# hammer template create --name vmware-cloud-init --file ~/vmware-cloud-init-template.erb --locations moline --organizations "Operations Department" --operatingsystem-ids 2 --type cloud-init
Provisioning template created.
# hammer template create --name vmware-userdata --file ~/vmware-userdata-template.erb --locations moline --organizations "Operations Department" --operatingsystem-ids 2 --type user_data
Provisioning template created.
```
Creat image in Satellite link the vCenter template
```
# hammer compute-resource image create --operatingsystem-id 2 --architecture-id 1 --compute-resource-id 4 --user-data true --uuid template-rhel8-cloudinit --username root --name img-rhel8-prem-server
Image created.
```



## References  
[Installing Satellite Server from a Connected Network](https://access.redhat.com/documentation/en-us/red_hat_satellite/6.9/html/installing_satellite_server_from_a_connected_network/index)   
[Simple Content Access](https://access.redhat.com/articles/simple-content-access)  
[Provisioning VMWare using userdata via Satellite 6.3-6.6](https://access.redhat.com/blogs/1169563/posts/3640721)  
[Understanding Red Hat Content Delivery Network Repositories and their usage with Satellite 6](https://access.redhat.com/articles/1586183). 
[CHAPTER 11. PROVISIONING VIRTUAL MACHINES IN VMWARE VSPHERE](https://access.redhat.com/documentation/en-us/red_hat_satellite/6.9/html/provisioning_guide/provisioning_virtual_machines_in_vmware_vsphere#Provisioning_Virtual_Machines_in_VMware_vSphere-Creating_a_VMware_vSphere_User)  
[What user permissions/roles are required for the VMware vCenter user account to provision from Satellite 6.x?](https://access.redhat.com/solutions/1339483)

