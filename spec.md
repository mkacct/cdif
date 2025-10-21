# cDIF specification

cDIF v1.0.1  
Madeline Kahn, 2025

## Rationale

cDIF (/ˈsiːdɪf/) is a textual data interchange format intended as an alternative to popular formats like JSON and YAML. It is designed to be easily readable and writable by users, as well as usable in many different software contexts. cDIF's notable attributes include:

* Intuitive syntax to JSON users
* Useful features seen in more recent formats
	* Line and block comments
	* Multiline strings
	* Reusable components
* Explicit type names for structures, usable when parsing to select custom logic
	* Usable to facilitate proper deserialization in strongly typed languages
* Trailing separators permitted, for clean diffs and easier editing
* Not opinionated regarding whitespace

The c stands for _cute_.

## Grammar

The included formal [grammar](syntax.cbnf) (written in [cBNF](https://github.com/mkacct/cbnf/blob/main/spec.md)) is part of this specification. It may be worth referencing for clarification on the syntax of certain constructs.

## Data syntax

The main unit of cDIF data is the **value**, which may be represented by either a **literal**, **structure**, or **component reference**. Literals and structures are described below; component references will be described in the [Components](#components) section.

### Literals

#### Numbers

Integers may be expressed in one of four bases:

* Decimal – no prefix (ex. `26`)
* Binary – prefixed with `0b` (ex. `0b11010`)
* Octal – prefixed with `0o` (ex. `0o32`)
* Hexadecimal – prefixed with `0x` (ex. `0x1A`)

Decimal integers may start with `0`. They should not be parsed as octal unless `0o` is used.

Floating-point numbers may only be expressed in decimal, with an optional exponential part (ex. `1.23e+4`). This notation requires a decimal point (ex. `3e5` is invalid).

`infinity` (case-sensitive) is supported as well.

Any number in any of these formats may be immediately preceded by a sign (`+` or `-`). `+` is equivalent to no sign.

Any number may use `_` as a separator for readability (ex. `123_456_789`). It must only appear directly between two digits.

All letters in these numeric formats (other than `infinity`) are case-insensitive.

#### Booleans

The special constants `true` and `false` (case-sensitive) are supported.

#### Characters

A character literal consists of a single character entity (except `'`) between single quotes (ex. `'A'`). A character entity is defined as a literal character (except `\`) or an escape sequence.

"Literal character" refers to any printable (non-control) character, or the horizontal tabulation.

The following escape sequences are supported:

* `\b` – backspace
* `\f` – form feed
* `\n` – newline
* `\r` – carriage return
* `\t` – tab
* `\v` – vertical tab
* `\'` – single quote
* `\"` – double quote
* `\\` – backslash
* `\/` – forward slash (supported but not required)
* `\uXXXX` – Unicode code point 
* `\UXXXXXXXX` – Unicode code point (8 hex digits)

#### Strings

cDIF supports four types of string literals:

A "standard" string literal consists of zero or more character entities (as defined previously, except `"`) between double quotes. (Ex. `"Hello\nWorld"`)

A "verbatim" string literal consists of zero or more literal characters (as defined previously, except `` ` ``) between backticks. Backslashes have no special meaning in this context. (Ex. `` `C:\Users` ``)

A "block" string literal consists of zero or more "multiline character entities" between triple double quotes (`"""`). A "multiline character entity" can be either a standard character entity, a newline (treated literally), or an escaped newline (i.e. a backslash followed by a newline). An escaped newline is called a "line continuation" and is considered to evaluate to the empty string. While likely rare, the delimiter may consist of more than three double quotes, requiring the same number of double quotes to close the string. Parsers should intuitively trim whitespace from the string in the manner described below.

A "verbatim block" string literal consists of zero or more literal characters (as defined previously) and/or newlines between triple backticks (` ``` `). Backslashes have no special meaning in this context. Like normal block strings, the delimiter may consist of more than three backticks, requiring the same number of backticks to close the string. The below whitespace handling rules also apply.

Whitespace in block strings is handled as follows. Trailing whitespace (at the ends of lines) should already have been stripped (see [File syntax](#file-syntax)). In non-verbatim block strings, all whitespace handling occurs before any escape sequences are processed (including line continuations).

1. If there are no line breaks, stop here; all whitespace is preserved.
2. If the first line contains no non-whitespace characters, remove the entire line including its following line break.
	* Otherwise, the first line is preserved as-is.
3. If the last line contains no non-whitespace characters, remove the entire line including its preceding line break.
	* Otherwise, the last line is treated as a normal line.
4. Finally, the following applies to all remaining non-empty lines (except for a potential preserved first line):
	* Determine the longest sequence of whitespace characters shared by all lines. Remove this sequence from the beginning of each line. (For example, if each line is indented by at least two tab characters, remove two tab characters from the beginning of each line.)

As an example of the above rules, the following two strings are equivalent:

```cdif
"""
    function foo() {
        return "hi";
    }

    let bar = "hello \
    world";
"""
```

```cdif
"function foo() {\n    return \"hi\";\n}\n\nlet bar = \"hello world\";"
```

#### Null

The special constant `null` (case-sensitive) is supported. A property with a null value may be treated differently than the absence of that property.

### Names

A **name** is used to identify an object property, type, or component. The characters allowed in a name are letters, digits (`0`-`9`), `$`, and `_`. Of these, the name's first character cannot be a digit or `$`. Names are case-sensitive.

The set of characters allowed as "letters" may be implementation-defined. However, all parsers must at least support the ASCII letters `a`-`z` and `A`-`Z`.

### Structures

The two types of structures are **objects** and **collections**. Both types of structure may be immediately preceded by a type identifier, which may be used by certain parsers depending on their configuration to determine how to deserialize the structure. A type identifier is a name as defined in the previous section, but may not be one of the following reserved words: `infinity`, `true`, `false`, `null`, `undef`.

The naming convention for type identifiers is `PascalCase`.  
A structure without a type identifier is said to be "anonymous".  
If a type identifier is provided without a following structure, the empty object (`{}`) is implied. (Ex. `{foo: Thing}` is equivalent to `{foo: Thing {}}`.)

#### Objects

An **object**, defined using curly braces, consists of zero or more mappings. A mapping consists of a property name (which is a name as defined in the previous section), followed by a colon, followed by a value. Multiple mappings may separated by either commas or semicolons (though they may not be mixed within a single object). A final trailing comma/semicolon is permitted but not required.

The naming convention for object properties is `camelCase`.  
Object mappings are not ordered and should never be parsed as such.

As an example, the following two objects are equivalent:

```cdif
Date {year: 2025, month: 4, day: 5}
```

```cdif
Date {
    year:  2025;
    month: 4;
    day:   5;
}
```

The same property name may be used multiple times in the same object. In this case, parsers should use the last assigned value for that property. (This behavior is primarily intended for use with the spread operator when working with components.)

`undef` is a special keyword that may be used in place of an object mapping's value. Parsers should consider a property with an `undef` value to be absent from the object. (For example, `{foo: undef}` is equivalent to `{}`.)

#### Collections

A **collection**, defined using square brackets, consists of zero or more values. Values may be separated by either commas or semicolons (though they may not be mixed within a single collection). A final trailing comma/semicolon is permitted but not required.

Collections are usually considered to be ordered, but may be parsed as unordered in certain contexts.

As an example, the following two collections are equivalent:

```cdif
["foo", "bar", "baz"]
```

```cdif
[
    "foo";
    "bar";
    "baz";
]
```

## File syntax

A cDIF file's main content is a single value, which is the only required part of the file. This value may be followed by an optional final semicolon. Optional whitespace is allowed between most tokens (see grammar for details). Other optional file contents are described below.

Trailing whitespace at the ends of lines should always be ignored (including within block strings). Parsers should strip all trailing whitespace as a preprocessing step.

### Comments

A comment may start anywhere arbitrary whitespace is permitted. C-style line comments and block comments are both allowed. Examples:

```cdif
// line comment
/* block comment
on two lines */
```

### Directives

A parser directive starts with `#` and must be the only content on its line. (Comments are not supported on directive lines.) The following directives are currently supported.

#### `# cDIF`

While technically optional, it is recommended that all cDIF files include this directive to explicitly indicate the cDIF version in use. For the current version, you would use it like this:

```cdif
# cDIF 1.0.1
```

The "cDIF" directive may only occur as the first line of the file.

#### `# components`

The "components" directive indicates the start of the components section, and will be described in the next section.

## Components

A **component** is a reusable value that may be referenced from the file's main value or other components.

### Defining components

Place the `# components` directive after the file's main value (and optional final semicolon) to indicate the start of the components section. The components section consists of exactly one anonymous object (which, like the main value, may be followed by an optional final semicolon). This object defines the file's components by mapping component names to their values.

### Referencing components

A component may be referenced anywhere a value is expected by inserting its name preceded by a dollar sign (ex. `$foo`). The value of the component is used in place of the reference.

Component references may be used in other components. Circular references are invalid, and parsers should throw an error if they occur.

### Spread operator

The spread operator (`...`) may be employed by preceding a component reference used in an object or collection. In an object, a spread component reference should be used in place of an entire mapping. (In a collection, it should be used in place of a value.) The spread operator is only valid if the destination context and referenced component are either both objects or both collections. When using the spread operator, any type identifier on the spread component will always be ignored.

Spread component references are not permitted in the top-level object of the components section itself.

As an example of component references (including spread component references), the following two cDIF files are equivalent:

```cdif
# cDIF 1.0.1
{
    name: $myName;
    displayColor: $hotPink;
    items: ["hat", ...$defaultItems, $specialItem];
    stats: {
        atk: 5;
        ...$defaultStats;
        def: 3;
        crv: 19;
    };
};

# components
{
    myName: "Maddie";
    hotPink: Color {red: $maxByte, green: 51, blue: 153};
    defaultItems: ItemList ["phone", "wallet", "keys"];
    defaultStats: {atk: 1, def: 1, hp: 20};
    specialItem: "cake";
    maxByte: 255;
};
```

```cdif
# cDIF 1.0.1
{
    name: "Maddie";
    displayColor: Color {red: 255, green: 51, blue: 153};
    items: ["hat", "phone", "wallet", "keys", "cake"];
    stats: {
        atk: 1;
        def: 3;
        hp: 20;
        crv: 19;
    };
};
```
