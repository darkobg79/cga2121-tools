# Extracting the Administrative Account from an SBB CGA2121 Configuration Backup

This document records the configuration-backup workflow used on an
**SBB-provisioned CGA2121** gateway to decrypt `GatewaySettings.bin` and
read the administrative credentials stored in the Broadcom
configuration.

> **Use this only on equipment and configuration backups you own or are
> explicitly authorized to administer.**
>
> The decrypted configuration contains credentials and other sensitive
> provisioning data. Never commit a real backup, decrypted
> configuration, MAC address, passwords, or extracted secrets to a
> public repository.

## Overview

The tested `GatewaySettings.bin` format used:

``` text
96-byte header
        |
        v
AES-256-ECB encrypted payload
        |
        v
Broadcom configuration
        |
        v
bcm2cfg
        |
        +--> userif.remote_acc_user
        `--> userif.remote_acc_pass
```

For the tested CGA2121 backup:

-   encrypted payload begins at byte **96**;
-   a big-endian `uint32` at offset **92** describes the size as
    `payload length + 80`;
-   encryption is **AES-256-ECB**;
-   the AES key is `SHA256()` of the gateway's **literal cable-modem MAC
    string**;
-   decrypted data uses AES zero padding;
-   the plaintext identifies itself with `CTNV` / `CGA2121`;
-   the backup checksum is based on MD5 with an additional fixed suffix
    described below.

The final decrypted file can be inspected with `bcm2cfg`.

## Requirements

You need:

-   a configuration backup exported from your own CGA2121;
-   the device's cable-modem MAC address in the exact format expected by
    the backup;
-   Python 3;
-   `pycryptodome`;
-   `bcm2cfg` for parsing the decrypted Broadcom configuration.

Keep all working files private.

## 1. Save the original backup

Assume the exported file is:

``` text
GatewaySettings.bin
```

Before doing anything else, keep an untouched copy:

``` bash
cp GatewaySettings.bin GatewaySettings.bin.original
```

Do not publish this file. It can contain device-specific and
subscriber-specific configuration.

## 2. Backup structure

The tested file contains a **96-byte header** followed by the encrypted
payload.

Conceptually:

``` text
0                              95 96
+--------------------------------+-------------------------+
|          96-byte header        | AES encrypted payload   |
+--------------------------------+-------------------------+
```

The payload therefore starts at:

``` text
offset = 96
```

A 32-bit big-endian value at offset `92` was found to represent:

``` text
encrypted_payload_length + 80
```

This information is useful for validating that a backup matches the
format described here.

## 3. Key derivation

The AES-256 key is derived from the **literal MAC-address string**:

``` text
AES_KEY = SHA256(MAC_STRING)
```

For example, use the gateway's actual cable-modem MAC in the same
textual representation expected by the configuration tool:

``` text
08:7e:64:XX:XX:XX
```

The MAC above is deliberately redacted.

The important detail is that the **text representation itself** is
hashed; this is not a conversion of the MAC to six raw bytes.

Conceptually:

``` python
from hashlib import sha256

mac = "08:7e:64:XX:XX:XX"
key = sha256(mac.encode()).digest()
```

The resulting digest is 32 bytes, which is the AES-256 key.

## 4. Install the Python dependency

The original workflow used PyCryptodome:

``` bash
python3 -m pip install pycryptodome
```

Prefer a Python virtual environment if installing packages on a managed
system.

## 5. Decrypt the backup

The original work used a helper script named `cgacfg.py`, with
operations such as:

``` bash
python3 cgacfg.py --info GatewaySettings.bin
python3 cgacfg.py --verify GatewaySettings.bin
python3 cgacfg.py -d -mac '08:7e:64:XX:XX:XX' GatewaySettings.bin
```

The exact command-line wrapper is less important than the format it
implements:

1.  read `GatewaySettings.bin`;
2.  preserve/inspect the first 96 bytes as the header;
3.  take the encrypted payload beginning at offset 96;
4.  calculate `SHA256(literal_MAC_string)`;
5.  decrypt the payload using AES-256-ECB;
6.  remove trailing AES zero padding as appropriate;
7.  write the resulting Broadcom configuration as a separate file.

The decrypted output used in the tested workflow was named:

``` text
GatewaySettings_decrypted.bin
```

### Minimal equivalent Python logic

The core decryption can be represented as:

``` python
from hashlib import sha256
from Crypto.Cipher import AES

INPUT = "GatewaySettings.bin"
OUTPUT = "GatewaySettings_decrypted.bin"

# Replace with the MAC from YOUR gateway.
MAC_STRING = "08:7e:64:XX:XX:XX"

with open(INPUT, "rb") as f:
    data = f.read()

header = data[:96]
encrypted = data[96:]

key = sha256(MAC_STRING.encode()).digest()

cipher = AES.new(key, AES.MODE_ECB)
plaintext = cipher.decrypt(encrypted)

# The tested format used zero padding.
plaintext = plaintext.rstrip(b"\x00")

with open(OUTPUT, "wb") as f:
    f.write(plaintext)

print(f"Wrote {OUTPUT}")
```

Do not use an example MAC copied from another subscriber. The key is
device-specific.

## 6. Validate the plaintext

A correctly decrypted CGA2121 configuration should contain recognizable
Broadcom/CGA2121 structure rather than random ciphertext.

In the tested backup, useful identifying data included:

``` text
CTNV
CGA2121
```

If the output remains random-looking, first verify:

-   the correct backup file;
-   payload offset `96`;
-   the correct cable-modem MAC;
-   exact MAC string formatting;
-   SHA-256 key derivation;
-   AES ECB mode.

Do not start modifying the file until decryption has been verified.

## 7. Backup checksum

During analysis of the tested format, the backup checksum was determined
to use:

``` text
MD5(file[16:] + b"2Pslc;u(egmd0-'x")
```

In other words, the checksum calculation includes the backup data
beginning at offset `16` followed by the fixed byte string:

``` text
2Pslc;u(egmd0-'x
```

This detail matters primarily when **rebuilding or validating** a
configuration backup. It is not required merely to read credentials from
an already decrypted configuration.

## 8. Read the configuration with `bcm2cfg`

Once `GatewaySettings_decrypted.bin` has been produced, use `bcm2cfg`
with the `generic` profile.

To read the remote administrative username:

``` bash
bcm2cfg -P generic get GatewaySettings_decrypted.bin userif.remote_acc_user
```

To read the remote administrative password:

``` bash
bcm2cfg -P generic get GatewaySettings_decrypted.bin userif.remote_acc_pass
```

On the tested SBB CGA2121 configuration, these two fields contained the
existing **SBB administrative account credentials**.

Do not include the returned values in screenshots, terminal transcripts,
Git commits, shell scripts, or issue reports.

## 9. Other related fields

The same configuration namespace also contained remote-access settings.

Examples encountered during the original analysis included:

``` text
userif.remote_acc_methods
userif.telnet_ipstacks
userif.ssh_ipstacks
```

For inspection, use `get` rather than changing the backup:

``` bash
bcm2cfg -P generic get GatewaySettings_decrypted.bin userif.remote_acc_methods
```

``` bash
bcm2cfg -P generic get GatewaySettings_decrypted.bin userif.telnet_ipstacks
```

``` bash
bcm2cfg -P generic get GatewaySettings_decrypted.bin userif.ssh_ipstacks
```

The original investigation also established that `bcm2cfg` supports
configuration changes using `set`, and that `+value` syntax can add
bitmask-style values. That is outside the scope of this
credential-reading guide; for extraction and documentation, keep the
decrypted backup read-only.

## 10. Complete extraction flow

``` text
Export GatewaySettings.bin
            |
            v
      inspect header
            |
            +--> header length = 96 bytes
            |
            v
 extract encrypted payload
            |
            v
 SHA256(literal CM MAC string)
            |
            v
      32-byte AES key
            |
            v
       AES-256-ECB
            |
            v
GatewaySettings_decrypted.bin
            |
            v
         bcm2cfg
            |
            +--> userif.remote_acc_user
            |
            `--> userif.remote_acc_pass
```

## 11. Troubleshooting

### Decrypted output looks like random data

The most likely causes are an incorrect MAC string, wrong MAC
formatting, wrong payload offset, or a backup from a different
firmware/format.

Verify the assumptions before trying alternate cryptography.

### `bcm2cfg` cannot parse the file

Confirm that you are passing the **decrypted payload**, not the original
encrypted `GatewaySettings.bin`.

Also verify that the plaintext contains the expected CGA2121/Broadcom
identifiers.

### `remote_acc_user` or `remote_acc_pass` is missing

Firmware and ISP provisioning can differ. Do not assume every CGA2121
backup contains the same keys.

You can inspect the configuration structure with `bcm2cfg`, but avoid
publishing the complete decrypted configuration because it can expose
unrelated credentials and provisioning data.

### The extracted account does not work

The presence of credentials in a backup does not guarantee that a
particular management service is enabled or reachable. Access can
additionally depend on interface bindings, local/remote access policy,
firmware settings, and ISP provisioning.

This guide documents **reading the values from your own backup**, not
bypassing network access controls.

## 12. Security and privacy

Before publishing any research based on a gateway backup, remove:

-   administrative usernames and passwords;
-   Wi-Fi credentials;
-   SIP/VoIP credentials;
-   subscriber identifiers;
-   device MAC addresses;
-   serial numbers;
-   public IP addresses where identifying;
-   TR-069/ACS credentials;
-   certificates and private keys;
-   raw or decrypted configuration backups.

The safest public repository contains only the **method, sanitized
examples, and tooling**, never an actual subscriber backup.

## 13. Disclaimer

This documentation is intended for backup analysis, interoperability,
recovery, and administration of devices you own or are explicitly
authorized to manage. ISP terms and applicable regulations may apply.
