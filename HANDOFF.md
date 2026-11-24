# Handoff: WhateverXe on ThinkPad X1 Yoga Gen 7

Read this before you do anything else. It covers the work from a previous session that ran on a different Mac (not the laptop).

**Delete this file before any upstream pull request.** The repo's AGENTS.md says investigation notes don't belong in the base.

## Machine
- **Laptop:** Lenovo ThinkPad X1 Yoga Gen 7, i7-1265U.
- **GPU:** Alder Lake-P, device ID `0x46A8`, Iris Xe 96 EU.
- **macOS:** Tahoe 26.7.3 (25G342), SMBIOS MacBookPro16,2.
- **Boot EFI:** lives on the USB drive (`/Volumes/USB/EFI/OC/`). The `EFI/` folder in the repo clone is an older copy and is not committed.
- **Main kexts:**
  - Lilu 1.7.2 DEBUG
  - WhateverXe (this repo)
  - AMFIPass
  - BlueToolFixup
  - IntelBluetoothFirmware 2.4.0
  - IntelBTPatcher 2.4.0
  - BTLEAdvFixup 0.9.4 (ForbiddenItlwm)
  - ForbiddenItlwm
  - USBToolBox + UTBDefault (no port map)
  - VoodooI2C/PS2
  - YogaSMC

## Code state
- **Fork:** `https://github.com/micr0softstore/WhateverXe-Fork` (this is `origin`; the upstream is `b00t0x/WhateverXe`).
- **Branch:** `fix/adlp-edp-boot-handoff`, based on upstream 0.8.4 (`4939720`). The fix is in commit `88190d9`.
- **Setup:**
  - `git clone -b fix/adlp-edp-boot-handoff https://github.com/micr0softstore/WhateverXe-Fork`
  - Clone MacKernelSDK into the repo root (it is gitignored).
  - You need a DEBUG `Lilu.kext` locally.
- **Build:** `python3 tools/build.py --lilu <path>/Lilu.kext`. Add `--release --output build/release` for a release build.
- **Tests:** `python3 tools/test.py` should report 221 passed and 0 failed. Native replays are skipped because the Apple drivers are not present.

### What the fix does
The internal panel went black and acceleration was dead. Now it is solved and confirmed on the laptop.
1. **`kern_vbt.hpp` `panelFacts`:** when the VBT panel type is 0xFF and the EDID is valid but missing from the table, fall back to panel type 0. This is what Linux `get_panel_type()` does.
2. **`kern_alderlake_n.hpp`:**
   - Bit 3 of TRANS_DDI_FUNC_CTL2 (0x60404) is `CMTG_SECONDARY_MODE`. It is now accepted on the boot eDP.
   - New helpers: `sourceCTL2`, `stoppedCTL2`, `releaseBootCMTG`, `releasedCTL2Write`.
3. **`kern_xe.cpp` `display13BootPanelDSCOff`:** once transcoder A is otherwise stopped, release the firmware's CMTG (0x6FA88). Clear the CTL2 bit first, then CMTG_ENABLE, the same order as Linux `intel_cmtg_disable_all`.
4. **`kern_xe.cpp` `wrapDisplay13GMBusWrite`:** after that release, Apple's driver wrote CTL2=8 again, and pipe A froze at scanline 1198, so the screen stayed black. Native writes to 0x60404 now lose bit 3 while CMTG A is off.

### Removed temporary code
- All `WEX-TEMP-OBS` logging is gone.
- The `-wex-edp-full-modeset` boot-arg experiment is gone. It forced a full modeset at boot instead of keeping the firmware link.

## Pending tasks, in priority order
1. **Verify a boot without the old flag.**
   - Every successful boot so far used `-wex-edp-full-modeset`, which forced the full-modeset path.
   - The new kext always takes the normal keep/fastset path. That matches Linux, which keeps the CMTG configuration it inherits from the firmware when the display version is below 20; Alder Lake-P is display 13.
   - This path has **not been hardware-tested**. I removed the flag from the USB config's boot-args. The user must reset NVRAM once so the change takes effect.
   - The user must also copy the new kext into `/Volumes/USB/EFI/OC/Kexts/`. The builds were at `~/Desktop/WEX-test-build/` on the old Mac. Rebuild from the branch if they are not there.
   - If the panel is black: making the full modeset the default is the fallback. The old branch logic was `if(forceFull || !owned())` in `XE::selectBootMode`.
2. **Bluetooth turns on, then off again** (Intel AX211, 8087:0033).
   - Config checked:
     - `ExecutablePath` was empty for IntelBluetoothFirmware and IntelBTPatcher; I fixed it. Backup: `config.plist.bak-bt` on the USB drive.
     - The NVRAM Delete list contains `bluetoothExternalDongleFailed` and `bluetoothInternalControllerInfo`.
     - Kext order is OK.
   - Main suspect: `BTLEAdvFixup.kext`. It patches `/usr/sbin/bluetoothd`, the same target BlueToolFixup patches. Next step: disable it, reset NVRAM, and test.
   - If Bluetooth still fails, collect logs on the laptop:
     ```
     { ioreg -p IOUSB -l -w0 | grep -E '"(USB Product Name|idVendor|idProduct)"'; \
       system_profiler SPBluetoothDataType; \
       log show --last boot --predicate 'process == "bluetoothd" OR eventMessage CONTAINS[c] "IntelBluetooth" OR eventMessage CONTAINS[c] "BTPatcher"' | tail -400; \
       grep -iE "btlfx|ibt|BTPatcher|BTLEAdv" /var/log/Lilu_*.txt; } > ~/Desktop/bt-debug.txt 2>&1
     ```
3. **HDMI shows no signal.** This needs new feature work.
   - In the kernel log, Apple's framebuffer handles ddi 1 / FB1 as `Connected alt mode` (USB-C DisplayPort), not native HDMI.
   - The TV's 4K EDID was read, but then: `fActiveNumberOfLanes = 0, linkSymbolClock = 0 ... PixelClock = 297000000`, so no signal goes out.
   - After replugging: `FB1: Resources allocation failed`.
   - At boot, WEX's readout says `hdmiB=0`; nothing was connected then.
   - Upstream README lists DP/USB-C on Alder Lake-H/P/U as further implementation required.
   - Next step: boot **with HDMI already connected** and collect the Lilu dump plus the kernel log, to see how the firmware/VBT wires that port.
4. **USB-C TV:** the GPU logged no display event; the TV only appeared as USB plus power delivery. Unplugging hard-froze the OS, and nothing was logged. Probably USB/TCSS hotplug, not WEX, but that is unconfirmed.
5. **Upstreaming (only if the user asks):**
   - Delete this file.
   - Base on upstream `codex/next-release` if it exists.
   - Update the README and CHANGELOG per AGENTS.md, keeping README.md and README_ja.md in sync.

## How to collect logs on the laptop
- Use boot-arg `liludump=90`. Do **not** prefix it with a dash; `-liludump=90` produced a stale dump. Don't use `-liludbgall`, because it overflows Lilu's ~264 KB buffer.
- WEX/Lilu lines are **only** in `/var/log/Lilu_*.txt`, not in the unified log.
- The Lilu dump covers early boot only; the buffer fills up. Hotplug events need the kernel log: `log show --last 2h --predicate 'sender == "AppleIntelTGLGraphicsFramebuffer"'`, or all kernel messages.
- The user's script is `wex-debug.sh`, which is on the USB drive. Results went into `./wex*` folders in the repo clone, which are not committed.

## Other config notes already given to the user
- AMFIPass 1.4.1 is in use, not AMFIAnchor.
- `csr-active-config` is `0xA03`; `0x803` is the recommended value.
- The installed WhateverXe Graphics pkg receipts show 0.4.5.
- VoodooI2CHID panics seen earlier are unrelated to graphics.
