# Style guide

Busrpc protobuf file style is very similar to the one suggested in the protocol buffer's official [documentation](https://developers.google.com/protocol-buffers/docs/style) (with some additions related to busrpc specifics). All places where busprc style guide violates the official one are explicitly stated.

* [Basic rules](#basic-rules)
* [File structure](#file-structure)
* [Protobuf entities naming](#protobuf-entities-naming)
  * [Enums naming](#enums-naming)
* [Busrpc entities naming](#busrpc-entities-naming)
* [Imports](#imports)
  * [Import order](#import-order)
* [Documenting](#documenting)

## Basic rules

* Use proto3 syntax
* Keep the line length to 120 characters (80 in the official style guide)
* Use 2 spaces for indentation
* Prefer to place each protobuf `message` and `enum` in a separate file unless they are tightly coupled and not intended to be used separately

## File structure

All protobuf files must be structured in the following way:
1. Line `syntax  =  "proto3";`
2. Package name
3. Imports (ordered, see [this](#imports) section)
4. [File options](https://developers.google.com/protocol-buffers/docs/proto3#options)
5. Type definitions

## Protobuf entities naming

* Name can only contain letters (a-zA-Z), digits (0-9) and underscores and must start with letter
* Top-level protobuf package name must be *busrpc*
* Package names must correspond to the directory hierarchy, for example types from the directory *dir1/dir2/dir3* must be placed to *busrpc.dir1.dir2.dir3* package
* Use CamelCase for protobuf `message` name
* Use lower case underscore-separated names for protobuf `message` fields
* Use pluralized names for `repeated` fields

Example:
```
syntax = "proto3";
package busrpc.dir1.dir2.dir3;

message SomeMessage {
  int32 some_field_1 = 1;
  string some_field_2 = 2;
  repeated int32 other_fields = 3;
}
```

### Enums naming

* Use CamelCase for `enum` name and upper case underscore-separated value names
* Add `enum` name as a prefix to all value names
* Every `enum` must contain a zero value (required by protobuf), however busrpc style guide do not recommend to use it as indication that value is not set (prefer `optional` instead) and do not require it to have `UNSPECIFIED` suffix (as the official style guide does)

Example:
```
enum Status {
  STATUS_OK = 0;
  STATUS_INTERNAL_ERROR = 1;
  STATUS_INVALID_INPUT = 2;
}
```

## Busrpc entities naming

* Use lower case underscore-separated names for busrpc namespaces, classes, methods and services
* Busrpc structures and enumerations correspond directly to protobuf `message` and `enum` types and must follow the rules specified by the [previous](#protobuf-entities-naming) section
* Methods that do not have a return value (not to be confused with methods that have empty return value!) usually can be considered an event sinks and may be prefixed with *on* for uniformity (for example, *on_signed_in*, *on_profile_updated*, etc.)

## Imports

* Use path relative to busrpc root directory  (the one which contains *api/* and *services/* directories) when importing protobuf files (for example, if file *dir1/dir2/file1.proto* should be imported in any other file (even in the same directory) do this with `import "dir1/dir2/some.proto";`)
* Service description file *service.proto* must import description files of all methods implemented or invoked by the service
* Prefer to follow next recommendations to guarantee that generated source files will contain all necessary types:
  * import *busrpc.proto* in every class description file *class.proto*
  * import class description file *class.proto* in the description files *method.proto* of every method of this class

### Import order

* If several files are imported, prefer to order them by their nesting level: first files from the same directory, then from the parent directory, etc.
* Prefer to order each group of imported files with the same nesting level alphabetically
* Inside the method description file *method.proto* prefer to import class description file *class.proto* first and then other files in the order specified above
* Inside the service description file *service.proto* prefer to import method description files in the following order:
  1. description files for methods, implemented by the service
  2. description files for methods, invoked by the service

## Documenting

Busrpc API should follow code as documentation principle, which implies that protobuf files should contain appropriate comments and [documentation commands](busrpc.md#documentation-commands). Documentation comment should appear **right before** the entity to which it relates. Busrpc development tool [`validate`]() command issues a warning if any rule from the list below is violated.

* Every non-predefined busrpc structure or enumeration should be documented with a comment describing it
* Predefined structures may not be documented unless explicitly requested by the rules of this section
* Every structure field or enumeration constant should be documented
* Every service/class/method descriptor (`ServiceDesc`, `ClassDesc` or `MethodDesc`) should be documented with a comment describing service/class/method purpose
* Every `import` statement for a method description file *method.proto* found in the service description file *service.proto* should be documented; documentation comment should contain information whether method is implemented or invoked by the service (see [service documentation commands](./busrpc.md#service-documentation-commands))
