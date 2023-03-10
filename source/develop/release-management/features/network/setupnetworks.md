---
title: SetupNetworks
authors:
  - drankevi
  - mkolesni
  - moti
  - roy
  - ykaul
---

# Setup Networks

## Network features design

### Owner

*   Name: Roy Golan
*   Email: <rgolan@redhat.com>

### Abstract

Setup networks api will enable complex network provisioning of a host i.e. add/remove/bond network with a single call.
 Hereby we describe code POV changes for Engine as well as VDSM API implications.

#### Scope

1.  add/remove network/s
2.  add/remove bond/s
3.  attach/detach networks to/from bonds
4.  **check connectivity & connectivity timeout:**
    after the new topology layed by VDSM, it tests if any client(i.e engine) has interacted with it in a given period timeout.
     When no activity seen, it reverts to the previously backed-up topology and fail the command.
5.  **force:** VDSM will not validate parameters passed to set the network configuration.

#### Backward compatibility

Cluster version is 3.1. Add record in the action_version_map table:

| action_id  | cluster_minimal_version   | storage_pool_minimal_version    |
|------------|---------------------------|---------------------------------|
| 158        | 3.1                       | 3.1                             |
{: .bordered}

#### Setup Networks sequence diagram

![File:Diagram1.png](/images/wiki/Diagram1.png)

#### Added classes

1. `SetupNetworksCommand.java`
2. `SetupNetworksVdsCommand.java`
3. `SetupNetworksCommandParameters.java`
4. `SetupNetworksVdsCommandParameters.java`

##### Class Diagram

![File:SetupNetworksClassDiagram.png](/images/wiki/SetupNetworksClassDiagram.png)

### VDSM

##### API

**configNetwork.py**

```python
def setupNetworks(networks={}, bondings={}, **options):
```

from the python doc
```python
"""
 Add/Edit/Remove configuration for networks and bondings.
    Params:
        networks - dict of key=network, value=attributes
                   where 'attributes' is a dict with the following optional items:
                        vlan=<id>
                        bonding="<name>" | nic="<name>"
                        (bonding and nics are mutually exclusive)
                        ipaddr="<ip>"
                        netmask="<ip>"
                        gateway="<ip>"
                        bootproto="..."
                        delay="..."
                        onboot="yes"|"no"
                        (other options will be passed to the config file AS-IS)
                        -- OR --
                        remove=True (other attributes can't be specified)
        bondings - dict of key=bonding, value=attributes
                   where 'attributes' is a dict with the following optional items:
                        nics=["<nic1>" , "<nic2>", ...]
                        options="<bonding-options>"
                        -- OR --
                        remove=True (other attributes can't be specified)
        options - dict of options, such as:
                        force=0|1
                        connectivityCheck=0|1
                        connectivityTimeout=<int>
                        explicitBonding=0|1
    Notes:
        Bondings are removed when they change state from 'used' to 'unused'.

        By default, if you edit a network that is attached to a bonding, it's not
        necessary to re-specify the bonding (you need only to note the attachement
        in the network's attributes). Similarly, if you edit a bonding, it's not
        necessary to specify its networks.
        However, if you specify the 'explicitBonding' option as true, the function
        will expect you to specify all networks that are attached to a specified
        bonding, and vice-versa, the bonding attached to a specified network.

"""
```


The changes we need to communicate are in the "network" structure for bridge and MTU fields

**network business entity**

![File:Diagram2.png](/images/wiki/Diagram2.png)

> note:  integer types are serialized to an xml String type on the wire - its up to the VDSM network dict to handle.

##### Error codes

*   ERR_BAD_PARAMS = 21
    -   Cannot specify any attribute when removing
    -   Don't specify both nic and bonding
    -   Must specify either nic or bonding
    -   Must specify nics for bonding
    -   Network %s requires unspecified bonding
    -   Bonding %s is associated with unspecified network
    -   No action specified
    -   No action specified
    -   Unknown action specified

*   ERR_BAD_ADDR = 22
    -   Bad IP address
    -   IP address is already in use
    -   Bad netmask
    -   Bad gateway
    -   Must specify netmask to configure ip for bridge
    -   Specified netmask or gateway but not ip
    -   Must specify netmask to configure ip for bridge
    -   Specified netmask or gateway but not ip
    -   Bad IP address

*   ERR_BAD_NIC = 23
    -   not all nics are enslaved
    -   unknown nic
    -   delNetwork: nics are not all nics enslaved to bond
    -   unknown nic
    -   unknown nics

*   ERR_USED_NIC = 24
    -   nic is already bound to bridge
    -   nic already used by vlans
    -   nic already enslaved to bonds
    -   Setup attached both network and bonding to nic

*   ERR_BAD_BONDING = 25
    -   is not a valid bonding device name
    -   Bonding options specified without bonding
    -   bonding is already member of bridge
    -   bonding already has members
    -   multiple nics require a bonding device
    -   Cannot remove bonding Doesn\'t exist'
    -   Nic is attached to two different bondings in setup

*   ERR_BAD_VLAN = 26
    -   vlan id out of range
    -   vlan id is not a number

*   ERR_BAD_BRIDGE = 27
    -   bridge doesn't exist

*   ERR_USED_BRIDGE = 28
    -   bridge is in use

*   ERR_LOST_CONNECTION = 10
    -   client is not seed during check_connectivity lapse

###### structure

Callers should be able to tell which network attribute is making the problem

e.g the network's vlan configuration is not valid

```xml
 <error>
  <code>26</code>
  <name>ERR_BAD_VLAN</name>
  <message/>vlan id {network.vlan_id} for network {network.name} must be a number</message>
  <network>
   <name>Yellow</name>
   <vlan_id>3oo</vlan_id>
  </network>
 </error>
```

### GUI

![File:general.png](/images/wiki/General.png)

![File:more_cases.png](/images/wiki/More_cases.png)

### UI Alternative Suggestion

I would like to describe a suggestion for a dialog, where the state of the network elements will be described using Blocks and Links.

The Networking Elements here are taken from the RHEV Technical guide: [RHEV Technical Reference Guide](https://web.archive.org/web/20120331001716/http://docs.redhat.com/docs/en-US/Red_Hat_Enterprise_Virtualization/3.0/html/Technical_Reference_Guide/sect-Technical_Reference_Guide-Network_Architecture-Networking_in_Hosts_and_Virtual_Machines.html)

The UI Blocks describe the following networking elements: (Each element will be represented by a block, with a specific color)

1.  Network Interface Card (NIC) (eth0, eth1)
2.  Bond between two or more network Interfaces (bond01, bond1)
3.  Virtual Network (either bridge or bridgeless)
4.  VLAN on one Virtual Network.

*   Blocks can be connected to each-other by a visible line.
*   The scenarios for connecting block are specified below
*   Dbl-click or Click on each block will open it's setting dialog.
*   A block that can be removed (like a bond, or a VLAN) will have a red X icon.
*   A line can be removed by right clicking on it.
*   Lines will be automatically removed when their connecting elements are deleted.
*   There is a possibility to reconnect lines when they are automatically deleted (when a bond is removed, reconnect the line to one of the unbonded NICs).

#### What cannot be connected

The connection of the following elements will be prevented:

1.  Connecting a Network with no VLAN to a NIC/Bond that is already connected to a Network.
2.  Connecting a Network to a NIC/Bond that is already connected to a Bridgeless Network

#### What can be connected

It will be possible to drag blocks on one another. The following connections are supported:

| Drag Action | Result | Illustration |
|-------------|--------|--------------|
|Drag a Network on a NIC, Drag a NIC on a Network|Create a connection between the NIC and the Bridge|1-3|
|Right-click on the Network?                     |Create a VLAN on the Logical Network
|Drag a VLAN/Network on a NIC, Drag a NIC on a VLAN/Network|Create a connection between the NIC and the VLAN4-6|
|Drag an unbonded NIC onto another unbonded NIC|Create a new Bond element for the two NICS|7,8|
|Drag a Bond on a VLAN/Network|Create a connection between the Bond and the VLAN/Network|11|
{: .bordered}

#### Images

### New Setup Networks Mockup

------------------------------------------------------------------------

![File:SetupNetworksNew.png](/images/wiki/SetupNetworksNew.png)

![File:Mockup-rollover-notfinish.png](/images/wiki/Mockup-rollover-notfinish.png)

### REST

#### Resource representation

##### Scheme

```xml
 <link href="/api/hosts/{host:id}/nics/setupnetworks" rel="setupnetworks">
  <request>
   <http_method>POST</http_method>
   <headers>
    <header>
     <name>Content-Type</name>
     <value>application/xml|json</value>
   </header>
   </headers>
   <body>
    <type>Action</type>
    <parameters_set>
     <parameter required="false" type="collection"><name>action.host_nics.host_nic</name>
      <parameters_set>
       <parameter required="false" type="xs:string"><name>host_nic.network.id|name</name></parameter>
       <parameter required="false" type="xs:string"><name>host_nic.name</name></parameter>
       <parameter required="false" type="xs:string"><name>host_nic.ip.gateway</name></parameter>
       <parameter required="false" type="xs:string"><name>host_nic.boot_protocol</name></parameter>
       <parameter required="false" type="xs:string"><name>host_nic.mac</name></parameter>
       <parameter required="false" type="xs:string"><name>host_nic.ip.address</name></parameter>
       <parameter required="false" type="xs:string"><name>host_nic.ip.netmask</name></parameter>
       <parameter required="false" type="collection"><name>host_nic.bonding.options.option</name>
       <parameters_set>
        <parameter required="false" type="xs:string"><name>option.name</name></parameter>
        <parameter required="false" type="xs:string"><name>option.value</name></parameter>
        <parameter required="false" type="xs:string"><name>option.type</name></parameter>
       </parameters_set>
    </parameter>
    <parameter required="false" type="collection"><name>bonding.slaves.host_nic</name>
     <parameters_set> 
      <parameter required="false" type="xs:string"><name>host_nic.name|id</name></parameter>
     </parameters_set>
    </parameter>
    </parameters_set>
    </parameter>
    <parameter required="false" type="xs:boolean"><name>action.checkConnectivity</name></parameter>
    <parameter required="false" type="xs:int"><name>action.connectivityTimeout</name></parameter>
    <parameter required="false" type="xs:boolean"><name>action.force</name></parameter>
    </parameters_set>
  </body>
  </request>
  <response>
   <type>Response</type>
  </response>
 </link>
```

#### Ssage

##### Attaching a network to a NIC

`POST /api/hosts/{host:id}/nics/setupnetworks`

```xml
 <action>
  <host_nics>
    <host_nic>
        <name>em1</name>
        <network id="00000000-0000-0000-0000-000000000009"/>
        <ip address="10.35.1.247" netmask="255.255.254.0" gateway="10.35.1.254"/>
        <boot_protocol>dhcp</boot_protocol>
    </host_nic>
    <checkConnectivity>true</checkConnectivity>
    <connectivityTimeout>60</connectivityTimeout>
    <force>false</false>
  </host_nics>
 </action>
```

##### Attaching several (VLAN'd) networks to a NIC

The expected VLAN device should be sent with the parameters of the network which is created, in addition to the NIC that the network is being added on.

The expected name for this device is: NIC_NAME.VLAN_ID

`POST /api/hosts/{host:id}/nics/setupnetworks`

```xml
 <action
  <host_nics>
    <host_nic>
        <name>em1</name>
    </host_nic>
    <host_nic>
        <name>em1.100</name>
        <network>
            <name>vlan100</name>
        </network>
        <ip address="10.35.100.247" netmask="255.255.254.0" gateway="10.35.100.254"/>
        <boot_protocol>dhcp</boot_protocol>
    </host_nic>
    <host_nic>
        <name>em1.200</name>
        <network>
            <name>vlan200</name>
        </network>
        <ip address="10.35.200.247" netmask="255.255.254.0" gateway="10.35.200.254"/>
        <boot_protocol>dhcp</boot_protocol>
    </host_nic>
    <checkConnectivity>true</checkConnectivity>
    <connectivityTimeout>60</connectivityTimeout>
    <force>false</false>
  </host_nics>
 </action>
```

### Open issues

1.  input validation: whats the MTU max value? how do we calculate it?

A (ykaul): You don't know what the MTU max value is. Theoretically, around 9000 bytes, perhaps 9216 in some cases. Practically, some interfaces may support less, some may (in the future?) support more. The max. frame size is vendor-dependent. I wouldn't bother with limiting it. Note also that since it's interface specific (hw+firmware+driver), no validation would really work apart from setting it up for real and hoping for an error code if it's not supported.
