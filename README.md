# CVE Request: MinaliC Webserver 2.0.0 — Remote Buffer Overflow via HTTP POST

## Summary

Stack-based buffer overflow in MinaliC Webserver 2.0.0 allows remote unauthenticated
attackers to execute arbitrary code via a crafted HTTP POST request. The overflow
occurs in the URI parsing routine when processing the POST method, which follows a
separate code path from GET. Existing CVEs for MinaliC (CVE-2012-0273 for
cookie/directory/filename handling, CVE-2024-58306 for oversized GET) do not cover
this vulnerability.

---

## MITRE Form Fields

**Request Type:** Report Vulnerability / Request CVE ID

**Vulnerability Type:** Buffer Overflow (CWE-121: Stack-based Buffer Overflow)

**Vendor:** Hans Alshoff (MinaliC project, minalic.sourceforge.net)

**Product:** MinaliC Webserver

**Version:** 2.0.0

**Attack Type:** Remote, unauthenticated

**Impact:**
- Code Execution: yes
- Denial of Service: yes
- Information Disclosure: no (unless post-exploitation)

**Affected Component:** HTTP POST request URI handler (port 8080, default)

---

## Vulnerability Details

MinaliC Webserver 2.0.0 copies the POST request URI into a fixed-size stack buffer
without proper bounds checking. By sending a POST request with a URI of approximately
240+ bytes (adjusted depending on the minalic installation path length), an attacker
can overwrite the saved return address (EIP) on the stack and redirect execution to
attacker-controlled shellcode.

The vulnerability is distinct from the known GET-method overflow because the POST and
GET handlers use separate code paths internally. The GET-method overflow has been
documented under CVE-2012-0273 and various Exploit-DB entries (EDB-27554, etc.),
but none of them target the POST method.

### Trigger condition

The URI length threshold depends on the MinaliC installation path. For a typical
install at `c:\minalic\bin` (14 characters), approximately 240 bytes of junk
in the POST URI are enough to reach the saved EIP. Longer install paths reduce the
required junk length accordingly.

### Exploitation

The exploit sends a single HTTP POST request:

```
POST /<junk + EIP overwrite> HTTP/1.1
Host: <padding>
User-Agent: <NOP sled + shellcode>
```

The URI carries the overflow payload including a small stub (`add ecx,4; jmp ecx`)
that pivots into the User-Agent header where the main shellcode sits. Tested with
bind shell payload (`windows/shell_bind_tcp`, port 1337) via msfvenom, encoded with
shikata_ga_nai to avoid bad characters (0x00, 0x0a, 0x0d).

### Tested targets

- Windows Server 2003 Standard (no service pack) — `jmp esp` at 0x77FB8BAB (ntdll.dll)
- Windows Server 2003 SP1 — `jmp esp` at 0x7C86FED3 (ntdll.dll)
- Windows Server 2003 SP2 — `jmp esp` at 0x7C86A01B (ntdll.dll)

All three targets confirmed remote code execution with full Administrator privileges.

---

## Timeline

- **April 2013:** Vulnerability discovered and exploit developed
- **2026:** Exploit code cleaned up, requesting CVE assignment

---

## Discoverer

Antonius — Blue Dragon Security (bluedragonsec.com)
https://github.com/bluedragonsecurity

---

## References

- Exploit source code and PoC screenshots: https://github.com/bluedragonsecurity
- MinaliC Webserver project: https://minalic.sourceforge.net/
- Related (GET-method, different code path): CVE-2012-0273, CVE-2024-58306

---

## Suggested CVE Description

MinaliC Webserver 2.0.0 contains a stack-based buffer overflow in the HTTP POST
request URI handler. A remote unauthenticated attacker can send a crafted POST
request with an oversized URI to overwrite the return address on the stack and
execute arbitrary code. This is a different vulnerability than CVE-2012-0273
(which affects cookie, directory, and filename handling) and CVE-2024-58306
(which affects GET requests).

---

## CVSS 4.0 Estimate

- Attack Vector: Network
- Attack Complexity: Low
- Privileges Required: None
- User Interaction: None
- Impact: High (code execution as the service user)

Estimated base score: ~9.3 (Critical)
