Custom costs balancer (CCBalancer) Floodlight module
====================================================

Custom costs balancer is a Floodlight module, made to setup custom costs on the OpenFlow links cached by the Floodlight
Topology module.

The Floodlight controller implements the Openflow protocol, which specifications can be found here:
 [Openflow spec](http://www.openflow.org/documents/openflow-spec-v1.0.0.pdf)

This project depends on Floodlight, which can be found here:
 [Floodlight project on GitHub](https://github.com/floodlight/floodlight).

It has been tested with Mininet, which can be found here:
 [Mininet project on GitHub](https://github.com/mininet/mininet).


License
=======

This sofware is licensed under the Apache License, Version 2.0.

Information can be found here:
 [Apache License 2.0](http://www.apache.org/licenses/LICENSE-2.0).


The Custom costs balancer module
================================

Using the standard Topology module of Floodlight, link costs are only set to 1 by the controller itself. Thus, best path
are just calculated considering the minimum number of hops between two OpenFlow devices.
With custom costs balancer (CCBalancer) module, users can setup via new REST API integer custom costs for each link in
each direction. Thus, using external alghoritm, a user can also provide his own metric to calculate best paths.
At setup time the module sets all costs to 1.


Installation and configuration
==============================

This project has been developed and tested on Floodlight v0.90.

The module consists of the following components:
  * it.garr.ccbalancer package
  * it.garr.ccbalancer.web package
  * it.garr.ccbalancer.web.serializers package

  * TopologyInstanceCCBalancer.java
  * TopologyManagerCCBalancer.java
  * ITopologyServiceCCBalancer.java

Eclipse
-------

Using Eclipse, after the import of the Floodlight project:
  * Copy all it.garr.ccbalancer.* packages in the project folder (outside net.floodlightcontroller)
  * Copy TopologyInstanceCCBalancer.java, TopologyManagerCCBalancer.java and ITopologyServiceCCBalancer
    in net.floodlightcontroller.topology
  * Modify the file src/main/resources/META-INF/services/net.floodlightcontroller.core.module.IFloodlightModule
      + net.floodlightcontroller.topology.TopologyManagerCCBalancer instead of
       net.floodlightcontroller.topology.TopologyManager
      + add it.garr.ccbalancer.CCBalancer
  * Modify the file src/main/resources/floodlight.properties
      + net.floodlightcontroller.topology.TopologyManagerCCBalancer instead of
       net.floodlightcontroller.topology.TopologyManager
      + add it.garr.ccbalancer.CCBalancer

Runnable file
-------------

For production environment, a jar version of the module is downloadable as well from the root directory of this
GitHub repository.

Alternatively, it is possible to create your own ``ccbalancer.jar`` with the compiled files from this project.

According to Floodlight command sintax, you can integrate the jar file to your Floodlight installation running the
command:
```
java -cp floodlight.jar:ccbalancer.jar net.floodlightcontroller.core.Main -cf floodlight.properties
```

The parameters specified have the following meaning:
 * ``floodlight.properties`` is the file specifying the properties for the running instance of Floodlight,
   it is configred to start the KHopMetric module provided with this project.


REST APIs
=========

This project offers the following REST APIs:

 * ``http://controller-ip:8080/wm/ccbalancer/topocosts/json``: it only supports POST actions and it allows to set
     via REST API custom costs for each link of the network topology, for each of the two directions.
 * ``http://controller-ip:8080/wm/topology/links/json``: it only supports GET actions and it allows to read actual
     amount of bit transmitted (tx) for each active port of the entire topology.     


Utilities
=========

In the scripts folder, this projects provides some utility file that could be helpful to perform typical operations and
examples:

 * ``chktx.py`` is a python script that reads transmitted bits values of all interfaces and save results to a ms excel
   datasheet.
 * ``update.py`` is a python script that reads the transmitted bits values (is some values are saved) from a ms excel
   datasheet, compute the values to an integer cost through a sample algorithm and sends final values to the topology
   module on the controller.


update.py algorithm
===================
The update.py script receives the number of transmit bits from each interface activly partecipating in the network and
it sends out to the controller integer costs, thus optimizing network traffic and load.
Here is the sample algorithm used for computation:

cost = r * b(used)/b(tot) + s * [b(tot)-b(used)] + t * G

G = 0 if r * b(used)/b(tot) < h
G = r * [[b(used) - [h * b(tot)]]/[b(tot) - [h * b(tot)]] otherwise

* where h is a percent limit of bandwidth usage, manually expressed by user in update.py
* G is a function that makes the cost degrade if the usage of the single link is higher than h
* b(used) is the used bandwidth of that link in that specific direction
* b(tot) is the total capacity of that link in that specific direction
* r, s, t are three normalization constants, also depending of the biggest capacity used in the topology

Using this the algorithm provided with the new developed Java module permit to optimize traffic load on the network,
avoiding to have overloads and at the same time empty paths.

Use case
--------

The following use case has been implemented:
 * A full mesh network composed with 6 nodes have been emulated through Mininet
 * For test purposes link capacity has been set to 10 Mbit
 * With iperf we simulated a constant traffic between two or more sources
 * With chktx.py we recorded the capacity used during the transmission
 * Then, we sent another update through update.py to change link costs, depending on the traffic loads of the network.
 * With another instance of iperf we started another flow between other two hosts attached to the same two switches
   where the first flow is going through.
 * The second flow went through a different path from the first one, depending on the new link costs.

In all experiments we noted the ability of the network to adapt to different growth of traffic, avoiding overloads on
congested links. Of course, using ms Excel is just a sample to easy obtain quick results and build graphs at the same
time but the system can be integrated with passive monitoring systems as well (i.e. IPFIX).

REST API tutorial
---------------------

Below, an example of the REST API used to set new costs can be found:

 * JSON to define new link costs through http://controller-ip:8080/wm/ccbalancer/topocosts/json
   ```
   [
       {'src':'00:00:00:00:00:00:00:02','outPort':'2','dst':'00:00:00:00:00:00:00:03','inPort':'2','cost':'10'},
       {'src':'00:00:00:00:00:00:00:04','outPort':'4','dst':'00:00:00:00:00:00:00:06','inPort':'1','cost':'10'},
       {'src':'00:00:00:00:00:00:00:01','outPort':'1','dst':'00:00:00:00:00:00:00:02','inPort':'1','cost':'10'},
       {'src':'00:00:00:00:00:00:00:01','outPort':'2','dst':'00:00:00:00:00:00:00:03','inPort':'1','cost':'10'}
   ]
   ```
