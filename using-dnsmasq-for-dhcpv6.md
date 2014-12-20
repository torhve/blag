Using dnsmasq on a Linux router for DHCPv6
==========================================

[Dnsmasq](http://www.thekelleys.org.uk/dnsmasq/doc.html) is a nice little supertool for your home networks. At my house it provides DHCPv4, DHCPv6, tftp, and DNS services for my all my LAN clients. Recently my ISP started offering native IPv6 using [IPv6 Prefix Delegation](http://en.wikipedia.org/wiki/Prefix_delegation) and I want to offer IPv6 connectivity to my LAN clients using dnsmasq. Historically the software package `radvd` was commonly used for just the RA-part of this. But dnsmasq offers a more complete setup. This document will try to provide some information for others looking to do the same, or you know, for myself when I forget how I did it.

Configuring dhcpv6
------------------

To request IPv6 address and subnet information from my WAN connection I use the package `wide-dhcpv6-client`. 

Lets try to exlain the configuration file I set up for it:
> `/etc/wide-dhcpv6/dhcp6c.conf`


    # eth0 is my external facing interface (WAN)
    interface eth0 { 
    # request a non-temporary address   
        send ia-na 1;
        # request prefix delegation address
        send ia-pd 1;
        # send rapid commit, don't wait for RA
        send rapid-commit;
        # we'd like information about DNS, too
        request domain-name-servers;
        request domain-name;
        # script provided by my distribution, it adds nameservers to resolv.conf
        script "/etc/wide-dhcpv6/dhcp6c-script";
    };

    id-assoc pd 1 {
        # internal facing interface (LAN), you can duplicate this section if you want more subnets for more interfaces   
        prefix-interface eth1 { 
            # subnet. Combined with ia-pd to configure the subnet for this interface.   
            sla-id 0; 
            #IP address "postfix". if not set it will use EUI-64 address of the interface. Combined with SLA-ID'd prefix to create full IP address of interface. In my case, ifid 1 means that eth1 will get a IPv6 ending with ::1
            ifid 1; 
            # prefix bits assigned. Take the prefix size you're assigned (something like /48 or /56) and subtract it from 64. In my case I was being assigned a /56, so 64-56=8
            sla-len 8; 
        };
    };

    id-assoc na 1 {
        # id-assoc for eth1
    };

Alright, and to get your linux networking stack to accept Route Advertisements and act as a router you need the following sysctl parameters:

    net/ipv6/conf/all/forwarding=1
    net/ipv6/conf/eth0/accept_ra=2

Update your sysctl.conf and activate them
    
After this, you should be able to start wide-dhcp6-client service and check that you have successfully received addressed on both your WAN interface (`eth0`) and your LAN interface(s) `eth1`. Use command `ip -6 addr` to list all IPv6 addresses on your system. Please keep in mind that any addresses starting with `fe80` or `::1` are either link-local or loopback.

Configuring dnsmasq
-------------------

Now that we have got valid IPv6 addresses on the interfaces, lets use dnsmasq to provide the information to the clients on your LAN. I will not show you every possible setting in dnsmasq, but the most critical ones.

> `/etc/dnsmasq.conf`

    # don't ever listen to anything on eth0
    except-interface=eth0
    
    # don't send bogus requests out on the internets
    bogus-priv
    
    # enable IPv6 Route Advertisements
    enable-ra
    
    # Construct a valid IPv6 range from reading the address set on the interface. The ::1 part refers to the ifid in dhcp6c.conf. Make sure you get this right or dnsmasq will get confused.
    dhcp-range=tag:eth1,::1,constructor:eth1, ra-names, 12h
    
    # ra-names enables a mode which gives DNS names to dual-stack hosts which do SLAAC  for  IPv6.
    # Add your local-only LAN domain
    local=/lan.mydomain.no/
    
    #  have your simple hosts expanded to domain
    expand-hosts
    
    # set your domain for expand-hosts
    domain=lan.mydomain.no
    
    # provide a IPv4 dhcp range too
    dhcp-range=lan,172.16.36.64,172.16.36.127,12h
    
    # set authoritative mode
    dhcp-authoritative
    
Restart your dnsmasq server to activate the new settings and verify that your clients successfully receieve IPv6 addresses.

Firewall settings
-----------------

It's important to note that the addresses your clients will be reachable publically, much unlike the standard IPv4+NAT setup commonly used. A full firewall configuration is out of the scope of this article but please atleast install these minimal settings:

    
    # Default policy DROP
    ip6tables -P FORWARD DROP
    # Allow established connections
    ip6tables -I FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT
    # Accept packets FROM LAN to everywhere
    ip6tables -I FORWARD -i eth1 -j ACCEPT

These settings will in effect block everything from the internets to your LAN (except answer to traffic you requests) but allow clients access to WAN.    
 

Next steps
----------

Check that your IPv6 connectitivty works fully, including DNS, and maybe test your IPv6 speed:
http://ipv6-test.com/speedtest/

