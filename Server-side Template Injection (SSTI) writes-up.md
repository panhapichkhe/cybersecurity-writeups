<img width="1133" height="276" alt="image" src="https://github.com/user-attachments/assets/01640007-ec62-4470-bd54-d03bd540610d" />

# Server-Side Template Injection (SSTI)

## Overview

This room introduced Server-Side Template Injection (SSTI) vulnerabilities and how template engines can become dangerous when user input is rendered insecurely.

The room covered:
- Identifying SSTI vulnerabilities
- Testing template engines with payloads
- Exploiting SSTI in different environments
- Basic mitigation techniques

---

## Environment Setup

Added the target machine to `/etc/hosts`:

```bash
10.49.132.98 ssti.thm
```

Then accessed the target through:

```text
http://ssti.thm
```
