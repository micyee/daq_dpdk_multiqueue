# --- daq-2.2.1_dpdk16.07_mq.patch ---

This is a patch for adding DPDK 16.07 multi-queue support for DAQ 2.2.1 to use
with Snort 3.0 with multi threading support.

## Why this module?
We needed a DAQ module that could use DPDK and exploit the Snort 3.0's multi thread capability.
Furthermore, we needed full DPDK RSS support to fully avoid locking contention in the DAQ module.
There is a good DPDK-DAQ for 2.9.8, however that was single threaded. http://seclists.org/snort/2016/q2/385
Took that DAQ 2.0.6 module patch for Snort 2.9.8 and modified it to Snort 3.0 alpha 4 with multi-queue +
DPDK RSS support.

So here it is.

## How to setup

1. Download DPDK 16.07 and build it

2. Download DAQ 2.2.1 and modify and build it as follows:
```bash
# tar zxf daq-2.2.1.tar.gz
# patch -t -p1 < daq-2.2.1_dpdk16.07_mq.patch
# cd daq-2.2.1
# autoconf
# automake
# ./ configure --with-dpdk-includes=<dpdk-16.07 dir>/x86_64-native-linuxapp-gcc/include \
--with-dpdk-libraries=<dpdk-16.07 dir>/x86_64-native-linuxapp-gcc/lib
# make install
```
3. Download Snort 3.0 and build/install it. - currently, only alpha 4 is available.


## How to use

Snort in inline mode (IPS) using 14 RSS queues and 14 cores on 1 interface pair:
```bash
# taskset -c 0-13 snort --daq dpdk --daq-var dpdk_argc="-n4" -i "dpdk0:dpdk1" -Q -z 14 --daq-var dpdk_queues=14
```

Snort in inline mode (IPS) using 2 RSS queues and 14 cores on 2 interface pairs:
```bash
# taskset -c 0-13 snort --daq dpdk --daq-var dpdk_argc="-n4" -i "dpdk0:dpdk1 dpdk2:dpdk3" -Q -z 14 --daq-var dpdk_queues=2
```

Snort in passive mode (IDS) using 14 RSS queues and 14 cores on 2 interfaces:
```bash
# taskset -c 0-13 snort --daq dpdk --daq-var dpdk_argc="-n4" -i "dpdk0 dpdk1" -z 14 --daq-var dpdk_queues=14
```

Snort in passive mode (IDS) using 2 RSS queues and 4 cores on 1 interfaces:
```bash
# taskset -c 0-13 snort --daq dpdk --daq-var dpdk_argc="-n4" -i "dpdk0" -z 4 --daq-var dpdk_queues=2
```

Note: taskset is used here to pinn the cores used by Snort to run only on NUMA 0. This is done do avoid QPI 
utilization. Run the cores on the same NUMA node as the NIC is sitting on.

a. Only one interface specifier is supported, but that may contain multiple interfaces and/or interface pairs.

b. When specifying inline mode (-Q) the interface specification must only contain pairs, separated by ":" 
   (dpdk0:dpdk1 dpdk2:dpdk3).
   
c. Each pair in inline mode is bidirectional, thus both directions are handled by each thread. This way Snort 
   keeps track of bi-directional protocols.
   
d. If more threads than interfaces/pairs is specified, then the number of threads are equally distributes over 
   the interfaces specified. If 1 queues is specified, then each queue will get multiple threads that 
   reads/transmits from/to it. In this case, locking contension must be expected to a certain degree.
   
e: To avoid locking contention, you can make use of the multi-queue (RSS) feature on the HW. This is done with 
   "--daq-var dpdk_queues=<number of queues>". Again the number of threads specified (-z option) is distributed 
   over the number of specifed interfaces, over the number of queues. First queue 0 is assigned to a thread 
   on all interfaces then queue 2 etc. until all queues has been assigned a thread. if more threads specified, 
   then it starts over with queue 0 again. this means that each queue in a multiple queue system may have multiple
   threads (with some degree of lock contention).
   
f. A good idea is to have the number of threads used equal an even number of interfaces times number of queues 
   (i.e. 2 interfaces and 2 queues = 4 or 8 or 12... threads). However, typically the number of queues should 
   be incresed with the number of threads, so that no locking contention occurs.
   
g. When using RSS (multiple queues) the HW hashing algorithm used (at least specified) is as follows:  
   ETH_RSS_IP | ETH_RSS_UDP | ETH_RSS_TCP This means that the key is generated by IP address UDP/TCP ports numbers.
   
h. Additionally an option "--daq-var debug" may be specified. This will print out various interesting information 
   about the Rx and Tx trafic and this is done per thread, so that the load balancing may be visualized.

## Performance
Xeon 2697 v3 - 14 core - Intel XL710 2 x 40G - bare metal testruns:
Xena transmitter 68 byte packets Eth+IP+UDP with incrementing IP addresses.
Packets decoded by Snort and passed. No explicit rules applied.

16 Gbps thoroughput (20.8 Gbps wire speed) 29.5 Mpps - using 14 cores for Snort and 14 queues in a RSS setup.

