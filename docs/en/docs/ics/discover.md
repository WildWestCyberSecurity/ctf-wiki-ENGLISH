# ICS_CTF Discovery

> The content of this section on ICS CTF competitions comes from the author's own competition experience. If there are any inaccuracies, please feel free to point them out.


## ICS Device Discovery

ICS device discovery is the prerequisite for ICS competitions. Currently, for ICS device scanning, a large number of tools are integrated in Nmap, Metasploit, and Censys for discovering online PLCs, DCS, and other ICS devices.


## ICS Scanning Scripts


### Port-Based ICS Information Scanning Scripts


When dealing with a large number of IPs, how do you discover ICS devices? Besides ICS-specific ports, most ports run normal services such as FTP, SSH, Telnet, SMTP, NTP, and other standard network services. The following table lists the currently available open-source ICS scanning scripts.


|Port|Protocol/Device|Source|
|:-----|:------|:------|
|102(TCP)|siemens s7|nmap --script s7-info.nse -p 102 [host] <br>nmap -sP --script      s71200-enumerate-old.nse -p 102 [host]|
|502(TCP)|modbus|nmap --script modicon-info -p 502 [host]|
|2404(TCP)|IEC 60870-5-104|nmap -Pn -n -d --script iec-identify.nse  --script-args='iec-identify.timeout=500' -p 2404 [host]|
|20000(TCP)|DNP3|nmap -sT --script dnp3-enumerate.nse -p 20000 [host] <br>nmap --script dnp3-info -p 20000 [host]|
|44818(TCP)|Ethernet/IP|nmap --script enip-enumerate -sU  -p 44818 [host]|
|47808(UDP)|BACnet|nmap --script BACnet-discover-enumerate.nse -sU  -p 47808 [host]|
|1911(TCP)|Tridium Nixagara Fo|nmap --script fox-info.nse -p 1911 [host]|
|789(TCP)|Crimson V3|nmap --scripts cr3-fingerprint.nse -p 789 [host]|
|9600(TCP)|OMRON FINS|nmap --script ormontcp-info -p 9600 [host]|
|1962 (TCP)|PCWorx|nmap --script pcworx-info -p 1962 [host]|
|20547(TCP)|ProConOs|nmap --script proconos-info -p 20547 [host]|
|5007(TCP)|Melsec-Q|nmap -script melsecq-discover -sT -p 5007 [host]|
|5006|Melsec-Q|nmap -script melsecq-discover-udp.nse -sU -p 5006 [host]|
|956(TCP)|CSPV4|Unknown|
|4840(TCP)|OPCUA|Unknown|
|18245(TCP)|GE SRTP|Unknown|
|1200(TCP)|Codesys|nmap –script codesys-v2-discover.nse [host]|
|10001|atg|nmap --script atg-info -p 10001 [host]|
|2222|cspv4|nmap --script cspv4-info -p 2222 [host]|
|1911|fox|nmap --script fox-info.nse -p 1911 [host]|
|4800|moxa|nmap -sU --script moxa-enum -p 4800 [host]|
|137|siemens wincc|sudo nmap -sU --script Siemens-WINCC.nse -p137 [host]|
|445|stuxnet|nmap --script stuxnet-detect -p 445 [host]|

The above scripts do not fully list all currently available script information. To be continued...

### Configuration Software-Based Component Scanning Methods

Various ICS vendors often come with their own configuration software. When the configuration software connects to devices on the current intranet, it can automatically discover target PLC devices.

|Port|Protocol/Device|Connection Method|
|:-----|:------|:------|
|102(TCP)|siemens s7|Siemens Step7 software has a built-in function to scan PLC devices in the current network segment|
|502(TCP)|modbus|Schneider SoMachine Basic has a built-in function to scan PLC devices in the intranet network segment when connecting to PLC devices|


## ICS Scanning and Discovery Engines

### Shodan Engine

*Shodan is a cyberspace search engine that primarily searches for devices on the Internet, including servers, cameras, ICS devices, smart home devices, etc. It can identify their versions, locations, ports, services, and other information. Shodan added ICS protocol detection in 2013. Users can directly search for all data of a specific ICS protocol using its port, and users can also use feature Dorks to directly search for corresponding device data.*

### ZoomEye Engine

*ZoomEye is a cyberspace search engine built by Knownsec. ZoomEye launched its ICS section (ics.zoomeye.org) in March 2015. ZoomEye supports data retrieval for 12 ICS protocols. Users can also use ICS protocol ports and feature Dork keywords to discover ICS hardware and software exposed on the Internet. For ICS protocol type data, ZoomEye has enabled a protection policy, and regular users cannot view it directly.*

### FOFA Engine

*FOFA is a cyberspace asset search engine launched by BAIMAOHUI. It can help users quickly perform network asset matching and accelerate subsequent work processes, such as vulnerability impact scope analysis, application distribution statistics, and application popularity ranking statistics.*

### Ditecting Engine

*Ditecting is a cyberspace ICS device search engine. The name is derived from the mythical beast "Diting" that can discern all things, with the intent to search for industrial control system networked devices exposed on the Internet, helping security vendors maintain ICS system security and track malicious actors.*

### Censys Engine

*Censys is a search engine that allows computer scientists to understand the devices and networks that make up the Internet. Powered by Internet-wide scanning, Censys enables researchers to find specific hosts and create comprehensive reports on the configuration and deployment information of devices, websites, and certificates.*

Different vulnerability search engines have different content, and there are significant differences in their configurations and deployment nodes. Currently, Shodan and Ditecting are more specialized in ICS searching. However, in terms of port coverage, the publicly announced search methods of each engine vary.

### Search Engine Comparison

To be continued...
