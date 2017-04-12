# PyPortRedirector
A simple Linux TCP client-server proxy able to redirect all data from one address to another by preserving source IP address written in Python

## About
The client can listen on one or more ports then transfer all data to the server. The server accepts connections from the client(s) on a specified port. On new connections client sends the original source IP address and port with the original destination port to the server. Then server creates iptables specific SNAT and DNAT rules for all connections to be able to return packets on the same route back. This way the service will see the original address as source address and think it communicates with that. From this point both server and client(s) act as a TCP proxy between the peers and the service. When peer disconnects or server stops all related rules are deleted.

Both client and server are asynchronous, so it is very lightweight and has small latency. No threads or forking. Theroretically it can handle 10k peers, but it is not tested (yet).

### What is it good for 

Mainly it is good for if you have services with different default routes than the entry firewall.
 E.g. you have a firewall with ISP1 and another one with ISP2. The service port should be redirected from ISP1, but the machine which has the service application has default route through ISP2. This way DNAT won't work, because TCP packets arrive at the ISP1 firewall, but leave on ISP2. You can create SNAT to make packets back to ISP1, but this way your service application will see ISP1 firewall as source address, not the original sender. This is a problem if you need logging or filtering based on source IP.
 
 Also DNAT/SNAT rules won't work if you have one interface to route. So multi ISP DNAT is not an option.
 
Of course you can use proxy servers for HTTP traffic, where the proxy can set "X-Forwarded-For" like headers. But what about the other kind of services e.g. HTTPS, SMTP, IMAP, etc? PyPortRedirector is working with these protocols too. 
 
You can use SSH tunnel for TCP traffics, but all connections will have the same IP address again. You can try to create VPN, but the same problem is you need to route all traffic back through VPN interface to make DNAT work. But what if you need both ISP...
 
Another use case if you need to make a service available on another machine as well e.g. because you changed your service provider, but you need to serve on the original IP address too. With  
 
## Usage
 
In the example 1.1.1.1 is the IP of machine1 (M1), 2.2.2.2 is machine2 (M2). The server should listen on the service machine on port 12345. The service is on port 80 and 443.

On the server you need to enable local routing, to make DNAT rules originated from localhost work:
```bash
sysctl -w net.ipv4.conf.eth0.route_localnet=1
# Or
# echo "1" >/proc/sys/net/ipv4/conf/eth0/route_localnet
```

Start the client on M1:
```bash
portredirector.py -c 2.2.2.2:10000 -p 80 -p 443
```

Start the server on M2
```bash
portredirector.py -l 0.0.0.0:10000 -p 80 -p 443
```

If your service on M2 is listening on other port (e.g. 10080 and 10443), you can replace ports got from client:
```bash
portredirector.py -l 0.0.0.0:10000 -p 10080 -p 10443 -r 10080.80 -r 10443.443
```

More info is in the help:
```bash
portredirector.py -h
```
Or of course in the source code.

### What about security

PyPortRedirector transfers the same data as arrived on the client part of it. So if the data is encrypted, it will be encrypted as well. But the server needs to know the original sender's IP address. Because of this the 1st packet of every connection will contain these informations. So if a malicious person can access the redirector server, he can easily send spoofed packets and the service will see the spoofed IP address. So he can bypass source IP filters and logging on the service. Of course it can be logged because the server can log the connection.

The solution or best practice is to make firewall rules to make server only communicate with the client(s). For this you need to know the IP address of it.
Another solution is to create an SSH tunnel between the 2 machines. For this you don't need fix client IP.