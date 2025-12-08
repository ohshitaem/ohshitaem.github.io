---
title: "Understanding `resource.resolver.mapping` in Apache Sling's JcrResourceResolverFactoryImpl"
date: 2025-12-08
updated: 2025-12-08
draft: false
type: post
tags: ["Apache Sling", "AEM", "OSGi", "ResourceResolver", "URL Rewriting", "JCR"]
---

# Understanding `resource.resolver.mapping` in Apache Sling's JcrResourceResolverFactoryImpl

When working with URL rewriting and resource resolution in Apache Sling and Adobe Experience Manager (AEM), the `resource.resolver.mapping` property is a powerful but often overlooked configuration option. This post explains what it does, how it works, and when to use it.

## What Is `resource.resolver.mapping`?

`resource.resolver.mapping` is an OSGi configuration property on `org.apache.sling.jcr.resource.internal.JcrResourceResolverFactoryImpl` that allows you to define **simple string-based path mappings** directly in OSGi configuration, rather than exclusively in the JCR content tree under `/etc/map`.

It is a multi-value string array where each entry defines a mapping rule that the Sling `ResourceResolver` applies when:
- Mapping a resource path to an external URL (outgoing links)
- Resolving a request URL to a resource path (incoming requests)

**Important:** Unlike JCR-based mappings, `resource.resolver.mapping` does **not support regular expressions**. It only supports simple path-based mappings.

## How It Works

The `JcrResourceResolverFactoryImpl` loads mappings from two sources:

1. **OSGi configuration** — rules defined in `resource.resolver.mapping` (simple string patterns, no regex)
2. **JCR content tree** — rules stored under the path specified by `resource.resolver.map.location` (default: `/etc/map`) (supports regex via `sling:match`)

Both sets of rules are compiled into internal mapping tables at startup and are treated uniformly at runtime. The ResourceResolver applies them top-down, transforming paths or URLs until a rule matches or the result stabilizes.

### Syntax and Format

Each `resource.resolver.mapping` entry is a simple path-based transformation rule. OSGi entries do **not** use regex syntax; instead, they define literal mappings. For example:

```
resource.resolver.mapping = [
  "/content/myapp/en.html:/",
  "/content/myapp/de.html:/de/"
]
```

These rules participate in both the **incoming mapping** (resolving request paths to resources) and **outgoing mapping** (converting resource paths to external URLs) tables.

### Important Limitation: No Regex Support

Unlike JCR-based mappings (which support `sling:match` with regex), `resource.resolver.mapping` **does not accept regular expressions**. If you need regex patterns with capture groups and complex transformations, you must use JCR-based mappings instead.

## OSGi Mappings vs. JCR Mappings

| Aspect | OSGi `resource.resolver.mapping` | JCR `/etc/map` |
|--------|-----------------------------------|----------------|
| **Pattern Type** | Simple string paths (no regex) | Regex patterns via `sling:match` |
| **Flexibility** | Limited; exact path matching only | Full regex support; complex hierarchies |
| **Storage** | Code/configuration | Content repository |
| **Host-specific rules** | Not supported | Supported via `sling:match` and tree structure |
| **Best for** | Environment-wide, static, simple rules | Multi-domain, complex, content-driven rules |
| **Versioning** | Code-as-config, deployable | Content packages, less versioned |

The Sling and AEM documentation generally recommends using `/etc/map` (configured via `resource.resolver.map.location`) for complex, multi-domain, or host-specific scenarios. This is primarily because JCR mappings offer greater flexibility, including support for regex expressions and domain-aware reverse mapping, which is crucial for multi-tenant setups and cross-linking between sites. Conversely, `resource.resolver.mapping` is ideal for simple, environment-wide mappings that can be version-controlled and deployed as code—but only when regex is not needed.

## Use Cases

**Use `resource.resolver.mapping` when you want to:**

- Apply simple, exact path mappings globally across the instance
- Keep mappings version-controlled and deployable as code
- Avoid storing mappings in the JCR

**Avoid `resource.resolver.mapping` (use `/etc/map` instead) when you need to:**

- Use regex patterns or capture groups
- Configure different rules per host or domain
- Manage complex, multi-level hierarchies of mappings
- Allow content authors to create or modify mappings without code deployment
- Support vanity URLs with per-domain variations

For complex mapping scenarios with regex, **always use JCR-based mappings** under `/etc/map` with `sling:match` properties instead.

## Impact on ResourceResolver Behavior

Both OSGi and JCR mappings influence how `ResourceResolver` behaves:

- **`ResourceResolver.map(...)`** — Converts resource paths to external URLs. OSGi and JCR mappings are applied, potentially generating short, SEO-friendly URLs for links in rendered pages.
- **`ResourceResolver.resolve(...)`** — Converts request URLs to resource paths. Incoming request URLs are matched against mappings to find the underlying resource.

Because changing `resource.resolver.mapping` reinitializes the `JcrResourceResolverFactoryImpl`, updates can trigger restarts of dependent components. Therefore, this property should be managed as controlled, as-code configuration rather than edited frequently at runtime.

## Practical Examples

### Simple OSGi Mapping (No Regex)

Suppose your AEM instance serves content under `/content/mysite/en/` but you want public URLs to map to `/en/`. You could add to your OSGi configuration:

```
resource.resolver.mapping = [
  "/content/mysite/en.html:/en/",
  "/content/mysite/de.html:/de/"
]
```

This maps the exact paths `/content/mysite/en.html` to `/en/` and `/content/mysite/de.html` to `/de/`.

### Complex Regex Mapping (Must Use JCR)

If you need to map all product URLs like `/content/mysite/products/shoes.html` to `/products/shoes.html` using a regex pattern like `^/content/mysite/(.*)$`, you **cannot** use `resource.resolver.mapping`. Instead, create a mapping under `/etc/map/http` in the JCR:

```
/etc/map/http/products
  - sling:resourceType: sling:Mapping
  - sling:match: ^/content/mysite/products/(.*)$
  - sling:target: /products/$1
```

This approach supports regex and is the correct way to handle complex URL transformations.

## Key Takeaways

- `resource.resolver.mapping` is an OSGi property for simple, code-based path mappings only.
- **It does NOT support regular expressions.** Use JCR-based mappings (`/etc/map`) if you need regex.
- OSGi mappings work alongside JCR mappings, and both are applied uniformly at runtime.
- Use `resource.resolver.mapping` only for simple, global, exact-path mappings; use `/etc/map` for complex, host-specific, or regex-based scenarios.
- Changes to this property reinitialize the resource resolver factory, so treat it as controlled infrastructure configuration.
- Both OSGi and JCR mappings affect both incoming request resolution and outgoing link generation.
- When in doubt about whether you need regex or complex transformations, use `/etc/map` instead.

---

**Further Reading:**
- [Apache Sling: Mappings for Resource Resolution](https://sling.apache.org/documentation/the-sling-engine/mappings-for-resource-resolution.html)
- [Adobe AEM: Resource Mapping Configuration](https://experienceleague.adobe.com/en/docs/experience-manager-65/content/implementing/deploying/configuring/resource-mapping)
- [AEM Concepts: Configuring Sling Mappings](https://aemconcepts.com/home/configuring-sling-mappings-for-multiple-development-environments-in-aemaacs/)