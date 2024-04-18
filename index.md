# Universal data format

We are building the **Universal data format** (Udf). Having taken the JSON-data format as baseline, with the Udf we can create structures like this:

```
{
  # This is a comment
  key <String>: "key without quotes and a value with type-constraint",
  "more" <String Or Number>: "key with double-quotes and a value with union type-constraint",
  'key2': "key with single quotes",
  prettyPrint, # a boolean-key, the value is set as `true` by default
  other: [
    "hello", {}, [], true, false,
    empty, null, 123.45, # the empty -value will be omitted from the list
    """this is a
       string that spans
       over multiple lines"""
  ]
}
```

From this tiny example, it should be instantly obvious that Udf expands the JSON by introducing

* Comments (starting with `#`)
* Type-constraints (between '<' and '>', e.g. `<String>`)
* Empty-value (`empty`)

These are examples of something that are lacking from JSON. JSON, introduced in 1999, has served us well, but in order to build the
next generation solutions for 2020's and beyond we need to go further and look for a better data format.

Udf is a well thought out data format which fixes the key problems with JSON without introducing too much complexity.

## Conforming to Udf

| Feature      | Description                                         | Example
|--------------|-----------------------------------------------------|--------
| Comments     | UDF supports comments                               | `{key: "value"} # this is a comment`
| Empty-value  | Empty-value can be declared. It is omitted when data is serialized | `{key: empty}`
| Boolean keys | Object's key can have value omitted. In this case the key is interpreted as Boolean, and the value will be true | `{hidden}`
| Path-value   | Path-value can be declared. It is serialized as string | `{key: ~.otherKey[1].value}`
| Multiline String value       | Basic multiline String-value can be declared | `{key: """line1\n  line2\n  line3"""}`
| Multiline String value """\|  | Multiline String-value can be declared | `{key: """\|line1\n  line2\n  line3"""}`
| Multiline String value """>  | Multiline String-value can be declared | `{key: """>line1\n  line2\n  line3"""}`
| Raw-value    | Raw-value can be declared inside backticks | ```{key: `raw value`}```
| Metadata     | Object's field may have metadata (or configuration options). Metadata is declared in an udf-object itself | `{mykey {hidden}: "my-value"}`
| Constraints, support level 1 | Constraints declared with angle brackets are supported, but they are not evaluated | `{myfield <String>: "my-value"}`
| Constraints, support level 2 | Constraints are supported and basic data-types are evaluated | `{myfield <String>: "my-value"}`
| Constraints, support level 3 | Constraints are supported and basic data-types and "Or"-expressions are evaluated | `{myfield <String Or Empty>: "my-value"}`
| Constraints, support level 4 | Constraints are supported with support for basic functions and all logical expressions from Udf-expression language | `{myfield <String And => length() < 20>: "my-value"}`
| Constraints, support level 5 | Constraints are supported with rich expressions from Udf-expression language | `{myfield <.bin[1] = "bai">: {bin: ["bai", "baa"]}}`
| Constraints, support level 6 | Constraints are supported with full Udf-expression language | `{myfield <When => key() = "bin": .bin => length() < 3; Otherwise => value() => length() < 20;>: {bin: ["bai", "baa"]}}`

## Udf-grammar

Here's the Universal data format's grammar expresses with ANTLR4:

```
// 
// License: MIT
// 

grammar UDF;

udf
   : value
   ;

obj
   : '{' pair (',' pair)* '}'
   | '{' '}'
   ;

pair
   //: (STRING | ID) ':' value
   // Supports field configurations, e.g. {foo {hidden} <String>: "bar"}
   // Supports keys without giving value (Boolean-key)
   : (STRING | ID) obj? constraint_udf_value? (':' (value))?
   ;

constraint_udf_value
    : '<' (ESC | SAFECODEPOINT)* '>'
    ;

array
   : '[' value (',' value)* ']'
   | '[' ']'
   ;

//
// "Path" cannot be lowercase, since would be confused with function-name.
// - Path(...)
//    - with parentheses, so we can use e.g. Filter-expr. Empty path can be given as Path().
//   Examples:
//    - Path($foo.names[1]) --> resolve root-value from named Value.
//    - Path(foo().names[1]) --> resolve root-value from function foo.
//    - Path(.names[1]) --> no root-value, it must be set manually.
// - ~...
//    - Concise expr for path. Does not support empty path.
// - ~(...)
//      Allow using Filter-expr after Path. Empty path can be given as ~().
//
path_value
   : ('Path' | '~') '(' (CONST_ID | (ID ':')* ID '(' ')')? (('.' ID) | ('[' NUMBER ']'))* ')'
   | '~' (CONST_ID | (ID ':')* ID '(' ')')? (('.' ID) | ('[' NUMBER ']'))+
   | '~' '(' (('.' ID) | ('[' NUMBER ']'))* ')'
   ;

value
   : STRING
   | NUMBER
   | obj
   | array
   | TRUE
   | FALSE
   | NULL
   | EMPTY_VALUE
   | RAW_STRING
   | SINGLE_QUOTED_STRING
   | MULTILINE_STRIPPED_STRING
   | MULTILINE_PADDED_LINES_STRING
   | MULTILINE_STRING
   | path_value
   ;

FALSE
   : 'false'
   ;
   
TRUE
   : 'true'
   ;
   
NULL
   : 'null'
   ;
   
STRING
   : '"' (ESC | SAFECODEPOINT)* '"'
   ;

SINGLE_QUOTED_STRING
   : '\'' (SINGLE_QUOTED_ESC | '"' | SINGLE_QUOTED_SAFECODEPOINT)* '\''
   ;

RAW_STRING
   : '`' (BINARY_ALLOWED | BINARY_SAFECODEPOINT)* '`'
   ;

//
// Initial paddings for each line are ignored.
// The lines are concatenated and new-lines removed.
//
MULTILINE_STRIPPED_STRING
   : '"""|' (MULTILINE_ESC | SAFECODEPOINT)* '"""'
   ;

//
// Initial paddings for each line are ignored.
// The lines are concatenated and new-lines kept.
//
MULTILINE_PADDED_LINES_STRING
   : '""">' (MULTILINE_ESC | SAFECODEPOINT)* '"""'
   ;

MULTILINE_STRING
   : '"""' (MULTILINE_ESC | SAFECODEPOINT)* '"""'
   ;

EMPTY_VALUE
   : 'empty'
   ;

fragment ESC
   : '\\' (["\\/bfnrt] | UNICODE)
   ;

fragment MULTILINE_ESC
   : '\r' | '\n' | '\t' | ('\\' (["\\/bfrnt] | UNICODE))
   ;

fragment SINGLE_QUOTED_ESC
   : '\\' (['\\/bfrnt] | UNICODE)
   ;

//
// Allow these outside from the BINARY_SAFECODEPOINT:
//
fragment BINARY_ALLOWED
   : [\b\r\n\t\f]
   | '\\`'
   ;

fragment UNICODE
   : 'u' HEX HEX HEX HEX
   ;


fragment HEX
   : [0-9a-fA-F]
   ;


fragment SAFECODEPOINT
   : ~ ["\\\u0000-\u001F]
   ;

fragment BINARY_SAFECODEPOINT
   : ~ [`\\\u0000-\u001F]
   ;

fragment SINGLE_QUOTED_SAFECODEPOINT
   : ~ ['\\\u0000-\u001F]
   ;

NUMBER
   : '-'? INT ('.' [0-9] +)? EXP?
   ;


fragment INT
   : '0' | [1-9] [0-9]*
   ;

//
// allow: 1e-03 (INT does not allow this since exp has leading zeros)
//
fragment EXP_INT
   : '0' | [0-9] [0-9]*
   ;

// no leading zeros
// Note: has \- since - means "range" (regexp) inside [...]
fragment EXP
   : [Ee] [+\-]? EXP_INT
   ;

WS
   : [ \t\n\r] + -> skip
   ;

CONST_ID
    : [\\$][a-zA-Z]+[a-zA-Z_0-9]*
    ;

ID
    : [a-zA-Z_][a-zA-Z0-9_\-]*
    ;

COMMENT
    : '#' ~[\r\n]* -> skip
    ;
```

## Boolean keys

The object's key may have value omitted. In this case the key will be interpreted as a Boolean -value and the value will be set as `true`.

### Grammar

```
obj
   : '{' pair (',' pair)* '}'
   | '{' '}'
   ;

pair
   : (STRING | ID) obj? constraint_udf_value? (':' (value))?
   ;
```

### Example

```
{
  prettyPrint,
  otherOption: false
}
```

The corresponding JSON is as follows:

```json
{
  "prettyPrint": true,
  "otherOption": false
}
```

## Path -type in Universal data format

Examples of UDF Paths are given below:

* `Path(.names[1])`
  * This path is not plugged into any specific root-value, the application must define or know it implicitly or based on other context information.

* `Path($foo.names[1])`
  * The application context has a named value "foo" which is the starting point for the path.

* `Path(foo().names[1])`
  * The application context has a named function "foo", and executing it gives the value for the path's starting point.

With UDF it is also possible to use the UDF Metadata to advice the application context how the root-value should be resolved. The example below highlights this point:

```
{
  mykey {
    contextValue: "foo"
  }: Path(.names[1])
}
```
### UDF Paths Navigation

* **.names**: This refers to a key within a JSON/UDF object. In the path `Path(.names[1])`, it indicates that we are looking for the value associated with the key `"names"`.

* **[1]**: This represents an index within a list or array. In the path `Path(.names[1])`, it means we are interested in the value at the first position (index 1) of the list associated with the `"names"` key.

**Example:**

Given a JSON structure like this:

```json
{
  "names": ["Alice", "Bob", "Charlie"]
}
```

The path `Path(.names[1])` would target the value 'Alice'.

In essence, `.names` specifies the key within the JSON/UDF object, and `[1]` selects the element at the first position (index 1) from the value found using the key. This allows you to navigate through nested structures within your JSON/UDF data.

### Use cases for Paths

There are several use cases that could benefit from UDF Paths with their separation of concerns between path navigation and value interpretation:

1. **Modular Data Access:** Imagine large and complex JSON structures representing different entities or data models. UDF Paths allow developers to define reusable paths for accessing specific data points within these structures. This separation promotes modularity and reduces code duplication. The receiving application can handle named values based on its context, keeping the path definition independent.

2. **Versioning and Flexibility:**  As data structures evolve, UDF Paths can remain relatively stable. Updates to the data model can be handled by the application logic interpreting the named value, without needing to modify the paths themselves. This enhances flexibility and simplifies future changes.

3. **Context-Specific Value Interpretation:** UDF Paths can be used for accessing data points that require context-specific interpretation. For instance, a path might access a numerical value representing a currency amount. The receiving application, understanding the context, can interpret the value based on the defined currency and potentially display it with a symbol or formatting.

4. **Customizable Data Processing:**  UDF Paths could be used in conjunction with data processing pipelines. By separating path navigation from value interpretation, developers can define standard paths for accessing data, while allowing for custom logic within the application to handle specific named values in different ways.

5. **Improved Code Readability:**  UDF Paths with named values can enhance code readability. Paths become self-documenting, clearly indicating the data point being accessed. The application context handles the interpretation of named values, keeping the path definition focused on navigation.

Overall, UDF Paths with their separation of path navigation and value interpretation offer a flexible and modular approach to accessing data within complex JSON structures. This can lead to improved code maintainability, easier handling of data model changes, and context-specific value processing.

### Path -grammar

```
path_value
   : ('Path' | '~') '(' (CONST_ID | (ID ':')* ID '(' ')')? (('.' ID) | ('[' NUMBER ']'))* ')'
   | '~' (CONST_ID | (ID ':')* ID '(' ')')? (('.' ID) | ('[' NUMBER ']'))+
   | '~' '(' (('.' ID) | ('[' NUMBER ']'))* ')'
   ;
```

* `Path` cannot be written in lowercase.

* Path may be empty, e.g. `Path()`.

* Path can be written in a shorthand form with `~`. E.g. `~.names[1]`.
  * This shorthand form does not support empty paths.

* Path can be written in a shorthand form with `~(...)`. E.g. `~(.names[1]`.
  * This shorthand form supports also empty paths, e.g. `~()`.

### Paths and one-based indexing

Returning all UDF Paths as an array (where each element is a `Path` object) from a zero-index can be a powerful feature with several potential use cases:

**1. Dynamic Schema Exploration:**

* Imagine a scenario where the application doesn't have predefined knowledge of the exact structure of incoming JSON data. By using a zero-index to get all UDF Paths, the application can dynamically explore the schema and identify the available keys and nested structures within the data. This allows for handling data with flexible or evolving schemas.

**2. Automatic Data Validation:**

* With all UDF Paths in hand, the application can define validation rules against these paths. It can then iterate through the paths and validate the presence of expected keys and data types at those locations within the JSON structure. This enables automated validation for ensuring data integrity.

**3. Generic Data Processing Pipelines:**

* By having all paths, you can design generic data processing pipelines that work on various JSON structures. The pipeline can iterate through the paths, extract the values, and perform the necessary operations without needing to be specific about the exact data layout. This promotes code reusability and simplifies handling diverse data formats. 

**4. Documentation Generation:**

* Extracting all UDF Paths can be useful for generating automatic documentation for the data format. The paths themselves act as comments, indicating the available data points within the JSON structure. This improves code readability and understanding for developers working with the data.

**5. Runtime Path Selection:**

* In some cases, the application might need to choose the appropriate path based on runtime conditions. Having all paths readily available allows for dynamic path selection based on specific criteria, offering greater flexibility in data access.

**Considerations:**

* As mentioned earlier, retrieving all paths can be computationally expensive for very large JSON structures. Implementing efficient algorithms for path retrieval is crucial.
* Defining clear semantics around how to handle arrays within arrays (nested structures) is important to avoid ambiguity when generating all paths.

Overall, while there are some challenges to consider, the ability to return all UDF Paths as an array offers a powerful approach for schema exploration, data validation, generic processing pipelines, and more. It promotes flexibility and simplifies handling diverse JSON data structures within your UDF framework.

### Book-analogy

Here's an analogy using zero-based indexing to return all UDF Paths, likened to a book and its table of contents:

**Imagine a book** with various chapters, sections, and subsections. The table of contents provides a roadmap to navigate the book's content.

* **Regular UDF Paths (one-based indexing):** This is like following specific entries in the table of contents. For instance, "Chapter 3.2" would be a UDF Path targeting a specific subsection within Chapter 3.

* **Zero-based indexing to return all UDF Paths:** This is analogous to getting a complete list of all entries from the table of contents, presented as an ordered list. Each entry in the list would represent a UDF Path.

Here's a breakdown of the analogy:

* **Book Structure:** The book represents the JSON data structure. Chapters are like top-level objects, sections like nested objects within chapters, and subsections like deeper levels of nesting.
* **Table of Contents Entries:** Each entry in the table of contents corresponds to a UDF Path. It specifies the path to navigate from the root of the book (JSON structure) to a specific piece of information.
* **Zero-Based Index:** The index in this analogy refers to the position of an entry within the complete list generated from the table of contents. Just like a table of contents starts with the first entry (index 0), the zero-based indexing for UDF Paths would provide all paths in an ordered list, starting from index 0.

**Benefits (like having a complete table of contents):**

* **Schema Exploration:** Just like skimming the table of contents gives you an idea of the book's structure, having all UDF Paths allows the application to understand the available data points and their nesting within the JSON structure.
* **Automatic Processing:** With a complete list of paths, the application can process the data in a generic way, iterating through each path and extracting the corresponding values. This is similar to how you can follow different sections of the book based on your needs.
* **Validation:**  Similar to checking the table of contents for expected chapters and sections, having all paths facilitates data validation. The application can ensure the presence of required data points within the JSON structure.

**Considerations (like limitations of a table of contents):**

* **Complexity:** A very large book with a complex table of contents might be overwhelming. Similarly, retrieving all UDF Paths for a massive JSON structure can be computationally expensive.
* **Clarity:** The table of contents might not provide details about the content within each section. Likewise, UDF Paths only specify the navigation steps, not the data type or meaning of the value at the end of the path.

In conclusion, zero-based indexing to return all UDF Paths offers a comprehensive approach to understanding and processing JSON data structures. It provides flexibility and simplifies handling diverse data formats, but considerations regarding efficiency and clarity need to be addressed when implementing this functionality within your UDF framework.
