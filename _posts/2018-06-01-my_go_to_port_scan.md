---
layout: post
title: "My go to full range scan"
---

## Intro
Recently I have been using a specific scan for my initial checks on a target system. The method I use scans the entire range of ports on both the TCP and UDP channels. As it has helped me out so much, I thought I would share.

Disclaimer: This technique is most likely known and used by many, but fingers crossed some viewers will find use for it! I have only ever performed this within CTF’s/lab environments, please be sure to understand your target environment before firing off scans blindly.

## The command:

```
nmap -sT -sU --min-rate 5000 --max-retries 1  -p- <target>
```

## Arguments:

* -sT : TCP Connect() technique
* -sU : enable UDP scanning
* --min-rate : sets the slowest amount packets that can be sent each second
* --max-retries : limits the number of port scan probe retransmissions
* -p- : Specifies to scan the entire port range (1 to 65535)

To view all other possibilities, check out nmap’s man page.

## Benefits:

* Very quick scan
* Probes every possible port on the target, not just the top 1000 common ports like other scans
* Can help identify obscure ports that are being utilised by services

## Detriments:

* Not the quiestest traffic (see below for test results)
* Can produce results that misses a port that is found in another scan

## Tests:

In order to help understand one of the detriments I listed above, I wanted to perform a couple of nmap scans against a chosen target, on a safe network. I used iptables to gather the results. In order to make sure the traffic was gathered during a scan, I configured iptables using the following set of commands.

```
root@kali:~/Documents# iptables -I INPUT 1 -s 10.11.1.5 -j ACCEPT
root@kali:~/Documents# iptables -I OUTPUT 1 -d 10.11.1.5 -j ACCEPT
root@kali:~/Documents# iptables -Z
```

The main focus of the tests was to gain an understanding of the amount of bytes that were sent across the network during a scan, relative to the time it took to complete and the ports that were found during a scan. I made sure to clear the obtained data before each different scan (-Z argument).

Test 1:

My chosen technique, scanning the entire port range across both TCP and UDP channels.

```
root@kali:~/Documents# nmap -sT -sU --min-rate 5000 --max-retries 1  -p- 10.11.1.5
Not shown: 61350 closed ports, 57737 filtered ports, 11980 open|filtered ports
PORT    STATE SERVICE
135/tcp open  msrpc
139/tcp open  netbios-ssn
135/udp open  msrpc
MAC Address: 00:50:56:89:70:15 (VMware)
Nmap done: 1 IP address (1 host up) scanned in 51.75 seconds
```

Results:

```
root@kali:~/Documents# iptables -vn -L
Chain INPUT (policy ACCEPT 201K packets, 29M bytes)
pkts bytes target     prot opt in     out     source               destination
201K 9725K ACCEPT     all  ---  *      *       10.11.1.5            0.0.0.0/0
0     0 ACCEPT     all  ---  *      *       10.11.1.5            0.0.0.0/0
0     0 ACCEPT     all  ---  *      *       10.11.1.5            0.0.0.0/0
Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
pkts bytes target     prot opt in     out     source               destination
Chain OUTPUT (policy ACCEPT 254K packets, 33M bytes)
pkts bytes target     prot opt in     out     source               destination
254K   11M ACCEPT     all  ---  *      *       0.0.0.0/0            10.11.1.5
0     0 ACCEPT     all  ---  *      *       0.0.0.0/0            10.11.1.5
```

Test 2:

Notes: Default nmap scan, top 1000 common ports

```
root@kali:~/Documents# nmap 10.11.1.5
Starting Nmap 7.60 ( https://nmap.org ) at 2018-05-26 16:29 BST
Nmap scan report for 10.11.1.5
Host is up (0.13s latency).
Not shown: 997 closed ports
PORT     STATE SERVICE
135/tcp  open  msrpc
139/tcp  open  netbios-ssn
3389/tcp open  ms-wbt-server
MAC Address: 00:50:56:89:70:15 (VMware)
Nmap done: 1 IP address (1 host up) scanned in 7.11 seconds
```

Results:

```
root@kali:~/Documents# iptables -vn -L
Chain INPUT (policy ACCEPT 1378 packets, 205K bytes)
pkts bytes target     prot opt in     out     source               destination
1185 47980 ACCEPT     all  ---  *      *       10.11.1.5            0.0.0.0/0
0     0 ACCEPT     all  ---  *      *       10.11.1.5            0.0.0.0/0
0     0 ACCEPT     all  ---  *      *       10.11.1.5            0.0.0.0/0
Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
pkts bytes target     prot opt in     out     source               destination
Chain OUTPUT (policy ACCEPT 1302 packets, 167K bytes)
pkts bytes target     prot opt in     out     source               destination
1283 56440 ACCEPT     all  ---  *      *       0.0.0.0/0            10.11.1.5
0     0 ACCEPT     all  ---  *      *       0.0.0.0/0            10.11.1.5
```

Test 3:

```
root@kali:~/Documents# nmap -sT 10.11.1.5
Starting Nmap 7.60 ( https://nmap.org ) at 2018-05-26 17:43 BST
Nmap scan report for 10.11.1.5
Host is up (0.12s latency).
Not shown: 998 closed ports
PORT    STATE SERVICE
135/tcp open  msrpc
139/tcp open  netbios-ssn
MAC Address: 00:50:56:89:70:15 (VMware)
Nmap done: 1 IP address (1 host up) scanned in 20.35 seconds
```

Results:

```
root@kali:~/Documents# iptables -vn -L
Chain INPUT (policy ACCEPT 1264 packets, 184K bytes)
pkts bytes target     prot opt in     out     source               destination
1199 48576 ACCEPT     all  ---  *      *       10.11.1.5            0.0.0.0/0
0     0 ACCEPT     all  ---  *      *       10.11.1.5            0.0.0.0/0
0     0 ACCEPT     all  ---  *      *       10.11.1.5            0.0.0.0/0
Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
pkts bytes target     prot opt in     out     source               destination
Chain OUTPUT (policy ACCEPT 1206 packets, 175K bytes)
pkts bytes target     prot opt in     out     source               destination
1200 71968 ACCEPT     all  ---  *      *       0.0.0.0/0            10.11.1.5
0     0 ACCEPT     all  ---  *      *       0.0.0.0/0            10.11.1.5
```

## Conclusion and my thoughts:
After completing the above tests, I was able to view the amount of traffic that was sent over the network using my chosen technique compared to the others. Yes, lots of data was sent across to complete the task, however scanning results were presented in just under 52 seconds. Personally, if I know the environment can handle this scan – I will happily use it as a starting point.

In this case, the entire port range scan didn’t pick up any obscure ports on the system – however it did miss a service port that was found within the default scan. As I mentioned before, it can sometimes return incorrect or misleading data but if the system had a couple of services running on higher, uncommon ports – the chances of spotting them with this scan are much higher than the other tested methods. I would also accompany this scan with additional scans, to help back these results up however the scope of ports to check will hopefully be narrowed down by this point.

I would be interested to hear feedback about this technique and whether or not there are further ways to increase the reliability of the results as well as maintaining the speed. Let me know your thoughts on twitter.

Thanks for reading.
