# Security policy

## Reporting a vulnerability

Please report security problems **privately**: on this repository's **Security** tab choose
**Report a vulnerability**. Do not open a public issue for them.

Include your Home Assistant version, what an attacker could do, and the steps to reproduce it. We aim to reply
within a few days and to publish a fix, with credit if you want it, once it is ready.

## Scope

This blueprint runs entirely inside your own Home Assistant. It makes no network calls; its only actions are
reloading the Bluetooth adapter, logbook entries, a Home Assistant notification and, if turned on, a restart of
Home Assistant. Reports about Home Assistant itself belong with the Home Assistant project.
