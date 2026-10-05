# Example: Public IPv4 Passthrough to Your Own Router on an SBB CGA2121

This example documents a working setup where an **SBB Technicolor
CGA2121** remained in router/gateway mode, while a downstream
**Grandstream GWN7003** received its **own public IPv4 address directly
by DHCP**.

This was useful when full bridge mode was not available or was not
required.

> This is not the same as DMZ and it is not ordinary double NAT. The
> CGA2121 feature used here is explicitly named **IP Address
> Pass-through by MAC Address**.

## Goal

The topology was:

``` text
SBB / DOCSIS
     |
     v
Technicolor CGA2121
(router mode)
     |
     |  IP Address Pass-through
     |  matched by WAN MAC
     v
Grandstream GWN7003
WAN: DHCP
     |
     v
Own LAN / VLANs
```

The desired result was for the GWN7003 WAN interface to receive a public
address instead of an RFC1918 address such as `192.168.x.x`.

## 1. Obtain administrative access

On the tested SBB CGA2121, the additional administrative
interface/settings were accessible after recovering the provisioned
administrative account from the router configuration backup.

That procedure is documented separately in the main CGA2121 backup
README.

Once logged in with the appropriate administrative account, the CGA2121
exposed the setting:

``` text
IP Address Pass-through by MAC Address
```

No configuration-backup modification was required for the passthrough
itself.

## 2. Identify the WAN MAC of your own router

The passthrough rule must target the MAC address of the downstream
router interface physically connected to the CGA2121.

In the tested setup this was the **WAN SFP/Port 11 interface of the
Grandstream GWN7003**.

The actual MAC is intentionally not published here.

Use the MAC shown by your own router for the relevant WAN interface.

Conceptually:

``` text
Own router WAN MAC:
XX:XX:XX:XX:XX:XX
```

Be careful not to use the MAC of a LAN interface or another physical
port.

## 3. Configure the downstream router WAN

The GWN7003 WAN connection remained simple:

``` text
Addressing: DHCP / Dynamic IP
PPPoE:      Disabled
Static IP:  Not configured
```

The downstream router does not need to know the public address in
advance. It requests an address normally via DHCP.

## 4. Enable IP Address Pass-through on the CGA2121

In the CGA2121 administrative interface, locate:

``` text
IP Address Pass-through by MAC Address
```

Add the WAN MAC address of your own router:

``` text
XX:XX:XX:XX:XX:XX
```

Save/apply the configuration.

The important relationship is:

``` text
CGA2121 passthrough MAC
          =
GWN7003 physical WAN-interface MAC
```

## 5. Renew the downstream WAN lease

After enabling passthrough, force the downstream router to obtain a new
DHCP lease.

Depending on the router, this can be done by:

-   disconnecting/reconnecting its WAN interface;
-   using DHCP Release/Renew;
-   or rebooting the downstream router.

The GWN7003 then requested DHCP again through the CGA2121.

## 6. Verify the result

Before passthrough, a router-behind-router setup would normally result
in a private WAN address and double NAT.

After enabling the CGA2121 MAC passthrough, the tested setup showed
**two different public IPv4 addresses**:

``` text
CGA2121 WAN/RG IPv4:   24.135.90.xxx
GWN7003 WAN IPv4:      89.216.206.xxx
```

The exact host portions above are deliberately redacted.

A browser behind the GWN7003 also reported the same public address
assigned to the GWN7003 WAN:

``` text
89.216.206.xxx
```

This confirmed that traffic from the downstream network was leaving
through the GWN7003's public IPv4 rather than being NATed behind the
CGA2121's public IPv4.


## 7. Two public IPv4 addresses: management side vs. passthrough side

A useful way to understand the tested setup is that the CGA2121 effectively participates with **two different public IPv4 addresses serving different purposes**.

The first public IPv4 belongs to the CGA2121's normal residential-gateway/service side. The CGA2121 remains an SBB-managed device: SBB can continue to manage the gateway, and subscriber-facing SBB functionality such as Wi-Fi configuration through the provider application can continue to operate.

Traffic using this normal gateway path still goes through the CGA2121's own router/NAT function:

```text
SBB / Internet
      |
      |  Public IPv4 #1
      v
   CGA2121
   SBB-managed
   router + NAT
      |
      v
private/internal network
```

The second public IPv4 is obtained by the downstream router through **IP Address Pass-through by MAC Address**. It is assigned by DHCP to the physical port on our router that we configured as WAN.

From the downstream router's point of view, this address is directly on its WAN interface:

```text
SBB / Internet
      |
      |  Public IPv4 #2
      v
IP Address Pass-through
      |
      v
GWN7003 port configured as WAN
Public IPv4 directly on WAN
      |
      v
GWN7003 firewall / NAT / VLANs
      |
      v
our internal networks
```

The important practical distinction is:

```text
Public IPv4 #1
  -> CGA2121's normal managed gateway side
  -> CGA2121 performs NAT for clients using that path
  -> SBB management / provider Wi-Fi-management functionality remains available

Public IPv4 #2
  -> passed to our own router by MAC
  -> appears directly on our router's WAN interface
  -> no CGA2121 NAT in front of that WAN address
  -> our own router becomes the NAT/firewall boundary for our LANs
```

This is why the setup is more useful than simply placing a privately addressed downstream router in DMZ. The CGA2121 can retain its provider-managed gateway functionality while our own router independently receives a public IPv4 without being behind the CGA2121's NAT.

In the tested setup, the two addresses were visibly different:

```text
CGA2121 public/RG IPv4:  24.135.90.xxx
GWN7003 WAN IPv4:        89.216.206.xxx
```

An external IP check from a client behind the GWN7003 matched the **second address**, confirming that the client's Internet path used the GWN7003 public WAN rather than the CGA2121 NAT path.

## 8. Why this is different from DMZ

A DMZ-host configuration typically looks like:

``` text
Internet
   |
public IP
   |
CGA2121 NAT
   |
private IP
   |
own router
```

The downstream router still has a private WAN address and the connection
still involves the upstream NAT layer.

With the tested SBB **IP Address Pass-through by MAC Address** setup:

``` text
Internet
   |
CGA2121
   |
   +---- CGA2121 keeps its own public/RG address
   |
   `---- selected downstream MAC receives another public IPv4
                              |
                              v
                           GWN7003
```

The GWN7003 therefore sees a public address directly on its WAN
interface.

## 9. Why this is different from full bridge mode

In full bridge mode, the ISP gateway generally stops acting as the
normal residential gateway for the bridged connection and behaves
primarily as a modem/bridge.

In this tested configuration the CGA2121 remained operational as a
router while the selected downstream device received a public IPv4
through MAC-based passthrough.

That allowed:

``` text
CGA2121 router functionality
            +
GWN7003 public WAN IPv4
```

without converting the CGA2121 to full bridge mode.

## 10. Working configuration summary

``` text
SBB gateway:
  Device: Technicolor CGA2121
  Mode: Router/gateway
  Feature: IP Address Pass-through by MAC Address
  Passthrough target: MAC of the GWN7003 port configured as WAN

Downstream router:
  Device: Grandstream GWN7003
  WAN mode: DHCP
  PPPoE: No
  Result: Public IPv4 assigned by DHCP
```

The real-world verification was:

``` text
CGA2121 public address != GWN7003 public address
```

and an external IP check from a client behind the GWN7003 matched the
**GWN7003 WAN public IP**.

## 11. Troubleshooting

### GWN7003 still receives a private IP

Check that the MAC entered in the CGA2121 is the MAC of the exact
physical WAN interface connected to it.

Then renew the WAN DHCP lease.

### No address after enabling passthrough

Remove/re-add the rule, verify the WAN MAC, and reconnect the downstream
WAN interface. ISP DHCP behavior and lease state can affect how quickly
a new address is issued.

### External IP still matches the CGA2121

Verify the downstream router's WAN status page. A successful setup
should show a public IPv4 on the downstream WAN itself.

### Do I need DMZ as well?

Not for the tested setup. MAC-based IP passthrough provided the desired
public WAN address directly to the GWN7003.

## 12. Important observation

The key discovery from this setup was:

> **Lack of bridge mode did not prevent using our own router with a
> public IPv4.**

The CGA2121's **IP Address Pass-through by MAC Address** feature allowed
the selected GWN7003 WAN interface to obtain a separate public IPv4 via
DHCP while the SBB gateway itself remained in router mode.

This is particularly useful when you want your own router to handle
firewalling, NAT, VPNs, VLANs, routing, and other network services
without relying on a conventional double-NAT/DMZ arrangement.

## Privacy

When publishing screenshots or diagnostics, redact:

-   full public IPv4 addresses where appropriate;
-   WAN MAC addresses;
-   SBB administrative credentials;
-   modem serial numbers;
-   subscriber identifiers;
-   configuration backups.

## Disclaimer

This example documents a configuration performed on equipment and
service under the administrator's control. ISP provisioning and
available CGA2121 features can differ between subscribers and firmware
versions.
