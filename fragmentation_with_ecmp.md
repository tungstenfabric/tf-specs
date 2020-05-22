```
Fragmentation support - More cases
----------------------------------
This spec covers the following cases which is supported:

1) Packet mode + Fragmentation + ECMP destination
2) Packet mode + Fragmentation + ECMP destination + Mirror
3) Packet mode + Fragmentation + ECMP destination + Mirror ECMP

Background:
-----------
ATT discovered issues where large SIP packets were getting dropped. Refer 
jira ticket CEM-11118. After investigation, we found that the above three
cases were not supported in vRouter. This story is for supporting the same.

Code changes:
-------------
In case of packet mode to an ECMP destination, we need to compute the component
ECMP through which the packet needs to be sent. The way to achieve this is by
using hash of the 5-tuple in the packet. But, for fragments, we cannot have
the L4 port infomration. Hence, those packets need to be subjected to the 
fragment assembler. This was not happening. So, the fragmented packets were
getting dropped and an "Invalid Nexthop" counter was getting incremented.
With this story, we are now subjecting the fragments to the fragment assembler.

Another case is that of mirroring. In the earlier code, the fragment entries
in the fragment table were not deleted when the last fragment is received.
This was causing the fragment entries to be deleted during a "table reap" after
a particualr timeout. In this interval, if more such fragments arrive, the
fragment table was getting full and new fragments were getting dropped.

To address this problem, we have added another field to the fragment key, called
"custom" field. This will distinguish mirror packets from the original packets.
Using this approach, we can delete the fragment entry in the fragment table when
the last fragment arrives. This will then not cause the fragment table to fill-
up. So, vRouter can handle higher rate of fragmented packets.

Test cases:
-----------
Packet mode:
1   Fragmentation + ECMP + IPv6
2   Fragmentation + ECMP + IPv4
3   Fragmentation + ECMP + Mirror + IPv6
4   Fragmentation + ECMP + Mirror + IPv4
5   Fragmentation + ECMP + Mirror ECMP + IPv6
6   Fragmentation + ECMP + Mirror ECMP + IPv4
7   No Fragmentation + ECMP + IPv6  FT
8   No Fragmentation + ECMP + IPv4  FT
9   No Fragmentation + ECMP + Mirror + IPv6 FT
10  No Fragmentation + ECMP + Mirror + IPv4 FT

Flow mode:
1   Fragmentation + ECMP + IPv6
2   Fragmentation + ECMP + IPv4
3   Fragmentation + ECMP + Mirror + IPv6
4   Fragmentation + ECMP + Mirror + IPv4
5   Fragmentation + ECMP + Mirror ECMP + IPv6
6   Fragmentation + ECMP + Mirror ECMP + IPv4
7   No Fragmentation + ECMP + IPv6  FT
8   No Fragmentation + ECMP + IPv4  FT
9   No Fragmentation + ECMP + Mirror + IPv6 FT
10  No Fragmentation + ECMP + Mirror + IPv4 FT

UI impact:
----------
None

Schema changes:
--------------
None
```
