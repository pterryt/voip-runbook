# VoIP.ms and Polycom IP550 setup runbook

## What we know

- The provider name appears to be **VoIP.ms**, not `void.ms`.
- The requested point of presence (POP) is Houston 2, whose SIP server is
  `houston2.voip.ms`.
- Polycom SoundPoint IP550 phones can be configured manually for VoIP.ms.
- The IP550 is not listed in VoIP.ms's current automatic-provisioning device
  list.

This runbook assumes "cloud PBX" means the PBX features built into VoIP.ms.
If a separate PBX product is involved, stop before configuring a phone. The
phones would normally register to that PBX instead of directly to VoIP.ms.

## Three facts to confirm

1. Does "four lines" mean four phone numbers (DIDs), four physical IP550
   phones, or four line keys on one phone?
2. Is the cloud PBX VoIP.ms itself, or a separate product such as 3CX,
   FreePBX, or another hosted provider?
3. For incoming calls, should each number ring one phone, should a main number
   ring every phone, or should callers hear an IVR menu?

Do not send account or SIP passwords in chat, screenshots, or this repository.

## Recommended architecture

For four physical phones registered directly to VoIP.ms, create one unique
SIP subaccount per phone. Do not reuse one subaccount on several phones.
Register all phones to `houston2.voip.ms`, and set every incoming DID's POP to
the same Houston 2 server.

A simple extension plan is:

| Phone | Subaccount suffix | Value entered for extension | Dialed extension |
| --- | --- | ---: | ---: |
| Reception | `reception` | 1 | 101 |
| Phone 2 | `phone2` | 2 | 102 |
| Phone 3 | `phone3` | 3 | 103 |
| Phone 4 | `phone4` | 4 | 104 |

VoIP.ms adds the leading `10` to the value entered in its Internal Extension
field. Test this plan with one phone before creating the other three.

## Phase 1: inventory one pilot phone

Use one phone that can be interrupted during testing.

1. Photograph its current cabling and record its MAC address, IP address,
   firmware version, and any existing provider or provisioning-server setting.
2. Confirm it has power, an Ethernet connection, and an IP address other than
   `0.0.0.0`.
3. Do not factory-reset it until the owner confirms the existing configuration
   is no longer needed. A former provider's provisioning server can also
   overwrite manual settings after a reboot.
4. From the phone, find its IP under `Menu > Status > Network > TCP/IP
   Parameters`. Menu wording can vary slightly with firmware.
5. From a computer on the same LAN, open the phone's IP address in a browser.
   Older firmware may expose only HTTP. The historical default administrator
   credentials are user `Polycom` and password `456`; use the owner's current
   credentials if they were changed.

## Phase 2: create the pilot subaccount

In the VoIP.ms customer portal:

1. Open `Sub Accounts > Create Sub Account`.
2. Set Protocol to `SIP`.
3. Set Authentication to `User/Password Authentication`.
4. Choose a unique suffix such as `reception` and generate a strong, unique
   SIP password. The resulting username has a form such as
   `123456_reception`.
5. Select the device type for an ATA, IP phone, or softphone.
6. Set NAT to `Yes` when the phone is behind the usual office router.
7. Choose an outbound caller-ID number that belongs to the customer.
8. Initially disable international calling unless it is actually required.
9. In Internal Extension Number, enter `1` for extension `101`.
10. Save the subaccount. Store the SIP password in an approved password
    manager, not in this runbook.

## Phase 3: configure line 1 on the pilot IP550

Exact labels vary by firmware. In the phone's web interface:

1. Open the `SIP` page and configure Server 1:
   - Address: `houston2.voip.ms`
   - Port: `5060`
   - Transport: `UDPonly` or `UDP`
   - Expires: `60`
   - Register: enabled or `1`
   - Server 2: blank
2. Submit the SIP settings and allow the phone to reboot.
3. Open the `Lines` or `Line` page and configure Line 1:
   - Display Name: the user's name or an approved caller-ID name
   - Address: the full VoIP.ms subaccount username
   - Auth User ID: the same full subaccount username
   - Auth Password: that subaccount's SIP password
   - Label: `101` or a short desk name
4. Submit the line settings and allow another reboot.
5. Under audio or codec settings, make `G.711 mu-law` (`G711Mu` or `PCMU`)
   the first codec with a 20 ms payload. Ensure the subaccount permits the same
   codec.
6. Set DTMF to RFC 2833, sometimes labeled RTP Events or `telephone-event`.
7. Optionally set the message-center callback contact to `*97` after a VoIP.ms
   mailbox is associated with the subaccount.

## Phase 4: verify the pilot before routing live calls

1. In `Main Menu > Portal Home`, find Sub-Account Registration Status and
   confirm the pilot subaccount shows as registered.
2. From the phone, dial `4443` for the VoIP.ms echo test. Confirm audio works
   in both directions.
3. Place a normal outbound call and confirm audio in both directions and the
   displayed caller ID.
4. For one test DID, open `DID Numbers > Manage DIDs > Edit DID`:
   - set its POP to Houston 2;
   - route it directly to the pilot subaccount; and
   - save and apply the changes at the bottom of the page.
5. Call that DID from a mobile phone. Confirm ringing, answering, two-way
   audio, and hangup.
6. If emergency service is required, activate it with the correct service
   address and make the subaccount's caller ID exactly match the enabled DID.
   Test with `1-555-555-0911`; do **not** place a test call to 911.

## Phase 5: add the remaining phones and call flow

Once the pilot passes every test, repeat Phases 1 through 4 with a unique
subaccount and password for each physical phone.

Then choose one incoming design:

- One DID per phone: route each DID directly to its phone's subaccount.
- One main number rings several phones: create a ring group containing the
  desired subaccounts, then route the DID to that ring group.
- Menu-driven routing: create destinations first, then an IVR, and route the
  main DID to the IVR.

All ring-group members and their DID must use the same POP. Internal extension
calling also requires the phones to be registered on the same server.

## If the pilot does not work

- Not registered: recheck the full subaccount username, SIP password, server,
  port, transport, and whether old provisioning settings return after reboot.
- Registered but no incoming calls: confirm the DID is routed to the correct
  subaccount and that the DID POP and phone server are both Houston 2.
- Calls drop or registration disappears: confirm NAT is `Yes`; check router
  UDP timeout or keepalive behavior. Disable SIP ALG if it is altering SIP
  traffic.
- One-way or no audio: check the router/firewall and SIP ALG, then confirm both
  sides allow `G.711 mu-law`.
- Extensions fail: verify all subaccounts are on Houston 2. An older Polycom
  digit map may also need to allow three-digit patterns such as `[1-9]xx`.
- Web settings cannot be changed: look for an active provisioning server,
  provider lock, unknown administrator password, or obsolete firmware before
  resetting anything.

## References

- [VoIP.ms subaccounts](https://wiki.voip.ms/article/Sub_Accounts)
- [Phone setup](https://wiki.voip.ms/article/Polycom_SoundPoint_IP_501)
- [DID troubleshooting](https://wiki.voip.ms/article/DID_Troubleshooting)
- [VoIP.ms ring groups](https://wiki.voip.ms/article/Ring_Group)
- [VoIP.ms emergency services](https://wiki.voip.ms/article/E911)
- Poly Quick Tip 44011, *Registering Standalone Polycom SoundPoint IP,
  SoundStation IP, and VVX 1500 Phones* (available from HP Support)
