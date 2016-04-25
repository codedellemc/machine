# Docker Machine Driver for RackHD Vagrant Example

This repo contains all the necessary instructions, scripts, and documentation to use the [Docker Machine Driver for RackHD ](https://github.com/emccode/docker-machine-rackhd) with the [Vagrant Demo environment of RackHD](https://github.com/RackHD/RackHD/tree/master/example). 

[![Docker Machine Driver for RackHD Vagrant Setup and Testing](http://i.imgur.com/346xWSZ.png)](https://youtu.be/7Y4-F4aCvuo "Docker Machine Driver for RackHD Vagrant Setup and Testing")

## Requirements

* VirtualBox 5.0.10+
* >4GB of Available RAM
* Internet connection

## Pre-requisites

1. Prepare [Vagrant setup of RackHD](https://github.com/RackHD/RackHD/tree/master/example) and set the head to commit `d3e545a1e`.
   ```
   $ git clone https://github.com/RackHD/RackHD
   $ cd RackHD
   $ cd example
   ```

2. (Optional) If you are going to be memory constrained on your laptop, edit the `Vagrantfile` and set the Memory to 2048 with `v.memory = 2048`. 

3. Set the PXE Count to 3.
  * `$ cp config/monorail_rack.cfg.example config/monorail_rack.cfg`
  * `$ vi config/monorail_rack.cfg` (set PXE to 3)
  * or copy [monorail_rack.cfg](https://github.com/emccode/machine/blob/master/rackhd/monorail_rack.cfg)

4. Modify the `bin/monorail_rack` shell script to give the PXE machines a secondary NIC set to NAT and 1024MB of RAM. The 2nd NIC is required to be on NAT so it has internet access to be able to install Docker. The first NIC remains on the closed/private network for RackHD internal communication.
  * Set the aforementioned manually after deployment or modify the script
  * Change the `DEPLOY PXE CLIENTS` section to match the following or copy [monorail_rack.sh](https://github.com/emccode/machine/blob/master/rackhd/monorail_rack.sh)
   ```bash
######################
# DEPLOY PXE CLIENTS #
######################

if [ $PXE_COUNT ]
  then
    for (( i=1; i <= $PXE_COUNT; i++ ))
      do
        vmName="pxe-$i"
        if [[ ! -e $vmName.vdi ]]; then # check to see if PXE vm already exists
            echo "deploying pxe: $i"
            VBoxManage createvm --name $vmName --register;
            VBoxManage createhd --filename $vmName --size 8192;
            VBoxManage storagectl $vmName --name "SATA Controller" --add sata --controller IntelAHCI
            VBoxManage storageattach $vmName --storagectl "SATA Controller" --port 0 --device 0 --type hdd --medium $vmName.vdi
            VBoxManage modifyvm $vmName --ostype Ubuntu --boot1 net --memory 1024 --cpus 2;
            VBoxManage modifyvm $vmName --nic1 intnet --intnet1 closednet --nicpromisc1 allow-all;
            VBoxManage modifyvm $vmName --nictype1 82540EM --macaddress1 auto;
            VBoxManage modifyvm $vmName --nic2 NAT;
            VBoxManage modifyvm $vmName --nictype2 82540EM --macaddress2 auto;
            VBoxManage modifyvm $vmName --natpf2 "guestssh,tcp,,2$i2$i,,22";
            VBoxManage modifyvm $vmName --ioapic off;
            VBoxManage modifyvm $vmName --rtcuseutc on;
        fi
      done
fi
   ```

5. Kick off the script. `bin/monorail_rack`. This should kick off the monorail services and have it up and running.

6. Open another terminal tab/window and SSH into the monorail server `vagrant ssh`

7. Unpack and Install a new CentOS ISO
   ```
$ cd /tmp
$ wget http://mirrors.mit.edu/centos/7/isos/x86_64/CentOS-7-x86_64-DVD-1511.iso
$ sudo python ~/src/on-http/data/templates/setup_iso.py /tmp/CentOS-7-x86_64*.iso /var/mirrors --link=/home/vagrant/src
   ```

8. Edit the CentOS KickStart script `/home/vagrant/src/on-http/data/templates/centos-ks`. Copy and paste [centos-ks](https://github.com/emccode/machine/blob/master/rackhd/centos-ks) or make the following modifications:
    * comment out `graphical` and uncomment `text` to make the installed completely text based
    * add `--port=2376:tcp --port=3376:tcp` to the firewall configuration which allows incoming communication to Docker and Swarmturn off SELinux with `selinux --disabled`
    * Activate the 2nd network adapter `network --device=enp0s8 --noipv6 --activate`
    * Add new users to the Sudoers file by adding `echo "<%=user.name%> ALL=(ALL)      NOPASSWD:ALL" >> /etc/sudoers` under the For Each User loop in the Post installation process. ([line 89](https://github.com/emccode/machine/blob/master/rackhd/centos-ks#L89))

9. Open another terminal tab/window and create a new file called `centos_workflow.json`. Copy and paste the below or copy from [centos_workflow.json](https://github.com/emccode/machine/blob/master/rackhd/centos_workflow.json). Take note of the user and password fields. Change as necessary.
   ```
{
  "friendlyName": "VirtualBox Default Install CentOS",
  "injectableName": "Graph.DefaultVirtualBox.InstallCentOS",
  "options": {
    "defaults": {
      "version": "7",
      "dnsServers": ["8.8.8.8", "8.8.4.4"],
      "repo": "{{api.server}}/Centos/7.0",
      "rootPassword": "root",
      "users": [{
        "name": "rackhd",
        "password": "rackhd123",
        "uid": 1010
      }]
      },
    "install-centos": {
      "schedulerOverrides": {
        "timeout": 3600000
      }
    }
  },
  "tasks": [
    {
      "label": "create-noop-obm-settings",
      "taskDefinition": {
        "friendlyName": "Create VirtualBox No-Op OBM settings",
        "injectableName": "Task.Obm.Vbox.Noop.CreateSettings",
        "implementsTask": "Task.Base.Obm.CreateSettings",
        "options": {
        "service": "noop-obm-service",
        "config": {}
        },
        "properties": {
        "obm": {
            "type": "virtualbox"
        }
      }}
    },
    {
      "label": "install-centos",
      "taskName": "Task.Os.Install.CentOS",
      "waitOn": {
        "create-noop-obm-settings": "succeeded"
      }
    }
  ]
}
   ```

10. Create a new file called `virtualbox_sku_centos.json`. Copy and paste the below or copy from [virtualbox_sku_centos.json](https://github.com/emccode/machine/blob/master/rackhd/virtualbox_sku_centos.json)
   ```
{
    "name": "Noop OBM settings for VirtualBox nodes",
    "discoveryGraphName": "Graph.DefaultVirtualBox.InstallCentOS",
    "discoveryGraphOptions": {},
    "rules": [
        {
            "path": "dmi.System Information.Product Name",
            "equals": "VirtualBox"
        }
    ]
}
   ```

11. Upload the new `.json` files to the RackHD server.
   ```
$ curl -H "Content-Type: application/json" -X PUT --data @centos_workflow.json http://localhost:9090/api/1.1/workflows
$ curl -H "Content-Type: application/json" -X POST --data @virtualbox_sku_centos.json http://localhost:9090/api/1.1/skus
   ```

12. At this point, CentOS will automatically begin installation after it has PXE booted and added to RackHD's inventory. Open up VirtualBox and power on one of the machines or power on using the command line with `VBoxManage controlvm poweron pxe-1`. Wait and revel in the majesty that is automation.

13. Install Docker Client, Docker Machine and the RackHD Driver. Go back to the 2nd terminal that is SSH'd into the vagrant RackHD/monorail. The following pieces require root permission. In this version, all Docker Machine related commands require execution to be done on the RackHD server where [on-http](https://github.com/RackHD/on-http/) is being hosted because RackHD is only able to provide IP addresses for nodes that acquired addresses from RackHD's DHCP server. There is an [in-band management](https://github.com/RackHD/specs/blob/master/inband_management.md) spec for adding the capability to manage hosts via SSH outside of RackHD (or via the 2nd NIC) in a future release.
   ```
$ sudo su
# curl -L https://get.docker.com/builds/Linux/x86_64/docker-1.11.0.tgz > docker-1.11.0.tgz && tar -xf docker-1.11.0.tgz && cp docker/docker /usr/local/bin/docker && chmod +x /usr/local/bin/docker
# curl -L https://github.com/docker/machine/releases/download/v0.7.0/docker-machine-`uname -s`-`uname -m` >/usr/local/bin/docker-machine && chmod +x /usr/local/bin/docker-machine
# curl -L https://github.com/emccode/docker-machine-rackhd/releases/download/v0.1.0/docker-machine-driver-rackhd.`uname -s`-`uname -m` > /usr/local/bin/docker-machine-driver-rackhd && chmod +x /usr/local/bin/docker-machine-driver-rackhd
   ```

14. Retrieve the node ID from the PXE server(s) that was powered on in step 11. Open up the web browser and go to `http://localhost:9090/ui` to view the Node ID or go to the 3rd terminal and issue the following:
   ```
$ curl http://localhost:9090/api/1.1/nodes | python -m json.tool
   ```
Look for a "compute" node and retrieve the `ID` in a format similar to `56e2436a07366d7a09efc5c8`. 

15. Use Docker Machine to now manage this node and install Docker. Go to the 2nd terminal window that is SSH'd into RackHD Vagrant image
View all the RackHD parameters using the `--help` flag
   ```
# docker-machine create --driver rackhd --help
   ```
Create the new host by specifying the `--rackhd-node-id` that was gathered from querying the Nodes and specify `--rackhd-ssh-user` & `--rackhd-ssh-password` that was set in [centos_workflow.json](https://github.com/emccode/machine/blob/master/rackhd/centos_workflow.json). Lastly, provide a name for the host. 
   ```
# docker-machine create --driver=rackhd --rackhd-node-id 56e2436a07366d7a09efc5c8 --rackhd-ssh-user rackhd --rackhd-ssh-password rackhd123 lodidodi
   ```

---

RackHD is experiencing heavy development and the APIs are changing by the hour. This current version relies on the [1.1 API](https://github.com/RackHD/on-http/blob/master/static/monorail.yml) but the [2.0 API](https://github.com/RackHD/on-http/blob/master/static/monorail-2.0.yaml) is around the corner. The underlying components are the API bindings for [gorackhd](https://github.com/emccode/gorackhd) and [gorackhd-redfish](https://github.com/emccode/gorackhd-redfish)
