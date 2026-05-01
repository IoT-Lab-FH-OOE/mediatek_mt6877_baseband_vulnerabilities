| Title             | Transient Denial of Service Vulnerabilities in MediaTek Dimensity 1080/7050 (MT6877)'s 5G Baseband |
| ----------------- | -------------------------------------------------------------------------------------------------- |
| Affected Product  | MediaTek Dimensity 1080/7050 (MT6877) system-on-chip (SoC) with integrated 5G baseband             |
| Affected Firmware | `MOLY.NR15.R3.TC8.PR5.SP.V5.P141` (older versions may also be affected)                            |
| CVE IDs           | [CVE-2025-20792](https://www.cve.org/CVERecord?id=CVE-2025-20792), [CVE-2026-20422](https://www.cve.org/CVERecord?id=CVE-2026-20422), [CVE-2026-20421](https://www.cve.org/CVERecord?id=CVE-2026-20421), [CVE-2026-20402](https://www.cve.org/CVERecord?id=CVE-2026-20402), [CVE-2026-20401](https://www.cve.org/CVERecord?id=CVE-2026-20401), [CVE-2026-20420](https://www.cve.org/CVERecord?id=CVE-2026-20420) |
| Vendor Website    | [https://www.mediatek.com](https://www.mediatek.com)                                               |
| Identified in     | July - August 2025                                                                                 |
| Identified by     | IoT Lab, University of Applied Sciences Upper Austria, Campus Hagenberg                            |
| Website           | [https://www.fh-ooe.at/si/](https://www.fh-ooe.at/si/)                                             |
| Team              | Denis Krämer B.Sc., Dieter Vymazal M.Sc., DI Markus Zeilinger                                      |
| Contact           | Denis Krämer <br> <denis.kraemer@students.fh-hagenberg.at> <br> 7B5B 5FCD 14A2 4A20 C43A A6D0 8B78 FC61 E205 BE39 |

## Vendor Description

*"MediaTek designs and develops advanced system-on-chip (SoC) solutions for mobile devices, home entertainment, connectivity, and IoT (Internet of Things) products. [...]"*

Source: [https://www.mediatek.com/company/discover](https://www.mediatek.com/company/discover)

## Overview

During the 5G NR initial attachment procedure, unauthenticated radio resource control (RRC) messages are used to set up a connection between the baseband and a base station (gNB) of the 5G mobile network. By modifying downlink RRC setup messages in a specific manner, several reachable assertions and fatal errors in the baseband firmware of the MediaTek Dimensity 1080 (rebranded as Dimensity 7050) can be triggered, each of which result in an exception and restart of the modem. Consequently, the mobile network connection of the Xiaomi Redmi Note 12 Pro+ smartphone (which uses this SoC) becomes temporarily unavailable, and the modem keeps restarting until the transmission of the modified RRC packets via the gNB is stopped. This classifies each of the vulnerabilities as transient DoS.

## Impact

An attacker can use a malicious gNB to send modified RRC setup messages to nearby MediaTek basebands and continuously trigger a modem reset, which results in a loss of mobile network connectivity for these subscribers until the attack is stopped. The attack requires no knowledge of any confidential information stored on the universal integrated circuit cards (UICCs) of the targets, because the RRC setup message is sent by the network to the baseband before any mutual authentication (5G-AKA) is performed. Only the mobile country code (MCC) and mobile network code (MNC) of the target's network is required to successfully execute such an attack. Since the [ITU-T assigns and publishes the MCC/MNC of public networks](http://handle.itu.int/11.1002/pub/82153a48-en), and only a handful of them exist per country, we argue that this requirement adds little complexity to the attack. Moreover, while the attack requires physical proximity to the target, it requires no interaction on behalf of the user.

## Proof of Concept

In the following proof of concept (PoC) exploitation video for vulnerability V1, a modified RRC setup message is sent to the baseband upon connecting to a malicious nearby gNB. An exception in the firmware is triggered and the modem restarts, which can be verified by the Android Debug Bridge (adb) logs of the smartphone. The adb logs contain further information about the location of the exception. Moreover, a temporary loss of connectivity of the Android smartphone can be observed.

<video controls muted preload><source src="/mediatek_mt6877_baseband_vulnerabilities/assets/videos/V1_poc_exploit_2x.mp4" type="video/mp4"></video>

## Reproduction

We used the [5Ghoul packet interception API](https://github.com/asset-group/5ghoul-5g-nr-attacks#4--create-your-own-5g-exploits-test-cases) to write test cases that modify downlink traffic and trigger the vulnerabilities, but any toolset capable of performing such modifications is suitable to execute the attacks. We then used an RF shielded test environment comprised of a software-defined radio (SDR) and the affected user equipment (UE) to transmit modified RRC setup messages to the baseband. The traffic captured on the NR-Uu interface while triggering the vulnerabilities is shown in the provided Wireshark screenshots, which each depict the original RRC message on the left-hand side (not sent to the baseband) and the modified RRC message on the right-hand side (sent to the baseband). Finally, we observed several modem exceptions by inspecting the adb logs of the smartphone.

## Vulnerability V1: invalid pdsch-ServingCellConfig and csi-MeasConfig setup ([CVE-2025-20792](https://www.cve.org/CVERecord?id=CVE-2025-20792))

Triggering vulnerability V1 results in the following adb log entry:

```
[Fatal error(MPU_NOT_ALLOW)] err_code1:0x0000001D err_code2:0x90FE3166 err_code3:0x00000000
```

To reproduce this issue, starting from the MAC-NR layer, change the byte at offset 123 of a valid RRC setup message to any of the following values: `{0x42}` or `{0x4c}`, e.g. `{0x4c}` (as shown in Figure 1). This corresponds to changing the 1st and 2nd byte of the `pdsch-ServingCellConfig` field inside the RRC payload, e.g. to `{0x09, 0x90}`. According to Wireshark, this changes its `setup` into a `release` field. Moreover, this modification alters several of the configuration lists contained in the subsequent `csi-MeasConfig setup` field.

<details open>
<summary>Figure 1: Comparison of the original and modified RRC setup message for vulnerability V1 in Wireshark.</summary>
<img src="assets/images/V1_rrc_setup_original_vs_modified.png" alt="">
</details>

Note: this vulnerability can also be triggered by changing the byte at offset 128 of a valid RRC setup message to the value `{0xe0}` or `{0xe1}`. The latter results in the following adb log entry (note the different `err_code2`):

```
[Fatal error(MPU_NOT_ALLOW)] err_code1:0x0000001D err_code2:0x90FE3088 err_code3:0x00000000
```

This corresponds to changing the 2nd and 3rd byte of the `csi-MeasConfig setup` field inside the RRC payload to `{0x5c, 0x30}`. According to Wireshark, this alters the type of the therein contained `csi-SSB-ResourceSetToAddModList`, `csi-ResourceConfigToAddModList` and `csi-ReportConfigToAddModList` fields or changes their contents to invalid values.

## Vulnerability V2: invalid pdsch-Config and truncated uplinkConfig ([CVE-2026-20422](https://www.cve.org/CVERecord?id=CVE-2026-20422))

Triggering vulnerability V2 results in any of the following adb log entries:

```
[Fatal error(task)] err_code1:0x00003107 err_code2:0x00010008 err_code3:0xCCCCCCCC
[Fatal error(task)] err_code1:0x00003107 err_code2:0x00010040 err_code3:0xCCCCCCCC
[Fatal error] err_code1:0x00000020 err_code2:0x7CE70FC3 err_code3:0x00000000
[ASSERT] file:dsp3/coresonic/msonic/modem/tx/nr/tx/src/nr_tx_hwctrl.c line:2258
```

To reproduce this issue, starting from the MAC-NR layer, change the following bytes at the respective offsets of a valid RRC setup message (as shown in Figure 2):

| Offset | Value |
| ------ | ----- |
| 60     | 0xe5  |
| 61     | 0xdc  |

This corresponds to changing the 46th to 48th byte inside the RRC payload to `{0x1c, 0xbb, 0x89}`. According to Wireshark, this alters the `pdsch-Config` field and truncates the `uplinkConfig` field, which are both contained in the `spCellConfig` field.

<details open>
<summary>Figure 2: Comparison of the original and modified RRC setup message for vulnerability V2 in Wireshark.</summary>
<img src="assets/images/V2_rrc_setup_original_vs_modified.png" alt="">
</details>

Note: After the crash, the modem hangs for several minutes and requires a subsequent reset in order to resume normal operation (i.e. re-establish connectivity with the mobile network). Occasionally, the modem crashes again with the following adb log entry:

```
[Others] MD watchdog timeout interrupt
```

The loss of connectivity sometimes persists even after a modem reset, necessitating a manual reboot of the UE in order to resume normal operation.

## Vulnerability V3: invalid csi-ResourceConfigToAddModList and csi-ReportConfigToAddModList ([CVE-2026-20421](https://www.cve.org/CVERecord?id=CVE-2026-20421))

Triggering vulnerability V3 results in the following adb log entry:

```
[Fatal error(MPU_NOT_ALLOW)] err_code1:0x0000001D err_code2:0x90FE3AD4 err_code3:0x00000001
```

To reproduce this issue, starting from the MAC-NR layer, change the following bytes at the respective offsets of a valid RRC setup message (as shown in Figure 3):

| Offset | Value |
| ------ | ----- |
| 153    | 0xaa  |
| 154    | 0x33  |

This corresponds to changing the 139th to 141st byte inside the RRC payload to `{0x15, 0x46, 0x60}`. According to Wireshark, this alters the `csi-ResourceConfigToAddModList` and `csi-ReportConfigToAddModList` inside the `csi-MeasConfig` field.

<details open>
<summary>Figure 3: Comparison of the original and modified RRC setup message for vulnerability V3 in Wireshark.</summary>
<img src="assets/images/V3_rrc_setup_original_vs_modified.png" alt="">
</details>

## Vulnerability V4: invalid and truncated spCellConfig ([CVE-2026-20402](https://www.cve.org/CVERecord?id=CVE-2026-20402))

Triggering vulnerability V4 results in the following adb log entry:

```
[ASSERT] file:mcu/l1/nl1/internal/md97/src/tx/nr_tx_database_mcu.c line:4058
```

To reproduce this issue, starting from the MAC-NR layer, change the following bytes at the respective offsets of a valid RRC setup message (as shown in Figure 4):

| Offset | Value |
| ------ | ----- |
| 36     | 0xa9  |
| 37     | 0x04  |
| 38     | 0x8d  |
| 39     | 0x81  |

This corresponds to changing the 23rd to 26th byte inside the RRC payload to `{0x20, 0x91, 0xb0, 0x3f}`. According to Wireshark, this alters and truncates the `spCellConfig` field. Notably, it changes the type of several fields contained therein, leading to their reinterpretation as `radioLinkMonitoringConfig` and `uplinkBWP-ToAddModList` fields.

<details open>
<summary>Figure 4: Comparison of the original and modified RRC setup message for vulnerability V4 in Wireshark.</summary>
<img src="assets/images/V4_rrc_setup_original_vs_modified.png" alt="">
</details>

## Vulnerability V5: invalid monitoringSymbolsWithinSlot ([CVE-2026-20401](https://www.cve.org/CVERecord?id=CVE-2026-20401))

Triggering vulnerability V5 results in the following adb log entry:

```
[ASSERT] file:dsp3/coresonic/msonic/modem/brp/nr/nr_cbrp/src/nr_cbrp_cfg_task.c line:4890
```

To reproduce this issue, starting from the MAC-NR layer, change the following bytes at the respective offsets of a valid RRC setup message (as shown in Figure 5):

| Offset | Value |
| ------ | ----- |
| 50     | 0x17  |
| 51     | 0xbf  |

This corresponds to changing the 37th and 38th byte inside the RRC payload to `{0xf7, 0xe0}`. According to Wireshark, this changes the `monitoringSymbolsWithinSlot` field of the `searchSpacesToAddModList` inside the `spCellConfig pdcch-Config` field to an invalid value.

<details open>
<summary>Figure 5: Comparison of the original and modified RRC setup message for vulnerability V5 in Wireshark.</summary>
<img src="assets/images/V5_rrc_setup_original_vs_modified.png" alt="">
</details>

## Vulnerability V6: invalid pucch-Config and pusch-Config ([CVE-2026-20420](https://www.cve.org/CVERecord?id=CVE-2026-20420))

Triggering vulnerability V6 results in the following adb log entry:

```
[ASSERT] file:dsp3/coresonic/msonic/modem/tx/nr/tx/src/nr_tx_pwr_ctrl.c line:4691
```

To reproduce this issue, starting from the MAC-NR layer, change the following bytes at the respective offsets of a valid RRC setup message (as shown in Figure 6):

| Offset | Value |
| ------ | ----- |
| 94     | 0xc9  |
| 95     | 0x84  |
| 96     | 0x51  |
| 97     | 0x44  |
| 98     | 0xc3  |
| 99     | 0x69  |

This corresponds to changing the 80th to 85th byte inside the RRC payload to `{0x39, 0x30, 0x8a, 0x28, 0x98, 0x6d}`. According to Wireshark, this alters the `pucch-Config` and `pusch-Config` fields inside the `spCellConfig uplinkConfig` field, and truncates the remaining RRC setup message.

<details open>
<summary>Figure 6: Comparison of the original and modified RRC setup message for vulnerability V6 in Wireshark.</summary>
<img src="assets/images/V6_rrc_setup_original_vs_modified.png" alt="">
</details>

## Affected Product and Firmware

The following product and firmware were tested and found to be affected by the vulnerabilities:

* Smartphone: Xiaomi Redmi Note 12 Pro+ (MediaTek Dimensity 1080/7050 [MT6877] with integrated 5G baseband)
* Firmware: Android 14, OS version: `1.0.16.0.UMOEUXM`, Build: `UP1A.231005.007`, Patch level: 2025-04-01, Baseband version: `MOLY.NR15.R3.TC8.PR5.SP.V5.P141` (older versions may also be affected)

SHA-256 hash of the affected baseband firmware image (bundled by Xiaomi):
```
1170c9646450d25bd7263ab32a0c7df04d2a1a9c9b57fe1e6d4e572fa699b99c  md1img.img
```

## Solution

MediaTek publicly disclosed vulnerability V1 as part of their [December 2025 Product Security Bulletin](https://corp.mediatek.com/product-security-bulletin/December-2025) and the vulnerabilities V2 - V6 as part of their [February 2026 Product Security Bulletin](https://corp.mediatek.com/product-security-bulletin/February-2026). Moreover, MediaTek stated that they have released updated baseband firmware which fixes these vulnerabilities. MediaTek recommends that users reach out to their device original equipment manufacturer (OEM) if they would like to confirm which baseband firmware version includes the patch.

## Vendor Statement

The vendor has requested to include the following official statement by Tiger Hsu, [Product Security](https://corp.mediatek.com/security-announcement) Officer at MediaTek, in the public release of these vulnerabilities:

*"Security and user protection remain top priorities for all MediaTek platforms. In response to the cellular baseband modem vulnerabilities identified by researchers from IoT-Lab-FH-OOE (University of Applied Sciences Upper Austria/Secure Information Systems/IoT-Lab), we have worked closely with our partners to validate the issues and provide timely patches to all affected ODM/OEM customers. At this time, we have no evidence that these vulnerabilities are being actively exploited in the wild. We strongly encourage users to keep their devices updated with the latest security patches."*

## Communication Timeline

### Vulnerability Report 1 (Vulnerability V1)

| Date       | Sender       | Description |
| ---------- | ------------ | ----------- |
| 2025-08-04 | Denis Krämer | Contacted MediaTek using their PGP public key and attached the first vulnerability report. |
| 2025-08-08 | -            | Received information that the mail has been rejected by MediaTek's mail server due to its size. |
| 2025-08-08 | Denis Krämer | Provided the vulnerability report in two chunks. |
| 2025-08-20 | Denis Krämer | Reminded MediaTek of the 90 days disclosure deadline. |
| 2025-09-02 | Denis Krämer | Sent the report again using another mail account, included a statement that 1/3 of the time until disclosure has passed. |
| 2025-09-03 | MediaTek     | Responds that they did not receive any previous mails or attachments, requests upload of attachments to Google Drive. |
| 2025-09-03 | Denis Krämer | Provided the report and screenshots of failed communication attempts via Google Drive. |
| 2025-09-03 | MediaTek     | Acknowledges receipt of the report. |
| 2025-09-08 | MediaTek     | States that the previous mails were incorrectly classified as malicious or phishing attempts. |
| 2025-09-11 | MediaTek     | Asks for clarification regarding the public disclosure date. |
| 2025-09-11 | Denis Krämer | Provided the updated disclosure date (2025-12-02). |
| 2025-09-11 | MediaTek     | Asks how the disclosure will take place and for a draft of the documents, offers to provide an official statement. |
| 2025-09-15 | Denis Krämer | Outlined the form and content of the advisory, offered to include the official statement, asked for updates regarding the vulnerabilities. |
| 2025-09-16 | MediaTek     | States that some of the vulnerabilities were previously reported by another researcher. |
| 2025-09-16 | MediaTek     | Requests a link to the blog where the advisories will be published. |
| 2025-09-16 | Denis Krämer | Provided an example of a previous advisory from IoT-Lab-FH-OOE. |
| 2025-09-17 | MediaTek     | Thanks for the additional information. |
| 2025-09-18 | MediaTek     | Clarifies that some of the vulnerabilities have been classified as duplicates (these were omitted from this advisory). |
| 2025-09-19 | MediaTek     | States that two vulnerabilities share the same root cause and have received a high severity rating (these have been cumulated in this advisory). |
| 2025-09-22 | -            | MediaTek shares the fix for vulnerability V1 with their OEM partners (according to a statement by MediaTek from 2025-11-25). |
| 2025-09-23 | Denis Krämer | Requested the CVE IDs and provided the information for the acknowledgements page. |
| 2025-09-25 | MediaTek     | Acknowledges the information, states that CVE IDs for duplicate findings will be available after their public disclosure. |
| 2025-10-14 | MediaTek     | Assigns CVE-2025-20792 to vulnerability V1, plans to release a patch and disclose it after an additional two months. |
| 2025-10-16 | Denis Krämer | Inquired whether CVE-2025-20792 will be included in MediaTek's December 2025 product security bulletin. |
| 2025-10-16 | MediaTek     | Clarifies that CVE-2025-20792 will be included in their December 2025 product security bulletin. |
| 2025-11-06 | MediaTek     | Provides an official statement to be included in the advisory, requests a draft before public release. |
| 2025-11-07 | Denis Krämer | Confirmed that MediaTek will receive a draft before release, asked for the firmware version that fixes CVE-2025-20792. |
| 2025-11-07 | MediaTek     | States that it cannot share this information. |
| 2025-11-24 | MediaTek     | Asks for a draft of the advisory again, clarifies that the vulnerabilities are reachable assertions that only lead to DoS. |
| 2025-11-24 | Denis Krämer | Provided a draft of this advisory. |
| 2025-11-25 | MediaTek     | Clarifies that it has no control over the firmware version string that different OEMs may use and thus cannot provide the information I requested before. |
| 2025-11-25 | Denis Krämer | Clarified that I want end-users to be able to verify if they are running a firmware version that contains fixes for these issues. Asked if MediaTek wants to provide an alternative text for the *Solution* section. |
| 2025-11-25 | MediaTek     | Provides an alternative text for this section. |
| 2025-11-25 | Denis Krämer | Incorporated the changes from MediaTek, asked if they greenlight the resulting text. |
| 2025-11-25 | MediaTek     | States that they are okay with the revised text. |
| 2025-11-25 | MediaTek     | States that they have shared a fix for CVE-2025-20792 with their OEM partners on 2025-09-22. |
| 2025-11-25 | Denis Krämer | Replied that I have included this information in the timeline. |
| 2025-11-26 | MediaTek     | Asks to share the updated version of the advisory. |
| 2025-11-26 | Denis Krämer | Provided the revised draft of the advisory. |
| 2025-11-26 | MediaTek     | Thanks for the additional information. |
| 2025-12-01 | -            | MediaTek releases their December 2025 security bulletin. |
| 2025-12-01 | Denis Krämer | Asked to correct the credit information on their acknowledgements page, since it was incomplete. |
| 2025-12-01 | MediaTek     | States that the credit information was updated. |
| 2025-12-01 | Denis Krämer | Asked to verify if the update was successful, since it still showed the old information. |
| 2025-12-01 | -            | The credit information was corrected. |
| 2025-12-02 | MediaTek     | States that the credit information was already updated. |

### Vulnerability Report 2 (Vulnerabilities V2 - V6)

| Date       | Sender       | Description |
| ---------- | ------------ | ----------- |
| 2025-10-07 | Denis Krämer | Contacted MediaTek using their PGP public key and attached the second vulnerability report. |
| 2025-10-08 | MediaTek     | Acknowledges receipt of the report. |
| 2025-10-16 | MediaTek     | Inquires if a public disclosure is planned. |
| 2025-10-16 | Denis Krämer | Communicated the 90 days disclosure plan with the intention of a simultaneous disclosure. |
| 2025-10-21 | MediaTek     | Is still reviewing the report, states that a disclosure in February 2026 is likely. |
| 2025-10-21 | Denis Krämer | Confirmed that the vulnerabilities will be kept confidential until their official disclosure. |
| 2025-10-23 | MediaTek     | Requests additional information on vulnerability V2. |
| 2025-10-23 | Denis Krämer | Provided the requested information to MediaTek. |
| 2025-10-27 | MediaTek     | Confirms receipt of the information. |
| 2025-10-29 | MediaTek     | Provides an intermediary update on their assessment of the reported issues (any issues marked as duplicates were omitted from this advisory). |
| 2025-11-03 | MediaTek     | Is unable to reproduce vulnerability V2, asks to reproduce the issue on a Dimensity 9400 (MT6991)-based device. |
| 2025-11-03 | Denis Krämer | Asked if MediaTek it is willing to provide such a device, alternatively offered to update a Dimensity 1080 (MT6877)-based device to the latest firmware to try to reproduce the issue there. |
| 2025-11-03 | MediaTek     | States that it cannot provide such a device and recommends using their latest chipsets for security testing. |
| 2025-11-03 | Denis Krämer | Informed MediaTek that I am not able to perform the requested tests without such a device, asked if MediaTek is interested in maintaining the security of its older chipsets. |
| 2025-11-04 | MediaTek     | Clarifies that it still regards issues for its older chipsets as valid. |
| 2025-11-04 | MediaTek     | Provides an intermediary update on their assessment of the reported issues, requests further information on vulnerability V2. |
| 2025-11-04 | MediaTek     | Requests even more information on vulnerability V2. |
| 2025-11-04 | Denis Krämer | Provided the requested information to MediaTek. |
| 2025-11-04 | MediaTek     | Acknowledges the additional information. |
| 2025-11-07 | Denis Krämer | Informed MediaTek that I will provide a crash dump for vulnerability V2 once Xiaomi has granted the bootloader unlock for the testing device. |
| 2025-11-17 | MediaTek     | States that it was able to reproduce vulnerability V2, plans to disclose the vulnerabilities V2 - V6 in their February 2026 security bulletin. |
| 2025-11-19 | Denis Krämer | Thanked MediaTek for the information. |
| 2025-11-20 | MediaTek     | Provides further information on their assessment of the other reported issues. |
| 2025-12-01 | MediaTek     | States that it will disclose the issues in February 2026 along with the previous credit information. |
| 2026-01-19 | MediaTek     | Asks for a draft of the advisory. |
| 2026-01-20 | Denis Krämer | Replied that I require more time to finalize the draft. |
| 2026-03-16 | Denis Krämer | Provided a draft of the advisory. |
| 2026-03-17 | MediaTek     | Acknowledges receipt of the draft. |
| 2026-04-20 | Denis Krämer | Inquired about the current status on reviewing the draft. |
| 2026-04-21 | MediaTek     | States that it has reviewed the draft and has no comments. |

## Version History

| Date       | Version | Changes                                   |
| ---------- | ------- | ----------------------------------------- |
| 2025-11-24 | v0.1    | Initial draft                             |
| 2025-11-25 | v0.2    | Incorporated changes from MediaTek        |
| 2025-11-26 | v0.3    | Cosmetic changes                          |
| 2025-12-01 | v0.4    | Updated timeline                          |
| 2025-12-02 | v1.0    | Initial release of vulnerability V1       |
| 2026-05-01 | v2.0    | Add disclosure of vulnerabilities V2 - V6 |