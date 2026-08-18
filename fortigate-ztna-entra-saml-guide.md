# FortiGate ZTNA Access Proxy with Entra ID SAML — FortiOS 7.2.x Field Guide

A working, tested guide to publishing an internal web application through the FortiGate ZTNA access proxy with Microsoft Entra ID SAML authentication and group-based authorization, on **FortiOS 7.2.13**.

Written after deploying three independent proxies on one FortiGate and hitting essentially every failure mode available. Everything here is transcribed from configuration verified working on real hardware, not from vendor examples.

**Why this exists:** the official documentation and most blog posts target FortiOS 7.4/7.6, where several things happen automatically that do not happen on 7.2. Following a 7.6 guide on 7.2 produces a proxy that silently fails authentication.

---

## Applicability

|                            |                                                                           |
| -------------------------- | ------------------------------------------------------------------------- |
| **Verified on**            | FortiOS 7.2.13                                                            |
| **Likely applies to**      | FortiOS 7.2.x generally                                                   |
| **May NOT fully apply to** | 7.4, 7.6+                                                                 |
| **Scope**                  | Agentless (browser) HTTPS access. No FortiClient/EMS required             |
| **IdP**                    | Microsoft Entra ID. Concepts transfer to other SAML IdPs; specifics don't |

---

## ⚠ The GUI Does Not Complete This Configuration

On 7.2.13, creating a ZTNA server through the GUI produces an **incomplete** access proxy. Verified: a GUI-created proxy was missing all of the following:

- `client-cert` setting
- `auth-portal enable`
- `auth-virtual-host`
- **The entire samlsp api-gateway** — no SAML SP endpoint exists at all
- Gateway 1 `url-map`, `url-map-type`, `service`
- Real server `port`

The GUI also has two destructive behaviors:

- It generates a **new numbered `auto-*` virtual-host object on each edit** rather than reusing the existing one, producing duplicates that claim the same hostname.
- Editing an existing proxy has been observed to **rewrite gateway 1's `url-map` to `"*"`/`wildcard`** and clear `service` and the realserver port, breaking SAML on a previously working proxy.

Use the GUI for what it does well, then complete in the CLI. Steps marked **🔧 CLI REQUIRED** are not optional. After any GUI edit, re-verify with `show firewall access-proxy <name>`.

---

## Version Differences

On **FortiOS 7.6.1+**, when SAML authentication is used in a ZTNA access-proxy configuration, FortiOS implicitly creates a virtual-host from the domain in the SAML server config and implicitly registers the `/remote/saml/login` path for api-gateway matching.

**On 7.2.x none of this is automatic.** This is the single biggest reason 7.6-era guides don't work here.

---

## The Five Non-Obvious Things

These cost the most time and are not in the documentation.

### 1. The application gateway needs `virtual-host`; the samlsp gateway cannot have one

Once a virtual-host object owns a hostname, the *application* gateway must bind to it or every request returns "API Gateway Denied" — including `/favicon.ico`, which is a useful tell that it's host-scoped rather than path-specific.

The *samlsp* gateway has exactly four fields and rejects `virtual-host`, `url-map`, and `url-map-type`:

```
command parse error before 'virtual-host'
Command fail. Return code -61
```

This asymmetry is undocumented and easy to get backwards.

### 2. The samlsp gateway does NOT select which SAML server is used

Counter-intuitively, `set saml-server` on the samlsp gateway does not determine which IdP the AuthnRequest is sent to. **The authentication rule does.**

With multiple proxies and multiple auth rules, the first matching rule wins regardless of which proxy the traffic actually hit. You can have a perfectly configured proxy that authenticates against a completely different application's IdP.

### 3. Authentication rule `dstaddr` matches the REAL SERVER, not the VIP

From `diagnose wad debug`:

```
wad_http_req_check_policy ... (203.0.113.10:21761@39 -> 10.0.1.20:443@94)
```

The destination in policy and auth-rule evaluation is the **internal real server**, not the public VIP address. Scoping an authentication rule by the VIP's public IP silently never matches.

Corollary: **do not** narrow the *proxy policy's* `dstaddr` either. It doesn't evaluate against the VIP, and narrowing it breaks the policy entirely.

### 4. Two access proxies cannot share a real server

Given #2 and #3: if two proxies front the same backend, source and destination are identical and no auth rule can distinguish their traffic. The first-listed rule captures everything.

If both applications genuinely live on one host, use a second IP on that host, a second listening port, or one access proxy serving both FQDNs via multiple virtual hosts.

### 5. Group membership resolves once, at authentication time

A stale session reports the old `g_id` no matter what you fix in the config or the IdP. Run `diagnose wad user clear` and use a fresh incognito window between **every** test. Several hours were lost to a correct fix that appeared not to work.

---

## Architecture

```
Browser
  └─► https://<fqdn>[:<port>]
        └─► firewall vip (type access-proxy)          [GUI]
              └─► firewall access-proxy                [GUI + CLI]
                    ├─ auth-virtual-host ──► vhost      [CLI to bind]
                    ├─ api-gateway 1  → realserver      [CLI to complete]
                    └─ api-gateway 3  → service samlsp  [CLI only]
              └─► authentication rule                   [selects the SAML server]
                    └─ dstaddr = REAL SERVER address
              └─► firewall proxy-policy                 [authorization]
                    └─ groups → user group → IdP group GUID
```

---

## Prerequisites

- FortiGate admin access — **GUI and CLI both required**
- Entra ID Application Administrator in the target tenant
- Public DNS A record for the proxy FQDN → VIP external IP
- Public certificate with a SAN covering the FQDN, imported to the FortiGate
- FortiGate → real server reachability on the application port
- Entra security group created, target users assigned, object GUID recorded
- External IP:port confirmed free (check admin-sport, SSL-VPN, existing VIPs)
- **A real server IP distinct from every existing access proxy**

Throughout, substitute:

| Placeholder       | Meaning                                  |
| ----------------- | ---------------------------------------- |
| `app.example.com` | Public FQDN of the published application |
| `203.0.113.10`    | VIP external (public) IP                 |
| `10.0.1.20`       | Real server internal IP                  |
| `<tenant-id>`     | Entra tenant GUID                        |
| `<group-guid>`    | Entra security group object GUID         |
| `MyApp`           | Your application name                    |

---

# Part 1 — Deployment

## Phase A — Entra ID

Do Entra first; it produces the certificate and URL values the FortiGate needs.

> **Every setting below is per-application.** Nothing carries over from another app in the same tenant. This caught two separate deployments.

### A1. Create the enterprise application

**Enterprise applications** → **New application** → **Create your own application** → *Integrate any other application you don't find in the gallery (Non-gallery)*.

### A2. Basic SAML configuration

**Single sign-on** → **SAML** → **Basic SAML Configuration** → Edit:

| Entra field            | Value                                           |
| ---------------------- | ----------------------------------------------- |
| Identifier (Entity ID) | `https://app.example.com/remote/saml/metadata/` |
| Reply URL (ACS)        | `https://app.example.com/remote/saml/login`     |
| Sign on URL            | `https://app.example.com/`                      |
| Logout URL             | `https://app.example.com/remote/saml/logout`    |

If the VIP is not on 443, include the port in all four: `https://app.example.com:10443/...`

### A3. Signing option — mandatory

**SAML Signing Certificate** → Edit:

- **Signing Option: Sign SAML response and assertion**
- Signing Algorithm: **SHA-256**

Entra's default signs the assertion only. FortiOS looks for a signature on the Response element and fails with:

```
SAML low level error: Signature element not found
```

This is per-app. It will bite you again on the next application.

### A4. Group claim

**Attributes & Claims** → **Add a group claim**:

- Source: **Groups assigned to the application**
- Source attribute: **Group ID** (emits the object GUID)

"Groups assigned to the application" is preferred over "All groups" or "Security groups" because of the **group overage limit** — if a user belongs to more than ~150 groups, Entra drops the claim entirely and substitutes a Graph API link that FortiOS cannot follow. The symptom is clean authentication with zero groups.

### A5. Token encryption

Confirm **disabled**. FortiOS 7.2 cannot process `<EncryptedAssertion>` and fails with the same signature error as A3.

### A6. Assign users and groups

Assign the authorization group under **Users and groups**. With "Groups assigned to the application" as the claim source, an unassigned group emits nothing.

**Guest accounts:** if you're testing as a guest in another tenant, verify your membership in that specific group. Guest group membership is easy to assume and easy to get wrong.

Record the group's **Object ID** — that GUID goes into the FortiGate.

### A7. Record IdP values, download the certificate

From **Set up \<app name\>**, record the Login URL, Microsoft Entra Identifier, and Logout URL. Download the **Certificate (Base64)**.

---

## Phase B — FortiGate GUI

### B1. Import the Entra signing certificate

**System → Certificates → Create/Import → Remote Certificate**

Confirm the assigned name — FortiOS names these `REMOTE_Cert_N`:

```
get vpn certificate remote
```

### B2. Verify the server certificate SAN

The certificate you serve must have a SAN covering the new FQDN. A wildcard covers it; an explicit-SAN certificate for a different host does not, and produces a browser name-mismatch with no CLI error.

### B3. Create the VIP

**Policy & Objects → Virtual IPs → Create New → Virtual IP**

| Field         | Value                   |
| ------------- | ----------------------- |
| Type          | Access Proxy            |
| External IP   | `203.0.113.10`          |
| Interface     | WAN                     |
| Server type   | HTTPS                   |
| External port | 443 (or an unused port) |
| Certificate   | your public certificate |

> **🔧 CLI REQUIRED — TLS floor**
> The GUI does not expose `ssl-min-version`, and the platform default is `tls-1.1`.
> 
> ```
> config firewall vip
>     edit "ZTNA-MyApp"
>         set ssl-min-version tls-1.2
>     next
> end
> ```

### B4. Create the virtual host

Host = `app.example.com` (must match the SAML URLs exactly), plus the certificate.

> **⚠ Check for duplicates before and after**
> 
> ```
> show firewall access-proxy-virtual-host
> ```
> 
> Two entries with the same `host` value means FortiOS cannot resolve which owns the hostname. Bind to the one carrying the certificate and delete the other.

### B5. Create the ZTNA server (partial)

**Policy & Objects → ZTNA → ZTNA Servers → Create New** — set the VIP, the virtual host, and a real server with internal IP and port.

**This produces an incomplete object.** Phase C completes it. Don't test yet.

### B6. Create the SAML server object

**User & Authentication → SAML SSO → Create New**

| Field                             | Value                                                                |
| --------------------------------- | -------------------------------------------------------------------- |
| IdP entity ID / SSO URL / SLO URL | from A7                                                              |
| IdP certificate                   | the `REMOTE_Cert_N` from B1                                          |
| SP entity ID                      | **paste from Entra Identifier verbatim**                             |
| SP single sign-on URL             | `https://app.example.com/remote/saml/login`                          |
| SP single logout URL              | `https://app.example.com/remote/saml/logout`                         |
| Username attribute                | `http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress` |
| Group name attribute              | `http://schemas.microsoft.com/ws/2008/06/identity/claims/groups`     |

**Claim URI warning.** The username attribute must reference a claim Entra actually emits. Standard Entra claims include `.../claims/emailaddress` and `.../claims/name`. **`.../claims/username` is not an Entra default** and produces:

```
user.saml.<server>.user-name='...claims/username' was NOT FOUND in the SAML Assertion Response
```

**Entity ID scheme.** Whether it's `http://` or `https://`, with or without a trailing slash, doesn't matter — but it must match the Entra Identifier field **character-for-character**. Copy from Entra; do not normalize.

### B7. Create the user group with GUID match

**User & Authentication → User Groups → Create New** — Type Firewall, Remote Groups → the SAML server from B6, Groups = the Entra group **object GUID** (not the display name).

The SAML server must exist first, or the group creation fails with `entry not found in datasource`, which then cascades into the match block failing silently.

### B8. Authentication scheme and rule

**Policy & Objects → Authentication Rules** — Scheme: method SAML, server from B6. Rule: source interface WAN, source all, active auth method = the scheme.

> **🔧 CLI REQUIRED — mandatory flags and destination scoping**
> 
> ```
> config firewall address
>     edit "myapp-realserver"
>         set subnet 10.0.1.20/32
>     next
> end
> 
> config authentication rule
>     edit "MyApp SAML"
>         set ip-based disable
>         set web-auth-cookie enable
>         set dstaddr "myapp-realserver"
>     next
> end
> ```
> 
> `ip-based disable` and `web-auth-cookie enable` are required for the SAML flow. `dstaddr` must be the **real server** — see [non-obvious thing #3](#3-authentication-rule-dstaddr-matches-the-real-server-not-the-vip). Required whenever more than one access proxy exists.

### B9. Proxy policy

**Policy & Objects → Proxy Policy → Create New**

| Field                   | Value                   |
| ----------------------- | ----------------------- |
| Type                    | Explicit/Access Proxy   |
| Access proxy            | the ZTNA server from B5 |
| Incoming interface      | any                     |
| Source / Destination    | all / all               |
| Schedule / Action       | always / ACCEPT         |
| **Source → User Group** | the group from B7       |
| Log                     | All Sessions            |

> **A policy with ACCEPT and no user group passes traffic with NO authentication whatsoever.** Confirm the group is set before the VIP is publicly reachable. This is a real and easy mistake — it presents as "the app loads fine," which looks like success.

**Leave Destination as `all`.** Narrowing it to the VIP address breaks the policy.

### B10. DNS

Public A record: `app.example.com` → `203.0.113.10`.

---

## Phase C — 🔧 CLI Required

None of this is reachable from the GUI on 7.2.13.

```
config firewall access-proxy
    edit "ZTNA-MyApp"
        set client-cert disable
        set auth-portal enable
        set auth-virtual-host "vh-myapp"
        config api-gateway
            edit 1
                set url-map "/"
                set url-map-type sub-string
                set service https
                set virtual-host "vh-myapp"
                config realservers
                    edit 1
                        set ip 10.0.1.20
                        set port 443
                    next
                end
            next
            edit 3
                set service samlsp
                set saml-server "MyApp Entra SSO"
            next
        end
    next
end
```

**Gateway 1** requires `virtual-host`. Use `url-map "/"` with `sub-string`. **Do not use `"*"` with `wildcard`** — a wildcard on the lower gateway ID shadows the SAML SSO path and produces `SAMLSP gateway ... has wrong configuration`.

**Gateway 3** is the SAML SP. Complete field set:

```
id                  : 3
service             : samlsp
saml-server         : <name>
saml-redirect       : disable
```

Verify before testing:

```
show firewall access-proxy ZTNA-MyApp
```

---

## Phase D — Verification

Fresh incognito window, external network, after `diagnose wad user clear`.

### D1. Confirm the correct SAML server fires — do this first

With multiple proxies, the wrong IdP firing is the most common failure and is completely invisible from the browser.

```
diagnose debug reset
diagnose debug application samld -1
diagnose debug console timestamp enable
diagnose debug enable
```

In the AuthnRequest, confirm:

```
AssertionConsumerServiceURL="https://app.example.com/remote/saml/login"
<saml:Issuer>https://app.example.com/remote/saml/metadata/</saml:Issuer>
```

Wrong FQDN → the authentication rule isn't scoping correctly, and nothing downstream will work. `diagnose debug disable` when done.

This same debug prints the full decoded assertion — the fastest way to see exactly which claims Entra sent.

### D2. Confirm identity and policy binding

```
diagnose wad user list
```

```
ID: 1, VDOM: root, IPv4: 198.51.100.5
  user name   : user@example.com
  auth_method : SAML
  pol_id      : 1
  g_id        : 49
```

- Username populated
- **`g_id` non-zero** — zero means group resolution failed even if the page loads
- **`pol_id` non-zero and correct** — note that policy IDs do **not** renumber when you `move` policies

### D3. Functional checks

- `https://app.example.com/` redirects to `login.microsoftonline.com`
- Application loads after authentication
- Forward traffic log carries the username
- A user outside the group is denied
- **Every other proxy on the box still works**

---

# Part 2 — Multiple Access Proxies

## Hard requirement: distinct real servers

See [non-obvious things #2, #3, #4](#the-five-non-obvious-things). Two proxies sharing a backend cannot be distinguished by any authentication rule.

## What is and isn't shared

| Object                | Shared?                          |
| --------------------- | -------------------------------- |
| VIP                   | No — distinct extip:extport      |
| Virtual host          | No                               |
| Real server           | **No — hard requirement**        |
| Entra enterprise app  | No                               |
| SAML server           | No                               |
| User group            | No                               |
| Authentication scheme | No                               |
| Authentication rule   | No — and must be dstaddr-scoped  |
| Proxy policy          | No                               |
| Entra security group  | Yes, may be reused               |
| Public certificate    | Yes, if the SAN covers all FQDNs |

## Scoping order

Scope the **new** rule first and test. If it fires correctly on its own, leave existing rules alone — an unscoped rule that works is worth more than symmetry. Modifying a working proxy to accommodate a new one is how a live application went down twice during the original deployment.

---

# Part 3 — CLI Reference

Complete configuration in dependency order. Use to build from scratch or to audit a GUI-built proxy.

> **Substitute every placeholder before pasting.** FortiOS rejects `<` and `>` as XSS characters, and a mid-block failure **silently discards subsequent commands** — this orphaned a SAML server, a user group, and an api-gateway in one paste, producing three unrelated-looking failures.

```
# 1. VIP
config firewall vip
    edit "ZTNA-MyApp"
        set type access-proxy
        set extip 203.0.113.10
        set extintf "WAN"
        set server-type https
        set extport 443
        set ssl-certificate "wildcard-example-com"
        set ssl-min-version tls-1.2
    next
end

# 2. Virtual host
config firewall access-proxy-virtual-host
    edit "vh-myapp"
        set host "app.example.com"
        set ssl-certificate "wildcard-example-com"
    next
end

# 3. SAML server
config user saml
    edit "MyApp Entra SSO"
        set entity-id "https://app.example.com/remote/saml/metadata/"
        set single-sign-on-url "https://app.example.com/remote/saml/login"
        set single-logout-url "https://app.example.com/remote/saml/logout"
        set idp-entity-id "https://sts.windows.net/TENANT-ID-HERE/"
        set idp-single-sign-on-url "https://login.microsoftonline.com/TENANT-ID-HERE/saml2"
        set idp-single-logout-url "https://login.microsoftonline.com/TENANT-ID-HERE/saml2"
        set idp-cert "REMOTE_Cert_1"
        set user-name "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress"
        set group-name "http://schemas.microsoft.com/ws/2008/06/identity/claims/groups"
    next
end

# 4. User group
config user group
    edit "MyApp-Entra-SSO"
        set member "MyApp Entra SSO"
        config match
            edit 1
                set server-name "MyApp Entra SSO"
                set group-name "GROUP-OBJECT-GUID-HERE"
            next
        end
    next
end

# 5. Access proxy
config firewall access-proxy
    edit "ZTNA-MyApp"
        set vip "ZTNA-MyApp"
        set client-cert disable
        set auth-portal enable
        set auth-virtual-host "vh-myapp"
        config api-gateway
            edit 1
                set url-map "/"
                set url-map-type sub-string
                set service https
                set virtual-host "vh-myapp"
                config realservers
                    edit 1
                        set ip 10.0.1.20
                        set port 443
                    next
                end
            next
            edit 3
                set service samlsp
                set saml-server "MyApp Entra SSO"
            next
        end
    next
end

# 6. Authentication scheme and rule
config authentication scheme
    edit "MyApp SAML Scheme"
        set method saml
        set saml-server "MyApp Entra SSO"
    next
end

config firewall address
    edit "myapp-realserver"
        set subnet 10.0.1.20/32
    next
end

config authentication rule
    edit "MyApp SAML"
        set srcintf "WAN"
        set srcaddr "all"
        set dstaddr "myapp-realserver"
        set ip-based disable
        set web-auth-cookie enable
        set active-auth-method "MyApp SAML Scheme"
    next
end

# 7. Proxy policy
config firewall proxy-policy
    edit 1
        set name "ZTNA-MyApp"
        set proxy access-proxy
        set access-proxy "ZTNA-MyApp"
        set srcintf "any"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set logtraffic all
        set groups "MyApp-Entra-SSO"
    next
end
```

## Field-level notes

- **`digest-method`** — left at the `sha1` default and working, despite Entra signing SHA-256. Don't change it while troubleshooting something else.
- **`clock-tolerance`** — default 15 seconds. Visible only via `show full-configuration`.
- **`saml-redirect`** — default `disable` on the samlsp gateway. Untested; the only remaining knob on that gateway.
- **`config authentication setting`** — empty on the verified deployment. No captive-portal value required on 7.2, contrary to the 7.0 vendor example.
- **Proxy policy `srcintf`** — both `any` and `WAN` verified working.

---

## Troubleshooting

| Symptom                                                                                               | Cause                                                                                                                                                 | Fix                                                                                                                                                                   |
| ----------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ERR_EMPTY_RESPONSE`                                                                                  | Wrong port, VIP conflict, or nothing bound to extip:extport                                                                                           | Verify VIP `extport`; check admin-sport and other VIPs                                                                                                                |
| Same, with `client-cert enable`                                                                       | `empty-cert-action block`, no client cert presented                                                                                                   | `set empty-cert-action accept` or `client-cert disable`. Lives at access-proxy level, **not** in api-gateway; hidden when client-cert is disabled                     |
| `API Gateway Denied`                                                                                  | Application gateway not bound to the matched virtual host                                                                                             | `set virtual-host` on gateway 1                                                                                                                                       |
| Same, wrong proxy named in debug                                                                      | Two VIPs on the same extip:extport                                                                                                                    | Check for duplicate extip/extport across VIPs                                                                                                                         |
| Page loads with **no** IdP prompt                                                                     | Proxy policy has no user group                                                                                                                        | Add the group — open-access condition, remediate immediately                                                                                                          |
| `SAML low level error: Signature element not found`                                                   | Entra signing assertion only                                                                                                                          | **Per-app**: Sign SAML response and assertion                                                                                                                         |
| Same, after signing change                                                                            | Token encryption enabled                                                                                                                              | Disable token encryption                                                                                                                                              |
| `SAMLSP gateway for user.saml.<name> has wrong configuration`                                         | **Most common: the proxy has no samlsp gateway.** GUI-created proxies never have one                                                                  | Add gateway 3                                                                                                                                                         |
| Same, samlsp gateway present                                                                          | Host/port/path mismatch between the SAML server's `single-sign-on-url` and the vhost bound to gateway 1; or gateway 1 `url-map` set to `"*"`/wildcard | Verify all three agree; restore gateway 1 to `/` + sub-string                                                                                                         |
| Same, and duplicate vhosts exist                                                                      | Two vhost objects claim the same `host`                                                                                                               | Bind to the one with the certificate, delete the other                                                                                                                |
| `Policy restriction! No policy matched! No end-point info found. Client certificate is not provided.` | Almost always group resolution                                                                                                                        | The certificate/endpoint wording is **boilerplate** — FortiOS pads the page with every possible reason. Operative phrase is **No policy matched**. Check `g_id` first |
| `No policy matched`, `g_id: 0`                                                                        | Group claim missing, GUID mismatch, or user not in the group                                                                                          | Read the assertion via samld debug                                                                                                                                    |
| `No policy matched`, `g_id` non-zero but wrong                                                        | Wrong SAML server fired — auth rule not scoped                                                                                                        | Scope auth rule `dstaddr` by real server                                                                                                                              |
| `g_id: 0` persists after a correct fix                                                                | **Stale session**                                                                                                                                     | `diagnose wad user clear` + fresh incognito                                                                                                                           |
| `g_id: 0` on genuinely fresh session                                                                  | Group overage (>150 groups drops the claim), or user genuinely not in the group                                                                       | Claim source → Groups assigned to the application; verify membership                                                                                                  |
| `<claim URI> was NOT FOUND`                                                                           | Username attribute references a claim Entra doesn't emit                                                                                              | Use `.../claims/emailaddress`                                                                                                                                         |
| AuthnRequest names the wrong FQDN                                                                     | Auth rule for another proxy matched first                                                                                                             | Scope by real server address                                                                                                                                          |
| Auth rule `dstaddr` scoping has no effect                                                             | Scoped to the VIP IP instead of the real server                                                                                                       | Use the real server address                                                                                                                                           |
| Proxy policy breaks after narrowing destination                                                       | Proxy policy doesn't evaluate destination against the VIP                                                                                             | Revert to `all`                                                                                                                                                       |
| Working proxy breaks after a GUI edit                                                                 | GUI rewrote gateway 1's url-map to wildcard, cleared service/port                                                                                     | `show firewall access-proxy <name>`, restore                                                                                                                          |
| Duplicate `auto-*` virtual host objects                                                               | GUI generates a new numbered object per edit                                                                                                          | Audit and remove orphans                                                                                                                                              |
| `command parse error` mid-paste                                                                       | FortiOS rejects `<` and `>` as XSS characters                                                                                                         | Substitute real values first; re-verify with `show`                                                                                                                   |

### Debug commands

**SAML — which SP fired, and what came back:**

```
diagnose debug reset
diagnose debug application samld -1
diagnose debug console timestamp enable
diagnose debug enable
```

Prints the AuthnRequest (the ACS URL identifies which proxy) and the fully decoded assertion with every claim. This is the highest-value debug in the whole stack.

**Access proxy / gateway matching:**

```
diagnose debug reset
diagnose debug console timestamp enable
diagnose wad debug enable category all
diagnose wad debug enable level verbose
diagnose debug enable
```

Extremely verbose — trigger the request, then `diagnose debug disable` immediately.

Lines worth knowing:

| Line                                                | Meaning                                           |
| --------------------------------------------------- | ------------------------------------------------- |
| `<vs-id>:<proxy>: matching vhost by: <fqdn>`        | Which proxy is handling the request               |
| `matched vhost(<name>)` / `no host matched`         | SNI resolution result                             |
| `matching gwy by (<path>) with vhost(<name>)`       | Gateway matching, scoped to the vhost             |
| `No matched gwy.`                                   | No gateway bound to that vhost                    |
| `wad_http_req_check_policy ... (src -> DST)`        | **The destination used for policy/auth matching** |
| `wad_http_policy_match_one: fw_pol_id=N`            | Candidate policy                                  |
| `__wad_fw_policy_match_user: matched cached grp:NA` | Group check failed                                |
| `match policy-id=0` + `POLICY DENIED`               | No policy matched                                 |

**Session state:**

```
diagnose wad user list      # username, g_id, pol_id
diagnose wad user clear     # before every retest
```

> `diagnose wad vs list` **does not exist on 7.2.13** (`command parse error before 'vs'`). The vhost/gateway hit-counter output shown in 7.6 documentation is unavailable here.

### Troubleshooting discipline

Hard-won:

- **One variable per test.** Stacked changes make it impossible to attribute a result.
- **`diagnose wad user clear` between every test**, plus a fresh incognito window.
- **Never paste a block containing `<placeholder>`.** A mid-block failure silently discards subsequent commands.
- **Verify a working proxy still works** after any change intended for a different one.
- **Read the debug before hypothesizing.** Every correct diagnosis came from debug output. Several confident theories derived from reading config alone turned out to be wrong.

---

## Configuration capture

```
config system console
    set output standard
end

get system status
show full-configuration firewall vip <vip-name>
show full-configuration firewall access-proxy
show firewall access-proxy-virtual-host
show firewall proxy-policy
show full-configuration user saml
show user group
show authentication scheme
show authentication rule
show authentication setting
get vpn certificate remote
diagnose wad user list
```

Restore with `set output more`.

`show` prints only non-default values — use `show full-configuration` to see defaults such as `saml-redirect disable` and `clock-tolerance 15`. This matters when comparing a broken proxy against a working one.

---

## Security note

This configuration authenticates the **user** and performs **no device verification**. With `client-cert disable` and no EMS tags, any device with valid credentials reaches the published application. At the FortiGate layer it's a SAML-authenticating reverse proxy, not zero trust.

That may be entirely appropriate — but it should be a deliberate decision, with device posture enforced somewhere (IdP conditional access, or FortiClient/EMS tags on the FortiGate), not an unnoticed gap.

### Adding FortiClient/EMS device posture

Verified working, though the deployment this guide came from ultimately chose IdP conditional access instead.

1. Deploy FortiClient with EMS registration to authorized endpoints
2. Build a Zero Trust Tagging Rule in EMS
3. Confirm the tag appears on the FortiGate as a dynamic address — `diagnose firewall dynamic list`. **Tag propagation is the most common silent failure**
4. `set client-cert enable` and `set empty-cert-action block` on the access proxy (`empty-cert-action` is hidden until `client-cert` is enabled)
5. `set ztna-ems-tag <tag>` on the proxy policy, using the exact string from `diagnose firewall dynamic list`
6. Test three cases: compliant (allow), non-compliant (deny), unregistered browser (ZTNA error 002)

For browser-based HTTPS access you do **not** need ZTNA Destination rules — those are for TCP forwarding of non-HTTP services or internal-only hostnames. EMS installs the client certificate locally and the browser presents it during the TLS handshake.

Suppress Chrome's per-session certificate prompt with the `AutoSelectCertificateForUrls` policy, scoped to the proxy FQDN.

**Known issue:** in one test, adding `ztna-ems-tag` produced an indefinitely hanging page rather than a clean denial. Root cause was not established. If you hit this, capture `diagnose wad debug` during the hang and confirm the tag resolves to a populated member set.

**Per-user exceptions** (allowing specific users from unmanaged devices) require two ordered proxy-policies, the exception policy carrying no `ztna-ems-tag`. This only works with `empty-cert-action accept`, which removes the certificate requirement as a hard floor for everyone. A cleaner split is two access proxies with separate FQDNs and real servers — one enforcing, one not.

---

## Rollback

**Emergency:**

```
config firewall proxy-policy
    edit <id>
        set status disable
    next
end
config firewall vip
    edit "<vip-name>"
        set status disable
    next
end
```

**Full removal** — in this order; FortiOS refuses to delete referenced objects:

```
config firewall proxy-policy
    delete <id>
end
config firewall access-proxy
    delete "ZTNA-MyApp"
end
config firewall vip
    delete "ZTNA-MyApp"
end
config firewall access-proxy-virtual-host
    delete "vh-myapp"
end
config authentication rule
    delete "MyApp SAML"
end
config authentication scheme
    delete "MyApp SAML Scheme"
end
config user group
    delete "MyApp-Entra-SSO"
end
config user saml
    delete "MyApp Entra SSO"
end
```

---

## Contributing

Corrections welcome, particularly:

- Confirmed GUI menu paths for specific 7.2.x builds
- Behavior on 7.2 releases other than 7.2.13
- Root cause of the `ztna-ems-tag` hanging-page issue
- Any documented mechanism for scoping authentication rules per access proxy other than real-server `dstaddr`

Please include your exact FortiOS build and relevant `diagnose` output.

---

## Disclaimer

Provided as-is with no warranty. Verified on FortiOS 7.2.13 only. Test in a lab before applying to production, and be aware that several steps here — particularly modifying authentication rules and proxy policies on a device with existing access proxies — can take down working applications. That happened twice during the deployment this guide documents.
