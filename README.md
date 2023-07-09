Readme
Author: Linta Joseph

# Architecture
![image]()

# Demonstration 
[Google Drive link to the video]()

# Presentation
[Google Slides link to presentation](https://docs.google.com/presentation/d/1bdDMfEQ3tYdK1lGkH5KHnMsGwzbkqkJ0AcXlb6eBV8A/edit?usp=drive_link)

# Steps to replicate:
0. Reserve 3 A8 nodes in grenoble Iot-Lab and clone the RIOT repository into the A8 directory
1. Open four terminals
2. Run `sh ./transfer_files_to_IoT-Lab.sh -l <iot-lab_login>` in __one__ terminal
3. SSH into IoT-Lab and navigate to the A8 directory of an A8 node and run `iotlab_reset`
4. Run `sh ./setup_border_router_from_A8.sh` to run a border-router (this will be __A8-m3_1__)
5. In a __second__ terminal SSH into IoT-Lab and navigate to the A8 directory of an A8 node and run `iotlab_reset`
6. Run `ip -6 -o addr show eth0` to get the ip address of the A8 node
7. Run `broker_mqtts config_IoT.conf` (this will be __A8-m3_2__)
8. In a __third__ terminal SSH into IoT-Lab and navigate to the A8 directory of an A8 node and run `iotlab_reset`
9. Run `sh ./setup_mqtt_script.sh` (this will be __A8-m3_3__)
10. Type and the run `con {ip address from A8-m3_2} 1885`
11. After successfully connecting, run `start`
12. In a __forth__ terminal SSH to the grenoble server
13. Run `python3 mqtt_bridge/pybash_bridge.py <IP of A8-m3_2> <IP AWS EC2 Instance_1>`
14. Start two EC2 instances like described here: [Setup the AWS EC2 Instance](#setup-the-aws-ec2-instance)
15. On one of them create a file named "config_AWS.conf" and copy the contents of ./scripts/utils/config_AWS.conf into it.
16. Then run `mosquitto -c config_AWS.conf` set up a MQTT broker (this will be __AWS EC2 Instance_1__)
17. On the other one run `mosquitto_sub -h <Public IP of AWS EC2 Instance_1> -p 1886 -t data`

# Scripts
The functionality the scripts in _"/scripts"_ are executing:

## setup_run_border-router.sh: 
```
sh ./setup_run_border-router.sh -l <iot-lab_login> -n <node_number>
```
This script is not reliably working. Instead use [this](#setup_border_router_from_a8sh) <br>
__<iot-lab_login>__: Your login for the IoT-Lab testbed
<br>
__<node_number>__: The number of the A8 node you reserved for the border-router (if you reserved node a8-102 you would need to provide "102" here)

This Script can be executed after you established [SSH Access](https://www.iot-lab.info/legacy/tutorials/ssh-access/index.html) to IoT-Lab, a A8-node was reserved on IoT-Lab and the RIOT repository was cloned to the A8 directory (explained [here](https://www.iot-lab.info/legacy/tutorials/riot-public-ipv6-a8-m3/index.html) in the steps 1 - 4). 

The script will copy the _"gnrc_border_router.elf"_ to the ./A8 directory. It then will ssh onto the specified A8 Board, build and run the border-router.

## transfer_files_to_IoT-Lab.sh:
```
sh ./transfer_files_to_IoT-Lab.sh -l <iot-lab_login>
```

Transfers all files that are needed onto the A8 directory on IoT-Lab

## setup_mqtt_script.sh:
```
sh ./setup_mqtt_script.sh
```

This script runs the driver-application and publishes the data via MQTT.

## setup_border_router_from_A8.sh: 
```
sh ./setup_border_router_from_A8.sh
```

This script sets up a border router. This script can only be run when you already ssh'ed onto an A8 node and your current
directory is the A8 directory. To have script 
on the A8 folder the __transfer_files_to_IoT-Lab.sh__ was already run from your host mashine.

# Setup the AWS EC2 Instance 
0. From the AWS Learners Lab click on the "AWS" button to connect to "AWS Console"
1. When on the EC2 subpage on AWS (looking something like this: https://us-east-1.console.aws.amazon.com/ec2/home?region=us-east-1#Home:)
, press on __launch Instances__
2. Name the Instance
3. Select Ubuntu as OS Image
4. Set instance type to t2.micro
5. For Key-Pair set __"Proceed without a key pair"__
6. For Network setting click on "Edit"
7. Set "Auto-assign public ip" to "Enable"
8. For Inbound security group rules add a new group with "Type" = "All traffic" and "Source Type" = "anywhere"
9. Click "Launch Instance"
10. Wait until the instance is up and running and the click "Connect" and enter
the EC2 terminal
11. Continue with [Execution of commands on EC2](#execution-of-commands-on-ec2)


## Execution of commands on EC2
On EC2 Instance run the following commands:
```
#Update the list of repositories with one containing the latest version of #Mosquitto and update the package lists
sudo apt-add-repository ppa:mosquitto-dev/mosquitto-ppa
sudo apt-get update

#Install the Mosquitto broker, Mosquitto clients and the aws cli
sudo apt-get install mosquitto
sudo apt-get install mosquitto-clients
sudo apt install awscli
```

# How everything works
In Order to send data from an IoT device from IoT-Lab to the AWS cloud the 
application consists of 6 different services. 
1. Border-router
2. Rsmb MQTT Broker in IoT-Lab
3. IoT-Device with driver to supply data
4. Bridge to route MQTT messages from IoT-Lab to AWS
5. Mosquitto MQTT Broker on AWS
6. EC2 Instance to receive data on AWS

### Border-Router
The border-router is responsible to route messages between the 6Lo network and 'normal' IPv6 networks. So in order to send messages from an A8 node to the internet, a border-router is needed. To accomplish that, the border-router example aplication from RIOT was compiled and saved in the repository as "gnrc_border_router.elf, because it is easily implemented and was tested and therefore sped up implementation time.

### Rsmb Broker in IoT-Lab
The Really Small Message Broker is the MQTT broker of choice for MQTT-SN in the RIOT emcute_mqttsn tutorial which made it an intriguing option to look into. After finding out that it also offered the functionality to translate from MQTT-SN to MQTT and it was already tested and the configuration was known it was chosen as Broker within the IoT-Lab 6Lo network. It Was important to be able to translate between MQTT and MQTT-SN because the IoT-Device driver application that was designed during this project only sent MQTT-SN, but AWS was configured to only use MQTT.

### IoT-Device driver application
The IoT-Device driver application was designed to send data over the MQTT-SN protocol to a broker, that can be configuered in the command line by the application user. The application uses a SAUL-driver that generates data that starts at 0 and counts up to 9 iterativly, every second. The driver needs to be initialized and and registered in the SAUL registry. It can than be fetched from the registry in the main programm and the read function can be called to get the generated data. When the project is compiled and then started using the setup_mqtt_script.sh script, the user can type `con <ip adress of the rsmb>` to connect to the rsmb. As soon as this connection-building was successful the user can start the driver by running the command `start`.

### MQTT Bridge between IoT-Lab and AWS
To be able to send the MQTT messages from the driver over the rsmb to AWS-Broker a services that bridges between those two was needed. To accomplish that the way of least resistence was chosen, which was programming a python script to forward the message and running it on the grenoble server. The script uses the "subprocess" library to run shell commands within a python script. The script subscribes to the rsmb using `mosquitto_sub` and then publishes the message that was received to the AWS-Broker using `mosquitto _pub`

### Broker on AWS
The broker on AWS is configured using the config_AWS.conf file. Mosquitto is the library that is used to set up the broker on an EC2 istance.

### Data Receiver
To receive the data within the AWS an EC2 instance is used to run the `mosquitto_sub` command and thus subscribe to the Broker running on the other EC2 instance.
