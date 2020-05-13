# 1. Introduction
This blue print describes the implementation of vrinfo framework which can work across DPDK and kernel to provide a way to dump internal structures of vRouter and send it via Sandesh.

In this document, we will discuss about the dpdkinfo tool which reads the internal data structures and unstructured data from vRouter and DPDK library and display on the console.

# 2. Problem statement
Currently, we dont have tools to dump the below information for DPDK
- Bond Info - Information displayed in linux as part of "/proc/net/bonding/bond0".
- LACP Info - LACP configuration, statistics etc.
- Mempool info - DPDK mbuf utilization for different mempools
- lcore to interface/queue info - Informatoin about lcore mapping to interfaces.
- NIC Info - Inforamtion about NIC tx/rx counters and errors.

# 3. Proposed solution

1. Need to develop a generic infrastructure which can work across DPDK and kernel to provide a way to dump internal state of the vrouter and send it to a CLI using sandesh messaging.

2. The sandesh limitation is 4KB. But the info that needs to be dumped can be more than 4KB. So this infra needs to have an ability to send it in chunks.

3. The infra should able to support multiple CLI clients in parallel.

4. The infra needs to provide an easy-to-use API interface for the users who needs to add more functionality.

   ## 3.2 Alternatives considered

   None

   ## 3.3 API schema changes

   Not Applicable

   ## 3.3 User workflow impact

   None

   ## 3.4 UI changes

   None

   ## 3.5 Notification impact

   None

# 4. Implementation
## 4.1 vrinfo Workflow Diagram

![dpdkinfo_1](./images/dpdkinfo_1.png)

### Working Mechanism:

<u>Clients:</u>
		Clients are command line utilities used to connect with vrouter(server) and displays the messages on console. Vrinfo framework can able to handle multiple CLI commands(eg: dpdkinfo) in parallel and provide output.

<u>Dp-core:</u>
		DP-core is part of vRouter module where it handles the platform independent part. In this case, it receives the client request and process it and call corresponding callback function.

<u>Msg. buffer table:</u>
		Sandesh has limitation of sending 4K bytes in one interation,  so message buffer table is used to store the output buffers and send it in chunks.

<u>Users Callback table:</u>
		When user register for a callback function, it would get registered in the callback table. Users have to register thier callback function in vr_info.h file. The registered callback can either be in kernel or DPDK.

<u>Sample Code:</u>

```
#define VR_INFO_REG(X) \
			X(INFO_VER,  info_get_version, KERNEL) \
			X(INFO_BOND, info_get_bond,    DPDK) \
```

​	Callback API for DPDK should start with dpdk\_\<registered fn.name> and for kernel lh\_\<registered fn. name>. Eg: dpdk_info_get_bond & lh_info_get_version.

​	From above diagram, Client CLI request for a particular message from dp-core. For eg: Requetsting bond information. So, Dp-core will search that information in the registered users callback table and call corresponding function and retreive message and display on the console.

<u>From Client CLI – DPcore</u>:

​	A Client CLI(dpdkinfo) would be developed to query data from DPDK through Sandesh protocol. Once the request has been received by vrouter(dp-core), it will create an entry in Msg. buffer table. To support multiple CLI clients in parallel, we need to store in its corresponding index in msg. buffer table.

​	Assumption: Message buffer table size has fixed to 64 because vRouter(server) can able to process maximum of 64 clients in parallel.

<u>From DPCore – DPDK:</u>

​	Once the message received in dp-core, it will look into users callback table and call corresponding callback function with below arguments.

```
/* vr_info callback function arguments.
 * msg_req->inbuf      => vr_info provide some input to callback function(kind of filter).
 *              Eg: Incase for bondinfo, CLI wants to show a particular slave.
 * msg_req->inbuf_len  => Buffer length
 * msg_req->outbuf     => Callback function should allocate memory buffer and fill contents.
 * msg_req->outbuf_len => Callback function should provide Output buffer length.
 * msg_req->bufsz      => Optional: Send output buffer size from CLI
 * */
#define VR_INFO_ARGS vr_info_t *msg_req
```

<u>From DPDK – DPCore:</u>

​	Vrinfo provides VI_PRINTF macro, so DPDK should use this macro to fill the message buffer contents which will get dispayed in CLI. Once the message has been completed, vrinfo reads the contents from "msg_req->outbuf" pointer and display on the console.

```
#define VI_PRINTF(...) \
{ \
    len = snprintf((msg_req->outbuf + msg_req->outbuf_len), \
            (msg_req->bufsz - msg_req->outbuf_len), __VA_ARGS__ ); \
    if(len < 0) {  \
        vr_printf("VrInfo: snprintf - Message copy failed at %d\n", \
                msg_req->outbuf_len); \
        return VR_INFO_FAILED; \
    } \
    if (len > (msg_req->bufsz - msg_req->outbuf_len)) { \
            vr_printf("VrInfo: Message copy to buffer failed at %d\n", \
                    msg_req->outbuf_len); \
            return VR_INFO_FAILED; \
    } \
    msg_req->outbuf_len += len; \
}
```

 <u>From DPCore – CLI:</u>

​            Sandesh has a limitation of sending only 4K size, so we have created a Message buffer table. So, If the message buffer size is more than 4K, vRouter(DP-core) will split the data and send it serially to Client CLI. Once all the message has been sent to Clinet, vRouter(Dp-core) will free the buffer.

### 4.1.1 Vrinfo sandesh message

Below sandesh message have been newly added to process the CLI requests.

```
buffer sandesh vr_info_req {
    1:  sandesh_op      h_op;
    2:  i16             vdu_rid;
    3:  i16             vdu_index;
    4:  i16             vdu_buff_table_id;
    5:  i16             vdu_marker;
    6:  i16             vdu_msginfo;
    7:  i32             vdu_outbufsz;
    8:  list <byte>     vdu_inbuf;
    9:  list <byte>     vdu_proc_info;
}
```

### 4.1.2 Newly added CLI

Dpdkinfo CLI has been added inside dpdk container to support the following commands.

```
Usage: dpdkinfo [--help]
                 --version|-v               																		Show DPDK Version
                 --bond|-b                  																		Show Master/Slave bond information
                 --lacp|-l     <all/conf>   																		Show LACP information from DPDK
                 --mempool|-m  <all/<mempool-name>> 														Show Mempool information
                 --stats|-n    <eth>                														Show Stats information
                 --xstats|-x   < =all/ =0(Master)/ =1(Slave(0))/ =2(Slave(1))>  Show Extended Stats information
                 --lcore|-c                                                     Show Lcore information
                 --app|-a                                                       Show App information
       Optional: --buffsz      <value>                                          Send output buffer size
```

### 4.1.3 CLI Outputs

Following section describes each of dpdkinfo command options in detail.

<u>BondInfo:</u>

- Command: **dpdkinfo --bond**
- Display the bond information for master and slaves.

```
DPDK Version: DPDK 18.05.1
No. of bond slaves: 2
Bonding Mode: 802.3AD Dynamic Link Aggregation
Transmit Hash Policy: Layer 3+4 (IP Addresses + UDP Ports) transmit load balancing
MII status: UP
MII Link Speed: 20000
MII Polling Interval (ms): 10
Up Delay (ms): 0
Down Delay (ms): 0

802.3ad info :
LACP Rate: slow
Aggregator selection policy (ad_select): Stable
System priority: 65535
System MAC address:90:e2:ba:54:6b:ac
Active Aggregator Info:
	Aggregator ID: 0
	Number of ports: 2
	Actor Key: 8448
	Partner Key: 3328
	Partner Mac Address: 88:d9:8f:d2:bc:a0

Slave Interface(0): 0000:04:00.0
Slave Interface Driver: net_bonding
MII status: UP
MII Link Speed: 20000
MII Polling Interval (ms): 10
Permanent HW addr:90:e2:ba:54:6b:ac
Aggregator ID: 0
Duplex: half
802.3ad info
LACP Rate: slow
Bond MAC addr:90:e2:ba:54:6b:ac
Details actor lacp pdu:
	system priority: 65535
	system mac address:90:e2:ba:54:6b:ac
	port key: 8448
	port priority: 65280
	port number: 256
	port state: 61
Details partner lacp pdu:
	system priority: 32512
	system mac address:88:d9:8f:d2:bc:a0
	port key: 3328
	port priority: 32512
	port number: 10496
	port state: 63

Slave Interface(1): 0000:04:00.1
Slave Interface Driver: net_bonding
MII status: UP
MII Link Speed: 20000
MII Polling Interval (ms): 10
Permanent HW addr:90:e2:ba:54:6b:ad
Aggregator ID: 0
Duplex: half
802.3ad info
LACP Rate: slow
Bond MAC addr:90:e2:ba:54:6b:ac
Details actor lacp pdu:
	system priority: 65535
	system mac address:90:e2:ba:54:6b:ad
	port key: 8448
	port priority: 65280
	port number: 512
	port state: 61
Details partner lacp pdu:
	system priority: 32512
	system mac address:88:d9:8f:d2:bc:a0
	port key: 3328
	port priority: 32512
	port number: 9472
	port state: 63
```

<u>LACPInfo:</u>

- Command: **dpdkinfo --lacp all**.
- Display lacp configuration for slow and fast timers and port details for actor and partner state.

```
LACP Rate: slow

Fast periodic (ms): 900
Slow periodic (ms): 29000
Short timeout (ms): 3000
Long timeout (ms): 90000
Aggregate wait timeout (ms): 2000
Tx period (ms): 500
Update timeout (ms): 100
Rx marker period (ms): 2000

Slave Interface(0): 0000:04:00.0
Details actor lacp pdu:
	port state: 61
Details partner lacp pdu:
	port state: 63

Slave Interface(1): 0000:04:00.1
Details actor lacp pdu:
	port state: 61
Details partner lacp pdu:
	port state: 63
```

<u>Mempool Info:</u>

- Command: **dpdkinfo --mempool all**
- Show summary of used available mbuf's from all mempools.
  Individual mempool information can be queried by specifying the mempool name eg: "dpdkinfo --mempool rss_mempool"

```
----------------------------------------------------
Name			Size	Used	Available
----------------------------------------------------
rss_mempool         	16384	2435	13949
frag_direct_mempool 	4096	0	4096
frag_indirect_mempool	4096	0	4096
slave_port0_pool    	8193	266	7927
slave_port1_pool    	8193	330	7863
packet_mbuf_pool    	8192	131	8061
```

<u>NIC Info:</u>

- NIC info will display packet statistics and extended packet statistics.
- Command: **dpdkinfo --stats eth**

```
Master Info:
RX Device Packets:64259, Bytes:7578680, Errors:0, Nombufs:0
Dropped RX Packets:0
TX Device Packets:330433, Bytes:39739287, Errors:0
Queue Rx: [0]64259
      Tx: [0]330433
      Rx Bytes: [0]7578680
      Tx Bytes: [0]39566125
      Errors:
------------------------------------------------------------
Slave Info(0000:04:00.0):
RX Device Packets:26697, Bytes:3449853, Errors:0, Nombufs:0
Dropped RX Packets:0
TX Device Packets:147100, Bytes:16700441, Errors:0
Queue Rx: [0]26697
      Tx: [0]147100
      Rx Bytes: [0]3449853
      Tx Bytes: [0]16527329
      Errors:
------------------------------------------------------------
Slave Info(0000:04:00.1):
RX Device Packets:37562, Bytes:4128827, Errors:0, Nombufs:0
Dropped RX Packets:0
TX Device Packets:183333, Bytes:23038846, Errors:0
Queue Rx: [0]37562
      Tx: [0]183333
      Rx Bytes: [0]4128827
      Tx Bytes: [0]23038796
      Errors:
------------------------------------------------------------
```

<u>Extended NIC statistics:</u>

- Command: **dpdkinfo --xstats all**
- Individual interfaces can be querid by specying the interface id. ( Master->0, Slave_0->1, Slave_1 ->2 )

```
Master Info:
Rx Packets:
	rx_good_packets: 64319
	rx_q0packets: 64319
Tx Packets:
	tx_good_packets: 330768
	tx_q0packets: 330768
Rx Bytes:
	rx_good_bytes: 7584973
	rx_q0bytes: 7584973
Tx Bytes:
	tx_good_bytes: 39781099
	tx_q0bytes: 39607797
Errors:
Others:
----------------------------------------------------------------------


Slave Info(0):0000:04:00.0
Rx Packets:
	rx_good_packets: 26726
	rx_q0packets: 26726
	rx_size_64_packets: 4
	rx_size_65_to_127_packets: 18927
	rx_size_128_to_255_packets: 3608
	rx_size_256_to_511_packets: 3809
	rx_size_512_to_1023_packets: 116
	rx_size_1024_to_max_packets: 262
	rx_multicast_packets: 7363
	rx_total_packets: 26726
Tx Packets:
	tx_good_packets: 147263
	tx_q0packets: 147263
	tx_total_packets: 147263
	tx_size_64_packets: 12382
	tx_size_65_to_127_packets: 17980
	tx_size_128_to_255_packets: 116714
	tx_size_256_to_511_packets: 26
	tx_size_512_to_1023_packets: 12
	tx_size_1024_to_max_packets: 149
	tx_multicast_packets: 116703
	tx_broadcast_packets: 718
Rx Bytes:
	rx_good_bytes: 3452816
	rx_q0bytes: 3452816
	rx_total_bytes: 3452816
Tx Bytes:
	tx_good_bytes: 16717649
	tx_q0bytes: 16544397
Errors:
Others:
	out_pkts_untagged: 147263
----------------------------------------------------------------------


Slave Info(1):0000:04:00.1
Rx Packets:
	rx_good_packets: 37593
	rx_q0packets: 37593
	rx_size_64_packets: 1506
	rx_size_65_to_127_packets: 28384
	rx_size_128_to_255_packets: 3609
	rx_size_256_to_511_packets: 3802
	rx_size_512_to_1023_packets: 14
	rx_size_1024_to_max_packets: 278
	rx_broadcast_packets: 700
	rx_multicast_packets: 7357
	rx_total_packets: 37593
Tx Packets:
	tx_good_packets: 183505
	tx_q0packets: 183505
	tx_total_packets: 183505
	tx_size_64_packets: 7
	tx_size_65_to_127_packets: 31526
	tx_size_128_to_255_packets: 151822
	tx_size_256_to_511_packets: 16
	tx_size_512_to_1023_packets: 12
	tx_size_1024_to_max_packets: 122
	tx_multicast_packets: 116702
	tx_broadcast_packets: 3
Rx Bytes:
	rx_good_bytes: 4132157
	rx_q0bytes: 4132157
	rx_total_bytes: 4132157
Tx Bytes:
	tx_good_bytes: 23063450
	tx_q0bytes: 23063400
Errors:
	mac_remote_errors: 4
Others:
	out_pkts_untagged: 183505
----------------------------------------------------------------------
```

<u>Lcore Info:</u>

- Command: **dpdkinfo --lcore all**
- Display the Rx queue interface mapping with forwarding scores.
  Individual lcore information can be queried by specying lcore id. Eg: dpdkinfo --lcore\<lcore-id\>

```
No. of forwarding lcores: 8 
No. of interfaces: 11 
Lcore 0: 
	Interface: bond0.2050          Queue ID: 0 
	Interface: vhost0              Queue ID: 0 
	Interface: tap7840d00d-b0      Queue ID: 0
  Interface: tap5b2f8f70-2e      Queue ID: 1 

Lcore 1: 
	Interface: bond0.2050          Queue ID: 0 
	Interface: tap97379553-5e      Queue ID: 0 

Lcore 2: 
	Interface: bond0.2050          Queue ID: 0 
	Interface: tapb91fc32a-5a      Queue ID: 0
  Interface: tap97379553-5e      Queue ID: 1 

Lcore 3: 
	Interface: bond0.2050          Queue ID: 0 
	Interface: tapa8f7009c-b9      Queue ID: 0 

Lcore 4: 
	Interface: bond0.2050          Queue ID: 0 
	Interface: tapa24f889d-ee      Queue ID: 0 

Lcore 5: 
	Interface: bond0.2050          Queue ID: 0 
	Interface: tap56665616-af      Queue ID: 0 

Lcore 6: 
	Interface: bond0.2050          Queue ID: 0 
	Interface: tape3c5d645-a2      Queue ID: 0 

Lcore 7: 
	Interface: bond0.2050          Queue ID: 0 
	Interface: tap5b2f8f70-2e      Queue ID: 0 

```

<u>App Info:</u>

- Command: **dpdkinfo --app**
- Display the system overall information like actual physical interface name, number of cores, vlan and queues etc.

```No. of cores: 18 
No. of cores: 18 
No. of forwarding lcores: 8 
Fabric interface: bond0.2050
Slave interface(0): enp4s0f0 
Slave interface(1): enp4s0f1 
Vlan name: bond0 
Vlan tag: 2050 
Vlan vif: bond0 
Ethdev (Master):
	Max rx queues: 128
	Max tx queues: 64
	Ethdev nb rx queues: 8
	Ethdev nb tx queues: 64
	Ethdev nb rss queues: 8
	Ethdev reta size: 128
	Ethdev port id: 2
	Ethdev nb slaves: 2 
	Ethdev slaves: 0 1 0 0 0 0 

Ethdev (Slave 0): 0000:04:00.0
	Nb rx queues: 8
	Nb tx queues: 64
	Ethdev reta size: 128

Ethdev (Slave 1): 0000:04:00.1
	Nb rx queues: 8
	Nb tx queues: 64
	Ethdev reta size: 128

Tapdev:
	fd: 63	vif name: bond0 
	fd: 72	vif name: vhost0 
```

# 5. Performance and scaling impact

None

    ## 5.1 API and control plane
None

    ## 5.2 Forwarding performance
None

# 6. Upgrade
None

# 7. Deprecations
None

# 8. Dependencies
None

# 9. Testing
## 9.1 Unit tests
## 9.2 Dev tests
## 9.3 System tests

a. Bond

    1. Setup a system with bond interface and verify the output
    2. Setup a system without bond interface and check output should not get displayed and display only error message.

b. LACP

    1. Setup a system with bond and configure fast lacp timers and check output.
    2. Setup a system without bond and check for error message.
    3. Repeat the same for different cli arguments(conf/status/all)

c. Mempool

    1. Configure system with DPDK and check mempool information.

d. xstats

    1. Verify xstats output for different NIC cards(X710, 82599).

e. Stats

    1. Check packet stats output and verify against with vif output for bond and non-bond cases.

f. lcore

    1. Configure multiple lcores and verify the output.

# 10. Documentation Impact

# 11. References
