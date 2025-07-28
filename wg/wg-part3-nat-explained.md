# WireGuard Unlocked (Part 3): When and Why to Use NAT

I wasn’t planning to write a dedicated post about NAT — because in most situations, it’s simply not needed.

But when I was learning WireGuard myself, I ran into a flood of tutorials that used NAT without explaining why. Instead of helping, they often created confusion. After reviewing many of these setups, I realized NAT was being applied unnecessarily — sometimes even interfering with how WireGuard is supposed to work.

## What’s Important?

The most critical concept to understand is **AllowedIPs**. This single setting controls both routing and access in WireGuard.

- For **outgoing** traffic: the destination IP must match an entry in `AllowedIPs`.
- For **incoming** traffic: the source IP must match an entry in `AllowedIPs`.

Because of this, `AllowedIPs` must be carefully mirrored on both peers. Whether you like this design or not, you cannot override or bypass it using traditional routing or NAT techniques.

## So, Why NAT at All?

The most commonly used form of NAT is **source NAT (SNAT)** with port overloading — also known as **masquerading**. This is the default in most home routers, where private IPs (e.g., `192.168.1.x`) are translated into a single public IP from the ISP.

If you have multiple peers using the same subnet, you can’t include them all in `AllowedIPs` — because their addresses will conflict. In such cases, NAT becomes useful. You can masquerade each peer’s LAN behind a unique WireGuard tunnel IP (like `10.x.x.x`), allowing communication without exposing overlapping LANs.

Another practical case is when you want to send the LAN’s internet traffic through the WireGuard tunnel to the server — for example, to route internet access through a central location. In this setup, masquerading ensures that outbound packets are properly routed through the tunnel and back.

Also, just like traditional NAT, this breaks inbound connections. But in some cases, that’s a feature — a form of implicit security. If you don’t want to expose your LAN to the server side, masquerading is a valid solution.

## NAT Implementation Example

Let’s reuse the topology from Part 1 or Part 2, where the server-side LAN is `172.19.2.0/24`.

Now, say the client has a LAN subnet `192.168.8.0/24`, but you don’t want to expose that network to the server — or you want to route LAN traffic to the internet via the server, or another peer already has the same subnet. Here’s what to do:

- **Do not** include `192.168.8.0/24` in the server’s `AllowedIPs`.
- On the **client side**, masquerade LAN traffic behind the WireGuard tunnel IP (e.g., `10.52.53.7`) with this command:

```bash
iptables -t nat -A POSTROUTING -s 192.168.8.0/24 -o wg0 -j MASQUERADE
```

This ensures the server sees all packets as coming from `10.52.53.7`, not from the internal LAN (`192.168.8.x`).

## Verifying on the Client

To confirm the NAT rule is active, inspect the `POSTROUTING` chain:

```bash
root@vpn-client:~# iptables -t nat -nvL POSTROUTING
Chain POSTROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
   20  1688 MASQUERADE  all  --  *      wg0     192.168.8.0/24       0.0.0.0/0
```

## Final Thoughts

WireGuard is clean and predictable — but only if you understand `AllowedIPs` and don’t blindly apply NAT.

