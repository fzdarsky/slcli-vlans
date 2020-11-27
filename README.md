# slcli-vlans
Tool to manipulate VLAN trunking on IBM Cloud via the SoftLayer APIs.

It is possible to trunk VLANs to bare metal machines on IBM Cloud, but neither the [IBM Cloud web GUI](https://cloud.ibm.com/login) nor the available CLIs ([ibmcloud](https://github.com/IBM-Cloud/ibm-cloud-cli-release) and [slcli](https://github.com/softlayer/softlayer-python)) support this. Instead, one has to exercise the [corresponding SoftLayer APIs](https://cloud.ibm.com/docs/vlans?topic=vlans-vlans-faqs#how-do-i-trunk-my-vlans-to-my-servers-) directly.

`slcli-vlans` has a very limited purpose: Simplifying getting, adding, and removing VLAN trunks from network interfaces.

## Getting the tool
Download the tool using `curl`, make it executable, and move it into a directory in your `$PATH`:

    $ curl -LOs https://raw.githubusercontent.com/fzdarsky/slcli-vlans/main/slcli-vlans
    $ chmod a+x slcli-vlans
    $ mkdir -p $HOME/bin && mv slcli-vlans $HOME/bin
    $ export PATH=$PATH:$HOME/bin

## Requirements
You'll need `python3` with `pip` installed. Use `pip3` to install the SoftLayer Python libraries:

    $ pip3 install softlayer

Next, create an IBM Cloud API key following the [documentation](https://cloud.ibm.com/docs/account?topic=account-userapikey#create_user_key) and add it to a `.softlayer` secret file in your home directory so it looks like this:

    $ cat ~/.softlayer
    [softlayer]
    username = apikey
    api_key = YOUR_API_KEY_HERE
    endpoint_url = https://api.softlayer.com/rest/v3.1/
    timeout = 40

## Usage
### Listing hardware
You can list all hardware in your account like this:

    $ slcli-vlans list_hardware
    id      fullyQualifiedDomainName eth0_ip      eth0_net     eth1_ip         eth1_net        
    ------------------------------------------------------------------------------------
    3763273 bastion.ibm.cloud        10.85.40.1   private-dev  158.177.107.201 public          
    3900606 bastion.example.com      10.85.100.1  private-prod 158.177.107.202 public          
    3036480 machine-01.example.com   10.85.100.2  private-prod 10.194.188.114  storage
    3036478 machine-02.example.com   10.85.100.3  private-prod 10.194.188.113  storage   
    3036458 machine-03.example.com   10.85.100.4  private-prod 10.194.188.95   storage   

Results can be filtered using the `--domain`, `--hostnames`, and `--hostname-prefix` flags:

    $ slcli-vlans list_hardware --hostname-prefix machine
    id      fullyQualifiedDomainName eth0_ip      eth0_net     eth1_ip         eth1_net        
    ------------------------------------------------------------------------------------
    3036480 machine-01.example.com   10.85.100.2  private-prod 10.194.188.114  storage
    3036478 machine-02.example.com   10.85.100.3  private-prod 10.194.188.113  storage  
    3036458 machine-03.example.com   10.85.100.4  private-prod 10.194.188.95   storage  

    $ slcli-vlans list_hardware --hostnames bastion,machine-01
    id      fullyQualifiedDomainName eth0_ip      eth0_net     eth1_ip         eth1_net        
    ------------------------------------------------------------------------------------
    3763273 bastion.ibm.cloud        10.85.40.1   private-dev  158.177.107.201 public          
    3900606 bastion.example.com      10.85.100.1  private-prod 158.177.107.202 public          
    3036480 machine-01.example.com   10.85.100.2  private-prod 10.194.188.114  storage

It is possible to output results in JSON format using the `--json` flag:

    $ slcli-vlans list_hardware --domain ibm.cloud --json
    [{'id': 3763273, 'fullyQualifiedDomainName': 'bastion.ibm.cloud', 'eth0_ip': '10.85.40.1', 'eth0_net': 'private-dev', 'eth1_ip': '158.177.107.201', 'eth1_net': 'public'}]

### Listing VLANs
You can list all VLANs in your account like this:

    $ slcli-vlans list_vlans
    id      name         vlanNumber 
    --------------------------------
    2879300 public       1791       
    2960720 private-dev  1020       
    2960721 private-prod 2001       
    2960722 private-svc1 2002       
    2960723 private-svc2 2003       

### Querying an interface's native VLAN
To query the native VLAN of a hardware's network interface, you need to specify the target interface in one of three formats that allow the interface to be uniquely identified within the account:
* the interface's primary IP address,
* the FQDN of the host plus the host's local interface name (comma-separated), or
* the FQDN of the host plus the network (native VLAN) name the interface connects to (comma-separated).

The following invocations are therefore equivalent:

    $ slcli-vlans get_vlan 10.85.100.2
    private-prod
    
    $ slcli-vlans get_vlan machine-01.example.com,eth0
    private-prod

    $ slcli-vlans get_vlan machine-01.example.com,private-prod
    private-prod

### Managing an interface's trunked VLANs
Use the `get_vlan_trunks`, `add_vlan_trunks`, and `clear_vlan_trunks` commands to query, add, and remove all interface's VLAN trunks, respectively. Each command requires an identifier of the target interface in one of the three formats shown in the previous section. `add_vlan_trunks` further requires a comma-separated list of VLAN names to add.

Example:

    $ slcli-vlans get_vlan_trunks machine-01.example.com,private-prod
    []
    
    $ slcli-vlans add_vlan_trunks machine-01.example.com,private-prod private-svc1,private-svc2
    [private-svc1,private-svc2]

    $ slcli-vlans clear_vlan_trunks machine-01.example.com,private-prod
    []

