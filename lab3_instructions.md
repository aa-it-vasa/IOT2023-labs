# Lab 3 - Organising Things with Gateways

## Overview

In the previous lab, we secured the connection between the MQTT broker and the
clients. However, there is still scope for unintended inter-device cross-talk.
You can restrict this by designing good policies and this was covered in one of
the optional tasks of the last lab. If the intention is to organise Things
together in a logical fashion, for e.g. at the customer level or at a level of
a floor of a building, then setting per customer and per device policy does not
scale. We need a better way of organising them and in process make sure that
the devices of one customer cannot unintentionally connect with the devices of another customer. 

Gateways are one good way to achieve vertical hierarchy among things. Policies
are good way of managing horizontal hierarchy.  For instance, it is useful to
have write only policy to a certain topic for all the data coming from all
temperature sensors sold to all customers, while the gateways prevent reading
one customer's data by another (malicious) customer. Gateways also act as an
edge and provide local computational infrastructure. 

There are several concerns with the gateway pattern. You need to design and
sell an additional piece of hardware. The Things and gateway need to discover
each other and establish a secure connection to allow dynamic scaling of Things
infrastructure. A problem that is exacerbated when Things share or change
association between the gateways. The communication should not only be
coordinated between gateway and Things, but also between the Things and your
cloud server.  The logic (software) for collecting and analysing data from
Things connected to each gateway may differ and this requires a scalable
solution to deploy and manage logic across all of the gateways.

In this lab, we will demonstrate a secure gateway deployment using AWS
Greengrass. AWS Greengrass is a software that extends cloud capabilities to
local devices, negating many of the concerns associated with the gateway
pattern expressed above. Among other packages, Greengrass software consists of
a MQTT broker that acts as a bridge between AWS IoT broker and Things, a
computing platform similar to cloud instances but on the gateway, and a PKI
based secure communication setup.  Greengrass coordinates with AWS IoT service
and helps your Things to "automagically" discover a closest deployed gateway
and connect to it. Once the connection is established, the Things can collect
and analyse data closer to the source of information, react autonomously to
local events, and communicate securely with each other within local network
even when the gateway is cutoff from the internet.

![](./figs/l3_gg_overview.jpg)

The goal of this lab is to understand how scalable gateways can be deployed,
while managing a secure discovery and connection processes between Things.  You
will also test your setup for unintentional cross-talk. [Appendix](#appendix) gives a brief
explanation on various concepts that you will come across during the lab
exercise and walks you through discovery and connection management.

## Setup the hardware

1. We will use a Raspberry Pi as the device gateway. The device is already
   powered up and connected to network. To connect to the Raspberry Pi,
   you need the IP address of the gateway. The IP address should be given
   to you during the lab.
   Of course to be able to reach your gateway from the VM, your laptop needs to 
   be also conected to the same network using the same wireless network as in Lab 1. 
2. To connect to the gateway, in your VM open a terminal and ssh into it.
   The username is `pi` and password is `raspberry`.
   Commands run in Raspberry Pi is preceded `pi>` and in your VM as `vm>`.

   ```bash
   vm> ssh pi@IPADDRESS
   ```

## Setup Greengrass Core Device

To setup the Greengrass software on the RPI we will use the Greengrass device setup script provided by AWS. This script will
1. Configure your device and installs the AWS IoT Greengrass Core software.
2. Configure your cloud-based resources.

Perform the following steps to setup the Greengrass software:

1. In the AWS Console Go to _Services > Internet of Things > IoT Greengrass_. 
2. Under _Manage_ select _Greengrass devices > Core devices_. 
3. Press the button _Set up one core device_. 
4. Give the core device a unique name that you will remember. For example _GreenGrassCore-groupname_. Write down the name of the core.
5. Select _Enter a new group name_ under _Thing group_. Give the group a unique name, for example, _GreengrassGroup-groupname_. Write down the name of the group.
6. Select _Linux_ under _Operating System_.
7. Some credentials are needed for the script to be able to communicate with the corresponding AWS services. Depending on the setup, these can be different for each gateway device or shared. For simplicity and security, these will be provided to each group during the lab. It is easiest if these are saved as environmental variables in the shell you are using on the RPI. Execute the following commands on the RPI.
```
pi> export AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
pi> export AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
pi> export AWS_SESSION_TOKEN=AQoDYXdzEJr1K...o5OytwEXAMPLE=
```
where the keys are replaced with their correct values. Also remember to remove the `pi>` statement. 
8. Download and start the script according to the instructions on the AWS Console. You can use `curl` to download the script, for example:
```
curl https://d1onfpft10uf5o.cloudfront.net/greengrass-device-setup/downloads/gg-device-setup-latest.sh > gg-device-setup-latest.sh && chmod +x ./gg-device-setup-latest.sh && sudo -E ./gg-device-setup-latest.sh bootstrap-greengrass-interactive 
```
9. Run the installer by executing the **specific** command given under _Run the installer_. Make sure you understand what the command does and that it executes successfully, i.e., that there are no error messages.
10. Press _View core devices_ to return to the list. Your device should now be visible in the list and its status should be _Healthy_. 


## Create Things

Unlike in previous labs, we will not use Arduino WiFi board to connect to the
Greengrass Core.  We will create a simulated Thing instead. This is because the
discovery process requires additional library that interacts with AWS IoT core
using REST APIs. In theory, we can do this with Arduino, but this will require
writing an additional library, something Arduino is lacking at this time.
Connecting Arduino WiFi to Greengrass is a nice little project of its own and
you are welcome to try it in your free time :)

The simulated Thing will be your terminal from your computer. To create a new
thing, we will create the key pair and register the public key with AWS IoT in
one step. 

1. Under _Manage > Greengrass devices > Core devices_ open the Greengrass core device you created in the previous step by clicking on its name. 
2. There are several tabs on the detail page of the core device (the RPI's Greengrass instance). 
- Components: Here all the different software components deployed to the core device is listed. For example, we have an MQTT broker (Moquette) and a MQTT bridge deployed. It is also possible to deploy what components are deployed under _Manage > Greengrass devices > Deployments_. We will look at this in the next lab.
- Client devices: here we can add local Things that will communicate through the Greengrass instance installed on the RPI. These can be, e.g., sensors or actuators (or our simulated thing in this case).
3. To register the thing select the tab _Client devices_ and press the button _Associate client devices_. Then press the button _View AWS IoT things_ which will open a list of the existing things. 
4. Press _Create things_ and select _Create single thing_ and press _Next_.  
5. Enter a unique name for your thing, e.g., `simthing_groupname` and press _Next_. Remember this name!
6. We want AWS to generate the certificates and keys that will be used to authorize the communication from the thing, so use the default _Auto-generate a new certificate (recommended)_ and press _Next_. 
7. In the _Policies_ window, do not select a policy and just press _Next_. 
8. In the window _Download certificates and keys_, download the device certificate, public and private keys and store them in the folder 
   `Publisher_Sim` on the VM. You can rename the certificate, public and private key as
   `publisher_sim.pem.crt`, `publisher_sim-public.pem.key` and
   `publisher_sim-private.pem.key`. Note that if you forget to save these at this stage, you will need to recreate the thing!
6. Finally, download the RSA 2048 root ca from the window. You can also download it from
   [here](https://www.amazontrust.com/repository/AmazonRootCA1.pem). Save it as `root_ca.pem` in the same folder as they previous certificate and keys. 
7. Make sure that the Thing you created is visible in the list of things. 
8. You can now close this tab to go back to the _Associate client devices with core device_ popup window. 
9. Write the name of the created thing in the field _AWS IoT thing name_ and press _Add_. Press _Associate_. Make sure that your thing is now in the list _Associated client devices_ on the _Client devices_ tab.
10. We also need to allow the group the things is part of to send and receive MQTT messages to and from the MQTT broker running on the Greengrass device. Select the tab _Thing groups_ and press the name of the group. Select the tab _Policies_ and press the buttom _Manage policies_. Press _Add policy_ and select _GreengrassV2IoTThingPolicy_ and press _Update policies_. It should look like this:


![](./figs/thing_group_policies.png)

**Optional:** If you want to do the optional task repeat the previous steps to create another Thing. Call it _Snoopy_subscriber_GROUPNAME_. We will
try to simulate a malicious subscriber using this Thing. After creation, select the _Snoopy_subscriber_GROUPNAME_ thing and press _Disassociate_ to deactivate it.

To setup a connection between your device and the cloud, through the gateway we will need to setup the MQTT broker on the Greengrass core device. 

1. In the _Client devices_ tab press the button _Manage endpoints_ under _MQTT broker endpoints_. 
2. In the field _Endpoint_ enter the IP address of your RPI. Enter _8883_ as the port and press _Update_. 

## Create the MQTT subscriptions

We also need to make sure that the MQTT messages are passed on through the broker on the Greengrass core device instance running on the RPI:
1. Under _Manage > Greengrass devices > Deployments_ press select the one corresponding to your Greengrass Core device group, e.g., with _GreenGrassCore-groupname_ as the _Target name_ and press _Revise_. 
2. Press _Next_ to advance to _Step 2_.
3. In the box _Public components_ make sure that _Show only selected components_ is disable to show the full list. Then make sure at least the following `aws.greengrass.Cli`, `aws.greengrass.clientdevices.mqtt.Bridge`, `aws.greengrass.clientdevices.mqtt.Moquette` are selected and press _Next_. 
4. In _Step 3_ select `aws.greengrass.clientdevices.mqtt.Bridge` and press _Configure component_. 
5. In the window _Configuration to merge_ insert the following
```
{
   "mqttTopicMapping": 
      {
      "GroupName_LocalToCloud": {
         "topic": "saiot/GROUPNAME/publish",
         "source": "LocalMqtt",
         "target": "IotCore"
      }
   }
}
```
where `GROUPNAME` should be a unique identifier for your group. Remember the MQTT topic, i.e., `saiot/GROUPNAME/publish`.
4. Press _Confirm_.
5. Press _Next_ on _Step 4_ and then _Deploy_ on _Step 5_.

Now go back to _Manage > Greengrass devices > Deployments_. The deployment you modified will first have the status `Active` which indicates that the changes are being processed on the Greengrass instance on the RPI. After it is finished it will change to `Completed` (it might take some minutes).

To troubleshoot you can view the Greengrass logs on the RPI by executing the following command
```
rpi> sudo cat /greengrass/v2/logs/greengrass.log
```
or 
```
rpi> sudo tail -f /greengrass/v2/logs/greengrass.log
```
to also print new lines that are printed in the log file. 

## Test the connection

1. From _AWS IoT_ -> _Test_ -> _MQTT test client_, setup a new subscriber to the topic
   _saiot/GROUPNAME/publish_. Select _Display payloads as strings (more
   accurate)_ option and then _Subscribe_.
2. In your VM, run the following commands
    
    ```
    vm> cd ./Publisher_Sim
    vm> python3 ../Lab3/pubSub.py -e ENDPOINT -r root_ca.pem -c publisher_sim.pem.crt -k publisher_sim-private.pem.key -n Publisher_GROUPNAME -t saiot/GROUPNAME/publish -m publish -M "Hello World" 
    ```
    
    You can get the ENDPOINT from _AWS IoT_ -> _Settings_ and under _Custom
    Endpoint_ . The names of topic must match with topic name in all the
    above steps. Name of the Thing give as `-n` option must match the name of
    the thing.

    **Verify**: If connection is not established, verify endpoint is correct.
    Also verify that the certificate paths are correct. The Thingname should be
    same as the Thingname in the Group under Greengrass page. Also verify that
    the topic exists.

   If you run these commands on your own machine (not using the VM), you will probably need to install the AWSIoTPythonSDK module in Python: `pip install AWSIoTPythonSDK`.

## To do

1. Get a working understanding of the code `pubsub.py`. The documentation can
   be found
   [here](https://s3.amazonaws.com/aws-iot-device-sdk-python-docs/sphinx/html/index.html#module-AWSIoTPythonSDK.core.greengrass.discovery.providers)
2. Explain how the Thing connects to your gateway, including TLS mutual
   authentication. Use relevant diagrams. Check the folder `groupCA` in your directory. (3p)
3. Modify `pubSub.py` to measure the latency of establishing the connection to the gateway. Use `time`
   module and use `perf_counter()` to measure the time. See
   [documentation](https://docs.python.org/3/library/time.html#time.perf_counter). Compare the measured time with the ping value optained from the VM to your gateway. (7p)

## Optional Tasks

### Verify privacy

Instead of using the `Publisher_Sim`, use `Snoopy_Subscriber_GROUPNAME` and try
to publish or subscribe to the same topic. 

### Hacked Thing

Lets say `Publisher_Sim` Thing was stolen and was hacked. From the device, the
hackers were able to retrieve group CA, topic, certificate, and thing name. As
the private key was stored in the EEPROM of the device, it remained a secret
and it was not stolen. Now check if you can connect to the Group and subscribe
or publish to the same topic. You need another piece of code `snoopy.py`
instead of `pubsub.py`. You also need to locate GROUPCA. It usually is in the
`Publisher_Sim`.

```
vm> cd ./Publisher_Sim
vm> python3 ../Lab3/snoopy.py -i IPADDRESS -c publisher_sim.pem.crt -g GROUPCA -k Snoopy_Subscriber_GROUPNAME/snoopy_subscriber-private.pem.key -n Publisher_GROUPNAME -t saiot/GROUPNAME/publish -M "Snooping..."

```

What does this tell about a good practice to setup security in IoT devices?

## Appendix

A Greengrass _Group_ consists of exactly one _Core_ along with a logical
collection of Things. The _Core_ manages connection between Things, gateway and
IoT services along with providing a platform to deploy compute instances from
the cloud. AWS calls this computation platform as AWS Lambda and if time
permits, we will explore this in the next lab. 

A Thing gets associated with a Group when you associate Thing's certificate
with the Group. This is defined in the AWS IoT cloud interface. The Thing then
discovers the associated gateway through AWS IoT cloud service as shown in the
figure below. Information such as local IP address and Root CA is exchanged
between the Thing and AWS IoT service. The Thing can now contact the gateway
through the local IP address and after successful connection share data between
gateway and the AWS IoT service through gateway's MQTT broker.

![](./figs/l3_gg_interact.png)

The security setup is given in the figure below. 

(_A_) - AWS Greengrass and AWS IoT needs to have enough permission to access
each others services.

(_B_) - An X.509 certificate used to authenticate a specific gateway

(_C_) - An X.509 certificate used to authenticate a specific Thing

(_D_) - Group role. Similar to (_A_), but this time the Greengrass core running
on the gateway needs specific permissions to access other AWS cloud
infrastructure.

(_E_) - Group CA certificate. A root certificate used by Things to validate the
certificate presented by a gateway during TLS mutual authentication.

![](./figs/l3_gg_security.png)

The mutual authentication happens as follows: 

1. The Thing contacts the AWS IoT service via HTTPS. Mutual authentication
   between the Thing and server occurs and authenticity of the AWS IoT server
   is established.
2. Similarly, the gateway also connects to the AWS IoT service via HTTPS. TLS
   mutual authentication is used to verify both AWS server and the gateway.
3. The AWS IoT server then sends all the groups a Thing is associated to, its
   respective IP address and their respective group CAs. Because we trust AWS
   IoT server (due to mutual authentication carried out in the previous step),
   the validity of group CA is also verified.
4. The Thing then connects to an interested gateway through the use of group
   CA. TLS mutual authentication happens between gateway and Thing now, and can
   be through MQTT. Local IP address is used to locate the MQTT broker, unlike
   the first and second step where an _endpoint_ was used.
5. Once connection is established, data now flows through the gateway to the
   server using MQTT protocol.

## References

1. [AWS Greengrass Documentation](https://docs.aws.amazon.com/greengrass/latest/developerguide/what-is-gg.html)
2. [AWS Greengrass SDK Documentation](https://s3.amazonaws.com/aws-iot-device-sdk-python-docs/sphinx/html/index.html#module-AWSIoTPythonSDK.core.greengrass.discovery.providers)
3. [AWS IoT Python SDK and samples](https://github.com/aws/aws-iot-device-sdk-python)
