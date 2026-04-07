# S7Comm Sniffer

This project monitors **S7Comm communication** and forwards the extracted values to an **OPC UA server**.

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
cd S7-Sniffer/s7comm_sniffer
```

------------------------------------------------------------------------

## 4. Build the Docker Image

``` bash
sudo docker build -t s7comm .
```

Rebuild the Docker image **after every change to the project files**.

------------------------------------------------------------------------

## 5. Check Available Network Interfaces

Run the container with a shell:

``` bash
docker run -it   --entrypoint /bin/sh   --network host   --cap-add=NET_ADMIN   --cap-add=NET_RAW s7comm
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
python3 main.py --live --debug --opcua-endpoint tcp.opc://192.168.2.5:5000/S7Comm-monitoring/server/ --iface 1
```
You can add other arguments as needed. The available arguments are:

**Command Arguments:**
``` text
  --live                 : Use live network capture 
  --iface NUM            : Network interface number for capturing
  --pcap FILE            : Use a PCAP file instead of live capture
  --debug                : Enable verbose terminal output
  --list-cases           : Show available special-case mappings
  --case NAME            : Select special case (anlage, test, etc.)
  --opcua-endpoint URL   : OPC UA endpoint
  --tshark PATH          : Path to tshark binary (not required in Docker)
  --only-changes         : Only display and publish changed values
```
------------------------------------------------------------------------

## 7. Build the Docker Image

``` bash
sudo docker build -t s7comm .
```
Rebuild the Docker image **after every change to the project files**.


------------------------------------------------------------------------


## 8. Run the Sniffer

``` bash
docker run -it   --network host   --cap-add=NET_ADMIN   --cap-add=NET_RAW s7comm
```

The sniffer will now **capture S7Comm communication and forward the data to the OPC UA server**.

Example output:
``` text
======================================================================
PDU 259  |  Mar 12, 2026 09:11:52.112023000
----------------------------------------------------------------------
ITEM    NAME                       STATUS      VALUE               
----------------------------------------------------------------------
1       DB 1.DBX 21.0 CHAR 11      Success     RUN                 
2       DB 1.DBX 16.0 REAL 1       Success     8.399998664855957   
3       DB 1.DBX 0.0 BIT 1         Success     True                
4       DB 1.DBX 2.0 DWORD 1       Success     1750                
5       DB 1.DBX 6.0 WORD 1        Success     175                 
6       DB 1.DBX 8.0 BYTE 1        Success     15                  
7       DB 1.DBX 10.0 DINT 1       Success     150000              
8       DB 1.DBX 14.0 INT 1        Success     15                  
```
----------------------------------------------------------------------
## 9. Map Values to Names
To map values to specific names or variables, use the **current variable addresses**.

For example, to map: 
``` text
DB 1.DBX 21.0 CHAR 11 -> Status
DB 1.DBX 16.0 REAL 1  ->  Sensor_value
```
Follow these steps:

1. Open `special_cases.py`.

2. Under `CASE_SETS`, add a new PLC name and its variables. Example:


``` bash
CASE_SETS = {
    "rk3_anlage": {
        "DB 1.DBX 0.0 WORD 1":  ("Außentemperatur",   0.1, "°C"),
        "DB 1.DBX 2.0 WORD 1":  ("Verbrauchertank",   0.1, "°C"),
        "DB 1.DBX 68.0 WORD 1": ("ΔT Maschinenkreis", 0.1, "°C"),
        "DB 1.DBX 50.0 WORD 1": ("Kühlturmtank",      0.1, "°C"),
        "DB 1.DBX 56.0 WORD 1": ("ΔT Kühlturmkreis",  0.1, "°C"),
        "DB 1.DBX 4.0 WORD 1":  ("MS102",             0.1, "°C"),
        "DB 1.DBX 54.0 WORD 1": ("MS121",             0.1, "°C"),
        "DB 1.DBX 52.0 WORD 1": ("MS122",             0.1, "°C"),
        "DB 1.DBX 64.0 WORD 1": ("MS101",             0.1, "°C"),
    },
    "new_plc": {
        "DB 1.DBX 21.0 CHAR 11": ("Status",       1, ""),
        "DB 1.DBX 16.0 REAL 1":  ("Sensor_value", 1, ""),
    },
}
```
Template for adding new PLC cases:
``` bash
 "PLC_NAME": {
     "Address": ("Label", scale, "Unit"),
 }
```

3. Update `start.sh` to include the argument: 

``` bash
--case new_plc
```
4. Rebuild the Docker image. 

5. Rerun the sniffer.
