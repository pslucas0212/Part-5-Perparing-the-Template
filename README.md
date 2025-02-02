# Part 5: Preparing the VMWare VM Template

[Tutorial Menu](https://github.com/pslucas0212/RedHat-Satellite-VM-Provisioning-to-vSphere-Tutorial)    


### Create VM Template on VMWare

We will now create a VM template on VMware which we will use when provisioning RHEL VMs from Satellite.  We will create a RHEL 8.3 template for the exercise which prepares us for a future tutorial where we update our RHEL VM via Satellite. 

First you will need to upload any RHEL ISO files to the VMware environment you will need for the RHEL VM we will be creating.  For RHEL 8.3 I uploaded the rhel-8.3-x86_64-dvd.iso file.

Create and start the RHEL VM.  Choose 1 CPU with 2 GB of RAM and 20GB disk space.  You will use the web console to interact with the VM and configure it.  You will need to set the language, define the disk and set the root password.  Start the installation and reboot the system per the installation instruction.  After the system has rebooted, login as root and start a terminal session for the remainder of the configuration.

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

To support the creation of the VM template, we will install the following the cloud-init, open-vm-tools and perl packages.
```
# yum -y install cloud-init open-vm-tools perl
```

Enable the CA certificates for the image.
```
# update-ca-trust enable 
```

Download the katello-server-ca.crt file from Satellite Server.
```
# wget -O /etc/pki/ca-trust/source/anchors/cloud-init-ca.crt http://sat01.example.com/pub/katello-server-ca.crt
```

To update the record of certificates, enter the following command.
```
# update-ca-trust extract
```

Configure cloud-init to skip networking.
```
# cat << EOF > /etc/cloud/cloud.cfg.d/01_network.cfg
network:
  config: disabled
EOF
```  

We setup the cloud-init to make a call back to Satellite.
```
# cat << EOF > /etc/cloud/cloud.cfg.d/10_foreman.cfg
datasource_list: [NoCloud]
datasource:
  NoCloud:
    seedfrom: http://sat01.example.com/userdata/
EOF
```

Make up a backup of the default cloud-init file.
```
# cp /etc/cloud/cloud.cfg ~/cloud.cfg.`date -I`
```
Replace the default cloud-init file.
```
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
We will create a clean up script.
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
Now we will return to the vCenter console and convert the VM to a template.  Name the template ‘template-rhel8-cloudinit’

### Back on the Satellite side...

First we will define RHEL 8.3 as an operating system choice.  Make sure you are in the Operations Department organization and the moline locations.  On the side menu choose Hosts -> Operating Systems.

![Host -> Operating Systems](/images/sat83.png)

On the Operating Systems page click the blue Create Operating System button. 

![Blue Operating System button](/images/sat84a.png)

On the Operating Systems > Create Operating Systems page fill in or choose options from the following table, and click the blue Submit button.  We will only be filling in information on the Operating System tab and accept the default settings for any other tabs on this page.  We will revisit this page in a few minutes to update the template tab.  After filling in the information, click the blue Submit button.

Name | Choice
---- | ------
Name* | RedHat
Major Version* | 8
Minor Version | 3
Family | Red Hat
Architecture | x86_64

![Define Operating System](/images/sat85a.png)

Next we will create a cloud-init template.  Login into Satellite server's command line and switch to root user.  In the root user's home direct create the following cloud-init template.  Note: The cloud-init template we are creating will also register your RHEL VM to Satellite and Insights.

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
For the user data we can use the UserData open-vm-tools template that is provided with Satellite.  We will need to associate the UserData open-vm-tools template with RHEL 8.3.  Chose Hosts -> Provisioning Templates from the side menu.

![Hosts -> Provisioning Templates](/images/sat64.png)

On the Provisioning Templates page, type 'userdata' in the search field and the click Search button.  On the resulting list, click on the UserData open-vm-tools link.

![Filter on userdata and click UserData open-vm-tools link](/images/sat65.png)

On the Provisioning Templates > Edit UserData open-vm-tools page click the Association tab.  Next click RedHat 8.3 in the Applicable Operating Systems section All Items list and it will move over to the Selected Items list.  Click the blue Submit button.

![Associate UserData open-vm-tools with RedHat 8.3](/images/sat66.png)

Creat an image in Satellite link the vCenter template
```
# hammer compute-resource image create --operatingsystem-id 2 --architecture-id 1 --compute-resource-id 4 --user-data true --uuid template-rhel8-cloudinit --username root --name img-rhel8-prem-server
Image created.
```

Let's update the RHEL 8.3 Operating System to use the correct templates for our deployment.

Make sure you are in the Operations Department organization and the moline locations.  On the side menu choose Hosts -> Operating Systems.

![Host -> Operating Systems](/images/sat83.png)

Now click on the RedHat 8.3 Link.

![RedHat 8.3](/images/sat93.png)

On the Operating Systems > Editing RedHat 8.3 page, click on the Templates tab.  Chose following options from the appropriate dropdown list.  When you are finished click the blue Submit button.

Option Name | Choice
----------- | ------
Cloud-init template | vmware-cloud-int
User data template | UserData open-vm-tools

![Template Optons](/images/sat94a.png)

We are finished here.

I would also recommend checking to see if the VM we created for our template on vSphere has been removed from Satellite.  If not, we will want to delete it.  First make sure you have set organization to Any Organization and location to Any Location.  From the side menu choose Hosts -> All Hosts.

![Hosts -> All Hosts](/images/sat49.png/)

Click on the Edit drop down button in the far right column for the localhost.localdomain host, and choose Delete.  

![Delete button](/images/sat50.png)

In the dialog box that says "Are you sure you want to delete host localhost.localdomain? This action is irreversible.", click the OK button.

## References  
[Installing Satellite Server from a Connected Network](https://access.redhat.com/documentation/en-us/red_hat_satellite/6.9/html/installing_satellite_server_from_a_connected_network/index)   
[Simple Content Access](https://access.redhat.com/articles/simple-content-access)  
[Provisioning VMWare using userdata via Satellite 6.3-6.6](https://access.redhat.com/blogs/1169563/posts/3640721)  
[Understanding Red Hat Content Delivery Network Repositories and their usage with Satellite 6](https://access.redhat.com/articles/1586183)
