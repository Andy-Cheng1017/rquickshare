# Known issues (BLE / bandwidth-upgrade path)

Two rare, phone-side, retry-recoverable failures observed during PC↔Pixel
testing. Neither is reproducible on demand, and neither appeared in the last
retained session log (07-24, 18:32–19:32Z: 7 PC→phone sends, 6 successes, the
one non-success being the WIFI_LAN upgrade stall since fixed). Both are deferred
until they resurface *with logcat already capturing* — they can't be root-caused
from our side's log alone.

## A. Outbound first-connection handshake flake

The first PC→phone BLE send after launching the app occasionally fails during
UKEY2, then succeeds on retry.

- Our side: `Handling State::SentUkeyClientInit frame` followed by
  `peer closed the Weave connection` / an EOF reading the UKEY2 server init.
- Phone side (when captured): the Nearby Weave socket errors during UKEY2,
  BLE GATT `status 133`, `onDisconnected`.
- Recovery: the next attempt connects and transfers normally.

To root-cause, capture phone logcat *before* launching and send PC→phone
immediately, then line up the `NearbyConnections` UKEY2 lines
(`startClient`, Weave socket error, `status 133`, `EOFException`,
`onDisconnected`) against our `SentUkeyClientInit` timestamp.

## B. Role confusion after a transfer

The phone occasionally holds a stale *receiver* role, so a subsequent
connection comes in as the wrong role.

- Our side: a Response frame arrives where an introduction is expected; we
  detect it (`peer connected as a receiver …`), disconnect, and fail cleanly so
  the transfer can be retried.
- Suspected trigger: the Quick Share extension making the phone drop and
  reconnect WiFi mid-session, leaving its medium-selection / role state stale.

To root-cause, capture phone logcat spanning the end of one transfer and the
start of the next, and check the `NearbySharing` role / advertising state
between them.

## Capturing phone logcat

```powershell
$adb = "C:\Program Files (x86)\Android\android-sdk\platform-tools\adb.exe"
& $adb logcat -c
& $adb logcat -v time > "$env:LOCALAPPDATA\dev.mandre.rquickshare\logs\logcat.txt"
```

This drops `logcat.txt` next to `rquickshare.log`, so both sides of a failure
line up by timestamp.
