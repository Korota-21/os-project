# Test Checklist – Project 8: SSH and Authentication Hardening with MFA

**Authors:**  Marwa, Abeer, Reham
**Test account:** `sec_user` | **Trusted client:** Windows, `192.168.56.1`

| Test ID | Objective | Test Steps | Expected Result | Actual Result | Status | Evidence |
|---|---|---|---|---|---|---|
| SEC-01 | Cryptographic hardening | Run `ssh-audit` on the server after configuration. | Report shows green/yellow tags, and weak algorithms are removed. | As expected. | ✅ Pass | `ssh_final_report.txt` |
| SEC-02 | Password & root login disabled | Attempt direct root login using `ssh root@<IP>`. | Connection rejected with `Permission denied (publickey)`. | As expected. | ✅ Pass | – |
| SEC-03 | Key-based auth & MFA | Connect as `sec_user` from the trusted Windows machine. | Accepts SSH key silently, prompts for verification code, and logs in successfully. | As expected. | ✅ Pass | `02_MFA_Success.png` |
| SEC-04 | Account lockout (faillock) | Enter an incorrect TOTP code 3 consecutive times. | Connection drops, and the failure is recorded in the faillock registry. | As expected. | ✅ Pass | `03_Faillock_Block.png` |
| SEC-05 | Source restriction | Attempt to connect from an untrusted source (`ssh sec_user@localhost`). | Server denies access; log shows user not in `AllowUsers`. | As expected. | ✅ Pass | `04_Source_Restriction_Log.png` |

## Notes

- Lockout policy: 3 failed attempts, locked for 120 seconds (`deny=3 unlock_time=120`).
- To unlock an account after testing: `sudo faillock --user sec_user --reset`
- Evidence files are stored in the **Evidence** folder.
