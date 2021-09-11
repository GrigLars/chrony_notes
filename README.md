# Chrony notes
### My notes on installing an Ubuntu Chrony Server dredged from much Ubuntu headache

#### Variables used in these notes:
    Name of Time server: my-timeserver
    IP address of server: 192.168.9.2 (subnet is a /24)
    Distro used: Ubuntu 18.04 LTS
    
So, I had to install chrony, a time client/server, on AWS systems because ntp is antiquated, and they recommend it.  This led to installing a chrony server of some kind for a local cluster, and I found that despite a lot of how-tos, I ran into a lot of errors and headaches.  I hope this helps someone else.

#### Remove the NTP package
Pretty straight forward.  I am using Debian's "apt" here, on Ubuntu, but you can use whatever your distro has: yum, dnf, apt-get, and so on.

    sudo apt remove ntp
    
#### Then you install chrony

    sudo apt install chrony 

#### Then it wouldn't work

Then I had to test the server from a client.

    sudo chronyd -q 'server 192.168.9.2 iburst'
    2021-09-11T21:21:16Z chronyd version 3.2 starting (+CMDMON +NTP +REFCLOCK +RTC +PRIVDROP +SCFILTER +SECHASH +SIGND +ASYNCDNS +IPV6 -DEBUG)
    2021-09-11T21:21:16Z Initial frequency -5.571 ppm
    2021-09-11T21:21:27Z No suitable source for synchronisation
    2021-09-11T21:21:27Z chronyd exiting

#### I had to fix a lot of config file weirdness 

Was my firewall up?  No, I have no firewall on this internal network.  I did a HECK of of a lot of searching.  I did tcpdump and showed that requests were being sent to ntp.  The problem is, all distros have files in different places, and some people were doing different thngs than I wanted to do.  A majority of help guides just assumed everything worked out of the box or my clients were fine with ntp.ubuntu.com pools.  It took a day to gather all these notes of things I had to change on Ubuntu 18.04 LTS with systemd.

For reasons I am unable to explain, a lot of the conf files were put in /etc and/or referred to as such.  To save yourself a lot of headache, make sure everything is in /etc/chrony and move anything from /etc to there. In my case, some files were in BOTH places, and I was editing the wrong one. Most other services depend on everything being in /etc/chrony/ like systemd which I had to find out the hard way.

    mv /etc/chrony.conf /etc/chrony/chrony.conf
    mv /etc/chrony.keys /etc/chrony/chrony.keys
    mv /etc/chrony.drift /etc/chrony/chrony.drift

In the config file /etc/chrony/chrony.conf, change everything to reflect the new place. If they are already there and correct, YAY! You're ahead of the game. Give yourself a cookie. In this git repo, find my server config with notes.

#### Why the hell are my clients using other time sources?

Man, this was a stumper. Here's an example output.  In this readout of "chronyc sources -v," my-timeserver is the correct one.  The 192.168.1.1 was an old, old gateway/router I haven't used for years, so of course it wasn't giving correct time.  And I didn't want time1.google.com AND my local time server "combined," because they'd always have some kind of difference simply due to comparitive network latency.

    glarson@localhost:~$ chronyc sources -v
    210 Number of sources = 3

      .-- Source mode  '^' = server, '=' = peer, '#' = local clock.
     / .- Source state '*' = current synced, '+' = combined , '-' = not combined,
    | /   '?' = unreachable, 'x' = time may be in error, '~' = time too variable.
    ||                                                 .- xxxx [ yyyy ] +/- zzzz
    ||      Reachability register (octal) -.           |  xxxx = adjusted offset,
    ||      Log2(Polling interval) --.      |          |  yyyy = measured offset,
    ||                                \     |          |  zzzz = estimated error.
    ||                                 |    |           \
    MS Name/IP address         Stratum Poll Reach LastRx Last sample               
    ===============================================================================
    ^+ my-timeserver                 2   6     1     1   -260us[ -260us] +/-   15ms
    ^? 192.168.1.1                   0   6     0     -     +0ns[   +0ns] +/-    0ns
    ^* time1.google.com              1   6     1     1  -4759us[-4759us] +/-   17ms

After a lot of digging, I found these were cached in two places:

    /var/lib/dhcp/chrony.servers.enp4s0     # (where enp4s0 was my first ethernet card)
    /var/run/chrony-helper/added_servers

In each, were these:

/var/lib/dhcp/chrony.servers.enp4s0

    192.168.1.1 iburst
    216.239.35.0 iburst

/var/run/chrony-helper/added_servers

    192.168.1.1
    216.239.35.0
    
You have to shut chrony off, delete all the lines in these files, and turn chrony back on.  I guess you COULD put your time server in there, but the system will do so automatically anyway.  Why do more work?

    sudo systemctl stop chrony 
    sudo echo "" >  /var/lib/dhcp/chrony.servers.enp4s0
    sudo echo "" >  /var/run/chrony-helper/added_servers
    sudo systemctl start chrony

If you just do a "restart," it will re-fill those files again.  You have to stop, edit, start again.

In addition, you have to make sure you don't have anything listed for "NTP=" in this file: /etc/systemd/timesyncd.conf (assuming you have systemd). If you do, comment it out under [Time] like so:

    #  This file is part of systemd.
    #
    #  systemd is free software; you can redistribute it and/or modify it
    #  under the terms of the GNU Lesser General Public License as published by
    #  the Free Software Foundation; either version 2.1 of the License, or
    #  (at your option) any later version.
    #
    # Entries in this file show the compile time defaults.
    # You can change settings by editing this file.
    # Defaults can be restored by simply deleting this file.
    #
    # See timesyncd.conf(5) for details.

    [Time]
    #NTP=
    #FallbackNTP=ntp.ubuntu.com
    #RootDistanceMaxSec=5
    #PollIntervalMinSec=32
    #PollIntervalMaxSec=2048

If you had to make changes, you need to reload systemd:

    sudo systemctl daemon-reload
    sudo timedatectl set-ntp off
    sudo timedatectl set-ntp on
    sudo systemctl status systemd-timesyncd

In your dnsmasq.conf of your dnsmasq DHCP service, you can also set the time servers like I did.

    # Set the NTP time server addresses to 192.168.0.4 and 10.10.0.5
    #dhcp-option=option:ntp-server,192.168.0.4,10.10.0.5
    dhcp-option=option:ntp-server,192.168.9.1

    # Set the NTP time server address to be the same machine as
    # is running dnsmasq is another option.
    #dhcp-option=42,0.0.0.0
    

#### Now it's working!

    sudo chronyd -q 'server 192.168.9.2 iburst'
    2021-09-11T23:39:51Z chronyd version 3.2 starting (+CMDMON +NTP +REFCLOCK +RTC +PRIVDROP +SCFILTER +SECHASH +SIGND +ASYNCDNS +IPV6 -DEBUG)
    2021-09-11T23:39:51Z Initial frequency -7.027 ppm
    2021-09-11T23:39:55Z System clock wrong by -0.000577 seconds (step)
    2021-09-11T23:39:55Z chronyd exiting

    chronyc sources -v
    210 Number of sources = 1

      .-- Source mode  '^' = server, '=' = peer, '#' = local clock.
     / .- Source state '*' = current synced, '+' = combined , '-' = not combined,
    | /   '?' = unreachable, 'x' = time may be in error, '~' = time too variable.
    ||                                                 .- xxxx [ yyyy ] +/- zzzz
    ||      Reachability register (octal) -.           |  xxxx = adjusted offset,
    ||      Log2(Polling interval) --.      |          |  yyyy = measured offset,
    ||                                \     |          |  zzzz = estimated error.
    ||                                 |    |           \
    MS Name/IP address         Stratum Poll Reach LastRx Last sample               
    ===============================================================================
    ^* yakko                         2   9   377   495  -1212us[-1305us] +/-   18ms

    chronyc sourcestats 
    210 Number of sources = 1
    Name/IP Address            NP  NR  Span  Frequency  Freq Skew  Offset  Std Dev
    ==============================================================================
    yakko                      26  12   61m     -0.019      0.571    -87ns   650us

But OMG that took so long to figure all out.  I hope my pain saves someone else some headache. 
