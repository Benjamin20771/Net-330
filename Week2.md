# NET-330 — Week 02

## Simple Network — Hardware Lab

This week we rebuilt the same Laptop–Switch–Router–Switch–Server topology from Week 01's Packet Tracer lab, but on physical hardware

**Equipment used:**
- **1841 Router** 
- **Switches** — small Netgear switches 
- **Workstations** — classroom computers, booted off USB thumb drives into a live Kali Linux environment

---

### Step 1: Cabling
- Standard Ethernet cables connecting workstations to switches to the router.

### Step 2: Console Into the Router
1. Plugged the light-blue USB console cable into the workstation's USB port.
2. Checked Device Manager -> Ports (COM & LPT) to find the assigned COM number.
3. Connected the other end (also light blue) to the router's console port.
4. Powered everything on.
5. Opened PuTTY -> **Connection -> Serial**:

| Setting | Value |
|---|---|
| Serial line | COM port from Device Manager |
| Speed (baud) | 9600 |
| Data bits | 8 |
| Parity | None |
| Stop bits | 1 |
| Flow control | None |

6. Confirmed the **Session** tab connection type was set to Serial with the matching COM port.
7. Clicked **Open**, hit Enter -> dropped into the router CLI.

### Step 3: Boot Workstations to Kali
1. Plugged the USB thumb drive into the powered-off computer.
2. Pressed **F10** at the boot splash screen.
3. Boot Menu -> selected **UEFI Vendor Product**.
4. Selected **Live System (amd64)** -> first option.
5. Logged into the live Kali desktop (`kali` / `kali` if prompted).

Repeated for both workstations.

### Step 4: Router Configuration

```
enable
configure terminal

interface FastEthernet 0/0
 ip address 192.168.1.1 255.255.255.0
 no shutdown
 exit

interface FastEthernet 0/1
 ip address 192.168.2.1 255.255.255.0
 no shutdown
 exit

end
```

### Step 5: IP Configuration on Kali Workstations

Kali's live-boot network manager can be inconsistent, so IPs were set directly via the terminal. First, checked the actual interface name with `ip a`.

**Workstation on the one side:**
```
sudo ip addr add 192.168.1.2/24 dev <interface>
sudo ip route add default via 192.168.1.1
```

**Workstation on the other side:**
```
sudo ip addr add 192.168.2.2/24 dev <interface>
sudo ip route add default via 192.168.2.1
```

### Step 6: Verify Connectivity

From the router:
```
show interfaces
```
Confirmed both FastEthernet interfaces showed as **up/up**.

From the workstations:
```
ping 192.168.1.2
ping 192.168.2.2
```
Both came back with successful replies.

### Step 7: Save Config & Power Cycle Test

From **privileged EXEC mode** (`Router#`, *not* `Router(config)#`):
```
copy running-config startup-config
```
Then physically powered the router off, then back on, and re-checked the interface IPs to confirm they survived the reboot.

### Step 8: Reset Router to Blank State

Once testing was confirmed, the config was rolled back so the next group could start clean:

```
enable
configure terminal

interface FastEthernet 0/0
 no ip address
 shutdown
 exit

interface FastEthernet 0/1
 no ip address
 shutdown
 exit

end
```

---

## Reference Charts

Quick-lookup material for subnetting and address classes — useful any week this comes back up.

<img width="415" height="739" alt="image" src="https://github.com/user-attachments/assets/d06627b3-3fe8-4b11-ab57-0aa0cbe6eead" />

<img width="387" height="516" alt="image" src="https://github.com/user-attachments/assets/6f0cee73-65d4-4880-89c4-364b763e56f5" />

---

## Assignments
