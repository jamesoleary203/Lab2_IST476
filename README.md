# Configuring an IPsec VPN Server with pfSense

**Course:** IST 476 Applied Information Security, Syracuse University iSchool
**Lab:** Configuring a VPN Server with pfSense (Jones & Bartlett *Ethical Hacking, 4e*)
**Author:** James O'Leary

> **Authorized classroom simulation.** Everything here was configured inside an isolated,
> instructor-provided VMware lab on a private segment. The pfSense firewall, the
> certificate authority, and the `remoteworker01` account are lab fixtures. No production
> VPN, certificate, or credential is involved. The point of the exercise is the chain of
> trust and traffic protection that a remote-access VPN actually rests on.

---

## What this lab covers

Standing up a remote-access VPN end to end on pfSense: building the certificate authority
that anchors trust, issuing the server certificate, negotiating the two IPsec phases for
mobile clients, and writing the firewall rule that lets authenticated VPN traffic through.
The VPN uses IKEv2 with IPsec in tunnel mode, which is the right choice when remote
workers cross the open internet to reach an internal network.

## What was built

| Step | Object created | Purpose |
|---|---|---|
| 1 | CA `IPsecVPN_James` | Root of trust that signs the server certificate |
| 2 | Server cert `IKEv2VPN_pfSense` | Identifies the VPN server to connecting clients |
| 3 | Phase 1 (IKE) mobile-client tunnel | Authenticates the endpoints and sets up the secure channel |
| 4 | Phase 2 (IPsec) entry, tunnel mode | Encrypts the actual traffic through the tunnel |
| 5 | Pre-shared key for `remoteworker01` | Per-user authentication secret |
| 6 | IPsec firewall rule | Permits IPv4 traffic from authenticated VPN users |

## Why it is built this way

**Certificate authority first.** The server certificate is only as trustworthy as whatever
signs it. Creating a dedicated CA and issuing the server cert from it means a client can
verify the server's identity against a known anchor instead of trusting a bare
self-signed cert.

**Two phases, two jobs.** Phase 1 (IKE) handles authentication and builds the encrypted
management channel. Phase 2 (IPsec) carries the user data. Keeping them separate is how
IPsec lets you rekey the data channel without re-authenticating the whole connection.

**Tunnel mode, not transport.** Phase 2 runs in tunnel mode because the remote worker's
traffic crosses untrusted networks. Tunnel mode encrypts the entire original packet,
header included, and wraps it in a new header carrying only the gateway addresses. An
eavesdropper sees traffic between two VPN endpoints, not which internal host the user is
actually reaching.

**The rule comes last for a reason.** Configuring the tunnel does not pass traffic on its
own. Until the IPsec-interface firewall rule exists, authenticated users connect and still
get dropped. The rule is the step that turns a built tunnel into a usable one.

## Research note: Tunnel mode vs transport mode

The two IPsec modes differ in how much of the packet they protect.

- **Tunnel mode** encrypts the entire original IP packet and wraps it in a new header
  carrying the VPN endpoint addresses. It is the standard for site-to-site and
  remote-access VPNs across the internet, it frustrates traffic analysis since an observer
  sees only gateway addresses, and it traverses NAT cleanly because NAT rewrites only the
  outer header (with ESP wrapped in UDP 4500 for NAT traversal).
- **Transport mode** encrypts only the payload and leaves the original IP header in place.
  It fits host-to-host links inside a trusted network and is lighter since it adds no outer
  header, but it exposes source, destination, and timing, and it can break when NAT alters
  the original addresses.

Tunnel mode is the fit for traffic crossing untrusted networks. Transport mode is the fit
for host-to-host links inside a trusted one.

## References

- Barker, E., Dang, Q., Frankel, S., Scarfone, K., & Wouters, P. (2020). *Guide to IPsec VPNs* (NIST SP 800-77 Rev. 1). National Institute of Standards and Technology.
- Kent, S. (2005). *IP Encapsulating Security Payload (ESP)* (RFC 4303). IETF.
- Kent, S., & Seo, K. (2005). *Security architecture for the Internet Protocol* (RFC 4301). IETF.
