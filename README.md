# IPv6 on BL602/PineCone Embedded Device 📡

A group research project exploring whether two low-cost PineCone BL602 
boards can communicate with each other using IPv6 only — no routers, 
no gateways, no internet connection. Just two tiny RISC-V boards 
talking directly over Wi-Fi using a protocol most embedded firmware 
doesn't even properly support yet.

Built as part of my Embedded Systems course at Hochschule Schmalkalden.

**Group members:** Mrithu Bashini Bhalkrishnan, Monika Hosadurga 
Thippeswamy, Ameerali Kudukkathu Valappil, Anshul Sharma, Purvika Panwar

---

## What we were trying to do

Configure one BL602 board as a Wi-Fi Access Point and another as a 
Station, then get them to exchange UDP packets over IPv6 only — using 
lwIP and FreeRTOS, without any IPv4 fallback.

Sounds simple. It really wasn't.

---

## What we actually solved

**Phase 1 — Heap exhaustion**
Enabling IPv6 in lwIP adds neighbor cache tables, ICMPv6, multicast 
tracking, and SLAAC state machines. On BL602, this blew past the 
default FreeRTOS heap limit immediately. Fixed by carefully increasing 
heap allocation and tuning lwIP compile-time flags.

**Phase 2 — AP deadlock**
The Access Point initialization was being called inside a Wi-Fi event 
callback. The Wi-Fi manager's internal state machine deadlocked. 
SSID was invisible to other devices even though logs showed AP had 
started. Fixed by moving AP init out of the callback into a separate 
FreeRTOS task.

**Phase 3 — Station disconnection due to IPv4 dependency**
The BL602 Wi-Fi manager only considers a connection valid after 
receiving a WIFI_EVENT_GOT_IP event — which only fires when an IPv4 
address is assigned. In an IPv6-only setup, the station kept getting 
forcibly disconnected even with a fully working IPv6 link-local 
address. Workaround: inject a dummy IPv4 configuration to satisfy 
the state machine.

**Phase 4 — IPv6 UDP routing failure**
Both devices got valid link-local IPv6 addresses. UDP sockets created 
fine. But packets failed at the routing layer with ERR_IF errors. 
Most likely cause: the closed-source Bouffalo SDK Wi-Fi driver doesn't 
correctly handle IPv6 EtherType frames or NDP multicast MAC mapping 
at the link layer. This is documented as an open problem.

---

## What we got working

✅ IPv6 enabled on BL602 via lwIP  
✅ SLAAC link-local address assignment  
✅ Wi-Fi AP + Station connection  
✅ ICMPv6 and Neighbor Discovery operational  
✅ UDP sockets created over IPv6  
⚠️ Full UDP packet exchange blocked at driver/link layer

---

## Stack
Hardware:  PineCone BL602 (RISC-V, Wi-Fi + BT)
RTOS:      FreeRTOS
Network:   lwIP (IPv6, ICMPv6, NDP, SLAAC, UDP)
SDK:       Bouffalo Lab BL602 SDK
Language:  C

---

## How to build
```bash
# Requires Bouffalo Lab BL602 SDK
# https://github.com/bouffalolab/bl_iot_sdk

make
make flash
```

---

## Key insight

Enabling IPv6 on an embedded platform is not just a config flag. 
It touches memory management, firmware state machines, driver 
assumptions, and protocol stack architecture simultaneously. The 
BL602's Wi-Fi driver was written with IPv4 in mind — and that 
assumption runs deep enough to block pure IPv6 operation at the 
link layer even when everything above it is working correctly.

Full report available in `/docs`.
