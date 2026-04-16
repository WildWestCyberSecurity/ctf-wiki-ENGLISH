# WIFI

> `802.11` is the current standard for wireless LANs. Common authentication methods include:
>
> - No security enabled
> - `WEP`
> - `WPA/WPA2-PSK` (Pre-Shared Key)
> - `PA/WPA2 802.1X` (`radius` authentication)

## WPA-PSK

The general authentication process is shown in the figure below:

![wpa-psk](./figure/wpa-psk.png)

The four-way handshake process:

![eapol](./figure/eapol.png)

1. The four-way handshake is initiated by the authenticator (AP), which generates a random value (ANonce) and sends it to the supplicant.
2. The supplicant also generates its own random SNonce, then uses both Nonces along with the PMK to generate the PTK. The supplicant replies with message 2 to the authenticator, along with a MIC (Message Integrity Code) for PMK verification.
3. The authenticator first verifies the MIC and other information sent by the supplicant in message 2. After successful verification, it generates the GTK if needed, then sends message 3.
4. The supplicant receives message 3, verifies the MIC, installs the keys, and sends message 4 as a confirmation. The authenticator receives message 4, verifies the MIC, and installs the same keys.

## Example

> Shiyanba: `shipin.cap`

Based on the large number of `Deauth` attacks, it can be determined that this is a traffic capture from a `wifi` cracking attack.

The handshake packets were also successfully identified:

![shiyanba-wpa](./figure/shiyanba-wpa.png)

Next, we crack the password:

- `linux`: `aircrack` suite
- `windows`: `wifipr`, faster than `esaw`, `GTX850` can reach nearly `10w\s  :`)

After obtaining the password `88888888`, enter it in `wireshark` via `Edit -> Preferences -> Protocols -> IEEE802.11 -> Edit` in the format `key:SSID` to decrypt the `wifi` packets and view the plaintext traffic.

> KRACK related: https://www.krackattacks.com/

## References

- http://www.freebuf.com/articles/wireless/58342.html
- http://blog.csdn.net/keekjkj/article/details/46753883
