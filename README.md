#  JSON X (.jsonx)

JSON X is JSON, but with a friendlier syntax, simple to use schemas and efficient binary encoding.


## Example

    # This is a .jsonx file

    $database_server: string(default: "localhost")

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

JSON is great, but it has a number of pain points. 

It's somewhat cumbersome to write JSON by hand due to the quotes around field names, lack of comments etc. 

Projects that use JSON for configuration often have vague documentation, with example snippets of JSON you don't know where to put in your own configuration file, and no way to check if it's valid. 

For APIs, JSON files are quite wasteful, since field names are constantly repeated.

In addition, it's common to end up with a bunch of configuration files that are tiny variations of each other for slightly different use cases.

JSON X aims to remove all of these pain points.

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

Schemas are written in TON files. The supported types are `int`, `float`, `string`, `bool`, `array`, `object`, `map`, `variant` and `json`.

    configuration: object(fields: {
        title: string()
        owner: object(fields: {
            name: string()
            dob: object(fields: {year: int(), month: int(), day: int()})
        })
        query: type(url: "query")
        body: string()
    })

    query: variant(options: {
        term: map(contains: string())
        bool: object(fields: {
            must: array(contains: type(url: "query"), required: false)
            must_not: array(contains: type(url: "query"), required: false),
            should: array(contains: type(url: "query"), required: false),
            minimum_should_match: int(default: 1)
        }) 
        match_all: object()
    })


# Binary encoding

The binary encoding uses the schema to avoid storing type information and field names in the binary data. TODO.
