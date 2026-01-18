---
title: Troubleshooting
weight: 7
---

# Troubleshooting

Common issues and solutions for algorithm development.

## Sections

- [Common Errors](common-errors): Frequently encountered errors and fixes
- [Debugging Tips](debugging-tips): Strategies for debugging algorithms

## Quick Diagnostics

### Check Installation

```bash
cd rcabench-platform
./main.py self test
```

### Check Dataset Access

```bash
ls data/trainticket-pandora-v1/
# Should show numbered directories (0, 1, 2, ...)
```

### Check Algorithm Registration

```bash
./main.py list-algorithms
# Should show your algorithm in the list
```

## Getting Help

If you can't resolve an issue:

1. Check the [Common Errors](common-errors) page
2. Review [Debugging Tips](debugging-tips)
3. Search existing issues on GitHub
4. Open a new issue with:
   - Error message
   - Steps to reproduce
   - Environment details (OS, Python version)
   - Relevant code snippets
