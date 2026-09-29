# GQJ Limitations and Workarounds

This document records deliberate GQJ V1 limitations and practical workarounds. The normative language definition is in `language-specification.md`.

## Cross-Scope Field Comparisons

GQJ V1 field references resolve within the current entity scope. Direct comparisons between fields from different entity scopes are not supported.

### Workarounds

1. **Computed field or value function** — expose a reusable domain-level value that captures the cross-scope logic.
2. **Registered semantic function** — expose a named server-side operation for reusable complex logic.
3. **Multiple GQJ queries** — run separate queries and correlate bounded results in the caller.
4. **Specialized API/tool** — use a dedicated operation when performance or dataset size makes client-side composition unsuitable.

## No Pagination or Sorting

GQJ V1 does not define `page` or `sort`.

`limit` is an unordered cardinality bound. It does not imply stable ordering, pagination, or top-N semantics.

When deterministic ordering or pagination is required, use a specialized API/tool or a future language extension.

## Relationship Limits

`limit` may be applied to any to-many relationship, including a top-level relationship from the implied root.

Example:

```json
{
  "Devices": {
    "limit": 10,
    "select": ["Name"],
    "Profiles": {
      "limit": 5,
      "select": ["Name"]
    }
  }
}
```

`limit` is invalid on to-one relationships.

When more matching elements exist than the limit permits, any matching subset may be returned. Repeated executions are not guaranteed to return the same subset.

The result contains no truncation metadata indicating whether additional matches existed.

## No Whole-Graph Limit

GQJ V1 defines no whole-result graph-size limit.

Per-relationship `limit` is the only graph-size control in the language. Runtime safety ceilings such as maximum depth, response size, cost, or execution time are implementation concerns and are not part of GQJ syntax.

## Type-Changing Relationship Transformations

A relationship transformation function may preserve or change the current element type.

If it preserves the current entity type, nested relationships remain available.

If it changes the type, nested relationship traversal ends at that scope, even if the new output type is another entity type.

`select`, including `select: []`, remains valid after the transformation.

### Workarounds

1. **Run another GQJ query** from an appropriate root relationship.
2. **Expose a graph relationship explicitly** when the relationship is a meaningful domain concept.
3. **Use a specialized API/tool** for analytical or shape-changing workflows that require additional traversal.

## One Relationship Transformation per Scope

GQJ V1 permits at most one relationship transformation function at a relationship scope.

If several transformation steps are needed, expose one domain-specific composite transformation function that performs the internal chain.

## No Value-Function Chaining

GQJ V1 does not recursively interpret a value function's JSON argument as another value-function invocation.

If chained computation is required, expose a composite value function with the required behavior.

## No `groupBy`

GQJ V1 does not define a `groupBy` construct.

Use existing graph relationships, domain-specific transformations, multiple queries, or a specialized analytical API/tool when grouping is required.

## No Core `distinct`

`distinct` is not a GQJ V1 keyword.

A domain may expose a transformation with equivalent semantics when needed. Such a function follows the normal relationship-transformation rules, including the rule that nested traversal ends if the transformation changes the current element type.
