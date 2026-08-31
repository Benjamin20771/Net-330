# NET-330 — Week 01

## Password Recovery Labs: Catalyst 3750 Switch & Cisco 1841 Router

This week's hands-on assignment covered password recovery on two Cisco devices: a **Catalyst 3750 switch** and a **Cisco 1841 router**. Both procedures require physical console access and interrupting the normal boot process.

---

## Info found online
- Google was primarily used
- Cisco Community

## Prerequisites

- Console cable connected from your PC to the device's console port
- [PuTTY](https://www.putty.org/) (or another terminal emulator)
- Device's COM port number (Windows)

### Finding Your COM Port
1. Press `Win + X` → select **Device Manager**
2. Expand **Ports (COM & LPT)**
3. Note the COM number next to your USB-to-serial adapter (e.g. `COM3`, `COM4`)

### PuTTY Serial Settings
| Setting | Value |
|---|---|
| Connection type | Serial |
| Serial line | Your COM port (e.g. `COM3`) |
| Speed (baud) | `9600` |

---

## 1. Catalyst 3750 Switch — Password Recovery

> The 3750 has **no power switch or shutdown command** the power is controlled entirely by plugging/unplugging the power cord.

### Step 1: Enter Recovery Mode
1. Unplug the power cord from the back of the switch.
2. Press and hold the physical **Mode** button on the front-left panel.
3. While still holding Mode, plug the power cord back in.
4. Keep holding Mode until the **SYST LED** flashes amber, then turns solid green, then release.
5. PuTTY should now show a `switch:` prompt.

### Step 2: Bypass the Startup Config
```
flash_init
dir flash:
rename flash:config.text flash:config.old
boot
```
- When prompted `Would you like to enter the initial configuration dialog? [yes/no]:` → type **no**

### Step 3: Restore Config & Set New Password
```
enable
rename flash:config.old flash:config.text
copy flash:config.text system:running-config
configure terminal
enable secret [your-new-password]
end
copy running-config startup-config
```

### Step 4: Test It
```
disable
enable
```
Enter the new password when prompted (input will not display on screen, this is normal).

**Result:** Password reset, original configuration preserved, changes saved to startup-config.

---

## 2. Cisco 1841 Router — Password Recovery

### Step 1: Interrupt the Boot Sequence
1. Move the console cable to the 1841's console port.
2. Power the router **off**, then back **on**.
3. Immediately press **Ctrl + Break** repeatedly within the first 60 seconds.
   - Laptop keyboards: try `Fn + Ctrl + Pause/Break` or `Ctrl + Shift + Esc`
4. You should land at a `rommon 1 >` prompt.

### Step 2: Change the Configuration Register
```
confreg 0x2142
reset
```
- This tells the router to ignore its saved startup config on the next boot.
- When prompted `Would you like to enter the initial configuration dialog? [yes/no]:` → type **no**
- Press **Enter** at `Press RETURN to get started.`

### Step 3: Restore Config, Set Password, Fix Register
```
enable
copy startup-config running-config
configure terminal
enable secret [your-new-password]
config-register 0x2102
end
copy running-config startup-config
```

> **`config-register 0x2102` is supa dupa important.** Skipping it means the router will keep ignoring its saved config on every future reboot.

#### We got: `%% Non-volatile configuration memory invalid or not present.`
This means there's no existing startup-config to load (fresh/erased device). Skip the `copy startup-config running-config` step and go straight to:
```
configure terminal
enable secret [your-new-password]
config-register 0x2102
end
copy running-config startup-config
```

### Step 4: Test It
1. Power-cycle the router (off → wait 5 sec → on).
2. Confirm it boots **straight to a normal prompt** (no initial config dialog).
3. At `Router>`, type:
```
enable
```
4. Enter your new password when prompted.

**Result:** If you land at `Router#`, the password was saved correctly, and the boot register is back to normal (`0x2102`).

---

## Quick Reference: Command Summary

| Device | Recovery Trigger | Key Commands |
|---|---|---|
| Catalyst 3750 | Hold **Mode** button while powering on | `flash_init`, `rename config.text/.old`, `boot` |
| Cisco 1841 | **Ctrl + Break** during boot | `confreg 0x2142`, `reset`, `config-register 0x2102` |

---

## Notes
- Passwords never echo to the terminal in PuTTY — this is expected Cisco behavior, not a bug.
- Neither device has a graceful shutdown command; power is cut directly via cord/switch.
- Always confirm the register is restored to `0x2102` on routers, or the fix won't persist across reboots.

---

## Assignments
### Assignment 1 — IP Addressing Review
 
Covers binary conversion, CIDR notation, and calculating Net ID / Host ID from a mask. The same core process runs through all of it: **decimal ↔ binary conversion using place values, then AND the IP with the mask to split it into network vs. host.**
 
#### Binary Place Values (per octet)
Each octet is 8 bits. Add up the "on" bits to get the decimal value:
 
| 128 | 64 | 32 | 16 | 8 | 4 | 2 | 1 |
|---|---|---|---|---|---|---|---|
| 1 | 0 | 0 | 1 | 1 | 0 | 0 | 1 |
 
`128 + 16 + 8 + 1 = 153` → octet `153` = `10011001`
 
#### CIDR Notation
The `/x` number = total count of "1" bits across all 4 octets of the mask.
 
| Mask | Binary (last non-zero octet) | 1s in that octet | /x |
|---|---|---|---|
| 255.255.248.0 | 11111000 | 5 | /21 |
| 255.255.255.224 | 11100000 | 3 | /27 |
| 255.255.240.0 | 11110000 | 4 | /20 |
 
Quick reference — common octet values and their bit count:
`128=/1  192=/2  224=/3  240=/4  248=/5  252=/6  254=/7  255=/8`
 
#### Net ID / Host ID
1. Convert both IP and mask to binary.
2. **AND** them together (bit matches 1 only if both are 1) → this gives the **Net ID**.
3. Whatever's left over (the host bits, wherever the mask has 0s) is the **Host ID**.
Example: `146` (host bit) with mask `192`:
```
10010010   (146)
11000000   (mask 192)
--------
10000000   (128 → Net ID octet)
```
Host ID octet = `146 - 128 = 18`
 
#### Finding Your Own IP Info (Mac/Linux: `ifconfig`, Windows: `ipconfig /all`)
- Look for the **active** interface (`status: active` on Mac, or the one with a real IP on Windows).
- **Default gateway** isn't shown in `ifconfig` — get it separately with `netstat -nr | grep default` (Mac/Linux) or it's listed directly in `ipconfig /all` (Windows).
- A `fe80::...` address is **link-local only** (not globally routable), if that's the *only* IPv6 address listed, your network isn't handing out real IPv6 addresses, which is a good real-world example of IPv6 adoption still being incomplete.
#### IPv6 Adoption Article
**"18 Years Later, IPv6 Reaches Majority"** April 21, 2026
https://pulse.internetsociety.org/en/blog/2026/04/18-years-later-ipv6-reaches-majority/
 
- Access to Google's services via native IPv6 passed 50% for the first time on March 28, 2026.
- First time IPv6 has been measured as the *majority* protocol for reaching Google.
- Internet Society's Pulse dashboard aggregates several independent measurement sources.
- IPv4 isn't going away, so the majority reflects measured traffic preference.
 
---
 
### Assignment 2 — Packet Tracer Review Lab
 
Basic topology: Laptop — Switch — Router — Switch — Server, two subnets joined at the router.
 
**Topology:** `Foster Laptop - Foster Switch - CNCS Router (Fa0/0 | Fa0/1) - Skiff Switch - Skiff Server`
 
#### IP Plan
 
| Device | Interface | IP Address | Mask | Gateway |
|---|---|---|---|---|
| CNCS Router | Fa0/0 | 192.168.3.1 | /24 | — |
| CNCS Router | Fa0/1 | 192.168.1.1 | /24 | — |
| Foster Laptop | NIC | 192.168.3.2 | /24 | 192.168.3.1 |
| Skiff Server | NIC | 192.168.1.2 | /24 | 192.168.1.1 |
 
**Key idea:** the router's interface IP on a segment is the default gateway for every device on that segment.
 
#### Cabling
- **Straight-Through** for everything here (laptop↔switch, switch↔router, switch↔server). Crossover is only needed for like-to-like connections (switch↔switch, PC↔PC), which didn't come up in this lab.
#### Router Config (CLI)
```
enable
configure terminal
interface fastEthernet0/0
 ip address 192.168.3.1 255.255.255.0
 no shutdown
exit
interface fastEthernet0/1
 ip address 192.168.1.1 255.255.255.0
 no shutdown
exit
end
copy running-config startup-config
```
Watch for `%LINK-5-CHANGED: ... changed state to up` — if it doesn't show up, the cable's probs in the wrong interface.
 
#### Ping Test
From laptop Command Prompt: `ping 192.168.1.2` → looking for `Reply from 192.168.1.2`.
 
#### Simulation Mode — Reading HTTP Traffic
1. Stopwatch icon (bottom right) - Simulation Mode.
2. Laptop - Web Browser - enter server's IP - Go.
3. Capture/Forward through the event list — you'll see it play out in order:
   **TCP handshake (SYN - SYN-ACK - ACK) - HTTP GET - HTTP response**
4. Click any HTTP (purple) row - PDU Details.
   - **Outbound** = packet leaving that device (e.g. laptop sending the GET)
   - **Inbound** = packet arriving at that device (e.g. server receiving the GET, or laptop receiving the response)
   - Rule of thumb: pick Outbound if the device is the source of that hop, Inbound if it's the destination.
#### NVRAM Save / Power Cycle Test
1. Back to Realtime mode (clock icon).
2. Router - Config tab (or CLI `copy running-config startup-config`) to save to NVRAM.
3. Router - Physical tab - power off - power back on.
4. Fast Forward Time to skip the boot wait.
5. `show running-config` (or `show ip interface brief`) → confirm Fa0/0 / Fa0/1 IPs survived the reboot.
If the IPs vanish after reboot, the save didn't take — redo the `copy running-config startup-config` and repeat the power cycle.
