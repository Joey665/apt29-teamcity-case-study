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
