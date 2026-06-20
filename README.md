# When the Build Server Becomes the Target: A Case Study on APT29 and the TeamCity Compromise

*Security analysis by Joseph Bright*

## Why This Case Study

In December 2023, the FBI, CISA, NSA, and three allied agencies published a joint advisory. Russia's Foreign Intelligence Service — APT29, also known as Midnight Blizzard, the same group behind SolarWinds — had been exploiting a vulnerability in JetBrains TeamCity since September 2023.

TeamCity is a build automation tool. Developers use it to compile, test, and ship software. That makes it infrastructure, not an application. Compromise the build server, and you don't just compromise one company. You compromise everything that company ships afterward.

I picked this advisory for one reason. It is one of the most thoroughly documented nation-state campaigns ever published by Western intelligence agencies, and it follows the exact pattern I've been tracking in my own writing — attackers no longer breaking through the front door, but walking in through the tools developers trust.

This is a structured breakdown: what happened, how a defender would catch it, and how I would explain the risk to someone who doesn't read CVE numbers for a living.

---

## Part 1 — The Attack, Mapped

I pulled seven distinct techniques out of the advisory and mapped them to MITRE ATT&CK. This is the adversary's actual playbook, not a hypothetical one.

| Tactic | Technique ID | What Actually Happened |
|---|---|---|
| Initial Access | T1190 | APT29 exploited CVE-2023-42793 on internet-facing TeamCity servers — an authorization bypass that handed them arbitrary code execution with no authentication required |
| Discovery | T1033 / T1592.002 | Once inside, they ran basic Windows commands — whoami, nltest, tasklist, netstat — to map the host and the domain around it |
| Credential Access | T1003.001 / T1003.002 | Mimikatz, run entirely in memory, pulled credentials straight out of LSASS and the SAM database |
| Defense Evasion | T1027.001 / T1564.001 | This is the part that stood out to me. Their backdoor, GraphicalProton, exfiltrated stolen data by hiding it inside randomly generated BMP image files, then uploaded those images to OneDrive and Dropbox |
| Persistence | T1053.005 | Scheduled tasks, disguised with names like "WindowsDefenderService," kept the backdoor alive across reboots |
| Command and Control | T1572 | A modified open-source SOCKS tunneler, renamed rr.exe, tunneled traffic out to their infrastructure |
| Exfiltration | T1567 | The SYSTEM, SAM, and SECURITY registry hives — effectively the keys to the whole domain — were zipped up and pushed out through the same OneDrive/Dropbox channel |

The detail I keep coming back to is the BMP file trick. Nobody's network monitoring tool flags an image upload to OneDrive. That's the same principle I wrote about in the npm/Hugging Face campaign a few weeks ago — abuse a platform everyone already trusts, and your traffic disappears into the noise.

---

## Part 2 — If This Hit Your Network, Here's the Response

A technical playbook means nothing if it stays theoretical, so I wrote this the way a SOC would actually have to run it, phase by phase.

### Identification

You're looking for two things that don't belong. First, unexpected entries in the TeamCity server log showing modifications to internal.properties by a user ID that shouldn't be making that change. Second, any `reg.exe` process running with `save` against `HKLM\SYSTEM`, `HKLM\SAM`, or `HKLM\SECURITY` — that command has almost no legitimate reason to run outside of a domain controller backup job. CISA published working SIGMA rules for exactly this. Deploy them before you need them, not after.

### Containment

Pull the compromised TeamCity host off the network immediately. Do not let it finish any build job that's in progress — a compromised build server can push a malicious artifact downstream before you've even confirmed the breach. Kill every active session token tied to that host. Block outbound traffic to OneDrive and Dropbox from it specifically. A CI server has no legitimate reason to be talking to consumer cloud storage, so that traffic pattern is itself worth a standing detection rule, breach or not.

### Eradication

Patch CVE-2023-42793. If you haven't by the time you're reading this, that's the first five minutes, not the last step. Hunt across your entire environment — not just the one host — for the GraphicalProton DLLs by name: `AclNumsInvertHost.dll`, `ModeBitmapNumericAnimate.dll`, `UnregisterAncestorAppendAuto.dll`, and the others CISA listed. Strip out every scheduled task matching their known naming patterns. Then don't trust in-place cleanup — rebuild the host from a known-clean image.

### Recovery

Bring the server back from a verified clean build, not a patched version of the compromised one. Rotate every credential and certificate that host had access to. Assume anything reachable from that machine is burned. Only resume production builds once the patch is confirmed and your new detection rules are live.

### Lessons Learned

The CVE wasn't really the root cause. The root cause is that a build server was reachable from the open internet in the first place. Fix the architecture, not just the patch level. Build infrastructure should sit behind the same segmentation you'd put around a domain controller, because functionally, that's what it is.

---

## Part 3 — Explaining This to Someone Who Doesn't Read CVE Advisories

This is the part most security write-ups skip, and it's the part that actually gets budget approved.

**What happened, in plain terms:** A state-linked intelligence service broke into a tool that software companies use to build and release their products. From inside that tool, they had a path to source code, the certificates that make software look legitimate, and the pipeline that pushes updates to customers. This is the same category of access that led to the SolarWinds breach in 2020 — a single compromised build step turning into a supply chain attack against thousands of downstream customers.

**Why it matters financially:** This isn't just a downtime risk. If a build server is compromised, the bigger cost is rebuilding trust — with regulators, with customers, with anyone who has to ask "was the version of your software I'm running actually safe?" That question is expensive to answer honestly, and it's catastrophic if you can't.

**Where this maps to controls you already report against:** Under the NIST Cybersecurity Framework, this sits squarely in Protect (patch management, PR.IP) and Detect (continuous monitoring, DE.CM). Under ISO 27001, it falls under A.12.6 — technical vulnerability management — and A.14.2, security built into the development process itself.

**What I'd put in front of leadership, in order:**

1. Patch internet-facing build and CI/CD infrastructure within 48 hours of any vendor disclosure. Treat it as Tier 1 critical, not an internal tool nobody worries about.
2. Require MFA on every administrative path into build and deployment systems.
3. Get CI/CD infrastructure off the open internet entirely. It should be segmented the way you'd segment a domain controller.
4. Run red-team exercises specifically against the build pipeline, on a quarterly cadence — not just the general network penetration test that already happens once a year.

---

## What This Confirms

I've written about this pattern three times now — Nx Console, the npm/Hugging Face campaign, and now this. The technique doesn't change. The infrastructure attackers abuse to hide does. OneDrive and Dropbox here. Hugging Face there. The lesson holds across all three: the more your security posture assumes a known platform is automatically safe, the more exposed you are to exactly this kind of attack.

The TeamCity CVE has been patched since September 2023. The technique it enabled has not gone anywhere.
