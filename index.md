# Universal data format

We are building the **Universal data format** (Udf). Having taken the JSON-data format as baseline, with the Udf we can create structures like this:

```
{
  # This is a comment
  key <String>: "this is a value with type-constraint",
  other: [
	  "hello", {}, [], true, false, empty, null, 123.45 
  ],
  "more" <String Or Number>: "this is a value with union type-constraint"
}
```

From this tiny example, it should be instantly obvious that Udf expands the JSON by introducing

* comments (starting with '#')
* type-constraints (between '&lt;' and '>')
* empty-value (empty)

These are examples of something that are lacking from JSON. JSON has served us well, but in order to build the
next generation solutions we need to go further and look for a better data format.

## Conforming to Udf

| Feature      | Description                                         | Example
|--------------|-----------------------------------------------------|--------
| Comments     | UDF supports comments                               | {key: "value"} # this is a comment
| Empty-value  | Empty-value can be declared. It is omitted when data is serialized | {key: empty}
| Boolean keys | Object can have value omitted. The key is interpreted as Boolean, and the value will be true | {hidden}
| Path-value   | Path-value can be declared. It is serialized as string | {key: ~.otherKey[1].value}
| Multiline String value       | Basic multiline String-value can be declared | \n{\n  key: """line1\n  line2\n  line3"""\n  }
| Multiline String value """|  | Multiline String-value can be declared | \n{\n  key: """|line1\n  line2\n  line3"""\n  }
| Multiline String value """>  | Multiline String-value can be declared | \n{\n  key: """>line1\n  line2\n  line3"""\n  }
| Raw-value    | Raw-value can be declared inside backticks | 
| Configuration options | Object's field may have configuration options. This is an object itself | {mykey {hidden}: "my-value"}
| Constraints, support level 1 | Constraints are supported, but they are not evaluated | {myfield &lt;String>: "my-value"}
| Constraints, support level 2 | Constraints are supported and basic data-types are evaluated | {myfield &lt;String>: "my-value"}
| Constraints, support level 3 | Constraints are supported and basic data-types with "And" and "Or" are evaluated | {myfield &lt;String Or Empty>: "my-value"}
| Constraints, support level 4 | Constraints are supported with basic expressions | {myfield &lt;String And => length() &lt; 20>: "my-value"}
| Constraints, support level 5 | Constraints are supported with rich expressions | {myfield &lt;String And => in("foo", "bar", "baz") &lt; 20>: "my-value"}
