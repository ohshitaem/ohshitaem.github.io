---
title: "A Deep Dive into Apache Sling's Resource Resolver Mappings"
date: 2025-12-08
updated: 2025-12-08
draft: false
type: post
tags: ["Apache Sling", "AEM", "OSGi", "ResourceResolver", "URL Rewriting", "JCR"]
---

# A Deep Dive into Apache Sling's Resource Resolver Mappings

When working with URL rewriting and resource resolution in Apache Sling and Adobe Experience Manager (AEM), developers have two primary mechanisms for defining mapping rules: the JCR-based configuration under `/etc/map` and the OSGi-based `resource.resolver.mapping` property. This post explains what they do, how they work together, and when to use each.

## What Are The Mapping Mechanisms?

1.  **`resource.resolver.map.location`**: An OSGi configuration property on `org.apache.sling.jcr.resource.internal.JcrResourceResolverFactoryImpl` that points to a path in the JCR where mapping rules are defined. The default and conventional path is `/etc/map`. This mechanism is powerful, supporting regular expressions and complex hierarchical rule sets.

2.  **`resource.resolver.mapping`**: A multi-value `String[]` OSGi property on the same factory. It allows you to define **simple, literal string-based path mappings** directly in your configuration, which is ideal for environment-specific, code-managed rules.

Both mechanisms are used by the Sling `ResourceResolver` to:
- Map a resource path to an external URL (outgoing links).
- Resolve a request URL to a resource path (incoming requests).

## How It Works

The `JcrResourceResolverFactoryImpl` loads mappings from both sources at startup or upon configuration change.

1.  **OSGi configuration** — rules defined in `resource.resolver.mapping` (simple string patterns, no regex).
2.  **JCR content tree** — rules stored under the path specified by `resource.resolver.map.location` (supports regex via `sling:match`).

These rules are compiled into a single, unified list of `MapEntry` objects.

### The Regular Expression Engine

When regular expressions are used (exclusively via `sling:match` in the JCR), Sling uses Java's standard regex engine, `java.util.regex.Pattern`. This means any pattern compatible with Java's regex flavor is supported.

### Evaluation Order

A common point of confusion is how rules from different sources are prioritized. The key is that **all rules are merged into a single list and sorted**, with no inherent precedence given to the source (OSGi vs. JCR). The sorting logic is defined in the [`MapEntry.java`](https://github.com/apache/sling-org-apache-sling-resourceresolver/blob/org.apache.sling.resourceresolver-1.10.0/src/main/java/org/apache/sling/resourceresolver/impl/mapping/MapEntry.java)'s `compareTo` method and follows these criteria:

1.  **Pattern Length (Descending)**: Longer, more-specific patterns are always evaluated before shorter, more-general ones. For example, a rule for `/content/myapp/en/` will be checked before a rule for `/content/myapp/`. This is the most critical factor in determining order.
2.  **Lexicographical Order**: If two patterns have the same length, they are sorted alphabetically.
3.  **Vanity Order**: For vanity paths with identical patterns, the `sling:vanityOrder` property is used as a final tie-breaker.

This unified and sorted list is then traversed top-down at runtime until a rule matches.

## A Deeper Look at `sling:` Directives

The behavior of JCR-based mappings under `/etc/map` is controlled by several `sling:` properties:

-   `sling:match`: Defines a **regular expression** for matching the incoming request URL or path. If this property is absent, the node's name is used as a literal prefix match. This is the primary way to enable regex-based matching.

-   `sling:internalRedirect`: Specifies the target path for an **internal rewrite**. The resource resolver transparently continues the resolution process using the new path without informing the client. It can be a multi-valued property (`String[]`) to provide multiple candidate paths. This is signaled internally by a `MapEntry` with a status of `-1`.

-   `sling:redirect`: Specifies a target URL for an **external redirect**. This causes the resolver to issue an HTTP 3xx response to the client, telling it to navigate to the new URL.

-   `sling:status`: Used with `sling:redirect` to specify the HTTP status code for the external redirect (e.g., `301` for permanent, `302` for temporary).

-   `sling:alias`: Set on a content resource (not a mapping rule) to provide an alternative, "friendly" name for that resource. This is used for generating outgoing links and can also be used to resolve incoming requests. For example, a resource at `/content/products/p-001` could have an alias `shiny-red-ball`, allowing it to be accessed via `/content/products/shiny-red-ball`.

-   `sling:vanityPath` & `sling:vanityOrder`: `sling:vanityPath` is set on a content resource to claim a user-friendly URL. These are loaded into the mapping tables as internal redirects. `sling:vanityOrder` (a `Long`) resolves conflicts if multiple resources claim the same vanity path.

## Hierarchical Mapping Structures

One of the most powerful features of JCR-based mappings is the ability to organize `sling:Mapping` nodes hierarchically under `/etc/map`. The hierarchy is not merely organizational—it directly determines how URL patterns are constructed and matched.

### How Hierarchies Create Patterns

The mapping resolution engine processes the JCR tree through a recursive, depth-first traversal implemented in the [`gather` method of `MapEntries.java`](https://github.com/apache/sling-org-apache-sling-resourceresolver/blob/org.apache.sling.resourceresolver-1.10.0/src/main/java/org/apache/sling/resourceresolver/impl/mapping/MapEntries.java). As it descends the tree, it **progressively builds URL patterns by concatenating path segments** from each level of the hierarchy.

For each node in the tree:

1. **Pattern Segment Determination**: The node contributes a segment to the pattern based on:
    - The value of its `sling:match` property (if present), or
    - Its node name followed by `/` (if `sling:match` is absent)

2. **Pattern Concatenation**: This segment is appended to the pattern inherited from its parent node, creating a cumulative pattern that reflects the full path from the root mapping node.

3. **Recursive Processing**: Before creating a mapping entry for the current node, the engine recursively processes all of its child nodes with the current cumulative pattern as their parent path.

4. **Entry Creation**: After all children have been processed, a `MapEntry` is created for the current node using its fully constructed pattern.

### Example: Building Patterns from Hierarchy

Consider this JCR structure:

```
/etc/map/
  +-- http/
       +-- my.host.com/
            - sling:internalRedirect = "/content/mysite"
            +-- products/
                 - sling:match = "(.+)\\.html$"
                 - sling:internalRedirect = "/content/mysite/products/$1"
```

**Pattern Construction Process:**

1. Starting at `/etc/map/http` with base pattern `http/`
2. Processing child `my.host.com`:
    - No `sling:match`, so uses node name
    - Pattern becomes: `http/my.host.com/`
    - Creates entry: `^http/my.host.com/` → `/content/mysite`
3. Processing grandchild `products`:
    - Has `sling:match = "(.+)\\.html$"`
    - Pattern becomes: `http/my.host.com/(.+)\\.html$`
    - Creates entry: `^http/my.host.com/(.+)\\.html$` → `/content/mysite/products/$1`

This hierarchical approach enables you to:
- **Organize rules by domain or host** (e.g., `/etc/map/http/example.com/`, `/etc/map/https/other.com/`)
- **Share common path prefixes** across multiple rules
- **Inherit context** from parent nodes while adding specificity at child levels
- **Create complex multi-segment patterns** without repeating the full pattern in each rule

The resulting patterns from all levels are then added to the unified list of `MapEntry` objects and sorted by specificity for runtime evaluation.

## Handling Multivalued Directives

Several of the `sling:` properties can be defined as a multi-valued string array (`String[]`), which affects how they are processed.

### `sling:internalRedirect`

When a mapping node has a multivalued `sling:internalRedirect` property, it provides a list of potential target paths for the rewrite. While multiple paths can be defined, the resource resolver's implementation will use the **first successful path** generated from the array for continuing the resolution process. This provides a fallback mechanism, but not a one-to-many rewrite for a single request.

### `sling:alias`

A content resource can have multiple aliases. This has two main effects:

-   **Incoming Resolution**: Any of the defined aliases can be used in a URL to resolve to the resource. For example, if a resource has aliases `foo` and `bar`, both `/content/mypage/foo` and `/content/mypage/bar` would resolve to the same underlying resource.
-   **Outgoing Mapping**: When generating URLs for the resource (e.g., via `ResourceMapper.getAllMappings()`), all its aliases are considered valid "friendly" names and will be included in the list of possible mappings.

### `sling:vanityPath`

Each entry in a multivalued `sling:vanityPath` property creates a separate, independent mapping rule. This allows a single content resource to be accessible from multiple distinct vanity URLs. For example, a resource can have `sling:vanityPath` set to `["/my-product", "/products/my-product"]`, and both URLs will resolve to it.

## OSGi Mappings vs. JCR Mappings

| Aspect                | OSGi `resource.resolver.mapping`        | JCR `/etc/map`                                       |
| --------------------- | --------------------------------------- | ---------------------------------------------------- |
| **Pattern Type**      | Simple string paths (no regex)          | Regex patterns via `sling:match`                     |
| **Flexibility**       | Limited; exact path matching only       | Full regex support; complex hierarchies              |
| **Storage**           | Code/configuration (`.cfg.json`)        | Content repository (JCR)                             |
| **Host-specific rules** | Not supported                           | Supported via `sling:match` and tree structure       |
| **Best for**          | Environment-wide, static, simple rules  | Multi-domain, complex, content-driven rules          |
| **Versioning**        | Code-as-config, deployable              | Content packages, less versioned                     |

## Use Cases

**Use `resource.resolver.mapping` when you want to:**

- Apply simple, exact path mappings globally across the instance.
- Keep mappings version-controlled and deployable as code.
- Avoid storing critical infrastructure configuration in the JCR.

**Avoid `resource.resolver.mapping` (use `/etc/map` instead) when you need to:**

- Use regex patterns or capture groups (``).
- Configure different rules per host or domain.
- Manage complex, multi-level hierarchies of mappings.
- Allow content authors to create or modify mappings without a code deployment.

## Key Takeaways

-   Sling offers two ways to define mappings: the OSGi property `resource.resolver.mapping` for simple, literal rules, and the JCR path `resource.resolver.map.location` (e.g., `/etc/map`) for complex, regex-enabled rules.
-   **Regular expressions are only supported in JCR-based mappings** via the `sling:match` property.
-   All rules are merged and **sorted by pattern length (descending)**, meaning specificity, not origin, determines the evaluation order.
-   A rich set of `sling:` properties (`sling:internalRedirect`, `sling:redirect`, `sling:alias`, etc.) provides fine-grained control over resolution behavior.
-   When in doubt, prefer JCR-based mappings under `/etc/map` for their flexibility and power.

---

**Further Reading:**
- [Apache Sling: Mappings for Resource Resolution](https://sling.apache.org/documentation/the-sling-engine/mappings-for-resource-resolution.html)
- [Apache Sling Resource Resolver Source Code (Version 1.10.0)](https://github.com/apache/sling-org-apache-sling-resourceresolver/tree/org.apache.sling.resourceresolver-1.10.0)
- [Adobe AEM: Resource Mapping Configuration](https://experienceleague.adobe.com/en/docs/experience-manager-65/content/implementing/deploying/configuring/resource-mapping)