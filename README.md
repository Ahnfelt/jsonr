#  JSONR (.jsonr)

JSONR is JSON, but with concise syntax, simpler schemas and efficient binary encoding.


## Example

    # This is a .jsonr file
    
    $database_server: "localhost"

    path: "/tmp"

    database: {
        server: $database_server
        port: 2345
        max_connections: 5000
        enabled: true
    }

    hosts: [
      "tiger"
      "bobcat"
    ]

    query: term("Name": "Mike")



## Why?

JSON is a great format, but it has a number of pain points. 

It's somewhat cumbersome to write JSON by hand due to the required quotes around field names and the inability to move things around in the file without editing commas. 

The lack of comments means that you can't specify why something is like it is in a JSON configuration file.

Projects that use JSON for configuration often have vague documentation, with example snippets of JSON you don't know where to put in your own configuration file, and no way to check if it's valid. 

For APIs, JSON files are quite wasteful, since field names are constantly repeated.

In addition, it's common to end up with a bunch of configuration files that are tiny variations of each other for slightly different use cases.

JSONR aims to remove all of these pain points, while strictly adhering to the JSON value model.


## Why not YAML, TOML, XML?

These formats have complex syntaxes that are hard to parse, and they each deviate from the simple JSON value model. 

In contrast, JSONR values and JSON values are one and the same, and the syntax is a conservative extension of JSON.


## Syntax and features

All JSON syntax is accepted, and all JSONR values are also JSON values. In addition:

 * Top level curly braces may be omitted
 * Double quotes around field names and strings matching `[A-Za-z_][A-Za-z0-9_]*` may be omitted
 * Comments are allowed: `# comments last until the end of the line` 
 * Commas are optional
 * JSONR files can be checked against a schema (defined in a separate JSONR file)
 * JSONR files with a schema has a simple and very compact binary encoding
 * Typical sum type encoding such as `{"term": {"title": "hello"}}` has shorthand syntax `term(title: "hello")`
 * Parameters `$foo` are supported, as well as branching on their value `font_size: $font(big: 40, small: 20)`

## Parameters

Parameters allow a JSONR file to be a template for e.g. configurations. They are declared in the top of the file with the syntax `$my_parameter: default_value`, where `default_value` is the default value of the parameter. A parameter must not be declared twice. 

Immediately after its declaration, a parameter can be used in place of a value. The value is then the value of the parameter.

It's possible to branch on parameters, using the syntax `$my_parameter(value1: 10, value2: 20)`. This means that if `$my_parameter` is the string `"value1"`, return `10`. Otherwise, if it's `"value2"`, return `20`. Otherwise, fail.

When parsing a JSONR file with parameter declarations, a map of parameter values may be supplied that is then used instead of the default value of the parameter declaration.


## Schemas

Schemas are written in JSONR files. The supported types are `int`, `float`, `string`, `bool`, `array`, `object`, `map`, `variant`, `binary` and `any`.

    configuration: object(of: {
        title: string()
        owner: object(of: {
            name: string()
            dob: object(of: {year: int(), month: int(), day: int()}, required: ["year"])
        }, required: ["name"])
        query: type(of: "query")
        body: string()
    })

    query: variant(of: {
        term: map(of: string())
        bool: type(of: "bool_query")
        match_all: object(of: {})
    })
    
    bool_query: object(of: {
        must: array(of: type(of: "query"))
        must_not: array(of: type(of: "query"))
        should: array(of: type(of: "query"))
        minimum_should_match: int()
    }) 
    

## Parsing JSONR

A JSONR file consists of zero or more parameter definitions followed by either a value or zero or more fields. 

```
file = {'$' string ':' value [',']} (value | fields)
parameter = '$' string ['(' fields ')']
value = parameter | json_string | json_number | object | array | 'true' | 'false' | 'null'
string = /[A-Za-z_][A-Za-z0-9_]*/ | json_string
object = '{' fields '}' | string '(' (value | fields) ')'
array = '[' {value [',']} ']'
fields = {string ':' value [',']}
```

The `json_number` and `json_string` rules are exactly as JSON numbers and JSON strings respectively. Whitespace is as in JSON and comments begin with `#` and last to the end of the file or the end of the line `\n`, whichever comes first.

|Variant shorthand|Equivalent JSON|
| :------ | :------------ |
| `foo()` | `{"foo": {}}` |
| `foo(x: 1, y: 2)` | `{"foo": {"x": 1, "y": 2}}` | 
| `foo(42)` | `{"foo": 42}` | 


## Binary encoding

The binary encoding uses the schema to avoid storing type information and field names in the binary data. TODO.
