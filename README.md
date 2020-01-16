#  Typed Object Notation (.ton)

TON is JSON, but with a friendlier syntax, simple to use schemas and efficient binary encoding.

All JSON syntax is accepted, and all TON values are also JSON values. In addition:

 * Top level curly braces may be omitted
 * Double quotes around field names matching `[A-Za-z0-9_]+` may be omitted
 * Comments are allowed `# comments last until the end of the line` 
 * Commas are optional
 * TON files can be checked against a schema (defined in a separate TON file)
 * TON files with a schema has a simple and very compact binary encoding
 * Typical sum type encoding such as `{"term": {"title": "hello"}}` has shorthand syntax `term(title: "hello")`
 * Variables `$foo` are supported, as well as branching on their value `font_size: $font(big: 40, small: 20, 30)`

## Example

    # This is a .ton file

    path: "/tmp"

    database: {
        server: "192.168.1.42"
        port: 2345
        max_connections: 5000
        enabled: true
    }

    hosts: [
      "tiger"
      "bobcat"
    ]

    query: term("Name": "Mike")
