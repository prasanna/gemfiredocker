# Instructions to run a two node gemfire cluster

* Downloads gemfire 9.1.0 tar.gz package from https://network.pivotal.io/products/pivotal-gemfire. You need to register/login to download. Download the tar.gz file in archive folder.
* docker build -t gemfire91 .
* Open tree tabs and start following three containers
* ```docker run --name locator1 --hostname=locator1 -v $(pwd)/data/:/data/ -it gemfire91 /bin/bash```
* ```docker run --name server1 --hostname=server1 -v $(pwd)/data/:/data/  --cap-add=NET_ADMIN -it gemfire91 /bin/bash```
* ```docker run --name server2 --hostname=server2 -v $(pwd)/data/:/data/ --cap-add=NET_ADMIN -it gemfire91 /bin/bash```
* In the locator container run following command
  + ```gfsh```
  + ```gfsh> start locator --name=locator1 --port=9009 --J=-Dgemfire.distributed-system-id=1 --mcast-port=0 --dir=/data/locator1```
* In server1 container run following command. MAKE SURE THAT SERVER START/STOP is done from server containers gfsh and not from locator gfsh. Else While starting the server it starts servers on locator node.
  + ```gfsh```
  + ```start server --name=server1 --mcast-port=0 --locators="172.17.0.2[9009]" --server-port=8085 --properties-file=/pivotal-gemfire-9.1.0/config/gemfire.properties --dir=/data/server1```
* In server2 container run following command
  + ```gfsh```
  + ```start server --name=server2 --mcast-port=0 --locators="172.17.0.2[9009]" --server-port=8085 --properties-file=/pivotal-gemfire-9.1.0/config/gemfire.properties  --dir=/data/server2```  
* In the locator container run following command
  + If you want separate disk store, Create disk store
     ```gfsh> create disk-store --name=PDX_TYPES --dir=pdx_types```
  + Set pdx configuration
     ```gfsh> configure pdx --disk-store=DEFAULT --read-serialized=true --auto-serializable-classes=com.gemfire.functions.*,com.gemfire.models.*```
  + ```gfsh> create region --name=Positions --type=PARTITION_PERSISTENT --total-num-buckets=7```
  + ```gfsh> create region --name=MarketPrices --type=PARTITION_PERSISTENT --total-num-buckets=7```
  + ```gfsh> create region --name=FxRates --type=PARTITION_PERSISTENT --total-num-buckets=7```
* In server1 container run following command to restart server1.
  + ```gfsh> connect --locator=172.17.0.2[9009]```
  + ```gfsh> stop server --name=server1```
  + ```gfsh> start server --name=server1 --mcast-port=0 --locators="172.17.0.2[9009]" --server-port=8085 --dir=/data/server1```
  + ```gfsh> connect --locator=172.17.0.2[9009]```
* In server2 container run following command to restart server2.
  + ```gfsh> connect --locator=172.17.0.2[9009]```
  + ```gfsh> stop server --name=server2```
  + ```gfsh> start server --name=server2 --mcast-port=0 --locators="172.17.0.2[9009]" --server-port=8085 --dir=/data/server2```
  + ```gfsh> connect --locator=172.17.0.2[9009]```  
  
  * To test network partition.
  * ssh into server2 and install and use tc to put network latency of 16 seconds.
    + ```docker exec -it server2 /bin/bash```
    + ```bash-4.4# apk add iproute2```
    + ```bash-4.4# tc qdisc add dev eth0 root netem delay 16000ms```
    + ```bash-4.4# tc -s qdisc```
    * Wait for some time. Monitor server2 and locator1 logs. Locator will not receive heart beat 
    * and will remove server2 from view, server2 will shutdown but will have reconnect thread running.
    * Now remove network latency by following command
    + ```bash-4.4# tc qdisc del dev eth0 root netem```
    * Observe network queue clearing up.
    + ```bash-4.4# tc -s qdisc```
    * Observe server2 logs. It reconnects to cluster and gets all the new configuration from locator including deployed jars
 
 
 To start a separate cluster and configuring gateway/receiver 
 
 Start two clusters one as above and one as following
 
 In gemfire.properties of one cluster add following 
 log-level=info
 distributed-system-id=1
 remote-locators=172.17.0.5[9009]
 
 In gemfire.properties of another cluster add following
 log-level=info
 distributed-system-id=2
 remote-locators=172.17.0.2[9009]

docker run --name locator1 --hostname=locator1 -v $(pwd)/data/:/data/ -it gemfire91 /bin/bash
docker run --name server1 --hostname=server1 -v $(pwd)/data/:/data/  --cap-add=NET_ADMIN -it gemfire91 /bin/bash
docker run --name server2 --hostname=server2 -v $(pwd)/data/:/data/  --cap-add=NET_ADMIN -it gemfire91 /bin/bash

docker run --name locator2 --hostname=locator2 -v $(pwd)/data/:/data/ -it gemfire91 /bin/bash
docker run --name server4 --hostname=server4 -v $(pwd)/data/:/data/  --cap-add=NET_ADMIN -it gemfire91 /bin/bash
docker run --name server5 --hostname=server5 -v $(pwd)/data/:/data/  --cap-add=NET_ADMIN -it gemfire91 /bin/bash


 
start locator --name=locator1 --port=9009 --J=-Dgemfire.distributed-system-id=1 --J=-Dgemfire.remote-locators=172.17.0.5[9009] --mcast-port=0 --dir=/data/locator1
start server --J=-XX:InitialHeapSize=4g --J=-XX:MaxHeapSize=4g --J=-XX:NewRatio=3 --J=-XX:SurvivorRatio=1 --J=-XX:+UseConcMarkSweepGC --J=-XX:+UseParNewGC  --J=-Xloggc:/data/server1/gc.log --J=-XX:+PrintGC --J=-XX:+PrintGCApplicationConcurrentTime --J=-XX:+PrintGCApplicationStoppedTime --J=-XX:+PrintGCDateStamps --J=-XX:+PrintGCDetails --J=-XX:+PrintGCTimeStamps --J=-XX:+PrintTenuringDistribution --name=server1 --mcast-port=0 --locators="172.17.0.2[9009]" --server-port=8085 --dir=/data/server1
start server --J=-XX:InitialHeapSize=4g --J=-XX:MaxHeapSize=4g --J=-XX:NewRatio=3 --J=-XX:SurvivorRatio=1 --J=-XX:+UseConcMarkSweepGC --J=-XX:+UseParNewGC  --J=-Xloggc:/data/server2/gc.log --J=-XX:+PrintGC --J=-XX:+PrintGCApplicationConcurrentTime --J=-XX:+PrintGCApplicationStoppedTime --J=-XX:+PrintGCDateStamps --J=-XX:+PrintGCDetails --J=-XX:+PrintGCTimeStamps --J=-XX:+PrintTenuringDistribution --name=server2 --mcast-port=0 --locators="172.17.0.2[9009]" --server-port=8085 --dir=/data/server2
connect --locator=172.17.0.2[9009]
configure pdx --disk-store=DEFAULT --read-serialized=true --auto-serializable-classes=com.gemfire.functions.*,com.gemfire.models.*
create region --name=Positions --type=PARTITION_PERSISTENT
create region --name=Transactions --type=PARTITION_PERSISTENT
create region --name=MarketPrices --type=PARTITION_PERSISTENT
 create region --name=FxRates --type=PARTITION_PERSISTENT
 
  
 start locator --name=locator2 --port=9009 --mcast-port=0 --J=-Dgemfire.distributed-system-id=2 --dir=/data/locator2
 start server --name=server4 --mcast-port=0 --locators="172.17.0.5[9009]" --server-port=8085  --dir=/data/server4
 start server --name=server5 --mcast-port=0 --locators="172.17.0.5[9009]" --server-port=8085  --dir=/data/server5
 connect --locator=172.17.0.5[9009]
 
 configure pdx --disk-store=DEFAULT --read-serialized=true --auto-serializable-classes=com.gemfire.functions.*,com.gemfire.models.*
 create region --name=Positions --type=PARTITION_PERSISTENT --total-num-buckets=7 --off-heap
 create region --name=MarketPrices --type=PARTITION_PERSISTENT --total-num-buckets=7 --off-heap
 create region --name=FxRates --type=PARTITION_PERSISTENT --total-num-buckets=7 --off-heap
 create region --name=Transactions --type=PARTITION_PERSISTENT --off-heap

  
  create disk-store --name=gateway_store --dir=gateway_store  
  create gateway-sender --id=parallelPositionPersist --parallel=true --remote-distributed-system-id=2 --enable-persistence=true --disk-store-name=gateway_store 
  alter region --name=Positions --gateway-sender-id=parallelPositionPersist 
 
  On locator2 execute following
  create gateway-receiver 
 
 remote-locators=172.17.0.2[9009]
 
 remote-locators=172.17.0.5[9009]
 
 
 start locator --name=locator1 --port=9009 --J=-Dgemfire.distributed-system-id=1 --mcast-port=0 
 
 
  connect --locator=localhost[9009]

 //cms
 start server --J=-agentpath:/home/unmesh/softwares/YourKit-JavaProfiler-2019.1/bin/linux-x86-64/libyjpagent.so=disablestacktelemetry,exceptions=disable,delay=10000 --off-heap-memory-size=2g --J=-XX:InitialHeapSize=2g --J=-XX:MaxHeapSize=2g --J=-XX:NewRatio=3 --J=-XX:SurvivorRatio=1 --J=-XX:+UseConcMarkSweepGC --J=-XX:+UseParNewGC  --J=-Xloggc:gc1.log --J=-XX:+PrintGC --J=-XX:+PrintGCApplicationConcurrentTime --J=-XX:+PrintGCApplicationStoppedTime --J=-XX:+PrintGCDateStamps --J=-XX:+PrintGCDetails --J=-XX:+PrintGCTimeStamps --J=-XX:+PrintTenuringDistribution --name=server1 --mcast-port=0 --locators="192.168.0.119[9009]" --server-port=8085
 
 start server --J=-agentpath:/home/unmesh/softwares/YourKit-JavaProfiler-2019.1/bin/linux-x86-64/libyjpagent.so=disablestacktelemetry,exceptions=disable,delay=10000 --off-heap-memory-size=2g --J=-XX:InitialHeapSize=2g --J=-XX:MaxHeapSize=2g --J=-XX:NewRatio=3 --J=-XX:SurvivorRatio=1 --J=-XX:+UseConcMarkSweepGC --J=-XX:+UseParNewGC  --J=-Xloggc:gc2 --J=-XX:+PrintGC --J=-XX:+PrintGCApplicationConcurrentTime --J=-XX:+PrintGCApplicationStoppedTime --J=-XX:+PrintGCDateStamps --J=-XX:+PrintGCDetails --J=-XX:+PrintGCTimeStamps --J=-XX:+PrintTenuringDistribution --name=server2 --mcast-port=0 --locators="192.168.0.119[9009]" --server-port=8086
 
 //g1gc
 
 start server --J=-agentpath:/home/unmesh/softwares/YourKit-JavaProfiler-2019.1/bin/linux-x86-64/libyjpagent.so=disablestacktelemetry,exceptions=disable,delay=10000 --off-heap-memory-size=2g --J=-XX:InitialHeapSize=2g --J=-XX:MaxHeapSize=2g --J=-XX:+UseG1GC  --J=-Xloggc:gc1.log --J=-XX:+PrintGC --J=-XX:+PrintGCApplicationConcurrentTime --J=-XX:+PrintGCApplicationStoppedTime --J=-XX:+PrintGCDateStamps --J=-XX:+PrintGCDetails --J=-XX:+PrintGCTimeStamps --J=-XX:+PrintTenuringDistribution --name=server1 --mcast-port=0 --locators="192.168.0.119[9009]" --server-port=8085
  
 start server --J=-agentpath:/home/unmesh/softwares/YourKit-JavaProfiler-2019.1/bin/linux-x86-64/libyjpagent.so=disablestacktelemetry,exceptions=disable,delay=10000 --off-heap-memory-size=2g --J=-XX:InitialHeapSize=2g --J=-XX:MaxHeapSize=2g --J=-XX:+UseG1GC --J=-Xloggc:gc2.log --J=-XX:+PrintGC --J=-XX:+PrintGCApplicationConcurrentTime --J=-XX:+PrintGCApplicationStoppedTime --J=-XX:+PrintGCDateStamps --J=-XX:+PrintGCDetails --J=-XX:+PrintGCTimeStamps --J=-XX:+PrintTenuringDistribution --name=server2 --mcast-port=0 --locators="192.168.0.119[9009]" --server-port=8086
 
 
 off-heap
 //cms
  start server --J=-agentpath:/home/unmesh/softwares/YourKit-JavaProfiler-2019.1/bin/linux-x86-64/libyjpagent.so=disablestacktelemetry,exceptions=disable,delay=10000 --off-heap-memory-size=3g --J=-XX:InitialHeapSize=1g --J=-XX:MaxHeapSize=1g --J=-XX:NewRatio=3 --J=-XX:SurvivorRatio=1 --J=-XX:+UseConcMarkSweepGC --J=-XX:+UseParNewGC  --J=-Xloggc:gc1.log --J=-XX:+PrintGC --J=-XX:+PrintGCApplicationConcurrentTime --J=-XX:+PrintGCApplicationStoppedTime --J=-XX:+PrintGCDateStamps --J=-XX:+PrintGCDetails --J=-XX:+PrintGCTimeStamps --J=-XX:+PrintTenuringDistribution --name=server1 --mcast-port=0 --locators="192.168.0.119[9009]" --server-port=8085
                                                                                                                                                                                             
  start server --J=-agentpath:/home/unmesh/softwares/YourKit-JavaProfiler-2019.1/bin/linux-x86-64/libyjpagent.so=disablestacktelemetry,exceptions=disable,delay=10000 --off-heap-memory-size=3g --J=-XX:InitialHeapSize=1g --J=-XX:MaxHeapSize=1g --J=-XX:NewRatio=3 --J=-XX:SurvivorRatio=1 --J=-XX:+UseConcMarkSweepGC --J=-XX:+UseParNewGC  --J=-Xloggc:gc2 --J=-XX:+PrintGC --J=-XX:+PrintGCApplicationConcurrentTime --J=-XX:+PrintGCApplicationStoppedTime --J=-XX:+PrintGCDateStamps --J=-XX:+PrintGCDetails --J=-XX:+PrintGCTimeStamps --J=-XX:+PrintTenuringDistribution --name=server2 --mcast-port=0 --locators="192.168.0.119[9009]" --server-port=8086
  
  //g1gc
   
   start server --J=-agentpath:/home/unmesh/softwares/YourKit-JavaProfiler-2019.1/bin/linux-x86-64/libyjpagent.so=disablestacktelemetry,exceptions=disable,delay=10000 --off-heap-memory-size=3g --J=-XX:InitialHeapSize=1g --J=-XX:MaxHeapSize=1g --J=-XX:+UseG1GC  --J=-Xloggc:gc1.log --J=-XX:+PrintGC --J=-XX:+PrintGCApplicationConcurrentTime --J=-XX:+PrintGCApplicationStoppedTime --J=-XX:+PrintGCDateStamps --J=-XX:+PrintGCDetails --J=-XX:+PrintGCTimeStamps --J=-XX:+PrintTenuringDistribution --name=server1 --mcast-port=0 --locators="192.168.0.119[9009]" --server-port=8085
    
   start server --J=-agentpath:/home/unmesh/softwares/YourKit-JavaProfiler-2019.1/bin/linux-x86-64/libyjpagent.so=disablestacktelemetry,exceptions=disable,delay=10000 --off-heap-memory-size=3g --J=-XX:InitialHeapSize=1g --J=-XX:MaxHeapSize=1g --J=-XX:+UseG1GC --J=-Xloggc:gc2.log --J=-XX:+PrintGC --J=-XX:+PrintGCApplicationConcurrentTime --J=-XX:+PrintGCApplicationStoppedTime --J=-XX:+PrintGCDateStamps --J=-XX:+PrintGCDetails --J=-XX:+PrintGCTimeStamps --J=-XX:+PrintTenuringDistribution --name=server2 --mcast-port=0 --locators="192.168.0.119[9009]" --server-port=8086
   