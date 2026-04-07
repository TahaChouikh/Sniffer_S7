# S7CommPlus Sniffer

This project monitors **S7CommPlus communication** and forwards the extracted values to an **OPC UA server**.

------------------------------------------------------------------------

# Setup Guide

## 1. Get the IP Addresses

First, determine the **IP addresses of the PLC and the HMI**.

------------------------------------------------------------------------

## 2. Configure the IoT Device Network
The **IoT device IP address must be in the same subnet as the PLC and HMI**; otherwise, traffic cannot be monitored


1. Open the network configuration

``` bash
sudo nano /etc/dhcpcd.conf
```

2. If an interface configuration already exists, modify the static IP address. Otherwise, scroll to the bottom of the file and add the following lines with the new IP address:

``` bash
interface eth0
static ip_address=192.168.1.50/24
```

3. Save the changes then reboot:

``` bash
sudo reboot
```


------------------------------------------------------------------------

## 3. Navigate to the Project Directory

``` bash
cd S7-Sniffer/s7comm_plus_sniffer
```

------------------------------------------------------------------------

## 4. Build the Docker Image

``` bash
sudo docker build -t s7commplus .
```

Rebuild the Docker image **after every change to the project files**.

------------------------------------------------------------------------

## 5. Check Available Network Interfaces

Run the container with a shell:

``` bash
docker run -it   --entrypoint /bin/sh   --network host   --cap-add=NET_ADMIN   --cap-add=NET_RAW   s7commplus
```

Inside the container, list the available interfaces:

``` bash
tshark -D
```

Example output:

    1. eth0
    2. wlan0
    3. any

Write down the **interface number**, as it will be required for the `--iface` option.

In the example above, the interface number of **eth0** is **1**.



Exit the container:

``` bash
exit
```

------------------------------------------------------------------------

## 6. Configure start.sh

1. Open the file `start.sh`. 

2. Modify the `OPC UA endpoint` and `network interface number` as needed.

Example:

``` bash
python3 main.py --live --debug   --opcua-endpoint tcp.opc://192.168.2.5:5000/S7CommPlus-monitoring/server/   --iface 1
```
You can add other arguments as needed. The available arguments are:

**Command Arguments:**
``` text
--live                    Use live capture instead of PCAP file
--iface NUM               Network interface number for capturing
--pcap FILE               Use a PCAP file instead of live mode
--debug                   Enable verbose terminal output
--list-cases              Show available special-case mappings
--case NAME               Select special case (anlage, test, ...)
--opcua-endpoint URL      OPC UA endpoint
--tshark PATH             Path to tshark binary (not needed in Docker)
```
------------------------------------------------------------------------
## 7. Build the Docker Image

``` bash
sudo docker build -t s7commplus .
```

Rebuild the Docker image **after every change to the project files**.

## 8. Run the Sniffer

``` bash
docker run -it   --network host   --cap-add=NET_ADMIN   --cap-add=NET_RAW   s7commplus
```

The sniffer will now **capture S7CommPlus communication and forward the data to the OPC UA server**.

Example output:
``` text
======================================================================
Arrival Time : Mar 12, 2026 09:14:48.718754000
Object ID   : 0x70400002
----------------------------------------------------------------------
Item                 Type                      Value
----------------------------------------------------------------------
2                    Real                      2,100000
4                    Byte                      6
5                    DWord                     100150
6                    Int                       3
7                    DInt                      300
8                    Word                      15
```
----------------------------------------------------------------------
## 9. Map Values to Names
To map values to specific names or variables, use the **Object ID and the Item Number**.


For example, to map: 
``` text
Object ID   : 0x70400002
2  Real 2,100000 -> Sensor_value
4  Byte 6        -> Counter
```
Follow these steps:

1. Open `special_cases.py`.

2. Under `CASE_SETS`, add a new PLC name and its variables. Example:


``` bash
CASE_SETS = {
    "ca12": {
        ("0x70400004", "3"): "KW_Temp",
        ("0x70400004", "5"): "Druck_bar",
        ("0x70400004", "22"): "Leitfähigkeit_uS_cm",
        ("0x70400004", "35"): "WW_Temp",
        ("0x70400004", "37"): "WW_Level_cm",
        ("0x70400004", "41"): "KW_Level_cm"
    },
    "new_plc": {
        ("0x70400002", "2"): "Sensor_value",
        ("0x70400002", "4"):  "Counter"
    },
}
```
Template for adding new PLC cases:
``` bash
 "PLC_NAME": {
      ("Object ID", "Item Number"): "name",
 }
```

3. Update `start.sh` to include the argument: 

``` bash
--case new_plc
```
4. Rebuild the Docker image.
5. Rerun the sniffer.
