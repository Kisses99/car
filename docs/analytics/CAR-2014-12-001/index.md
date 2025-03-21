---
title: "CAR-2014-12-001: Remotely Launched Executables via WMI"
layout: analytic
submission_date: 2014/12/02
information_domain: Host, Network
subtypes: PCAP
analytic_type: TTP
contributors: MITRE
---

Adversaries can use [Windows Management Instrumentation (WMI)](https://attack.mitre.org/techniques/T1047) to move laterally by launching executables remotely. For adversaries to achieve this, they must open a WMI connection to a remote host. This RPC activity is currently detected by [CAR-2014-11-007](CAR-2014-11-007). After the WMI connection has been initialized, a process can be remotely launched using the command: `wmic /node:"<hostname>" process call create "<command line>"`, which is detected via [CAR-2016-03-002](CAR-2016-03-002).

This leaves artifacts at both a network (RPC) and process (command line) level. When wmic.exe (or the schtasks API) is used to remotely create processes, Windows uses RPC (135/tcp) to communicate with the the remote machine.

After RPC authenticates, the RPC endpoint mapper opens a high port connection, through which the schtasks Remote Procedure Call is actually implemented. With the right packet decoders, or by looking for certain byte streams in raw data, these functions can be identified.

When the command line is executed, it has the parent process of `C:\windows\system32\wbem\WmiPrvSE.exe`. This analytic looks for these two events happening in sequence, so that the network connection and target process are output.

Certain strings can be identifiers of the WMI by looking up the interface UUID for IRemUnknown2 in different formats

-   UUID `00000143-0000-0000-c000-000000000046` (decoded)
-   Hex `43 01 00 00 00 00 00 00 c0 00 00 00 00 00 00 46` (raw)
-   ASCII `CF` (printable text only)

This identifier is present three times during the RPC request phase. Any sensor that has access to the byte code as raw, decoded, or ASCII could implement this analytic.
The transfer syntax is 

-   UUID `8a885d04-1ceb-11c9-9fe8-08002b104860` (decoded)
-   Hex `04 5d 88 8a eb 1c c9 11 9f e8 08 00 2b 10 48 60` (raw)
-   ASCII \`]+H\`\` (printable text only)

Thus, a great ASCII based signature is

-   `*CF*]+H*CF*CF*host*"`

### Output Description

Identifies the process that initiated the RPC request (such as wmic.exe or powershell.exe), as well as the source and destination information of the network connection that triggered the alert.


### ATT&CK Detection

|Technique|Tactic|Level of Coverage|
|---|---|---|
|[Windows Management Instrumentation](https://attack.mitre.org/techniques/T1047/)|[Execution](https://attack.mitre.org/tactics/TA0002/)|High|

### Data Model References

|Object|Action|Field|
|---|---|---|
|[flow](/data_model/flow) | [message](/data_model/flow#message) | [dest_port](/data_model/flow#dest_port) |
|[flow](/data_model/flow) | [message](/data_model/flow#message) | [proto_info](/data_model/flow#proto_info) |
|[flow](/data_model/flow) | [message](/data_model/flow#message) | [src_port](/data_model/flow#src_port) |
|[process](/data_model/process) | [create](/data_model/process#create) | [command_line](/data_model/process#command_line) |
|[process](/data_model/process) | [create](/data_model/process#create) | [exe](/data_model/process#exe) |
|[process](/data_model/process) | [create](/data_model/process#create) | [parent_exe](/data_model/process#parent_exe) |


### Implementations

#### Pseudocode

Look for instances of the WMI querying in network traffic, and find the cases where a process is launched immediately after a connection is seen. This essentially merges the request to start a remote process via WMI with the process execution. If other processes are spawned from `wmiprvse.exe` in this time frame, it is possible for race conditions to occur, and the wrong process may be merged. If this is the case, it may be useful to look deeper into the network traffic to see if the desired command can be extracted.


```
processes = search Process:Create
wmi_children = filter processes where (parent_exe == "wmiprvse.exe")

flows = search Flow:Message
wmi_flow = filter flows where (src_port >= 49152 and dest_port >= 49152 and proto_info.rpc_interface == "IRemUnknown2")

remote_wmi_process = join wmi_children, wmi_flow where (
 wmi_flow.time < wmi_children.time < wmi_flow.time + 1sec and 
 wmi_flow.hostname == wmi_children.hostname 
)

output remote_wmi_process
```


