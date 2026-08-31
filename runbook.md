# VoIP.ms and Polycom IP550 setup runbook

## What we know

- The provider name appears to be **VoIP.ms**, not `void.ms`.
- The requested point of presence (POP) is Houston 2, whose SIP server is
  `houston2.voip.ms`.
- The account has four inbound phone numbers (DIDs).
- The cloud PBX is the PBX functionality hosted by VoIP.ms.
- A call to any of the four DIDs should ring about 15 physical phones together.
- Polycom SoundPoint IP550 phones can be configured manually for VoIP.ms.
- The IP550 is not listed in VoIP.ms's current automatic-provisioning device
  list.

## Confirmed call flow

```text
DID 1 --\
DID 2 ---\
DID 3 ----> "All Phones" queue --> every available phone subaccount
DID 4 ---/
```

Do not send account or SIP passwords in chat, screenshots, or this repository.

## Recommended architecture

Create one unique SIP subaccount for each physical phone registered directly
to VoIP.ms. Do not reuse one subaccount on several phones. Register every phone
to `houston2.voip.ms`, and set all four incoming DIDs' POP to the same Houston
2 server. Route all four DIDs to one Calling Queue using the `Ringall`
strategy. Add every phone's subaccount as a static queue member with the same
priority.

The first four entries in a simple extension plan could be:

| Phone | Subaccount suffix | Value entered for extension | Dialed extension |
| --- | --- | ---: | ---: |
| Reception | `reception` | 1 | 101 |
| Phone 2 | `phone2` | 2 | 102 |
| Phone 3 | `phone3` | 3 | 103 |
| Phone 4 | `phone4` | 4 | 104 |

VoIP.ms adds the leading `10` to the value entered in its Internal Extension
field. Test this plan with one phone before creating the remaining accounts.

A normal VoIP.ms ring group is not suitable because it permits only eight SIP
members. VoIP.ms documents no fixed limit for static members in a Calling
Queue, and its `Ringall` strategy rings every available member until one
answers.

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
4. Temporarily use one DID for the pilot test. Open
   `DID Numbers > Manage DIDs > Edit DID`:
   - set its POP to Houston 2;
   - route it directly to the pilot subaccount; and
   - save and apply the changes at the bottom of the page.
5. Call that DID from a mobile phone. Confirm ringing, answering, two-way
   audio, and hangup.
6. If emergency service is required, activate it with the correct service
   address and make the subaccount's caller ID exactly match the enabled DID.
   Test with `1-555-555-0911`; do **not** place a test call to 911.

## Phase 5: add the remaining phones

Once the pilot passes every test, repeat Phases 1 through 3 with a unique
subaccount, extension, and password for each physical phone. For each phone,
confirm portal registration, echo-test audio, and outbound calling. Do not
route a separate DID directly to each phone.

## Phase 6: create the All Phones queue

In the VoIP.ms portal:

1. Open `DID Numbers > Calling Queues`.
2. Select `Create New Call Queue`.
3. Choose an unused Queue Number and name it `All Phones`.
4. Leave Queue Password, Caller ID Prefix, Join Announcement, and Agent
   Announcement blank initially.
5. Set Ring Strategy to `Ringall`.
6. Set Member Delay and Wrap-up Time to `0`.
7. Set Agent Ring Timeout to about 25 seconds.
8. Set Maximum Wait Time to about 30 seconds.
9. Set Retry Timer to about 5 seconds.
10. Configure callers to fail over when no members are registered or available.
11. Set the Timeout and unavailable failovers to a shared voicemail box or the
    customer's preferred fallback.
12. Disable position and estimated-wait announcements for a simple
    ring-every-phone experience.
13. Select an acceptable hold sound. The queue starts billing when a call
    enters it, and the caller may hear hold audio while phones ring.
14. Save the queue.

The exact `Join when empty`, `Leave when empty`, and `Ring in-use` labels can
be confusing. During the pilot, verify these outcomes rather than relying only
on the labels:

- an unregistered phone does not trap the caller in the queue;
- a caller reaches voicemail after about 30 seconds with no answer; and
- a phone already on a call is skipped unless call waiting is desired.

## Phase 7: add static queue members

1. Edit the static members for the `All Phones` queue.
2. Add the pilot phone's subaccount and give it priority `0`.
3. As each later phone is configured, add its unique subaccount with the same
   priority.
4. Do not require staff to log in and out. Static members participate whenever
   their SIP subaccounts are registered.

VoIP.ms allows as many static members as needed in a queue. Equal priority and
the `Ringall` strategy make all available members ring together.

## Phase 8: route all four DIDs to the queue

Repeat these steps for each of the four DIDs:

1. Open `DID Numbers > Manage DIDs` and edit the DID.
2. Set its POP to Houston 2.
3. Set Routing to `Calling Queue` and select `All Phones`.
4. Review any Busy, Unreachable, or No Answer failover settings.
5. Scroll to the bottom and apply the changes.

Changing routing affects live incoming calls, so perform this phase during an
agreed test window. It also replaces the temporary direct-to-pilot route.

## Phase 9: acceptance test

Call each of the four DIDs from an outside phone and record the result:

| Test | Expected result |
| --- | --- |
| DID 1 through DID 4 | Every available phone rings |
| Answer on one phone | Other phones stop ringing |
| Two-way audio | Both parties hear each other |
| Caller hangs up | The phone releases the call |
| No phone answers | Shared voicemail or chosen failover runs |
| Internal extension | The selected phone rings directly |

Answer at least one test call on each physical phone. If simultaneous inbound
calls matter, also place two calls at once. Available concurrent calls depend
on each DID's billing plan and channel capacity shown in the portal.

All queue members and all four DIDs must use the same POP. Internal extension
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
- [Calling queues](https://wiki.voip.ms/article/Calling_Queues)
- [VoIP.ms emergency services](https://wiki.voip.ms/article/E911)
- Poly Quick Tip 44011, *Registering Standalone Polycom SoundPoint IP,
  SoundStation IP, and VVX 1500 Phones* (available from HP Support)
