# Network Telemetry Lab Guide

## Streaming Telemetry with IOS-XE

Traditionally SNMP has been highly successful for monitoring enterprise networks, but it has limitations: unreliable transport, inconsistent encoding between versions, limited filtering and data retrieval options, as well as the impact to the CPU and memory of the running device when multiple Network Monitoring Solutions poll the device simultaneously. Model-Driven Telemetry addresses many of the shortfalls of legacy monitoring capabilities and provides an additional interface in which telemetry is now available to be published from.

With the availability of more programmatic interfaces and greater level of data available in controller based networking this starts to present even more opportunties for utilising telemetry data from the infrastructure. This lab will walk through how we can look to leverage some of that available data to create a simple network operations centre dashboard.

Since the IOS XE 16.6 release there has been support for model driven telemetry, which provides network operators with additional options for getting information from their network.

In this blog we'll use the TIG stack (Telegraf - to recieve data from the network device, InfluxDB - time series database and Grafana - Visualisation and dashboards) to collect, store and display the network and device data from our IOS-XE devices. The objective of this lab is to get you comfortable using TIG stack and Streaming Telemetry protocols. As you gain more expertise you'll be able to leverage these technologies to build more complex monitoring systems to gather data from a multitude of devices and systems and display in more useful ways.

![](https://github.com/sttrayno/Network-Telemetry-Lab-Guide/blob/master/images/tigstack.png?raw=true)

Before we get started with telemetry we'll need a network device to configure receive data from. One of the easiest test environments you'll find is on the Cisco DevNet Sandbox which has multiple options. These are completely free and can in some cases be accessed within seconds. https://developer.cisco.com/docs/sandbox/#!overview/all-networking-sandboxes

Most popular sandboxes include:

- IOS-XE (CSR) - Always-On
- IOS-CR - Always-On
- Multi IOS test environment (VIRL based) - Reservation required
- Cisco SD-WAN environment - Always-On
- Cisco DNA-C environment - Always-On

Please note you are free to use this with your own hardware or test environment. However the commands in this lab guide have been tested for the sandboxes they correspond to. For this lab guide we will be using the reservable IOS XE on CSR Recommended Code Sandbox which can be found on the Sandbox catalogue https://devnetsandbox.cisco.com/RM/Topology

![](https://github.com/sttrayno/Ansible-Lab-Guide/blob/master/images/sandbox-screen.png)

There is also a Network Streaming Telemetry sandbox which you can also find in the catalogue.

### Step 0 - Configure and Install Docker

For this lab we need to use Docker to host containers needed to run the TIG stack and yang-explorer which allows us to explore the streaming telemetry models available for our network device. If you already have docker

Alternatively, you can use Model Driven Telemetry sandbox already available in the DevNet sandbox which has a developer box with both of these containers already configured and running which will save you a number of these steps. For completeness we'll walk through all the steps here.

### Step 1 - Setup our TIG stack

First thing we need to do is configure our TIG stack on our local machine. Thankfully, Jeremy Cohoe has created a fantastic docker container with all the needed components preinstalled. You can pull the container from the Docker hub with the following shell command.

```docker pull jeremycohoe/tig_mdt```

Let that pull down the required image from Docker hub then run the following command to start the container. 

```docker run -ti -p 3000:3000 -p 57500:57500 jeremycohoe/tig_mdt```

### Step 2 - Setup our Yang explorer

Secondly we also need

```docker pull robertcsapo/yang-explorer```

Again, let that pull down the required image from Docker hub then run the following command to start the container. 

```docker run -it --rm -p 8088:8088 robertcsapo/yang-explorer```

Verify you can access the explorer at http://localhost:8088 then leave it aside for now, we'll come back to this later to look for the valid topics that we can receive data from.

Note: Yang explorer is only supported on Linux and Mac systems so if you are running windows it may make more sense to use the developer box within your DevNet sandbox reservation.

### Step 3 - Configure IOS-XE device for streaming telemetry and verify

Configuration of telemetry on IOS-XE can be done either by tradition CLI with the "telemetry ietf subscription" commands, or programmatically with XML. In this guide we'll use the traditional CLI to configure a couple of models which are available on the device.

Please note the receiver ip address will depend on the IP address of your device, as in this guide we're connected to the sandbox via a VPN we can find our IP address from the anyconnect statistics menu, alternatively if you're using the DevBox the IP address will be shown from within your sandbox environment.

![](https://github.com/sttrayno/Network-Telemetry-Lab-Guide/blob/master/images/ip-check.gif?raw=true)

For ease of use now we're going to use some predefined xpath's to collect data from, however should you wish to use your own device you can and use the yang explorer to find the xpath of the valid operational models.

To configure the ietf subscriptions for this lab use the below configs

```
   telemetry ietf subscription 101
    encoding encode-kvgpb
    filter xpath /process-cpu-ios-xe-oper:cpu-usage/cpu-utilization
    stream yang-push
    update-policy periodic 500
    receiver ip address 192.168.63.1 57500 protocol grpc-tcp


   telemetry ietf subscription 102
    encoding encode-kvgpb
    filter xpath /interfaces-ios-xe-oper:interfaces/interface/statistics
    stream yang-push
    update-policy periodic 500
    receiver ip address 192.168.63.1 57500 protocol grpc-tcp
  
   telemetry ietf subscription 103
    encoding encode-kvgpb
    filter xpath /memory-ios-xe-oper:memory-statistics/memory-statistic
    stream yang-push
    update-policy periodic 500
    receiver ip address 192.168.63.1 57500 protocol grpc-tcp

   telemetry ietf subscription 104
    encoding encode-kvgpb
    filter xpath /interfaces-ios-xe-oper:interfaces/interface
    stream yang-push
    update-policy periodic 500
    receiver ip address 192.168.63.1 57500 protocol grpc-tcp
```
    
When the 4 ietf subscriptions have been configured as above. You can verify the status of them with the command

```
csrv1000#show telemetry ietf subscription all
  Telemetry subscription brief

  ID               Type        State       Filter type
  --------------------------------------------------------
  101              Configured  Valid       xpath

csrv1000#
csrv1000#show telemetry ietf subscription 101 detail
Telemetry subscription detail:

  Subscription ID: 101
  Type: Configured
  State: Valid
  Stream: yang-push
  Filter:
    Filter type: xpath
    XPath: /process-cpu-ios-xe-oper:cpu-usage/cpu-utilization
  Update policy:
    Update Trigger: periodic
    Period: 500
  Encoding: encode-kvgpb
  Source VRF:
  Source Address:
  Notes:

  Receivers:
    Address                                    Port     Protocol         Protocol Profile
    -----------------------------------------------------------------------------------------
    192.168.63.1                               57500    grpc-tcp


csrv1000#
csrv1000#show telemetry ietf subscription 101 receiver
Telemetry subscription receivers detail:

  Subscription ID: 101
  Address: 192.168.63.1
  Port: 57500
  Protocol: grpc-tcp
  Profile:
  State: Connected
  Explanation:
```

### Step 4 - Configure Grafana dashboards

Now we should have our network device connected to our TIG stack pushing the telemetry data at the configured intervals, all thats now left to do is start to configure our dashboards

To log into your Grafana, navigate to http://localhost:3000/ and login with the username/password admin/Cisco123




