#  Typed Object Notation (.ton)

TON is JSON, but with a friendlier syntax, simple to use schemas and efficient binary encoding.


## Example

    # This is a .ton file

    $database_server: string

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


# Syntax and features

All JSON syntax is accepted, and all TON values are also JSON values. In addition:

 * Top level curly braces may be omitted
 * Double quotes around field names matching `[A-Za-z0-9_]+` may be omitted
 * Comments are allowed `# comments last until the end of the line` 
 * Commas are optional
 * TON files can be checked against a schema (defined in a separate TON file)
 * TON files with a schema has a simple and very compact binary encoding
 * Typical sum type encoding such as `{"term": {"title": "hello"}}` has shorthand syntax `term(title: "hello")`
 * Parameters `$foo` are supported, as well as branching on their value `font_size: $font(big: 40, small: 20, 30)`


# Schemas

Schemas are written in TON files. The supported types are `int`, `float`, `string`, `bool`, `object`, `map` and `variant`.

    configuration: object(
        title: string
        owner: object(fields: {
            name: string
            dob: object(fields: {year: int, month: int, day: int})
        })
        query: variant(options: {
            term: map(values: type(name: "query"))
            match_all: object()
        })
        body: string
    )


# Binary encoding

The binary encoding uses the schema to avoid storing type information and field names in the binary data. TODO.
