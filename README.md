# DHCP Lease and Queue Management Script

This script automates the management of DHCP leases and the associated network queues on MikroTik devices. It checks for active leases, assigns bandwidth limits based on address lists, manages queues dynamically, and logs unrecognized devices.

## Overview

The script performs the following tasks:

1. **Logs Debugging Information:** Logs details about the DHCP lease such as MAC address, IP, and bound status.
2. **Hostname Detection:** Retrieves the hostname of the device or defaults to `no-hostname` if none is found.
3. **Address List Matching:** Assigns different bandwidth limits based on the address list the device IP belongs to.
4. **Device Port Identification:** Determines the port where the device is connected (from bridge host or ARP table).
5. **Queue Management:** Creates or updates simple queues for devices with active leases and removes queues for devices whose leases are no longer active.
6. **ARP Handling:** Ensures devices in the ARP table also get managed under the same queue logic.

## Script Workflow

### 1. **Logging Information**
The script starts by logging details about the DHCP lease:
:log info "Lease MAC: $leaseActMAC, IP: $leaseActIP, Bound: $leaseBound";
This helps with debugging by tracking the lease state and associated device information.

### 2. **Hostname Retrieval**
The script attempts to get the hostname of the device:
:local hostName [/ip dhcp-server lease get [find where active-mac-address=$leaseActMAC && active-address=$leaseActIP] host-name];
If the hostname is not available, it defaults to `no-hostname`.

### 3. **Address List Matching**
It checks if the device's IP belongs to any address list and assigns bandwidth limits accordingly:
:foreach list in=[/ip firewall address-list find where address=$leaseActIP] do={ ... }
For example:
- **Family:** `20M/20M`
- **Servers:** `100M/100M`
- **CCTV:** `50M/50M`
- **IOT:** `50M/50M`
- **ALEXA:** `100M/100M`
- **Blocklist:** `56K/56K`

### 4. **Device Port Identification**
The script checks the bridge host or ARP table to find the port to which the device is connected:
:local devicePort "Unknown";
If the device is not found in these tables, the port remains as `"Unknown"`.

### 5. **Queue Management**
It creates or updates a simple queue for each device with an active lease. The queue's name is based on the device's comment and hostname. The script also adjusts the speed limit based on the address list:
:local queueName ("[" . $comment . "] " . $hostName);
It checks if a queue already exists for the device, and updates it if needed:
:if ([:len [/queue simple find name=$queueName]] > 0) do={ ... }
If no queue exists, it creates a new one:
:/queue simple add name=$queueName target=($leaseActIP . "/32") max-limit=$speedLimit comment=$comment;

### 6. **Removing Queues When Lease is Expired**
If the lease is no longer bound, the script removes the associated queue:
:log info "Removing queue for $leaseActIP";
:if ([:len [/queue simple find name=$queueName]] > 0) do={ /queue simple remove [find name=$queueName]; }

### 7. **ARP Table Handling**
The script ensures that devices in the ARP table are also included in simple queues. It creates a queue for each ARP entry:
:foreach arpEntry in=[/ip arp print] do={ ... }
The logic for creating or updating ARP queues is similar to the DHCP lease handling.

## Conclusion

This script automates the entire process of managing DHCP lease queues, dynamically creating and updating simple queues based on address lists and device status. It also ensures that ARP table entries are handled the same way as active DHCP leases.
