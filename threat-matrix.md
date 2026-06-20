| Tactic | Technique ID | What Actually Happened |
|---|---|---|
| Initial Access | T1190 | APT29 exploited CVE-2023-42793 on internet-facing TeamCity servers — an authorization bypass that handed them arbitrary code execution with no authentication required |
| Discovery | T1033 / T1592.002 | Once inside, they ran basic Windows commands — whoami, nltest, tasklist, netstat — to map the host and the domain around it |
| Credential Access | T1003.001 / T1003.002 | Mimikatz, run entirely in memory, pulled credentials straight out of LSASS and the SAM database |
| Defense Evasion | T1027.001 / T1564.001 | This is the part that stood out to me. Their backdoor, GraphicalProton, exfiltrated stolen data by hiding it inside randomly generated BMP image files, then uploaded those images to OneDrive and Dropbox |
| Persistence | T1053.005 | Scheduled tasks, disguised with names like "WindowsDefenderService," kept the backdoor alive across reboots |
| Command and Control | T1572 | A modified open-source SOCKS tunneler, renamed rr.exe, tunneled traffic out to their infrastructure |
| Exfiltration | T1567 | The SYSTEM, SAM, and SECURITY registry hives — effectively the keys to the whole domain — were zipped up and pushed out through the same OneDrive/Dropbox channel |
