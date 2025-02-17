# Records in UDL

[Our simple TodoEntry](../types/records.md) is defined in UDL as:

```idl
dictionary TodoEntry {
    boolean done;
    u64? due_date;
    string text;
};
```

All the usual types are supported.

# Object references

Our dictionary can refer to obects - here, a `User`

```idl
interface User {
    // Some sort of "user" object that can own todo items
};

dictionary TodoEntry {
    User owner;
    string text;
}
```

The Rust struct will have `owner` as an `Arc<>`.

## Default values for fields

Fields can be specified with a default value.

```idl
dictionary TodoEntry {
    boolean done = false;
    string text;
};
```

## Optional fields and default values

Fields can be made optional using a `T?` type.

```idl
dictionary TodoEntry {
    boolean done;
    string? text;
};
```

The corresponding Rust struct would need to look like this:

```rust
struct TodoEntry {
    done: bool,
    text: Option<String>,
}
```

### Optional null values

Optional fields can also be set to a default `null` value:

```idl
dictionary TodoEntry {
    boolean done;
    string? text = null;
};
```

# Interfaces in UDL

You expose our [`TodoList` example](../types/interfaces.md) in UDL like:

```idl
interface TodoList {
    constructor();
    void add_item(string todo);
    sequence<string> get_items();
};
```

The `constructor()` calls the Rust's `new()` method.

## Alternate Named Constructors

In addition to the default constructor connected to the `::new()` method, you can specify
alternate named constructors to create object instances in different ways. Each such constructor
must be given an explicit name, provided in the UDL with the `[Name]` attribute like so:

```idl
interface TodoList {
    // The default constructor makes an empty list.
    constructor();
    // This alternate constructor makes a new TodoList from a list of string items.
    [Name=new_from_items]
    constructor(sequence<string> items);
    // This alternate constructor is async.
    [Async, Name=new_async]
    constructor(sequence<string> items);
    ...
```

For each alternate constructor, UniFFI will expose an appropriate static-method, class-method or similar
in the foreign language binding, and will connect it to the Rust method of the same name on the underlying
Rust struct.

Constructors can be async, although support for async primary constructors in bindings is minimal.

## Traits

It's possible to have UniFFI expose a Rust trait as an interface by specifying a `Trait` attribute.

For example, in the UDL file you might specify:

```idl
[Trait]
interface Button {
    string name();
};

namespace traits {
    sequence<Button> get_buttons();
    Button press(Button button);
};
```

### Foreign implementations

Use the `WithForeign` attribute to allow traits to also be implemented on the foreign side passed into Rust, for example:

```idl
[Trait, WithForeign]
interface Button {
    string name();
};
```

would allow foreign implementations of that trait - eg, passing one back into Rust from Python:

```python
class PyButton(uniffi_module.Button):
    def name(self):
        return "PyButton"

uniffi_module.press(PyButton())
```

### Traits construction

Because any number of `struct`s may implement a trait, they don't have constructors.
# The UDL file

A UDL file allows you to define an interface externally from your Rust code.
It defines which functions, methods and types are exposed to the foreign-language bindings.

For example, here we describe a [namespace](../types/namespace.md) with 2 [records](../types/records.md) and an [interface](../types/interfaces.md)

```udl
namespace sprites {
  Point translate([ByRef] Point position, Vector direction);
};

dictionary Point {
  double x;
  double y;
};

dictionary Vector {
  double dx;
  double dy;
};

interface Sprite {
  constructor(Point? initial_position);
  Point get_position();
  void move_to(Point position);
  void move_by(Vector direction);
};
```
# Functions in UDL

All top-level *functions* get exposed through the UDL's `namespace` block.

The UDL file will look like:

```idl
namespace Example {
    string hello_world();
}
```

## Default values

Function arguments can be marked `optional` with a default value specified.

In the UDL file:

```idl
namespace Example {
    string hello_name(optional string name = "world");
}
```

## Async

Async functions can be exposed using the `[Async]` attribute:

```idl
namespace Example {
    [Async]
    string async_hello();
}
```
# Using types defined outside a UDL.

Often you need to refer to types described outside of this UDL - they
may be defined in a proc-macro in this crate or defined in an external crate.

You declare such types using:
```idl
typedef [type] [TypeName];
```
`TypeName` is then able to be used as a normal type in this UDL (ie, be returned from functions, in records, etc)

`type` indicates the actual type of `TypeName` and can be any of the following values:
* "enum" for Enums.
* "record", "dictionary" or "struct" for Records.
* "object", "impl" or "interface" for objects.
* "trait", "callback" or "trait_with_foreign" for traits.
* "custom" for Custom Types.

for example, if this crate has:
```rust
#[derive(::uniffi::Object)]
struct MyObject { ... }
```
our UDL could use this type with:
```
typedef interface MyObject;
```

# External Crates

The `[External="crate_name"]` attribute can be used whenever the type is in another crate - whether in UDL or in a proc-macro.

```
[External = "other_crate"]
typedef interface OtherObject;
```
# Errors in UDL

You expose our [error example](../types/errors.md) in UDL like:

```
[Error]
enum ArithmeticError {
  "IntegerOverflow",
};


namespace arithmetic {
  [Throws=ArithmeticError]
  u64 add(u64 a, u64 b);
}
```

Note that in the above example, `ArithmeticError` is "flat" - the associated
data is not exposed - the foreign bindings see this as a simple enum-like object with no data.
If you want to expose the associated data as fields on the exception, use this syntax:

```
[Error]
interface ArithmeticError {
  IntegerOverflow(u64 a, u64 b);
};
```

# Interfaces as errors

In our [error interface example](../types/errors.md) we are throwing the object `MyError`.

```idl
namespace error {
  [Throws=MyError]
  void bail(string message);
}

[Traits=(Debug)]
interface MyError {
  string message();
};
```
# Enums in UDL

[Our simple enum example](../types/enumerations.md) is defined in UDL as:

```idl
enum Animal {
  "Dog",
  "Cat",
};
```

## Enums with fields

Enumerations with associated data require a different syntax,
due to the limitations of using WebIDL as the basis for UniFFI's interface language.
An enum like `IpAddr` is specifiedl in UDL like:

```idl
[Enum]
interface IpAddr {
  V4(u8 q1, u8 q2, u8 q3, u8 q4);
  V6(string addr);
};
```

## Remote, non-exhaustive enums

One corner case is an enum that's defined in another crate and has the [non_exhaustive` attribute](https://doc.rust-lang.org/reference/attributes/type_system.html#the-non_exhaustive-attribute).

In this case, UniFFI needs to generate a default arm when matching against the enum variants, or else a compile error will be generated.
Use the `[NonExhaustive]` attribute to handle this case:

```idl
[Enum]
[NonExhaustive]
interface Message {
  Send(u32 from, u32 to, string contents);
  Quit();
};
```

**Note:** since UniFFI generates a default arm, if you leave out a variant, or if the upstream crate adds a new variant, this won't be caught at compile time.
Any attempt to pass that variant across the FFI will result in a panic.
# Docstrings

UDL file supports docstring comments. The comments are emitted in generated bindings without any
transformations. What you see in UDL is what you get in generated bindings. The only change made to
UDL comments are the comment syntax specific to each language. Docstrings can be used for most
declarations in UDL file

Docstrings in UDL are comments prefixed with `///`.

Docstrings are parsed as AST nodes, so incorrectly placed docstrings will
generate parse errors

## Docstrings in UDL
```java
/// The list of supported capitalization options
enum Capitalization {
    /// Lowercase, i.e. `hello, world!`
    Lower,

    /// Uppercase, i.e. `Hello, World!`
    Upper
};

namespace example {
    /// Return a greeting message, using `capitalization` for capitalization
    string hello_world(Capitalization capitalization);
}
```

## Docstrings in generated Kotlin bindings
```kotlin
/**
 * The list of supported capitalization options
 */
enum class Capitalization {
    /**
     * Lowercase, i.e. `hello, world!`
     */
    LOWER,

    /**
     * Uppercase, i.e. `Hello, World!`
     */
    UPPER;
}

/**
 * Return a greeting message, using `capitalization` for capitalization
 */
fun `helloWorld`(`capitalization`: Capitalization): String { .. }
```

## Docstrings in generated Swift bindings
```swift
/**
 * The list of supported capitalization options
 */
public enum Capitalization {
    /**
     * Lowercase, i.e. `hello, world!`
     */
    case lower

    /**
     * Uppercase, i.e. `Hello, World!`
     */
    case upper
}

/**
 * Return a greeting message, using `capitalization` for capitalization
 */
public func helloWorld(capitalization: Capitalization) -> String;
```

## Docstrings in generated Python bindings
```python
class Capitalization(enum.Enum):
    """The list of supported capitalization options"""

    LOWER = 1
    """Lowercase, i.e. `hello, world!`"""

    UPPER = 2
    """Uppercase, i.e. `Hello, World!`"""

def hello_world(capitalization: "Capitalization") -> "str":
    """Return a greeting message, using `capitalization` for capitalization"""
    ..
```
