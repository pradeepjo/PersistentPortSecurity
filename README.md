# PersistentPortSecurity
EOS-SDK agent modifying port security behavior

Introduction:
Port security feature allows network administrators to specify the number of mac-addresses that can communicate through a switch port. When the specified mac-address limit is reached, a switch prevents frames sourced by new mac-addresses from ingressing the port. When an interface on the switch is flapped, it clears learnt mac-address cache on the interface and starts relearning mac-addresses until the user specified mac-address count is reached, before enforcing port security again on that interface. This can pose a security threat if an attacker has physical access to the switch. The attacker can unplug an existing cable on a switch port and reconnect it to a rogue host thus compromising the network. Existing port security feature does not protect the network in this case. 

PersistentPortSecurity agent protects the network in such cases by ensuring learned mac-address cache is not cleared during a link flap. The agent accepts interface range and mac-address count as inputs to enable users to decide which interfaces to activate this feature on and how many mac-addresses to allow to communicate through those interfaces. PersistentPortSecurity agent monitors each newly learned mac-address on the switch and configures mac access-lists per interface when the number of learned mac-addresses reaches allowed limit on user specified interfaces. Mac access-lists remain intact on interfaces irrespective of their operational status, thereby protecting the network from physical attackers. 


How to run PersistentPortSecurity agent:
1.	Download and install EOS SDK
https://github.com/aristanetworks/EosSdk/wiki/Downloading-and-Installing-the-SDK

2.	Enable “PersistentPortSecurity” agent
Copy “PersistentPortSecurityAgent.py” to “/mnt/flash:” of the switch and use following commands to enable agent - 

switch(config)#daemon PersistentPortSecurity
switch(config-daemon-PersistentPortSecurity)#exec /mnt/flash/PersistentPortSecurityAgent.py
switch(config-daemon-PersistentPortSecurity)#no shutdown

3.	Configuration options 
a)	Interface -
The “interface” option allows a user to specify which interface(s) to enable PersistentPortSecurity agent on. Users can specify any interface or interface range (Ethernet or Port-Channel) in the same format that eAPI allows -

switch(config)#daemon PersistentPortSecurity
switch(config-daemon-PersistentPortSecurity)#option interface value <Interface_range>

“Interface_range” accepts the same input that eAPI accepts - For example, 
option interface value Ethernet1,2,21/1,24/4
option interface value Ethernet1-28/1
option interface value Port-Channel1
option interface value Port-Channel2-100

In a specified range, PersistentPortSecurity agent is only enabled on interfaces which are in switchport mode. To add further interfaces, simply use the same command specifying new interfaces to enable the feature on.

b)	Mac-address count -
This option allows a user to configure number of allowed mac-addresses. When the number of mac-addresses on an interface exceeds the specified count, this feature applies a MAC-ACL to the layer-2 interface, blocking traffic from new hosts (mac-addresses). Mac-address count can be enabled using the following command -

switch(config)#daemon PersistentPortSecurity
switch(config-daemon-PersistentPortSecurity)#option mac_addr_count value <count>

c)	Purge -
“Purge” option allows a user to specify which interface(s) to disable PersistentPortSecurity agent on - 

switch(config)#daemon PersistentPortSecurity
switch(config-daemon-PersistentPortSecurity)#option purge value <interface_range>

“Interface_range” in “purge” option also accepts Ethernet and Port-Channel interfaces. For example, 
option purge value Ethernet1,2,21/1,24/4
option purge value Ethernet1-28/1
option purge value Port-Channel1
option purge value Port-Channel2-100


Points to note:
1.	This agent needs “eAPI over Unix socket” to be enabled on the switch. 
eAPI over Unix socket can be enabled using following commands from switch CLI - 

switch(config)#management api http-commands
switch(config-mgmt-api-http-cmds)#protocol unix-socket
switch(config-mgmt-api-http-cmds)#no shutdown

The link below has more information on eAPI over Unix socket -
https://eos.arista.com/eapi-and-unix-domain-socket/

2.	Enabling/Disabling PersistentPortSecurity agent
To enable or disable PersistentPortSecurity on interfaces, use “option interface ...” and “option purge ...” commands respectively. Using “no” in front of “option” command may exhibit unexpected results.

3.	Shutting down the agent
The cleanest way to shut the agent down is to use “shutdown” command as shown below to ensure all mac access-lists and mac access-groups are removed from the switch before terminating the agent -

switch(config)#daemon PersistentPortSecurity
switch(config-daemon-PersistentPortSecurity)#shutdown


Caveat:
Repeating the same interface_range with either “interface” or “purge” option does not work.

For example, 
if you enable this feature using the following “interface” option -
switch(config-daemon-PersistentPortSecurity)#option interface value Ethernet21/1-28/4

And disable it using “purge” option -
switch(config-daemon-PersistentPortSecurity)#option purge value Ethernet21/1-24/4

Now if you use the same “interface_range” (Ethernet21/1-28/4) again, PersistentPortSecurity won’t be enabled on interfaces 21/1-24/4 -
switch(config-daemon-PersistentPortSecurity)#option interface value Ethernet21/1-28/4

The workaround is to change “interface_range” in “interface” option to be different from that in the previous “interface” option command. The same outcome can be achieved using the following two commands - 
switch(config-daemon-PersistentPortSecurity)#option interface value Ethernet21/1-28/3
switch(config-daemon-PersistentPortSecurity)#option interface value Ethernet28/4


Monitoring and visibility:
The following command shows operational status of the agent -

switch(config-if-Po2)#sh daemon PersistentPortSecurity
Agent: PersistentPortSecurity (running)
Configuration:
Option               Value
-------------------- -----------------
interface            Ethernet21/1-28/4
mac_addr_count       1
purge                Ethernet21/1-24/4

Status:
Data               Value
------------------ ---------------------
Ethernet25/1       []
Ethernet25/2       []
Ethernet25/3       []
Ethernet25/4       []
Ethernet26/1       []
Ethernet26/2       []
Ethernet26/3       []
Ethernet26/4       []
Ethernet27/1       ['00:1c:73:41:9d:e9']
Ethernet27/2       ['00:00:80:02:c3:8c']
Ethernet27/3       ['00:00:80:03:f7:bb']
Ethernet27/4       ['00:00:80:04:0b:df']
Ethernet28/1       ['00:00:80:05:63:f4']
Ethernet28/2       ['00:00:80:06:ca:c1']
Ethernet28/3       []
Ethernet28/4       []

The “Status” part of the output shows interfaces that PersistentPortSecurity agent is enabled on and mac-addresses learned on the interface. The above output only shows one mac-address against each interface as the mac-address count is set to 1. 

For more visibility into the agent, you can enable tracing using following CLI command and monitor “/var/log/agent/PersistentPortSecurity-<PID>” file from bash -
trace PersistentPortSecurity enable PersistentPortSecurityAgent all
