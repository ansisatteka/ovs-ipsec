..
      Licensed under the Apache License, Version 2.0 (the "License"); you may
      not use this file except in compliance with the License. You may obtain
      a copy of the License at

          http://www.apache.org/licenses/LICENSE-2.0

      Unless required by applicable law or agreed to in writing, software
      distributed under the License is distributed on an "AS IS" BASIS, WITHOUT
      WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the
      License for the specific language governing permissions and limitations
      under the License.

      Convention for heading levels in Open vSwitch documentation:

      =======  Heading 0 (reserved for the title in a document)
      -------  Heading 1
      ~~~~~~~  Heading 2
      +++++++  Heading 3
      '''''''  Heading 4

      Avoid deeper levels because they do not render well.

==============================================
How to Encrypt Open vSwitch Tunnels with IPsec
==============================================

This document describes how to use Open vSwitch integration with strongSwan
5.1 or later to provide IPsec security for STT, GENEVE, GRE and VXLAN tunnels.
This document assumes that you have already installed Open vSwitch.


Limitations
-----------

There are several limitations:

1) Currently only Debian-based platforms are supported.

2) There is no backwards compatibility with the old IPsec implementation
   that uses Racoon instead of strongSwan for IKE keying.

3) Some older Open vSwitch datapath kernel modules (in Linux Kernel tree)
   do not support route lookups with transport L4 ports properly.  In
   this case Ethernet over L4 tunneling protocols (e.g. STT, GENEVE)
   would not work.  However, GRE would still work because it does not
   have concept of L4 ports.

4) Some strongSwan versions might not support certain features.  For
   example:

   a) AES GCM ciphers that improve performance.
   b) xfrm_acq_expires setting in strongSwan configuration file.
      This setting tells strongSwan how aggressively to retry
      establishing tunnel, if peer did not respond to previous keying
      request.
   c) set_proto_port_transport_sa in charon configuration file that tells
      Linux Kernel to filter out unexpected packets that came over IPsec
      tunnel.
   d) IPsec transport mode for NAT deployment.

5) Since IPsec decryption is done from the software when packet has already
   gotten past NIC hardware, then benefits of the most tunneling offloads,
   like GENEVE or TSO for STT can't be leveraged unless NIC has IPsec offloads
   as well and the NIC knows how to leverage them.


Setup
-----

Install strongSwan and openvswitch-ipsec debian packages::

      $ apt-get install strongswan
      $ dpkg -i openvswitch-ipsec_<version>_amd64.deb


Configuration
-------------

IPsec with ovs-monitor-ipsec daemon can be configured in three
different forwarding policy modes:

1) ovs-monitor-ipsec assumes that packets from all OVS tunnels by default
   should be allowed to exit unencrypted unless particular tunnel is
   explicitly configured with IPsec parameters::

      % ovs-vsctl add-port br0 ipsec_gre0 -- \
                  set interface ipsec_gre0 type=gre \
                                     options:remote_ip=1.2.3.4 \
                                     options:psk=swordfish])


   The problem with this mode is that there is inherent race condition
   between ovs-monitor-ipsec and ovs-vswitchd daemons where ovs-vswitchd
   could enable forwarding before ovs-monitor-ipsec actually had a chance
   to configure IPsec policies.
   This mode should be used only and only if tunnel configuration is static
   and/or if there is firewall that can drop the plain packets that
   occasionally leak the tunnel unencrypted on OVSDB (re)configuration
   events.


2) ovs-monitor-ipsec assumes that packets marked with a given SKB mark
   must be encrypted and hence should not be allowed to leave the system
   unencrypted::

     % ovs-vsctl set Open_vSwitch . other_config:default_ipsec_skb_mark=1/1
     % ovs-vsctl add-port br0 ipsec_gre0 -- \
                 set interface ipsec_gre0 type=gre \
                                   options:remote_ip=1.2.3.4 \
                                   options:psk=swordfish])

   With this mode ovs-monitor-ipsec makes strongSwan to install IPsec shunt
   policies that serve as safety net and prevent unencrypted tunnel packets
   to leave the host in case ovs-vswitchd configured datapath before
   ovs-monitor-ipsec installed IPsec policies.
   However, assumption here is that OpenFlow controller was careful
   and installed OpenFlow rule with set SKB mark action specified in
   OVSDB Open_vSwitch table before the first packets were able to leave
   the OVS tunnel.

3) ovs-monitor-ipsec assumes that packets coming from all OVS tunnels
   by default need to be protected with IPsec unless the tunnel is explicitly
   allowed to leave unencrypted.  This is inverse behavior of the second
   mode::

     % ovs-vsctl set Open_vSwitch . other_config:default_ipsec_skb_mark=0/1
     % ovs-vsctl add-port br0 ipsec_gre0 -- \
                 set interface gre0 type=gre \
                                   options:remote_ip=1.2.3.4 \
                                   options:psk=swordfish])

   With this solution ovs-monitor-ipsec tells strongSwan
   to install IPsec shunt policies to match on skb mark 0
   which should be the default skb mark value for all tunnel packets
   going through NetFilter/XFRM hooks.  As a result IPsec assumes
   that all packets coming from tunnels should be encrypted unless
   OpenFlow controller explicitly set skb mark to a non-zero value.
   This is the most secure mode, but at the same time the most intrusive
   one, because the OpenFlow pipeline needs to be overhauled just because
   now IPsec is enabled.

   Note that instead of using least significant bit of SKB mark, you could
   just as well have used any other bit.


Troubleshooting
---------------

Use following ovs-apptcl command to get ovs-monitor-ipsec internal
representation of tunnel configuration::

    % ovs-appctl -t ovs-monitor-ipsec tunnels/show

If there is misconfiguration then ovs-appctl should indicate why.
For example::

   Interface name: gre0 v5 (CONFIGURED) <--- Should be set to CONFIGURED.
                                             Otherwise, error message will
                                             be provided
   Remote IP:      192.168.2.129
   Tunnel Type:    gre
   Local IP:       0.0.0.0
   Use SSL cert:   False
   My cert:        None
   My key:         None
   His cert:       None
   PSK:            swordfish
   Ofport:         1          <--- Whether ovs-vswitchd has assigned Ofport
                                   number to this Tunnel Port
   CFM state:      Up         <--- Whether CFM declared this tunnel healthy
   Kernel policies installed:
   ...                          <--- IPsec policies for this OVS tunnel in
                                     Linux Kernel installed by strongSwan
   Kernel security associations installed:
   ...                          <--- IPsec security associations for this OVS
                                     tunnel in Linux Kernel installed by
                                     strongswan
   Strongswan connections that are active:
   ...                          <--- strongSwan "connections" for this OVS
                                     tunnel


Bug Reporting
-------------

If you think you may have found a bug with security implications, like

1) IPsec protected tunnel accepted packets that came unencrypted; OR
2) IPsec protected tunnel allowed packets to leave unencrypted;

Then report such bugs according to `security guidelines
<Documentation/internals/security.rst>`__.

If bug does not have security implications, then report it according to
instructions in `<REPORTING-bugs.rst>`__.

There is also a possibility that there is a bug in strongSwan.  In that
case report it to strongSwan mailing list.