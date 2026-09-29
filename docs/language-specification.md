# GQJ V1 Language Specification

## 1. Purpose

GQJ (Graph Query JSON) is a minimal JSON-native query language in which a typed domain graph directly defines query scope and result structure. It is designed for reliable generation by AI agents and deterministic execution by backend planners.

This specification defines only the GQJ query, schema, and result formats and their semantics. Transport, authentication, authorization, HTTP behavior, runtime limits, execution strategy, error envelopes, and other implementation concerns are outside the language specification.

## 2. Query Document

A GQJ query is a JSON object containing exactly one top-level relationship from the implied global root.

The query body contains no version field and no wrapper such as `entity`, `where`, or `via`.

Version information, when required, is provided outside the query document by the embedding implementation.

Valid:

```json
{
  "Devices": {
    "select": ["Name"]
  }
}
```

Invalid:

```json
{
  "Devices": {
    "select": ["Name"]
  },
  "Profiles": {
    "select": ["Name"]
  }
}
```

Multiple independent queries are issued as separate GQJ query documents.

## 3. Typed Domain Graph

A GQJ schema defines a typed domain graph.

The graph has an implied global root. Top-level query keys are relationship names exposed by that root.

Relationship names are used consistently at every level of traversal. Entity type names do not appear in query traversal syntax.

Example:

```text
root
 └─ Devices -> Device
      ├─ Profiles -> Profile
      │    └─ Configurations -> Configuration
      └─ Applications -> Application
```

A query against this graph may be:

```json
{
  "Devices": {
    "select": ["Name"],
    "Profiles": {
      "select": ["Name"]
    }
  }
}
```

## 4. Naming

Recommended naming conventions are:

- relationship names: PascalCase;
- field names: PascalCase;
- GQJ keywords and operators: lowercase.

Names are schema-defined and canonical. GQJ V1 defines no aliases.

Type names are globally unique within a schema.

Field and relationship names are unique within their containing entity type. A field and a relationship may not share the same name in the same entity type.

Function names are unique per input type.

## 5. Relationship Cardinality

Relationships are either to-one or to-many.

Schema cardinality determines result shape.

- to-many relationship -> JSON array;
- required to-one relationship -> JSON object;
- optional to-one relationship -> JSON object or `null`.

The same query syntax is used for both cardinalities.

Collection-only constructs, including `limit` and relationship transformation functions, are invalid on to-one relationships.

## 6. Relationship Scope

Each relationship object defines a query scope.

A relationship scope may contain:

- predicates;
- nested relationship predicates or materialized relationship branches;
- at most one relationship transformation function for a to-many relationship;
- optional `limit` for a to-many relationship;
- optional `select`.

Sibling qualifying expressions at the same scope compose with implicit AND semantics.

## 7. Selection and Materialization

`select` has two roles:

1. it marks the current relationship scope for materialization in the result graph;
2. it defines the ordered values emitted for each materialized node.

### 7.1 Omitted `select`

If `select` is absent, the relationship scope is qualification-only and does not appear in the result.

Example:

```json
{
  "Devices": {
    "select": ["Name"],
    "Profiles": {
      "eq": ["Status", {"value": "Active"}]
    }
  }
}
```

`Profiles` qualifies Devices but is not materialized.

### 7.2 Empty `select`

`select: []` materializes the structural relationship node but exposes no selected values.

Example:

```json
{
  "Devices": {
    "select": [],
    "Profiles": {
      "select": ["Name"]
    }
  }
}
```

### 7.3 Materialization Path Rule

If a descendant relationship contains `select`, every ancestor relationship on the path from the root to that descendant must also contain `select`.

This is invalid:

```json
{
  "Devices": {
    "Profiles": {
      "select": ["Name"]
    }
  }
}
```

This is valid:

```json
{
  "Devices": {
    "select": [],
    "Profiles": {
      "select": ["Name"]
    }
  }
}
```

To materialize Profiles without Devices, the schema must expose a root relationship to Profiles and the query must start there.

## 8. Select Entries

A `select` array may contain:

- field references;
- literal values;
- value-function invocations.

Duplicate entries are valid.

The order of `select` entries is part of the language contract and is preserved exactly in the result `values` array.

Example:

```json
{
  "Devices": {
    "select": [
      "SerialNumber",
      "Name",
      {"value": "Device"},
      {"riskScore": {}},
      "Name"
    ]
  }
}
```

## 9. Result Format

Selected values are emitted under the reserved key `values`.

Relationship names remain structural keys in the result.

Example query:

```json
{
  "Devices": {
    "select": ["Name"],
    "Profiles": {
      "select": ["Name"],
      "Configurations": {
        "select": ["Type", "CameraDisabled"]
      }
    },
    "Applications": {
      "select": ["Name", "Version"]
    }
  }
}
```

Example result:

```json
{
  "Devices": [
    {
      "values": ["Device 1"],
      "Profiles": [
        {
          "values": ["Profile 1"],
          "Configurations": [
            {
              "values": ["FeatureControl", true]
            }
          ]
        }
      ],
      "Applications": [
        {
          "values": ["Outlook", "16.0"]
        }
      ]
    }
  ]
}
```

For `select: []`, `values` may be omitted. The node may therefore be an empty object or contain only nested materialized relationships.

## 10. Operands

Where an operand is accepted, GQJ supports the following forms.

### 10.1 Field Reference Shorthand

A bare string is a field reference:

```json
"Status"
```

Equivalent explicit form:

```json
{"field": "Status"}
```

### 10.2 Literal

A literal uses `value`:

```json
{"value": "Active"}
```

The literal value may be any valid JSON value, including `null`.

### 10.3 Value Function

A value function invocation has the form:

```json
{"<functionName>": <any-valid-JSON-value>}
```

The JSON value is the function's explicit argument payload. GQJ assigns no special meaning to `{}`, `null`, `""`, `[]`, or any other JSON value.

The current value/entity scope is implicit and is provided to the function by the implementation.

Value functions may appear anywhere an operand is accepted, including predicates and `select`.

Value functions do not recursively invoke other value functions through their argument JSON. Function chaining is not part of GQJ V1. A domain requiring chained behavior may expose a dedicated composite function.

A registered value function may accept a relationship or relationship collection when its contract explicitly permits it.

Raw relationship references are not valid operands for core comparison operators.

## 11. Core Boolean and Comparison Operators

GQJ V1 defines:

- `and`
- `or`
- `not`
- `eq`
- `ne`
- `lt`
- `lte`
- `gt`
- `gte`
- `in`

Core keywords are reserved and cannot be registered as function names.

### 11.1 Binary Comparison Operators

`eq`, `ne`, `lt`, `lte`, `gt`, and `gte` each contain exactly two operands:

```json
{
  "eq": [
    "Status",
    {"value": "Active"}
  ]
}
```

### 11.2 IN

`in` contains exactly two operands. The first is the tested value. The second must evaluate to a collection of values whose element type is compatible with the first operand.

Example:

```json
{
  "in": [
    "Status",
    {"value": ["Active", "Pending"]}
  ]
}
```

This is true when the first operand is equal to at least one value in the second operand.

### 11.3 AND and OR

`and` and `or` contain arrays of one or more predicate objects.

```json
{
  "and": [
    {"eq": ["Status", {"value": "Active"}]},
    {"eq": ["Region", {"value": "Canada"}]}
  ]
}
```

Empty `and` and `or` arrays are invalid.

### 11.4 NOT

`not` contains exactly one predicate object.

```json
{
  "not": {
    "eq": ["Status", {"value": "Active"}]
  }
}
```

That object may itself contain `and`, `or`, a relationship predicate, or another valid predicate form.

### 11.5 Implicit AND

Sibling qualifying expressions at the same scope use implicit AND.

For example:

```json
{
  "Devices": {
    "eq": ["Region", {"value": "Canada"}],
    "Profiles": {
      "eq": ["Status", {"value": "Active"}]
    },
    "select": ["Name"]
  }
}
```

qualifies Devices satisfying both conditions.

When duplicate JSON keys would be required, explicit `and` or `or` arrays must be used.

## 12. Type Semantics

Comparison operands must be compatible according to the schema and registered function contracts.

GQJ V1 performs no implicit type coercion.

JSON `null` is a normal literal.

- `eq` with `null` means is null;
- `ne` with `null` means is not null;
- ordering comparisons with `null` are invalid.

Field-to-field comparisons are allowed within the same entity scope.

Direct cross-scope field comparisons are not supported in V1.

## 13. Relationship Qualification

A relationship predicate participates in qualification of its parent scope.

For a to-many relationship, qualification is existential:

> the parent matches if at least one related node matches the relationship predicate.

If the relationship is materialized, only matching related nodes are included in that relationship result.

Example:

```json
{
  "Devices": {
    "select": [],
    "Profiles": {
      "eq": ["Status", {"value": "Active"}],
      "select": ["Name"]
    }
  }
}
```

A Device is returned only if it has at least one Active Profile, and only matching Active Profiles are materialized.

For a to-one relationship, the predicate is true when the relationship exists and its target matches.

Relationship predicates may appear directly at a scope or inside `and`, `or`, and `not`.

## 14. NOT over Relationships

`not` applied to a relationship predicate has NOT EXISTS semantics.

Example:

```json
{
  "Devices": {
    "not": {
      "Profiles": {
        "eq": ["Status", {"value": "Active"}]
      }
    },
    "select": ["Name"]
  }
}
```

This selects Devices for which no related Profile matches `Status == "Active"`.

This is intentionally different from:

```json
{
  "Devices": {
    "Profiles": {
      "ne": ["Status", {"value": "Active"}]
    },
    "select": ["Name"]
  }
}
```

which selects Devices having at least one related Profile whose Status is not Active.

## 15. Relationship Transformation Functions

A domain may register relationship transformation functions.

They:

- are valid only on to-many relationship scopes;
- operate on the current relationship collection implicitly;
- receive any valid JSON value as their explicit argument payload;
- are domain-defined rather than core GQJ keywords;
- are limited to at most one relationship transformation function per relationship scope in V1.

There is no `transform` wrapper. A registered non-core key in relationship-scope position is resolved through the schema.

If multiple transformation stages are needed, the domain may expose one composite transformation function.

### 15.1 Output Type and Traversal

A transformation declares its output type.

If the transformation preserves the current entity type, that entity's nested relationships remain available.

If the transformation changes the type in any way, nested relationship traversal from that scope is no longer allowed, even if the new output type is another entity type.

This prevents unrelated graph relationships from appearing structurally beneath the original relationship name.

`select`, including `select: []`, remains valid after a type-changing transformation.

## 16. Limit

`limit` is a core GQJ construct for to-many relationship scopes.

Example:

```json
{
  "Devices": {
    "limit": 10,
    "select": ["Name"]
  }
}
```

Rules:

- `limit` is optional;
- it may be used on any to-many relationship, including top-level root relationships;
- it is invalid on to-one relationships;
- it bounds the number of matching elements returned at that relationship scope;
- it does not imply ordering;
- it is not pagination;
- it is not a top-N operation;
- when more elements match than the limit permits, any matching subset may be returned;
- repeated executions need not return the same subset;
- GQJ V1 defines no truncation metadata in the result.

GQJ V1 defines no `sort`, `page`, or whole-graph limit construct.

## 17. Relationship-Scope Evaluation Order

The semantic evaluation order of a relationship scope is:

1. relationship source;
2. qualification and filtering;
3. optional relationship transformation function;
4. optional `limit`;
5. optional `select`;
6. result materialization.

`select` is terminal projection. It does not determine which elements qualify.

## 18. Schema Document

A GQJ schema is a JSON object with exactly these top-level members:

- `root`
- `entities`
- `types`

Example:

```json
{
  "root": {
    "relationships": {
      "Devices": {
        "target": "Device",
        "cardinality": "many"
      }
    }
  },
  "entities": {
    "Device": {
      "fields": {
        "Name": {
          "type": "string",
          "nullable": false
        }
      },
      "relationships": {
        "Profiles": {
          "target": "Profile",
          "cardinality": "many"
        },
        "Owner": {
          "target": "User",
          "cardinality": "one",
          "nullable": true
        }
      }
    }
  },
  "types": {
    "string": {
      "operators": ["eq", "ne", "in"],
      "valueFunctions": {}
    },
    "Device": {
      "operators": [],
      "valueFunctions": {
        "riskScore": {
          "argumentSchema": {},
          "returnType": "number"
        }
      },
      "relationshipTransformations": {
        "recent": {
          "argumentSchema": {
            "type": "object",
            "properties": {
              "days": {"type": "integer"}
            }
          },
          "returnType": "Device"
        }
      }
    }
  }
}
```

### 18.1 Root

`root` contains one member:

- `relationships`: an object whose keys are top-level relationship names and whose values are relationship definitions.

### 18.2 Entities

`entities` is an object keyed by globally unique entity type names.

Each entity contains:

- `fields`: an object keyed by field name;
- `relationships`: an object keyed by relationship name.

Within an entity, field and relationship names are mutually unique.

A field definition contains:

- `type`: the globally unique type name;
- `nullable`: a Boolean indicating whether the field may be null.

A relationship definition contains:

- `target`: the globally unique entity type name reached by the relationship;
- `cardinality`: `"one"` or `"many"`;
- `nullable`: required for `"one"` relationships and omitted for `"many"` relationships.

### 18.3 Types

`types` is an object keyed by globally unique type name.

A type definition may contain:

- `operators`: the core comparison operators supported by values of that type;
- `valueFunctions`: registered value functions for that input type;
- `relationshipTransformations`: registered relationship transformation functions for that input type when applicable.

Operator applicability is defined by type, not per field.

Function names are unique per input type across all function categories.

### 18.4 Function Definitions

A value-function definition contains:

- `argumentSchema`: a JSON Schema describing the explicit JSON argument payload;
- either `returnType` or `returnSchema`.

A relationship-transformation definition contains:

- `argumentSchema`: a JSON Schema describing the explicit JSON argument payload;
- either `returnType` or `returnSchema`.

`returnType` references a named GQJ type.

`returnSchema` is JSON Schema for an ad hoc structured JSON result.

Exactly one of `returnType` and `returnSchema` must be present.

Domain-specific function names and semantics are not standardized by GQJ V1.

## 19. Non-Goals of V1

GQJ V1 does not define:

- pagination;
- sorting;
- `groupBy`;
- a core `distinct` operator;
- function chaining;
- multiple relationship transformations at one scope;
- direct cross-scope field comparisons;
- aliases;
- transport or protocol version representation inside the query;
- HTTP behavior;
- authentication or authorization;
- runtime safety limits;
- standard error envelopes;
- implementation execution strategy.

These concerns may be handled by the embedding implementation, domain-specific functions, specialized APIs/tools, multiple GQJ queries, or future language versions.
