# Type definitions

## `data`

[Product type](../types.md#products) definition. See [Types](../types.md) for details.

```aqua
data SomeType:
  fieldName: FieldType
  otherName: OtherType
  third: []u32
```

## `alias`

Aliasing a type to a name.

It may help with self-documented code and refactoring.

```aqua
alias PeerId: string
alias MyDomain: DomainType
```
