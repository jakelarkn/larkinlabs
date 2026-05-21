---
title: "The Purge (home networking edition)"
date: 2026-05-21
draft: true
tags: ["networking", "home-network", "gl-inet", "openwrt"]
---

A few weeks after putting my new home network gateway in place, I logged into the AdGuard Home admin panel to look at what was actually happening on the wire. The query log was supposed to be a curiosity, not a project. One client immediately stood out.

It was making DNS lookups for `log-config.samsungacr.com`, then `config.samsungads.com`, then `events.samsungads.com`, then `amauthprd.samsungcloudsolution.com`, then back around again, faster than once a minute. In the course of a few hours it had racked up several hundred outbound lookups and was climbing toward the low thousands. It was the chattiest device on the network by a wide margin: more than any laptop I was actually using, more than my phone, more than the always-on home server.

It was my Samsung Frame TV. In a room I hardly use. Which I hadn't realized was even powered on.

## Gateway setup

I recently added a GL.iNet Brume 3 to my home network as a dedicated gateway in front of my existing wireless setup. The two TP-Link units I'd been running (one as a router, one as an access point) are now both reconfigured as access points behind the new gateway.

The goal was network-wide filtering (ads, malware, tracking) and finer-grained controls for the devices on the network: scheduling, social-media blocks, focus-friendly defaults. TP-Link does offer some of those features in its stock firmware, but only through a smartphone app tied to a vendor cloud account. I object to that on principle. I want to configure my network from an admin panel like a normal person, not through a phone app talking to someone else's servers.

## What I considered

There are three vendor stories worth knowing for prosumer home networking.

**TP-Link.** Cheap, reliable, fine for a basic router. The web admin panel exposes a narrow configuration surface, and anything more interesting is pushed into the smartphone-app-plus-cloud-account model I complained about above. Some models support OpenWRT, but it isn't a first-class story for the vendor.

**Asus.** Long-time OpenWRT-adjacent platform via Merlin and the stock Asuswrt firmware. The problem is that a lot of the higher-end Asus hardware I looked at felt aging, and the third-party software stories around it were maintained by small teams or had gone quiet. I didn't want to bet a year-long project on community firmware that might lose its maintainer.

**Ubiquiti.** The serious-hobbyist favorite. Well-regarded hardware, an active community, and the polish of a company that started in the enterprise and WISP markets and worked down toward homes. The friction is that a full Ubiquiti setup wants multiple pieces of hardware (a gateway, separate access points, ideally a managed switch), and once you tally them up the total comes out higher than the single-device options I was looking at. The UniFi controller also reads as enterprise software put on a diet: capable, but more ceremony than I want in front of my home network. People who run UniFi tend to be very happy with it. I just didn't want to commit to that surface area.

**GL.iNet.** Chinese hardware, which is a valid thing to think about for a device that sits at the edge of your network. The mitigating factor is the community. GL.iNet is best known for travel routers: small portable WiFi devices that frequent travelers use to layer their own private, consistent network on top of whatever public WiFi they're connected to, often routed through a VPN. The appeal is a trusted network across hotels, cafes, and rented apartments, plus a security boundary between your devices and a connection you don't control. That product line built a deep relationship with the open-source firmware community. GL.iNet builds on OpenWRT, contributes upstream, and the hardware is documented well enough that you can flash vanilla OpenWRT if you want to leave the vendor firmware behind. They publish hardware compatibility tables for specific OpenWRT versions, which is the kind of thing a vendor hostile to community firmware does not do. I went with GL.iNet.

## Brume vs Flint, 2 vs 3

GL.iNet has two relevant product lines: the Brume (compact wired gateways, no WiFi) and the Flint (full-featured WiFi routers). I looked at four candidates, two generations of each:

|                 | Brume 2 (GL-MT2500)           | Brume 3 (GL-MT5000)           | Flint 2 (GL-MT6000)             | Flint 3 (GL-BE9300)             |
|-----------------|-------------------------------|-------------------------------|---------------------------------|---------------------------------|
| SoC             | MediaTek dual-core @ 1.3 GHz  | MediaTek quad-core @ 2.0 GHz  | MediaTek quad-core @ 2.0 GHz    | Qualcomm quad-core @ 1.5 GHz    |
| RAM             | 1 GB DDR4                     | 1 GB DDR4                     | 1 GB DDR4                       | 1 GB DDR4                       |
| Storage         | 8 GB eMMC                     | 8 GB eMMC                     | 8 GB eMMC                       | 8 GB eMMC                       |
| WiFi            | none                          | none                          | WiFi 6 (AX6000)                 | WiFi 7 (BE9300, tri-band)       |
| Ethernet        | 1 × 2.5 GbE + 1 × 1 GbE       | 3 × 2.5 GbE                   | 2 × 2.5 GbE + 4 × 1 GbE         | 5 × 2.5 GbE                     |
| Approx. price   | ~$80                          | ~$130                         | ~$170                           | ~$210                           |

Prices are from the GL.iNet store at the time of writing and tend to drift, so check before buying.

For my use case the wifi-router-versus-gateway split mattered more than any of the spec deltas inside each line. Both Flints would have replaced one of my TP-Link units but forced me to either rip out the access points entirely or run them in some half-disabled mode. The Brume 3 leaves the WiFi side of the network alone. Between the two Brumes, the 3 was worth the extra ~$50 over the 2: twice the CPU cores at a higher clock, every port at 2.5 GbE instead of mixed, and enough headroom to run AdGuard Home plus VPN plus firewall rules at gigabit-plus without hitting a wall.

## Why the gateway

I have two access points wired around the house that already work well. Replacing them with a single WiFi router would have meant either reduced coverage or buying additional GL.iNet APs to maintain it. There are also mixed reports about GL.iNet's own WiFi performance: not common, but real enough that I wasn't eager to put it on the critical path of a working network.

A dedicated gateway sidesteps all of that:

- Existing access points keep their job. They were already doing it well.
- The Brume 3's CPU is entirely available for routing, filtering, and DNS. Nothing is sharing cycles with a WiFi radio.
- Cheaper than the WiFi-bearing Flints, with effectively the same SoC class as the Flint 2.

The downside is that you have two devices to manage instead of one. In practice, the APs are dumb bridges, so the gateway is where everything interesting happens.

## Software: stock firmware and AdGuard Home

GL.iNet's stock web UI is a wrapper over OpenWRT with a friendlier presentation. For everything I wanted (DHCP, static leases, firewall rules, DNS overrides) the stock UI was enough. The interesting addition is **AdGuard Home**, which the firmware bundles as a one-click install. AdGuard Home runs as a network-wide DNS sink that filters ads, trackers, and malware domains using publicly maintained blocklists.

AdGuard's origin story is worth knowing about: it started as a Russian company and now operates out of Cyprus. That doesn't make it bad software, but it's the kind of detail you want to know when something is making DNS decisions for every device in your house. I had originally planned to use NextDNS partly for that reason, plus its nicer profile management (different filtering profiles per device, switchable remotely). What kept me on AdGuard was simply that it was already there, one click away in the firmware. Convenience won out over principle. It's fine for now, and NextDNS remains the upgrade path.

It's worth noting that the DNS-as-filter ecosystem has matured a lot in the last few years. Cloudflare alone now offers multiple flavors of its 1.1.1.1 resolver tuned for different filtering profiles (family safety, malware, adult content blocking), and you can apply them at the resolver level without running anything locally. If you don't want to operate AdGuard Home yourself, that path exists and it's free.

## Configuration

The pattern I've settled on for any kind of per-device policy:

1. **Pin the device to a static local IP via DHCP reservation.** The Brume 3 lets you do this in the LAN settings: match on MAC address, hand it the same IP every time.
2. **Apply rules against the IP or MAC.** Once the device has a stable identity on the network, AdGuard Home filtering, firewall rules, time-of-day access, and DNS profiles can all key off it.

That sounds obvious in retrospect, but it took me a beat to internalize that *everything* downstream is easier once devices have stable identities. Without that, your rules drift every time DHCP shuffles things around.

With identities pinned, the interesting tooling is in AdGuard Home itself. There are several layers, increasing in specificity:

- **Per-client upstream DNS.** Each pinned client can be assigned its own upstream resolver. The kids' devices route through a stricter filtering resolver; the work laptop goes to a faster, less aggressive one.
- **Community blocklists.** AdGuard ships with a starter pack and you can subscribe to the usual suspects: StevenBlack's hosts list, OISD, Perflyst's Smart-TV list, region-specific lists, ad-network-specific lists. These update automatically and do most of the heavy lifting.
- **Custom domain block rules.** Anything the lists don't catch can be added by hand, either as an outright block or as a regex or wildcard rule. Useful for one-off devices that talk to weird domains nothing else does.
- **Service-level blocks with schedules.** AdGuard has a built-in catalog of common services (Facebook, Instagram, YouTube, TikTok, Reddit, and a long tail of others). You can toggle each one on or off per client and attach a time-of-day schedule.

The service-block scheduler is the feature I most wanted, and also the one that doesn't quite land the way I'd hoped. The UI lets you define a single blocked or unblocked window per day per client. What I actually want is a workday-only block: bypass in the morning so I can read the news over coffee, blocked through the workday, then bypass again in the evening for the unprincipled bedtime scroll. That's two on/off transitions per day, not one, and there's no clean way to express it in the current UI.

My workaround is a manual juggle of rules and schedules. It works, but it's clumsy enough that I update it less often than I should. If AdGuard adds multi-window schedules I'll redo it properly. Until then, the rough version is in place and does most of what I wanted.

## The Frame TV, revisited

Back to the room I don't use.

My first move was modest. I pinned the Frame to its own DNS profile pointed at a stricter upstream resolver, one of the Cloudflare family-safety endpoints, on the theory that the manufacturer's chattiness was a known quantity and someone else's blocklist would handle most of it. The query count barely budged.

Next I added Perflyst's Smart-TV blocklist and a couple of Samsung-specific lists to AdGuard Home, applied to the TV's static IP. That cut a real fraction of the traffic. Telemetry endpoints went dark, ad-network calls stopped resolving, and the device settled into a less feverish rhythm. But it was still talking to Samsung more than I talk to most of my own services.

Eventually I gave up trying to negotiate and cut its network access entirely at the gateway. The TV still plays from local sources fine. Nothing I actually use it for required the connection. I'd had enough of a television I barely watch chatting up the internet on my behalf.
