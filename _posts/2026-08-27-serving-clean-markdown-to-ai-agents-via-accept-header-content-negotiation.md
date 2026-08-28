---
layout: post
title: Serving Clean Markdown to AI Agents via Accept Header Content Negotiation
date: 2026-08-27 21:36:28 -0400
description: 'Implement HTTP content negotiation with Accept: text/markdown to eliminate
  DOM clutter, lower fetch latency, and cut token costs for AI agents.'
categories:
- UI Engineering
- AI/ML
tags:
- http
- markdown
- content-negotiation
- web-infrastructure
- cdn
author: BIITS LLC
---

*Published August 27, 2026 at 9:36 PM ET*

When an AI agent or web-browsing model requests a page on your site, your server usually sends back hundreds of kilobytes of boilerplate HTML. That response includes client-side JavaScript bundles, navigation menus, consent modals, and layout wrappers long before reaching the core text. All of that structural overhead gets fed into an embedding pipeline or prompt context window, inflating token usage and slowing initial response times. Web servers can solve this problem elegantly without breaking standard browser behavior by leveraging HTTP content negotiation. By responding to requests containing an `Accept: text/markdown` header, your backend can deliver raw Markdown directly to automated clients while keeping standard HTML rendering intact for human visitors.

## How HTTP Content Negotiation Works for Markdown

Content negotiation is not a new concept in web architecture. The HTTP semantics specification in RFC 9110 defines how clients and servers agree on the best media format for a given resource. When a user requests a URL, the client sends an `Accept` request header specifying acceptable media types along with relative quality factors (q-values).

While browsers request `text/html`, modern agent tools can specifically request `text/markdown`, which is formally registered under RFC 7763. The interaction relies on standard HTTP request and response pairs:

```http
GET /posts/ui-engineering-patterns HTTP/1.1
Host: biits.dev
Accept: text/markdown, text/html;q=0.9
```

If your application server recognizes `text/markdown`, it returns the unadorned prose payload:

```http
HTTP/1.1 200 OK
Content-Type: text/markdown; charset=utf-8
Vary: Accept

# UI Engineering Patterns
...
```

If the server cannot provide `text/markdown` and strict content negotiation is enforced, it can respond with a `406 Not Acceptable` status code. In practice, most servers fall back to returning standard HTML rather than rejecting the request outright.

Testing this setup requires only simple command-line tools. You can inspect your headers or retrieve the raw payload directly with `curl`:

```bash
# Inspect response headers
curl -sI -H "Accept: text/markdown" https://example.com/article

# Fetch the raw Markdown body
curl -s -H "Accept: text/markdown" https://example.com/article
```

Initiatives like [acceptmarkdown.com](https://acceptmarkdown.com/) highlight how this protocol handshake enables sites to serve structured variants without creating secondary, agent-specific URL paths. Keeping a single URL canonical avoids splitting search index authority or complicating routing tables.

## Impact on Retrieval Latency and Token Economics

The mechanical benefit of serving raw Markdown over full HTML comes down to byte reduction and signal density. A typical modern web page often weighs between 500 KB and 2 MB once you factor in inline SVGs, metadata attributes, script tags, and complex DOM trees. In contrast, the underlying article text might only be 15 KB.

For Retrieval-Augmented Generation (RAG) applications, extra HTML boilerplate damages retrieval accuracy. Navigational links, sidebar widget text, and footers skew vector embeddings, diluting the actual relevance of the primary content. Dropping the layout components raises the overall signal-to-noise ratio.

Payload reduction translates directly into faster first-token generation. Because the agent receives a fraction of the raw bytes, network transfer time drops significantly. The automated parser skips DOM cleaning heuristics entirely, handing clean text to the model context window immediately.

## Caching Pitfalls and the Role of Vary Headers

Implementing content negotiation without updating your caching strategy will break your site for normal browser traffic. Reverse proxies and Content Delivery Networks (CDNs) cache responses by URL key. If an AI agent fetches `/about` with `Accept: text/markdown`, a naive cache stores that Markdown payload under the `/about` key. The next human visitor requesting `/about` in Chrome receives unstyled raw text instead of HTML.

To prevent cache poisoning, your server must include the `Vary: Accept` HTTP header in every response. This header instructs edge caches, CDN nodes, and downstream proxies to maintain separate cache entries for each unique `Accept` header value:

```http
HTTP/1.1 200 OK
Content-Type: text/markdown; charset=utf-8
Vary: Accept
```

Adding `Vary: Accept` causes some CDNs to bypass caching for non-standard Accept headers unless explicitly configured. Cloudflare and fast edge hosts often require tailored edge routing rules or Cloudflare Workers to partition cache keys by media type reliably without sacrificing global edge caching benefits.

## Integration Strategies Across Infrastructure and Frameworks

Implementing this mechanism depends on where your site renders content. If your web app uses a static site generator like Astro or Next.js with file-based Markdown content, serving the source format is straightforward.

In a Next.js App Router setup, middleware inspects incoming request headers and rewrites the request path to a dedicated API endpoint or static asset:

```typescript
// middleware.ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  const acceptHeader = request.headers.get('accept') || '';

  if (acceptHeader.includes('text/markdown')) {
    const url = request.nextUrl.clone();
    url.pathname = `/api/markdown${url.pathname}`;
    
    const response = NextResponse.rewrite(url);
    response.headers.set('Vary', 'Accept');
    return response;
  }

  return NextResponse.next();
}
```

For traditional proxy setups like Nginx, you can inspect incoming headers at the ingress controller and proxy agent requests to an upstream microservice or static directory:

```nginx
map $http_accept $variant {
    default       "html";
    "~text/markdown" "markdown";
}

server {
    listen 80;
    server_name example.com;

    location /articles/ {
        if ($variant = "markdown") {
            rewrite ^/articles/(.*)$ /markdown-exports/$1.md break;
        }
        try_files $uri $uri/ /index.html;
        add_header Vary Accept;
    }
}
```

Frameworks like Caddy, SvelteKit, Nuxt/Nitro, Apache, Express, Go, Django, WordPress, Discourse, Laravel, and Rails can adopt similar pattern matches in their routing layers or middleware stacks. The core requirement remains constant: check for `text/markdown`, render or load the plain text representation, set `Content-Type: text/markdown`, and always attach `Vary: Accept`.

## Tradeoffs and Maintenance Realities

While serving Markdown via HTTP content negotiation is clean in theory, practical deployment introduces maintainability challenges that teams must evaluate upfront.

If your site is built on a traditional database-backed CMS where articles are saved exclusively as compiled HTML or rich text blocks, generating clean Markdown on every incoming request introduces runtime CPU overhead. Converting HTML to Markdown using server-side libraries like Turndown can consume more server resources than serving pre-rendered HTML out of memory. Without an effective caching tier for generated Markdown, high agent traffic spikes server load.

Another issue is structural drift. When maintaining separate rendering pathways (one for React or Vue components and one for plain Markdown export), interactive UI elements like tabbed code blocks, callout containers, and custom embedded charts can easily lose meaning or get dropped entirely in the text representation. If your Markdown renderer drops critical data that exists in your visual UI, the AI agent receives an incomplete version of your content.

Agent implementation adherence remains uneven across the ecosystem. While custom crawler frameworks and automated research tools honor `Accept: text/markdown`, some commercial browsing models still send generic `Accept: */*` headers. If an agent fails to declare its capability, content negotiation falls back to standard HTML, forfeiting the performance optimization entirely.

## Further reading

- [https://acceptmarkdown.com/](https://acceptmarkdown.com/)
- [https://github.com/drumih/turbo-fieldfare](https://github.com/drumih/turbo-fieldfare)
- [https://www.docker.com/products/docker-sandboxes/](https://www.docker.com/products/docker-sandboxes/)
- [https://scalex.dev/blog/ai-agent-permissions-stats/](https://scalex.dev/blog/ai-agent-permissions-stats/)
- [https://www.codewithbullet.com](https://www.codewithbullet.com)
- [https://arxiv.org/abs/2608.02412](https://arxiv.org/abs/2608.02412)
- [https://line9.ai/diagram](https://line9.ai/diagram)
- [https://hoplite.sh](https://hoplite.sh)

