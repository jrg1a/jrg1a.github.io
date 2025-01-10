---
title: Network Enumeration
author: jrg1a
#date: 2025-01-10 20:00:00 -500
render_with_liquid: false
categories: [Red Team]
tags: [scanning, nmap, ports, recon, network, ip, ping, enumeration]
---

# Scanning

# Nmap - Network Enumeration

### Port States

Nmap considers different states when enumerating ports. TCP and UDP ports on a network host identify specific network services. Services adhere to network protocols and are linked to specific port numbers. *No two services can use the same port on a single IP address.*

**Basic Port States:**
- `Open Port:` Indicates the presence of a service actively listening on that port.
- `Closed Port:` Signifies the absence of any service listening on that port.

**Advanced Port States (Considering Firewalls):**
- `Open:` A service is actively listening on the port, and it is accessible.
- `Closed:` No service is listening, but the port is reachable and not blocked.
- `Filtered:` Inaccessibility of the port makes it unclear if it is open or closed, typically due to firewall interference.
- `Unfiltered:` The port is reachable, but it's unclear if it is open or closed, often identified during an ACK scan.
- `Open|Filtered:` Ambiguity in determining if the port is open or being filtered.
- `Closed|Filtered:` Uncertainty in discerning whether the port is closed or filtered.

#### Scanning Options:
- `-sS` SYN-scan (stealth)
- `-sV` Version Scan. Change intensity by using `--version-intensity` or similar alternatives like `--version-light`, `--version-all`
- `-A` Aggressive Scan
- `--disable-arp-ping`
- `-n`
- `-sT` Basic TCP Scan
- `-p-` All ports
- `-sC` default set of script scans
- `--packet-trace` shows packets
- `-PE` ICMP echo request for discovering live host (add `-sn` if you don’t want to follow that with a port scan)
- `-PM` ICMP address mask to discover live hosts
- `-PP` ICMP timestamp request to discover live hosts
- `-R` Perform DNS resolution for all addresses, including ones appearing offline
- `-S` Spoof IP-address
- `-D` Decoy address
- `-f` Fragment packets. More fragmentation can be done by: `-ff`, or `--mtu`
- `-sI` Zombie/Idle Scan
- `--reason` States how Nmap made its conclusion
- `--data-length`
- `-d` debugging


### Firewall and IDS/IPS Evasion

There are sevral ways to bypass firewall rules and IDS/IPS. Here you can read some methods to help you with this.

#### Determine Firewalls and the rules
Your packets can be registered as 'dropped' or 'rejected' after a scan, meaning the dropped packets are ignored by the target host, with no response. The rejected packets are returned with an RST flag, that contains differnt types of ICMP error codes, or nothing at all. 

Common errors might be:
- Net Unreachable
- Net Prohibited
- Host Unreachable
- Host Prohibited
- Proto Unreachable
- Port Unreachable

When scanning a target, the '-sA' flag increases the difficulty for a firewall and IDP/IPS system to detect than a regular '-sS' flag (SYN) or '-sT' Connect Scan. The TCP ACK scan (-sA) sends a TCP packet with only the ACK flag, and the host has to respond with an RST flag when a port is closed or open.

All connections attempts with SYN flag from extarlan networks are usually blocked by firewalls. When using ACK flag, it is often passed by a firewall, since it can not determine if there was established a connection first from the external or internal network. 💻

Here are some scanning options:
- Specify port: `-p`
- SYN scan: `-sS`
- ACK scan: `-sA`
- Disable ICMP Echo Request: `-Pn`
- Disable DNS resolution: `-n`
- Disable ARP ping: `--disable-arp-ping`
- Show packets: `--packet-trace`
  
##### SYN-Scan
```bash
sudo nmap 10.10.10.15 -p 21,22,25 -sS -n -Pn --disable-arp-ping --packet-trace
```

##### ACK-Scan
```bash
sudo nmap 10.10.10.15 -p 21,22,25 -Pn -n -sA --disable-arp-ping --packet-trace
```

### Detect IDS/IPS
The detection of IDS/IPS systems are more difficult since these are passive traffic monitoring systems. In short, the IDS system examine all connections between hosts and if it finds a packet containing something that its defined as suspicious, it will alert. 

IPS systems performs measures which is configured individually by an admin, to detect and prevent potential attacks automaticly. Keep in mind that the IDS and IPS are not the same application, and that the IPS serves as a compliment to IDS 😎

To determine if such systems are ran on the target network, we can use VPS (Virtual Private Servers) with different IP addresses, aka use several/many. By doing this, we avoid lockout if/when the target host detects something and blocks the IP-address. 

- The IDS systems alone are often there to aid the administrator by detecting potential attacks on their network. We can trigger the system by e.g., scanning one port very "agressively".
- To determine if an IPS system is present, scan from a single host (VPS), and if you are blocked and lose connection to the target network, the admin probably has configured some security measures. Then just continiue with anoter VPS.

### Decoys
Decoy scanning `-D` is the method which Nmap generates various random IP addresses inserted into the IP header to disguise the origin of the packet sent. We can generate random, and a specific number of IP-addresses seperated by a colon `:`. The real IP (yes, your IP address 🤔) is randomly placed between the generated IP addresses. 

###### Scan by using Decoys:
```bash
sudo nmap 10.10.10.15 -p 80 -sS -Pn -n --disable-arp-ping --packet-trace -D RND:5
```

*note*: the decoys must be alive, or else the service may be unreachable due to security measures to prevent SYN-flooding. 


##### Testing Firewall Rule
```bash
sudo nmap 10.10.10.15 -n -Pn -p445 -O
```

##### Scan By Using Different Source IP
```bash
sudo nmap 10.10.10.15 -n -Pn -p 455 -O -S 10.10.5.5 -e tun0
```

*note:* `-e tun0` sends all requests through the specified interface

### DNS Proxying
When scanning Nmap uses a reverse DNS resolution by default. DNS queries are often passed in most cases, since the web server is supposed to be found. DNS queries are made over the UDP port 53. Some also go through TCP port 53.

We can specify DNS servers ourself by doing `--dns-server <ns>, <ns> `. 

Companies often use their ofn DNS servers, and could be useful to avoid detection and interact with the hosts within the internal network. We can also make use of the TCP port 53, as an source port when scanning, if the admin uses the firewall to control this port and the IDS/IPS properly, the TCP packets from your machine will be trusted and passed along. 

###### SYN-Scan of a Filtered Port
```bash
sudo nmap 10.10.10.15 -p500000 -sS -Pn -n --disable-arp-ping --packet-trace
```

###### Syn-Scan From DNS Port
```bash
sudo nmap 10.10.10.15 -p50000 -sS -Pn -n --disable-arp-ping --packet-trace --source-port 53
```
If the firewall accepts TCP port 53, it's likely that IDS/IPS filters might also be configured weaker than others. Test this by trying to connect to this port by using Netcat.

###### Connect to the Filtered Port
```bash
ncat -nv --source-port 53 10.10.10.15 50000
```


#### Avoid supicous user-agent names
User-Agent can be logged by the remote web server when running Nmap scans with the -sC option when Nmap probes the web server
If an HTTP user agent isn't set at the time of running the given Nmap script, the logs on the target system could log a user agent containing Nmap Scripting Engine. This can be mitigated using the option --script-args http.useragent="CUSTOM_AGENT".
- `--script-args http.useragent="CUSTOM_AGENT"`

### More Scanning examples:
- TCP Null Scan: `sudo nmap -sN 10.10.242.146`
- TCP FIN Scan: `sudo nmap -sF 10.10.242.146`
- TCP Xmas Scan: `sudo nmap -sX 10.10.242.146`
- TCP Maimon Scan: `sudo nmap -sM 10.10.242.146`
- TCP ACK Scan: `sudo nmap -sA 10.10.242.146`
- TCP Window Scan: `sudo nmap -sW 10.10.242.146`
- Custom TCP Scan: `sudo nmap --scanflags URGACKPSHRSTSYNFIN 10.10.242.146`
- Spoofed Source IP: `sudo nmap -S SPOOFED_IP 10.10.242.146`
- Spoofed MAC Address: `--spoof-mac SPOOFED_MAC`
- Decoy Scan: `nmap -D DECOY_IP,ME 10.10.242.146`
- Idle (Zombie) Scan: `sudo nmap -sI ZOMBIE_IP 10.10.242.146`
- Scan Network Range: `sudo nmap 10.10.242.0/24 -sn -oA tnet | grep for | cut -d" " -f5`
- Service Detection: `nmap -sV --version-light 10.10.242.146`


Recommended process of enumeration:
1. **Enumerate Targets** - Determine the scope of network addresses to scan. This may involve a range of IP addresses, specific domains, or a list of hosts.
2. **Discover Live Hosts** - Use Nmap to identify which IP addresses within the specified range are active. This is typically done using a ping scan to detect hosts that respond.
3. **Reverse-DNS Lookup** - Perform a reverse-DNS lookup to gather the hostnames associated with the live IP addresses. This can provide more context about the nature of each host.
4. **Scan Ports** - Conduct a port scan on the discovered hosts to find open ports and infer potential services running on the host.
5. **Detect Versions** - Execute a service version detection scan with Nmap to determine the version of the services running on open ports. This can help in identifying known vulnerabilities.
6. **Detect OS** - Use Nmap’s OS detection feature to make an educated guess about the operating system running on each live host, which is crucial for further vulnerability assessment.
7. **Traceroute** - Perform a traceroute analysis with Nmap to understand the path packets take to reach each host. This information can be useful in network topology mapping.
8. **Scripts** - Utilize Nmap’s scripting engine to run scripts for more advanced enumeration such as vulnerability scanning, further service enumeration, or specific checks against known issues.
9. **Write Output** - Finally, save the results of the scans using Nmap’s output options for future reference or reporting. This might include normal, XML, or grepable formats for ease of analysis.
