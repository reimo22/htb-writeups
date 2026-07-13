# HTB Write-ups

Write-ups for retired Hack The Box machines and Starting Point boxes, documenting
recon, exploitation, and privilege escalation for each.

More write-ups will be added as boxes are completed and retire.

## Boxes

| Box | OS | Difficulty | Technique |
|---|---|---|---|
| [Cap](./cap) | Linux | Easy | IDOR → cleartext creds → Python capability abuse (cap_setuid) |
| [Appointment](./appointment) | Linux | Very Easy | SQL injection auth bypass |
| [Sequel](./sequel) | Linux | Very Easy | Unauthenticated MySQL misconfiguration |
| [Crocodile](./crocodile) | Linux | Very Easy | Anonymous FTP credential leak → directory brute-force |
| [Responder](./responder) | Windows | Very Easy | LFI/RFI → NTLMv2 capture (Responder) → offline crack → RDP/WinRM |