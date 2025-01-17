# Quick Introduction

Multicast DNS and DNS service discovery daemons deployed on various systems across the Internet are misconfigured and reply
to queries targeting their unicast addresses, including requests from their WAN interface.
These daemons could be leveraged by attackers for sensitive information disclosure and potentially
used in DDoS campaigns for reflection and in some cases amplification.

This vulnerability was made public in cordination with CERT (http://www.kb.cert.org/vuls/id/550620)

## mDNS

> Multicast DNS and its companion technology DNS-Based Service
> Discovery [RFC6763] were created to provide IP networking with the
> ease-of-use and autoconfiguration for which AppleTalk was well-known
> [RFC6760].  When reading this document, familiarity with the concepts
> of Zero Configuration Networking [Zeroconf] and automatic link-local
> addressing [RFC3927] [RFC4862] is helpful.

> Multicast DNS (mDNS) provides the ability to perform DNS-like
> operations on the local link in the absence of any conventional
> Unicast DNS server.  In addition, Multicast DNS designates a portion
> of the DNS namespace to be free for local use, without the need to
> pay any annual fee, and without the need to set up delegations or
> otherwise configure a conventional DNS server to answer for those
> names.

> In specialized applications there may be rare situations where it
> makes sense for a Multicast DNS querier to send its query via unicast
> to a specific machine.  When a Multicast DNS responder receives a
> query via direct unicast, it SHOULD respond as it would for "QU"
> questions, as described above in Section 5.4.  Since it is possible
> for a unicast query to be received from a machine outside the local
> link, responders SHOULD check that the source address in the query
> packet matches the local subnet for that link (or, in the case of
> IPv6, the source address has an on-link prefix) and silently ignore
> the packet if not.

quotes taken from [Multicast DNS - IANA RFC6762](http://tools.ietf.org/html/rfc6762)

## Who, What, Where?

mDNS is a lot like SSDP (the reason I decided to test it in the first place) in the sense that it 
allows devices on a local network to easily discover one another, exchange information about 
available services, and reduce or elminate configuration needs. Because these are the primary uses 
for mDNS today it can be found across a wide range of consumer devices such as phones, printers, NAS systems, 
media devices, etc.  There are also daemons available for all major OS's (Windows, OS X, and Linux).  
Notable uses of mDNS include ZeroConf networking, Apple's Bonjour & AirDrop, and ChromeCast, none of which 
appeared vulnerable in my testing (with the exception of a single MacBook Air found in the wild).  Not all 
implementations are vulnerable over the Internet, but some that are exist on embedded devices that are unlikely to
recieve updates from their respective vendors or users.

## What's out there?

As part of the initial research into the impact of these vulnerable daemons broad scans were ran that identified 
over 100,000 devices that replied to mDNS queries over the internet.  These devices include several NAS boxes and printers
as well as Windows and Linux machines.  Some of these machines were located on larger networks such as corporations and
universities, and appeared to be poorly secured, if secured at all.  Some vendors have already stated they will not be
fixing this issue on older devices that are currently vulnerable in the wild.

## How do you query these machines?

When I initially started the research my first tests used `dig` to confirm a reply could be triggered.

```
$ dig +short @[target_ip] -p 5353 -t any _services._dns-sd._udp.local
_workstation._tcp.local.
_http._tcp.local.
_afpovertcp._tcp.local.
_device-info._tcp.local.
_dacp._tcp.local.
_daap._tcp.local.
```

Once I confirmed there were some machines out in the wild I wrote a quick and dirty Scapy script that I could use to
easily automate my testing going forward as well as test viable reflection in a lab enviornment.  I've removed 
the reflection pieces (for reasons that should be obvious) and am releasing the recon portion which I've 
named `mdns_recon`, which we'll cover below shortly.

## Information disclosure

The replies from mDNS queries vary a good bit, in some cases they won't leak much if any usable information
in other cases they can leak sensitive details such network information, administration information, and
device information such manufacturer, model, serial number, etc.  Below are some examples of devices found in the 
wild leaking some of the aforementioned examples.  I have cleaned up the responses for easier reading and to conserve space.

### hostnames & MACs
```
inventory [00:18:51:2a:a2:0d]
administrator-X8DT6 [00:25:90:38:6c:c4]
ispadmin [00:0c:29:5d:08:2f]
vpsadmin [00:16:3e:d1:61:50]
```

### networking details
```
0 0 548 KenNas.local.
fe80::211:32ff:fe13:7de7
192.168.1.110
```

### NAS leaking details, Password=false
```
DiskStation(WebDAV)
vendor=Synology model=DS114 serial=[REMOVED]
version_major=5 version_minor=1 version_build=5021
admin_port=5000 secure_admin_port=5001 
mac_address=00:11:32:2A:97:9D
model=Xserve
txtvers=1 Database ID= Machine ID= Machine Name=DiskStation
mtd-version=0.2.4.1 
iTSh Version=131073
Version=196610
Password=false
```

### Printers
```
Phaser 8560DN (00:00:aa:d4:b8:2b)
test-178307
adminurl=http://test-178307.local.
qtotal=1 priority=80 txtvers=1
product=(Phaser 8560DN) ty=Xerox Phaser 8560DN
Bind=F Copies=T Sort=F Binary=T Transparent=T TBCP=F Staple=F Color=T Duplex=T Collate=T PaperCustom=T
```

## DDoS reflection and amplification

Since mDNS uses UDP, all of hosts that reply could potentially be used by malicious
actors as reflectors in DDoS campaigns.  There is also the potential for amplification since replies
tend to be larger than the initial queries.  An attacker can expect at least a 1:1 reflection, 
in some of my testing, some services amplified by as much as 975%.  The true amplification rate is
hard to predict since the replies vary a lot based on server configuration and the size of the 
query packet itself, which changes based on the service being queried, but a safe estimate would be 
around 130%+ amplifcation on average.

# Mitigation

## As a reflector

Block UDP traffic destined for port 5353.

## As a target

Block UDP traffic with sourcing from port 5353.

# mdns_recon.py

`mdns_recon.py` is a quick and dirty Scapy script that was used in my testing to identify mDNS clients
in the wild that would reply to unicast queries directed at them.  Since it's goal was to quickly test
and log these replies and their respective lengths for research purposes, that is literally all it does.
If you want to add features, you found a bug, etc. fork away, pull requests will gladly be accepted if
they're warranted.

If you want pretty replies that will please your human brain, I suggest you utilize `dig`, but if you're
doing large scale automated scanning `mdns_recon` can be used with a multi-threaded middle layer.

## scanning a target
```
    # ./mdns_recon.py [target]
```

### example output
```
TARGET_IP - START
[_services._dns-sd._udp.local]===
E  «ìÍ  íè.À¨é 5 Kn         	_services_dns-sd_udplocal   À     
 _pdl-datastream_tcpÀ#À     
 _printerÀJÀ     
 _ippÀJÀ     
 _httpÀJ
[_services._dns-sd._udp.local]===

[_pdl-datastream._tcp.local.]===
E ¿ìÎ  íÓ.À¨é 5«Í8        _pdl-datastream_tcplocal   À     
 RICOH IPSiO SP C810ÀÀ8 !    
     #	RNPBC3B0AÀ!À8     
	txtvers=1qtotal=1pdl=application/postscript!product=(RICOH IPSiO SP C810 PS3)ty=RICOH IPSiO SP C810note=PaperMax=legal-A4priority=10 adminurl=http://RNPBC3B0A.local/Bind=FColor=T	Collate=FCopies=TDuplex=FPaperCustom=FSort=FStaple=FPunch=FBinary=TTransparent=FTBCP=TÀ`     
 .
[_pdl-datastream._tcp.local.]===

[_printer._tcp.local.]===
E ÈìÏ  íÉ.À¨é 5´¿        _printer_tcplocal   À     
 RICOH IPSiO SP C810ÀÀ1 !    
     	RNPBC3B0AÀÀ1     
+	txtvers=1qtotal=1rp=filetype_RPSpdl=application/postscript!product=(RICOH IPSiO SP C810 PS3)ty=RICOH IPSiO SP C810note=PaperMax=legal-A4priority=20 adminurl=http://RNPBC3B0A.local/Bind=FColor=T	Collate=FCopies=TDuplex=FPaperCustom=FSort=FStaple=FPunch=FBinary=TTransparent=FTBCP=TÀY     
 .
[_printer._tcp.local.]===

[_ipp._tcp.local.]===
E ¿ìÐ  íÑ.À¨é 5«ä        _ipp_tcplocal   À     
 RICOH IPSiO SP C810ÀÀ- !    
     w	RNPBC3B0AÀÀ-     
&	txtvers=1qtotal=1
rp=printerpdl=application/postscript!product=(RICOH IPSiO SP C810 PS3)ty=RICOH IPSiO SP C810note=PaperMax=legal-A4priority=30 adminurl=http://RNPBC3B0A.local/Bind=FColor=T	Collate=FCopies=TDuplex=FPaperCustom=FSort=FStaple=FPunch=FBinary=TTransparent=FTBCP=TÀU     
 .
[_ipp._tcp.local.]===

[_http._tcp.local.]===
None
[_http._tcp.local.]===

{'_printer._tcp.local.': 456, '_services._dns-sd._udp.local': 171, '_pdl-datastream._tcp.local.': 447, '_ipp._tcp.local.': 447}

TARGET_IP - END
```

### Scanning a range
```
# for oct in {1..254}; do echo "192.32.68.$oct"; done | xargs -I{} ./mdns_recon.py {}

192.32.68.4 - START
[_services._dns-sd._udp.local]===
E  fN  ñr DÀ¨u    E  J   1PÀ¨r D 5é 6:ø          	_services_dns-sd_udplocal   
[_services._dns-sd._udp.local]===
{'_services._dns-sd._udp.local': 102}
192.32.68.4 - END

192.32.68.6 - START
[_services._dns-sd._udp.local]===
E  fH¨  3Æzr DÀ¨u    E  J   1NÀ¨r D 5é 6:ö          	_services_dns-sd_udplocal   
[_services._dns-sd._udp.local]===
{'_services._dns-sd._udp.local': 102}
192.32.68.6 - END

192.32.68.8 - START
[_services._dns-sd._udp.local]===
E  fe  3ªr DÀ¨u    E  J   1LÀ¨r D 5é 6:ô          	_services_dns-sd_udplocal   
[_services._dns-sd._udp.local]===
{'_services._dns-sd._udp.local': 102}
192.32.68.8 - END

192.32.68.10 - START
[_services._dns-sd._udp.local]===
E  fa/  3­ïr D
À¨u    E  J   1JÀ¨r D
 5é 6:ò          	_services_dns-sd_udplocal   
[_services._dns-sd._udp.local]===
{'_services._dns-sd._udp.local': 102}
192.32.68.10 - END
```

### Scanning a range to file
```
# for oct in {1..254}; do echo "192.32.68.$oct"; done | xargs -I{} ./mdns_recon.py {} >> mdns_replies.txt
```
