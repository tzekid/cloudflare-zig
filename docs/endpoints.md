# Cloudflare endpoint guide

[Back to the README](../README.md)

The REST paths below are relative to `https://api.cloudflare.com/client/v4`. An **endpoint** is a particular HTTP method and path. `GET` reads information; `POST`, `PUT`, `PATCH`, and `DELETE` have endpoint-specific effects explained in the linked reference. Browser Run also uses POST for rendering tasks.

`{name}` marks a value you supply, such as an account, server, or domain ID. The tables list routes exposed by the current library, not every provider API. The official reference defines request bodies, permissions, service availability, and response fields. These tables describe code coverage; they are not a claim that every operation has been tested against a live account.

Rows marked **schema** link to the upstream OpenAPI definition because a matching public operation page could not be verified. Check those definitions and the provider's current availability before relying on a legacy or restricted route.

## Find an operation

- [Accounts and identity](#accounts-and-identity) — 66 operations
- [Domains and DNS](#domains-and-dns) — 110 operations
- [Load balancing and health checks](#load-balancing-and-health-checks) — 75 operations
- [Website security and rules](#website-security-and-rules) — 128 operations
- [Email](#email) — 28 operations
- [Access](#access) — 118 operations
- [Tunnels](#tunnels) — 22 operations
- [Zero Trust and Gateway](#zero-trust-and-gateway) — 34 operations
- [Security reports and audit](#security-reports-and-audit) — 18 operations
- [Logs](#logs) — 18 operations
- [Certificates and TLS](#certificates-and-tls) — 28 operations
- [Zone settings and performance](#zone-settings-and-performance) — 9 operations
- [Browser Run](#browser-run) — 8 HTTP operations, with explicit engine selection

## Using the routes

For common reads, use the `Client` methods: `getAccounts`, `getZones`, `getDnsRecords`, or the appropriate `get...Endpoint` method. The helper column links to the corresponding route implementation and its argument types.

For an explicit DNS change, build its path and supply the body from the linked Cloudflare operation. This function sends a real request when called:

```zig
pub fn createDnsRecord(
    client: cloudflare.Client,
    io: std.Io,
    allocator: std.mem.Allocator,
    zone_id: []const u8,
    record_json: []const u8,
) !cloudflare.Response {
    const path = try cloudflare.routes.dnsRecordMutationPath(
        allocator, .create, .{ .zone_id = zone_id },
    );
    defer allocator.free(path);
    const url = try std.fmt.allocPrint(allocator, "{s}{s}", .{
        client.base_url_override, path,
    });
    defer allocator.free(url);
    const response = try client.requestJson(io, allocator, .POST, url, record_json);
    errdefer response.deinit(allocator);
    const status = @intFromEnum(response.status);
    if (status < 200 or status >= 300) return error.CloudflareHttpError;
    const envelope = try std.json.parseFromSlice(
        struct { success: bool }, allocator, response.body,
        .{ .ignore_unknown_fields = true },
    );
    defer envelope.deinit();
    if (!envelope.value.success) return error.CloudflareApiError;
    return response;
}
```

`std` and `cloudflare` are the imports used in the README example. The returned response belongs to the caller: use `defer response.deinit(allocator)` and read its JSON body for the created resource or action. The [create-record reference](https://developers.cloudflare.com/api/operations/dns-records-for-a-zone-create-dns-record) provides the JSON fields and token permissions. Your application chooses the record type, name, content, TTL, and proxy setting.

To inspect the intended operation without sending a request, use the matching preview helper:

```zig
const plan = try cloudflare.routes.dnsRecordMutationPlanJson(
    allocator, .create, .{ .zone_id = zone_id },
);
defer allocator.free(plan);
```

A preview contains method, path, operation metadata, and the request schema reference. It does not construct or validate your JSON payload, authorize a change, or execute it. The `...Path` helpers allocate paths; the `...PlanJson` helpers allocate preview JSON. Both are freed with the caller's allocator.

For larger collections, follow Cloudflare's `result_info` or the pagination format documented for that endpoint. `Client.get` accepts a full URL, so an application can explicitly request additional pages or filters. Row parsers normalize selected fields; retain `response.body` when you need the original envelope or fields not represented by the provided models.

## Accounts and identity

Use these routes to discover accounts, inspect who has access, manage API tokens, and label resources. Account and zone IDs identify existing resources; they are not domain names.

| HTTP method and path | What it does / official reference | Zig route helper |
| --- | --- | --- |
| `GET /accounts` | [List Accounts](https://developers.cloudflare.com/api/resources/accounts/methods/list/) | [`accountsUrl`](../src/routes.zig#L7572) |
| `POST /accounts` | [Create an Account](https://developers.cloudflare.com/api/resources/accounts/methods/create/) | [`accountMutationPath`](../src/routes.zig#L7604) |
| `POST /accounts/move` | Batch move accounts — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accountMutationPath`](../src/routes.zig#L7604) |
| `DELETE /accounts/{account_id}` | [Delete a specific account](https://developers.cloudflare.com/api/resources/accounts/methods/delete/) | [`accountMutationPath`](../src/routes.zig#L7604) |
| `GET /accounts/{account_id}` | [Account Details](https://developers.cloudflare.com/api/resources/accounts/methods/get/) | [`accountEndpointPath`](../src/routes.zig#L7595) |
| `PUT /accounts/{account_id}` | [Update Account](https://developers.cloudflare.com/api/resources/accounts/methods/update/) | [`accountMutationPath`](../src/routes.zig#L7604) |
| `GET /accounts/{account_id}/iam/permission_groups` | [List Account Permission Groups](https://developers.cloudflare.com/api/resources/iam/subresources/permission_groups/methods/list/) | [`accountIamCollectionPath`](../src/routes.zig#L7709) |
| `GET /accounts/{account_id}/iam/permission_groups/{permission_group_id}` | [Permission Group Details](https://developers.cloudflare.com/api/resources/iam/subresources/permission_groups/methods/get/) | [`accountIamResourcePath`](../src/routes.zig#L7721) |
| `GET /accounts/{account_id}/iam/resource_groups` | [List Resource Groups](https://developers.cloudflare.com/api/resources/iam/subresources/resource_groups/methods/list/) | [`accountIamCollectionPath`](../src/routes.zig#L7709) |
| `POST /accounts/{account_id}/iam/resource_groups` | [Create Resource Group](https://developers.cloudflare.com/api/resources/iam/subresources/resource_groups/methods/create/) | [`accountIamGroupMutationPath`](../src/routes.zig#L7729) |
| `DELETE /accounts/{account_id}/iam/resource_groups/{resource_group_id}` | [Remove Resource Group](https://developers.cloudflare.com/api/resources/iam/subresources/resource_groups/methods/delete/) | [`accountIamGroupMutationPath`](../src/routes.zig#L7729) |
| `GET /accounts/{account_id}/iam/resource_groups/{resource_group_id}` | [Resource Group Details](https://developers.cloudflare.com/api/resources/iam/subresources/resource_groups/methods/get/) | [`accountIamResourcePath`](../src/routes.zig#L7721) |
| `PUT /accounts/{account_id}/iam/resource_groups/{resource_group_id}` | [Update Resource Group](https://developers.cloudflare.com/api/resources/iam/subresources/resource_groups/methods/update/) | [`accountIamGroupMutationPath`](../src/routes.zig#L7729) |
| `GET /accounts/{account_id}/iam/user_groups` | [List User Groups](https://developers.cloudflare.com/api/resources/iam/subresources/user_groups/methods/list/) | [`accountIamCollectionPath`](../src/routes.zig#L7709) |
| `POST /accounts/{account_id}/iam/user_groups` | [Create User Group](https://developers.cloudflare.com/api/resources/iam/subresources/user_groups/methods/create/) | [`accountIamGroupMutationPath`](../src/routes.zig#L7729) |
| `DELETE /accounts/{account_id}/iam/user_groups/{user_group_id}` | [Remove User Group](https://developers.cloudflare.com/api/resources/iam/subresources/user_groups/methods/delete/) | [`accountIamGroupMutationPath`](../src/routes.zig#L7729) |
| `GET /accounts/{account_id}/iam/user_groups/{user_group_id}` | [User Group Details](https://developers.cloudflare.com/api/resources/iam/subresources/user_groups/methods/get/) | [`accountIamResourcePath`](../src/routes.zig#L7721) |
| `PUT /accounts/{account_id}/iam/user_groups/{user_group_id}` | [Update User Group](https://developers.cloudflare.com/api/resources/iam/subresources/user_groups/methods/update/) | [`accountIamGroupMutationPath`](../src/routes.zig#L7729) |
| `GET /accounts/{account_id}/iam/user_groups/{user_group_id}/members` | [List User Group Members](https://developers.cloudflare.com/api/resources/iam/subresources/user_groups/subresources/members/methods/list/) | [`accountUserGroupMembersPath`](../src/routes.zig#L7758) |
| `POST /accounts/{account_id}/iam/user_groups/{user_group_id}/members` | [Add User Group Members](https://developers.cloudflare.com/api/resources/iam/subresources/user_groups/subresources/members/methods/create/) | [`accountUserGroupMemberMutationPath`](../src/routes.zig#L7780) |
| `PUT /accounts/{account_id}/iam/user_groups/{user_group_id}/members` | [Update User Group Members](https://developers.cloudflare.com/api/resources/iam/subresources/user_groups/subresources/members/methods/update/) | [`accountUserGroupMemberMutationPath`](../src/routes.zig#L7780) |
| `DELETE /accounts/{account_id}/iam/user_groups/{user_group_id}/members/{member_id}` | [Remove User Group Member](https://developers.cloudflare.com/api/resources/iam/subresources/user_groups/subresources/members/methods/delete/) | [`accountUserGroupMemberMutationPath`](../src/routes.zig#L7780) |
| `GET /accounts/{account_id}/iam/user_groups/{user_group_id}/members/{member_id}` | [Get User Group Member](https://developers.cloudflare.com/api/resources/iam/subresources/user_groups/subresources/members/methods/get/) | [`accountUserGroupMemberPath`](../src/routes.zig#L7772) |
| `GET /accounts/{account_id}/members` | [List Members](https://developers.cloudflare.com/api/resources/accounts/subresources/members/methods/list/) | [`accountCollectionPath`](../src/routes.zig#L7645) |
| `POST /accounts/{account_id}/members` | [Add Member](https://developers.cloudflare.com/api/resources/accounts/subresources/members/methods/create/) | [`accountMemberMutationPath`](../src/routes.zig#L7665) |
| `DELETE /accounts/{account_id}/members/{member_id}` | [Remove Member](https://developers.cloudflare.com/api/resources/accounts/subresources/members/methods/delete/) | [`accountMemberMutationPath`](../src/routes.zig#L7665) |
| `GET /accounts/{account_id}/members/{member_id}` | [Member Details](https://developers.cloudflare.com/api/resources/accounts/subresources/members/methods/get/) | [`accountResourcePath`](../src/routes.zig#L7657) |
| `PUT /accounts/{account_id}/members/{member_id}` | [Update Member](https://developers.cloudflare.com/api/resources/accounts/subresources/members/methods/update/) | [`accountMemberMutationPath`](../src/routes.zig#L7665) |
| `POST /accounts/{account_id}/move` | [Move account to organization](https://developers.cloudflare.com/api/resources/accounts/subresources/account_organizations/methods/create/) | [`accountMutationPath`](../src/routes.zig#L7604) |
| `GET /accounts/{account_id}/organizations` | List account organizations — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accountEndpointPath`](../src/routes.zig#L7595) |
| `GET /accounts/{account_id}/profile` | [Get account profile](https://developers.cloudflare.com/api/resources/accounts/subresources/account_profile/methods/get/) | [`accountEndpointPath`](../src/routes.zig#L7595) |
| `PUT /accounts/{account_id}/profile` | [Update account profile](https://developers.cloudflare.com/api/resources/accounts/subresources/account_profile/methods/update/) | [`accountMutationPath`](../src/routes.zig#L7604) |
| `GET /accounts/{account_id}/roles` | [List Roles](https://developers.cloudflare.com/api/resources/accounts/subresources/roles/methods/list/) | [`accountCollectionPath`](../src/routes.zig#L7645) |
| `GET /accounts/{account_id}/roles/{role_id}` | [Role Details](https://developers.cloudflare.com/api/resources/accounts/subresources/roles/methods/get/) | [`accountResourcePath`](../src/routes.zig#L7657) |
| `DELETE /accounts/{account_id}/tags` | [Delete tags from an account-level resource](https://developers.cloudflare.com/api/resources/resource_tagging/subresources/account_tags/methods/delete/) | [`resourceTaggingMutationPath`](../src/routes.zig#L10005) |
| `GET /accounts/{account_id}/tags` | [Get tags for an account-level resource](https://developers.cloudflare.com/api/resources/resource_tagging/subresources/account_tags/methods/get/) | [`resourceTaggingAccountReadPath`](../src/routes.zig#L9959) |
| `PUT /accounts/{account_id}/tags` | [Set tags for an account-level resource](https://developers.cloudflare.com/api/resources/resource_tagging/subresources/account_tags/methods/update/) | [`resourceTaggingMutationPath`](../src/routes.zig#L10005) |
| `GET /accounts/{account_id}/tags/keys` | [List tag keys](https://developers.cloudflare.com/api/resources/resource_tagging/subresources/keys/methods/list/) | [`resourceTaggingAccountReadPath`](../src/routes.zig#L9959) |
| `GET /accounts/{account_id}/tags/resources` | [List tagged resources](https://developers.cloudflare.com/api/resources/resource_tagging/methods/list/) | [`resourceTaggingAccountReadPath`](../src/routes.zig#L9959) |
| `GET /accounts/{account_id}/tags/values/{tag_key}` | [List tag values](https://developers.cloudflare.com/api/resources/resource_tagging/subresources/values/methods/list/) | [`resourceTaggingAccountReadPath`](../src/routes.zig#L9959) |
| `GET /accounts/{account_id}/tokens` | [List Tokens](https://developers.cloudflare.com/api/resources/accounts/subresources/tokens/methods/list/) | [`accountTokenEndpointPath`](../src/routes.zig#L10039) |
| `POST /accounts/{account_id}/tokens` | [Create Token](https://developers.cloudflare.com/api/resources/accounts/subresources/tokens/methods/create/) | [`accountTokenMutationPath`](../src/routes.zig#L10059) |
| `GET /accounts/{account_id}/tokens/permission_groups` | [List Permission Groups](https://developers.cloudflare.com/api/resources/accounts/subresources/tokens/subresources/permission_groups/methods/list/) | [`accountTokenEndpointPath`](../src/routes.zig#L10039) |
| `GET /accounts/{account_id}/tokens/verify` | [Verify Token](https://developers.cloudflare.com/api/resources/accounts/subresources/tokens/methods/verify/) | [`accountTokenEndpointPath`](../src/routes.zig#L10039) |
| `DELETE /accounts/{account_id}/tokens/{token_id}` | [Delete Token](https://developers.cloudflare.com/api/resources/accounts/subresources/tokens/methods/delete/) | [`accountTokenMutationPath`](../src/routes.zig#L10059) |
| `GET /accounts/{account_id}/tokens/{token_id}` | [Token Details](https://developers.cloudflare.com/api/resources/accounts/subresources/tokens/methods/get/) | [`accountTokenPath`](../src/routes.zig#L10051) |
| `PUT /accounts/{account_id}/tokens/{token_id}` | [Update Token](https://developers.cloudflare.com/api/resources/accounts/subresources/tokens/methods/update/) | [`accountTokenMutationPath`](../src/routes.zig#L10059) |
| `PUT /accounts/{account_id}/tokens/{token_id}/value` | [Roll Token](https://developers.cloudflare.com/api/resources/accounts/subresources/tokens/subresources/value/methods/update/) | [`accountTokenMutationPath`](../src/routes.zig#L10059) |
| `GET /ips` | [Cloudflare/JD Cloud IP Details](https://developers.cloudflare.com/api/resources/ips/methods/list/) | [`ipsPath`](../src/routes.zig#L7582) |
| `GET /memberships` | [List Memberships](https://developers.cloudflare.com/api/resources/memberships/methods/list/) | [`identityEndpointUrl`](../src/routes.zig#L10132) |
| `DELETE /memberships/{membership_id}` | [Delete Membership](https://developers.cloudflare.com/api/resources/memberships/methods/delete/) | [`membershipMutationPath`](../src/routes.zig#L10148) |
| `GET /memberships/{membership_id}` | [Membership Details](https://developers.cloudflare.com/api/resources/memberships/methods/get/) | [`membershipPath`](../src/routes.zig#L10142) |
| `PUT /memberships/{membership_id}` | [Update Membership](https://developers.cloudflare.com/api/resources/memberships/methods/update/) | [`membershipMutationPath`](../src/routes.zig#L10148) |
| `GET /user` | [User Details](https://developers.cloudflare.com/api/resources/user/methods/get/) | [`identityEndpointUrl`](../src/routes.zig#L10132) |
| `GET /user/tenants` | [List user tenants](https://developers.cloudflare.com/api/resources/user/subresources/tenants/methods/list/) | [`identityEndpointUrl`](../src/routes.zig#L10132) |
| `GET /user/tokens` | [List Tokens](https://developers.cloudflare.com/api/resources/user/subresources/tokens/methods/list/) | [`userTokenEndpointUrl`](../src/routes.zig#L10084) |
| `POST /user/tokens` | [Create Token](https://developers.cloudflare.com/api/resources/user/subresources/tokens/methods/create/) | [`userTokenMutationPath`](../src/routes.zig#L10107) |
| `GET /user/tokens/permission_groups` | [List Token Permission Groups](https://developers.cloudflare.com/api/resources/user/subresources/tokens/subresources/permission_groups/methods/list/) | [`userTokenEndpointUrl`](../src/routes.zig#L10084) |
| `GET /user/tokens/verify` | [Verify Token](https://developers.cloudflare.com/api/resources/user/subresources/tokens/methods/verify/) | [`userTokenEndpointUrl`](../src/routes.zig#L10084) |
| `DELETE /user/tokens/{token_id}` | [Delete Token](https://developers.cloudflare.com/api/resources/user/subresources/tokens/methods/delete/) | [`userTokenMutationPath`](../src/routes.zig#L10107) |
| `GET /user/tokens/{token_id}` | [Token Details](https://developers.cloudflare.com/api/resources/user/subresources/tokens/methods/get/) | [`userTokenUrl`](../src/routes.zig#L10091) |
| `PUT /user/tokens/{token_id}` | [Update Token](https://developers.cloudflare.com/api/resources/user/subresources/tokens/methods/update/) | [`userTokenMutationPath`](../src/routes.zig#L10107) |
| `PUT /user/tokens/{token_id}/value` | [Roll Token](https://developers.cloudflare.com/api/resources/user/subresources/tokens/subresources/value/methods/update/) | [`userTokenMutationPath`](../src/routes.zig#L10107) |
| `DELETE /zones/{zone_id}/tags` | [Delete tags from a zone-level resource](https://developers.cloudflare.com/api/resources/resource_tagging/subresources/zone_tags/methods/delete/) | [`resourceTaggingMutationPath`](../src/routes.zig#L10005) |
| `GET /zones/{zone_id}/tags` | [Get tags for a zone-level resource](https://developers.cloudflare.com/api/resources/resource_tagging/subresources/zone_tags/methods/get/) | [`resourceTaggingZoneReadPath`](../src/routes.zig#L9995) |
| `PUT /zones/{zone_id}/tags` | [Set tags for a zone-level resource](https://developers.cloudflare.com/api/resources/resource_tagging/subresources/zone_tags/methods/update/) | [`resourceTaggingMutationPath`](../src/routes.zig#L10005) |

## Domains and DNS

Start by finding the zone for a domain, then use its zone ID to read or change records. DNS records control where names point; DNSSEC adds signed DNS data. Secondary DNS and DNS Firewall are separate services.

| HTTP method and path | What it does / official reference | Zig route helper |
| --- | --- | --- |
| `GET /accounts/{account_id}/dns_firewall` | [List DNS Firewall Clusters](https://developers.cloudflare.com/api/resources/dns_firewall/methods/list/) | [`dnsFirewallReadPath`](../src/routes.zig#L7870) |
| `POST /accounts/{account_id}/dns_firewall` | [Create DNS Firewall Cluster](https://developers.cloudflare.com/api/resources/dns_firewall/methods/create/) | [`dnsFirewallMutationPath`](../src/routes.zig#L7882) |
| `DELETE /accounts/{account_id}/dns_firewall/{dns_firewall_id}` | [Delete DNS Firewall Cluster](https://developers.cloudflare.com/api/resources/dns_firewall/methods/delete/) | [`dnsFirewallMutationPath`](../src/routes.zig#L7882) |
| `GET /accounts/{account_id}/dns_firewall/{dns_firewall_id}` | [DNS Firewall Cluster Details](https://developers.cloudflare.com/api/resources/dns_firewall/methods/get/) | [`dnsFirewallReadPath`](../src/routes.zig#L7870) |
| `PATCH /accounts/{account_id}/dns_firewall/{dns_firewall_id}` | [Update DNS Firewall Cluster](https://developers.cloudflare.com/api/resources/dns_firewall/methods/edit/) | [`dnsFirewallMutationPath`](../src/routes.zig#L7882) |
| `GET /accounts/{account_id}/dns_firewall/{dns_firewall_id}/dns_analytics/report` | [Table](https://developers.cloudflare.com/api/resources/dns_firewall/subresources/analytics/subresources/reports/methods/get/) | [`dnsFirewallAnalyticsPath`](../src/routes.zig#L7911) |
| `GET /accounts/{account_id}/dns_firewall/{dns_firewall_id}/dns_analytics/report/bytime` | [By Time](https://developers.cloudflare.com/api/resources/dns_firewall/subresources/analytics/subresources/reports/subresources/bytimes/methods/get/) | [`dnsFirewallAnalyticsPath`](../src/routes.zig#L7911) |
| `GET /accounts/{account_id}/dns_firewall/{dns_firewall_id}/reverse_dns` | [Show DNS Firewall Cluster Reverse DNS](https://developers.cloudflare.com/api/resources/dns_firewall/subresources/reverse_dns/methods/get/) | [`dnsFirewallReadPath`](../src/routes.zig#L7870) |
| `PATCH /accounts/{account_id}/dns_firewall/{dns_firewall_id}/reverse_dns` | [Update DNS Firewall Cluster Reverse DNS](https://developers.cloudflare.com/api/resources/dns_firewall/subresources/reverse_dns/methods/edit/) | [`dnsFirewallMutationPath`](../src/routes.zig#L7882) |
| `GET /accounts/{account_id}/dns_records/usage` | [Get DNS Record Usage for Account](https://developers.cloudflare.com/api/resources/dns/subresources/usage/subresources/account/methods/get/) | [`accountDnsRecordUsagePath`](../src/routes.zig#L10185) |
| `GET /accounts/{account_id}/dns_settings` | [Show DNS Settings](https://developers.cloudflare.com/api/resources/dns/subresources/settings/subresources/account/methods/get/) | [`accountDnsSettingsPath`](../src/routes.zig#L10173) |
| `PATCH /accounts/{account_id}/dns_settings` | [Update DNS Settings](https://developers.cloudflare.com/api/resources/dns/subresources/settings/subresources/account/methods/edit/) | [`dnsSettingsMutationPath`](../src/routes.zig#L7917) |
| `GET /accounts/{account_id}/secondary_dns/acls` | [List ACLs](https://developers.cloudflare.com/api/resources/dns/subresources/zone_transfers/subresources/acls/methods/list/) | [`secondaryDnsAccountCollectionPath`](../src/routes.zig#L7808) |
| `POST /accounts/{account_id}/secondary_dns/acls` | [Create ACL](https://developers.cloudflare.com/api/resources/dns/subresources/zone_transfers/subresources/acls/methods/create/) | [`secondaryDnsAccountMutationPath`](../src/routes.zig#L7828) |
| `DELETE /accounts/{account_id}/secondary_dns/acls/{acl_id}` | [Delete ACL](https://developers.cloudflare.com/api/resources/dns/subresources/zone_transfers/subresources/acls/methods/delete/) | [`secondaryDnsAccountMutationPath`](../src/routes.zig#L7828) |
| `GET /accounts/{account_id}/secondary_dns/acls/{acl_id}` | [ACL Details](https://developers.cloudflare.com/api/resources/dns/subresources/zone_transfers/subresources/acls/methods/get/) | [`secondaryDnsAccountResourcePath`](../src/routes.zig#L7820) |
| `PUT /accounts/{account_id}/secondary_dns/acls/{acl_id}` | [Update ACL](https://developers.cloudflare.com/api/resources/dns/subresources/zone_transfers/subresources/acls/methods/update/) | [`secondaryDnsAccountMutationPath`](../src/routes.zig#L7828) |
| `GET /accounts/{account_id}/secondary_dns/peers` | [List Peers](https://developers.cloudflare.com/api/resources/dns/subresources/zone_transfers/subresources/peers/methods/list/) | [`secondaryDnsAccountCollectionPath`](../src/routes.zig#L7808) |
| `POST /accounts/{account_id}/secondary_dns/peers` | [Create Peer](https://developers.cloudflare.com/api/resources/dns/subresources/zone_transfers/subresources/peers/methods/create/) | [`secondaryDnsAccountMutationPath`](../src/routes.zig#L7828) |
| `DELETE /accounts/{account_id}/secondary_dns/peers/{peer_id}` | [Delete Peer](https://developers.cloudflare.com/api/resources/dns/subresources/zone_transfers/subresources/peers/methods/delete/) | [`secondaryDnsAccountMutationPath`](../src/routes.zig#L7828) |
| `GET /accounts/{account_id}/secondary_dns/peers/{peer_id}` | [Peer Details](https://developers.cloudflare.com/api/resources/dns/subresources/zone_transfers/subresources/peers/methods/get/) | [`secondaryDnsAccountResourcePath`](../src/routes.zig#L7820) |
| `PUT /accounts/{account_id}/secondary_dns/peers/{peer_id}` | [Update Peer](https://developers.cloudflare.com/api/resources/dns/subresources/zone_transfers/subresources/peers/methods/update/) | [`secondaryDnsAccountMutationPath`](../src/routes.zig#L7828) |
| `GET /accounts/{account_id}/secondary_dns/tsigs` | [List TSIGs](https://developers.cloudflare.com/api/resources/dns/subresources/zone_transfers/subresources/tsigs/methods/list/) | [`secondaryDnsAccountCollectionPath`](../src/routes.zig#L7808) |
| `POST /accounts/{account_id}/secondary_dns/tsigs` | [Create TSIG](https://developers.cloudflare.com/api/resources/dns/subresources/zone_transfers/subresources/tsigs/methods/create/) | [`secondaryDnsAccountMutationPath`](../src/routes.zig#L7828) |
| `DELETE /accounts/{account_id}/secondary_dns/tsigs/{tsig_id}` | [Delete TSIG](https://developers.cloudflare.com/api/resources/dns/subresources/zone_transfers/subresources/tsigs/methods/delete/) | [`secondaryDnsAccountMutationPath`](../src/routes.zig#L7828) |
| `GET /accounts/{account_id}/secondary_dns/tsigs/{tsig_id}` | [TSIG Details](https://developers.cloudflare.com/api/resources/dns/subresources/zone_transfers/subresources/tsigs/methods/get/) | [`secondaryDnsAccountResourcePath`](../src/routes.zig#L7820) |
| `PUT /accounts/{account_id}/secondary_dns/tsigs/{tsig_id}` | [Update TSIG](https://developers.cloudflare.com/api/resources/dns/subresources/zone_transfers/subresources/tsigs/methods/update/) | [`secondaryDnsAccountMutationPath`](../src/routes.zig#L7828) |
| `GET /zones` | [List Zones](https://developers.cloudflare.com/api/resources/zones/methods/list/) | [`zonesUrl`](../src/routes.zig#L10191) |
| `POST /zones` | [Create Zone](https://developers.cloudflare.com/api/resources/zones/methods/create/) | [`zoneMutationPath`](../src/routes.zig#L10282) |
| `DELETE /zones/{zone_id}` | [Delete Zone](https://developers.cloudflare.com/api/resources/zones/methods/delete/) | [`zoneMutationPath`](../src/routes.zig#L10282) |
| `PATCH /zones/{zone_id}` | [Edit Zone](https://developers.cloudflare.com/api/resources/zones/methods/edit/) | [`zoneMutationPath`](../src/routes.zig#L10282) |
| `PUT /zones/{zone_id}/activation_check` | [Rerun the Activation Check](https://developers.cloudflare.com/api/resources/zones/subresources/activation_check/methods/trigger/) | [`zoneMutationPath`](../src/routes.zig#L10282) |
| `GET /zones/{zone_id}/analytics/latency` | Argo Analytics for a zone — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`zoneLifecycleReadPath`](../src/routes.zig#L10333) |
| `GET /zones/{zone_id}/analytics/latency/colos` | Argo Analytics for a zone at different PoPs — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`zoneLifecycleReadPath`](../src/routes.zig#L10333) |
| `GET /zones/{zone_id}/argo/smart_routing` | [Get Argo Smart Routing setting](https://developers.cloudflare.com/api/resources/argo/subresources/smart_routing/methods/get/) | [`zoneLifecycleReadPath`](../src/routes.zig#L10333) |
| `PATCH /zones/{zone_id}/argo/smart_routing` | [Patch Argo Smart Routing setting](https://developers.cloudflare.com/api/resources/argo/subresources/smart_routing/methods/edit/) | [`zoneLifecycleMutationPath`](../src/routes.zig#L10347) |
| `GET /zones/{zone_id}/argo/tiered_caching` | [Get Tiered Caching setting](https://developers.cloudflare.com/api/resources/argo/subresources/tiered_caching/methods/get/) | [`zoneLifecycleReadPath`](../src/routes.zig#L10333) |
| `PATCH /zones/{zone_id}/argo/tiered_caching` | [Patch Tiered Caching setting](https://developers.cloudflare.com/api/resources/argo/subresources/tiered_caching/methods/edit/) | [`zoneLifecycleMutationPath`](../src/routes.zig#L10347) |
| `GET /zones/{zone_id}/available_plans` | [List Available Plans](https://developers.cloudflare.com/api/resources/zones/subresources/plans/methods/list/) | [`zoneLifecycleReadPath`](../src/routes.zig#L10333) |
| `GET /zones/{zone_id}/available_plans/{plan_identifier}` | [Available Plan Details](https://developers.cloudflare.com/api/resources/zones/subresources/plans/methods/get/) | [`zoneLifecycleReadPath`](../src/routes.zig#L10333) |
| `GET /zones/{zone_id}/available_rate_plans` | [List Available Rate Plans](https://developers.cloudflare.com/api/resources/zones/subresources/rate_plans/methods/get/) | [`zoneLifecycleReadPath`](../src/routes.zig#L10333) |
| `GET /zones/{zone_id}/cache/cache_reserve` | [Get Cache Reserve setting](https://developers.cloudflare.com/api/resources/cache/subresources/cache_reserve/methods/get/) | [`zoneLifecycleReadPath`](../src/routes.zig#L10333) |
| `PATCH /zones/{zone_id}/cache/cache_reserve` | [Change Cache Reserve setting](https://developers.cloudflare.com/api/resources/cache/subresources/cache_reserve/methods/edit/) | [`zoneLifecycleMutationPath`](../src/routes.zig#L10347) |
| `GET /zones/{zone_id}/cache/cache_reserve_clear` | [Get Cache Reserve Clear](https://developers.cloudflare.com/api/resources/cache/subresources/cache_reserve/methods/status/) | [`zoneLifecycleReadPath`](../src/routes.zig#L10333) |
| `POST /zones/{zone_id}/cache/cache_reserve_clear` | [Start Cache Reserve Clear](https://developers.cloudflare.com/api/resources/cache/subresources/cache_reserve/methods/clear/) | [`zoneLifecycleMutationPath`](../src/routes.zig#L10347) |
| `GET /zones/{zone_id}/cache/origin_post_quantum_encryption` | [Get Origin Post-Quantum Encryption setting](https://developers.cloudflare.com/api/resources/origin_post_quantum_encryption/methods/get/) | [`zoneLifecycleReadPath`](../src/routes.zig#L10333) |
| `PUT /zones/{zone_id}/cache/origin_post_quantum_encryption` | [Change Origin Post-Quantum Encryption setting](https://developers.cloudflare.com/api/resources/origin_post_quantum_encryption/methods/update/) | [`zoneLifecycleMutationPath`](../src/routes.zig#L10347) |
| `GET /zones/{zone_id}/cache/regional_tiered_cache` | [Get Regional Tiered Cache setting](https://developers.cloudflare.com/api/resources/cache/subresources/regional_tiered_cache/methods/get/) | [`zoneLifecycleReadPath`](../src/routes.zig#L10333) |
| `PATCH /zones/{zone_id}/cache/regional_tiered_cache` | [Change Regional Tiered Cache setting](https://developers.cloudflare.com/api/resources/cache/subresources/regional_tiered_cache/methods/edit/) | [`zoneLifecycleMutationPath`](../src/routes.zig#L10347) |
| `DELETE /zones/{zone_id}/cache/tiered_cache_smart_topology_enable` | [Delete Smart Tiered Cache setting](https://developers.cloudflare.com/api/resources/cache/subresources/smart_tiered_cache/methods/delete/) | [`zoneLifecycleMutationPath`](../src/routes.zig#L10347) |
| `GET /zones/{zone_id}/cache/tiered_cache_smart_topology_enable` | [Get Smart Tiered Cache setting](https://developers.cloudflare.com/api/resources/cache/subresources/smart_tiered_cache/methods/get/) | [`zoneLifecycleReadPath`](../src/routes.zig#L10333) |
| `PATCH /zones/{zone_id}/cache/tiered_cache_smart_topology_enable` | [Patch Smart Tiered Cache setting](https://developers.cloudflare.com/api/resources/cache/subresources/smart_tiered_cache/methods/edit/) | [`zoneLifecycleMutationPath`](../src/routes.zig#L10347) |
| `POST /zones/{zone_id}/cache/tiered_cache_smart_topology_enable` | [Create Smart Tiered Cache setting](https://developers.cloudflare.com/api/resources/cache/subresources/smart_tiered_cache/methods/create/) | [`zoneLifecycleMutationPath`](../src/routes.zig#L10347) |
| `DELETE /zones/{zone_id}/cache/variants` | [Delete variants setting](https://developers.cloudflare.com/api/resources/cache/subresources/variants/methods/delete/) | [`zoneLifecycleMutationPath`](../src/routes.zig#L10347) |
| `GET /zones/{zone_id}/cache/variants` | [Get variants setting](https://developers.cloudflare.com/api/resources/cache/subresources/variants/methods/get/) | [`zoneLifecycleReadPath`](../src/routes.zig#L10333) |
| `PATCH /zones/{zone_id}/cache/variants` | [Change variants setting](https://developers.cloudflare.com/api/resources/cache/subresources/variants/methods/edit/) | [`zoneLifecycleMutationPath`](../src/routes.zig#L10347) |
| `GET /zones/{zone_id}/cloud_connector/rules` | [Rules](https://developers.cloudflare.com/api/resources/cloud_connector/subresources/rules/methods/list/) | [`zoneLifecycleReadPath`](../src/routes.zig#L10333) |
| `PUT /zones/{zone_id}/cloud_connector/rules` | [Put Rules](https://developers.cloudflare.com/api/resources/cloud_connector/subresources/rules/methods/update/) | [`zoneLifecycleMutationPath`](../src/routes.zig#L10347) |
| `GET /zones/{zone_id}/dns_analytics/report` | [Table](https://developers.cloudflare.com/api/resources/dns/subresources/analytics/subresources/reports/methods/get/) | [`dnsAnalyticsPath`](../src/routes.zig#L10217) |
| `GET /zones/{zone_id}/dns_analytics/report/bytime` | [By Time](https://developers.cloudflare.com/api/resources/dns/subresources/analytics/subresources/reports/subresources/bytimes/methods/get/) | [`dnsAnalyticsPath`](../src/routes.zig#L10217) |
| `GET /zones/{zone_id}/dns_records` | [List DNS Records](https://developers.cloudflare.com/api/resources/dns/subresources/records/methods/list/) | [`dnsRecordsUrl`](../src/routes.zig#L10207) |
| `POST /zones/{zone_id}/dns_records` | [Create DNS Record](https://developers.cloudflare.com/api/resources/dns/subresources/records/methods/create/) | [`dnsRecordMutationPath`](../src/routes.zig#L10251) |
| `POST /zones/{zone_id}/dns_records/batch` | [Batch DNS Records](https://developers.cloudflare.com/api/resources/dns/subresources/records/methods/batch/) | [`dnsRecordMutationPath`](../src/routes.zig#L10251) |
| `GET /zones/{zone_id}/dns_records/export` | [Export DNS Records](https://developers.cloudflare.com/api/resources/dns/subresources/records/methods/export/) | [`dnsRecordReadPath`](../src/routes.zig#L10232) |
| `POST /zones/{zone_id}/dns_records/import` | [Import DNS Records](https://developers.cloudflare.com/api/resources/dns/subresources/records/methods/import/) | [`dnsRecordMutationPath`](../src/routes.zig#L10251) |
| `GET /zones/{zone_id}/dns_records/scan/review` | [List Scanned DNS Records](https://developers.cloudflare.com/api/resources/dns/subresources/records/methods/scan_list/) | [`dnsRecordReadPath`](../src/routes.zig#L10232) |
| `POST /zones/{zone_id}/dns_records/scan/review` | [Review Scanned DNS Records](https://developers.cloudflare.com/api/resources/dns/subresources/records/methods/scan_review/) | [`dnsRecordMutationPath`](../src/routes.zig#L10251) |
| `POST /zones/{zone_id}/dns_records/scan/trigger` | [Trigger DNS Record Scan](https://developers.cloudflare.com/api/resources/dns/subresources/records/methods/scan_trigger/) | [`dnsRecordMutationPath`](../src/routes.zig#L10251) |
| `GET /zones/{zone_id}/dns_records/usage` | [Get DNS Record Usage](https://developers.cloudflare.com/api/resources/dns/subresources/usage/subresources/zone/methods/get/) | [`dnsRecordReadPath`](../src/routes.zig#L10232) |
| `DELETE /zones/{zone_id}/dns_records/{dns_record_id}` | [Delete DNS Record](https://developers.cloudflare.com/api/resources/dns/subresources/records/methods/delete/) | [`dnsRecordMutationPath`](../src/routes.zig#L10251) |
| `GET /zones/{zone_id}/dns_records/{dns_record_id}` | [DNS Record Details](https://developers.cloudflare.com/api/resources/dns/subresources/records/methods/get/) | [`dnsRecordReadPath`](../src/routes.zig#L10232) |
| `PATCH /zones/{zone_id}/dns_records/{dns_record_id}` | [Update DNS Record](https://developers.cloudflare.com/api/resources/dns/subresources/records/methods/edit/) | [`dnsRecordMutationPath`](../src/routes.zig#L10251) |
| `PUT /zones/{zone_id}/dns_records/{dns_record_id}` | [Overwrite DNS Record](https://developers.cloudflare.com/api/resources/dns/subresources/records/methods/update/) | [`dnsRecordMutationPath`](../src/routes.zig#L10251) |
| `PATCH /zones/{zone_id}/dns_settings` | [Update DNS Settings](https://developers.cloudflare.com/api/resources/dns/subresources/settings/subresources/zone/methods/edit/) | [`dnsSettingsMutationPath`](../src/routes.zig#L7917) |
| `DELETE /zones/{zone_id}/dnssec` | [Delete DNSSEC records](https://developers.cloudflare.com/api/resources/dns/subresources/dnssec/methods/delete/) | [`dnssecMutationPath`](../src/routes.zig#L10408) |
| `GET /zones/{zone_id}/dnssec` | [DNSSEC Details](https://developers.cloudflare.com/api/resources/dns/subresources/dnssec/methods/get/) | [`zoneEndpointPath`](../src/routes.zig#L10321) |
| `PATCH /zones/{zone_id}/dnssec` | [Edit DNSSEC Status](https://developers.cloudflare.com/api/resources/dns/subresources/dnssec/methods/edit/) | [`dnssecMutationPath`](../src/routes.zig#L10408) |
| `GET /zones/{zone_id}/dnssec/zsk` | [List DNSSEC ZSKs](https://developers.cloudflare.com/api/resources/dns/subresources/dnssec/subresources/zsk/methods/list/) | [`zoneEndpointPath`](../src/routes.zig#L10321) |
| `GET /zones/{zone_id}/environments` | [List zone environments](https://developers.cloudflare.com/api/resources/zones/subresources/environments/methods/list/) | [`zoneLifecycleReadPath`](../src/routes.zig#L10333) |
| `PATCH /zones/{zone_id}/environments` | [Partially update zone environments](https://developers.cloudflare.com/api/resources/zones/subresources/environments/methods/edit/) | [`zoneLifecycleMutationPath`](../src/routes.zig#L10347) |
| `POST /zones/{zone_id}/environments` | [Create zone environments](https://developers.cloudflare.com/api/resources/zones/subresources/environments/methods/create/) | [`zoneLifecycleMutationPath`](../src/routes.zig#L10347) |
| `PUT /zones/{zone_id}/environments` | [Upsert zone environments](https://developers.cloudflare.com/api/resources/zones/subresources/environments/methods/update/) | [`zoneLifecycleMutationPath`](../src/routes.zig#L10347) |
| `DELETE /zones/{zone_id}/environments/{environment_id}` | [Delete zone environment](https://developers.cloudflare.com/api/resources/zones/subresources/environments/methods/delete/) | [`zoneLifecycleMutationPath`](../src/routes.zig#L10347) |
| `POST /zones/{zone_id}/environments/{environment_id}/purge_cache` | [Purge Cached Content by Environment](https://developers.cloudflare.com/api/resources/cache/methods/purge_environment/) | [`zoneMutationPath`](../src/routes.zig#L10282) |
| `POST /zones/{zone_id}/environments/{environment_id}/rollback` | [Roll back zone environment](https://developers.cloudflare.com/api/resources/zones/subresources/environments/methods/rollback/) | [`zoneLifecycleMutationPath`](../src/routes.zig#L10347) |
| `DELETE /zones/{zone_id}/hold` | [Remove Zone Hold](https://developers.cloudflare.com/api/resources/zones/subresources/holds/methods/delete/) | [`zoneLifecycleMutationPath`](../src/routes.zig#L10347) |
| `GET /zones/{zone_id}/hold` | [Get Zone Hold](https://developers.cloudflare.com/api/resources/zones/subresources/holds/methods/get/) | [`zoneLifecycleReadPath`](../src/routes.zig#L10333) |
| `PATCH /zones/{zone_id}/hold` | [Update Zone Hold](https://developers.cloudflare.com/api/resources/zones/subresources/holds/methods/edit/) | [`zoneLifecycleMutationPath`](../src/routes.zig#L10347) |
| `POST /zones/{zone_id}/hold` | [Create Zone Hold](https://developers.cloudflare.com/api/resources/zones/subresources/holds/methods/create/) | [`zoneLifecycleMutationPath`](../src/routes.zig#L10347) |
| `POST /zones/{zone_id}/purge_cache` | [Purge Cached Content](https://developers.cloudflare.com/api/resources/cache/methods/purge/) | [`zoneMutationPath`](../src/routes.zig#L10282) |
| `POST /zones/{zone_id}/secondary_dns/force_axfr` | [Force AXFR](https://developers.cloudflare.com/api/resources/dns/subresources/zone_transfers/subresources/force_axfr/methods/create/) | [`secondaryDnsZoneMutationPath`](../src/routes.zig#L10388) |
| `DELETE /zones/{zone_id}/secondary_dns/incoming` | [Delete Secondary Zone Configuration](https://developers.cloudflare.com/api/resources/dns/subresources/zone_transfers/subresources/incoming/methods/delete/) | [`secondaryDnsZoneMutationPath`](../src/routes.zig#L10388) |
| `GET /zones/{zone_id}/secondary_dns/incoming` | [Secondary Zone Configuration Details](https://developers.cloudflare.com/api/resources/dns/subresources/zone_transfers/subresources/incoming/methods/get/) | [`secondaryDnsZoneReadPath`](../src/routes.zig#L10382) |
| `POST /zones/{zone_id}/secondary_dns/incoming` | [Create Secondary Zone Configuration](https://developers.cloudflare.com/api/resources/dns/subresources/zone_transfers/subresources/incoming/methods/create/) | [`secondaryDnsZoneMutationPath`](../src/routes.zig#L10388) |
| `PUT /zones/{zone_id}/secondary_dns/incoming` | [Update Secondary Zone Configuration](https://developers.cloudflare.com/api/resources/dns/subresources/zone_transfers/subresources/incoming/methods/update/) | [`secondaryDnsZoneMutationPath`](../src/routes.zig#L10388) |
| `DELETE /zones/{zone_id}/secondary_dns/outgoing` | [Delete Primary Zone Configuration](https://developers.cloudflare.com/api/resources/dns/subresources/zone_transfers/subresources/outgoing/methods/delete/) | [`secondaryDnsZoneMutationPath`](../src/routes.zig#L10388) |
| `GET /zones/{zone_id}/secondary_dns/outgoing` | [Primary Zone Configuration Details](https://developers.cloudflare.com/api/resources/dns/subresources/zone_transfers/subresources/outgoing/methods/get/) | [`secondaryDnsZoneReadPath`](../src/routes.zig#L10382) |
| `POST /zones/{zone_id}/secondary_dns/outgoing` | [Create Primary Zone Configuration](https://developers.cloudflare.com/api/resources/dns/subresources/zone_transfers/subresources/outgoing/methods/create/) | [`secondaryDnsZoneMutationPath`](../src/routes.zig#L10388) |
| `PUT /zones/{zone_id}/secondary_dns/outgoing` | [Update Primary Zone Configuration](https://developers.cloudflare.com/api/resources/dns/subresources/zone_transfers/subresources/outgoing/methods/update/) | [`secondaryDnsZoneMutationPath`](../src/routes.zig#L10388) |
| `POST /zones/{zone_id}/secondary_dns/outgoing/disable` | [Disable Outgoing Zone Transfers](https://developers.cloudflare.com/api/resources/dns/subresources/zone_transfers/subresources/outgoing/methods/disable/) | [`secondaryDnsZoneMutationPath`](../src/routes.zig#L10388) |
| `POST /zones/{zone_id}/secondary_dns/outgoing/enable` | [Enable Outgoing Zone Transfers](https://developers.cloudflare.com/api/resources/dns/subresources/zone_transfers/subresources/outgoing/methods/enable/) | [`secondaryDnsZoneMutationPath`](../src/routes.zig#L10388) |
| `POST /zones/{zone_id}/secondary_dns/outgoing/force_notify` | [Force DNS NOTIFY](https://developers.cloudflare.com/api/resources/dns/subresources/zone_transfers/subresources/outgoing/methods/force_notify/) | [`secondaryDnsZoneMutationPath`](../src/routes.zig#L10388) |
| `GET /zones/{zone_id}/secondary_dns/outgoing/status` | [Get Outgoing Zone Transfer Status](https://developers.cloudflare.com/api/resources/dns/subresources/zone_transfers/subresources/outgoing/subresources/status/methods/get/) | [`secondaryDnsZoneReadPath`](../src/routes.zig#L10382) |
| `GET /zones/{zone_id}/smart_shield` | [Get Smart Shield Settings](https://developers.cloudflare.com/api/resources/smart_shield/methods/get/) | [`zoneLifecycleReadPath`](../src/routes.zig#L10333) |
| `PATCH /zones/{zone_id}/smart_shield` | [Patch Smart Shield Settings](https://developers.cloudflare.com/api/resources/smart_shield/methods/update/) | [`zoneLifecycleMutationPath`](../src/routes.zig#L10347) |
| `GET /zones/{zone_id}/smart_shield/cache_reserve_clear` | [Get Cache Reserve Clear](https://developers.cloudflare.com/api/resources/smart_shield/subresources/cache_reserve_clear/methods/status/) | [`zoneLifecycleReadPath`](../src/routes.zig#L10333) |
| `POST /zones/{zone_id}/smart_shield/cache_reserve_clear` | [Start Cache Reserve Clear](https://developers.cloudflare.com/api/resources/smart_shield/subresources/cache_reserve_clear/methods/clear/) | [`zoneLifecycleMutationPath`](../src/routes.zig#L10347) |
| `GET /zones/{zone_id}/subscription` | [Zone Subscription Details](https://developers.cloudflare.com/api/resources/zones/subresources/subscriptions/methods/get/) | [`zoneLifecycleReadPath`](../src/routes.zig#L10333) |
| `POST /zones/{zone_id}/subscription` | [Create Zone Subscription](https://developers.cloudflare.com/api/resources/zones/subresources/subscriptions/methods/create/) | [`zoneLifecycleMutationPath`](../src/routes.zig#L10347) |
| `PUT /zones/{zone_id}/subscription` | [Update Zone Subscription](https://developers.cloudflare.com/api/resources/zones/subresources/subscriptions/methods/update/) | [`zoneLifecycleMutationPath`](../src/routes.zig#L10347) |

## Load balancing and health checks

Use pools and monitors to observe the availability of origin servers. Load balancer changes affect how traffic is distributed; health-check results help a dashboard explain an outage.

| HTTP method and path | What it does / official reference | Zig route helper |
| --- | --- | --- |
| `GET /accounts/{account_id}/diagnostics/endpoint-healthchecks` | [List Endpoint Health Checks](https://developers.cloudflare.com/api/resources/diagnostics/subresources/endpoint-healthchecks/methods/list/) | [`endpointHealthCheckReadPath`](../src/routes.zig#L8174) |
| `POST /accounts/{account_id}/diagnostics/endpoint-healthchecks` | [Endpoint Health Check](https://developers.cloudflare.com/api/resources/diagnostics/subresources/endpoint-healthchecks/methods/create/) | [`healthCheckMutationPath`](../src/routes.zig#L8262) |
| `DELETE /accounts/{account_id}/diagnostics/endpoint-healthchecks/{id}` | [Delete Endpoint Health Check](https://developers.cloudflare.com/api/resources/diagnostics/subresources/endpoint-healthchecks/methods/delete/) | [`healthCheckMutationPath`](../src/routes.zig#L8262) |
| `GET /accounts/{account_id}/diagnostics/endpoint-healthchecks/{id}` | [Get Endpoint Health Check](https://developers.cloudflare.com/api/resources/diagnostics/subresources/endpoint-healthchecks/methods/get/) | [`endpointHealthCheckReadPath`](../src/routes.zig#L8174) |
| `PUT /accounts/{account_id}/diagnostics/endpoint-healthchecks/{id}` | [Update Endpoint Health Check](https://developers.cloudflare.com/api/resources/diagnostics/subresources/endpoint-healthchecks/methods/update/) | [`healthCheckMutationPath`](../src/routes.zig#L8262) |
| `GET /accounts/{account_id}/load_balancers/monitor_groups` | [List Monitor Groups](https://developers.cloudflare.com/api/resources/load_balancers/subresources/monitor_groups/methods/list/) | [`loadBalancingAccountReadPath`](../src/routes.zig#L7972) |
| `POST /accounts/{account_id}/load_balancers/monitor_groups` | [Create Monitor Group](https://developers.cloudflare.com/api/resources/load_balancers/subresources/monitor_groups/methods/create/) | [`loadBalancingMutationPath`](../src/routes.zig#L8116) |
| `DELETE /accounts/{account_id}/load_balancers/monitor_groups/{monitor_group_id}` | [Delete Monitor Group](https://developers.cloudflare.com/api/resources/load_balancers/subresources/monitor_groups/methods/delete/) | [`loadBalancingMutationPath`](../src/routes.zig#L8116) |
| `GET /accounts/{account_id}/load_balancers/monitor_groups/{monitor_group_id}` | [Monitor Group Details](https://developers.cloudflare.com/api/resources/load_balancers/subresources/monitor_groups/methods/get/) | [`loadBalancingAccountReadPath`](../src/routes.zig#L7972) |
| `PATCH /accounts/{account_id}/load_balancers/monitor_groups/{monitor_group_id}` | [Patch Monitor Group](https://developers.cloudflare.com/api/resources/load_balancers/subresources/monitor_groups/methods/edit/) | [`loadBalancingMutationPath`](../src/routes.zig#L8116) |
| `PUT /accounts/{account_id}/load_balancers/monitor_groups/{monitor_group_id}` | [Update Monitor Group](https://developers.cloudflare.com/api/resources/load_balancers/subresources/monitor_groups/methods/update/) | [`loadBalancingMutationPath`](../src/routes.zig#L8116) |
| `GET /accounts/{account_id}/load_balancers/monitor_groups/{monitor_group_id}/references` | [List Monitor Group References](https://developers.cloudflare.com/api/resources/load_balancers/subresources/monitor_groups/subresources/references/methods/get/) | [`loadBalancingAccountReadPath`](../src/routes.zig#L7972) |
| `GET /accounts/{account_id}/load_balancers/monitors` | [List Monitors](https://developers.cloudflare.com/api/resources/load_balancers/subresources/monitors/methods/list/) | [`loadBalancingAccountReadPath`](../src/routes.zig#L7972) |
| `POST /accounts/{account_id}/load_balancers/monitors` | [Create Monitor](https://developers.cloudflare.com/api/resources/load_balancers/subresources/monitors/methods/create/) | [`loadBalancingMutationPath`](../src/routes.zig#L8116) |
| `DELETE /accounts/{account_id}/load_balancers/monitors/{monitor_id}` | [Delete Monitor](https://developers.cloudflare.com/api/resources/load_balancers/subresources/monitors/methods/delete/) | [`loadBalancingMutationPath`](../src/routes.zig#L8116) |
| `GET /accounts/{account_id}/load_balancers/monitors/{monitor_id}` | [Monitor Details](https://developers.cloudflare.com/api/resources/load_balancers/subresources/monitors/methods/get/) | [`loadBalancingAccountReadPath`](../src/routes.zig#L7972) |
| `PATCH /accounts/{account_id}/load_balancers/monitors/{monitor_id}` | [Patch Monitor](https://developers.cloudflare.com/api/resources/load_balancers/subresources/monitors/methods/edit/) | [`loadBalancingMutationPath`](../src/routes.zig#L8116) |
| `PUT /accounts/{account_id}/load_balancers/monitors/{monitor_id}` | [Update Monitor](https://developers.cloudflare.com/api/resources/load_balancers/subresources/monitors/methods/update/) | [`loadBalancingMutationPath`](../src/routes.zig#L8116) |
| `POST /accounts/{account_id}/load_balancers/monitors/{monitor_id}/preview` | [Preview Monitor](https://developers.cloudflare.com/api/resources/load_balancers/subresources/monitors/subresources/previews/methods/create/) | [`loadBalancingMutationPath`](../src/routes.zig#L8116) |
| `GET /accounts/{account_id}/load_balancers/monitors/{monitor_id}/references` | [List Monitor References](https://developers.cloudflare.com/api/resources/load_balancers/subresources/monitors/subresources/references/methods/get/) | [`loadBalancingAccountReadPath`](../src/routes.zig#L7972) |
| `GET /accounts/{account_id}/load_balancers/pools` | [List Pools](https://developers.cloudflare.com/api/resources/load_balancers/subresources/pools/methods/list/) | [`loadBalancingAccountReadPath`](../src/routes.zig#L7972) |
| `PATCH /accounts/{account_id}/load_balancers/pools` | [Patch Pools](https://developers.cloudflare.com/api/resources/load_balancers/subresources/pools/methods/bulk_edit/) | [`loadBalancingMutationPath`](../src/routes.zig#L8116) |
| `POST /accounts/{account_id}/load_balancers/pools` | [Create Pool](https://developers.cloudflare.com/api/resources/load_balancers/subresources/pools/methods/create/) | [`loadBalancingMutationPath`](../src/routes.zig#L8116) |
| `DELETE /accounts/{account_id}/load_balancers/pools/{pool_id}` | [Delete Pool](https://developers.cloudflare.com/api/resources/load_balancers/subresources/pools/methods/delete/) | [`loadBalancingMutationPath`](../src/routes.zig#L8116) |
| `GET /accounts/{account_id}/load_balancers/pools/{pool_id}` | [Pool Details](https://developers.cloudflare.com/api/resources/load_balancers/subresources/pools/methods/get/) | [`loadBalancingAccountReadPath`](../src/routes.zig#L7972) |
| `PATCH /accounts/{account_id}/load_balancers/pools/{pool_id}` | [Patch Pool](https://developers.cloudflare.com/api/resources/load_balancers/subresources/pools/methods/edit/) | [`loadBalancingMutationPath`](../src/routes.zig#L8116) |
| `PUT /accounts/{account_id}/load_balancers/pools/{pool_id}` | [Update Pool](https://developers.cloudflare.com/api/resources/load_balancers/subresources/pools/methods/update/) | [`loadBalancingMutationPath`](../src/routes.zig#L8116) |
| `GET /accounts/{account_id}/load_balancers/pools/{pool_id}/health` | [Pool Health Details](https://developers.cloudflare.com/api/resources/load_balancers/subresources/pools/subresources/health/methods/get/) | [`loadBalancingAccountReadPath`](../src/routes.zig#L7972) |
| `POST /accounts/{account_id}/load_balancers/pools/{pool_id}/preview` | [Preview Pool](https://developers.cloudflare.com/api/resources/load_balancers/subresources/pools/subresources/health/methods/create/) | [`loadBalancingMutationPath`](../src/routes.zig#L8116) |
| `GET /accounts/{account_id}/load_balancers/pools/{pool_id}/references` | [List Pool References](https://developers.cloudflare.com/api/resources/load_balancers/subresources/pools/subresources/references/methods/get/) | [`loadBalancingAccountReadPath`](../src/routes.zig#L7972) |
| `GET /accounts/{account_id}/load_balancers/preview/{preview_id}` | [Preview Result](https://developers.cloudflare.com/api/resources/load_balancers/subresources/previews/methods/get/) | [`loadBalancingAccountReadPath`](../src/routes.zig#L7972) |
| `GET /accounts/{account_id}/load_balancers/regions` | [List Regions](https://developers.cloudflare.com/api/resources/load_balancers/subresources/regions/methods/list/) | [`loadBalancingAccountReadPath`](../src/routes.zig#L7972) |
| `GET /accounts/{account_id}/load_balancers/regions/{region_id}` | [Get Region](https://developers.cloudflare.com/api/resources/load_balancers/subresources/regions/methods/get/) | [`loadBalancingAccountReadPath`](../src/routes.zig#L7972) |
| `GET /accounts/{account_id}/load_balancers/search` | [Search Resources](https://developers.cloudflare.com/api/resources/load_balancers/subresources/searches/methods/list/) | [`loadBalancingAccountReadPath`](../src/routes.zig#L7972) |
| `GET /user/load_balancers/monitors` | List Monitors — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`loadBalancingUserReadPath`](../src/routes.zig#L8049) |
| `POST /user/load_balancers/monitors` | Create Monitor — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`loadBalancingMutationPath`](../src/routes.zig#L8116) |
| `DELETE /user/load_balancers/monitors/{monitor_id}` | Delete Monitor — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`loadBalancingMutationPath`](../src/routes.zig#L8116) |
| `GET /user/load_balancers/monitors/{monitor_id}` | Monitor Details — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`loadBalancingUserReadPath`](../src/routes.zig#L8049) |
| `PATCH /user/load_balancers/monitors/{monitor_id}` | Patch Monitor — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`loadBalancingMutationPath`](../src/routes.zig#L8116) |
| `PUT /user/load_balancers/monitors/{monitor_id}` | Update Monitor — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`loadBalancingMutationPath`](../src/routes.zig#L8116) |
| `POST /user/load_balancers/monitors/{monitor_id}/preview` | Preview Monitor — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`loadBalancingMutationPath`](../src/routes.zig#L8116) |
| `GET /user/load_balancers/monitors/{monitor_id}/references` | List Monitor References — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`loadBalancingUserReadPath`](../src/routes.zig#L8049) |
| `GET /user/load_balancers/pools` | List Pools — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`loadBalancingUserReadPath`](../src/routes.zig#L8049) |
| `PATCH /user/load_balancers/pools` | Patch Pools — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`loadBalancingMutationPath`](../src/routes.zig#L8116) |
| `POST /user/load_balancers/pools` | Create Pool — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`loadBalancingMutationPath`](../src/routes.zig#L8116) |
| `DELETE /user/load_balancers/pools/{pool_id}` | Delete Pool — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`loadBalancingMutationPath`](../src/routes.zig#L8116) |
| `GET /user/load_balancers/pools/{pool_id}` | Pool Details — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`loadBalancingUserReadPath`](../src/routes.zig#L8049) |
| `PATCH /user/load_balancers/pools/{pool_id}` | Patch Pool — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`loadBalancingMutationPath`](../src/routes.zig#L8116) |
| `PUT /user/load_balancers/pools/{pool_id}` | Update Pool — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`loadBalancingMutationPath`](../src/routes.zig#L8116) |
| `GET /user/load_balancers/pools/{pool_id}/health` | Pool Health Details — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`loadBalancingUserReadPath`](../src/routes.zig#L8049) |
| `POST /user/load_balancers/pools/{pool_id}/preview` | Preview Pool — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`loadBalancingMutationPath`](../src/routes.zig#L8116) |
| `GET /user/load_balancers/pools/{pool_id}/references` | List Pool References — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`loadBalancingUserReadPath`](../src/routes.zig#L8049) |
| `GET /user/load_balancers/preview/{preview_id}` | Preview Result — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`loadBalancingUserReadPath`](../src/routes.zig#L8049) |
| `GET /user/load_balancing_analytics/events` | List Healthcheck Events — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`loadBalancingUserReadPath`](../src/routes.zig#L8049) |
| `GET /zones/{zone_id}/healthchecks` | [List Health Checks](https://developers.cloudflare.com/api/resources/healthchecks/methods/list/) | [`zoneHealthCheckReadPath`](../src/routes.zig#L8218) |
| `POST /zones/{zone_id}/healthchecks` | [Create Health Check](https://developers.cloudflare.com/api/resources/healthchecks/methods/create/) | [`healthCheckMutationPath`](../src/routes.zig#L8262) |
| `POST /zones/{zone_id}/healthchecks/preview` | [Create Preview Health Check](https://developers.cloudflare.com/api/resources/healthchecks/subresources/previews/methods/create/) | [`healthCheckMutationPath`](../src/routes.zig#L8262) |
| `DELETE /zones/{zone_id}/healthchecks/preview/{healthcheck_id}` | [Delete Preview Health Check](https://developers.cloudflare.com/api/resources/healthchecks/subresources/previews/methods/delete/) | [`healthCheckMutationPath`](../src/routes.zig#L8262) |
| `GET /zones/{zone_id}/healthchecks/preview/{healthcheck_id}` | [Health Check Preview Details](https://developers.cloudflare.com/api/resources/healthchecks/subresources/previews/methods/get/) | [`zoneHealthCheckReadPath`](../src/routes.zig#L8218) |
| `DELETE /zones/{zone_id}/healthchecks/{healthcheck_id}` | [Delete Health Check](https://developers.cloudflare.com/api/resources/healthchecks/methods/delete/) | [`healthCheckMutationPath`](../src/routes.zig#L8262) |
| `GET /zones/{zone_id}/healthchecks/{healthcheck_id}` | [Health Check Details](https://developers.cloudflare.com/api/resources/healthchecks/methods/get/) | [`zoneHealthCheckReadPath`](../src/routes.zig#L8218) |
| `PATCH /zones/{zone_id}/healthchecks/{healthcheck_id}` | [Patch Health Check](https://developers.cloudflare.com/api/resources/healthchecks/methods/edit/) | [`healthCheckMutationPath`](../src/routes.zig#L8262) |
| `PUT /zones/{zone_id}/healthchecks/{healthcheck_id}` | [Update Health Check](https://developers.cloudflare.com/api/resources/healthchecks/methods/update/) | [`healthCheckMutationPath`](../src/routes.zig#L8262) |
| `GET /zones/{zone_id}/load_balancers` | [List Load Balancers](https://developers.cloudflare.com/api/resources/load_balancers/methods/list/) | [`loadBalancingZoneReadPath`](../src/routes.zig#L8106) |
| `POST /zones/{zone_id}/load_balancers` | [Create Load Balancer](https://developers.cloudflare.com/api/resources/load_balancers/methods/create/) | [`loadBalancingMutationPath`](../src/routes.zig#L8116) |
| `DELETE /zones/{zone_id}/load_balancers/{load_balancer_id}` | [Delete Load Balancer](https://developers.cloudflare.com/api/resources/load_balancers/methods/delete/) | [`loadBalancingMutationPath`](../src/routes.zig#L8116) |
| `GET /zones/{zone_id}/load_balancers/{load_balancer_id}` | [Load Balancer Details](https://developers.cloudflare.com/api/resources/load_balancers/methods/get/) | [`loadBalancingZoneReadPath`](../src/routes.zig#L8106) |
| `PATCH /zones/{zone_id}/load_balancers/{load_balancer_id}` | [Patch Load Balancer](https://developers.cloudflare.com/api/resources/load_balancers/methods/edit/) | [`loadBalancingMutationPath`](../src/routes.zig#L8116) |
| `PUT /zones/{zone_id}/load_balancers/{load_balancer_id}` | [Update Load Balancer](https://developers.cloudflare.com/api/resources/load_balancers/methods/update/) | [`loadBalancingMutationPath`](../src/routes.zig#L8116) |
| `GET /zones/{zone_id}/smart_shield/healthchecks` | [List Health Checks](https://developers.cloudflare.com/api/resources/smart_shield/subresources/health_checks/methods/list/) | [`smartShieldHealthCheckReadPath`](../src/routes.zig#L8252) |
| `POST /zones/{zone_id}/smart_shield/healthchecks` | [Create Health Check](https://developers.cloudflare.com/api/resources/smart_shield/subresources/health_checks/methods/create/) | [`healthCheckMutationPath`](../src/routes.zig#L8262) |
| `DELETE /zones/{zone_id}/smart_shield/healthchecks/{healthcheck_id}` | [Delete Health Check](https://developers.cloudflare.com/api/resources/smart_shield/subresources/health_checks/methods/delete/) | [`healthCheckMutationPath`](../src/routes.zig#L8262) |
| `GET /zones/{zone_id}/smart_shield/healthchecks/{healthcheck_id}` | [Health Check Details](https://developers.cloudflare.com/api/resources/smart_shield/subresources/health_checks/methods/get/) | [`smartShieldHealthCheckReadPath`](../src/routes.zig#L8252) |
| `PATCH /zones/{zone_id}/smart_shield/healthchecks/{healthcheck_id}` | [Patch Health Check](https://developers.cloudflare.com/api/resources/smart_shield/subresources/health_checks/methods/edit/) | [`healthCheckMutationPath`](../src/routes.zig#L8262) |
| `PUT /zones/{zone_id}/smart_shield/healthchecks/{healthcheck_id}` | [Update Health Check](https://developers.cloudflare.com/api/resources/smart_shield/subresources/health_checks/methods/update/) | [`healthCheckMutationPath`](../src/routes.zig#L8262) |

## Website security and rules

Read the current rules or security posture before preparing a change. Rulesets, IP access rules, Page Shield, and API Shield address different kinds of traffic or application risk.

| HTTP method and path | What it does / official reference | Zig route helper |
| --- | --- | --- |
| `DELETE /accounts/{account_id}/cloudforce-one/rules` | Delete all rules — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`cloudforceOneRuleMutationPath`](../src/routes.zig#L8470) |
| `GET /accounts/{account_id}/cloudforce-one/rules` | List rules — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`cloudforceOneRuleReadPath`](../src/routes.zig#L8446) |
| `POST /accounts/{account_id}/cloudforce-one/rules` | Create a rule — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`cloudforceOneRuleMutationPath`](../src/routes.zig#L8470) |
| `GET /accounts/{account_id}/cloudforce-one/rules/managed` | Get managed rules — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`cloudforceOneRuleReadPath`](../src/routes.zig#L8446) |
| `GET /accounts/{account_id}/cloudforce-one/rules/search` | Search rules — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`cloudforceOneRuleReadPath`](../src/routes.zig#L8446) |
| `GET /accounts/{account_id}/cloudforce-one/rules/stats` | Get dashboard stats — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`cloudforceOneRuleReadPath`](../src/routes.zig#L8446) |
| `GET /accounts/{account_id}/cloudforce-one/rules/tree` | Get folder tree structure — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`cloudforceOneRuleReadPath`](../src/routes.zig#L8446) |
| `POST /accounts/{account_id}/cloudforce-one/rules/validate` | Validate rule with context — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`cloudforceOneRuleMutationPath`](../src/routes.zig#L8470) |
| `DELETE /accounts/{account_id}/cloudforce-one/rules/{id}` | Delete a rule — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`cloudforceOneRuleMutationPath`](../src/routes.zig#L8470) |
| `GET /accounts/{account_id}/cloudforce-one/rules/{id}` | Get a rule — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`cloudforceOneRuleReadPath`](../src/routes.zig#L8446) |
| `PUT /accounts/{account_id}/cloudforce-one/rules/{id}` | Update a rule — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`cloudforceOneRuleMutationPath`](../src/routes.zig#L8470) |
| `GET /accounts/{account_id}/custom_pages` | List custom pages — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`customPageReadPath`](../src/routes.zig#L8904) |
| `GET /accounts/{account_id}/custom_pages/assets` | List custom assets — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`customPageReadPath`](../src/routes.zig#L8904) |
| `POST /accounts/{account_id}/custom_pages/assets` | Create a custom asset — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`customPageMutationPath`](../src/routes.zig#L8914) |
| `DELETE /accounts/{account_id}/custom_pages/assets/{asset_name}` | Delete a custom asset — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`customPageMutationPath`](../src/routes.zig#L8914) |
| `GET /accounts/{account_id}/custom_pages/assets/{asset_name}` | Get a custom asset — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`customPageReadPath`](../src/routes.zig#L8904) |
| `PUT /accounts/{account_id}/custom_pages/assets/{asset_name}` | Update a custom asset — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`customPageMutationPath`](../src/routes.zig#L8914) |
| `POST /accounts/{account_id}/custom_pages/preview_tokens` | Create a preview token — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`customPageMutationPath`](../src/routes.zig#L8914) |
| `GET /accounts/{account_id}/custom_pages/{identifier}` | Get a custom page — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`customPageReadPath`](../src/routes.zig#L8904) |
| `PUT /accounts/{account_id}/custom_pages/{identifier}` | Update a custom page — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`customPageMutationPath`](../src/routes.zig#L8914) |
| `GET /accounts/{account_id}/firewall/access_rules/rules` | List IP Access rules — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`ipAccessRuleReadPath`](../src/routes.zig#L8513) |
| `POST /accounts/{account_id}/firewall/access_rules/rules` | Create an IP Access rule — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`ipAccessRuleMutationPath`](../src/routes.zig#L8528) |
| `DELETE /accounts/{account_id}/firewall/access_rules/rules/{rule_id}` | Delete an IP Access rule — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`ipAccessRuleMutationPath`](../src/routes.zig#L8528) |
| `GET /accounts/{account_id}/firewall/access_rules/rules/{rule_id}` | Get an IP Access rule — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`ipAccessRuleReadPath`](../src/routes.zig#L8513) |
| `PATCH /accounts/{account_id}/firewall/access_rules/rules/{rule_id}` | Update an IP Access rule — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`ipAccessRuleMutationPath`](../src/routes.zig#L8528) |
| `GET /accounts/{account_id}/rulesets` | [List account rulesets](https://developers.cloudflare.com/api/resources/rulesets/methods/list/) | [`rulesetReadPath`](../src/routes.zig#L8335) |
| `POST /accounts/{account_id}/rulesets` | [Create an account ruleset](https://developers.cloudflare.com/api/resources/rulesets/methods/create/) | [`rulesetMutationPath`](../src/routes.zig#L8385) |
| `GET /accounts/{account_id}/rulesets/phases/{ruleset_phase}/entrypoint` | [Get an account entry point ruleset](https://developers.cloudflare.com/api/resources/rulesets/subresources/phases/methods/get/) | [`rulesetReadPath`](../src/routes.zig#L8335) |
| `PUT /accounts/{account_id}/rulesets/phases/{ruleset_phase}/entrypoint` | Update an account entry point ruleset — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`rulesetMutationPath`](../src/routes.zig#L8385) |
| `GET /accounts/{account_id}/rulesets/phases/{ruleset_phase}/entrypoint/versions` | List an account entry point ruleset's versions — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`rulesetReadPath`](../src/routes.zig#L8335) |
| `GET /accounts/{account_id}/rulesets/phases/{ruleset_phase}/entrypoint/versions/{ruleset_version}` | [Get an account entry point ruleset version](https://developers.cloudflare.com/api/resources/rulesets/subresources/phases/subresources/versions/methods/get/) | [`rulesetReadPath`](../src/routes.zig#L8335) |
| `DELETE /accounts/{account_id}/rulesets/{ruleset_id}` | [Delete an account ruleset](https://developers.cloudflare.com/api/resources/rulesets/methods/delete/) | [`rulesetMutationPath`](../src/routes.zig#L8385) |
| `GET /accounts/{account_id}/rulesets/{ruleset_id}` | [Get an account ruleset](https://developers.cloudflare.com/api/resources/rulesets/methods/get/) | [`rulesetReadPath`](../src/routes.zig#L8335) |
| `PUT /accounts/{account_id}/rulesets/{ruleset_id}` | Update an account ruleset — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`rulesetMutationPath`](../src/routes.zig#L8385) |
| `POST /accounts/{account_id}/rulesets/{ruleset_id}/rules` | [Create an account ruleset rule](https://developers.cloudflare.com/api/resources/rulesets/subresources/rules/methods/create/) | [`rulesetMutationPath`](../src/routes.zig#L8385) |
| `DELETE /accounts/{account_id}/rulesets/{ruleset_id}/rules/{rule_id}` | [Delete an account ruleset rule](https://developers.cloudflare.com/api/resources/rulesets/subresources/rules/methods/delete/) | [`rulesetMutationPath`](../src/routes.zig#L8385) |
| `PATCH /accounts/{account_id}/rulesets/{ruleset_id}/rules/{rule_id}` | [Update an account ruleset rule](https://developers.cloudflare.com/api/resources/rulesets/subresources/rules/methods/edit/) | [`rulesetMutationPath`](../src/routes.zig#L8385) |
| `GET /accounts/{account_id}/rulesets/{ruleset_id}/versions` | List an account ruleset's versions — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`rulesetReadPath`](../src/routes.zig#L8335) |
| `DELETE /accounts/{account_id}/rulesets/{ruleset_id}/versions/{ruleset_version}` | Delete an account ruleset version — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`rulesetMutationPath`](../src/routes.zig#L8385) |
| `GET /accounts/{account_id}/rulesets/{ruleset_id}/versions/{ruleset_version}` | [Get an account ruleset version](https://developers.cloudflare.com/api/resources/rulesets/subresources/versions/methods/get/) | [`rulesetReadPath`](../src/routes.zig#L8335) |
| `GET /accounts/{account_id}/rulesets/{ruleset_id}/versions/{ruleset_version}/by_tag/{rule_tag}` | List an account ruleset version's rules by tag — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`rulesetReadPath`](../src/routes.zig#L8335) |
| `GET /user/firewall/access_rules/rules` | List IP Access rules — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`ipAccessRuleReadPath`](../src/routes.zig#L8513) |
| `POST /user/firewall/access_rules/rules` | Create an IP Access rule — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`ipAccessRuleMutationPath`](../src/routes.zig#L8528) |
| `DELETE /user/firewall/access_rules/rules/{rule_id}` | Delete an IP Access rule — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`ipAccessRuleMutationPath`](../src/routes.zig#L8528) |
| `GET /user/firewall/access_rules/rules/{rule_id}` | Get an IP Access rule — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`ipAccessRuleReadPath`](../src/routes.zig#L8513) |
| `PATCH /user/firewall/access_rules/rules/{rule_id}` | Update an IP Access rule — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`ipAccessRuleMutationPath`](../src/routes.zig#L8528) |
| `GET /zones/{zone_id}/ai-security/custom-topics` | [Get the AI Security for Apps custom topics of a zone](https://developers.cloudflare.com/api/resources/ai_security/subresources/custom_topics/methods/get/) | [`zoneSecurityPostureReadPath`](../src/routes.zig#L8683) |
| `GET /zones/{zone_id}/ai-security/settings` | [Get the AI Security for Apps status for a zone](https://developers.cloudflare.com/api/resources/ai_security/methods/get/) | [`zoneSecurityPostureReadPath`](../src/routes.zig#L8683) |
| `GET /zones/{zone_id}/api_gateway/configuration` | [Get session identifier settings](https://developers.cloudflare.com/api/resources/api_gateway/subresources/configurations/methods/get/) | [`apiShieldReadPath`](../src/routes.zig#L8632) |
| `GET /zones/{zone_id}/api_gateway/discovery` | [Export discovered API operations as OpenAPI schemas](https://developers.cloudflare.com/api/resources/api_gateway/subresources/discovery/methods/get/) | [`apiShieldReadPath`](../src/routes.zig#L8632) |
| `GET /zones/{zone_id}/api_gateway/discovery/operations` | [List discovered web and API operations](https://developers.cloudflare.com/api/resources/api_gateway/subresources/discovery/subresources/operations/methods/list/) | [`apiShieldReadPath`](../src/routes.zig#L8632) |
| `GET /zones/{zone_id}/api_gateway/discovery/operations/{discovery_id}` | Get a discovered web or API operation — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`apiShieldReadPath`](../src/routes.zig#L8632) |
| `GET /zones/{zone_id}/api_gateway/labels` | [List operation labels](https://developers.cloudflare.com/api/resources/api_gateway/subresources/labels/methods/list/) | [`apiShieldReadPath`](../src/routes.zig#L8632) |
| `GET /zones/{zone_id}/api_gateway/labels/managed/{name}` | [Get a managed operation label](https://developers.cloudflare.com/api/resources/api_gateway/subresources/labels/subresources/managed/methods/get/) | [`apiShieldReadPath`](../src/routes.zig#L8632) |
| `GET /zones/{zone_id}/api_gateway/labels/user/{name}` | [Get a user-defined operation label](https://developers.cloudflare.com/api/resources/api_gateway/subresources/labels/subresources/user/methods/get/) | [`apiShieldReadPath`](../src/routes.zig#L8632) |
| `GET /zones/{zone_id}/api_gateway/operations` | [List web and API operations](https://developers.cloudflare.com/api/resources/api_gateway/subresources/operations/methods/list/) | [`apiShieldReadPath`](../src/routes.zig#L8632) |
| `GET /zones/{zone_id}/api_gateway/operations/{operation_id}` | [Get a web or API operation](https://developers.cloudflare.com/api/resources/api_gateway/subresources/operations/methods/get/) | [`apiShieldReadPath`](../src/routes.zig#L8632) |
| `GET /zones/{zone_id}/api_gateway/schemas` | [Export web and API operations as OpenAPI schemas](https://developers.cloudflare.com/api/resources/api_gateway/subresources/schemas/methods/list/) | [`apiShieldReadPath`](../src/routes.zig#L8632) |
| `GET /zones/{zone_id}/bot_management` | [Get Zone Bot Management Config](https://developers.cloudflare.com/api/resources/bot_management/methods/get/) | [`zoneSecurityPostureReadPath`](../src/routes.zig#L8683) |
| `GET /zones/{zone_id}/certificate_authorities/hostname_associations` | [List Hostname Associations](https://developers.cloudflare.com/api/resources/certificate_authorities/subresources/hostname_associations/methods/get/) | [`apiShieldReadPath`](../src/routes.zig#L8632) |
| `GET /zones/{zone_id}/client_certificates` | [List Client Certificates](https://developers.cloudflare.com/api/resources/client_certificates/methods/list/) | [`apiShieldReadPath`](../src/routes.zig#L8632) |
| `GET /zones/{zone_id}/client_certificates/{client_certificate_id}` | [Client Certificate Details](https://developers.cloudflare.com/api/resources/client_certificates/methods/get/) | [`apiShieldReadPath`](../src/routes.zig#L8632) |
| `GET /zones/{zone_id}/content-upload-scan/payloads` | [List the Content Scanning custom expressions of a zone](https://developers.cloudflare.com/api/resources/content_scanning/subresources/payloads/methods/list/) | [`zoneSecurityPostureReadPath`](../src/routes.zig#L8683) |
| `GET /zones/{zone_id}/content-upload-scan/settings` | [Get the Content Scanning status for a zone](https://developers.cloudflare.com/api/resources/content_scanning/methods/get/) | [`zoneSecurityPostureReadPath`](../src/routes.zig#L8683) |
| `GET /zones/{zone_id}/ct/alerting` | [Get CT Alerting Subscription](https://developers.cloudflare.com/api/resources/zones/subresources/ct/subresources/alerting/methods/get/) | [`zoneSecurityPostureReadPath`](../src/routes.zig#L8683) |
| `GET /zones/{zone_id}/custom_pages` | List custom pages — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`customPageReadPath`](../src/routes.zig#L8904) |
| `GET /zones/{zone_id}/custom_pages/assets` | List custom assets — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`customPageReadPath`](../src/routes.zig#L8904) |
| `POST /zones/{zone_id}/custom_pages/assets` | Create a custom asset — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`customPageMutationPath`](../src/routes.zig#L8914) |
| `DELETE /zones/{zone_id}/custom_pages/assets/{asset_name}` | Delete a custom asset — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`customPageMutationPath`](../src/routes.zig#L8914) |
| `GET /zones/{zone_id}/custom_pages/assets/{asset_name}` | Get a custom asset — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`customPageReadPath`](../src/routes.zig#L8904) |
| `PUT /zones/{zone_id}/custom_pages/assets/{asset_name}` | Update a custom asset — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`customPageMutationPath`](../src/routes.zig#L8914) |
| `POST /zones/{zone_id}/custom_pages/preview_tokens` | Create a preview token — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`customPageMutationPath`](../src/routes.zig#L8914) |
| `GET /zones/{zone_id}/custom_pages/{identifier}` | Get a custom page — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`customPageReadPath`](../src/routes.zig#L8904) |
| `PUT /zones/{zone_id}/custom_pages/{identifier}` | Update a custom page — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`customPageMutationPath`](../src/routes.zig#L8914) |
| `GET /zones/{zone_id}/firewall/access_rules/rules` | List IP Access rules — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`ipAccessRuleReadPath`](../src/routes.zig#L8513) |
| `POST /zones/{zone_id}/firewall/access_rules/rules` | Create an IP Access rule — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`ipAccessRuleMutationPath`](../src/routes.zig#L8528) |
| `DELETE /zones/{zone_id}/firewall/access_rules/rules/{rule_id}` | Delete an IP Access rule — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`ipAccessRuleMutationPath`](../src/routes.zig#L8528) |
| `PATCH /zones/{zone_id}/firewall/access_rules/rules/{rule_id}` | Update an IP Access rule — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`ipAccessRuleMutationPath`](../src/routes.zig#L8528) |
| `GET /zones/{zone_id}/firewall/lockdowns` | [List Zone Lockdown rules](https://developers.cloudflare.com/api/resources/firewall/subresources/lockdowns/methods/list/) | [`zoneLegacyRuleReadPath`](../src/routes.zig#L8564) |
| `POST /zones/{zone_id}/firewall/lockdowns` | [Create a Zone Lockdown rule](https://developers.cloudflare.com/api/resources/firewall/subresources/lockdowns/methods/create/) | [`zoneLegacyRuleMutationPath`](../src/routes.zig#L8574) |
| `DELETE /zones/{zone_id}/firewall/lockdowns/{lock_downs_id}` | [Delete a Zone Lockdown rule](https://developers.cloudflare.com/api/resources/firewall/subresources/lockdowns/methods/delete/) | [`zoneLegacyRuleMutationPath`](../src/routes.zig#L8574) |
| `GET /zones/{zone_id}/firewall/lockdowns/{lock_downs_id}` | [Get a Zone Lockdown rule](https://developers.cloudflare.com/api/resources/firewall/subresources/lockdowns/methods/get/) | [`zoneLegacyRuleReadPath`](../src/routes.zig#L8564) |
| `PUT /zones/{zone_id}/firewall/lockdowns/{lock_downs_id}` | [Update a Zone Lockdown rule](https://developers.cloudflare.com/api/resources/firewall/subresources/lockdowns/methods/update/) | [`zoneLegacyRuleMutationPath`](../src/routes.zig#L8574) |
| `GET /zones/{zone_id}/firewall/ua_rules` | [List User Agent Blocking rules](https://developers.cloudflare.com/api/resources/firewall/subresources/ua_rules/methods/list/) | [`zoneLegacyRuleReadPath`](../src/routes.zig#L8564) |
| `POST /zones/{zone_id}/firewall/ua_rules` | [Create a User Agent Blocking rule](https://developers.cloudflare.com/api/resources/firewall/subresources/ua_rules/methods/create/) | [`zoneLegacyRuleMutationPath`](../src/routes.zig#L8574) |
| `DELETE /zones/{zone_id}/firewall/ua_rules/{ua_rule_id}` | [Delete a User Agent Blocking rule](https://developers.cloudflare.com/api/resources/firewall/subresources/ua_rules/methods/delete/) | [`zoneLegacyRuleMutationPath`](../src/routes.zig#L8574) |
| `GET /zones/{zone_id}/firewall/ua_rules/{ua_rule_id}` | [Get a User Agent Blocking rule](https://developers.cloudflare.com/api/resources/firewall/subresources/ua_rules/methods/get/) | [`zoneLegacyRuleReadPath`](../src/routes.zig#L8564) |
| `PUT /zones/{zone_id}/firewall/ua_rules/{ua_rule_id}` | [Update a User Agent Blocking rule](https://developers.cloudflare.com/api/resources/firewall/subresources/ua_rules/methods/update/) | [`zoneLegacyRuleMutationPath`](../src/routes.zig#L8574) |
| `GET /zones/{zone_id}/fraud_detection/settings` | [Get Fraud Detection Settings](https://developers.cloudflare.com/api/resources/fraud/methods/get/) | [`zoneSecurityPostureReadPath`](../src/routes.zig#L8683) |
| `GET /zones/{zone_id}/leaked-credential-checks` | [Get the Leaked Credential Checks status for a zone](https://developers.cloudflare.com/api/resources/leaked_credential_checks/methods/get/) | [`zoneSecurityPostureReadPath`](../src/routes.zig#L8683) |
| `GET /zones/{zone_id}/leaked-credential-checks/detections` | [List the custom detection locations of a zone](https://developers.cloudflare.com/api/resources/leaked_credential_checks/subresources/detections/methods/list/) | [`zoneSecurityPostureReadPath`](../src/routes.zig#L8683) |
| `GET /zones/{zone_id}/leaked-credential-checks/detections/{detection_id}` | [Get a custom detection location of a zone](https://developers.cloudflare.com/api/resources/leaked_credential_checks/subresources/detections/methods/get/) | [`zoneSecurityPostureReadPath`](../src/routes.zig#L8683) |
| `GET /zones/{zone_id}/page_shield` | [Get client-side security settings](https://developers.cloudflare.com/api/resources/page_shield/methods/get/) | [`pageShieldReadPath`](../src/routes.zig#L8611) |
| `PUT /zones/{zone_id}/page_shield` | [Update client-side security settings](https://developers.cloudflare.com/api/resources/page_shield/methods/update/) | [`pageShieldMutationPath`](../src/routes.zig#L8863) |
| `GET /zones/{zone_id}/page_shield/connections` | [List detected connections](https://developers.cloudflare.com/api/resources/page_shield/subresources/connections/methods/list/) | [`pageShieldReadPath`](../src/routes.zig#L8611) |
| `GET /zones/{zone_id}/page_shield/connections/{connection_id}` | [Get a detected connection](https://developers.cloudflare.com/api/resources/page_shield/subresources/connections/methods/get/) | [`pageShieldReadPath`](../src/routes.zig#L8611) |
| `GET /zones/{zone_id}/page_shield/cookies` | [List detected cookies](https://developers.cloudflare.com/api/resources/page_shield/subresources/cookies/methods/list/) | [`pageShieldReadPath`](../src/routes.zig#L8611) |
| `GET /zones/{zone_id}/page_shield/cookies/{cookie_id}` | [Get a detected cookie](https://developers.cloudflare.com/api/resources/page_shield/subresources/cookies/methods/get/) | [`pageShieldReadPath`](../src/routes.zig#L8611) |
| `GET /zones/{zone_id}/page_shield/policies` | [List content security rules](https://developers.cloudflare.com/api/resources/page_shield/subresources/policies/methods/list/) | [`pageShieldReadPath`](../src/routes.zig#L8611) |
| `POST /zones/{zone_id}/page_shield/policies` | [Create a content security rule](https://developers.cloudflare.com/api/resources/page_shield/subresources/policies/methods/create/) | [`pageShieldMutationPath`](../src/routes.zig#L8863) |
| `DELETE /zones/{zone_id}/page_shield/policies/{policy_id}` | [Delete a content security rule](https://developers.cloudflare.com/api/resources/page_shield/subresources/policies/methods/delete/) | [`pageShieldMutationPath`](../src/routes.zig#L8863) |
| `GET /zones/{zone_id}/page_shield/policies/{policy_id}` | [Get a content security rule](https://developers.cloudflare.com/api/resources/page_shield/subresources/policies/methods/get/) | [`pageShieldReadPath`](../src/routes.zig#L8611) |
| `PUT /zones/{zone_id}/page_shield/policies/{policy_id}` | [Update a content security rule](https://developers.cloudflare.com/api/resources/page_shield/subresources/policies/methods/update/) | [`pageShieldMutationPath`](../src/routes.zig#L8863) |
| `GET /zones/{zone_id}/page_shield/scripts` | [List detected scripts](https://developers.cloudflare.com/api/resources/page_shield/subresources/scripts/methods/list/) | [`pageShieldReadPath`](../src/routes.zig#L8611) |
| `GET /zones/{zone_id}/page_shield/scripts/{script_id}` | [Get a detected script](https://developers.cloudflare.com/api/resources/page_shield/subresources/scripts/methods/get/) | [`pageShieldReadPath`](../src/routes.zig#L8611) |
| `GET /zones/{zone_id}/pagerules` | [List Page Rules](https://developers.cloudflare.com/api/resources/page_rules/methods/list/) | [`zoneLegacyRuleReadPath`](../src/routes.zig#L8564) |
| `POST /zones/{zone_id}/pagerules` | [Create a Page Rule](https://developers.cloudflare.com/api/resources/page_rules/methods/create/) | [`zoneLegacyRuleMutationPath`](../src/routes.zig#L8574) |
| `DELETE /zones/{zone_id}/pagerules/{pagerule_id}` | [Delete a Page Rule](https://developers.cloudflare.com/api/resources/page_rules/methods/delete/) | [`zoneLegacyRuleMutationPath`](../src/routes.zig#L8574) |
| `GET /zones/{zone_id}/pagerules/{pagerule_id}` | [Get a Page Rule](https://developers.cloudflare.com/api/resources/page_rules/methods/get/) | [`zoneLegacyRuleReadPath`](../src/routes.zig#L8564) |
| `PATCH /zones/{zone_id}/pagerules/{pagerule_id}` | [Edit a Page Rule](https://developers.cloudflare.com/api/resources/page_rules/methods/edit/) | [`zoneLegacyRuleMutationPath`](../src/routes.zig#L8574) |
| `PUT /zones/{zone_id}/pagerules/{pagerule_id}` | [Update a Page Rule](https://developers.cloudflare.com/api/resources/page_rules/methods/update/) | [`zoneLegacyRuleMutationPath`](../src/routes.zig#L8574) |
| `GET /zones/{zone_id}/rulesets` | [List zone rulesets](https://developers.cloudflare.com/api/resources/rulesets/methods/list/) | [`rulesetReadPath`](../src/routes.zig#L8335) |
| `POST /zones/{zone_id}/rulesets` | [Create a zone ruleset](https://developers.cloudflare.com/api/resources/rulesets/methods/create/) | [`rulesetMutationPath`](../src/routes.zig#L8385) |
| `GET /zones/{zone_id}/rulesets/phases/{ruleset_phase}/entrypoint` | [Get a zone entry point ruleset](https://developers.cloudflare.com/api/resources/rulesets/subresources/phases/methods/get/) | [`rulesetReadPath`](../src/routes.zig#L8335) |
| `PUT /zones/{zone_id}/rulesets/phases/{ruleset_phase}/entrypoint` | [Update a zone entry point ruleset](https://developers.cloudflare.com/api/resources/rulesets/subresources/phases/methods/update/) | [`rulesetMutationPath`](../src/routes.zig#L8385) |
| `GET /zones/{zone_id}/rulesets/phases/{ruleset_phase}/entrypoint/versions` | [List a zone entry point ruleset's versions](https://developers.cloudflare.com/api/resources/rulesets/subresources/phases/subresources/versions/methods/list/) | [`rulesetReadPath`](../src/routes.zig#L8335) |
| `GET /zones/{zone_id}/rulesets/phases/{ruleset_phase}/entrypoint/versions/{ruleset_version}` | [Get a zone entry point ruleset version](https://developers.cloudflare.com/api/resources/rulesets/subresources/phases/subresources/versions/methods/get/) | [`rulesetReadPath`](../src/routes.zig#L8335) |
| `DELETE /zones/{zone_id}/rulesets/{ruleset_id}` | [Delete a zone ruleset](https://developers.cloudflare.com/api/resources/rulesets/methods/delete/) | [`rulesetMutationPath`](../src/routes.zig#L8385) |
| `GET /zones/{zone_id}/rulesets/{ruleset_id}` | [Get a zone ruleset](https://developers.cloudflare.com/api/resources/rulesets/methods/get/) | [`rulesetReadPath`](../src/routes.zig#L8335) |
| `PUT /zones/{zone_id}/rulesets/{ruleset_id}` | [Update a zone ruleset](https://developers.cloudflare.com/api/resources/rulesets/methods/update/) | [`rulesetMutationPath`](../src/routes.zig#L8385) |
| `POST /zones/{zone_id}/rulesets/{ruleset_id}/rules` | [Create a zone ruleset rule](https://developers.cloudflare.com/api/resources/rulesets/methods/create/) | [`rulesetMutationPath`](../src/routes.zig#L8385) |
| `DELETE /zones/{zone_id}/rulesets/{ruleset_id}/rules/{rule_id}` | [Delete a zone ruleset rule](https://developers.cloudflare.com/api/resources/rulesets/methods/delete/) | [`rulesetMutationPath`](../src/routes.zig#L8385) |
| `PATCH /zones/{zone_id}/rulesets/{ruleset_id}/rules/{rule_id}` | [Update a zone ruleset rule](https://developers.cloudflare.com/api/resources/rulesets/methods/update/) | [`rulesetMutationPath`](../src/routes.zig#L8385) |
| `GET /zones/{zone_id}/rulesets/{ruleset_id}/versions` | [List a zone ruleset's versions](https://developers.cloudflare.com/api/resources/rulesets/subresources/versions/methods/list/) | [`rulesetReadPath`](../src/routes.zig#L8335) |
| `DELETE /zones/{zone_id}/rulesets/{ruleset_id}/versions/{ruleset_version}` | [Delete a zone ruleset version](https://developers.cloudflare.com/api/resources/rulesets/subresources/versions/methods/delete/) | [`rulesetMutationPath`](../src/routes.zig#L8385) |
| `GET /zones/{zone_id}/rulesets/{ruleset_id}/versions/{ruleset_version}` | [Get a zone ruleset version](https://developers.cloudflare.com/api/resources/rulesets/subresources/versions/methods/get/) | [`rulesetReadPath`](../src/routes.zig#L8335) |
| `GET /zones/{zone_id}/rulesets/{ruleset_id}/versions/{ruleset_version}/by_tag/{rule_tag}` | List a zone ruleset version's rules by tag — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`rulesetReadPath`](../src/routes.zig#L8335) |
| `GET /zones/{zone_id}/settings/csam_scanner_third_party` | [Get CSAM Scanner setting](https://developers.cloudflare.com/api/resources/csam_scanner/methods/get/) | [`zoneSecurityPostureReadPath`](../src/routes.zig#L8683) |

## Email

Inspect email forwarding destinations and routing rules, domain authentication reports, and the supported sending or email-security settings.

| HTTP method and path | What it does / official reference | Zig route helper |
| --- | --- | --- |
| `GET /accounts/{account_id}/email-security/settings/allow_policies` | [List email allow policies](https://developers.cloudflare.com/api/resources/email_security/subresources/settings/subresources/allow_policies/methods/list/) | [`emailSecuritySettingsReadPath`](../src/routes.zig#L8837) |
| `GET /accounts/{account_id}/email-security/settings/allow_policies/{policy_id}` | [Get an email allow policy](https://developers.cloudflare.com/api/resources/email_security/subresources/settings/subresources/allow_policies/methods/get/) | [`emailSecuritySettingsReadPath`](../src/routes.zig#L8837) |
| `GET /accounts/{account_id}/email-security/settings/block_senders` | [List blocked email senders](https://developers.cloudflare.com/api/resources/email_security/subresources/settings/subresources/block_senders/methods/list/) | [`emailSecuritySettingsReadPath`](../src/routes.zig#L8837) |
| `GET /accounts/{account_id}/email-security/settings/block_senders/{pattern_id}` | [Get a blocked email sender](https://developers.cloudflare.com/api/resources/email_security/subresources/settings/subresources/block_senders/methods/get/) | [`emailSecuritySettingsReadPath`](../src/routes.zig#L8837) |
| `GET /accounts/{account_id}/email-security/settings/domains` | [List protected email domains](https://developers.cloudflare.com/api/resources/email_security/subresources/settings/subresources/domains/methods/list/) | [`emailSecuritySettingsReadPath`](../src/routes.zig#L8837) |
| `GET /accounts/{account_id}/email-security/settings/domains/{domain_id}` | [Get an email domain](https://developers.cloudflare.com/api/resources/email_security/subresources/settings/subresources/domains/methods/get/) | [`emailSecuritySettingsReadPath`](../src/routes.zig#L8837) |
| `GET /accounts/{account_id}/email-security/settings/impersonation_registry` | [List impersonation registry entries](https://developers.cloudflare.com/api/resources/email_security/subresources/settings/subresources/impersonation_registry/methods/list/) | [`emailSecuritySettingsReadPath`](../src/routes.zig#L8837) |
| `GET /accounts/{account_id}/email-security/settings/impersonation_registry/{impersonation_registry_id}` | [Get an impersonation registry entry](https://developers.cloudflare.com/api/resources/email_security/subresources/settings/subresources/impersonation_registry/methods/get/) | [`emailSecuritySettingsReadPath`](../src/routes.zig#L8837) |
| `GET /accounts/{account_id}/email-security/settings/sending_domain_restrictions` | [List sending domain restrictions](https://developers.cloudflare.com/api/resources/email_security/subresources/settings/subresources/sending_domain_restrictions/methods/list/) | [`emailSecuritySettingsReadPath`](../src/routes.zig#L8837) |
| `GET /accounts/{account_id}/email-security/settings/sending_domain_restrictions/{sending_domain_restriction_id}` | [Get a sending domain restriction](https://developers.cloudflare.com/api/resources/email_security/subresources/settings/subresources/sending_domain_restrictions/methods/get/) | [`emailSecuritySettingsReadPath`](../src/routes.zig#L8837) |
| `GET /accounts/{account_id}/email-security/settings/trusted_domains` | [List trusted email domains](https://developers.cloudflare.com/api/resources/email_security/subresources/settings/subresources/trusted_domains/methods/list/) | [`emailSecuritySettingsReadPath`](../src/routes.zig#L8837) |
| `GET /accounts/{account_id}/email-security/settings/trusted_domains/{trusted_domain_id}` | [Get a trusted email domain](https://developers.cloudflare.com/api/resources/email_security/subresources/settings/subresources/trusted_domains/methods/get/) | [`emailSecuritySettingsReadPath`](../src/routes.zig#L8837) |
| `GET /accounts/{account_id}/email-security/settings/url_ignore_patterns` | [List URL ignore patterns](https://developers.cloudflare.com/api/resources/email_security/subresources/settings/subresources/url_ignore_patterns/methods/list/) | [`emailSecuritySettingsReadPath`](../src/routes.zig#L8837) |
| `GET /accounts/{account_id}/email-security/settings/url_ignore_patterns/{pattern_id}` | [Get a URL ignore pattern](https://developers.cloudflare.com/api/resources/email_security/subresources/settings/subresources/url_ignore_patterns/methods/get/) | [`emailSecuritySettingsReadPath`](../src/routes.zig#L8837) |
| `GET /accounts/{account_id}/email/routing/addresses` | [List destination addresses](https://developers.cloudflare.com/api/resources/email_routing/subresources/addresses/methods/list/) | [`emailRoutingAccountReadPath`](../src/routes.zig#L8715) |
| `GET /accounts/{account_id}/email/routing/addresses/{destination_address_identifier}` | [Get a destination address](https://developers.cloudflare.com/api/resources/email_routing/subresources/addresses/methods/get/) | [`emailRoutingAccountReadPath`](../src/routes.zig#L8715) |
| `GET /accounts/{account_id}/email/sending/limits` | Get sending limits — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`emailSendingAccountReadPath`](../src/routes.zig#L8796) |
| `GET /zones/{zone_id}/email/auth/dmarc-reports` | [Get DMARC Report Status](https://developers.cloudflare.com/api/resources/email_auth/subresources/dmarc_reports/methods/get/) | [`emailAuthReadPath`](../src/routes.zig#L8772) |
| `GET /zones/{zone_id}/email/auth/spf/inspect` | [Inspect SPF Record](https://developers.cloudflare.com/api/resources/email_auth/subresources/spf/subresources/inspect/methods/get/) | [`emailAuthReadPath`](../src/routes.zig#L8772) |
| `GET /zones/{zone_id}/email/routing` | [Get Email Routing settings](https://developers.cloudflare.com/api/resources/email_routing/methods/get/) | [`emailRoutingZoneReadPath`](../src/routes.zig#L8737) |
| `GET /zones/{zone_id}/email/routing/dns` | [Email Routing - DNS settings](https://developers.cloudflare.com/api/resources/email_routing/subresources/dns/methods/get/) | [`emailRoutingZoneReadPath`](../src/routes.zig#L8737) |
| `GET /zones/{zone_id}/email/routing/rules` | [List routing rules](https://developers.cloudflare.com/api/resources/email_routing/subresources/rules/methods/list/) | [`emailRoutingZoneReadPath`](../src/routes.zig#L8737) |
| `GET /zones/{zone_id}/email/routing/rules/catch_all` | [Get catch-all rule](https://developers.cloudflare.com/api/resources/email_routing/subresources/rules/subresources/catch_alls/methods/get/) | [`emailRoutingZoneReadPath`](../src/routes.zig#L8737) |
| `GET /zones/{zone_id}/email/routing/rules/{rule_identifier}` | [Get routing rule](https://developers.cloudflare.com/api/resources/email_routing/subresources/rules/methods/get/) | [`emailRoutingZoneReadPath`](../src/routes.zig#L8737) |
| `GET /zones/{zone_id}/email/sending/subdomains` | [List sending subdomains](https://developers.cloudflare.com/api/resources/email_sending/subresources/subdomains/methods/list/) | [`emailSendingZoneReadPath`](../src/routes.zig#L8811) |
| `GET /zones/{zone_id}/email/sending/subdomains/{subdomain_id}` | [Get a sending subdomain](https://developers.cloudflare.com/api/resources/email_sending/subresources/subdomains/methods/get/) | [`emailSendingZoneReadPath`](../src/routes.zig#L8811) |
| `GET /zones/{zone_id}/email/sending/subdomains/{subdomain_id}/dns` | [Get sending subdomain DNS records](https://developers.cloudflare.com/api/resources/email_sending/subresources/subdomains/subresources/dns/methods/get/) | [`emailSendingZoneReadPath`](../src/routes.zig#L8811) |
| `GET /zones/{zone_id}/email/sending/subdomains/{subdomain_id}/dns/status` | Get sending subdomain DNS status — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`emailSendingZoneReadPath`](../src/routes.zig#L8811) |

## Access

Use Access routes for policies controlling who can reach applications, identity providers, groups, service tokens, and authentication configuration. Some routes exist at both account and zone scope.

| HTTP method and path | What it does / official reference | Zig route helper |
| --- | --- | --- |
| `GET /accounts/{account_id}/access/apps` | List Access applications — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessReadPath`](../src/routes.zig#L9024) |
| `POST /accounts/{account_id}/access/apps` | Add an Access application — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `GET /accounts/{account_id}/access/apps/ca` | List short-lived certificate CAs — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessReadPath`](../src/routes.zig#L9024) |
| `DELETE /accounts/{account_id}/access/apps/{app_id}` | Delete an Access application — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `GET /accounts/{account_id}/access/apps/{app_id}` | Get an Access application — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessReadPath`](../src/routes.zig#L9024) |
| `PUT /accounts/{account_id}/access/apps/{app_id}` | Update an Access application — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `DELETE /accounts/{account_id}/access/apps/{app_id}/ca` | Delete a short-lived certificate CA — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `GET /accounts/{account_id}/access/apps/{app_id}/ca` | Get a short-lived certificate CA — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessReadPath`](../src/routes.zig#L9024) |
| `POST /accounts/{account_id}/access/apps/{app_id}/ca` | Create a short-lived certificate CA — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `GET /accounts/{account_id}/access/apps/{app_id}/policies` | List Access application policies — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessReadPath`](../src/routes.zig#L9024) |
| `POST /accounts/{account_id}/access/apps/{app_id}/policies` | Create an Access application policy — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `DELETE /accounts/{account_id}/access/apps/{app_id}/policies/{policy_id}` | Delete an Access application policy — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `GET /accounts/{account_id}/access/apps/{app_id}/policies/{policy_id}` | Get an Access application policy — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessReadPath`](../src/routes.zig#L9024) |
| `PUT /accounts/{account_id}/access/apps/{app_id}/policies/{policy_id}` | Update an Access application policy — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `PUT /accounts/{account_id}/access/apps/{app_id}/policies/{policy_id}/make_reusable` | Convert an Access application policy to a reusable policy — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `POST /accounts/{account_id}/access/apps/{app_id}/revoke_tokens` | Revoke application tokens — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `PATCH /accounts/{account_id}/access/apps/{app_id}/settings` | Update Access application settings — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `PUT /accounts/{account_id}/access/apps/{app_id}/settings` | Update Access application settings — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `GET /accounts/{account_id}/access/apps/{app_id}/user_policy_checks` | Test Access policies — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessReadPath`](../src/routes.zig#L9024) |
| `GET /accounts/{account_id}/access/authenticator_device_aaguids` | List authenticator device AAGUIDs — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessReadPath`](../src/routes.zig#L9024) |
| `GET /accounts/{account_id}/access/certificates` | List mTLS certificates — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessReadPath`](../src/routes.zig#L9024) |
| `POST /accounts/{account_id}/access/certificates` | Add an mTLS certificate — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `GET /accounts/{account_id}/access/certificates/settings` | List all mTLS hostname settings — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessReadPath`](../src/routes.zig#L9024) |
| `PUT /accounts/{account_id}/access/certificates/settings` | Update an mTLS certificate's hostname settings — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `DELETE /accounts/{account_id}/access/certificates/{certificate_id}` | Delete an mTLS certificate — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `GET /accounts/{account_id}/access/certificates/{certificate_id}` | Get an mTLS certificate — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessReadPath`](../src/routes.zig#L9024) |
| `PUT /accounts/{account_id}/access/certificates/{certificate_id}` | Update an mTLS certificate — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `GET /accounts/{account_id}/access/custom_pages` | [List custom pages](https://developers.cloudflare.com/api/resources/zero_trust/subresources/access/subresources/custom_pages/methods/list/) | [`accessCustomPageReadPath`](../src/routes.zig#L8960) |
| `POST /accounts/{account_id}/access/custom_pages` | [Create a custom page](https://developers.cloudflare.com/api/resources/zero_trust/subresources/access/subresources/custom_pages/methods/create/) | [`accessCustomPageMutationPath`](../src/routes.zig#L8970) |
| `DELETE /accounts/{account_id}/access/custom_pages/{custom_page_id}` | [Delete a custom page](https://developers.cloudflare.com/api/resources/zero_trust/subresources/access/subresources/custom_pages/methods/delete/) | [`accessCustomPageMutationPath`](../src/routes.zig#L8970) |
| `GET /accounts/{account_id}/access/custom_pages/{custom_page_id}` | [Get a custom page](https://developers.cloudflare.com/api/resources/zero_trust/subresources/access/subresources/custom_pages/methods/get/) | [`accessCustomPageReadPath`](../src/routes.zig#L8960) |
| `PUT /accounts/{account_id}/access/custom_pages/{custom_page_id}` | [Update a custom page](https://developers.cloudflare.com/api/resources/zero_trust/subresources/access/subresources/custom_pages/methods/update/) | [`accessCustomPageMutationPath`](../src/routes.zig#L8970) |
| `GET /accounts/{account_id}/access/groups` | List Access groups — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessReadPath`](../src/routes.zig#L9024) |
| `POST /accounts/{account_id}/access/groups` | Create an Access group — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `DELETE /accounts/{account_id}/access/groups/{group_id}` | Delete an Access group — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `GET /accounts/{account_id}/access/groups/{group_id}` | Get an Access group — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessReadPath`](../src/routes.zig#L9024) |
| `PUT /accounts/{account_id}/access/groups/{group_id}` | Update an Access group — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `GET /accounts/{account_id}/access/identity_providers` | List Access identity providers — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessReadPath`](../src/routes.zig#L9024) |
| `POST /accounts/{account_id}/access/identity_providers` | Add an Access identity provider — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `DELETE /accounts/{account_id}/access/identity_providers/{identity_provider_id}` | Delete an Access identity provider — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `GET /accounts/{account_id}/access/identity_providers/{identity_provider_id}` | Get an Access identity provider — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessReadPath`](../src/routes.zig#L9024) |
| `PUT /accounts/{account_id}/access/identity_providers/{identity_provider_id}` | Update an Access identity provider — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `POST /accounts/{account_id}/access/identity_providers/{identity_provider_id}/saml_certificate` | [Create SAML encryption certificate for Identity Provider](https://developers.cloudflare.com/api/resources/zero_trust/subresources/identity_providers/subresources/saml_certificate/methods/create/) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `GET /accounts/{account_id}/access/identity_providers/{identity_provider_id}/scim/groups` | [List SCIM Group resources](https://developers.cloudflare.com/api/resources/zero_trust/subresources/identity_providers/subresources/scim/subresources/groups/methods/list/) | [`accessReadPath`](../src/routes.zig#L9024) |
| `GET /accounts/{account_id}/access/identity_providers/{identity_provider_id}/scim/users` | [List SCIM User resources](https://developers.cloudflare.com/api/resources/zero_trust/subresources/identity_providers/subresources/scim/subresources/users/methods/list/) | [`accessReadPath`](../src/routes.zig#L9024) |
| `GET /accounts/{account_id}/access/idp_federation_grants` | [List IdP federation grants](https://developers.cloudflare.com/api/resources/zero_trust/subresources/access/subresources/idp_federation_grants/methods/list/) | [`accessReadPath`](../src/routes.zig#L9024) |
| `POST /accounts/{account_id}/access/idp_federation_grants` | [Create an IdP federation grant](https://developers.cloudflare.com/api/resources/zero_trust/subresources/access/subresources/idp_federation_grants/methods/create/) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `DELETE /accounts/{account_id}/access/idp_federation_grants/{grant_id}` | [Delete an IdP federation grant](https://developers.cloudflare.com/api/resources/zero_trust/subresources/access/subresources/idp_federation_grants/methods/delete/) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `GET /accounts/{account_id}/access/idp_federation_grants/{grant_id}` | [Get an IdP federation grant](https://developers.cloudflare.com/api/resources/zero_trust/subresources/access/subresources/idp_federation_grants/methods/get/) | [`accessReadPath`](../src/routes.zig#L9024) |
| `GET /accounts/{account_id}/access/keys` | [Get the Access key configuration](https://developers.cloudflare.com/api/resources/zero_trust/subresources/access/subresources/keys/methods/get/) | [`accessReadPath`](../src/routes.zig#L9024) |
| `PUT /accounts/{account_id}/access/keys` | [Update the Access key configuration](https://developers.cloudflare.com/api/resources/zero_trust/subresources/access/subresources/keys/methods/update/) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `POST /accounts/{account_id}/access/keys/rotate` | [Rotate Access keys](https://developers.cloudflare.com/api/resources/zero_trust/subresources/access/subresources/keys/methods/rotate/) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `GET /accounts/{account_id}/access/logs/access_requests` | [Get Access authentication logs](https://developers.cloudflare.com/api/resources/zero_trust/subresources/access/subresources/logs/subresources/access_requests/methods/list/) | [`accessReadPath`](../src/routes.zig#L9024) |
| `GET /accounts/{account_id}/access/logs/scim/updates` | [List Access SCIM update logs](https://developers.cloudflare.com/api/resources/zero_trust/subresources/access/subresources/logs/subresources/scim/subresources/updates/methods/list/) | [`accessReadPath`](../src/routes.zig#L9024) |
| `GET /accounts/{account_id}/access/policies` | [List Access reusable policies](https://developers.cloudflare.com/api/resources/zero_trust/subresources/access/subresources/policies/methods/list/) | [`accessReadPath`](../src/routes.zig#L9024) |
| `POST /accounts/{account_id}/access/policies` | [Create an Access reusable policy](https://developers.cloudflare.com/api/resources/zero_trust/subresources/access/subresources/policies/methods/create/) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `DELETE /accounts/{account_id}/access/policies/{policy_id}` | [Delete an Access reusable policy](https://developers.cloudflare.com/api/resources/zero_trust/subresources/access/subresources/policies/methods/delete/) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `GET /accounts/{account_id}/access/policies/{policy_id}` | [Get an Access reusable policy](https://developers.cloudflare.com/api/resources/zero_trust/subresources/access/subresources/policies/methods/get/) | [`accessReadPath`](../src/routes.zig#L9024) |
| `PUT /accounts/{account_id}/access/policies/{policy_id}` | [Update an Access reusable policy](https://developers.cloudflare.com/api/resources/zero_trust/subresources/access/subresources/policies/methods/update/) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `POST /accounts/{account_id}/access/policy-tests` | [Start Access policy test](https://developers.cloudflare.com/api/resources/zero_trust/subresources/access/subresources/applications/subresources/policy_tests/methods/create/) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `GET /accounts/{account_id}/access/policy-tests/{policy_test_id}` | [Get the current status of a given Access policy test](https://developers.cloudflare.com/api/resources/zero_trust/subresources/access/subresources/applications/subresources/policy_tests/methods/get/) | [`accessReadPath`](../src/routes.zig#L9024) |
| `GET /accounts/{account_id}/access/policy-tests/{policy_test_id}/users` | [Get an Access policy test users page](https://developers.cloudflare.com/api/resources/zero_trust/subresources/access/subresources/applications/subresources/policy_tests/subresources/users/methods/list/) | [`accessReadPath`](../src/routes.zig#L9024) |
| `GET /accounts/{account_id}/access/saml_certificates` | [List SAML certificate sets](https://developers.cloudflare.com/api/resources/zero_trust/subresources/access/subresources/saml_certificates/methods/list/) | [`accessReadPath`](../src/routes.zig#L9024) |
| `GET /accounts/{account_id}/access/saml_certificates/{saml_cert_set_id}` | [Get SAML certificate set](https://developers.cloudflare.com/api/resources/zero_trust/subresources/access/subresources/saml_certificates/methods/get/) | [`accessReadPath`](../src/routes.zig#L9024) |
| `GET /accounts/{account_id}/access/saml_certificates/{saml_cert_set_id}/pem` | [Download current certificate in PEM format](https://developers.cloudflare.com/api/resources/zero_trust/subresources/access/subresources/saml_certificates/methods/get_pem/) | [`accessReadPath`](../src/routes.zig#L9024) |
| `POST /accounts/{account_id}/access/saml_certificates/{saml_cert_set_id}/rotate` | [Rotate SAML certificate](https://developers.cloudflare.com/api/resources/zero_trust/subresources/access/subresources/saml_certificates/methods/rotate/) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `GET /accounts/{account_id}/access/service_tokens` | List service tokens — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessReadPath`](../src/routes.zig#L9024) |
| `POST /accounts/{account_id}/access/service_tokens` | Create a service token — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `DELETE /accounts/{account_id}/access/service_tokens/{service_token_id}` | Delete a service token — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `GET /accounts/{account_id}/access/service_tokens/{service_token_id}` | Get a service token — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessReadPath`](../src/routes.zig#L9024) |
| `PUT /accounts/{account_id}/access/service_tokens/{service_token_id}` | Update a service token — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `POST /accounts/{account_id}/access/service_tokens/{service_token_id}/refresh` | [Refresh a service token](https://developers.cloudflare.com/api/resources/zero_trust/subresources/access/subresources/service_tokens/methods/refresh/) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `POST /accounts/{account_id}/access/service_tokens/{service_token_id}/rotate` | [Rotate a service token](https://developers.cloudflare.com/api/resources/zero_trust/subresources/access/subresources/service_tokens/methods/rotate/) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `GET /accounts/{account_id}/access/tags` | [List tags](https://developers.cloudflare.com/api/resources/zero_trust/subresources/access/subresources/tags/methods/list/) | [`accessReadPath`](../src/routes.zig#L9024) |
| `POST /accounts/{account_id}/access/tags` | [Create a tag](https://developers.cloudflare.com/api/resources/zero_trust/subresources/access/subresources/tags/methods/create/) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `DELETE /accounts/{account_id}/access/tags/{tag_name}` | [Delete a tag](https://developers.cloudflare.com/api/resources/zero_trust/subresources/access/subresources/tags/methods/delete/) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `GET /accounts/{account_id}/access/tags/{tag_name}` | [Get a tag](https://developers.cloudflare.com/api/resources/zero_trust/subresources/access/subresources/tags/methods/get/) | [`accessReadPath`](../src/routes.zig#L9024) |
| `PUT /accounts/{account_id}/access/tags/{tag_name}` | [Update a tag](https://developers.cloudflare.com/api/resources/zero_trust/subresources/access/subresources/tags/methods/update/) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `GET /zones/{zone_id}/access/apps` | List Access Applications — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessReadPath`](../src/routes.zig#L9024) |
| `POST /zones/{zone_id}/access/apps` | Add an Access application — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `GET /zones/{zone_id}/access/apps/ca` | List short-lived certificate CAs — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessReadPath`](../src/routes.zig#L9024) |
| `DELETE /zones/{zone_id}/access/apps/{app_id}` | Delete an Access application — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `GET /zones/{zone_id}/access/apps/{app_id}` | Get an Access application — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessReadPath`](../src/routes.zig#L9024) |
| `PUT /zones/{zone_id}/access/apps/{app_id}` | Update an Access application — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `DELETE /zones/{zone_id}/access/apps/{app_id}/ca` | Delete a short-lived certificate CA — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `GET /zones/{zone_id}/access/apps/{app_id}/ca` | Get a short-lived certificate CA — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessReadPath`](../src/routes.zig#L9024) |
| `POST /zones/{zone_id}/access/apps/{app_id}/ca` | Create a short-lived certificate CA — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `GET /zones/{zone_id}/access/apps/{app_id}/policies` | List Access policies — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessReadPath`](../src/routes.zig#L9024) |
| `POST /zones/{zone_id}/access/apps/{app_id}/policies` | Create an Access policy — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `DELETE /zones/{zone_id}/access/apps/{app_id}/policies/{policy_id}` | Delete an Access policy — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `GET /zones/{zone_id}/access/apps/{app_id}/policies/{policy_id}` | Get an Access policy — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessReadPath`](../src/routes.zig#L9024) |
| `PUT /zones/{zone_id}/access/apps/{app_id}/policies/{policy_id}` | Update an Access policy — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `POST /zones/{zone_id}/access/apps/{app_id}/revoke_tokens` | Revoke application tokens — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `PATCH /zones/{zone_id}/access/apps/{app_id}/settings` | Update application settings — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `PUT /zones/{zone_id}/access/apps/{app_id}/settings` | Update application settings — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `GET /zones/{zone_id}/access/apps/{app_id}/user_policy_checks` | Test Access policies — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessReadPath`](../src/routes.zig#L9024) |
| `GET /zones/{zone_id}/access/certificates` | List mTLS certificates — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessReadPath`](../src/routes.zig#L9024) |
| `POST /zones/{zone_id}/access/certificates` | Add an mTLS certificate — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `GET /zones/{zone_id}/access/certificates/settings` | List all mTLS hostname settings — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessReadPath`](../src/routes.zig#L9024) |
| `PUT /zones/{zone_id}/access/certificates/settings` | Update an mTLS certificate's hostname settings — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `DELETE /zones/{zone_id}/access/certificates/{certificate_id}` | Delete an mTLS certificate — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `GET /zones/{zone_id}/access/certificates/{certificate_id}` | Get an mTLS certificate — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessReadPath`](../src/routes.zig#L9024) |
| `PUT /zones/{zone_id}/access/certificates/{certificate_id}` | Update an mTLS certificate — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `GET /zones/{zone_id}/access/groups` | List Access groups — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessReadPath`](../src/routes.zig#L9024) |
| `POST /zones/{zone_id}/access/groups` | Create an Access group — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `DELETE /zones/{zone_id}/access/groups/{group_id}` | Delete an Access group — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `GET /zones/{zone_id}/access/groups/{group_id}` | Get an Access group — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessReadPath`](../src/routes.zig#L9024) |
| `PUT /zones/{zone_id}/access/groups/{group_id}` | Update an Access group — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `GET /zones/{zone_id}/access/identity_providers` | List Access identity providers — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessReadPath`](../src/routes.zig#L9024) |
| `POST /zones/{zone_id}/access/identity_providers` | Add an Access identity provider — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `DELETE /zones/{zone_id}/access/identity_providers/{identity_provider_id}` | Delete an Access identity provider — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `GET /zones/{zone_id}/access/identity_providers/{identity_provider_id}` | Get an Access identity provider — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessReadPath`](../src/routes.zig#L9024) |
| `PUT /zones/{zone_id}/access/identity_providers/{identity_provider_id}` | Update an Access identity provider — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `GET /zones/{zone_id}/access/service_tokens` | List service tokens — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessReadPath`](../src/routes.zig#L9024) |
| `POST /zones/{zone_id}/access/service_tokens` | Create a service token — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `DELETE /zones/{zone_id}/access/service_tokens/{service_token_id}` | Delete a service token — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessMutationPath`](../src/routes.zig#L9144) |
| `GET /zones/{zone_id}/access/service_tokens/{service_token_id}` | Get a service token — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessReadPath`](../src/routes.zig#L9024) |
| `PUT /zones/{zone_id}/access/service_tokens/{service_token_id}` | Update a service token — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`accessMutationPath`](../src/routes.zig#L9144) |

## Tunnels

Inspect tunnels, their configuration and connections, and the routes joining private networks to Cloudflare. These read helpers are useful for a connectivity inventory.

| HTTP method and path | What it does / official reference | Zig route helper |
| --- | --- | --- |
| `GET /accounts/{account_id}/cfd_tunnel` | [List Cloudflare Tunnels](https://developers.cloudflare.com/api/resources/zero_trust/subresources/tunnels/subresources/cloudflared/methods/list/) | [`tunnelReadPath`](../src/routes.zig#L9277) |
| `GET /accounts/{account_id}/cfd_tunnel/{tunnel_id}` | [Get a Cloudflare Tunnel](https://developers.cloudflare.com/api/resources/zero_trust/subresources/tunnels/subresources/cloudflared/methods/get/) | [`tunnelReadPath`](../src/routes.zig#L9277) |
| `GET /accounts/{account_id}/cfd_tunnel/{tunnel_id}/configurations` | [Get Tunnel configuration](https://developers.cloudflare.com/api/resources/zero_trust/subresources/tunnels/subresources/cloudflared/subresources/configurations/methods/get/) | [`tunnelReadPath`](../src/routes.zig#L9277) |
| `GET /accounts/{account_id}/cfd_tunnel/{tunnel_id}/connections` | [List Cloudflare Tunnel connections](https://developers.cloudflare.com/api/resources/zero_trust/subresources/tunnels/subresources/cloudflared/subresources/connections/methods/get/) | [`tunnelReadPath`](../src/routes.zig#L9277) |
| `GET /accounts/{account_id}/cfd_tunnel/{tunnel_id}/connectors/{connector_id}` | [Get Cloudflare Tunnel connector](https://developers.cloudflare.com/api/resources/zero_trust/subresources/tunnels/subresources/cloudflared/subresources/connectors/methods/get/) | [`tunnelReadPath`](../src/routes.zig#L9277) |
| `GET /accounts/{account_id}/cfd_tunnel/{tunnel_id}/token` | [Get a Cloudflare Tunnel token](https://developers.cloudflare.com/api/resources/zero_trust/subresources/tunnels/subresources/cloudflared/subresources/token/methods/get/) | [`tunnelReadPath`](../src/routes.zig#L9277) |
| `GET /accounts/{account_id}/teamnet/routes` | [List tunnel routes](https://developers.cloudflare.com/api/resources/zero_trust/subresources/networks/subresources/routes/methods/list/) | [`tunnelReadPath`](../src/routes.zig#L9277) |
| `GET /accounts/{account_id}/teamnet/routes/ip/{ip}` | [Get tunnel route by IP](https://developers.cloudflare.com/api/resources/zero_trust/subresources/networks/subresources/routes/subresources/ips/methods/get/) | [`tunnelReadPath`](../src/routes.zig#L9277) |
| `GET /accounts/{account_id}/teamnet/routes/{route_id}` | [Get tunnel route](https://developers.cloudflare.com/api/resources/zero_trust/subresources/networks/subresources/routes/methods/get/) | [`tunnelReadPath`](../src/routes.zig#L9277) |
| `GET /accounts/{account_id}/teamnet/virtual_networks` | [List virtual networks](https://developers.cloudflare.com/api/resources/zero_trust/subresources/networks/subresources/virtual_networks/methods/list/) | [`tunnelReadPath`](../src/routes.zig#L9277) |
| `GET /accounts/{account_id}/tunnels` | [List All Tunnels](https://developers.cloudflare.com/api/resources/zero_trust/subresources/tunnels/methods/list/) | [`tunnelReadPath`](../src/routes.zig#L9277) |
| `GET /accounts/{account_id}/warp_connector` | [List Mesh nodes](https://developers.cloudflare.com/api/resources/zero_trust/subresources/tunnels/subresources/warp_connector/methods/list/) | [`tunnelReadPath`](../src/routes.zig#L9277) |
| `GET /accounts/{account_id}/warp_connector/{tunnel_id}` | [Get a Mesh node](https://developers.cloudflare.com/api/resources/zero_trust/subresources/tunnels/subresources/warp_connector/methods/get/) | [`tunnelReadPath`](../src/routes.zig#L9277) |
| `GET /accounts/{account_id}/warp_connector/{tunnel_id}/configurations` | [Get Mesh node HA configuration](https://developers.cloudflare.com/api/resources/zero_trust/subresources/tunnels/subresources/warp_connector/subresources/configurations/methods/get/) | [`tunnelReadPath`](../src/routes.zig#L9277) |
| `GET /accounts/{account_id}/warp_connector/{tunnel_id}/connections` | [List Mesh node connections](https://developers.cloudflare.com/api/resources/zero_trust/subresources/tunnels/subresources/warp_connector/subresources/connections/methods/get/) | [`tunnelReadPath`](../src/routes.zig#L9277) |
| `GET /accounts/{account_id}/warp_connector/{tunnel_id}/connectors/{connector_id}` | [Get a Mesh node connector](https://developers.cloudflare.com/api/resources/zero_trust/subresources/tunnels/subresources/warp_connector/subresources/connectors/methods/get/) | [`tunnelReadPath`](../src/routes.zig#L9277) |
| `GET /accounts/{account_id}/warp_connector/{tunnel_id}/token` | [Get a Mesh node token](https://developers.cloudflare.com/api/resources/zero_trust/subresources/tunnels/subresources/warp_connector/subresources/token/methods/get/) | [`tunnelReadPath`](../src/routes.zig#L9277) |
| `GET /accounts/{account_id}/zerotrust/connectivity_settings` | [Get Zero Trust Connectivity Settings](https://developers.cloudflare.com/api/resources/zero_trust/subresources/connectivity_settings/methods/get/) | [`tunnelReadPath`](../src/routes.zig#L9277) |
| `GET /accounts/{account_id}/zerotrust/routes/hostname` | [List hostname routes](https://developers.cloudflare.com/api/resources/zero_trust/subresources/networks/subresources/hostname_routes/methods/list/) | [`tunnelReadPath`](../src/routes.zig#L9277) |
| `GET /accounts/{account_id}/zerotrust/routes/hostname/{hostname_route_id}` | [Get hostname route](https://developers.cloudflare.com/api/resources/zero_trust/subresources/networks/subresources/hostname_routes/methods/get/) | [`tunnelReadPath`](../src/routes.zig#L9277) |
| `GET /accounts/{account_id}/zerotrust/subnets` | [List Subnets](https://developers.cloudflare.com/api/resources/zero_trust/subresources/networks/subresources/subnets/methods/list/) | [`tunnelReadPath`](../src/routes.zig#L9277) |
| `GET /accounts/{account_id}/zerotrust/subnets/warp/{subnet_id}` | [Get WARP IP subnet](https://developers.cloudflare.com/api/resources/zero_trust/subresources/networks/subresources/subnets/subresources/warp/methods/get/) | [`tunnelReadPath`](../src/routes.zig#L9277) |

## Zero Trust and Gateway

Read device, network, filtering, organization, and Gateway configuration to build a private-network or security dashboard.

| HTTP method and path | What it does / official reference | Zig route helper |
| --- | --- | --- |
| `GET /accounts/{account_id}/access/organizations` | Get your Zero Trust organization — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`zeroTrustReadPath`](../src/routes.zig#L9376) |
| `GET /accounts/{account_id}/access/organizations/doh` | [Get your Zero Trust organization DoH settings](https://developers.cloudflare.com/api/resources/zero_trust/subresources/organizations/subresources/doh/methods/get/) | [`zeroTrustReadPath`](../src/routes.zig#L9376) |
| `GET /accounts/{account_id}/access/users` | [Get users](https://developers.cloudflare.com/api/resources/zero_trust/subresources/access/subresources/users/methods/list/) | [`zeroTrustReadPath`](../src/routes.zig#L9376) |
| `GET /accounts/{account_id}/access/users/{user_id}` | [Get a user](https://developers.cloudflare.com/api/resources/zero_trust/subresources/access/subresources/users/methods/get/) | [`zeroTrustReadPath`](../src/routes.zig#L9376) |
| `GET /accounts/{account_id}/access/users/{user_id}/active_sessions` | [Get active sessions](https://developers.cloudflare.com/api/resources/zero_trust/subresources/access/subresources/users/subresources/active_sessions/methods/list/) | [`zeroTrustReadPath`](../src/routes.zig#L9376) |
| `GET /accounts/{account_id}/access/users/{user_id}/active_sessions/{nonce}` | [Get single active session](https://developers.cloudflare.com/api/resources/zero_trust/subresources/access/subresources/users/subresources/active_sessions/methods/get/) | [`zeroTrustReadPath`](../src/routes.zig#L9376) |
| `GET /accounts/{account_id}/access/users/{user_id}/failed_logins` | [Get failed logins](https://developers.cloudflare.com/api/resources/zero_trust/subresources/access/subresources/users/subresources/failed_logins/methods/list/) | [`zeroTrustReadPath`](../src/routes.zig#L9376) |
| `GET /accounts/{account_id}/access/users/{user_id}/last_seen_identity` | [Get last seen identity](https://developers.cloudflare.com/api/resources/zero_trust/subresources/access/subresources/users/subresources/last_seen_identity/methods/get/) | [`zeroTrustReadPath`](../src/routes.zig#L9376) |
| `GET /accounts/{account_id}/devices/settings` | [Get device settings](https://developers.cloudflare.com/api/resources/zero_trust/subresources/devices/subresources/settings/methods/get/) | [`zeroTrustReadPath`](../src/routes.zig#L9376) |
| `GET /accounts/{account_id}/gateway` | [Get Zero Trust account information](https://developers.cloudflare.com/api/resources/zero_trust/subresources/gateway/methods/list/) | [`zeroTrustReadPath`](../src/routes.zig#L9376) |
| `GET /accounts/{account_id}/gateway/app_types` | [List application and application type mappings](https://developers.cloudflare.com/api/resources/zero_trust/subresources/gateway/subresources/app_types/methods/list/) | [`zeroTrustReadPath`](../src/routes.zig#L9376) |
| `GET /accounts/{account_id}/gateway/apps/review_status` | List applications review statuses — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`zeroTrustReadPath`](../src/routes.zig#L9376) |
| `GET /accounts/{account_id}/gateway/audit_ssh_settings` | [Get Zero Trust SSH settings](https://developers.cloudflare.com/api/resources/zero_trust/subresources/gateway/subresources/audit_ssh_settings/methods/get/) | [`zeroTrustReadPath`](../src/routes.zig#L9376) |
| `GET /accounts/{account_id}/gateway/categories` | [List categories](https://developers.cloudflare.com/api/resources/zero_trust/subresources/gateway/subresources/categories/methods/list/) | [`zeroTrustReadPath`](../src/routes.zig#L9376) |
| `GET /accounts/{account_id}/gateway/certificates` | [List Zero Trust certificates](https://developers.cloudflare.com/api/resources/zero_trust/subresources/gateway/subresources/certificates/methods/list/) | [`zeroTrustReadPath`](../src/routes.zig#L9376) |
| `GET /accounts/{account_id}/gateway/certificates/{certificate_id}` | [Get Zero Trust certificate details](https://developers.cloudflare.com/api/resources/zero_trust/subresources/gateway/subresources/certificates/methods/get/) | [`zeroTrustReadPath`](../src/routes.zig#L9376) |
| `GET /accounts/{account_id}/gateway/configuration` | [Get Zero Trust account configuration](https://developers.cloudflare.com/api/resources/zero_trust/subresources/gateway/subresources/configurations/methods/get/) | [`zeroTrustReadPath`](../src/routes.zig#L9376) |
| `GET /accounts/{account_id}/gateway/dns_destination_ips` | List Zero Trust Gateway DNS destination IPv4 address pairs — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`zeroTrustReadPath`](../src/routes.zig#L9376) |
| `GET /accounts/{account_id}/gateway/egress_cidr_pairs` | Get gateway egress CIDRs pairs assigned to this account — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`zeroTrustReadPath`](../src/routes.zig#L9376) |
| `GET /accounts/{account_id}/gateway/lists` | [List Zero Trust lists](https://developers.cloudflare.com/api/resources/zero_trust/subresources/gateway/subresources/lists/methods/list/) | [`zeroTrustReadPath`](../src/routes.zig#L9376) |
| `GET /accounts/{account_id}/gateway/lists/{list_id}` | [Get Zero Trust list details](https://developers.cloudflare.com/api/resources/zero_trust/subresources/gateway/subresources/lists/methods/get/) | [`zeroTrustReadPath`](../src/routes.zig#L9376) |
| `GET /accounts/{account_id}/gateway/lists/{list_id}/items` | [Get Zero Trust list items](https://developers.cloudflare.com/api/resources/zero_trust/subresources/gateway/subresources/lists/subresources/items/methods/list/) | [`zeroTrustReadPath`](../src/routes.zig#L9376) |
| `GET /accounts/{account_id}/gateway/locations` | [List Zero Trust Gateway locations](https://developers.cloudflare.com/api/resources/zero_trust/subresources/gateway/subresources/locations/methods/list/) | [`zeroTrustReadPath`](../src/routes.zig#L9376) |
| `GET /accounts/{account_id}/gateway/locations/{location_id}` | [Get Zero Trust Gateway location details](https://developers.cloudflare.com/api/resources/zero_trust/subresources/gateway/subresources/locations/methods/get/) | [`zeroTrustReadPath`](../src/routes.zig#L9376) |
| `GET /accounts/{account_id}/gateway/logging` | [Get logging settings for the Zero Trust account](https://developers.cloudflare.com/api/resources/zero_trust/subresources/gateway/subresources/logging/methods/get/) | [`zeroTrustReadPath`](../src/routes.zig#L9376) |
| `GET /accounts/{account_id}/gateway/operations` | List Zero Trust Gateway operations — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`zeroTrustReadPath`](../src/routes.zig#L9376) |
| `GET /accounts/{account_id}/gateway/operations/{operation_id}` | Zero Trust Gateway operation details — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`zeroTrustReadPath`](../src/routes.zig#L9376) |
| `GET /accounts/{account_id}/gateway/pacfiles` | [List PAC files](https://developers.cloudflare.com/api/resources/zero_trust/subresources/gateway/subresources/pacfiles/methods/list/) | [`zeroTrustReadPath`](../src/routes.zig#L9376) |
| `GET /accounts/{account_id}/gateway/pacfiles/{pacfile_id}` | [Get a PAC file](https://developers.cloudflare.com/api/resources/zero_trust/subresources/gateway/subresources/pacfiles/methods/get/) | [`zeroTrustReadPath`](../src/routes.zig#L9376) |
| `GET /accounts/{account_id}/gateway/proxy_endpoints` | [List proxy endpoints](https://developers.cloudflare.com/api/resources/zero_trust/subresources/gateway/subresources/proxy_endpoints/methods/list/) | [`zeroTrustReadPath`](../src/routes.zig#L9376) |
| `GET /accounts/{account_id}/gateway/proxy_endpoints/{proxy_endpoint_id}` | [Get a proxy endpoint](https://developers.cloudflare.com/api/resources/zero_trust/subresources/gateway/subresources/proxy_endpoints/methods/get/) | [`zeroTrustReadPath`](../src/routes.zig#L9376) |
| `GET /accounts/{account_id}/gateway/rules` | [List Zero Trust Gateway rules](https://developers.cloudflare.com/api/resources/zero_trust/subresources/gateway/subresources/rules/methods/list/) | [`zeroTrustReadPath`](../src/routes.zig#L9376) |
| `GET /accounts/{account_id}/gateway/rules/tenant` | [List Zero Trust Gateway rules inherited from the parent account](https://developers.cloudflare.com/api/resources/zero_trust/subresources/gateway/subresources/rules/methods/list_tenant/) | [`zeroTrustReadPath`](../src/routes.zig#L9376) |
| `GET /accounts/{account_id}/gateway/rules/{rule_id}` | [Get Zero Trust Gateway rule details](https://developers.cloudflare.com/api/resources/zero_trust/subresources/gateway/subresources/rules/methods/get/) | [`zeroTrustReadPath`](../src/routes.zig#L9376) |

## Security reports and audit

Read security findings or audit records when investigating a change. Collection routes can be paginated or filtered; request only the interval and resources you need.

| HTTP method and path | What it does / official reference | Zig route helper |
| --- | --- | --- |
| `GET /accounts/{account_id}/audit_logs` | [Get account audit logs](https://developers.cloudflare.com/api/resources/audit_logs/methods/list/) | [`auditLogReadPath`](../src/routes.zig#L9580) |
| `GET /accounts/{account_id}/intel/attack-surface-report/issue-types` | [Retrieves Security Center Issues Types](https://developers.cloudflare.com/api/resources/intel/subresources/attack_surface_report/subresources/issue_types/methods/get/) | [`securityCenterReadPath`](../src/routes.zig#L9482) |
| `GET /accounts/{account_id}/logs/audit` | [Get account audit logs (Version 2)](https://developers.cloudflare.com/api/resources/accounts/subresources/logs/subresources/audit/methods/list/) | [`auditLogReadPath`](../src/routes.zig#L9580) |
| `GET /accounts/{account_id}/security-center/insights` | Retrieves Security Center Insights — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`securityCenterReadPath`](../src/routes.zig#L9482) |
| `GET /accounts/{account_id}/security-center/insights/audit-log` | Retrieves Account Audit Log — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`securityCenterReadPath`](../src/routes.zig#L9482) |
| `GET /accounts/{account_id}/security-center/insights/class` | Retrieves Security Center Insight Counts by Class — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`securityCenterReadPath`](../src/routes.zig#L9482) |
| `GET /accounts/{account_id}/security-center/insights/severity` | Retrieves Security Center Insight Counts by Severity — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`securityCenterReadPath`](../src/routes.zig#L9482) |
| `GET /accounts/{account_id}/security-center/insights/type` | Retrieves Security Center Insight Counts by Type — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`securityCenterReadPath`](../src/routes.zig#L9482) |
| `GET /accounts/{account_id}/security-center/insights/{issue_id}/audit-log` | Retrieves Issue Audit Log — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`securityCenterReadPath`](../src/routes.zig#L9482) |
| `GET /accounts/{account_id}/security-center/insights/{issue_id}/context` | [Retrieves Security Center Insight Context](https://developers.cloudflare.com/api/resources/security_center/subresources/insights/subresources/context/methods/get/) | [`securityCenterReadPath`](../src/routes.zig#L9482) |
| `GET /organizations/{organization_id}/logs/audit` | [Get organization audit logs (Version 2)](https://developers.cloudflare.com/api/resources/organizations/subresources/logs/subresources/audit/methods/list/) | [`auditLogReadPath`](../src/routes.zig#L9580) |
| `GET /user/audit_logs` | [Get user audit logs](https://developers.cloudflare.com/api/resources/user/subresources/audit_logs/methods/list/) | [`auditLogReadPath`](../src/routes.zig#L9580) |
| `GET /zones/{zone_id}/security-center/insights` | Retrieves Zone Security Center Insights — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`securityCenterReadPath`](../src/routes.zig#L9482) |
| `GET /zones/{zone_id}/security-center/insights/audit-log` | Retrieves Zone Audit Log — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`securityCenterReadPath`](../src/routes.zig#L9482) |
| `GET /zones/{zone_id}/security-center/insights/class` | Retrieves Zone Security Center Insight Counts by Class — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`securityCenterReadPath`](../src/routes.zig#L9482) |
| `GET /zones/{zone_id}/security-center/insights/severity` | Retrieves Zone Security Center Insight Counts by Severity — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`securityCenterReadPath`](../src/routes.zig#L9482) |
| `GET /zones/{zone_id}/security-center/insights/type` | Retrieves Zone Security Center Insight Counts by Type — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`securityCenterReadPath`](../src/routes.zig#L9482) |
| `GET /zones/{zone_id}/security-center/insights/{issue_id}/audit-log` | Retrieves Zone Issue Audit Log — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`securityCenterReadPath`](../src/routes.zig#L9482) |

## Logs

Inspect log delivery jobs, available datasets, and received logs. Configuring a Logpush job and retrieving event data are different operations; follow the linked reference for the chosen route.

| HTTP method and path | What it does / official reference | Zig route helper |
| --- | --- | --- |
| `GET /accounts/{account_id}/logpush/datasets/{dataset_id}/fields` | List fields — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`logpushReadPath`](../src/routes.zig#L9658) |
| `GET /accounts/{account_id}/logpush/datasets/{dataset_id}/jobs` | List Logpush jobs for a dataset — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`logpushReadPath`](../src/routes.zig#L9658) |
| `GET /accounts/{account_id}/logpush/jobs` | List Logpush jobs — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`logpushReadPath`](../src/routes.zig#L9658) |
| `GET /accounts/{account_id}/logpush/jobs/{job_id}` | Get Logpush job details — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`logpushReadPath`](../src/routes.zig#L9658) |
| `GET /accounts/{account_id}/logs/explorer/datasets` | List account datasets — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`logExplorerReadPath`](../src/routes.zig#L9695) |
| `GET /accounts/{account_id}/logs/explorer/datasets/available` | List available account datasets — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`logExplorerReadPath`](../src/routes.zig#L9695) |
| `GET /accounts/{account_id}/logs/explorer/datasets/{dataset_id}` | Get an account dataset — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`logExplorerReadPath`](../src/routes.zig#L9695) |
| `GET /zones/{zone_id}/logpush/datasets/{dataset_id}/fields` | [List fields](https://developers.cloudflare.com/api/resources/logpush/subresources/datasets/subresources/fields/methods/get/) | [`logpushReadPath`](../src/routes.zig#L9658) |
| `GET /zones/{zone_id}/logpush/datasets/{dataset_id}/jobs` | [List Logpush jobs for a dataset](https://developers.cloudflare.com/api/resources/logpush/subresources/datasets/subresources/jobs/methods/get/) | [`logpushReadPath`](../src/routes.zig#L9658) |
| `GET /zones/{zone_id}/logpush/jobs` | [List Logpush jobs](https://developers.cloudflare.com/api/resources/logpush/subresources/jobs/methods/list/) | [`logpushReadPath`](../src/routes.zig#L9658) |
| `GET /zones/{zone_id}/logpush/jobs/{job_id}` | [Get Logpush job details](https://developers.cloudflare.com/api/resources/logpush/subresources/jobs/methods/get/) | [`logpushReadPath`](../src/routes.zig#L9658) |
| `GET /zones/{zone_id}/logs/control/retention/flag` | [Get log retention flag](https://developers.cloudflare.com/api/resources/logs/subresources/control/subresources/retention/methods/get/) | [`logsReceivedReadPath`](../src/routes.zig#L9718) |
| `GET /zones/{zone_id}/logs/explorer/datasets` | List zone datasets — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`logExplorerReadPath`](../src/routes.zig#L9695) |
| `GET /zones/{zone_id}/logs/explorer/datasets/available` | List available zone datasets — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`logExplorerReadPath`](../src/routes.zig#L9695) |
| `GET /zones/{zone_id}/logs/explorer/datasets/{dataset_id}` | Get a zone dataset — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`logExplorerReadPath`](../src/routes.zig#L9695) |
| `GET /zones/{zone_id}/logs/rayids/{ray_id}` | [Get logs RayIDs](https://developers.cloudflare.com/api/resources/logs/subresources/rayid/methods/get/) | [`logsReceivedReadPath`](../src/routes.zig#L9718) |
| `GET /zones/{zone_id}/logs/received` | [Get logs received](https://developers.cloudflare.com/api/resources/logs/subresources/received/methods/get/) | [`logsReceivedReadPath`](../src/routes.zig#L9718) |
| `GET /zones/{zone_id}/logs/received/fields` | [List fields](https://developers.cloudflare.com/api/resources/logs/subresources/received/subresources/fields/methods/get/) | [`logsReceivedReadPath`](../src/routes.zig#L9718) |

## Certificates and TLS

Inspect the certificates and TLS settings used to secure connections between visitors, Cloudflare, and your origin servers.

| HTTP method and path | What it does / official reference | Zig route helper |
| --- | --- | --- |
| `GET /accounts/{account_id}/custom_csrs` | List Custom CSRs — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`tlsReadPath`](../src/routes.zig#L9765) |
| `GET /accounts/{account_id}/custom_csrs/{custom_csr_id}` | Custom CSR Details — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`tlsReadPath`](../src/routes.zig#L9765) |
| `GET /certificates` | [List Certificates](https://developers.cloudflare.com/api/resources/origin_ca_certificates/methods/list/) | [`tlsReadPath`](../src/routes.zig#L9765) |
| `GET /certificates/{certificate_id}` | [Get Certificate](https://developers.cloudflare.com/api/resources/origin_ca_certificates/methods/get/) | [`tlsReadPath`](../src/routes.zig#L9765) |
| `GET /zones/{zone_id}/acm/custom_trust_store` | [List Custom Origin Trust Store Details](https://developers.cloudflare.com/api/resources/acm/subresources/custom_trust_store/methods/list/) | [`tlsReadPath`](../src/routes.zig#L9765) |
| `GET /zones/{zone_id}/acm/custom_trust_store/{custom_origin_trust_store_id}` | [Custom Origin Trust Store Details](https://developers.cloudflare.com/api/resources/acm/subresources/custom_trust_store/methods/get/) | [`tlsReadPath`](../src/routes.zig#L9765) |
| `GET /zones/{zone_id}/acm/total_tls` | [Total TLS Settings Details](https://developers.cloudflare.com/api/resources/acm/subresources/total_tls/methods/get/) | [`tlsReadPath`](../src/routes.zig#L9765) |
| `GET /zones/{zone_id}/custom_certificates` | [List SSL Configurations](https://developers.cloudflare.com/api/resources/custom_certificates/methods/list/) | [`tlsReadPath`](../src/routes.zig#L9765) |
| `GET /zones/{zone_id}/custom_certificates/{custom_certificate_id}` | [SSL Configuration Details](https://developers.cloudflare.com/api/resources/custom_certificates/methods/get/) | [`tlsReadPath`](../src/routes.zig#L9765) |
| `GET /zones/{zone_id}/custom_csrs` | List Custom CSRs — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`tlsReadPath`](../src/routes.zig#L9765) |
| `GET /zones/{zone_id}/custom_csrs/{custom_csr_id}` | Custom CSR Details — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`tlsReadPath`](../src/routes.zig#L9765) |
| `GET /zones/{zone_id}/hostnames/settings/{setting_id}` | [List TLS setting for hostnames](https://developers.cloudflare.com/api/resources/hostnames/subresources/settings/subresources/tls/methods/list/) | [`tlsReadPath`](../src/routes.zig#L9765) |
| `GET /zones/{zone_id}/hostnames/settings/{setting_id}/{hostname}` | [Get TLS setting for hostname](https://developers.cloudflare.com/api/resources/hostnames/subresources/settings/subresources/tls/methods/get/) | [`tlsReadPath`](../src/routes.zig#L9765) |
| `GET /zones/{zone_id}/keyless_certificates` | [List Keyless SSL Configurations](https://developers.cloudflare.com/api/resources/keyless_certificates/methods/list/) | [`tlsReadPath`](../src/routes.zig#L9765) |
| `GET /zones/{zone_id}/keyless_certificates/{keyless_certificate_id}` | [Get Keyless SSL Configuration](https://developers.cloudflare.com/api/resources/keyless_certificates/methods/get/) | [`tlsReadPath`](../src/routes.zig#L9765) |
| `GET /zones/{zone_id}/origin_tls_client_auth` | [List Certificates](https://developers.cloudflare.com/api/resources/origin_tls_client_auth/methods/list/) | [`tlsReadPath`](../src/routes.zig#L9765) |
| `GET /zones/{zone_id}/origin_tls_client_auth/hostnames` | List Hostname Associations — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`tlsReadPath`](../src/routes.zig#L9765) |
| `GET /zones/{zone_id}/origin_tls_client_auth/hostnames/certificates` | [List Certificates](https://developers.cloudflare.com/api/resources/origin_tls_client_auth/subresources/hostname_certificates/methods/list/) | [`tlsReadPath`](../src/routes.zig#L9765) |
| `GET /zones/{zone_id}/origin_tls_client_auth/hostnames/certificates/{certificate_id}` | [Get the Hostname Client Certificate](https://developers.cloudflare.com/api/resources/origin_tls_client_auth/subresources/hostname_certificates/methods/get/) | [`tlsReadPath`](../src/routes.zig#L9765) |
| `GET /zones/{zone_id}/origin_tls_client_auth/hostnames/{hostname}` | [Get the Hostname Status for Client Authentication](https://developers.cloudflare.com/api/resources/origin_tls_client_auth/subresources/hostnames/methods/get/) | [`tlsReadPath`](../src/routes.zig#L9765) |
| `GET /zones/{zone_id}/origin_tls_client_auth/settings` | [Get Enablement Setting for Zone](https://developers.cloudflare.com/api/resources/origin_tls_client_auth/subresources/settings/methods/get/) | [`tlsReadPath`](../src/routes.zig#L9765) |
| `GET /zones/{zone_id}/origin_tls_client_auth/{certificate_id}` | [Get Certificate Details](https://developers.cloudflare.com/api/resources/origin_tls_client_auth/methods/get/) | [`tlsReadPath`](../src/routes.zig#L9765) |
| `GET /zones/{zone_id}/settings/ssl_automatic_mode` | [Get Automatic SSL/TLS enrollment status for the given zone](https://developers.cloudflare.com/api/resources/ssl/subresources/automatic_upgrader/methods/get/) | [`tlsReadPath`](../src/routes.zig#L9765) |
| `GET /zones/{zone_id}/ssl/certificate_packs` | [List Certificate Packs](https://developers.cloudflare.com/api/resources/ssl/subresources/certificate_packs/methods/list/) | [`tlsReadPath`](../src/routes.zig#L9765) |
| `GET /zones/{zone_id}/ssl/certificate_packs/quota` | [Get Certificate Pack Quotas](https://developers.cloudflare.com/api/resources/ssl/subresources/certificate_packs/subresources/quota/methods/get/) | [`tlsReadPath`](../src/routes.zig#L9765) |
| `GET /zones/{zone_id}/ssl/certificate_packs/{certificate_pack_id}` | [Get Certificate Pack](https://developers.cloudflare.com/api/resources/ssl/subresources/certificate_packs/methods/get/) | [`tlsReadPath`](../src/routes.zig#L9765) |
| `GET /zones/{zone_id}/ssl/universal/settings` | [Universal SSL Settings Details](https://developers.cloudflare.com/api/resources/ssl/subresources/universal/subresources/settings/methods/get/) | [`tlsReadPath`](../src/routes.zig#L9765) |
| `GET /zones/{zone_id}/ssl/verification` | [SSL Verification Details](https://developers.cloudflare.com/api/resources/ssl/subresources/verification/methods/get/) | [`tlsReadPath`](../src/routes.zig#L9765) |

## Zone settings and performance

Read domain-level cache, performance, environment, plan, and subscription settings. A domain name is resolved to a zone ID before using these routes.

| HTTP method and path | What it does / official reference | Zig route helper |
| --- | --- | --- |
| `GET /zones/{zone_id}` | [Zone Details](https://developers.cloudflare.com/api/resources/zones/methods/get/) | [`zonePath`](../src/routes.zig#L10201) |
| `GET /zones/{zone_id}/dns_settings` | [Show DNS Settings](https://developers.cloudflare.com/api/resources/dns/subresources/settings/subresources/zone/methods/get/) | [`zoneEndpointPath`](../src/routes.zig#L10321) |
| `GET /zones/{zone_id}/settings` | [Get all zone settings](https://developers.cloudflare.com/api/resources/zones/subresources/settings/methods/get/) | [`zoneEndpointPath`](../src/routes.zig#L10321) |
| `GET /zones/{zone_id}/settings/aegis` | Get aegis setting — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`zoneEndpointPath`](../src/routes.zig#L10321) |
| `GET /zones/{zone_id}/settings/fonts` | [Get Cloudflare Fonts setting](https://developers.cloudflare.com/api/resources/zones/subresources/settings/methods/get/) | [`zoneEndpointPath`](../src/routes.zig#L10321) |
| `GET /zones/{zone_id}/settings/origin_h2_max_streams` | Get Origin H2 Max Streams Setting — [schema](https://github.com/cloudflare/api-schemas/blob/main/openapi.json) | [`zoneEndpointPath`](../src/routes.zig#L10321) |
| `GET /zones/{zone_id}/settings/origin_max_http_version` | [Get Origin Max HTTP Version Setting](https://developers.cloudflare.com/api/resources/zones/subresources/settings/methods/get/) | [`zoneEndpointPath`](../src/routes.zig#L10321) |
| `GET /zones/{zone_id}/settings/speed_brain` | [Get Cloudflare Speed Brain setting](https://developers.cloudflare.com/api/resources/zones/subresources/settings/methods/get/) | [`zoneEndpointPath`](../src/routes.zig#L10321) |
| `GET /zones/{zone_id}/settings/{setting_id}` | [Get zone setting](https://developers.cloudflare.com/api/resources/zones/subresources/settings/methods/get/) | [`zoneSettingPath`](../src/routes.zig#L10435) |

## Browser Run

Use `client.browserRun(account_id, .kitesurf)` or `client.browserRun(account_id, .chromium_default)` to obtain the typed client. `{browser_api}` below is `browser-run` for Kitesurf and `browser-rendering` for Chromium. Kitesurf adds `browser=kitesurf` to the request URL. Quick Actions also add `cacheTTL`, which defaults to zero.

| HTTP method and path | What it does | Typed method |
| --- | --- | --- |
| `POST /accounts/{account_id}/{browser_api}/content` | Return rendered HTML | `content` |
| `POST /accounts/{account_id}/{browser_api}/screenshot` | Return or stream an image of the page | `screenshot`, `screenshotTo` |
| `POST /accounts/{account_id}/{browser_api}/devtools/browser` | Create a browser session | `createSession` |
| `GET /accounts/{account_id}/{browser_api}/devtools/session` | List existing sessions | `listSessions` |
| `GET /accounts/{account_id}/{browser_api}/devtools/session/{session_id}` | Inspect one session | `getSession` |
| `DELETE /accounts/{account_id}/{browser_api}/devtools/browser/{session_id}` | Close a session | `closeSession` |
| `GET /accounts/{account_id}/{browser_api}/devtools/browser/{session_id}/json/list` | List page targets in a session | `listTargets` |
| `PUT /accounts/{account_id}/{browser_api}/devtools/browser/{session_id}/json/new` | Create a page target, optionally at a URL | `newTarget` |

See [the library's Browser Run guide](browser-run.md) for payloads, response ownership, and cleanup, and the official references for [content](https://developers.cloudflare.com/browser-run/quick-actions/content-endpoint/), [screenshots](https://developers.cloudflare.com/browser-run/quick-actions/screenshot-endpoint/), [HTTP session management](https://developers.cloudflare.com/api/resources/browser_rendering/subresources/devtools/), and [Kitesurf](https://developers.cloudflare.com/browser-run/kitesurf/). The library does not provide PDF, Markdown, crawl, or CDP WebSocket wrappers. Those provider capabilities are not implied by the supported rows above.

## Maintaining this reference

Update this guide when adding or changing a route. Paths and methods were checked against the implementation and the provider's current OpenAPI specification on 5 October 2026. [Cloudflare schema](https://github.com/cloudflare/api-schemas) is the upstream source; the linked operation pages are the reader-facing reference.
