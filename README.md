# GQJ — Graph Query JSON

GQJ is a minimal JSON-native query language in which a typed domain graph directly defines query scope and result structure. It is designed for reliable generation by AI agents and deterministic execution by backend planners.

**Status:** Draft V1 specification.

## Example

```json
{
  "Devices": {
    "select": ["Name"],

    "Profiles": {
      "eq": ["Status", {"value": "Active"}],
      "select": ["Name"]
    }
  }
}
```

The query structure follows the domain graph. In this example, `Devices` is the root relationship, `Profiles` is a relationship from each Device, and only matching Profiles are materialized.

## Documentation

- [GQJ V1 Language Specification](docs/language-specification.md)
- [Limitations and Workarounds](docs/limitations-and-workarounds.md)

## License

GQJ is licensed under the [Apache License 2.0](LICENSE).
