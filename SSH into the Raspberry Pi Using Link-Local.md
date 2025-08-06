Open configuration for Wired Network, make sure the method used is "Link-Local Only"

Check computer's IP address for ethernet interface, make sure it starts with 169.254. That means APIPA (Automatic Private IP Addressing) and computer is unable to obtain IP address from DHCP server.

Connect Raspberry Pi to computer with ethernet cable

Find IP address of the Raspberry Pi using  
`avahi-resolve --name <hostname>.local
