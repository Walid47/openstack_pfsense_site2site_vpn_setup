# Site-to-site VPN setup with Pfsense as an OpenStack instance:
We have a virtual OpenStack network and another remote host behind a NAT device, that we want to make available to a partner private network.  
The following sections present the basic steps for realising this setup, which includes an IPSec site-to-site VPN tunnel between two Pfsense-2.4.4 instance, the first will be running on OpenStack, the second on VirtualBox-5.2.18. The setup also includes an OpenVPN TLS tunnel to connect a remote remote host to our OpenStack pfsense VM.  
Both instances of pfsense will be behind a form of NAT: The OpenStack instance will have a floating IP (1:1 NAT), as for the Virtualbox instance, we'll use port forwarding for the UDP ports required for a NAT-T IPSec tunnel.  

# Setup:
As depicted in the figure bellow, we'll use two pfsense virtual machines that will act as our VPN Boxes. They will handle tunneling and routing of traffic between remote private networks, and the OpenStack instance will also connect the remote host through an OpenVPN TLS tunnel.   
![Network diagram for a testbed site-to-site IPSec VPN][100]
## VirtualBox Setup
Before creating our virutal machines, we need to create our NAT network which will act as pfsense's WAN and LAN networks.  
### Virtual Neworks
As depicted in the network diagram above, we'll need at least two network interfaces, the first for WAN, the second for LAN.  

**N.B.** The port forwarding configuration can be added once the LAN/WAN interfaces are set and configured from pfsense.  

Create two NAT Networks from ```File```->```Preferences```->```Network```:  

![VirtualBox 'Network Preferences' view][101]

- **WAN** Network:  
    
    ![VirtualBox 'Add Network' view][102]

	- Network Name: **WAN_net**
	- Network CIDR: **10.0.2.0/24**
	- Network Options: **Supports DHCP**
	- Port Forwarding:  
	    
	    ![VirtualBox 'Port Forwarding' view][103]
		- ***PFSense IPSec NAT-T***:
			- Protocol: **UDP**
			- Host IP: **0.0.0.0**
			- Host Port: **4500**
			- Guest IP: **10.0.2.4**
			- Guest Port: **4500**
		- ***PFSense ISAKMP***:
			- Protocol: **UDP**
			- Host IP: **0.0.0.0**
			- Host Port: **500**
			- Guest IP: **10.0.2.4**
			- Guest Port: **500**
- **LAN** Network:  
    
    ![VirtualBox 'Add Network' view][104]
 	- Network Name: LAN_net
	- Network CIDR: 10.99.99.0/24
	- Network Options: Supports DHCP  
### Virtual machine  
You can use the prepared virtual machine found [here][1], or create the virtual machine from scratch. If the former is the case, the rest of this section can be skipped.  
We'll start by creating a virtual machine on Virtual Box:  
1. **Download** the [pfsense-2.4.4 ISO][2]
2. **Create** a new virtual machine on Virtualbox:
	- Type: **BSD**
	- Version: **Free-BSD (64-bit)**
	- Minimum resources:
		- RAM: **512 MB**
		- Disk: **4 GB**
3. **Mount** the pfsense-2.4.4 ISO
4. **Start** the VM and **install pfsense**. Follow instructions [here][3]
5.  **Assign** WAN/LAN interfaces from pfsense console. Follow instructions found [here][4]
6. If you find issues accessing the pfsense Web-UI, see  ```The pfsense GUI issues``` section bellow.

## OpenStack VM: 
### Virtual Machine Image: 
You can use the prepared virtual machine found [here][5], or use the previously created VM in VirtualBox, export it as OVA, extract the disk file and convert it to qcow2 using **qemu-img** found in ```qemu-utils``` package.
~~~shell
tar xvf PFsense\ 2.4.4.ova
qemu-img convert -f vmdk -O qcow2 PFsense\ 2.4.4-disk001.vmdk PFsense\ 2.4.4.qcow2
~~~
#### Import the ```.qcow2``` image
You can import the QCOW2 image from OpenStack's Dashboard: ```Project```->```Compute```->```Images```: ```Create Image```
#### Launch an instance  
Instantiate the Pfsense-2.4.4 image with resource flavor of at least 20 GB of storage and 1GB of memory (the resources should be at least those of the VM created previously on VirtualBox)  
The VM should be attached to two Networks (see the sub-sections bellow for more details)
![OpenStack 'Compute Instances' view][106]  

### OpenStack Networks: 
 Attach at least two networks to the pfsense VM, one of which is routed to the external network (for Floating IP assignement)  
![OpenStack 'Network Topology' view][107]  
- The network with routes to the external network will be the WAN interface and it will have a public floating IP.
- The other network will be the LAN network.
### OpenStack Port Security:
By default, OpenStack ports are configured to drop any Egress packet with a source MAC/IP pair that does not match that of the port (considered to be IP spoofing). 
Since pfsense will be routing traffic between the LAN network and remote networks, any traffic coming from those networks will be dropped by OpenStack's Port Security policies. 
To overcome this, we'll need to add some entries in the ```Allowed Address Pairs``` section of pfsense VM's LAN port configuration from OpenStack dashboard.  
Locate the LAN Port by its IP address from ```Project```->```Network```->```Networks```->```Ports``` then click on it.  

![OpenStack 'Network Port' view][109]  
  
Add remote networks expected to communicate with the LAN Network ```Data_net```, the MAC address should be that of the VM Port itself

![OpenStack 'Port Allowed Address Pairs' view][108]  

### OpenStack Security group:  
The security group assigned to the pfsense VPN VM must allow the following ingress/egress traffic for IPSec VPN to work:
- **N.B:** The ESP and AH protocol are only here for reference, because the pfsense WAN interface is behind NAT (Floating IP) the connection will only use UDP 500 for Ikev2 traffic, and ESP/AH traffic will use NAT-Traversal (UDP 4500).

|  Name  |     Protocol    | Port |
|:------:|:---------------:|:----:|
| ISAKMP |       UDP       |  500 |
|  NAT-T |       UDP       | 4500 |
|   ESP  | IP Protocol #50 |  -   |
|   AH   | IP Protocol #51 |  -   |

## IPSec Tunnel Configuration: 
### OpenStack Pfsense:
1. Phase 1: Ikev2 configuration - From ```VPN``` -> ```IPSec``` -> ```Tunnels```: ```Add Phase 1```  
![Pfsense 'IPsec VPN Tunnel Phase 1' view][110]  

2. Phase 2: Security Associations - From ```VPN``` -> ```IPSec``` -> ```Tunnels```: ```Add Phase 2```  
![Pfsense 'IPsec VPN Tunnel Phase 2' view][111]  

### VirtualBox Pfsense:
1. Phase 1: Ikev2 configuration - From ```VPN``` -> ```IPSec``` -> ```Tunnels```: ```Add Phase 1```  
![Pfsense 'IPsec VPN Tunnel Phase 1' view][112]  

2. Phase 2: Security Associations - From ```VPN``` -> ```IPSec``` -> ```Tunnels```: ```Add Phase 2```  
![Pfsense 'IPsec VPN Tunnel Phase 2' view][113]  

## OpenVPN TLS Tunnel Configuration: 
You can find many straight-through tutorials for setting up OpenVPN on Pfsense like [this one][6].

# The pfsense GUI issues:
The pfsense web-ui should be accessed from or through a LAN host, if any issues arise, consult the following list for 
troubleshooting:

1. Disable HTTP Referer: From CLI 'OpenStack VM Console View':  
	~~~shell
	PHP shell + pfsense tools (12)
	disablereferercheck;
	enableallowallwan;
	exec;
	~~~

2.  Tunnel (ssh tunnel) traffic between the local working machine and a VM with access to one of the LANs to which the pfsense VM is connected: This is used to access the GUI from the LAN network if your machine in not the that LAN
	~~~shell
	ssh -f -i <KEY_FILE> -L 48001:10.47.1.27:80 root@<FLOATING_IP> sleep 6000
	firefox localhost:48001
	~~~

4. Disable the packet filter if necessary:

	~~~shell
	shell (8)
	pfctl -d
	~~~

5. Add firewall rule to allow GUI access from WAN:  
You can try accessing the web-ui from WAN (not recommended)
6. Add firewall rule to allow IPSec traffic (ISAKMP, NAT-T, ESP, AH):  
This step is not necessary because by default **Pfsense** automatically enables these rules when IPsec tunnel are in place.  
This behaviour can be disabled from ```System``` -> ```Advanced``` -> ```Firewall & NAT```: ```Disable all auto-added VPN rules```

	|  Name  |     Protocol    | Port |
	|:------:|:---------------:|:----:|
	| ISAKMP |       UDP       |  500 |
	|  NAT-T |       UDP       | 4500 |
	|   ESP  | IP Protocol #50 |      |
	|   AH   | IP Protocol #51 |      |


[1]: https://mega.nz/#!OaBURCZK!sZ_LfQoy8Dw_Er-F5iMdM4JttXdC6XgTy7XxMG0jun4
[2]: https://nyifiles.pfsense.org/mirror/downloads/pfSense-CE-2.4.4-RELEASE-p3-amd64.iso.gz
[3]: https://docs.netgate.com/pfsense/en/latest/install/installing-pfsense.html#performing-a-full-install-iso-memstick
[4]: https://docs.netgate.com/pfsense/en/latest/install/installing-pfsense.html#assign-interfaces-on-the-console
[5]: https://mega.nz/#!iWhHiApA!s2y9uMXHINVAMlTEHIYyHIg4wDx5nCJzitxK61sap_s
[6]: https://turbofuture.com/computers/How-to-Setup-a-Remote-Access-VPN-Using-pfSense-and-OpenVPN


[100]: illustrations/IPSec_VPN_Setup-Network_Diagram-Public.jpg
[101]: illustrations/VirtualBox_Networks_01.png
[102]: illustrations/VirtualBox_Networks_02.png
[103]: illustrations/VirtualBox_Networks_03.png
[104]: illustrations/VirtualBox_Networks_04.png
[105]: illustrations/OpenStack_Instance_Pfsense_01.png
[106]: illustrations/OpenStack_Instance_Pfsense_02.png
[107]: illustrations/VPN_Network_Topology.png
[108]: illustrations/OpenStack_Allowed_Address_Pairs.png
[109]: illustrations/OpenStack_Allowed_Address_Pairs_02.png

[110]: illustrations/PFsense-IPSec-A01.png
[111]: illustrations/PFsense-IPSec-A02.png
[112]: illustrations/PFsense-IPSec-B01.png
[113]: illustrations/PFsense-IPSec-B02.png
