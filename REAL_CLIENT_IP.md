# Real client IP — community application stacks

Requests reach a workload from the platform's Traefik pod, never directly from
the internet. The TCP peer an app sees is therefore a **pod address**
(`10.42.x.x`); the visitor's address — IPv4 **or** IPv6 — arrives in headers.

Traefik sets both:

| Header | Contents |
|---|---|
| `X-Forwarded-For` | the chain, with the **real client appended last** |
| `X-Real-Ip` | a single address — the visitor |

An app that reads the peer directly sees one internal address for the whole
internet, which breaks rate-limiting, geo-logic, audit logs, abuse blocking and
"last login IP".

This is **not** an IPv6 issue. The platform serves IPv6 on the public edge and
routes IPv4 internally, so a v6 visitor's address cannot survive as a TCP peer by
construction — and v4 visitors are masked identically. IPv6 only made it visible.

> The Official Catalog's runtimes and web images are covered separately in
> [application-catalog/REAL_CLIENT_IP.md](https://github.com/insulahq/application-catalog/blob/main/REAL_CLIENT_IP.md).

## The trust list

Where an app takes a proxy list, we set the same four ranges everywhere:

```
10.0.0.0/8,172.16.0.0/12,192.168.0.0/16,fd00::/8
```

Private ranges **only** — `fd00::/8` covers the dual-stack pod CIDR's IPv6 ULA.
A public client's own address can therefore never be a trusted proxy, so the
right-most untrusted entry of `X-Forwarded-For` is the true client. Since Traefik
*appends* the peer it observed, a visitor who sends their own `X-Forwarded-For`
only prepends noise and cannot displace themselves.

**Never `*` / `true` / "trust all".** Every one of these apps is reachable only
through the platform ingress, but a wildcard trust list means anyone who ever
reaches the pod directly can claim any address — bypassing every rate limit and
poisoning every audit trail.

## Configured by the platform — nothing for you to do

Set in each entry's `manifest.json` under `env_vars.fixed` (the deploy source —
chart values alone are **not** what the platform renders).

| App | Setting |
|---|---|
| BookStack | `APP_PROXIES` (Laravel TrustProxies) |
| Directus | `IP_TRUST_PROXY` |
| Gitea | `GITEA__security__REVERSE_PROXY_TRUSTED_PROXIES` |
| Immich | `IMMICH_TRUSTED_PROXIES` |
| Invoice Ninja | `TRUSTED_PROXIES` |
| Nextcloud | `TRUSTED_PROXIES` (space-separated, per nextcloud/docker) |
| n8n | `N8N_PROXY_HOPS=1` — hop count, not a CIDR list |
| Rocket.Chat | `HTTP_FORWARDED_COUNT=1` — hop count |
| Moodle | `REVERSEPROXY=true`, `SSLPROXY=true` |
| Paperless-ngx | `PAPERLESS_ALLAUTH_TRUSTED_CLIENT_IP_HEADER=X-Real-IP` |
| Vaultwarden | `IP_HEADER=X-Real-IP` (its default; set explicitly so it is not silently changed) |

Paperless-ngx is worth a note: its login/rate-limit IP handling is allauth's, and
allauth's documented knob for "my proxy sets a dedicated real-client-IP header"
is `ALLAUTH_TRUSTED_CLIENT_IP_HEADER`. That fits Traefik's `X-Real-Ip` exactly and
— unlike `PAPERLESS_TRUSTED_PROXIES`, which wants literal IP **addresses** — needs
no proxy list, which we could not supply anyway because Traefik's pod IP is not
stable.

## Correct already, by the framework's own defaults

Verified rather than assumed — no configuration added, because none is needed:

- **Discourse** — Rails' `ActionDispatch::RemoteIp` is enabled by default and
  already treats `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `127.0.0.1`,
  `::1` and `fc00::/7` as proxies, so `request.remote_ip` is the visitor.
- **PrestaShop** — `Tools::getRemoteAddr()` honours `HTTP_X_FORWARDED_FOR` when
  `REMOTE_ADDR` is in `127.*`, `10.*`, `172.16–31.*` or `192.168.*`. Confirmed
  against the source: the `10.0.0.0/8` branch is present, which is the one our
  pod CIDR needs. (Those checks are IPv4-only regexes — fine today, because
  cluster-internal traffic is IPv4; revisit if tenant Services ever go
  dual-stack.)
- **Plausible** — reads `X-Forwarded-For` directly and exposes no trust knob. It
  also does not store raw visitor IPs by design, using them only to derive
  location, so there is nothing to configure or leak.

## Needs an action in the app — no platform-side switch exists

| App | What to do |
|---|---|
| **Uptime Kuma** | **Settings → Reverse Proxy → HTTP Headers → Trust Proxy: Yes.** Stored in the app database, not an environment variable, so it cannot be preset at deploy time. |
| **Wiki.js** | Express-based with no exposed `trust proxy` setting. Read `X-Forwarded-For` in a custom rule if you need the visitor address. |
| **WordPress** | The official image ships no proxy environment variable. Add to `wp-config.php` **above** the "That's all, stop editing" line — see below. |
| **Jitsi**, **coturn** | Real-time media, not HTTP request logging; the client-IP question does not apply in the same form. |

WordPress:

```php
// Trust the platform ingress: the peer is always a Traefik pod in a private
// range, and Traefik appends the visitor last, so take the RIGHT-most entry.
if ( ! empty( $_SERVER['HTTP_X_FORWARDED_FOR'] )
     && preg_match( '/^(10\.|172\.(1[6-9]|2\d|3[01])\.|192\.168\.|127\.)/', $_SERVER['REMOTE_ADDR'] ) ) {
	$parts = array_map( 'trim', explode( ',', $_SERVER['HTTP_X_FORWARDED_FOR'] ) );
	$_SERVER['REMOTE_ADDR'] = end( $parts );
}
```

## Checking it

Deploy, visit the site, and confirm the app's log or admin UI shows a public
address rather than `10.42.x.x`. From a v6-capable host,
`curl -6 https://<your-domain>/` should surface the IPv6 address — if it shows the
pod IP, the app is not trusting the header.
