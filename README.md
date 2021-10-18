# Part 5: Perparing the Template

In this section we will prepare the VMWare template...

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

Now we will install our newly create teamplates.

```
# hammer template create --name vmware-cloud-init --file ~/vmware-cloud-init-template.erb --locations moline --organizations "Operations Department" --operatingsystem-ids 1 --type cloud-init

??? Need to cirlce back on this ???

### Create VM Template on VMWare

We will now create VM template on VMware.  We will create a RHEL 8.3 teamplate to have some practice later updating the RHEL VM. 

First you will need to upload any RHEL ISO files to the VMware environment you will need for the RHEL VM we will be creating.  

Create and start the RHEL VM.  Chose 1 CPU with 2 GB of RAM and 20GB disk space.  You will use the web console to interact with the VM and configure it.  You will need to set the language, define the disk and set the root password.  Start the installation and reboot the system per the installation instruction.  After the system has rebooted, login as root and start a terminal session for the remainder of the configuration.



