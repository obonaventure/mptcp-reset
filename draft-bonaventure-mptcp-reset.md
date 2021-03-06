-----
title: Processing of RST segments by Multipath TCP
abbrev: MPTCP RST
docname: draft-bonaventure-mptcp-rst-00
date: 2014-07-03
category: exp

ipr: trust200902
area: Transport
workgroup: MPTCP 
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
  ins: O. Bonaventure
  name: Olivier Bonaventure
  organization: UCLouvain
  email: Olivier.Bonaventure@uclouvain.be
 -
  ins: C. Paasch
  name: Christoph Paasch
  organization: UCLouvain
  email: Christoph.Paasch@uclouvain.be
 -
  ins: G. Detal
  name: Gregory Detal
  organization: UCLouvain
  email: Gregory.Detal@uclouvain.be

normative:
 RFC2119:
 RFC0793:
 RFC6824:
 RFC3360:
 RFC1122:
 RFC5961:
 RFC6528:

informative:
  IMC11:
    author:
      - ins: M. Honda
      - ins: Y. Nishida
      - ins: C. Raiciu
      - ins: A. Greenhalgh
      - ins: M. Handley
      - ins: H. Tokuda
    title: Is it still possible to extend TCP ?
    seriesinfo: Proceedings of the 2011 ACM SIGCOMM conference on Internet measurement conference (IMC '11)
    date: 2011
    target: http://doi.acm.org/10.1145/2068816.2068834 
  NDSS09:
   author:
     - ins: N. Weaver
     - ins: R. Sommer
     - ins: V. Paxson
   title: Detecting Forged TCP Reset Packets 
   seriesinfo: NDSS2009 
   date: 2009
  INFOCOM14:
    author: 
      - ins: Y. Lim
      - ins: Y. Chen
      - ins: E. Nahum
      - ins: D. Towsley
      - ins: K. Lee
    title: Cross-Layer Path Management in Multi-path Transport Protocol for Mobile Devices
    seriesinfo: IEEE INFOCOM'14
    date: 2014



--- abstract


This document discusses how a Multipath TCP implementation should   generate and process RST segments.

--- middle

Introduction
=========

   The Transmission Control Protocol (TCP) {{RFC0793}} was designed to provide a
   reliable data transfer between hosts attached to a single IP address.
   Multipath TCP {{RFC6824}} is a recent extension that enables a single TCP
   connection to use multiple paths.  For Multipath TCP, each path
   corresponds to a TCP connection established between the communicating
   hosts.  Each of these connections is identified by the classical
   four-tuple (source and destination IP addresses and source and
   destination port numbers).  Multipath TCP allows to group several of
   these TCP connections, called subflows in {{RFC6824}} inside a single
   Multipath TCP connection.  It should be noted that the number of
   subflows that are used for a given Multipath TCP connection is not
   fixed and can change during the lifetime of a Multipath TCP
   connection.

   In regular TCP, the RST flag is used to abruptly terminate a TCP
   connection.  When a host receives a valid TCP segment with the RST
   flag, it immediately terminates the connection.  This causes the loss
   of all in-flight data that has not yet been acknowledged.  In the
   original TCP specification, a host could generate a segment with the
   RST flag in the following cases:

   -  A host receives a non-SYN segment that corresponds to a TCP
      connection that does not exist (anymore).

   -  A host receives a SYN segment and does not want to establish the
      requested connection for any reason.

   -  A host has tried to retransmit the same data too many times
      without having received an acknowledgment.

   -  The corresponding connection has been idle for a long time and no
      answer has been received to the keepalive segments that the host
      has sent over this TCP connection {{RFC1122}}

   -  A host does not have enough resources anymore (e.g. memory) to
      support the established connection.

   Over the years, other reasons to use the RST flag have been added to
   TCP implementations.  The most important one is the possibility for
   an application, typically a server, to abruptly terminate a TCP
   connection by forcing the stack to send a segment with the RST flag
   instead of waiting for the normal FIN exchange and being forced to
   maintain state.

   Despite the fact that TCP is a transport protocol that is used only
   on endhosts, various types of middleboxes are known to spoof TCP
   segments that contain the RST flag to abruptly terminate TCP
   connections {{IMC11}}. Some of these middleboxes terminate TCP connections to
   block some applications such as P2P file transfers, others provide
   security services such as DPI and terminate TCP connections once they
   have identified suspicious data in the payload.  This behavior of
   middleboxes has been considered as harmful in {{RFC3360}}.

   Multipath TCP cannot simply behave like regular TCP when transmitting
   and receiving TCP segments with the RST flag.  Since a Multipath TCP
   connection is composed of several TCP subflows, the transmission or
   reception of a TCP RST on a subflow only terminates the
   corresponding subflow.  This does not necessarily terminate the
   Multipath TCP connection.

   This document is organized as follows.  We discuss in section
   Section 2 the reasons why a Multipath TCP host could decide to
   transmit a RST segment.  In section Section 3, we propose a new
   Multipath TCP option that can be used inside RST segments to convey
   additional information about the reason for this RST segment and
   explain how a Multipath TCP host should react to such segments.

Notational Conventions
---------------------------

   The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
   "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this
   document are to be interpreted as described in {{RFC2119}}.

Generation of RST segments
======================

   This section documents the various reasons why a Multipath TCP host
   could generate a segment with the RST flag.

Lack of ressources
----------------------

   A Multipath TCP implementation cannot usually support an unlimited
   number of TCP subflows associated to a single Multipath TCP
   connection.  A Multipath TCP implementation may face a lack of
   ressources when :

   -  It receives a SYN segment and accepting this subflow would exhaust
      the subflow table of the host.  

   -  It experiences memory pressure and needs to recover additional
      ressources.  Under these circumstances, it could decide to
      terminate some subflows for existing Multipath TCP connections.  

Administratively prohibited
-------------------------------

   A Multipath TCP implementation can be configured with policies that
   determine which subflows can be associated to a given Multipath TCP
   connection.  These policies could be specified on a per application
   basis, a per host basis or via other means that are outside the scope
   of this document.  Based on this configuration, a Multipath TCP host could
   generate a RST segment to refuse the establishment of a subflow when
   :

   -  It receives a SYN segment from a source address that conflicts
      with its policies.  For example, an enterprise server could be
      configured to only accept subflows that originate from the
      enterprise subnet and not from the global Internet.  

   -  It receives a SYN segment with a destination address that
      conflicts with its policies.  For example, a server may be
      configured to only expose some of its addresses to a subset of its
      clients.  

   -  It receives a SYN segment destined to a destination port that
      conflicts with its policies.  For example, a web server may be
      configured to only accept subflows targeted to port 80 even if the
      Multipath TCP specification allows to group together subflows on
      different destination ports.  

   Both ICMPv4 and ICMPv6 contain error codes that can be used to
   indicate similar error conditions.  However, ICMP messages are more
   likely to be dropped by network equipment than TCP segments.
   Multipath TCP's reaction to these ICMP messages and the TCP RST
   segments should be similar.

Too many already acknowledged data
--------------------------------------------

   Multipath TCP allows data that was transmitted over one subflow to be
   retransmitted over another subflow.  This situation can happen during
   a handover when a mobile host moves from one access network to
   another.  In this case, the same data can be transmitted twice over
   different subflows.  This is mandated by Multipath TCP to ensure that
   any middlebox on the path of the first subflow will see in-sequence
   segments.  However, retransmitting already acknowledged data over a
   subflow is not the best utilisation of the network ressources as
   reported in {{INFOCOM14}}.  It should be possible for a Multipath TCP host that
   has too many data that has already been acknowledged over one subflow
   but still needs to be retransmitted over another subflow to preserve
   the subflow ordering by terminating the subflow with too many
   outstanding but already acknowledged data.  Measurements or
   simulations are required to evaluate the best threshold to be used by
   a Multipath TCP implementation to decide when to terminate such a
   subflow.  In this document, we propose to terminate a subflow once
   there are more than one initial congestion window's worth of data
   that are outstanding on this subflow but have already been
   acknowledged over another subflow or there is no other data on this
   subflow.

   The situation described above is obviously transient.  The
   termination of such a subflow does not indicate that the path should
   not be used anymore.  Instead, the reception of a RST segment
   indicating such a cause should trigger the reestablishment of a
   subflow over this path.  The host that sends such a RST segment could
   also send a SYN segment at the same time.  However, it should be
   noted that there are situations such as a server sending the RST
   segment to a client connected behind a NAT where only the host that
   receives the RST segment is able to reestablish the subflow.

Unacceptable performance
--------------------------------

   A Multipath TCP host can send data over several subflows.  Some of
   these subflows may perform well while the performance of others could
   be affected by various performance problems :

   -  Excessive delay compared to the other subflows.  If the receive
      window used by Multipath TCP is too small, sending data over a
      long delay subflow would reduce the overall performance of the
      Multipath TCP connection.  

   -  Excessive losses.  The Multipath TCP congestion control scheme
      tries to move traffic away from congested paths.  If one of the
      subflows is more heavily congested than the others, this can
      severely impact the performance of the Multipath TCP connection.
      
   - Excessive reordering.  Excessive reordering at the subflow level
      may lower performance by making TCP's retransmission techniques
      less reactive.  This error condition is typically transient.

   A Multipath TCP host that detects that the performance of a Multipath
   TCP connection is severely affected by one of the underlying
   subflows, could decide to terminate the offending subflow.
   Depending on the number of remaining active subflows, it may be
   needed to reestablish another subflow to replace the terminated one.

Lifetime expired
------------------

   A Multipath TCP connection is composed of several subflows.  However,
   maintaining a large number of subflows can be costly from an
   implementation viewpoint.  A Multipath TCP host should monitor the
   usage of the underlying subflows and could terminate one subflow when
   :

   -  No reply has been received to keepalive probes.  The keepalive
      probes {{RFC1122}} can be used over each subflow to verify that their
      paths and the remote host are still active.  If no answer is
      received to these probes, the corresponding subflow should be
      terminated.  The Multipath TCP connection could be terminated once
      the last subflow is terminated.  Note that sending a regular RST
      over each subflow will only terminate the subflow but not the
      Multipath TCP connection {{RFC6824}}.


Removed address
---------------------

   When a host receives a REMOVE_ADDR option, it should send a TCP
   keepalive over each of the subflows using the removed address.  If a
   response to the keepalive is received, the subflow should not be
   terminated.  Otherwise, the lack of response to the keepalives will
   trigger a termination of the subflow as explained in section
   Section 2.5.

   When a host sends a REMOVE_ADDR option, it SHOULD send a RST segment
   over each of the subflows that were using the removed address.

Middlebox interference has been detected
-------------------------------------------------

   As explained in {{RFC6824}}, Multipath TCP includes several mechanisms to
   detect and possibly cope with middlebox interference.  There are
   unfortunately cases where Multipath TCP needs to terminate a subflow
   once it has detected middlebox interference.  The following cases are
   listed in {{RFC6824}} :

   o  As explained on page 42 of {{RFC6824}}, a host MUST close a subflow with a
      RST if the first ACK that it receives over this subflow does not
      contain the DSS option.

   o  As explained on page 43 of {{RFC6824}} a host MUST close a subflow by
      sending a RST segment with the MP_FAIL option if it receives a
      segment with an invalid DSS checksum.  The MP_FAIL option includes
      the data sequence number of the first byte of the payload of the
      affected segment.

Multipath TCP specific errors
----------------------------------

   {{RFC6824}} lists several error conditions that are specific to Multipath TCP
   and may lead to the termination of a subflow by transmitting a RST
   segment.  These error conditions are :

   - A SYN segment with the MP_JOIN option was received with an invalid
      HMAC or an unknown token.  In this case, the host may reply with a
      RST or silently ignore the error ({{RFC6824}} pages 22, 23 and 45).

   -  A TCP segment is received without a DSS checksum on a Multipath
      TCP connection where the usage of the checksum has been negotiated
      ({{RFC6824}} page 24).

   -  No DSS mapping has been received within a window of data ({{RFC6824}} page
      27).

Fast Close
-------------

   The MP_FASTCLOSE option is defined in {{RFC6824}} allows to quickly terminate
   a Multipath TCP connection.  This operation is described in section
   3.5 of {{RFC6824}}.

Unspecified TCP error
-------------------------

   A TCP implementation may send a RST segment for reasons that are
   unrelated to Multipath TCP.

The MPTCP RST option
===================

   The Multipath TCP specification {{RFC6824}} defines several Multipath TCP
   options.  Each option is encoded by using the Type-Length-Value
   format shown in the figure below.

~~~~~~~~~~~~~~~~~~~~~~

                        1                   2                   3
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
   +---------------+---------------+-------+-----------------------+
   |     Kind      |    Length     |Subtype|                       |
   +---------------+---------------+-------+                       |
   |                     Subtype-specific data                     |
   |                       (variable length)                       |
   +---------------------------------------------------------------+

~~~~~~~~~~~~~~~~~~~~~~
{: #figmptcpoption title="The Multipath TCP option format"}

  

   The proposed format for the Multipath TCP RST option is defined in
   the figure below.

~~~~~~~~~~~~~~~~~~~~

                      1                   2                   3
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 +---------------+---------------+-------+-----------------------+
 |     Kind      |    Length     |Subtype|U V W T|    Reason     |
 +---------------+---------------+-------+-----------------------+


~~~~~~~~~~~~~~~~~~~~
{: #figmptcprstoption title="The proposed Multipath TCP RST option format"}


   The Multipath TCP RST option contains a reason code that allows the
   sender of the option to provide more information about the reason for
   the termination of the subflow. {{RFC0793}} allowed the utilisation of the
   segment payload to provide additional information about the reason
   for the termination of a TCP connection and some middleboxes have
   used this facility {{NDSS09}}.  However, without a precise format for the
   reason code, the only thing that TCP implementations could do with
   this payload was to log the received data.  With a specific reason
   field, it becomes possible for a Multipath TCP implementation to
   intelligently and correctly react to the termination of a subflow.

   The `T` flag is used by the sender to indicate whether the error
   condition that is reported is Transient (T bit set to 1) or Permanent
   (T bit set to 0).  If the error condition is considered to be
   Transient by the sender of the RST segment, the recipient of this
   segment MAY try to reestablish a subflow over the failed path.  If
   the error condition is considered to be permanent, the receiver of
   the RST segment SHOULD NOT try to reestablish a subflow over this
   path. The `U`, `V` and `W` flags are not defined by this specification.

   The `Reason` code is an 8 bits field that indicates the reason for
   the termination of the subflow.  The following codes are defined in
   this document :

   -  Lack of ressources (code TBD1).  This code indicates that the
      sending host does not have enough ressources to support the
      terminated subflow.  

   -  Administratively prohibited (code TBD2).  This code indicates that
      the requested subflow is prohibited by the policies of the sending
      host.  

   -  Too many already acknowledged data (code TBD3).  This code
      indicates that there are too many data that need to be transmitted
      over the terminated subflow while having already been acknowledged
      over one or more other subflows.  

   -  Unacceptable performance (code TBD4).  This code indicates that
      the performance of this subflow was too low compared to the other
      subflows of this Multipath TCP connection.  

   -  Lifetime expired (code TBD5).  This code indicates that the
      lifetime of the subflow has expired.  

   -  Removed address (code TBD6).  This code indicates that the address
      associated to this subflow has been removed by the sender.  

   -  Middlebox interference (code TBD7).  Middlebox interference has
      been detected over this subflow.  

   -  Multipath TCP specific error (code TBD8).  An error has been
      detected in the processing of Multipath TCP options.  

   -  Fast Close (code TBD9).  This RST segment has been sent in
      response to a segment with the MP_FASTCLOSE option.  

   -  Unspecified TCP error (code TBD10).  An unspecified TCP error has
      been detected on the affected subflow.  


5.  IANA Considerations

   This document request the allocation of a new MPTCP option sub-type
   from IANA.  Furthermore, it defines a set of error conditions that
   can be encoded inside the MPTCP RST option.  This list of error
   conditions should be maintained by IANA.


 | Codepoint # |              Reason             |
 |-------------+---------------------------------|
 |     TBD1    |        Lack of ressources       |
 |     TBD2    |   Administratively prohibited   |
 |     TBD3    |   Too many unacknowledged data  |
 |     TBD4    |     Unacceptable performance    |
 |     TBD5    |         Lifetime expired        |
 |     TBD6    |         Removed address         |
 |     TBD7    | Middlebox interference detected |
 |     TBD8    |   Multipath TCP specific error  |
 |     TBD9    |      Response to Fast Close     |
 |    TBD10    |      Unspecified TCP error      |

{: #table title="Suggested MPTCP RST codepoints"}


Security Considerations
================


Single TCP is vulnerable to various forms of attacks that use RST segments. An off-path attacker could send spoofed RST segments to terminate existing TCP connections. Several techniques have been proposed to deal with such attacks {{RFC6528}} {{RFC5961}}. These techniques can also be used with Multipath TCP. The utilization of the proposed MPTCP RST option does not change anything to the applicability of these attack mitigation techniques. Since Multipath TCP supports break before make, it is important to note that a successful RST attack does not result in a release of the Multipath TCP connection. A host can decide to initiate a new subflow, over the same or another path, upon reception of a RST segment.

An on-path middlebox may generate RST segments to terminate some unwanted TCP connections {{NDSS09}} {{RFC3360}}. The attack mitigation techniques proposed in {{RFC6528}} and {{RFC5961}} are not suitable to defend against on-path attackers like middleboxes. As noted above, a host that receive a valid RST segment could still react by establishing another subflow, possibly over another path. The presence of the proposed RST option in the RST segment does not change these security considerations. 

Conclusion
=========


   This document has analyzed the various reasons that may cause a Multipath TCP implementation to generate a RST segment. Since a Multipath TCP connection can combine several TCP subflows, the termination of one subflow does not necessarily lead to the termination of the entire Multipath TCP connection. We propose the Multipath TCP RST option to convey additional information about the reason that motivated the transmission of the RST segment.

Acknowledgements
===============

   This work was partially supported by the FP7 Trilogy2 project.
