# HTB Write-ups
Write-ups for retired Hack The Box machines and Starting Point boxes, documenting
recon, exploitation, and privilege escalation for each.
More write-ups will be added as boxes are completed and retire.
## Boxes
| Box | OS | Difficulty | Technique |
|---|---|---|---|
| [Vaccine](./vaccine) | Linux | Very Easy | Anonymous FTP → offline crack → SQLi → hardcoded DB creds → sudo `vi` shell escape |
| [Three](./three) | Linux | Very Easy | Hidden vhost → misconfigured S3 bucket with write access → upload PHP web shell → RCE |
| [Cap](./cap) | Linux | Easy | IDOR → cleartext creds → Python capability abuse (cap_setuid) |
| [Appointment](./appointment) | Linux | Very Easy | SQL injection auth bypass |
| [Sequel](./sequel) | Linux | Very Easy | Unauthenticated MySQL misconfiguration |
| [Crocodile](./crocodile) | Linux | Very Easy | Anonymous FTP credential leak → directory brute-force |
| [Responder](./responder) | Windows | Very Easy | LFI/RFI → NTLMv2 capture (Responder) → offline crack → RDP/WinRM |

## CTF Challenges
| Challenge | Event | Category | Technique |
|---|---|---|---|
| [Gatery](./gatery) | HTB Cyber Apocalypse 2026 | Web | Cosmetic game UI over unsigned access-control cookie the backend never verifies |
| [Massagold](./massagold) | HTB Cyber Apocalypse 2026 | Web | Stored XSS → CSP bypass via JSONP gadget → admin bot session hijack |
| [Jailbreak](./jailbreak) | HTB Tryout 2026 | Web | XXE in firmware XML upload → arbitrary local file read |
| [TimeKORP](./timekorp) | HTB Tryout 2026 | Web | Unsanitized `format` param → OS command injection via `exec()` |