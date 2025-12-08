---
title: "Understanding `resource.resolver.mapping` in Apache Sling's JcrResourceResolverFactoryImpl"
date: 2025-12-08
draft: false
type: post
tags: ["Apache Sling", "AEM", "OSGi", "ResourceResolver", "URL Rewriting", "JCR"]
---

# Understanding `resource.resolver.mapping` in Apache Sling's JcrResourceResolverFactoryImpl

When working with URL rewriting and resource resolution in Apache Sling and Adobe Experience Manager (AEM), the `resource.resolver.mapping` property is a powerful but often overlooked configuration option. This post explains what it does, how it works, and when to use it.

## What Is `resource.resolver.mapping`?

`resource.resolver.mapping` is an OSGi configuration property on `org.apache.sling.jcr.resource.internal.JcrResourceResolverFactoryImpl` that allows you to define regex-based URL rewrite rules directly in OSGi configuration, rather than exclusively in the JCR content tree under `/etc/map`.

It is a multi-value string array where each entry defines a mapping rule that the Sling `ResourceResolver` applies when:
- Mapping a resource path to an external URL (outgoing links)
- Resolving a request URL to a resource path (incoming requests)

## How It Works

The `JcrResourceResolverFactoryImpl` loads mappings from two sources:

1. **OSGi configuration** — rules defined in `resource.resolver.mapping`
2. **JCR content tree** — rules stored under the path specified by `resource.resolver.map.location` (default: `/etc/map`)

Both sets of rules are compiled into internal mapping tables at startup and are treated uniformly at runtime. The ResourceResolver applies them top-down, transforming paths or URLs until a rule matches or the result stabilizes.

### Syntax and Format

Each `resource.resolver.mapping` entry is a regex-based transformation rule, typically written as `"<regex><replacement>"`. For example:

```
"^/content/myapp/en\.html$" -> "/"
"^/content/myapp/(.*)\.html$" -> "/$1"
```

These rules participate in both the **incoming mapping** (resolving request paths to resources) and **outgoing mapping** (converting resource paths to external URLs) tables.

## OSGi Mappings vs. JCR Mappings

| Aspect | OSGi `resource.resolver.mapping` | JCR `/etc/map` |
|--------|-----------------------------------|----------------|
| **Flexibility** | Simple, global rules | Host-specific, complex hierarchies |
| **Storage** | Code/configuration | Content repository |
| **Host-specific rules** | Not supported | Supported via `sling:match` |
| **Best for** | Environment-wide, static rules | Multi-domain, content-driven rules |
| **Versioning** | Code-as-config, deployable | Content packages, less versioned |

The Sling and AEM documentation generally recommends using `/etc/map` (configured via `resource.resolver.map.location`) for complex or host-specific scenarios. However, `resource.resolver.mapping` is ideal when you want simple, environment-wide mappings kept entirely in code.

## Use Cases

**Use `resource.resolver.mapping` when you want to:**

- Strip `.html` extensions globally across the instance
- Shorten `/content/myapp/en.html` to `/en/`
- Apply simple regex transformations that apply to all hosts
- Keep mappings version-controlled and deployable as code

**Use `/etc/map` (JCR mappings) when you need to:**

- Configure different rules per host or domain
- Manage complex, multi-level hierarchies of mappings
- Allow content authors to create mappings without code deployment
- Support vanity URLs with per-domain variations

## Impact on ResourceResolver Behavior

Both OSGi and JCR mappings influence how `ResourceResolver` behaves:

- **`ResourceResolver.map(...)`** — Converts resource paths to external URLs. OSGi and JCR mappings are applied, potentially generating short, SEO-friendly URLs for links in rendered pages.
- **`ResourceResolver.resolve(...)`** — Converts request URLs to resource paths. Incoming request URLs are matched against mappings to find the underlying resource.

Because changing `resource.resolver.mapping` reinitializes the `JcrResourceResolverFactoryImpl`, updates can trigger restarts of dependent components. Therefore, this property should be managed as controlled, as-code configuration rather than edited frequently at runtime.

## Practical Example

Suppose your AEM instance serves content under `/content/mysite/en/` but you want public URLs to be `/en/`. You could add to your OSGi configuration:

```
resource.resolver.mapping = [
  "^/content/mysite/en/(.*)$" -> "/$1",
  "^/content/mysite/de/(.*)$" -> "/de/$1"
]
```

Now when Sling generates a link to `/content/mysite/en/products/shoes.html`, it becomes `/products/shoes.html` in the rendered HTML. Conversely, when a request comes in for `/products/shoes.html`, the resolver matches the reverse mapping and finds `/content/mysite/en/products/shoes.html`.

## Key Takeaways

- `resource.resolver.mapping` is a convenient way to define regex-based URL mappings in OSGi configuration rather than in the JCR.
- It works alongside JCR mappings under `/etc/map`, and both are applied uniformly at runtime.
- Use it for simple, global, code-based mappings; use `/etc/map` for complex, host-specific, or content-driven scenarios.
- Changes to this property reinitialize the resource resolver factory, so treat it as controlled infrastructure configuration.
- Both OSGi and JCR mappings affect both incoming request resolution and outgoing link generation.

---

**Further Reading:**
- [Apache Sling: Mappings for Resource Resolution](https://sling.apache.org/documentation/the-sling-engine/mappings-for-resource-resolution.html)
- [Adobe AEM: Resource Mapping Configuration](https://experienceleague.adobe.com/en/docs/experience-manager-65/content/implementing/deploying/configuring/resource-mapping)
- [AEM Concepts: Configuring Sling Mappings](https://aemconcepts.com/home/configuring-sling-mappings-for-multiple-development-environments-in-aemaacs/)
