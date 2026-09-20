---
expect:
  email: string
---

{"email":"{{input.email}}","canonical":"{{input.email}}","mailbox_type":"Standard","is_disposable":false,"is_role":false,"is_free_provider":false,"findings":[],"score":{"value":10,"band":"clean","conclusive":true,"reasons":[{"code":"email.dmarc_enforced","label":"DMARC is enforced","weight":-10}]},"domain_scan":{"scanned":false,"mail_source":"live","mx":{"count":1},"auth":{"spf":{"present":true},"dkim":{"present":true},"dmarc":{"present":true}},"spoofability":{"verdict":"Spoofing quarantined","summary":"DMARC policy is enforced."}}}
