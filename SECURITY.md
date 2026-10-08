# Security Policy

## Supported Versions

Security fixes are applied to the latest release only. Older releases are not patched.

## Reporting a Vulnerability

Please report security vulnerabilities privately through GitHub's
[private vulnerability reporting](https://github.com/briar-systems/masc/security/advisories/new)
rather than opening a public issue.

masc reads untrusted input by design: assembly text and encoded code from any
source. A malformed input that makes masc crash, hang, or read or write out of
bounds, or that it encodes as something other than what was written, is a
vulnerability and is welcome as a report.

Expect an acknowledgement within a few days. If the report is confirmed we will
prepare a fix and coordinate disclosure with you. If it is declined we will
explain why.
