# **BFD PriorityQueue in Contrail**

## **1.	Introduction**

Implement priority queue for single-hop BFD

## **2.	Problem Statement**

In the current vrouter-Agent’s BFD (single-hop) design , there is no priority queue implemented for the BFD packets. All the proto packets received from the workload to the vRouter-Agent via Pkt0 interface are enqueued to a single queue and have the same priority across other protocol traffic (ARP, ICMP, DNS, etc).

As shown in Fig.1 and Fig.2., the various proto handlers in the PktHandler jobs have dependency on DB Table data structures and hence the PktHandler Task in Agent is run with mutual exclusivity with DB Task. In scale scenarios, if there happens to be big churn in DB Task events (e.g Route deletion, etc) causing the PktHandler Task to starve and its jobs are not run due to mutual exclusion with DB Task.  This is causing the BFD packets in the PktHandler Queue not being serviced resulting in BFD failures. Upon BFD failure, its interface is put in "inactive" state and no traffic is allowed during interface “inactive” state.


                                                +---------------------+
                                                 |     Agent           |
          +---------+       +-------------+      |   +-------------+   |
          |  VNF    |       |             |      |   |BFD          |   |
          |         |       |vRouter      |      |   |Proto Handler|   |
          |         |Tap I/F|[Kernel/DPDK]|PKT#0 I/F |             |   |
          |         |<----->|             |<-----|-->|             |   |
          |         |       |             |      |   |             |   |
          |         |       |             |      |   |             |   |
          |         |       |             |      |   |             |   |
          +---------+       +-------------+      |   +-------------+   |
                                                 +---------------------+
                Fig.1 BFD Packet Processing

+------------+    +-------------------------------------------------------+
|            |    |                           +--------------------+      |
|            |    |                           |     Proto Queues   |      |
|            |    |                           |  +-----------------|      |
|            |    |                           |  |  |              |      |
|            |    |                           |  |  |  ARP-        |      |
|            |    |                           |  |  |  Queue       |      |
|            |    |                           ^  |  |              |      |
|            |    |                          /|  +-----------------|      |
+ From       |    |                         / |  +-----------------|      |
| PKT0       |    |                        /  |  |  |              |      |
|            |    |  +-----------------+  /   |  |  |  ARP-Queue   |      |
+            |--->|  |  |  PktHandler  |/---->|  |  |              |      |
|            |    |  |  |  Queue       |----->|  +-----------------|      |
|            |    |  |  |              |---+  |   --   ---   ---   |      |
|            |    |  +-----------------+   |  |   --   ---   ---   |      |
|            |    |                        |  |  +-----------------|      |
|            |    |                        |  |  |  |              |      |
|            |    |                        |  |  |  |  BFDProto-   |      |
|            |    |                        |-->  |  |  Queue       |      |
|            |    |                           |  |  |              |      |
|            |    |                           |  +-----------------|      |
|            |    |                           +--------------------+      |
|Task: #ASIO |    |                   Task: #PktHandler                   |
+------------+    +-------------------------------------------------------+
			Fig.2 BFD Packet Processing in Agent

## **3. 	Proposed solution**

All the proto packets received by Agent via PKT0 interface are enqueued to the "PktHandler" Workqueue . The “PktHandler” Workqueue is scheduled by the “PktHandler” task which runs with mutual exclusion to “dB” Task. The mutual exclusion with “dB” task also contributed to the delay in scheduling the Task for processing of the oporto packets enqueued in “PktHandler”  workqueue.

In the proposed solution (show in Fig.3):

1. vRouter marks the BFD packets that are sent to vrouter_agent prior to sending on PKT0 interface

2. There will be two Task workqueues for the BFD packet processing

    1. BFD control packet workqueue and [a.k.a WorkQueue#1]

These packets are received when the BFD client session is Down (no session is

established). The workqueue jobs run with mutula exclusive to "dB “ task since its

processing has dependency with Db table.

    2. BFD keepalive packet workqueue  [a.k.a. WorkQueue#2]

BFD packets received when the BFD client session is up. The workqueue job has no dependency on other tasks, thereby being processed with priority.

3. Proto packet received in PKT0 interface that are marked BFD type are enqueued to one of the two BFD workqueues that are classified based on the type of the BFD packet.
+--------+   +------------------------------------------------+
|        |   |                         +--------------------+ |
|        |   |                         |     Proto Queues   | |
|        |   |                         |  +-----------------| |
|        |   |                         |  |  |              | |
|        |   |                         |  |  |  ARP-        | |
|        |   |                         |  |  |  Queue       | |
|        |   |                         ^  |  |              | |
|        |   |                        /|  +-----------------| |
|        |   |                       / |  +-----------------| |
|        |   |                      /  |  |  |              | |
|        |   |+-----------------+  /   |  |  |  ARP-        | |
+        |   ||  |              | /    |  |  |  Queue       | |
|        |--->|  |  PktHandler  |/---->|  |  |              | |
| From   |   ||  |  Queue       |----->|  +-----------------| |
| PKT0   |   ||  |              |---+  |   --   ---   ---   | |
+        |   |+-----------------+   |  |   --   ---   ---   | |
|        |   |                      |  |  +-----------------| |---------+
|        |   |                      |  |  |  |              | |         |
|        |   |                      |  |  |  |  BFD Init    | |  +------v---+
|        |   |                      |-->  |  |  PktsQueue   | | Task: #BFDProto
|        |   |                         |  |  |              | |  |          |
|        |   |                         |  +-----------------| |  |          |
|        |   |                         +--------------------+ |  |          |
|        |   |                 Task: #PktHandler              |  |          |
|        |   +------------------------------------------------+  |          |
|        |   +------------------------------------------------+  |          |
|        |   |              +-----------------+               |  |          |
|        |   |              |  |              |  Task: #BFDKA |  |          |
|        |   |              |  |  BFD         |               |->|          |
|        |   |              |  |  KeepAlive   |               |  |          |
Task: #ASIO  |              |  |  Queue       |               |  |          |
+--------+   |              +-----------------+               |  |          |
             +------------------------------------------------+  +----------+
                       Fig.3 BFD Init and KA Packet Processing in Agent

## **3.1 Alternatives considered**

To implement BFD protocol in vrouter instead of Agent.

## **3.2 ****API Schema changes**

N/A

## **3.3 ****UI changes / User workflow impact**

N/A

## **3.4	****Notification impact**

N/A

## **4.	****Implementation**

### 4.1 Vrouter

Vrouter will mark the BFD packets in the Agent Header prior to sending to PKT0 interface.

### 4.2 vrouter-Agent

### 4.3 vRouter-Agent Introspect changes

## **5.	****Performance and scaling impact**

None

## **6.	****Testing**

Unit Tests, Dev Testing and system tests are described in Test Plan Document: [CEM-16291 BFD Priority Queue Test Plan.xlsx](https://drive.google.com/file/d/1UnS_HRRFuauoHJxUVQGx4J7DwLkb-y9J/view?usp=sharing)

6.1 Unit Tests

6.2 Dev Tests

6.3 System Tests

## **7.	****Upgrade**

None

## **8.	****Deprecation**

None

## **9.	****Dependencies**

None

## **10.	****Documentation impact**

None

## **11.	****References**

JIRA story : [CEM-16291](https://contrail-jws.atlassian.net/browse/CEM-16291)

Test Plan Document: [CEM-16291 BFD Priority Queue Test Plan.xlsx](https://drive.google.com/file/d/1UnS_HRRFuauoHJxUVQGx4J7DwLkb-y9J/view?usp=sharing)

BFD RFC: [BFD_RFC5880](https://tools.ietf.org/html/rfc5880)

