# ONOS+P4 Tutorial

This directory contains the source code and instructions to run the ONOS+P4
tutorial exercises.

For help, please write to the mailing list
[brigade-p4@onosproject.org](mailto:brigade-p4@onosproject.org).

## mytunnel.p4

These exercises are based on a simple P4 program called
[mytunnel.p4](./pipeconf/src/main/resources/mytunnel.p4) designed for this
tutorial.

To start, have a look a the P4 source code. Even if this is the first time you
see P4 code, the program has been commented to provide an understanding of the
pipeline behaviour to anyone with basic programming and networking background
and an high level knowledge of P4. While checking the P4 program, try answering
to the following questions:

* Which protocol headers are being extracted from each packet?
* How can the parser distinguish a packet with MyTunnel encapsulation from one
    without? 
* How many match+action tables are defined in the P4 program?
* What is the first table in the pipeline applied to every packet?
* Which headers can be matched on table `t_l2_fwd`?
* Which type of match is applied to `t_l2_fwd`? E.g. exact match, ternary, or
    longest-prefix match?
* Which actions can be executed on matched packets?
* Which actions should be invoked to send the packet to the controller?
* What happens if a matching entry is not found in table `t_l2_fwd`? WHat's the
    next table to be applied to the packet?

## MyTunnel Pipeconf

The `mytunnel.p4` program is provided to ONOS as part of a "pipeconf", along
with the Java implementations of some ONOS driver behaviours necessary to
control this pipeline.

The following Java classes are provided:

* [PipeconfFactory.java](./pipeconf/src/main/java/org/onosproject/p4tutorial/pipeconf/PipeconfFactory.java):
This class is declared as an OSGi component which is "activated" once the
pipeconf application is loaded. The main purpose of this class is to instantiate
the Pipeconf object and register that with the corresponding service in ONOS.

* [PipelineInterpreterImple.java](./pipeconf/src/main/java/org/onosproject/p4tutorial/pipeconf/PipelineInterpreterImpl.java):
Implementation of the `PipelineInterpreter` ONOS driver behaviour. The main
purpose of this class is to provide a mapping between ONOS constructs and P4
program-specific ones, for example methods to map ONOS well-known header fields
and actions to those defined in the P4 program.

* [PortStatisticsDiscoveryImpl](./pipeconf/src/main/java/org/onosproject/p4tutorial/pipeconf/PipelineInterpreterImpl.java):
Implementation of the `PortStatisticsDiscovery` ONOS driver behaviour. As the
name suggests, this behaviour is used to report statistics on the switch ports
(e.g. count of packets/bytes received and transmitted for each port). This
implementation is based on P4 counters defined in `mytunnel.p4`.

## MyTunnel App

This application is used to provide connectivity between each pair of hosts via
the MyTunnel protocol. App implementation can be found
[here](./mytunnel/src/main/java/org/onosproject/p4tutorial/mytunnel/MyTunnelApp.java)

## Tutorial exercises

### Exercise 1

[Click here to go to this exercise instructions](./exercise-1.md)

This exercise shows how to start ONOS and Mininet with BMv2, it also
demonstrates connectivity between hosts using the pipeline-agnostic application
"Reactive Forwarding".

### Exercise 2

[Click here to go to this exercise instructions](./exercise-2.md)

Similar to exercise 1, here connectivity between hosts is demonstrated using
pipeline-specific application "MyTunnel".
