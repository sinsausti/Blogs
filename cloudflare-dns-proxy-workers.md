---
title: "Cloudflare for Personal Websites: DNS, Proxying, and Workers"
description: "A practical guide to Cloudflare DNS, proxying, CDN, security, and Workers, including Free and paid plan limits and upgrade recommendations."
author: "Sebastian Insausti"
date: "2026-09-04"
tags: ["Infrastructure", "Networking"]
canonical_url: "https://insaustis.com/blog/cloudflare-dns-proxy-workers.html"
---

# Cloudflare for Personal Websites: DNS, Proxying, and Workers

Cloudflare can be only your authoritative DNS provider, or it can sit in front of a website to provide TLS, caching, traffic filtering, analytics, and serverless code. Its Free plan is enough for many personal sites, but understanding where DNS ends and proxying or Workers begin prevents surprising behavior and costs.

> Cloudflare has two separate upgrade decisions: the plan attached to a domain and the Workers plan attached to the account.

## 1. Start with DNS

For the usual full setup, add the domain to Cloudflare, review every imported DNS record, and replace the authoritative nameservers at your registrar. You keep the domain registration and hosting where they are; only DNS authority moves to Cloudflare.

Before changing nameservers, export or record the current zone. Verify the website, mail records, DKIM, SPF, DMARC, and any domain-verification records after the change. Once the zone is active and resolving correctly, enable DNSSEC in Cloudflare and add the generated DS record at the registrar.

Cloudflare provides authoritative DNS without charging or capping DNS queries on Free, Pro, and Business plans. DNS analytics are also available, although retention and query windows depend on the plan. See the current [Cloudflare DNS FAQ](https://developers.cloudflare.com/dns/faq/) before planning around specific limits.

## 2. Understand the Orange Cloud

An A, AAAA, or CNAME record can be **Proxied** (orange cloud) or **DNS only** (gray cloud). With proxying enabled, visitors receive Cloudflare anycast addresses and HTTP/HTTPS traffic passes through Cloudflare. This enables CDN caching, Universal SSL, DDoS protection, WAF rules, redirects, and HTTP analytics while reducing direct exposure of the origin address.

DNS-only records return the real destination and bypass those HTTP features. Keep MX, TXT, domain-verification records, and unsupported non-HTTP services DNS-only. Do not assume the orange cloud protects an origin that remains reachable directly: restrict inbound access where practical and avoid publishing the same origin IP through unrelated DNS records.

Use **Full (strict)** TLS mode with a valid certificate on the origin. Flexible mode sends traffic from Cloudflare to the origin over HTTP and should not be used as a shortcut for missing origin TLS.

## 3. What Cloudflare Workers Are

The name you are looking for is **Cloudflare Workers**. A Worker is serverless code executed on Cloudflare's network when a request, scheduled task, queue message, or another supported event arrives. Common uses include redirects, lightweight APIs, authentication checks, header manipulation, request routing, webhooks, and scheduled automation.

Create and test a JavaScript Worker with Cloudflare's Wrangler CLI:

```
npm create cloudflare@latest -- my-first-worker
cd my-first-worker
npx wrangler dev
npx wrangler deploy
```

A minimal Worker can redirect an old path and pass every other request to the configured origin:

```
export default {
  async fetch(request) {
    const url = new URL(request.url);

    if (url.pathname === "/old-page") {
      return Response.redirect(`${url.origin}/new-page`, 301);
    }

    return fetch(request);
  }
};
```

Test locally, deploy first to the provided `workers.dev` hostname, and only then attach a production route or custom domain. Keep the project in Git and store secrets with Wrangler rather than in source code.

## 4. Free Versus Paid Website Plans

As of September 5, 2026, Cloudflare lists Free at $0, Pro at $20 per month when billed annually or $25 when billed monthly, and Business at $200 per month when billed annually or $250 when billed monthly. Prices and entitlements change, so verify the [current plan comparison](https://www.cloudflare.com/plans/) before purchasing.

- **Free:** suitable for personal, portfolio, lab, and low-risk sites. It includes DNS, CDN, Universal SSL, unmetered DDoS protection, and a basic managed WAF ruleset.
- **Pro:** useful for a professional site that needs deeper WAF and bot controls, more cache and configuration rules, image optimization, or better analytics and support options.
- **Business:** makes sense when the site generates important revenue or requires features such as a 100% uptime SLA, chat and ticket support, additional certificate flexibility, larger uploads, or partial CNAME setup.
- **Enterprise:** is a custom contract for mission-critical applications that need contractual support, advanced controls, higher limits, or organization-specific requirements.

Some capabilities—including Load Balancing, Argo, advanced certificates, and other products—can be separately billed add-ons. Upgrading the zone does not automatically include every Cloudflare product.

## 5. Workers Free Versus Workers Paid

Workers pricing is independent from the Free, Pro, or Business plan on a website. Workers Free currently allows 100,000 requests per day, 10 ms of CPU time per invocation, 128 MB of memory, 50 external subrequests per invocation, and five Cron Triggers per account. Waiting for a network request does not consume CPU time, but code execution does.

Workers Paid has a $5 monthly minimum. It includes 10 million requests and 30 million CPU milliseconds per month; additional usage is billed by requests and CPU time. It also raises execution, subrequest, Worker, and Cron Trigger limits. Static asset requests are free and unlimited under the current Standard pricing model.

Use the [Workers pricing](https://developers.cloudflare.com/workers/platform/pricing/) and [Workers limits](https://developers.cloudflare.com/workers/platform/limits/) pages as the source of truth. Configure CPU limits and billing notifications before moving an unbounded or externally triggered workload to a paid account.

## 6. When to Upgrade

Stay on Free while the site is non-critical, traffic fits comfortably within the limits, and the included security and caching controls meet the real requirement. Do not upgrade only because the site receives more DNS queries: DNS queries are not charged on the self-service website plans.

Consider a paid website plan when revenue, stronger managed security controls, support response, availability commitments, advanced caching, or certificate requirements justify the recurring cost. Upgrade Workers separately when you approach its request or CPU limits, need longer computation, more scheduled jobs or subrequests, or want a predictable production allowance.

## 7. Practical Recommendations

- Protect the Cloudflare account with phishing-resistant MFA and keep recovery codes offline.
- Use scoped API tokens instead of the Global API Key.
- Enable DNSSEC only after the zone is active, and coordinate DS changes when moving DNS providers.
- Use Full (strict) TLS and renew or monitor the origin certificate.
- Proxy web records, but keep mail and incompatible services DNS-only.
- Cache static content aggressively; cache authenticated or dynamic responses only with explicit rules.
- Store Worker code in Git, test routes outside production, and define rollback steps.
- Review analytics, WAF events, Worker errors, usage, and billing notifications regularly.

---

For a personal website, a strong starting point is Cloudflare Free with authoritative DNS, proxied web records, DNSSEC, Full (strict) TLS, and a small Worker for redirects or request handling. Pay when a measured technical or business requirement appears—not simply because a paid tier exists.
