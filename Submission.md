# Which Categories Best Fit Your Submission and Why?
###Best Contribution to Performance Improvements
We seek to improve throughput and reduce latency by modifying Zuul, then benchmark and bake off the two implementations.

## Describe your Submission
We plan to port Zuul to make use of the Netty stack. Netty will provide a SEDA and non-blocking outbound IO which should lead to throughput improvements.

The deliverables will be performance test results for Zuul and the Zuul-Netty port on equivalent hardware where the stub and client are not saturated. In addition, a Zuul-Netty codebase will be delivered as an interesting starting place for a full Netty migration which we, and the community at large,  may like to investigate further. 

## Provide Links to Github Repo's for your Submission
### Fork of Netflix repo used for performance testing
https://github.com/neilbeveridge/zuul
### Netty 3 Port of Zuul used for performance testing
https://github.com/neilbeveridge/zuul-netty

## Approach
A proxy will be built out in Netty 3 with the following features:

- Default proxy stage rather than leaving this up to a dynamic filter. 
- Less lenient filter API such that buffers may be left in Kernel space unless really necessary. 
- Asynchronous outbound connections so that worker threads are not tied up waiting on the remote service. 
- Per-route outbound connection pool so that inbound connection churn does not cause outbound connection churn. 
- Filter stage thread pool, isolating worker threads from filter work. 
- Back port of Zuul groovy filter chain onto Netty pipeline. 

## Performance Test Plan
The objective of this exercise was to run comparison tests against Tomcat based Zuul against the Netty ported Zuul. The environment was setup in the Amazon cloud sourcing different machine specs for the components involved. Major tasks planned were,
-	Develop a test stub to mimic the downstream applications, we used NodeJS for a simple HTTP endpoint.
-	Develop jmeter script to simulate the load test scenario. 
-	Tune Tomcat’s thread /GC and HTTP connector configurations and execute the baseline test.
-	Re-run the load test scenario against the Zuul Netty port and collect the results. Carry out any diagnostics and identify any improvements, if required.
-	Analyse results and prepare the comparison report.

Instance name|EC2 - Machine Type
---|---
Zuul Proxy|M1.medium
Zuul Netty Proxy|M1.medium
Load Injector|m1.xLarge
Stub Nodejs -1|m1.large
Stub Nodejs -3|m1.large
Stub Nodejs -2|m1.large
Stub Nodejs -4|m1.large
Stub Nodejs -5|m1.large


## Performance Test Outcomes
As a result of our performance tests, we did identify that a carefully tuned Tomcat based Zuul was able to sustain more than 1375 TPS with a m1.medium hardware. When rerunning the same scenario agains the Zuul Netty port we hit CPU issue quite early in the test which affected the response times and the scalability. However, at lower loads of around 300 TPS we found that the Netty response times were better than the tomcat by 6%. Hence we strongly believe that the NIO based Netty will result in better throughput once the CPU issue is fixed. 
Upon some profiling and threaddump analysis we think we have hit an exisiting issue in relation to the NIO implementation, https://github.com/netty/netty/issues/327. We are continuing our investigation and would update this project as soon as we have more information. 

The following section details the performance test observations.

### Response time comparison
The table shown below compares the response times that were recorded at various throughput levels, 

Load TPS|Tomcat Zuul -Latency\_ms|Tomcat Zuul-%CPU|Netty Zuul -Latency\_ms|Netty Zuul -%CPU
---|---|---|---|---
350|64|20|60|56
700|64|42|69|84
1000|68|66|116|92
1200|74|78|152|96
1400|82|85|204|96

### Issues faced:
-	Initially during the load tests we observed that the load on the ZUUL proxy instance was saturating while there was enough CPU left on the box.
  *	To eliminate network being a bottleneck we upgraded the hardware to an x-large machine on the LoadInjector and the Zuul Proxy.
  * Later further diagnostics we found that the stub box we used for the downstream connections was the limiting factor. Node.js was not very efficiently using multiple cores efficiently, and hence we had to create an ELB and spin up multiple instances to handle the downstream load.
-	CPU burn issue that was caused by the IO worker threads within Netty \[EPollArrayWrapper.epollWait\[native\](),\], we tried the workaround but didn’t solve the CPU issue. https://github.com/netty/netty/issues/327

###Observations - Tomcat Zuul Proxy
-	Tomcat instance was hosting the Zuul proxy module and was listening on 80. The downstream systems were called through an internal loadbalancer in EC2.
-	To take advantage of the Tomcat’s APR we had compiled the tc-native libraries on the server, Apache Tomcat Native library 1.1.27 using APR version 1.4.8. 
-	Further, to maximize the application throughput garbage collector and the HTTP connector settings were tuned appropriately for the tomcat instance.
  -	JVM options: -Dzuul.max.host.connections=500 -server -Xms500m -Xmx500m -Xmn250m -XX:+UseParallelGC -XX:+UseCompressedOops -XX:+PrintFlagsFinal
  -	Connector properties: 
               protocol="HTTP/1.1"
               connectionTimeout="20000" 
               acceptorThreadCount="2"
               maxKeepAliveRequests="500"
               maxThreads="500"
               socketBuffer="15000"
               pollTime="5000"
-	Based on the response time vs throughput graph shown below it is evident that upto 1000 TPS there was no response time degradation after which the response times started to increase.
-	The reason for the increase in the response time was due to the CPU reaching its limits on the Zuul proxy server. At our maximum load of 1375 TPS we had seen a 30% increase in the response time and the CPU spiking at 100%
-	Based on the above test we can safely assume that additional CPU cores will improve the application throughput proportionally. 

###Observations - Netty Zuul Proxy
-	Zuul Netty port instance was listening on 80 and wired to the same downstream internal loadbalancer in EC2.
-	Zuul Netty port was running on a similar m1.medium box similar to the Tomcat Zuul.
-	The Zuul netty port was bound by CPU and started to hit 100% at a load of 600 TPS due to which the response times had increased significantly and the throughput saturated as a result.
-	The instance was profiled using the sampler and thread dumps revealed that the IO worker threads of Netty had been consuming the CPU cycles. We were able to pin down the CPU burn to sun.nio.ch.EPollArrayWrapper.epollWait[native](), https://github.com/netty/netty/issues/327.
-	Based on the workaround advice we tried the -Dorg.jboss.netty.epollBugWorkaround=true switch and also triend increasing the -Dorg.jboss.netty.selectTimeout to 100ms from its default 10ms. But there was no change in the behavior.
-	We were unable to show the actual benefit of the NIO channel in Netty due to the above CPU issue. However, if we implement any work around and tune the TCP stack and the buffer sizes involved we will be able to achieve substantial improvements.

###Comparison Graphs - Tomcat Zuul vs Netty Zuul

![Alt text](/images/Resptime.png)

![Alt text](/images/JVM\_cpu.png)

![Alt text](/images/JVM\_Heap.png)

![Alt text](/images/hostCPU.png)

![Alt text](/images/hostNetwork.png)

![Alt text](/images/tcpStats.png)


## Further Work
- We are continuing our investigation on an efficient workaround for the Netty CPU issue, aiming to resolve this issue in a couple of weeks time. Beyond which we will carryout our performance improvement/tuning test runs and update this project with results.
- Batch isolation of filter changes from traffic so that a new collection of filters don't receive traffic on a partial, inconsistent state. 
- Tuning of filter stage. 
- Buffer tuning.
- Netflix OSS flavour proxy port. 

## Post-deadline Update
- The CPU thrashing issue was tracked down to context switching casued by a badly sized worker stage thread pool. This has now been pinned to #cpus since these threads will never block for io. Performance of zuul-netty is now >2000TPS for 10k payload and 50ms stub dither. Full performance results will be published as a module in the zuul-netty project proper.
- Dedicated test results: https://github.com/neilbeveridge/zuul-netty/blob/master/performance-dedicated.md.
- EC2 test results: https://github.com/neilbeveridge/zuul-netty/blob/master/performance-ec2.md. 

## Credits & Thanks
- Project lead: Neil Beveridge.
- Performance lead: Raamnath Mani.
- Contributors: Fanta Gizaw, Will Tomlin.

A special thank you to Hotels.com for allowing us to contribute to this project during working hours.
