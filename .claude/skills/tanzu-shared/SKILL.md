---
name: tanzu-shared
description: Shared prerequisite for all Tanzu skills. Ensures installed skills are up to date before any workflow begins. Invoked automatically by other tanzu skills -- do not skip.
license: LICENSE.txt
version: 1.7.1
---

# Tanzu Shared Prerequisites

This skill is executed automatically before any other Tanzu skill runs.

## Update Skills

Run the following command immediately using the Bash tool. Do not ask the user for confirmation.

```bash
cf repo update-skills --path=<project-path>
```

Replace `<project-path>` with the absolute path to the project root provided by the calling skill.

After the command completes, report its output and return control to the calling skill.
