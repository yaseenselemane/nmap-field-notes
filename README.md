# nmap field notes: scanning a home network (beginner)

My notes from learning **nmap**, the standard tool for finding out which devices are on a network and what they're running. This is a beginner's lab: five scans, what each one shows, and how to read the output.

> **About the output:** the results below are **example output for a typical home network** (made-up addresses: a router, two phones, a laptop, a smart TV and a printer), written to show what each scan returns. The commands are exactly the ones to run.
>
> **Only scan networks you own or have written permission to test.** Scanning other people's networks can break their rules and, in many countries, the law.

Part of my networking labs: [ping and traceroute](https://github.com/yaseenselemane/ping-and-traceroute) · **nmap** · [WiFi survey](https://github.com/yaseenselemane/wifi-survey) · [Packet Tracer](https://github.com/yaseenselemane/cisco-packet-tracer-labs) · [vantage](https://github.com/yaseenselemane/vantage)

---

## Before you start

1. Install nmap from [nmap.org/download](https://nmap.org/download.html) (keep **Npcap** ticked on Windows).
2. Find your own network range: `ipconfig` shows something like `IPv4 Address 192.168.1.20` and `Subnet Mask 255.255.255.0`, so the network is **192.168.1.0/24** (256 addresses).
3. Run scans from an **admin** terminal. Some scan types need it.

## Scan 1: who's on the network? (host discovery)

```
nmap -sn 192.168.1.0/24
```

`-sn` means "no port scan": just find which addresses are alive (ARP on the local network, plus ping).

```
Nmap scan report for 192.168.1.1
Host is up (0.0021s latency).
MAC Address: XX:XX:XX:XX:XX:01 (TP-Link Technologies)
Nmap scan report for 192.168.1.20
Host is up.                                    <- this laptop
Nmap scan report for 192.168.1.34
Host is up (0.045s latency).
MAC Address: XX:XX:XX:XX:XX:22 (Unknown)       <- phone with a random MAC
Nmap scan report for 192.168.1.41
Host is up (0.031s latency).
MAC Address: XX:XX:XX:XX:XX:33 (Apple)
Nmap scan report for 192.168.1.52
Host is up (0.012s latency).
MAC Address: XX:XX:XX:XX:XX:44 (Samsung Electronics)
Nmap scan report for 192.168.1.60
Host is up (0.008s latency).
MAC Address: XX:XX:XX:XX:XX:55 (HP)
Nmap done: 256 IP addresses (6 hosts up) scanned in 3.12 seconds
```

**How to read it:** 6 devices are on. nmap guesses the maker from the first half of the MAC address (the OUI). "Unknown" usually means a phone using a **random MAC**, which modern phones do for privacy.

## Scan 2: which ports are open? (default port scan)

```
nmap 192.168.1.1
```

With no options, nmap checks the **1,000 most common TCP ports**.

```
Nmap scan report for 192.168.1.1
PORT    STATE    SERVICE
53/tcp  open     domain
80/tcp  open     http
443/tcp open     https
22/tcp  filtered ssh
```

**The three states you'll see most:**

| State | Means |
|---|---|
| **open** | Something is listening and answered. This is a service you can reach. |
| **closed** | The device answered "nothing here". The host is up, but the port isn't in use. |
| **filtered** | No answer at all. A firewall is probably dropping the packets, so nmap can't tell. |

The router runs DNS (53) for the network and a web admin page (80 and 443). SSH (22) is filtered: blocked, which is good.

## Scan 3: what software is it? (service versions)

```
nmap -sV 192.168.1.1 192.168.1.52 192.168.1.60
```

`-sV` talks to each open port to identify the program and version behind it.

```
192.168.1.1   (router)
  53/tcp   open  domain    dnsmasq 2.80
  80/tcp   open  http      lighttpd
  443/tcp  open  ssl/http  lighttpd

192.168.1.52  (smart TV)
  8008/tcp open  http      Google Chromecast httpd
  8009/tcp open  ssl/castv2 Chromecast cast protocol

192.168.1.60  (printer)
  80/tcp   open  http      HP HTTP Server
  631/tcp  open  ipp       HP printer IPP
  9100/tcp open  jetdirect HP JetDirect
```

**How to read it:** each port maps to a job. The TV's 8008/8009 are **Chromecast**, which is how phones cast to it. The printer's **9100** accepts print jobs directly, and **631 (IPP)** is the standard printing protocol. Version numbers matter because old versions can have known security holes.

## Scan 4: what operating system? (OS detection)

```
nmap -O 192.168.1.1
```

`-O` compares how the device responds to odd packets against a database of fingerprints. It needs admin rights and at least one open and one closed port.

```
Device type: general purpose|router
Running: Linux 4.X|5.X
OS details: Linux 4.15 - 5.6
Network Distance: 1 hop
```

**How to read it:** it's a guess with a confidence level, not a fact. Most home routers run Linux, which also matches a ping reply with **TTL 64** (see [ping and traceroute](https://github.com/yaseenselemane/ping-and-traceroute)).

## Scan 5: all the ports, on one device

```
nmap -p- 192.168.1.60
```

`-p-` checks **all 65,535 TCP ports**, not just the top 1,000. It's slower, but finds services on unusual ports.

```
PORT     STATE SERVICE
80/tcp   open  http
631/tcp  open  ipp
9100/tcp open  jetdirect
5357/tcp open  wsdapi
```

The extra port, **5357 (WSD)**, is how Windows discovers printers automatically. The default scan missed it.

## What I found, and what I'd fix

| Finding | Risk | Fix |
|---|---|---|
| Router admin page on plain **http (80)** | The admin password crosses the WiFi unencrypted | Use the https (443) page only, and turn off http admin if the router allows it |
| Printer open on **9100** to every device | Anyone on the WiFi can send print jobs | Fine at home. On a shared or office network, put printers on their own VLAN |
| Chromecast ports on the TV | Anyone on the WiFi can cast to it | Use a **guest network** for visitors so they can't see home devices |
| SSH on the router **filtered** | None | Good: management is closed from the LAN |
| A phone with an unknown maker | Can't identify it by MAC | Check the router's DHCP list for its name, or enable MAC randomisation awareness |

## What I learned

1. **Finding devices and scanning ports are different jobs.** `-sn` only asks "who's here?" (fast, quiet). A port scan asks "what's running?" (slower, noisier).
2. **open / closed / filtered tells you a lot.** *Closed* proves the host is up. *Filtered* usually means a firewall. Only *open* is a reachable service.
3. **Ports are just numbers. `-sV` tells you the truth.** Port 80 is usually web, but anything can listen anywhere. Version detection asks the service itself.
4. **Every open port is something to look after.** The printer, the TV and the router admin page are all doors. Fewer open ports means less to secure.
5. **OS detection is an educated guess.** It agreed with what ping's TTL suggested, but it's a fingerprint match, not proof.
6. **The default scan doesn't check everything.** `-p-` found a port the top-1,000 scan missed.
7. **Guest networks exist for a reason.** Visitors on the main WiFi can see and reach every device nmap found. A guest network hides them.
8. **Permission comes first.** nmap is a normal admin tool on your own network and an intrusion attempt on someone else's.

## Cheat sheet

| Command | What it does |
|---|---|
| `nmap -sn 192.168.1.0/24` | Find live hosts, no port scan |
| `nmap 192.168.1.1` | Top 1,000 TCP ports |
| `nmap -p 22,80,443 192.168.1.1` | Only these ports |
| `nmap -p- 192.168.1.1` | All 65,535 TCP ports |
| `nmap -sV 192.168.1.1` | Service and version detection |
| `nmap -O 192.168.1.1` | Operating system guess (admin) |
| `nmap -A 192.168.1.1` | -sV + -O + scripts + traceroute in one |
| `nmap -sU --top-ports 20 192.168.1.1` | Top 20 UDP ports (slow) |
| `nmap -T4 ...` | Faster timing (T0 slowest … T5 fastest) |
| `nmap -oN scan.txt ...` | Save the output to a file |
