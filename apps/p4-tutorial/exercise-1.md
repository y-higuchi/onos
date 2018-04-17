# ONOS+P4 Tutorial: Exercise 1

The goal of this exercise is to introduce P4 and P4Runtime support in ONOS. This integration allows existing applications in ONOS to communicate to and program P4 devices on the network, and to operate in a pipeline agnostic manner, i.e. independently of the P4 program. 

To run this exercise you will need multiple terminal windows (or tabs) to operate with the CLI of Mininet, ONOS, and BMv2. We use the following convention to distinguish between commands of different CLIs.


* Commands starting with `$` are intended to be executed in the Ubuntu terminal prompt;
* `onos>` for commands in the ONOS CLI;
* `mininet>` for the Mininet CLI;
* `RuntimeCmd:` for the BMv2 CLI.

## Exercise steps

1. On terminal window 1, **start ONOS with a small subset of the applications** by executing the following command:

    ```
    $ cd $ONOS_ROOT
    $ ONOS_APPS=proxyarp,hostprovider,lldpprovider ok clean
    ```

    The `$ONOS_ROOT` environment variable points to the root ONOS directory. The `ok` command is an alias to run ONOS locally in your dev machine. Please note that if this the first time you run ONOS on this machine, or if you haven't built ONOS before, it can take some time (5-10 minutes depending on your Internet speed) to build and run ONOS. If you are running this exercise inside the tutorial VM, then ONOS should be already built for you.

    Once ONOS has started you should see log messages being print on the screen.
 
2. On terminal window 2, **activate the BMv2 driver and tutorial pipeconf** via the ONOS CLI.

    1. Use the following command to **access the ONOS CLI**:

        ```
        $ onos localhost
        ```

    2. Enter the following command to **activate the BMv2 driver**:

        ```
        onos> app activate org.onosproject.drivers.bmv2
        ```

        You should see the following message on the ONOS log:

        ```
        Application org.onosproject.drivers.bmv2 has been activated
        ```

    3. Enter the following command to **activate the pipeconf**:

        ```
        onos> app activate org.onosproject.p4tutorial.pipeconf
        ```

        You should see the following messages on the log:

        ```
        New pipeconf registered: p4-tutorial-pipeconf
        Application org.onosproject.p4tutorial.pipeconf has been activated
        ```

        Note the specific name used for this pipeconf `p4-tutorial-pipeconf`. We will later use this name to tell ONOS to deploy that specific P4 program to the switches.

    4. To **verify that you have activated all the required apps**, run the following command:

        ```
        onos> apps -a -s
        ```
       
        Make sure you see the following list of applications displayed:

        ```
        org.onosproject.generaldeviceprovider ... General Device Provider
        org.onosproject.drivers               ... Default Drivers
        org.onosproject.proxyarp              ... Proxy ARP/NDP
        org.onosproject.lldpprovider          ... LLDP Link Provider
        org.onosproject.protocols.grpc        ... gRPC Protocol Subsystem
        org.onosproject.protocols.p4runtime   ... P4Runtime Protocol Subsystem
        org.onosproject.p4runtime             ... P4Runtime Provider
        org.onosproject.drivers.p4runtime     ... P4Runtime Drivers
        org.onosproject.hostprovider          ... Host Location Provider
        org.onosproject.drivers.bmv2          ... BMv2 Drivers
        org.onosproject.p4tutorial.pipeconf   ... P4 Tutorial Pipeconf
        ```
    
    5. (optional) **Change flow rule polling interval**. Run the following command in the ONOS CLI:
    
        ```
        onos> cfg set org.onosproject.net.flow.impl.FlowRuleManager fallbackFlowPollFrequency 5
        ```
        
        This command tells ONOS to check the state of flow rules on switches every 5 seconds (default is 30). This is used to obtain more often flow rules stats such as byte/packet counters. It helps also resolving more quickly inconsistent states where some flow rules are installed in the ONOS store but not on the device (which can often happen when emulating a large number of devices in the same VM).
 
3. On terminal window 3, **run Mininet to set up a topology of BMv2 devices**.

    1. To **run Mininet**, use the following command:

        ```
        sudo -E mn --custom $BMV2_MN_PY --switch onosbmv2,pipeconf=p4-tutorial-pipeconf --controller remote,ip=127.0.0.1
        ```

        The `--custom` argument tells Mininet to use the given custom script to execute the BMv2 switch. The environment variable `$BMV2_MN_PY` points to the exact location of the custom Mininet script (you can use the command `echo $BMV2_MN_PY` to find out the exact location).

        The `--switch` argument specifies the kind of switch instance we want to run inside Mininet. In this case we are running a version of BMv2 that also produces some configuration files used by ONOS to discover the device (see next below), hence the name `onosbmv2`. The `pipeconf` sub-argument is used to tell ONOS which pipeconf to deploy on all devices.

        If needed, you can run BMv2 with debug logging enabled by passing the sub-argument `loglevel=debug`. For example:

        ```
        sudo -E mn ... --switch onosbmv2,loglevel=debug,pipeconf=p4-tutorial-pipeconf ...
        ```

    2. A set of **files are generated in the `/tmp` folder as part of this startup process**, to view them (on a separate terminal window):

        ```
        ls /tmp
        ```

    3. You will **find ONOS netcfg JSON files in this folder** for each BMv2 switch, open this file up, for example:

        ```
        cat /tmp/bmv2-1-netcfg.json
        ```

        It contains the configuration for the gRPC server and port used by the BMv2 switch instance for the P4Runtime service, the ID of pipeconf to deploy on the device, and other driver-specific information.

        **This file is pushed to ONOS automatically by Mininet when executing the switch instance**. If everything went as expected, you should see the ONOS log populating with messages like:

        ```
        Connecting to device device:bmv2:1 with driver bmv2
        ...
        Setting pipeline config for device:bmv2:1 to p4-tutorial-pipeconf...
        ...
        Device device:bmv2:1 connected
        ...
        ```

    4. **Check the BMv2 switch instance log**:

        ```
        $ less /tmp/bmv2-1-log
        ```

        By scrolling the BMv2 log, you should see a number of P4Runtime messages processed by the switch. These messages are used to install table entries and to read counters. You should also see many `PACKET_IN` and `PACKET_OUT` messages corresponding to packet-in/out processed by the switch and used for LLDP-based link discovery.

        These messages are generated by ONOS according to the flow rules generated by each application and based on the P4Info associated with the `p4-tutorial-pipeconf`.

        If you prefer to watch the BMv2 log updating in real time, you can use the following command to print on screen all new messages:

        ```
        $ bm-log 1
        ```

        This command will show the log of the switch instance with ID "1", which was used when starting the `simple_switch_grpc` process (look in the Mininet output for `Starting BMv2 target: simple_switch_grpc --device-id 1`...)

    5. **Check the flow rules inserted by each application in ONOS**. In the ONOS CLI type:

        ```
        onos> flows -s
        ```

        You should see the following output:

        ```
        deviceId=device:bmv2:1, flowRuleCount=3
            ADDED, bytes=798, packets=19, table=0, priority=40000, selector=[ETH_TYPE:arp], treatment=[immediate=[OUTPUT:CONTROLLER], clearDeferred]
            ADDED, bytes=0, packets=0, table=0, priority=40000, selector=[ETH_TYPE:bddp], treatment=[immediate=[OUTPUT:CONTROLLER], clearDeferred]
            ADDED, bytes=0, packets=0, table=0, priority=40000, selector=[ETH_TYPE:lldp], treatment=[immediate=[OUTPUT:CONTROLLER], clearDeferred]
        ```

        These flow rules are installed automatically for each device by the Proxy ARP and LLDP Link Discovery applications. The first one is used to intercept ARP requests (`selector=[ETH_TYPE:arp]`), which are sent to the controller (`treatment=[immediate=[OUTPUT:CONTROLLER]`), who in turn will reply with an ARP response or broadcast the requests to all hosts. The other two flow rules are used to intercept LLDP and BBDP packets (for the purpose of link discovery).

        These flow rules appear to be installed on table "0". This is a logical table number mapped by the pipeconf's interpreter to the P4 table named `t_l2_fwd` in [mytunnel.p4 (line 191)](./pipeconf/src/main/resources/mytunnel.p4#L191).
        
        This mapping is defined in the file [PipelineInterpreterImpl.java (line 103)](./pipeconf/src/main/java/org/onosproject/p4tutorial/pipeconf/PipelineInterpreterImpl.java#L103)
        
    6. **Compare ONOS flow rules to the table entries installed on the BMv2 switch**. To do this we can use the BMv2 CLI, on a separate terminal window type:
    
        ```
         $ bm-cli 1
        ```
        
        This will start the CLI for the BMv2 switch instance with ID "1". On the BMv2 CLI prompt, type the following command:
        
        ```
        RuntimeCmd: table_dump c_ingress.t_l2_fwd
        ```
        
        You should see exactly 3 entries, each one corresponding to a flow rule in ONOS. For example the flow rule matching on ARP packets should look like this in the BMv2 CLI:
        
        ```
        ...
        **********
        Dumping entry 0x2000002
        Match key:
        * standard_metadata.ingress_port: TERNARY   0000 &&& 0000
        * ethernet.dst_addr             : TERNARY   000000000000 &&& 000000000000
        * ethernet.src_addr             : TERNARY   000000000000 &&& 000000000000
        * ethernet.ether_type           : TERNARY   0806 &&& ffff
        Priority: 16737216
        Action entry: c_ingress.send_to_cpu -
        ...
        ```
        
        Note how the ONOS selector `[ETH_TYPE:arp]` has been translated to an entry matching only the header field `ethernet.ether_type`, while the bits of all other fields are set as "don't care" (mask is all zeros). While the ONOS treatment `[OUTPUT:CONTROLLER]` has been translated to the action `c_ingress.send_to_cpu`.
        
        **Important:**
        The BMv2 CLI is a powerful tool to debug the state of a BMv2 switch. Type `help` to show a list of possible commands. This CLI provides also auto-completion when pressing the `tab` key.
        
4.  It is finally time to **test connectivity between the hosts** of our Mininet network.

    1. On the Mininet prompt, start a ping between host2 and host1:

        ```
        mininet> h1 ping h2
        ```
        
        The ping should not work, and the reason is that we did not activate any ONOS application providing connectivity between these two hosts.
        
    2. While leaving the ping running on Mininet, **activate the Reactive Forwarding application using the ONOS CLI**:
    
        ```
        onos> app activate org.onosproject.fwd
        ```
        
        Once activated, you should see the the ping working. Indeed, this application installs the necessary flow rules to forward packets between the two hosts.
        
        Check the new flow rules in ONOS and the BMv2 CLI. You should see 3 new flow rules.
        
5. Congratulations, you completed the first exercise of the ONOS+P4 tutorial!

    To kill ONOS, press `ctrl-d` in the ONOS log terminal window. To kill Mininet, press `ctrl-d` in the Mininet CLI or type `exit`.
   
## Bonus exercise
 
As a bonus exercise, you can re-run Mininet with a larger topology to see how Exercise 1 works with a more complex topology.

* Rerun the steps in Exercise 1, replacing step 3.i. with the following:
    
    ```
    sudo -E mn --custom $BMV2_MN_PY --switch onosbmv2,pipeconf=p4-tutorial-pipeconf --topo tree,3 --controller remote,ip=127.0.0.1
    ```
    
    **Important:** due to the limited resources of the VM, when executing many BMv2 switches, it might happen that some flow rules are not installed correctly on the switch (showing state `PENDING_ADD` when using ONOS command `flows`). In this case, ONOS provides an automatic reconciliation mechanism that tries to re-install the failed entries. To force ONOS to perform this process more often, **make sure to apply step 2.v before starting Mininet**.

* After you activated the Reactive Forwarding application as in step 4.ii., you can ping all hosts in the network using the following Mininet command:

    ```
    mininet> pingall
    ```
    
* You can visualize the topology using the ONOS web UI. Open a browser from within the tutorial VM (e.g. Firefox) to the following URL [http://127.0.0.1:8181/onos/ui/](). When asked, use the username `onos` and password `rocks`. You should see a nice tree topology.
