# NET-330 — Week 01

## Password Recovery Labs: Catalyst 3750 Switch & Cisco 1841 Router

This week's hands-on assignment covered password recovery on two Cisco devices: a **Catalyst 3750 switch** and a **Cisco 1841 router**. Both procedures require physical console access and interrupting the normal boot process.

---

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

> The 3750 has **no power switch or shutdown command** — power is controlled entirely by plugging/unplugging the power cord.

### Step 1: Enter Recovery Mode
1. Unplug the power cord from the back of the switch.
2. Press and hold the physical **Mode** button on the front-left panel.
3. While still holding Mode, plug the power cord back in.
4. Keep holding Mode until the **SYST LED** flashes amber, then turns solid green — then release.
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
Enter the new password when prompted (input will not display on screen — this is normal).

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

> **`config-register 0x2102` is critical.** Skipping it means the router will keep ignoring its saved config on every future reboot.

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

## Notes / Gotchas
- Passwords never echo to the terminal in PuTTY — this is expected Cisco behavior, not a bug.
- Neither device has a graceful shutdown command; power is cut directly via cord/switch.
- Always confirm the register is restored to `0x2102` on routers, or the fix won't persist across reboots.

---

## Assignments
