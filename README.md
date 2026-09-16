# PENETRATION TESTING REPORT
### Footprinting & Network Scanning Phases

**W2-PM  |  CYBERSECURITY  |  NETWORKWALKS**

| Field | Detail |
|---|---|
| **Pentester Name (Cybersecurity Professional)** | **Alale Matthew** |
| **Program/Batch** | B083-Networkwalks |
| **Date** | 16 September 2026 |
| **Modules completed** | W2-PM1 (Multiple Kali Tools)<br>W2-PM1 (Footprinting & Reconnissance Attacks with multiple Kali Tools)<br>W2-PM5 (Zenmap Scanning) |
| **Client/Target** | 1. networkwalks.com (secured written permission already)<br>2. My own local Wi-Fi hotspot network |
| **Permission secured from client?** | Yes |
| **Phases covered** | **Phase 1:** Reconnaissance & Footprinting<br>**Phase 2:** Scanning & Network Discovery<br>**Phase 3-5:** In Progress |

# 1. Liability Disclaimer

Every task in this report was carried out either against `networkwalks.com`, for which the internship program already has documented written approval, or against a Wi-Fi network that belongs to me and that I administer myself. Nothing here is intended for anything beyond learning and personal skill-building. This document should not be used to target a system that has not been authorised for testing, that responsibility sits entirely with whoever chooses to do so, not with Networkwalks, the instructors, or me. Unauthorised access carries real legal consequences in most jurisdictions, whether or not any actual damage occurs.

# 2. Introduction

This report documents the second week of my cybersecurity internship at Networkwalks, split across two hands-on exercises. The first (W2-PM1) walks through footprinting the `networkwalks.com` domain with a set of Kali Linux command-line utilities. The second (W2-PM5) walks through scanning a network I control using Zenmap, the graphical front end for Nmap. Put together, the two exercises trace the path an attacker typically follows early in an engagement, starting with information that's already public, then moving on to actively probing which hosts and services are reachable.

Everything below was executed from a Kali Linux terminal. For each tool I've noted the exact command, what came back, a screenshot of the actual session, and a brief read on why that particular piece of information would matter to someone trying to profile this target.

# 3. Tools Used

| Tool | Purpose |
|---|---|
| Kali Linux | The OS the entire footprinting phase was carried out on. |
| whois | Pulls public registration data for a domain — registrar, key dates, name servers. |
| whatweb | Fingerprints the technology stack a website is built on. |
| nslookup | Resolves a hostname to its IP address via DNS. |
| curl -I | Pulls just the HTTP response headers without downloading the page body. |
| wafw00f | Checks whether a site sits behind a Web Application Firewall, and which one. |
| dnsrecon | Sweeps a domain for its DNS records — SOA, NS, MX, TXT, SRV, etc. |
| Zenmap (Nmap GUI) | Graphical Nmap front end, used here to sweep a subnet for live hosts and open ports. |
| **Terminal (MacOS)** | Local IP and MAC address identification. |

# 4. Activities Performed

## 4.1 Footprinting & Reconnaissance

I ran six different Kali tools against `networkwalks.com` — WHOIS, WhatWeb, Nslookup, curl, Wafw00f and DNSRecon, each one aimed at a different layer of the target's public footprint, from who owns the domain down to what's actually running on the server.

**WHOIS** came first, since domain registration data is usually the easiest starting point. It showed the domain sitting with **GoDaddy.com, LLC**, registered on **6 November 2019** and due to expire **6 November 2027**, last updated **12 November 2025**. The record carries the four standard client-side locks (`clientTransferProhibited`, `clientUpdateProhibited`, `clientRenewProhibited`, `clientDeleteProhibited`), so the domain itself is reasonably well protected against hijacking. The actual hosting runs through HostGator's name servers (`NS6135`/`NS6136.HOSTGATOR.COM`) rather than GoDaddy's own, and DNSSEC is not enabled.

Next was **WhatWeb**, to see what the site itself is built on. It flagged **WordPress 7.1** running the **WordPress Download Manager 3.3.58** plugin, sitting on an Apache server, with Bootstrap 7.1 and jQuery 3.7.1 loaded on the front end and it confirmed the hosting IP in the process.

**Nslookup** was a quick check against that IP, confirming the domain resolves to **192.232.216.135**.

Running **curl -I** against the site surfaced the raw response headers, which is where the WordPress REST API endpoint (`/wp-json/`) and a direct link to page ID 53 of that API showed up in the `Link` header, along with a `__wpdm_client` session cookie (marked `Secure` and `HttpOnly`) and a fairly strict `permissions-policy` header limiting Private State Token redemption/issuance to five named domains — Google, Google's static CDN, reCAPTCHA, Cloudflare Turnstile and hCaptcha. The `referrer-policy` was set to `no-referrer-when-downgrade`, and the caching headers (`x-endurance-cache-level: 0`, `x-nginx-cache: WordPress`) confirmed WordPress-aware edge caching is active.

**Wafw00f** was used to check for a firewall in front of the application, and it came back positive, the site is sitting behind **ModSecurity (SpiderLabs)**.

Last was **DNSRecon**, which gave the fullest picture: SOA and NS records (both name servers reporting BIND `9.16.23-RH`), the A record confirming `192.232.216.135`, an MX record pointing `mail.networkwalks.com` to that same IP, an SPF record in the TXT set (`v=spf1 +a +mx +ip4:50.87.144.87 +include:websitewelcome.com ~all`), a Google site-verification token, and eight SRV records for `_autodiscover._tcp`, all resolving to `cpanelemaildiscovery.cpanel.net` across six IP addresses in the `184.94.203.x` and `184.94.204.x` ranges on port 443, confirming mail client autodiscovery is fully outsourced to cPanel's hosted service.

## 4.2 Network Scanning with Zenmap

The second half of the week moved from passive lookups to actively scanning a live network, in this case, my own Wi-Fi hotspot rather than an organisational LAN.

I pointed Zenmap at `172.20.10.0/28` and ran it with the **Quick scan** profile, which under the hood executes `nmap -T4 -F 172.20.10.0/28`.

Out of the 16 addresses in that range, three hosts answered:

- `172.20.10.1` — open on `21/tcp` (ftp), `53/tcp` (domain) and `49152/tcp`; MAC `8A:64:40:30:87:64`
- `172.20.10.2` — open on `49152/tcp` only; MAC `A2:65:72:92:85:B0`
- `172.20.10.5` — open on `49157/tcp` only — this is the scanning machine itself

The scan wrapped up in 2.68 seconds (16 addresses scanned, 3 hosts up). Switching to the **Topology** tab, the Fisheye layout put the scanning host at the centre, with `172.20.10.2` and `172.20.10.5` drawn as green nodes since they had the fewest open ports, while `172.20.10.1` showed up yellow, reflecting the extra ports it had exposed compared to the other two.

# 5. Risk Analysis / Impact

Pulling together what each tool surfaced, here's how I'd rate the exposure:

| **#** | **Finding** | **Evidence / Observation** | **Potential Impact** | **Risk Level** |
|---|---|---|---|---|
| 1 | CMS and plugin version exposed | WhatWeb fingerprinted WordPress 7.1 and WordPress Download Manager 3.3.58 from the page's generator meta tag and asset query strings. | An attacker can cross-reference these exact version numbers against WPScan/CVE databases for known, unpatched file-download or access-control flaws in that plugin release. | **● Medium** |
| 2 | Hosting IP address exposed | Nslookup resolved `networkwalks.com` to `192.232.216.135`. | Gives an attacker a concrete address to run further port scans or vulnerability scans against, bypassing any need to guess the hosting provider. | **● Low** |
| 3 | REST API endpoint and cookie details visible in headers | curl -I revealed the `/wp-json/` discovery link (with a direct page-53 reference) and a Secure/HttpOnly `__wpdm_client` session cookie. | Makes it easier to map out available REST routes and session behaviour before attempting any application-layer testing. | **● Low** |
| 4 | WAF product identifiable | wafw00f confirmed ModSecurity (SpiderLabs) is in front of the site after 2 requests. | Knowing the specific WAF engine lets an attacker pull up public bypass techniques written specifically for ModSecurity's rule set. | **● Low** |
| 5 | Full DNS/mail footprint exposed | DNSRecon pulled SOA, NS, A, MX, SPF/TXT and eight SRV Autodiscover records spanning six cPanel IP addresses. | Builds a near-complete map of the domain's hosting and mail infrastructure, including a third-party SPF include (`websitewelcome.com`) worth verifying. | **● Medium** |
| 6 | Name-server software version disclosed | Both authoritative servers (`192.232.216.131` and `50.87.144.87`) reported BIND version `9.16.23-RH`. | A precise version banner can be checked directly against the CVE list for that BIND release without any further probing. | **● Medium** |
| 7 | Extra services open on a hotspot host | Zenmap found 3 live hosts on `172.20.10.0/28`; `172.20.10.1` alone had FTP (21) and DNS (53) open alongside a shared high port. | An open, possibly forgotten FTP service on a personal device can allow unauthenticated file access if it isn't actually needed or secured. | **● Medium** |

**Risk level key:** ● Critical ● Medium ● Low

None of the items above were exploited or confirmed as actual vulnerabilities, this was purely an information-gathering and host-discovery exercise. A version number, an open port, or a DNS record on its own doesn't prove a system is exploitable; it just narrows down where a deeper, authorised test would need to look.

# 6. Recommendations

1.  **Strip version detail out of what the stack advertises**
    WhatWeb only found the WordPress 7.1 and WP Download Manager 3.3.58 version numbers because WordPress prints them straight into the page's meta generator tag and into script/style query strings by default. Removing the generator tag (a one-line filter in `functions.php`: `remove_action('wp_head','wp_generator')`) and stripping version query strings from enqueued assets would mean a casual WhatWeb-style scan no longer hands over exact version numbers for free.

2.  **Patch WordPress core and the Download Manager plugin on a schedule, not reactively**
    WordPress Download Manager has had multiple file-download and access-control CVEs in past releases, so running 3.3.58 without checking it against the plugin's own changelog and the WPScan vulnerability database is a gap. A monthly check of both WordPress core and this plugin's changelog, tested on staging before pushing to `networkwalks.com`, would close that gap.

3.  **Trim what the response headers give away**
    The `Link` header currently exposes the full REST API discovery URL and a direct link to page 53 to anyone running `curl -I`. Since the site doesn't appear to need public REST discovery for logged-out visitors, adding `remove_action('wp_head','rest_output_link_wp_head')` would drop that line from the headers. It would also be worth moving the `referrer-policy` from `no-referrer-when-downgrade` to the stricter `strict-origin-when-cross-origin`, since the current setting still leaks the full referring URL over HTTPS-to-HTTPS navigation.

4.  **Re-check the DNS and mail records against what's actually in use**
    The SPF record currently authorises both `+ip4:50.87.144.87` and `+include:websitewelcome.com` to send mail as `networkwalks.com`, worth confirming both are still legitimate senders, since a stale SPF include is a common way spoofed mail slips through. The Google site-verification TXT record and the eight cPanel Autodiscover SRV records should also be reviewed periodically to confirm they still point at services actually in use.

5.  **Hide the BIND version banner on both name servers**
    DNSRecon read off BIND `9.16.23-RH` directly from `ns6135` and `ns6136.hostgator.com`. Adding a `version none;` (or a decoy string) line to `named.conf`'s options block stops that banner from being handed out on request. This is a hardening step, not a patch — the actual BIND release should still be checked against the current CVE list before assuming it's fully up to date.

6.  **Confirm the WAF is actually stopping attacks, not just present**
    wafw00f only confirms that ModSecurity (SpiderLabs) is sitting in front of the site, it doesn't confirm the rule set is current or that it blocks anything. Turning on ModSecurity's audit log, reviewing blocked requests weekly, and periodically running a handful of known-bad payloads (basic SQLi/XSS strings from the OWASP Core Rule Set test suite) against a staging copy would confirm the WAF is doing more than just existing.

7.  **Turn the Zenmap sweep into a repeatable weekly check**
    The quick scan (`nmap -T4 -F`) only checks the top 100 ports, so a slower, more thorough scan (`nmap -sV -p-`) run occasionally would catch anything a fast sweep misses. Keeping a simple baseline of the MAC addresses that are expected to show up (`8A:64:40:30:87:64`, `A2:65:72:92:85:B0` and the scanning host itself) makes it obvious the moment a new, unrecognised device joins the hotspot.

8.  **Work out why 172.20.10.1 has FTP and DNS open**
    `172.20.10.1` was the only host of the three exposing FTP (21/tcp) and DNS (53/tcp) on top of the shared high port, that's not typical for a phone acting purely as a hotspot client. It's worth checking whether that device is running a file-sharing or FTP server app that can be switched off, and confirming the DNS service isn't silently proxying or altering lookups for other devices on the same hotspot.

9.  **Keep a running log instead of a one-off scan result**
    Saving the Zenmap output (hosts, ports, MAC addresses) and the Topology graphic each time a scan is run — the way Section 8 of this report already does for this one, turns individual scans into a timeline. That makes it far quicker to spot when something has changed, rather than relying on memory of what a network looked like last time.

10. **Keep every test inside the written scope**
    Everything in this report stayed inside two clear boundaries: `networkwalks.com`, which already has documented internship permission, and my own `172.20.10.0/28` hotspot, which I own outright. Any future testing should start the same way, a written scope confirming the exact domain or IP range covered, before a single command runs, since a hotspot subnet in particular can occasionally overlap with a neighbouring device that isn't mine to test.

# 7. Conclusion

My week 2 gave me a hands-on run-through of the two things that typically kick off a security assessment: gathering what's publicly available about a target, and then actively scanning to see what's reachable on a network.

The footprinting half showed how much can be pieced together without touching the target directly, WHOIS for ownership and hosting history, WhatWeb for the technology stack, Nslookup for the resolving IP, curl for what the server volunteers in its headers, wafw00f for the firewall sitting in front, and DNSRecon for the full DNS and mail picture. None of it required exploitation, just careful reading of what each tool returned.

The Zenmap half shifted from reading to probing: sweeping my own hotspot subnet turned up three live devices, their open ports, and their MAC addresses, and the topology view made it easy to see at a glance which host was carrying more exposed services than the others.

If there's one takeaway from the week, it's that documentation matters as much as the technical work itself, a finding is only useful if it's written down clearly enough that someone else (or future me) can see what was run, what came back, and what it actually means in terms of risk. Everything here stayed within the scope I was authorised for: a domain with permission on file, and a network that's mine.

# 8. Evidences Collected

*Kali Linux Terminal and Zenmap GUI screenshots captured during testing*

**whois networkwalks.com**
![whois output](Screenshot%202026-09-14%20at%2010.21.43%20PM.png)

**whatweb networkwalks.com**
![whatweb output](Screenshot%202026-09-14%20at%2010.27.55%20PM.png)

**nslookup networkwalks.com**
![nslookup output](Screenshot%202026-09-14%20at%2010.28.42%20PM.png)

**curl -I https://networkwalks.com**
![curl output](Screenshot%202026-09-14%20at%2010.35.53%20PM.png)

**wafw00f networkwalks.com**
![wafw00f output](Screenshot%202026-09-14%20at%2010.38.25%20PM.png)

**dnsrecon -d networkwalks.com**
![dnsrecon output](Screenshot%202026-09-14%20at%2010.40.00%20PM.png)

**Zenmap — nmap -T4 -F 172.20.10.0/28 (Nmap Output)**
![Zenmap scan output](Screenshot%202026-09-15%20at%204.23.37%20PM.png)

**Zenmap — Network Topology**
![Zenmap topology](Screenshot%202026-09-15%20at%204.27.03%20PM.png)

### Author
**Alale Matthew**
Cybersecurity Professional (Intern)

### Project Information
**Program Name:** Cybersecurity Program at Networkwalks | **Week:** 02 | **linkedIn:** www.linkedin.com/in/matthewalale
