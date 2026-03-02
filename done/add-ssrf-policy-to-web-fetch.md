# Add SSRF Policy Support to web_fetch Tool

## Problem

Users running Surge (or similar TUN-based proxies) on macOS have all DNS resolving to `198.18.x.x` (RFC 2544 benchmark range). OpenClaw's SSRF guard correctly blocks these as private IPs, causing `web_fetch` to fail with:

```
Blocked: resolves to private/internal/special-use IP address
```

The SSRF layer already supports `allowRfc2544BenchmarkRange: true` in `SsrFPolicy`, and the browser tool already exposes `ssrfPolicy` in its config. But `web_fetch` has no way to pass an SSRF policy — it always uses the default (block everything).

## Goal

Expose `ssrfPolicy` in the `web.fetch` config (same shape as `browser.ssrfPolicy`), and wire it through to `fetchWithWebToolsNetworkGuard` → `fetchWithSsrFGuard` so users can set:

```json
{
  "tools": {
    "web": {
      "fetch": {
        "ssrfPolicy": {
          "allowRfc2544BenchmarkRange": true
        }
      }
    }
  }
}
```

## Files to Change

### 1. `src/config/types.tools.ts` — Add ssrfPolicy to fetch config type

Find the `fetch?` type definition (around line 478) and add `ssrfPolicy` field:

```typescript
fetch?: {
  // ... existing fields ...
  /** SSRF policy overrides for web fetch requests. */
  ssrfPolicy?: {
    allowPrivateNetwork?: boolean;
    dangerouslyAllowPrivateNetwork?: boolean;
    allowRfc2544BenchmarkRange?: boolean;
    allowedHostnames?: string[];
    hostnameAllowlist?: string[];
  };
};
```

### 2. `src/config/zod-schema.ts` — Add ssrfPolicy to fetch zod schema

Find where the fetch config schema is defined and add the same `ssrfPolicy` object. Reuse the same shape as the browser ssrfPolicy (around line 254). If there's no explicit zod schema for fetch, find where `tools.web.fetch` is validated and add:

```typescript
ssrfPolicy: z
  .object({
    allowPrivateNetwork: z.boolean().optional(),
    dangerouslyAllowPrivateNetwork: z.boolean().optional(),
    allowRfc2544BenchmarkRange: z.boolean().optional(),
    allowedHostnames: z.array(z.string()).optional(),
    hostnameAllowlist: z.array(z.string()).optional(),
  })
  .strict()
  .optional(),
```

**Note:** The browser ssrfPolicy schema does NOT include `allowRfc2544BenchmarkRange` yet — add it there too for consistency.

### 3. `src/config/schema.labels.ts` — Add labels for new config keys

Add labels for the new fetch ssrfPolicy keys (follow the pattern of the browser ones around line 451):

```typescript
"tools.web.fetch.ssrfPolicy": "Web Fetch SSRF Policy",
"tools.web.fetch.ssrfPolicy.allowPrivateNetwork": "Web Fetch Allow Private Network",
"tools.web.fetch.ssrfPolicy.dangerouslyAllowPrivateNetwork": "Web Fetch Dangerously Allow Private Network",
"tools.web.fetch.ssrfPolicy.allowRfc2544BenchmarkRange": "Web Fetch Allow RFC 2544 Benchmark Range",
"tools.web.fetch.ssrfPolicy.allowedHostnames": "Web Fetch Allowed Hostnames",
"tools.web.fetch.ssrfPolicy.hostnameAllowlist": "Web Fetch Hostname Allowlist",
```

Also add the missing `allowRfc2544BenchmarkRange` label for browser:

```typescript
"browser.ssrfPolicy.allowRfc2544BenchmarkRange": "Browser Allow RFC 2544 Benchmark Range",
```

### 4. `src/agents/tools/web-guarded-fetch.ts` — Accept and pass through ssrfPolicy

Currently `fetchWithWebToolsNetworkGuard` doesn't accept or pass any SSRF policy. Change it to accept an optional `policy` and pass it through:

```typescript
import type { SsrFPolicy } from "../../infra/net/ssrf.js";

type WebToolGuardedFetchOptions = Omit<GuardedFetchOptions, "proxy"> & {
  timeoutSeconds?: number;
  policy?: SsrFPolicy; // ADD THIS
};

export async function fetchWithWebToolsNetworkGuard(
  params: WebToolGuardedFetchOptions,
): Promise<GuardedFetchResult> {
  const { timeoutSeconds, policy, ...rest } = params; // ADD policy to destructure
  return fetchWithSsrFGuard({
    ...rest,
    policy, // ADD THIS — pass through to fetchWithSsrFGuard
    timeoutMs: resolveTimeoutMs({ timeoutMs: rest.timeoutMs, timeoutSeconds }),
    proxy: "env",
  });
}
```

Do the same for `withWebToolsNetworkGuard` if it also constructs fetch options.

### 5. `src/agents/tools/web-fetch.ts` — Read ssrfPolicy from config and pass it

In the function that performs the fetch (around line 526), read the ssrfPolicy from the resolved fetch config and pass it to `fetchWithWebToolsNetworkGuard`:

Find where `fetchWithWebToolsNetworkGuard` is called:

```typescript
const result = await fetchWithWebToolsNetworkGuard({
  url: params.url,
  maxRedirects: params.maxRedirects,
  timeoutSeconds: params.timeoutSeconds,
  // ...
});
```

Add the policy from config:

```typescript
const result = await fetchWithWebToolsNetworkGuard({
  url: params.url,
  maxRedirects: params.maxRedirects,
  timeoutSeconds: params.timeoutSeconds,
  policy: fetchConfig?.ssrfPolicy, // ADD THIS
  // ...
});
```

Make sure `fetchConfig` (from `resolveFetchConfig(cfg)`) is available at the call site. Trace how the config flows — the tool factory function at the bottom of the file receives `config?: OpenClawConfig`. The ssrfPolicy should flow: `config.tools.web.fetch.ssrfPolicy` → `fetchWithWebToolsNetworkGuard({ policy })` → `fetchWithSsrFGuard({ policy })`.

### 6. `src/infra/net/ssrf.ts` — Already supports `allowRfc2544BenchmarkRange`

The `SsrFPolicy` type already has `allowRfc2544BenchmarkRange?: boolean`. No changes needed here. Just verify it's in the type — if not, add it.

## Also update: browser ssrfPolicy zod schema

In `src/config/zod-schema.ts` around line 254, the browser `ssrfPolicy` schema is missing `allowRfc2544BenchmarkRange`. Add it:

```typescript
ssrfPolicy: z
  .object({
    allowPrivateNetwork: z.boolean().optional(),
    dangerouslyAllowPrivateNetwork: z.boolean().optional(),
    allowRfc2544BenchmarkRange: z.boolean().optional(),  // ADD THIS
    allowedHostnames: z.array(z.string()).optional(),
    hostnameAllowlist: z.array(z.string()).optional(),
  })
  .strict()
  .optional(),
```

## Testing

After changes, verify:

1. `npm run build` (or `pnpm build`) succeeds with no type errors
2. Existing tests pass: `npm test` (or `pnpm test`)
3. The new config is accepted without validation errors:
   ```json
   {
     "tools": {
       "web": {
         "fetch": {
           "ssrfPolicy": {
             "allowRfc2544BenchmarkRange": true
           }
         }
       }
     }
   }
   ```

## Context

- `SsrFPolicy` type: `src/infra/net/ssrf.ts` (line ~33)
- Browser ssrfPolicy config pattern: `src/config/zod-schema.ts` (line ~254)
- Browser ssrfPolicy labels: `src/config/schema.labels.ts` (line ~451)
- Fetch guard: `src/infra/net/fetch-guard.ts` — `fetchWithSsrFGuard` accepts `policy?: SsrFPolicy`
- Web guarded fetch: `src/agents/tools/web-guarded-fetch.ts`
- Web fetch tool: `src/agents/tools/web-fetch.ts`
- RFC 2544 check: `src/shared/net/ip.ts` (line ~258) — `isBlockedSpecialUseIpv4Address` already respects `allowRfc2544BenchmarkRange`
