# Complete Workflow: Outside xAPP + Physical BBU Connection
```
[Confirm BBU IP / Network Interface]
        ↓
[PC → SSH Connect to BBU]
        ↓
[Modify netcfg_xran.xml → Set srsUdpLogSocketIP = PC IP]
        ↓
[Prepare srs_udp_log_reader Folder & Permissions on PC]
        ↓
[Start RAN Service (wait ~100s)]
        ↓
[Run ./srs_udp_log_reader on PC]
        ↓
[Console + srs_udp_log.log Receive Logs]
        ↓
[Ctrl-C to Exit]
```

## 1️⃣ Confirm Physical BBU Environment
- **Find the BBU Server IP**
  - Ask your administrator or check documentation.
    - user:`greigns`
    - Ip address: `192.168.8.128`
    - passward:`Greigns-2022`
  - Use the **eno2** interface (since `eno1` is reserved for management).

---

## 2️⃣ Connect from Your PC to the BBU via SSH
```bash
ssh <username>@<BBU_IP>
```
```
======[Main Menu]==================================

        1: RAN Service UP             
        2: RAN Service DOWN           
        3: RAN Status Query           
        4: Reboot or Shutdown RAN 
        5: Configuration Change       
        6: SW Package Update          
        7: More Information           

===================================================
Input your choice(1-7): 

Current 5G System Status: 
  5GC:               OK
  L1:                OK   ( O-RU lost - [0] 72; )
  RAN:               OK
  Uptime:            00:02:00 ( 120 seconds )
```

## 3️⃣ Modify BBU Configuration File
Open the configuration file and set:
```
vim /home/htc_l1app/HTC_L1APP/bin/nr5g/gnb/l1/netcfg_xran.xml
```
```
srsUdpLogSocketIP = <Your_PC_IP>
```

## 4️⃣ Prepare Your PC Environment
Place the srs_udp_log_reader folder in any directory on your PC.
Set permissions:
```
chmod 777 -R srs_udp_log_reader
```

## 5️⃣ Start RAN Service
If RAN Service status is offline → click Start, wait ~100 seconds.

If RAN Service status is online → click Stop, wait ~30 seconds.


## 6️⃣ Run Log Reader on Your PC
Change directory:
```
cd /<your path>/srs_udp_log_reader
```
```
./srs_udp_log_reader
```
Output:

```
SRS UDP Log Reader Sample [Press Ctrl-C to Quit]
Waiting for SRS UDP log on port 2498 ...
```

Logs displayed in console.

File srs_udp_log.log created in the same folder.








