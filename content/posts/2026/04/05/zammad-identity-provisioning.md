---
title: "Zammad Identity Provisioning: An Audit Analysis"
description: "An audit analysis of Zammad's identity handling and provisioning
behaviors, with a focus on security implications and recommendations for
secure deployment."
author: "Tobias Schäfer"
date: 2026-04-05T14:00:00+01:00
draft: false
toc: false
tags:
  - zammad
  - security
  - oidc
  - saml
  - ldap
  - iam
  - public-sector
---

[Zammad](https://github.com/zammad/zammd) is widely used for ticketing and
case management, including in public services. In those environments,
**identity correctness is a security requirement**: wrong-account linking,
unexpected re-activation, or silent identity drift are not "ops problems",
they are audit and incident triggers.

This post documents identity and provisioning behaviors that can lead to
hard-to-defend security outcomes when Zammad is run with mixed identity
sources. Every claim is backed by source code links, verified against commit
[`77ffb905`](https://github.com/zammad/zammad/commit/77ffb90577fb2bb92fd649351bbdfc17639e43e8)
(main, 2026-04-03).

## Threat model

Most non-trivial deployments combine at least two of:

- local accounts (bootstrap, break-glass),
- LDAP/AD import (legacy provisioning),
- OIDC/SAML (SSO, MFA, conditional access).

The predictable failure modes of mixed-source operation are: duplicate
accounts for the same person, wrong-account linking, unexpected reactivation
of deprovisioned accounts, and inconsistent attributes across sources. All of
them have security impact: access control errors, orphaned access, unreliable
audit trails.

## Finding 1: SSO login can be silently linked to the wrong existing account

When `auth_third_party_auto_link_at_inital_login` (sic: upstream spelling) is
enabled and a first-time SSO login cannot find an existing account by
`login == uid`, both OIDC and SAML fall back to matching by email address with
no proof of ownership.

Source: [`app/models/authorization/provider/openid_connect.rb`](https://github.com/zammad/zammad/blob/77ffb90577fb2bb92fd649351bbdfc17639e43e8/app/models/authorization/provider/openid_connect.rb#L6-L14),
[`app/models/authorization/provider/saml.rb`](https://github.com/zammad/zammad/blob/77ffb90577fb2bb92fd649351bbdfc17639e43e8/app/models/authorization/provider/saml.rb#L6-L14)

The practical consequence: an LDAP-imported account for `alice@example.com`
with `login = alice` will not be found by `login == uid` when Alice first logs
in via OIDC (her IdP `uid` is `oidc-sub-9999`). The email fallback then links
her SSO session to the LDAP account. In a clean dataset this produces the
correct result. In a dataset with recycled addresses, shared mailboxes, or
a manipulated IdP claim, it is an account takeover vector.

The email fallback only activates when `user_email_multiple_use` is
**disabled** (the default). Enabling it suppresses the fallback but causes a
duplicate account instead. Neither option is safe in a mixed-source
deployment. The toggle trades one risk for another:

| `user_email_multiple_use` | `login` lookup misses | outcome |
|---|---|---|
| off (default) | email fallback fires | wrong-account linking |
| on | no fallback | duplicate account |

This toggle is documented as a multi-tenancy feature. Operators enabling it
for business reasons are unlikely to realize they are simultaneously trading
one risk profile for another.

## Finding 2: The stored user `login` may silently not match the IdP `uid`

Even when OIDC correctly sets `login = uid` before account creation, a
model-level validation callback - `check_login` - fires on every save and
can silently replace that value. It falls back to the email address if `login`
is blank, and appends a uniqueness suffix (`alice@example.com1`, `…2`, etc.)
if that email is already taken as a login by another account, for example,
one that was imported via LDAP.

Source: [`app/models/user.rb`](https://github.com/zammad/zammad/blob/77ffb90577fb2bb92fd649351bbdfc17639e43e8/app/models/user.rb#L838-L873)

The result: on the user's next SSO login, `find_by(login: uid)` misses, the
email fallback fires, and Finding 1 applies. The two issues compound: the
login mutation at account creation silently creates the precondition for
wrong-account linking at re-login.

## Finding 3: LDAP sync unconditionally reactivates disabled accounts

The LDAP import pipeline sets `active: true` on every synced user,
regardless of whether the account was manually disabled in Zammad. The
comment in the source is explicit about the intent:

Source: [`lib/sequencer/unit/import/ldap/user/attributes/static.rb`](https://github.com/zammad/zammad/blob/77ffb90577fb2bb92fd649351bbdfc17639e43e8/lib/sequencer/unit/import/ldap/user/attributes/static.rb#L14-L17)

```ruby
# we have to add the active state manually
# because otherwise disabled instances won't get
# re-activated if they should get synced again
active: true,
```

An account disabled in Zammad is re-enabled by the next LDAP sync run.
If deprovisioning means disabling accounts in Zammad rather than removing
them from the LDAP source, it does not hold across sync runs.

This is also the primary reason Finding 1 exists at scale: every LDAP-imported
account enters Zammad with a `login` derived from an AD attribute
(`sAMAccountName`), not from any IdP `uid`. These accounts are exactly what
causes the `login == uid` lookup to miss when SSO is introduced later.

## Finding 4: No claim mapping for OIDC or SAML - IdP cannot own user attributes

LDAP import has an explicit, configurable attribute mapping: the admin defines
which LDAP attributes map to which Zammad user fields (`firstname`, `lastname`,
`department`, custom object attributes, etc.).

OIDC and SAML have none. When Zammad creates a user from an SSO response, it
reads a fixed set of standard fields and this is an implementation dependency,
not a configuration option:

Source: [`app/models/user.rb`](https://github.com/zammad/zammad/blob/77ffb90577fb2bb92fd649351bbdfc17639e43e8/app/models/user.rb#L321-L337)

```ruby
data = {
  login:         hash['login'],
  firstname:     hash['info']['name'] || hash['info']['display_name'],
  email:         hash['info']['email'],
  image_source:  hash['info']['image'],
  web:           url,
  address:       hash['info']['location'],
  note:          hash['info']['description'],
  source:        hash['provider'],
  ...
}
if hash['info']['first_name'].present? && hash['info']['last_name'].present?
  data[:firstname] = hash['info']['first_name']
  data[:lastname]  = hash['info']['last_name']
end
```

There is no way to direct custom IdP claims - `department`, `cost_center`,
`employee_id`, `groups`, or any organization-specific attribute - into Zammad
user fields, regardless of what the IdP token contains. This is not a
configuration gap; it is a missing feature.

The practical consequence is that using the IdP as a single source of truth
for user attributes is structurally impossible. Attributes that the IdP
maintains authoritatively either go unset in Zammad or must be kept in sync
via a separate LDAP import job, reintroducing all the problems in Findings
1–3. The "use the IdP, avoid LDAP" advice breaks down here: without claim
mapping, the IdP can handle authentication but cannot fully replace LDAP as
the provisioning authority.

## What the settings actually control

Three settings together determine the outcome on every first-time SSO login:

- **`auth_third_party_auto_link_at_inital_login`** - master gate for all
  account-linking. If off, no existing account is ever linked; a new account
  is created (or blocked by the next setting). All wrong-account linking risk
  requires this to be on.
- **`auth_third_party_no_create_user`** - if on and no match is found, the
  login is rejected. Combined with `auto_link` off this is the most
  restrictive posture: only pre-provisioned accounts are accepted. It also
  removes just-in-time (JIT) provisioning entirely.
- **`user_email_multiple_use`** - controls whether the email fallback fires
  when the `login == uid` lookup misses (see Finding 1 table above).

Source: [`app/models/authorization/provider.rb`](https://github.com/zammad/zammad/blob/77ffb90577fb2bb92fd649351bbdfc17639e43e8/app/models/authorization/provider.rb#L22-L40)

There is no configuration that is simultaneously safe, JIT-capable, and
tolerant of a mixed-source account dataset. Safe operation requires
addressing the mixed-source condition and - until claim mapping exists -
accepting that LDAP cannot be fully replaced by the IdP.


## The root cause: mutable fields as identity anchors

Zammad uses mutable, heuristic-managed display fields as identity anchors for
SSO and deduplication. There is no immutable external identifier. Any operation
that modifies login or email, including silent model callbacks triggered by
unrelated saves, invalidates the identity contract that SSO, and LDAP sync
depend on.

## Recommendations

**For migrations from LDAP to SSO:** before enabling
`auth_third_party_auto_link_at_inital_login`, align `login` to the IdP `uid`
(for OIDC: the `sub` claim) on all accounts for users who will authenticate
via SSO. Do not align to email, it is the fallback key. After migration,
stop LDAP import for the migrated population; continuing to run it will
re-derive `login` from AD attributes on each sync, silently undoing the
alignment.

**For attribute provisioning:** until OIDC/SAML claim mapping is implemented,
there is no clean path to making the IdP the sole authority for user
attributes. If department, organizational unit, or other directory attributes
matter in Zammad (for routing, permissions, or reporting), LDAP sync remains
necessary, which means its interaction with SSO must be explicitly managed
as described above.

**For regulated environments:** disable auto-link and auto-creation
(`auth_third_party_auto_link_at_inital_login` off,
`auth_third_party_no_create_user` on) and pre-provision accounts via API or
Zammad's web user-interafce with `login` set to the IdP `uid`. This eliminates
wrong-account linking at the cost of JIT provisioning.

## What needs to change upstream

Two features are needed before Zammad can be operated as a proper IdP
relying party:

1. **A configurable, protected identity anchor.** Admins should be able to
   pin which IdP claim maps to the Zammad `login` (or a dedicated
   external-identifier field), and that value should not be overwritable by
   model-level fallback logic.

2. **Configurable claim mapping for OIDC and SAML.** The same mapping
   capability that LDAP has, directing directory attributes into Zammad user
   fields, needs to exist for SSO providers. Without it, the IdP cannot
   replace LDAP as the provisioning authority, and mixed-source operation
   with its associated risks is effectively unavoidable.

Until both exist, the identity resolution outcome in any Zammad deployment
is determined by the intersection of three settings, the historical shape of
the account dataset, and model-level validation side effects with no
operator-controlled guarantee that an SSO login binds to the correct account.

## Code receipts

- [OIDC `find_user` - `login` lookup with email fallback](https://github.com/zammad/zammad/blob/77ffb90577fb2bb92fd649351bbdfc17639e43e8/app/models/authorization/provider/openid_connect.rb#L6-L14)
- [SAML `find_user` - identical logic](https://github.com/zammad/zammad/blob/77ffb90577fb2bb92fd649351bbdfc17639e43e8/app/models/authorization/provider/saml.rb#L6-L14)
- [Base provider `find_user` - pure email, no `login` step](https://github.com/zammad/zammad/blob/77ffb90577fb2bb92fd649351bbdfc17639e43e8/app/models/authorization/provider.rb#L42-L46)
- [`fetch_user` control flow - all three settings](https://github.com/zammad/zammad/blob/77ffb90577fb2bb92fd649351bbdfc17639e43e8/app/models/authorization/provider.rb#L22-L40)
- [`check_login` - login mutation on every save](https://github.com/zammad/zammad/blob/77ffb90577fb2bb92fd649351bbdfc17639e43e8/app/models/user.rb#L838-L873)
- [`create_from_hash!` - fixed OmniAuth field mapping, no custom claims](https://github.com/zammad/zammad/blob/77ffb90577fb2bb92fd649351bbdfc17639e43e8/app/models/user.rb#L312-L343)
- [LDAP static attributes - `active: true` and `source` rewrite on every sync](https://github.com/zammad/zammad/blob/77ffb90577fb2bb92fd649351bbdfc17639e43e8/lib/sequencer/unit/import/ldap/user/attributes/static.rb#L11-L24)
- [LDAP configurable attribute mapping - what OIDC/SAML lacks](https://github.com/zammad/zammad/blob/77ffb90577fb2bb92fd649351bbdfc17639e43e8/lib/sequencer/unit/import/ldap/user/mapping.rb#L1-L11)
- [`no_create_user` guard](https://github.com/zammad/zammad/blob/77ffb90577fb2bb92fd649351bbdfc17639e43e8/app/models/authorization/provider.rb#L28-L34)
