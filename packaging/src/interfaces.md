# Interfaces

`setupInterfaces()` defines the network interfaces your service exposes and how they are made available to the user. This function runs on service install, update, and config save.

## Single Interface

For a service with one web interface:

```typescript
import { i18n } from './i18n'
import { sdk } from './sdk'

export const setInterfaces = sdk.setupInterfaces(async ({ effects }) => {
  const multi = sdk.MultiHost.of(effects, 'ui')
  const origin = await multi.bindPort(80, {
    protocol: 'http',
    preferredExternalPort: 80,
  })

  const ui = sdk.createInterface(effects, {
    name: i18n('Web Interface'),
    id: 'ui',
    description: i18n('The main web interface'),
    type: 'ui',
    masked: false,
    schemeOverride: null,
    username: null,
    path: '',
    query: {},
  })

  return [await origin.export([ui])]
})
```

## Multiple Interfaces

Expose multiple paths (e.g., web UI and admin panel) from the same port:

```typescript
export const setInterfaces = sdk.setupInterfaces(async ({ effects }) => {
  const multi = sdk.MultiHost.of(effects, 'web')
  const origin = await multi.bindPort(80, {
    protocol: 'http',
    preferredExternalPort: 80,
  })

  const ui = sdk.createInterface(effects, {
    name: i18n('Web UI'),
    id: 'ui',
    description: i18n('The web interface'),
    type: 'ui',
    masked: false,
    schemeOverride: null,
    username: null,
    path: '',
    query: {},
  })

  const admin = sdk.createInterface(effects, {
    name: i18n('Admin Panel'),
    id: 'admin',
    description: i18n('Admin interface'),
    type: 'ui',
    masked: false,
    schemeOverride: null,
    username: null,
    path: '/admin/',
    query: {},
  })

  return [await origin.export([ui, admin])]
})
```

Expose interfaces on separate ports:

```typescript
export const setInterfaces = sdk.setupInterfaces(async ({ effects }) => {
  const receipts = []

  // Web UI — HTTP
  const uiMulti = sdk.MultiHost.of(effects, 'ui')
  const uiOrigin = await uiMulti.bindPort(80, {
    protocol: 'http',
    preferredExternalPort: 80,
  })
  const ui = sdk.createInterface(effects, {
    name: i18n('Web Interface'),
    id: 'ui',
    description: i18n('The main browser interface'),
    type: 'ui',
    masked: false,
    schemeOverride: null,
    username: null,
    path: '',
    query: {},
  })
  receipts.push(await uiOrigin.export([ui]))

  // API — HTTPS with SSL termination
  const apiMulti = sdk.MultiHost.of(effects, 'api')
  const apiOrigin = await apiMulti.bindPort(8080, {
    protocol: 'https',
    preferredExternalPort: 8080,
    addSsl: {
      alpn: null,
      preferredExternalPort: 8080,
      addXForwardedHeaders: false,
    },
  })
  const api = sdk.createInterface(effects, {
    name: i18n('REST API'),
    id: 'api',
    description: i18n('Programmatic access'),
    type: 'api',
    masked: true,
    schemeOverride: null,
    username: null,
    path: '',
    query: {},
  })
  receipts.push(await apiOrigin.export([api]))

  // Peer — raw TCP (not HTTP)
  const peerMulti = sdk.MultiHost.of(effects, 'peer')
  const peerOrigin = await peerMulti.bindPort(9735, {
    protocol: null,
    addSsl: null,
    preferredExternalPort: 9735,
    secure: { ssl: false },
  })
  const peer = sdk.createInterface(effects, {
    name: i18n('Peer Interface'),
    id: 'peer',
    description: i18n('Peer-to-peer network connections'),
    type: 'p2p',
    masked: true,
    schemeOverride: null,
    username: null,
    path: '',
    query: {},
  })
  receipts.push(await peerOrigin.export([peer]))

  return receipts
})
```

The key steps are:

1. Create a `MultiHost` and bind a port with protocol and options
2. Create one or more interfaces using `sdk.createInterface()`
3. Export the interfaces from the origin and return the receipt(s)

## bindPort Options

| Option | Type | Description |
|--------|------|-------------|
| `protocol` | `'http'` \| `'https'` \| `null` | The protocol. Use `null` for raw TCP (non-HTTP). |
| `preferredExternalPort` | `number` | The port users will see in their URLs. |
| `addSsl` | `object` \| `null` | SSL termination options for HTTPS. Set to `null` for no SSL. |
| `addSsl.alpn` | `string` \| `null` | ALPN protocol negotiation (e.g., `'h2'`). Usually `null`. |
| `addSsl.preferredExternalPort` | `number` | External port for SSL connections. |
| `addSsl.addXForwardedHeaders` | `boolean` | Whether to add `X-Forwarded-*` headers. |
| `secure` | `{ ssl: boolean }` \| `null` | For non-HTTP protocols, whether the connection is secure. |

> [!WARNING]
> When `protocol: null`, the value of `secure` controls whether the binding is exposed on the LAN. Passing `secure: null` produces a Tor-only binding — no LAN iptables DNAT rule is written, and the port silently does not receive LAN traffic. To expose a non-HTTP service (raw TCP, UDP, peer-to-peer) on the LAN, you must pass `secure: { ssl: false }` (or `{ ssl: true }` if your service terminates TLS itself) **and** call `Origin.export([iface])` with at least one `createInterface` of `type: 'p2p'`. `bindPort()` alone does not expose the port — the `Origin.export()` step is what attaches the service interface and triggers the host-side forward-port script. This applies to UDP listeners (NetFlow, IPFIX, sFlow, WireGuard, DNS) and any raw-TCP peer service.

## Interface Options

```typescript
sdk.createInterface(effects, {
  name: i18n('Display Name'),      // Shown in UI (wrap with i18n)
  id: 'unique-id',                 // Used in sdk.serviceInterface.getOwn()
  description: i18n('Description'),// Shown in UI (wrap with i18n)
  type: 'ui',                      // 'ui', 'api', or 'p2p'
  masked: false,                   // Hide URLs with sensitive credentials?
  schemeOverride: null,            // Override URL scheme (see below)
  username: null,                  // Auth username embedded in URL
  path: '/some/path/',             // URL path
  query: {},                       // URL query params
})
```

| Option | Type | Description |
|--------|------|-------------|
| `name` | `string` | Display name shown to the user. Wrap with `i18n()`. |
| `id` | `string` | Unique identifier. Used to retrieve this interface in [main.ts](./main.md) via `sdk.serviceInterface.getOwn()`. |
| `description` | `string` | Description shown to the user. Wrap with `i18n()`. |
| `type` | `'ui'`, `'api'`, or `'p2p'` | `'ui'` for browser interfaces, `'api'` for programmatic endpoints, `'p2p'` for peer-to-peer connections. |
| `masked` | `boolean` | If `true`, the interface URL is shown as a copyable secret. Use for URLs containing credentials or tokens. |
| `schemeOverride` | `{ ssl: string \| null; noSsl: string \| null }` \| `null` | Override the URL scheme for custom protocols. For example, `{ ssl: 'lndconnect', noSsl: 'lndconnect' }` produces `lndconnect://` URLs. Use `null` for standard `http`/`https`. |
| `username` | `string` \| `null` | Username embedded in the URL (e.g., for `smp://fingerprint:password@host`). |
| `path` | `string` | URL path appended to the base address (e.g., `'/admin/'`). |
| `query` | `object` | URL query parameters as key-value pairs (e.g., `{ macaroon: 'abc123' }`). |

> [!TIP]
> The `id` you assign to an interface is what you use in `main.ts` to retrieve hostnames for that interface. For example, if you set `id: 'ui'`, you would call `sdk.serviceInterface.getOwn(effects, 'ui')` to get its address information. See [Main](./main.md#getting-hostnames) for details.

## TLS Termination

StartOS terminates TLS at the platform edge and proxies plain HTTP to your container. This has two important consequences any time your service generates URLs or makes scheme decisions:

**1. Inside the container, every request arrives over HTTP.** A reverse proxy like nginx will see `$scheme == "http"`, the `X-Forwarded-Proto` header is not authoritative by default, and there is no TLS certificate to terminate. Do not configure in-container HTTPS — StartOS is already doing it.

**2. The browser loaded the page over `https://`.** Any URL your service emits for the browser to consume (login redirects, API endpoints in a `config.json`, OAuth callbacks, absolute links in HTML) must use `https://`. If you emit `http://` or derive the scheme from `$scheme`, the browser will block the request as [mixed active content](https://developer.mozilla.org/en-US/docs/Web/Security/Mixed_content).

**Hardcode `https://` for browser-facing URLs** rather than interpolating `$scheme` or reading the protocol from the incoming request:

```nginx
# BAD — $scheme is always "http" inside the container
return 200 '{"api_url":"$scheme://$host/api"}';

# GOOD — match what the browser actually sees
return 200 '{"api_url":"https://$host/api"}';
```

This applies to any configuration file generated in `setupMain` or any runtime response that includes absolute URLs — not just nginx. When in doubt, hardcode `https://`.
