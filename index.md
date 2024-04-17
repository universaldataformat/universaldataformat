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
| Boolean keys | Object can have value omitted. The key is interpreted as Boolean, and the value will be true | `{hidden}`
| Path-value   | Path-value can be declared. It is serialized as string | `{key: ~.otherKey[1].value}`
| Multiline String value       | Basic multiline String-value can be declared | `{key: """line1\n  line2\n  line3"""}`
| Multiline String value """\|  | Multiline String-value can be declared | `{key: """\|line1\n  line2\n  line3"""}`
| Multiline String value """>  | Multiline String-value can be declared | `{key: """>line1\n  line2\n  line3"""}`
| Raw-value    | Raw-value can be declared inside backticks | ```{key: `raw value`}```
| Metadata     | Object's field may have metadata (or configuration options). Metadata is declared in an object itself | `{mykey {hidden}: "my-value"}`
| Constraints, support level 1 | Constraints are supported, but they are not evaluated | `{myfield <String>: "my-value"}`
| Constraints, support level 2 | Constraints are supported and basic data-types are evaluated | `{myfield <String>: "my-value"}`
| Constraints, support level 3 | Constraints are supported and basic data-types with "And" and "Or" are evaluated | `{myfield <String Or Empty>: "my-value"}`
| Constraints, support level 4 | Constraints are supported with basic expressions | `{myfield <String And => length() < 20>: "my-value"}`
| Constraints, support level 5 | Constraints are supported with rich expressions | `{myfield <String And => in("foo", "bar", "baz") < 20>: "my-value"}`

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
   : (STRING | ID) ':' value
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
