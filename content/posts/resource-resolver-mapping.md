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
    *   *Code Reference*: Defined in `ResourceResolverFactoryConfig.java` and used in `ResourceResolverFactoryActivator.java` ([L208](https://github.com/apache/sling-org-apache-sling-resourceresolver/blob/org.apache.sling.resourceresolver-1.10.0/src/main/java/org/apache/sling/resourceresolver/impl/ResourceResolverFactoryActivator.java#L208)). Default value `DEFAULT_MAP_ROOT` in `MapEntries.java` ([L98](https://github.com/apache/sling-org-apache-sling-resourceresolver/blob/org.apache.sling.resourceresolver-1.10.0/src/main/java/org/apache/sling/resourceresolver/impl/mapping/MapEntries.java#L98)) is `/etc/map`.

2.  **`resource.resolver.mapping`**: A multi-value `String[]` OSGi property on the same factory. It allows you to define **simple, literal string-based path mappings** directly in your configuration, which is ideal for environment-specific, code-managed rules.
    *   *Code Reference*: Defined in `ResourceResolverFactoryConfig.java` and used in `ResourceResolverFactoryActivator.java` ([L181](https://github.com/apache/sling-org-apache-sling-resourceresolver/blob/org.apache.sling.resourceresolver-1.10.0/src/main/java/org/apache/sling/resourceresolver/impl/ResourceResolverFactoryActivator.java#L181)). The `Mapping` class handles these mappings via string prefixes, not regex, as seen in `Mapping.java` methods like `mapUri` ([L91](https://github.com/apache/sling-org-apache-sling-resourceresolver/blob/org.apache.sling.resourceresolver-1.10.0/src/main/java/org/apache/sling/resourceresolver/impl/mapping/Mapping.java#L91)) and `mapHandle` ([L107](https://github.com/apache/sling-org-apache-sling-resourceresolver/blob/org.apache.sling.resourceresolver-1.10.0/src/main/java/org/apache/sling/resourceresolver/impl/mapping/Mapping.java#L107)).

Both mechanisms are used by the Sling `ResourceResolver` to:
- Map a resource path to an external URL (outgoing links).
- Resolve a request URL to a resource path (incoming requests).

## How It Works

The `JcrResourceResolverFactoryImpl` (implemented by `ResourceResolverFactoryActivator`) loads mappings from both sources at startup or upon configuration change.
*   *Code Reference*: `ResourceResolverFactoryActivator.java`'s `activate` method ([L171](https://github.com/apache/sling-org-apache-sling-resourceresolver/blob/org.apache.sling.resourceresolver-1.10.0/src/main/java/org/apache/sling/resourceresolver/impl/ResourceResolverFactoryActivator.java#L171)) handles initial loading, and the `modified` method ([L359](https://github.com/apache/sling-org-apache-sling-resourceresolver/blob/org.apache.sling.resourceresolver-1.10.0/src/main/java/org/apache/sling/resourceresolver/impl/ResourceResolverFactoryActivator.java#L359)) ensures reload on configuration changes.

1.  **OSGi configuration** — rules defined in `resource.resolver.mapping` (simple string patterns, no regex).
2.  **JCR content tree** — rules stored under the path specified by `resource.resolver.map.location` (supports regex via `sling:match`).

These rules are compiled into a single, unified list of `MapEntry` objects.
*   *Code Reference*: `MapEntries.java` extensively uses and creates `MapEntry` objects, e.g., in `gather` ([L752](https://github.com/apache/sling-org-apache-sling-resourceresolver/blob/org.apache.sling.resourceresolver-1.10.0/src/main/java/org/apache/sling/resourceresolver/impl/mapping/MapEntries.java#L752)) and `addEntry` ([L806](https://github.com/apache/sling-org-apache-sling-resourceresolver/blob/org.apache.sling.resourceresolver-1.10.0/src/main/java/org/apache/sling/resourceresolver/impl/mapping/MapEntries.java#L806)).

### The Regular Expression Engine

When regular expressions are used (exclusively via `sling:match` in the JCR), Sling uses Java's standard regex engine, `java.util.regex.Pattern`. This means any pattern compatible with Java's regex flavor is supported.
*   *Code Reference*: `MapEntry.java` uses `java.util.regex.Pattern` for `urlPattern` ([L64](https://github.com/apache/sling-org-apache-sling-resourceresolver/blob/org.apache.sling.resourceresolver-1.10.0/src/main/java/org/apache/sling/resourceresolver/impl/mapping/MapEntry.java#L64)) and `java.util.regex.Matcher` for matching ([L195](https://github.com/apache/sling-org-apache-sling-resourceresolver/blob/org.apache.sling.resourceresolver-1.10.0/src/main/java/org/apache/sling/resourceresolver/impl/mapping/MapEntry.java#L195)).

### Evaluation Order

A common point of confusion is how rules from different sources are prioritized. The key is that **all rules are merged into a single list and sorted**, with no inherent precedence given to the source (OSGi vs. JCR). The sorting logic is defined in the [`MapEntry.java`](https://github.com/apache/sling-org-apache-sling-resourceresolver/blob/org.apache.sling.resourceresolver-1.10.0/src/main/java/org/apache/sling/resourceresolver/impl/mapping/MapEntry.java#L241)'s `compareTo` method and follows these criteria:

1.  **Pattern Length (Descending)**: Longer, more-specific patterns are always evaluated before shorter, more-general ones. For example, a rule for `/content/myapp/en/` will be checked before a rule for `/content/myapp/`. This is the most critical factor in determining order.
    *   *Code Reference*: `MapEntry.java` `compareTo` ([L247-L251](https://github.com/apache/sling-org-apache-sling-resourceresolver/blob/org.apache.sling.resourceresolver-1.10.0/src/main/java/org/apache/sling/resourceresolver/impl/mapping/MapEntry.java#L247-L251)).
2.  **Lexicographical Order**: If two patterns have the same length, they are sorted alphabetically.
    *   *Code Reference*: `MapEntry.java` `compareTo` ([L255](https://github.com/apache/sling-org-apache-sling-resourceresolver/blob/org.apache.sling.resourceresolver-1.10.0/src/main/java/org/apache/sling/resourceresolver/impl/mapping/MapEntry.java#L255)).
3.  **Vanity Order**: For vanity paths with identical patterns, the `sling:vanityOrder` property is used as a final tie-breaker.
    *   *Code Reference*: `MapEntry.java` `compareTo` ([L256-L260](https://github.com/apache/sling-org-apache-sling-resourceresolver/blob/org.apache.sling.resourceresolver-1.10.0/src/main/java/org/apache/sling/resourceresolver/impl/mapping/MapEntry.java#L256-L260)).

This unified and sorted list is then traversed top-down at runtime until a rule matches.

## Node Structures for Resolver and Mapping Entries

Before diving into the specific `sling:` directives, it's important to understand how nodes are structured to create the two types of map entries: **Resolver Map Entries** (for incoming requests) and **Mapping Map Entries** (for outgoing URLs).

### Resolver Map Entries (Incoming: URL → Resource)

These entries map incoming request URLs to resource paths in the repository. They are created from two primary sources:

#### A. From `/etc/map` Nodes

Nodes are created hierarchically under `/etc/map` to represent the URL structure. The node type is typically `sling:Mapping`, though any node type with the correct properties will work.

**Example Structure:**

```
/etc/map/
  +-- http/  (sling:Mapping)
       +-- my.host.com.8080/  (sling:Mapping)
            - sling:internalRedirect = "/content/mysite"
            +-- products/  (sling:Mapping)
                 - sling:match = "product-(.*)"
                 - sling:internalRedirect = "/content/products/$1"
            +-- about-us.html  (sling:Mapping)
                 - sling:redirect = "https://company.com/about"
                 - sling:status = 301
```

**How Patterns are Formed:**

The resource resolver traverses this tree, and the path to a node, combined with its `sling:match` property (or the node name if `sling:match` is absent), creates the URL pattern.

- `/etc/map/http/my.host.com.8080` creates the pattern `http/my.host.com.8080/`
- The child node `products` with `sling:match = "product-(.*)"` creates the full pattern `http/my.host.com.8080/product-(.*)`

**Key Properties:**

- **`sling:internalRedirect`** (String or String[]): Specifies the repository path for an internal rewrite. The resolver transparently uses this new path to continue the resolution process.
- **`sling:redirect`** (String): Specifies a target URL for an external redirect, instructing the client to navigate to a new URL via an HTTP 3xx response.
- **`sling:status`** (Long): Used with `sling:redirect` to specify the HTTP status code (e.g., 301 for permanent, 302 for temporary). Defaults to 302 if omitted.
- **`sling:match`** (String): An optional regular expression used for a segment of the URL instead of the node's name, allowing for flexible pattern matching.

#### B. From Vanity Paths on Content

Vanity paths are defined directly on content nodes to make them accessible via "friendly" URLs.

**Example Structure:**

```
/content/mysite/en/products/
  +-- my-product/  (cq:Page)
       +-- jcr:content/
            - sling:vanityPath = "/my-product"
            - sling:redirect = true  (optional)
            - sling:redirectStatus = 301  (optional)
```

**Key Properties:**

- **`sling:vanityPath`** (String or String[]): Required. The desired user-friendly URL path(s). Creates an internal redirect from the vanity path to the resource.
- **`sling:redirect`** (Boolean): Optional. If true, the vanity path will issue an external redirect to the resource's mapped path instead of an internal rewrite.
- **`sling:redirectStatus`** (Long): Optional. Specifies the HTTP status code for the external redirect.
- **`sling:vanityOrder`** (Long): Optional. A numerical value to determine precedence when multiple resources claim the same vanity path. Higher values have higher precedence.

### Mapping Map Entries (Outgoing: Resource Path → URL)

These entries are used to generate public-facing URLs from repository paths. They are created from `/etc/map` nodes and `sling:alias` properties on content.

#### A. From `/etc/map` Nodes

The same node structure under `/etc/map` is used, but the `sling:internalRedirect` property is interpreted as the **source** (the resource path to match), and the node's path in the hierarchy is interpreted as the **destination** (the URL to generate).

**Example Structure:**

```
/etc/map/
  +-- http/
       +-- my.host.com.8080/
            - sling:internalRedirect = "/content/mysite"
```

This rule maps the resource path `/content/mysite` to the URL `http://my.host.com:8080/`.

**Key Properties:**

- **`sling:internalRedirect`** (String or String[]): Required. This property now defines the resource path pattern to be mapped.
- The node's path (e.g., `http/my.host.com.8080`) defines the output URL prefix.

**Note:** Rules with `sling:redirect` (external redirects) and regex in `sling:match` are ignored when building the mapping table.

**Note:** Rules with `sling:redirect` (external redirects) and regex in `sling:match` are ignored when building the mapping table.

**Critical Limitation - No Variables in Outgoing Mappings:**

There is a significant limitation on the syntax of `sling:internalRedirect` when creating Mapping Map Entries. The values **cannot contain variables from regular expression capture groups** (e.g., `$1`, `$2`).

Here's why:

1. **Role Reversal**: For incoming resolution (URL → Resource), `sling:match` captures parts of the URL, and `sling:internalRedirect` can use `$1` to insert those captured parts into the resource path. For outgoing mapping (Resource Path → URL), the roles are reversed - the value of `sling:internalRedirect` becomes the pattern to match against a resource's path, and the URL is the replacement.

2. **No Capture Groups Available**: When mapping an outgoing path like `/content/mysite/about.html`, there is no regular expression capture group in play. The path is a literal string. A pattern in `sling:internalRedirect` like `/content/$1/about.html` would be meaningless because there is no source for the `$1` variable.

3. **Code Enforcement**: The logic in `MapEntry.java`'s `createMapEntry` method explicitly prevents this by skipping any `sling:internalRedirect` values that contain a `$` character:
   ```java
   final String[] internalRedirect = props.get(ResourceResolverImpl.PROP_REDIRECT_INTERNAL, String[].class);
   if (internalRedirect != null) {
       for (final String redir : internalRedirect) {
           if (!redir.contains("$")) { // Enforces the limitation
               // create the MapEntry
           }
       }
   }
   ```

**In summary**: For creating outgoing Mapping Map Entries, the `sling:internalRedirect` property must contain only **literal paths** that you want to map from. Any entry containing a `$` will be ignored for the purpose of outgoing URL mapping.

#### B. From Aliases on Content

Aliases provide alternative, "friendly" names for resources within their parent, which affects outgoing URL generation.

**Example Structure:**

```
/content/mysite/en/
  +-- about_us/  (cq:Page)
       +-- jcr:content/
            - sling:alias = "about"
```

When mapping the resource `/content/mysite/en/about_us`, the resource resolver can generate the path `/content/mysite/en/about`.

**Key Properties:**

- **`sling:alias`** (String or String[]): Required. An alternative name or set of names for the resource. This name will be used instead of the actual node name when constructing a URL.

## Node Structures for Resolver and Mapping Entries

Before diving into the specific `sling:` directives, it's important to understand how nodes are structured to create the two types of map entries: **Resolver Map Entries** (for incoming requests) and **Mapping Map Entries** (for outgoing URLs).

### Resolver Map Entries (Incoming: URL → Resource)

These entries map incoming request URLs to resource paths in the repository. They are created from two primary sources:

#### A. From `/etc/map` Nodes

Nodes are created hierarchically under `/etc/map` to represent the URL structure. The node type is typically `sling:Mapping`, though any node type with the correct properties will work.

**Example Structure:**

```
/etc/map/
  +-- http/  (sling:Mapping)
       +-- my.host.com.8080/  (sling:Mapping)
            - sling:internalRedirect = "/content/mysite"
            +-- products/  (sling:Mapping)
                 - sling:match = "product-(.*)"
                 - sling:internalRedirect = "/content/products/$1"
            +-- about-us.html  (sling:Mapping)
                 - sling:redirect = "https://company.com/about"
                 - sling:status = 301
```

**How Patterns are Formed:**

The resource resolver traverses this tree, and the path to a node, combined with its `sling:match` property (or the node name if `sling:match` is absent), creates the URL pattern.

- `/etc/map/http/my.host.com.8080` creates the pattern `http/my.host.com.8080/`
- The child node `products` with `sling:match = "product-(.*)"` creates the full pattern `http/my.host.com.8080/product-(.*)`

**Key Properties:**

- **`sling:internalRedirect`** (String or String[]): Specifies the repository path for an internal rewrite. The resolver transparently uses this new path to continue the resolution process.
- **`sling:redirect`** (String): Specifies a target URL for an external redirect, instructing the client to navigate to a new URL via an HTTP 3xx response.
- **`sling:status`** (Long): Used with `sling:redirect` to specify the HTTP status code (e.g., 301 for permanent, 302 for temporary). Defaults to 302 if omitted.
- **`sling:match`** (String): An optional regular expression used for a segment of the URL instead of the node's name, allowing for flexible pattern matching.

#### B. From Vanity Paths on Content

Vanity paths are defined directly on content nodes to make them accessible via "friendly" URLs.

**Example Structure:**

```
/content/mysite/en/products/
  +-- my-product/  (cq:Page)
       +-- jcr:content/
            - sling:vanityPath = "/my-product"
            - sling:redirect = true  (optional)
            - sling:redirectStatus = 301  (optional)
```

**Key Properties:**

- **`sling:vanityPath`** (String or String[]): Required. The desired user-friendly URL path(s). Creates an internal redirect from the vanity path to the resource.
- **`sling:redirect`** (Boolean): Optional. If true, the vanity path will issue an external redirect to the resource's mapped path instead of an internal rewrite.
- **`sling:redirectStatus`** (Long): Optional. Specifies the HTTP status code for the external redirect.
- **`sling:vanityOrder`** (Long): Optional. A numerical value to determine precedence when multiple resources claim the same vanity path. Higher values have higher precedence.

### Mapping Map Entries (Outgoing: Resource Path → URL)

These entries are used to generate public-facing URLs from repository paths. They are created from `/etc/map` nodes and `sling:alias` properties on content.

#### A. From `/etc/map` Nodes

The same node structure under `/etc/map` is used, but the `sling:internalRedirect` property is interpreted as the **source** (the resource path to match), and the node's path in the hierarchy is interpreted as the **destination** (the URL to generate).

**Example Structure:**

```
/etc/map/
  +-- http/
       +-- my.host.com.8080/
            - sling:internalRedirect = "/content/mysite"
```

This rule maps the resource path `/content/mysite` to the URL `http://my.host.com:8080/`.

**Key Properties:**

- **`sling:internalRedirect`** (String or String[]): Required. This property now defines the resource path pattern to be mapped.
- The node's path (e.g., `http/my.host.com.8080`) defines the output URL prefix.

**Note:** Rules with `sling:redirect` (external redirects) and regex in `sling:match` are ignored when building the mapping table.

#### B. From Aliases on Content

Aliases provide alternative, "friendly" names for resources within their parent, which affects outgoing URL generation.

**Example Structure:**

```
/content/mysite/en/
  +-- about_us/  (cq:Page)
       +-- jcr:content/
            - sling:alias = "about"
```

When mapping the resource `/content/mysite/en/about_us`, the resource resolver can generate the path `/content/mysite/en/about`.

**Key Properties:**

- **`sling:alias`** (String or String[]): Required. An alternative name or set of names for the resource. This name will be used instead of the actual node name when constructing a URL.

## A Deeper Look at `sling:` Directives

The behavior of JCR-based mappings under `/etc/map` is controlled by several `sling:` properties:

### `sling:match` 
- Defines a **regular expression** for matching the incoming request URL or path. If this property is absent, the node's name is used as a literal prefix match. This is the primary way to enable regex-based matching.
    *   *Code Reference*: `MapEntries.java` `PROP_REG_EXP` ([L82](https://github.com/apache/sling-org-apache-sling-resourceresolver/blob/org.apache.sling.resourceresolver-1.10.0/src/main/java/org/apache/sling/resourceresolver/impl/mapping/MapEntries.java#L82)).

### `sling:internalRedirect` 
- Specifies the target path for an **internal rewrite**. The resource resolver transparently continues the resolution process using the new path without informing the client. It can be a multi-valued property (`String[]`) to provide multiple candidate paths. This is signaled internally by a `MapEntry` with a status of `-1`.
    *   *Code Reference*: Used in `MapEntry.java` `createResolveEntry` to create a `MapEntry` with status `-1` ([L133](https://github.com/apache/sling-org-apache-sling-resourceresolver/blob/org.apache.sling.resourceresolver-1.10.0/src/main/java/org/apache/sling/resourceresolver/impl/mapping/MapEntry.java#L133)).

### `sling:redirect`: 
- Specifies a target URL for an **external redirect**. This causes the resolver to issue an HTTP 3xx response to the client, telling it to navigate to the new URL.
    *   *Code Reference*: `MapEntries.java` `PROP_REDIRECT_EXTERNAL` ([L84](https://github.com/apache/sling-org-apache-sling-resourceresolver/blob/org.apache.sling.resourceresolver-1.10.0/src/main/java/org/apache/sling/resourceresolver/impl/mapping/MapEntries.java#L84)).

### `sling:status` 
- Used with `sling:redirect` to specify the HTTP status code for the external redirect (e.g., `301` for permanent, `302` for temporary). For vanity paths, the property `sling:redirectStatus` is specifically used to define the status code.
    *   *Code Reference*: `MapEntries.java` `PROP_REDIRECT_EXTERNAL_STATUS` ([L86](https://github.com/apache/sling-org-apache-sling-resourceresolver/blob/org.apache.sling.resourceresolver-1.10.0/src/main/java/org/apache/sling/resourceresolver/impl/mapping/MapEntries.java#L86)) and `PROP_REDIRECT_EXTERNAL_REDIRECT_STATUS` ([L88](https://github.com/apache/sling-org-apache-sling-resourceresolver/blob/org.apache.sling.resourceresolver-1.10.0/src/main/java/org/apache/sling/resourceresolver/impl/mapping/MapEntries.java#L88)). In `loadVanityPath`, `sling:redirectStatus` is used ([L911](https://github.com/apache/sling-org-apache-sling-resourceresolver/blob/org.apache.sling.resourceresolver-1.10.0/src/main/java/org/apache/sling/resourceresolver/impl/mapping/MapEntries.java#L911)).

### `sling:alias`
- Set on a content resource (not a mapping rule) to provide an alternative, "friendly" name for that resource. This is used for generating outgoing links and can also be used to resolve incoming requests. For example, a resource at `/content/products/p-001` could have an alias `shiny-red-ball`, allowing it to be accessed via `/content/products/shiny-red-ball`.
    *   *Code Reference*: Referenced as `ResourceResolverImpl.PROP_ALIAS` and handled in `MapEntries.java` in methods like `doAddAlias` ([L480](https://github.com/apache/sling-org-apache-sling-resourceresolver/blob/org.apache.sling.resourceresolver-1.10.0/src/main/java/org/apache/sling/resourceresolver/impl/mapping/MapEntries.java#L480)) and `loadAlias` ([L703](https://github.com/apache/sling-org-apache-sling-resourceresolver/blob/org.apache.sling.resourceresolver-1.10.0/src/main/java/org/apache/sling/resourceresolver/impl/mapping/MapEntries.java#L703)).

### `sling:vanityPath` & `sling:vanityOrder`: `sling:vanityPath` 
- is set on a content resource to claim a user-friendly URL. These are loaded into the mapping tables as internal redirects. `sling:vanityOrder` (a `Long`) resolves conflicts if multiple resources claim the same vanity path.
    *   *Code Reference*: `MapEntries.java` `PROP_VANITY_PATH` ([L90](https://github.com/apache/sling-org-apache-sling-resourceresolver/blob/org.apache.sling.resourceresolver-1.10.0/src/main/java/org/apache/sling/resourceresolver/impl/mapping/MapEntries.java#L90)) and `PROP_VANITY_ORDER` ([L92](https://github.com/apache/sling-org-apache-sling-resourceresolver/blob/org.apache.sling.resourceresolver-1.10.0/src/main/java/org/apache/sling/resourceresolver/impl/mapping/MapEntries.java#L92)) are used in methods like `loadVanityPaths` and `loadVanityPath`.

## Hierarchical Mapping Structures

One of the most powerful features of JCR-based mappings is the ability to organize `sling:Mapping` nodes hierarchically under `/etc/map`. The hierarchy is not merely organizational—it directly determines how URL patterns are constructed and matched.

### How Hierarchies Create Patterns

The mapping resolution engine processes the JCR tree through a recursive, depth-first traversal implemented in the [`gather` method of `MapEntries.java`](https://github.com/apache/sling-org-apache-sling-resourceresolver/blob/org.apache.sling.resourceresolver-1.10.0/src/main/java/org/apache/sling/resourceresolver/impl/mapping/MapEntries.java#L752). As it descends the tree, it **progressively builds URL patterns by concatenating path segments** from each level of the hierarchy.

For each node in the tree:

1. **Pattern Segment Determination**: The node contributes a segment to the pattern based on:
    - The value of its `sling:match` property (if present), or
    - Its node name followed by `/` (if `sling:match` is absent)
    *   *Code Reference*: `MapEntries.java` `gather` method, lines ([L761-L763](https://github.com/apache/sling-org-apache-sling-resourceresolver/blob/org.apache.sling.resourceresolver-1.10.0/src/main/java/org/apache/sling/resourceresolver/impl/mapping/MapEntries.java#L761-L763)) show pattern segment determination using `PROP_REG_EXP` or child name.

2. **Pattern Concatenation**: This segment is appended to the pattern inherited from its parent node, creating a cumulative pattern that reflects the full path from the root mapping node.
    *   *Code Reference*: `MapEntries.java` `gather` method, line ([L765](https://github.com/apache/sling-org-apache-sling-resourceresolver/blob/org.apache.sling.resourceresolver-1.10.0/src/main/java/org/apache/sling/resourceresolver/impl/mapping/MapEntries.java#L765)).

3. **Recursive Processing**: Before creating a mapping entry for the current node, the engine recursively processes all of its child nodes with the current cumulative pattern as their parent path.
    *   *Code Reference*: `MapEntries.java` `gather` method, lines ([L770-L771](https://github.com/apache/sling-org-apache-sling-resourceresolver/blob/org.apache.sling.resourceresolver-1.10.0/src/main/java/org/apache/sling/resourceresolver/impl/mapping/MapEntries.java#L770-L771)).

4. **Entry Creation**: After all children have been processed, a `MapEntry` is created for the current node using its fully constructed pattern.
    *   *Code Reference*: `MapEntries.java` `gather` method, line ([L775](https://github.com/apache/sling-org-apache-sling-resourceresolver/blob/org.apache.sling.resourceresolver-1.10.0/src/main/java/org/apache/sling/resourceresolver/impl/mapping/MapEntries.java#L775)).

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
*   *Code Reference*: In `MapEntry.java`, `createResolveEntry` filters out regex in `sling:internalRedirect` using `filterRegExp` ([L132](https://github.com/apache/sling-org-apache-sling-resourceresolver/blob/org.apache.sling.resourceresolver-1.10.0/src/main/java/org/apache/sling/resourceresolver/impl/mapping/MapEntry.java#L132)). The `replace` method ([L195](https://github.com/apache/sling-org-apache-sling-resourceresolver/blob/org.apache.sling.resourceresolver-1.10.0/src/main/java/org/apache/sling/resourceresolver/impl/mapping/MapEntry.java#L195)) iterates through `redirects` and returns the first successful one.

### `sling:alias`

A content resource can have multiple aliases. This has two main effects:

-   **Incoming Resolution**: Any of the defined aliases can be used in a URL to resolve to the resource. For example, if a resource has aliases `foo` and `bar`, both `/content/mypage/foo` and `/content/mypage/bar` would resolve to the same underlying resource.
-   **Outgoing Mapping**: When generating URLs for the resource (e.g., via `ResourceMapper.getAllMappings()`), all its aliases are considered valid "friendly" names and will be included in the list of possible mappings.
*   *Code Reference*: `MapEntries.java` `loadAlias` method ([L699](https://github.com/apache/sling-org-apache-sling-resourceresolver/blob/org.apache.sling.resourceresolver-1.10.0/src/main/java/org/apache/sling/resourceresolver/impl/mapping/MapEntries.java#L699)) processes multiple `sling:alias` values from `aliasArray`.

### `sling:vanityPath`

Each entry in a multivalued `sling:vanityPath` property creates a separate, independent mapping rule. This allows a single content resource to be accessible from multiple distinct vanity URLs. For example, a resource can have `sling:vanityPath` set to `["/my-product", "/products/my-product"]`, and both URLs will resolve to it.
*   *Code Reference*: `MapEntries.java` `loadVanityPath` method ([L890](https://github.com/apache/sling-org-apache-sling-resourceresolver/blob/org.apache.sling.resourceresolver-1.10.0/src/main/java/org/apache/sling/resourceresolver/impl/mapping/MapEntries.java#L890)) iterates through `pVanityPaths` to create individual entries.

## Syntax and Format for `resource.resolver.mapping` OSGi Property

The `resource.resolver.mapping` OSGi property, used by `ResourceResolverImpl`, expects a specific syntax and format for its entries. These entries are processed by the `org.apache.sling.resourceresolver.impl.mapping.Mapping` class.

Here's the syntax and format:

**Format:**
Each entry in the `resource.resolver.mapping` property follows the general pattern:
`<internalPathPrefix><operator><externalPathPrefix>`

**Syntax Details:**

*   **`<internalPathPrefix>`**: This represents a path prefix within the JCR repository (the "inside" or "from" path).
*   **`<externalPathPrefix>`**: This represents a path prefix for the external URL (the "outside" or "to" path).
*   **`<operator>`**: This single character determines the direction of the mapping:
    *   `:` (colon): The mapping is applied in **both** directions (inbound and outbound).
    *   `>` (greater than): The mapping is for **inbound** requests only (mapping a request URL path to a resource path).
    *   `<` (less than): The mapping is for **outbound** requests only (mapping a resource path to a URL path).

**Examples from the codebase (`Mapping.java`):**

The `Mapping.split(String map)` method ([L158](https://github.com/apache/sling-org-apache-sling-resourceresolver/blob/org.apache.sling.resourceresolver-1.10.0/src/main/java/org/apache/sling/resourceresolver/impl/mapping/Mapping.java#L158)), which parses these configuration strings, uses the following regular expression: `(.+)([:<>])(.+)` ([L58](https://github.com/apache/sling-org-apache-sling-resourceresolver/blob/org.apache.sling.resourceresolver-1.10.0/src/main/java/org/apache/sling/resourceresolver/impl/mapping/Mapping.java#L58)).

*   `"/:/"`: A bidirectional mapping where the root (`/`) maps to itself.
*   `"/content/:/"`: A bidirectional mapping where `/content/` maps to `/content/`.

**Backward Compatibility:**

The `split` method also handles a deprecated format for bidirectional mappings using a hyphen (`-`) instead of a colon (`:`):

*   `<internalPathPrefix>-<externalPathPrefix>`: Equivalent to `:<internalPathPrefix>:<externalPathPrefix>`.

If the string doesn't match any of the above patterns, it defaults to a bidirectional mapping where both prefixes are the same as the input string: `map - map`.

**Key Points:**

*   These mappings are **simple, literal string-based path prefixes**. They do **not** support regular expressions for the path matching itself, unlike the JCR-based `/etc/map` rules.
*   The `ResourceResolverImpl` uses these mappings during its `resolve` method and `map` method, leveraging the `MapEntry` objects derived from these configurations.

## OSGi Mappings vs. JCR Mappings

| Aspect                | OSGi `resource.resolver.mapping`        | JCR `/etc/map`                                       |
| :-------------------- | :-------------------------------------- | :--------------------------------------------------- |
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
