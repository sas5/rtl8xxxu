---
name: project-rtl8xxxu-kernel618-patch
description: rtl8xxxu Wi-Fi driver at ~/src/rtl8xxxu was patched for kernel 6.18 mac80211 API changes and to enable AP mode on RTL8188EU (TP-Link TL-WN722N v2/v3)
metadata: 
  node_type: memory
  type: project
  originSessionId: 3017029e-4032-4e26-9f68-103d1b73c47d
---

The out-of-tree `rtl8xxxu` driver (github.com/lwfinger/rtl8xxxu, cloned at `~/src/rtl8xxxu`) needed source patches to build and probe successfully on Raspberry Pi kernel 6.18.33+rpt-rpi-v8.

Two independent fixes were applied directly to the driver source (not upstreamed):

1. **Build fix** (`rtl8xxxu_core.c`): mac80211 in 6.18 changed several `ieee80211_ops` callback signatures (added `radio_idx`/`link_id`/`suspend` params for MLO and multi-radio support): `ieee80211_beacon_cntdwn_is_complete`, `ieee80211_csa_finish`, `.config`, `.set_rts_threshold`, `.stop`, `.get_antenna`. Fixed by adding the new parameters (passing `0` for link_id/radio_idx since this driver has no MLO/multi-radio support).

2. **Probe fix (`-ENOMEM` / `WARN_ON` in `ieee80211_alloc_hw_nm`)**: current mac80211 unconditionally requires every driver to either implement native channel-context ops or explicitly opt into `ieee80211_emulate_add_chanctx`/`remove_chanctx`/`change_chanctx`. `rtl8xxxu_ops` set neither, so `ieee80211_alloc_hw_nm()` returned NULL and probe failed with -12. Fixed by adding the three emulate hooks to `rtl8xxxu_ops` in `rtl8xxxu_core.c` (guarded by `LINUX_VERSION_CODE >= KERNEL_VERSION(6,15,0)`).

3. **AP mode enabled for RTL8188EU**: upstream `rtl8xxxu` has never set `.supports_ap = 1` for RTL8188EU's fops (`rtl8xxxu_8188e.c`) even though sibling gen2 chips (8192EU, 8723BU, 8710BU, 8188FU) have had it enabled one at a time by the maintainer after individual testing. This is a deliberate upstream gap, not a kernel regression. User's chip is exactly RTL8188EU (TL-WN722N v2/v3), and they needed AP mode for hostapd. Enabled `.supports_ap = 1` and `.max_macid_num = 16` (matching sibling RTL8188FU's values) — **not verified by upstream for this specific chip**, but tested live on this system: `hostapd@wlan1.service` (pre-existing systemd unit, `ssid=3D_PRINTER_2G`) went from crash-looping every ~3s with `nl80211: Could not configure driver mode` to `AP-ENABLED` with 0 restarts immediately after loading the patched module.

**Why:** kernel 6.18 broke compilation and probing entirely; user additionally wanted AP mode which upstream never shipped for this exact chip.

**How to apply:** If rtl8xxxu is rebuilt/re-cloned fresh from upstream, these three patches need to be reapplied — they are local-only changes, not present in the lwfinger/rtl8xxxu repo. Module is installed via plain `make install` (copies to `/lib/modules/<kver>/extra/`, blacklists in-tree `rtl8xxxu` via `/etc/modprobe.d/blacklist-rtl8xxxu.conf`, runs `depmod`) — **dkms is not installed on this system**, so the module will NOT automatically rebuild against a new kernel package after an apt upgrade; it only works for the exact kernel it was built against (6.18.33+rpt-rpi-v8). If the user upgrades the kernel later, these patches will need to be reapplied and rebuilt (or install dkms and use the repo's existing `dkms.conf`).
