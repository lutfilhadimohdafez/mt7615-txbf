# mt7615-txbf

Explicit TX beamforming (VHT SU beamformer) for the **MediaTek MT7615** in the
open-source [mt76](https://github.com/openwrt/mt76) driver used by OpenWrt and
ImmortalWrt.

Stock mt76 has never enabled beamforming on MT7615, even though the hardware
and firmware support it. This patch turns it on.

> **Status: experimental.** It has been tested on one router with one client. It
> is off by default and only does something when you load the module with
> `txbf=1`.

## Results

Tested on a Xiaomi **Redmi AC2100** (MT7621 + MT7615N 4x4, 5 GHz) running
ImmortalWrt 25.12.2 (kernel 6.12.103, mt76 `39c960c3`, N9 firmware 20200814):

| | Stock mt76 | With this patch (`txbf=1`) |
|---|---|---|
| VHT caps advertised | no beamformer | `SU Beamformer`, `SOUNDING-DIMENSION-4` |
| TX frames sent beamformed (from hardware TX status) | 0 | **97%** (126,730 of 130,396) |
| TX retries during a speedtest | 9% | 4% |

The client was a phone that supports SU beamformee. Numbers are from single
speedtest runs. A controlled range test is still to do (see [Help wanted](#help-wanted)).

## Should work on

Any device with an **MT7615 / MT7615N / MT7615D** in mt76 (driver `mt7615e`),
such as the Xiaomi Mi/Redmi AC2100 and other MT7621 + MT7615 routers. Only the
Redmi AC2100 has been tested. Reports for other boards are welcome.

The patch does nothing on MT7622 or MT7663, and nothing on MT7603 (2.4 GHz,
which has no beamforming).

## Use it

### 1. Build

Put the patch into your OpenWrt or ImmortalWrt tree, or into a matching SDK:

```sh
cp patches/999-mt7615-experimental-txbf.patch <tree>/package/kernel/mt76/patches/
make package/mt76/{clean,compile}
```

The patch applies to mt76 `39c960c3` (2026-03-19). It touches only `mt7615/`,
so it should apply to nearby mt76 versions too.

Then install the rebuilt `kmod-mt76-core`, `kmod-mt76-connac`,
`kmod-mt7615-common` and `kmod-mt7615e` packages. Their kernel version must
match your router's.

### 2. Enable

```sh
echo "mt7615-common txbf=1" > /etc/modules.d/mt7615-common
reboot
```

| Module parameter | Default | Meaning |
|---|---|---|
| `txbf` | `0` | Enable explicit beamforming (MT7615 only) |
| `txbf_snd_ms` | `100` | Interval for sounding each station, in ms. `0` leaves sounding to the firmware, which in practice means no beamforming (see below). |

### 3. Verify

```sh
# should show "SU Beamformer"
iw phy | grep -i beamform
# should include [SU-BEAMFORMER][SOUNDING-DIMENSION-4]
grep vht_capab /var/run/hostapd-phy*.conf

# put load on a 5 GHz client (speedtest), then:
D=$(ls -d /sys/kernel/debug/ieee80211/phy*/mt76 | while read d; do [ -e $d/txbf_txs_ebf ] && echo $d; done)
cat $D/txbf_txs_all $D/txbf_txs_ebf
```

`txbf_txs_ebf` counts frames that the hardware reports it actually sent with
explicit beamforming. If it stays at 0, check that the client supports SU
beamformee:
`ubus call hostapd.<iface> get_clients | grep su_beamformee`.

### Roll back

Delete `/etc/modules.d/mt7615-common` and reboot. With `txbf=0` the driver
behaves like stock.

## How it works

The firmware interface comes from MediaTek's vendor `mt7615d` driver (`mt_wifi`
`txbf/`, `mcu/`). Its source is available in
[coolsnowwolf/lede](https://github.com/coolsnowwolf/lede/tree/master/package/lean/mt/drivers/mt7615d).
Only the command IDs and struct layouts were taken from it, no code.

1. **Capability:** advertise the VHT SU beamformer and sounding dimensions as
   antennas − 1. This is applied after the EEPROM chainmask has been read.
2. **Init:** `TXBF_ACTION` turns sounding on in `BF_PROCESSING` mode, then
   `BF_HW_ENABLE_STATUS_UPDATE` enables eBF.
3. **Per station** (when the client supports VHT SU beamformee):
   - a `STA_REC_BF` record in the 36-byte MT7615 layout, which is shorter than
     the connac `sta_rec_bf`. PFMU is set to `0xffff` so the firmware allocates
     the profile.
   - the `WTBL_BF` `ebf_vht` bit
   - `BF_DATA_PACKET_APPLY`
   - **SU periodic sounding**

The last step is the key one. The vendor driver lets **firmware rate control**
trigger sounding. mt76 does rate control on the **host**, so the firmware never
sounds. With no sounding there is no profile, and no frame gets beamformed.
Steps 1 to 3 without periodic sounding give 0% beamformed frames, which is
what we measured.

## Not done yet

- MU-MIMO beamformer
- Beamformee (would also need `MT_WF_RFCR1_DROP_BF_POLL` cleared)
- Implicit BF (needs per-board phase calibration)
- 2.4 GHz / HT beamforming
- DBDC mode (vendor `BF_MOD_EN_CTRL`)
- Beamforming state after a firmware reset: it is not re-sent until the next
  reload.

## Help wanted

- Tests on other MT7615 boards, especially DBDC ones
- Range tests: `txbf=0` versus `txbf=1` from the same far spot, using iperf3
  downlink with `-R`
- Long-running stability reports
- Upstreaming: this commit passes `checkpatch --strict` and is meant to go to
  linux-wireless and mt76 eventually.

## License

The patch modifies mt76 and is under the same license as mt76
(BSD-3-Clause-Clear). Use at your own risk.
