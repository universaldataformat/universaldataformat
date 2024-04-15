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
