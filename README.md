# cloudflare-zig

Use Cloudflare from a Zig application: look up domains, manage DNS records, inspect security settings, or fetch a webpage through Cloudflare's hosted browser.

This is an **unofficial, dependency-free client for the Cloudflare v4 API**. A developer adds it to a Zig program to send requests with your Cloudflare account.

[Get started](#get-started) · [Endpoint guide](docs/endpoints.md) · [Browser Run guide](docs/browser-run.md) · [Official API documentation](https://developers.cloudflare.com/api/)

## What can you use it for?

- **Keep a domain pointed at the right server.** Find its Cloudflare zone (the domain's configuration), read its DNS records, then explicitly create or update the record that points to your server.
- **Build an infrastructure dashboard.** Read account details, domain settings, traffic reports, load balancer health, access policies, and audit logs.
- **Collect a rendered webpage or screenshot.** Use Browser Run when you need a browser to load a page before retrieving its content.

The library covers these areas:

| Area | Examples | Reference |
| --- | --- | --- |
| Accounts and identity | Accounts, members, groups, tokens, resource tags | [Endpoints](docs/endpoints.md#accounts-and-identity) |
| Domains and DNS | Zones, DNS records, DNSSEC, secondary DNS, DNS firewall | [Endpoints](docs/endpoints.md#domains-and-dns) |
| Traffic and availability | Load balancers, pools, monitors, health checks | [Endpoints](docs/endpoints.md#load-balancing-and-health-checks) |
| Website security | Rulesets, IP rules, Page Shield, API Shield, custom pages | [Endpoints](docs/endpoints.md#website-security-and-rules) |
| Email | Routing, authentication reports, sending settings, security settings | [Endpoints](docs/endpoints.md#email) |
| Private access and networking | Access, tunnels, Zero Trust and Gateway settings | [Endpoints](docs/endpoints.md#access), [Tunnels](docs/endpoints.md#tunnels), [Zero Trust](docs/endpoints.md#zero-trust-and-gateway) |
| Reports and logs | Security reports, audit logs, Logpush, Log Explorer | [Reports](docs/endpoints.md#security-reports-and-audit), [Logs](docs/endpoints.md#logs) |
| HTTPS and domain settings | Certificates, TLS, cache and performance settings | [TLS](docs/endpoints.md#certificates-and-tls), [Settings](docs/endpoints.md#zone-settings-and-performance) |
| Hosted browser | HTML, screenshots, browser sessions and targets | [Browser Run](docs/browser-run.md) |

**Read methods send requests; mutation helpers build routes and preview plans.** To change a resource, your application supplies the provider's JSON payload and explicitly calls `requestJson`. Browser Run has its own typed request methods. The [endpoint guide](docs/endpoints.md) shows this distinction, links matching public operation pages, and labels routes whose reference is the provider's schema.

## Get started

Requires **Zig 0.17.0**. The package is pre-1.0 and has no other Zig dependencies.

### 1. Add the dependency

From your Zig project's directory:

```sh
zig fetch --save=cloudflare git+https://github.com/tzekid/cloudflare-zig
```

This adds a dependency with a content hash to `build.zig.zon`. Commit that file so other builds use the same package. For a specific revision, append `#<commit>` to the repository URL.

In `build.zig`, after creating your executable, add its import:

```zig
const dependency = b.dependency("cloudflare", .{
    .target = target,
    .optimize = optimize,
});
exe.root_module.addImport("cloudflare", dependency.module("cloudflare"));
```

Here, `target`, `optimize`, and `exe` are the values from your existing build.

### 2. Create an API token

[Create a Cloudflare API token](https://developers.cloudflare.com/fundamentals/api/get-started/create-token/) with permission to read the accounts or zones you will query. Permissions vary by endpoint; the linked official reference lists them.

The example below reads `CLOUDFLARE_API_TOKEN` from the environment. The library itself does not load environment variables or configuration files.

```sh
export CLOUDFLARE_API_TOKEN='your-api-token'
```

### 3. List the accounts your token can access

This example makes a read-only request, checks the HTTP status and Cloudflare's `success` field, then prints account IDs from one response page.

```zig
const std = @import("std");
const cloudflare = @import("cloudflare");

pub fn main(init: std.process.Init) !void {
    const token = init.environ_map.get("CLOUDFLARE_API_TOKEN") orelse
        return error.MissingCloudflareToken;
    const client = cloudflare.Client.init(.{ .token = token });

    const response = try client.getAccounts(init.io, init.gpa);
    defer response.deinit(init.gpa);
    if (response.status != .ok) return error.CloudflareHttpError;

    const envelope = try std.json.parseFromSlice(
        struct { success: bool },
        init.gpa,
        response.body,
        .{ .ignore_unknown_fields = true },
    );
    defer envelope.deinit();
    if (!envelope.value.success) return error.CloudflareApiError;

    const accounts = try cloudflare.models.parseAccountRows(init.gpa, response.body);
    defer accounts.deinit(init.gpa);
    for (accounts.items) |account| {
        std.debug.print("{s}\n", .{account.id});
    }
}
```

For DNS, start with `client.getZones(io, allocator, "example.com")` to find the zone ID, then `client.getDnsRecords(io, allocator, zone_id)`. See [the endpoint guide](docs/endpoints.md#using-the-routes) for route builders, JSON requests, and mutation previews.

## Browser Run

Choose the browser engine explicitly. Kitesurf is a beta engine intended for short rendering and extraction tasks; Chromium is the default full browser. The library does not switch engines automatically.

```zig
const browser = try client.browserRun(account_id, .kitesurf);
const chromium = try client.browserRun(account_id, .chromium_default);
```

Browser Run methods require an API token with **Browser Rendering – Edit** permission. They support HTML, bounded screenshots, and HTTP session/target management. CDP WebSocket communication needs a separate client.

The [Browser Run guide](docs/browser-run.md) includes examples, response handling, URL restrictions, and session cleanup. Refer to [Cloudflare's Kitesurf guide](https://developers.cloudflare.com/browser-run/kitesurf/) for current engine capabilities and limits.

## Request and response basics

- **Errors:** network and allocation failures are Zig errors. Ordinary HTTP API failures are returned as `Response` values; check `status` and the JSON `success`/`errors` fields before parsing results. The convenience row parsers can return an empty list for malformed or unexpected JSON, so an empty list alone does not establish that a request succeeded.
- **Pagination:** list helpers fetch one response page. Follow the endpoint's pagination rules in your application; normalized rows do not retain the full response envelope. Raw JSON remains available in `response.body`.
- **Ownership:** clients borrow credentials and configuration strings. Responses, parsed rows, allocated paths, and preview plans own their allocations. Use `deinit` or `allocator.free` with the same allocator that created them.
- **Transport:** buffered responses are capped at 12 MiB; the HTTP transport configures 30-second socket timeouts on Linux and macOS. Authenticated requests do not follow redirects automatically. Requests are not automatically retried.
- **Authentication:** API tokens are preferred. The ordinary REST client also accepts legacy `.email` and `.key` credentials; Browser Run requires a token.

[`src/root.zig`](src/root.zig) is the public entry point. Use `Client` for requests, `routes` for paths and operation metadata, and `models` for the provided normalized response types. This is a partial client, not a complete set of typed request and response schemas for every Cloudflare product.

## Development

```sh
zig build test
```

The suite runs without API credentials. The separate [Browser Run live check](docs/browser-run.md#verification) is opt-in and makes real API requests.

This repository is the canonical source. Make library changes here; consumers such as [Cloudio](https://github.com/tzekid/cloudio) pin a revision independently. Keep endpoint changes and this guide in sync.

[MIT license](LICENSE) · [Changelog](CHANGELOG.md)
