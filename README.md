# OpenWrt Travel Router Setup Guide
## OpenWrt 25.12.2 · WireGuard · ProtonVPN · Travelmate

> These instructions were written and tested on a GL-iNet Beryl AX (MT3000) running OpenWrt 25.12.2. They should work on any OpenWrt router with two WiFi radios.

### What This Does

Turns any OpenWrt router into a travel router. It connects to hotel WiFi (or ethernet) as its upstream WAN connection, and broadcasts your own private WiFi for your devices. All traffic is routed through ProtonVPN via WireGuard. Your devices are firewalled and NAT'd behind the router.

- **radio0** (2.4 GHz) = wireless WAN — connects to hotel WiFi, managed by Travelmate
- **radio1** (5 GHz) = your private WiFi — your devices connect here
- **WAN port** = wired WAN — upstream internet connection (ethernet)
- **LAN port** = wired LAN — connect your laptop for setup

---

## Before You Start

You will need:
- An ethernet cable
- A computer with an ethernet port (or USB-ethernet adapter)
- A ProtonVPN WireGuard config file — see Step 13

---

## Part 1: Initial Connection

### Step 1 — Connect via ethernet

Connect your computer to the router's **LAN port** using an ethernet cable.

On your computer, set the ethernet interface to a **manual/static IP**:

| Setting | Value |
|---|---|
| IP Address | `192.168.1.2` |
| Subnet Mask | `255.255.255.0` |
| Router/Gateway | `192.168.1.1` |

> **Why manual?** The router's WiFi radios are disabled on a fresh OpenWrt install, so it cannot serve DHCP reliably over ethernet until configured. Manual IP guarantees you can reach it.

**On Mac:** System Settings → Network → your ethernet interface → Details → TCP/IP → Configure IPv4: Manually

### Step 2 — Set root password

Open a browser and go to `http://192.168.1.1`

You will be prompted to set a root password. Set a strong password and note it down.

### Step 3 — SSH in

Open a terminal and connect:

```bash
ssh root@192.168.1.1
```

Accept the host key fingerprint when prompted. Enter your root password.

---

## Part 2: Base Configuration (via SSH)

Run each of the following commands in order. All commands must be run as root via SSH.

### Step 4 — Change LAN IP

This avoids conflicts with hotel networks which commonly use 192.168.1.x.

```bash
uci set network.lan.ipaddr='10.20.30.1/24'
uci commit network
```

> **Important:** Include the `/24` suffix — without it the router loses its subnet mask and becomes unreachable after reboot.

### Step 5 — Create the wireless WAN network interface

```bash
uci set network.wwan='interface'
uci set network.wwan.proto='dhcp'
uci set network.wwan.peerdns='0'
uci set network.wwan.metric='100'
uci commit network
```

### Step 6 — Add wwan to the WAN firewall zone

```bash
uci add_list firewall.@zone[1].network='wwan'
uci commit firewall
```

### Step 7 — Enable WiFi radios and set country code

```bash
uci set wireless.radio0.disabled='0'
uci set wireless.radio1.disabled='0'
uci set wireless.radio0.country='GB'
uci set wireless.radio1.country='GB'
uci commit wireless
```

> Replace `GB` with your country code if needed (US, JP, DE, etc.)

### Step 8 — Set up your private WiFi on radio1

Replace `MyTravelWiFi` and `YourPassword` with your chosen SSID and password. Password must be at least 8 characters.

```bash
uci set wireless.default_radio1.ssid='MyTravelWiFi'
uci set wireless.default_radio1.encryption='psk2'
uci set wireless.default_radio1.key='YourPassword'
uci set wireless.default_radio1.disabled='0'
uci commit wireless
```

> **Important:** The `disabled='0'` line is required — the AP interface is disabled by default on a fresh install and must be explicitly enabled.

### Step 9 — Disable DNS rebind protection

This is required for captive portal (hotel login page) handling to work correctly.

```bash
uci set dhcp.@dnsmasq[0].rebind_protection='0'
uci commit dhcp
```

### Step 10 — Reboot

```bash
reboot
```

Wait approximately 2 minutes for the router to reboot.

---

## Part 3: Reconnect After Reboot

### Step 11 — Update your ethernet IP

The router's LAN IP is now `10.20.30.1`. Update your computer's ethernet settings:

| Setting | Value |
|---|---|
| IP Address | `10.20.30.2` |
| Subnet Mask | `255.255.255.0` |
| Router/Gateway | `10.20.30.1` |

### Step 12 — Verify SSH works

```bash
ssh root@10.20.30.1
```

If this connects successfully, the base setup is complete.

You should also be able to see your private WiFi SSID (e.g. `MyTravelWiFi`) broadcasting from the router at this point.

---

## Part 4: WireGuard VPN Setup (ProtonVPN)

### Step 13 — Get your WireGuard config file

1. Log in at `account.proton.me`
2. Go to **Downloads → WireGuard configuration**
3. Select a server and download the `.conf` file

The file will look like this:

```ini
[Interface]
PrivateKey = <your private key>
Address = 10.2.0.2/32, 2a07:b944::2:2/128
DNS = 10.2.0.1

[Peer]
PublicKey = <server public key>
AllowedIPs = 0.0.0.0/0, ::/0
Endpoint = 1.2.3.4:51820
PersistentKeepalive = 25
```

### Step 14 — Install WireGuard packages

The router needs internet access for this step. Connect your home broadband to the WAN port if you have not already.

Go to **System → Software** in LuCI (`https://10.20.30.1`):
1. Click **Update lists**
2. Install: `kmod-wireguard`
3. Install: `luci-proto-wireguard`
4. Reboot

### Step 15 — Create the WireGuard interface

Go to **Network → Interfaces → Add new interface**:

| Field | Value |
|---|---|
| Name | `wg0` |
| Protocol | `WireGuard VPN` |

Click **Create interface**.

On the **General Settings** tab, click **Import configuration** and load your `.conf` file. This fills in the Private Key and IP Addresses automatically.

### Step 16 — Add the VPN peer

Click the **Peers** tab. Your peer should have been imported. Click **Edit** on it and make sure **Route Allowed IPs** is checked. Click **Save**.

Click **Save & Apply**.

### Step 17 — Add wg0 to the WAN firewall zone

Go to **Network → Firewall → Zones** → click **Edit** on the `wan` zone → add `wg0` to **Covered networks** → **Save & Apply**.

### Step 18 — Set routing metrics so all traffic goes through VPN

Via SSH:

```bash
uci set network.wwan.metric='100'
uci set network.wg0.metric='10'
uci commit network
/etc/init.d/network restart
```

This makes wg0 (VPN) the preferred default route (lower metric = higher priority), while wwan remains as fallback if the VPN drops.

### Step 19 — Set DNS servers

Go to **Network → Interfaces → Edit wwan → Advanced Settings tab**:

- Uncheck **Use DNS servers advertised by peer**
- Add DNS servers:
  - `9.9.9.9` (Quad9)
  - `149.112.112.112` (Quad9 secondary)
  - `1.1.1.1` (Cloudflare backup)

Click **Save & Apply**.

### Step 20 — Verify VPN is working

Go to `https://whatismyip.com` from a device connected to your private WiFi. It should show your ProtonVPN server's IP address, not your real IP.

Go to `https://10.20.30.1` → **Status → WireGuard** — the peer should show a recent handshake timestamp.

---

## Part 5: Travelmate Setup

Travelmate manages upstream WiFi connections automatically. You add networks once and it connects to them automatically whenever they are in range.

> **Important:** Travelmate takes over radio0 as the upstream WiFi manager. Do not manually configure wireless client interfaces on radio0 — let Travelmate manage it.

> **Full documentation:** [Travelmate README on GitHub](https://github.com/openwrt/packages/blob/961b94904a08b424d7fdd2831c587b5b06b6b3ee/net/travelmate/files/README.md)

### Step 21 — Install Travelmate

The router needs internet access for this step — your home broadband should already be connected to the WAN port from Step 14.

Go to **System → Software** in LuCI:
1. Click **Update lists**
2. Search for and install: `travelmate`
3. Search for and install: `luci-app-travelmate`
4. Reboot the router when prompted

### Step 22 — Run the Interface Wizard

Go to **Services → Travelmate**

Click **Interface Wizard**. You will see pre-filled values:

| Field | Value |
|---|---|
| Interface name | `trm_wwan` |
| Firewall zone | `wan` |
| Metric | `100` |

These are all greyed out — do not change them. Click **Save**.

> **If you see "The interface already exists"**: This means a `wwan`-related interface already exists from a previous setup attempt. See the troubleshooting section.

### Step 23 — Add trm_wwan to the WAN firewall zone

The wizard may not add `trm_wwan` to the firewall automatically. Do this via SSH:

```bash
uci add_list firewall.@zone[1].network='trm_wwan'
uci commit firewall
reload_config
```

### Step 24 — Configure Travelmate

Go to **Services → Travelmate → General Settings**:

| Setting | Value |
|---|---|
| Enabled | ✔ checked |
| WWAN Interface | `trm_wwan` |
| Radio Selection | `radio0` |
| Captive Portal Detection | ✔ checked |
| VPN processing | ✘ unchecked (known bug — leave off) |
| ProActive Uplink Switch | ✔ checked (switches to higher priority network when available) |
| LAN Interface (Additional Settings tab) | `lan` |

Click **Save & Restart**.

### Step 25 — Enable Travelmate autostart

Via SSH:

```bash
/etc/init.d/travelmate enable
```

This ensures Travelmate starts automatically on every boot.

### Step 26 — Add your first upstream WiFi network

Go to **Services → Travelmate → Wireless Stations**:

1. Click **Scan on radio0**
2. Find your network in the list
3. Click **Add Uplink...**
4. Enter the WiFi password
5. Click **Save & Apply**

Wait 30–60 seconds. The status on the Overview tab should show `connected, net ok`.

> **Encryption tip:**
> - Network has a password → **WPA2-PSK**
> - No password, just a login page (captive portal) → **None**
> - WPA3 is supported but less common

### Step 27 — Verify everything works

Check that devices connected to your private WiFi have internet access.

Go to `https://whatismyip.com` — it should show your ProtonVPN server's IP address.

---

## Ongoing Use: Connecting to a New Hotel WiFi

Each time you arrive at a new hotel:

1. Power on the router — wait for it to fully boot (check your router's LED indicator)
2. Connect your devices to your private WiFi
3. Go to `https://10.20.30.1` → **Services → Travelmate → Wireless Stations**
4. Click **Scan on radio0**
5. Find the hotel WiFi → **Add Uplink...** → enter password → **Save & Apply**
6. Wait 30–60 seconds

Next time you visit the same hotel, Travelmate connects automatically with no input needed.

**If the hotel has ethernet instead of WiFi:** Plug the cable into the router's WAN port. No configuration needed.

**If there's a captive portal (hotel login page):** Open `http://neverssl.com` in a browser — you'll be redirected to the hotel login page.

---

## Switching to a Different ProtonVPN Server

1. Download a new `.conf` file from `account.proton.me` → Downloads → WireGuard configuration
2. Go to **Network → Interfaces → Edit wg0**
3. **Peers tab** → click **Delete** on the existing peer
4. Click **Import configuration as peer...** and load the new `.conf` file
5. Make sure **Route Allowed IPs** is checked on the new peer
6. **General Settings tab** → update **Private Key** and **IP Addresses** from the new `.conf` file
7. Click **Save & Apply**
8. Go to **Status → WireGuard** — confirm a handshake appears within 30 seconds

> Each ProtonVPN WireGuard config file contains a unique private key and IP address — you must update both the interface settings AND the peer when switching configs.

---

## Reference

| Thing | Value |
|---|---|
| Router admin (LuCI) | `https://10.20.30.1` |
| SSH | `ssh root@10.20.30.1` |
| LAN port | ethernet (your devices / setup) |
| WAN port | ethernet (upstream internet connection) |
| radio0 | 2.4 GHz — upstream client (managed by Travelmate) |
| radio1 | 5 GHz — your private AP |

---

## Troubleshooting

**Can't reach router after reboot**
- Make sure your ethernet is set to manual IP `10.20.30.2` / `255.255.255.0` / `10.20.30.1`
- Try unplugging and replugging the ethernet cable

**Travelmate wizard says "The interface already exists"**
- This happens if a `wwan` interface already exists from a previous manual setup
- Create `trm_wwan` manually via SSH:
```bash
uci set network.trm_wwan=interface
uci set network.trm_wwan.proto='dhcp'
uci set network.trm_wwan.peerdns='0'
uci set network.trm_wwan.metric='100'
uci del network.wwan2 2>/dev/null
uci commit network
uci add_list firewall.@zone[1].network='trm_wwan'
uci commit firewall
reload_config
```

**Travelmate says "travelmate is disabled" in logs**
- Make sure Enabled is checked in General Settings
- Make sure WWAN Interface is set to `trm_wwan`
- Run `/etc/init.d/travelmate enable && /etc/init.d/travelmate restart` via SSH

**Travelmate connects but no internet**
- Check `https://10.20.30.1` → Network → Interfaces — does `trm_wwan` show an IP address?
- Check Services → Travelmate → Log View for errors

**WiFi client on radio0 shows "Wireless is not associated"**
- Check for a stale bridge setting: `uci show network.wwan` — if you see `type='bridge'`, remove it:
```bash
uci del network.wwan.type
uci commit network
wifi down && wifi up
```

**Need to factory reset**
- Hold the reset button for 10 seconds while powered on
- Router resets to `192.168.1.1` with no password
- Redo this entire guide from Part 1

---

## Appendix: Using a Different VPN Provider (e.g. Mullvad)

The WireGuard setup steps in this guide work with any WireGuard-based VPN provider. The only differences are where you download the config file and what DNS server to use.

### Getting a Mullvad config file

1. Log in at `mullvad.net/en/account`
2. Go to **WireGuard configuration**
3. Select a server, generate a config, and download the `.conf` file

The file looks similar to ProtonVPN but uses Mullvad's DNS:

```ini
[Interface]
PrivateKey = <your private key>
Address = 10.64.x.x/32
DNS = 10.64.0.1

[Peer]
PublicKey = <server public key>
AllowedIPs = 0.0.0.0/0, ::/0
Endpoint = 1.2.3.4:51820
```

### What to do differently

Follow Steps 13–20 as written, with these differences:

- **Step 13:** Download your config from Mullvad instead of ProtonVPN
- **Step 19 (DNS):** Mullvad's DNS is `10.64.0.1`. You can use this instead of Quad9/Cloudflare, or keep Quad9/Cloudflare if you prefer not to use the provider's DNS

Everything else — interface creation, firewall zone, metrics, Travelmate — is identical.
