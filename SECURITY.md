# Security policy

This repository contains documentation only; it has no running code. Security findings about the BarBro service or its code belong to the `barbro` repository.

## Reporting a vulnerability

Please do not open a public issue for a security finding. Use GitHub's private vulnerability reporting ("Report a vulnerability" under the Security tab) on the `barbro` repository. Reports are read by the maintainer; you will get an acknowledgement and, once the finding is confirmed, a note when it is fixed.

## Scope

- In scope: the deployed BarBro service, the `barbro` code repository, and errors in this documentation that would lead to an insecure implementation.
- Out of scope: third-party services BarBro depends on (Google, DeepL, the LLM provider, Cloudflare, Hetzner, Sentry) — report those to the respective vendor.

## What is already known

The assets, trust boundaries, threats and planned controls are described in [`threat-model.md`](threat-model.md), including the risks accepted deliberately. A report that a listed accepted risk exists is not a finding; a report that a listed control is missing or does not work is.
