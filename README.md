# Part 7: Preparing the VMWare VM Template

[Tutorial Menu](https://github.com/pslucas0212/RedHat-Satellite-VM-Provisioning-to-vSphere-Tutorial)    

In this section we will prepare the VMWare template...

We will start our work on the VMWare side.

### Create VM Template on VMWare

We will now create VM template on VMware.  We will create a RHEL 8.3 teamplate to have some practice later updating the RHEL VM. 

First you will need to upload any RHEL ISO files to the VMware environment you will need for the RHEL VM we will be creating.  For RHEL 8.3 I uploaded the rhel-8.3-x86_64-dvd.iso file.

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

We will now need to temporarily subscribe to Satellite to access template packages. Note: Here for --org parameter we use the Operations Department label.
```
# rpm -ivh http://sat01.example.com/pub/katello-ca-consumer-latest.noarch.rpm
# subscription-manager register --org=operations --activationkey=ak-ops-rhel8-prem-server 
```

To support the creation of the VM teamplate, we willInstall the followeing the cloud-init, open-vm-tools and perl packages.
```
# yum -y install cloud-init open-vm-tools perl
```

Enable the CA certificates for the image:
```
# update-ca-trust enable 
```

Download the katello-server-ca.crt file from Satellite Server:
```
# wget -O /etc/pki/ca-trust/source/anchors/cloud-init-ca.crt http://sat01.example.com/pub/katello-server-ca.crt
```

To update the record of certificates, enter the following command:
```
# update-ca-trust extract
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
Unregistering from: sat01.example.com:443/rhsm
System has been unregistered.
# subscription-manager clean
All local data removed
```
We will creat a clean up script.
```
# cat > ~/clean.sh <<EOF
#!/bin/bash

# stop logging services
/usr/bin/systemctl stop rsyslog
/usr/bin/service stop auditd

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

### Back on the Satellite side...

First we will create a cloud-init template.  Login into Satellite and switch to roor user.  In the root user's home direct create the foloowing cloud-init teamplate.  Note: The cloud-init template we are creating will also register your RHEL VM to Satellite and Insights.

```
# cat > ~/vmware-cloud-init-template.erb <<EOF
#cloud-config
hostname: <%= @host.name %>
fqdn: <%= @host %>
manage_etc_hosts: true
users: {}
runcmd:
- touch ~/cloud-init

- |
<%= indent(2) { snippet 'redhat_register' } -%>
- |
<%= indent(2) { snippet 'insights' } -%>
- |

phone_home:
  url: <%= foreman_url('built') %>
  post: []
tries: 10
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

Now we will create our cloud-init template on Satellite

```
# hammer template create --name vmware-cloud-init --file ~/vmware-cloud-init-template.erb --locations moline --organizations "Operations Department" --operatingsystem-ids 2 --type cloud-init
Provisioning template created.
```
For the user data we can use the UserData open-vm-tools template that is provided with Satellite.  We will need to associate the UserData open-vm-tools template with RHEL 8.3.  Chose Hosts -> Provisioning Templates from the left naviagtion bar.

![Hosts -> Provisioning Templates](/images/sat64.png)

On the Provisioning Templates page, type 'userdata' in the search field and the click Search button.  On the resulting list, click on the UserData open-vm-tools link.

![Filter on userdata and click UserData open-vm-tools link](/images/sat65.png)

On the Provisioning Templates > Edit UserData open-vm-tools page click the Association tab.  Next click RedHat 8.3 in Applicable Operating Systems section All Items list and it will move over to the Selected Items list.  Click the blue Submit button.

![Associate UserData open-vm-tools with RedHat 8.3](/images/sat66.png)

Creat an image in Satellite link the vCenter template
```
# hammer compute-resource image create --operatingsystem-id 2 --architecture-id 1 --compute-resource-id 4 --user-data true --uuid template-rhel8-cloudinit --username root --name img-rhel8-prem-server
Image created.
```

I would also recommend checking to see if the VM we created for our template on vSphere has been removed from Satellite.  If not, we will want to delete it.  First make sure you have set organization to Any Organization and location to Any Location.  From the left navigation bar chose Hosts -> All Hosts.

![Hosts -> All Hosts](/images/sat49.png/)

Click on Edit drop down button in the far right column for the localhost.localdomain host, and chose Delete.  

![Delete button](/images/sat50.png)

In the dialog box that says "Are you sure you want to delete host localhost.localdomain? This action is irreversible.", click the OK button.

## References  
[Installing Satellite Server from a Connected Network](https://access.redhat.com/documentation/en-us/red_hat_satellite/6.9/html/installing_satellite_server_from_a_connected_network/index)   
[Simple Content Access](https://access.redhat.com/articles/simple-content-access)  
[Provisioning VMWare using userdata via Satellite 6.3-6.6](https://access.redhat.com/blogs/1169563/posts/3640721)  
[Understanding Red Hat Content Delivery Network Repositories and their usage with Satellite 6](https://access.redhat.com/articles/1586183). 
[CHAPTER 11. PROVISIONING VIRTUAL MACHINES IN VMWARE VSPHERE](https://access.redhat.com/documentation/en-us/red_hat_satellite/6.9/html/provisioning_guide/provisioning_virtual_machines_in_vmware_vsphere#Provisioning_Virtual_Machines_in_VMware_vSphere-Creating_a_VMware_vSphere_User)  
[What user permissions/roles are required for the VMware vCenter user account to provision from Satellite 6.x?](https://access.redhat.com/solutions/1339483)

