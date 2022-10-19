# Style guide

Busrpc *proto* file style is very similar to the one suggested in the protocol buffer's official [documentation](https://developers.google.com/protocol-buffers/docs/style) (with some additions related to busrpc specifics). All places where busprc style guide violates the official one are explicitly stated.

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

All *proto* files must be structured in the following way:
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
* Each `enum` must contain a zero value (required by protobuf), however busrpc style guide do not recommend to use it as indication that value is not set (prefer `optional` instead) and do not require it to have `UNSPECIFIED` suffix (as the official style guide does)

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

Each busrpc protobuf file should be self-contained, which means that including/importing it's generated source file also includes/imports generated source files of it's depenendecies. The rules specified in this section are aimed to achieve this.

* Use `import public`
* Use path relative to busrpc API root directory when importing *proto* files (for example, if file *dir1/dir2/file1.proto* should be imported to any other file (even in the same directory) do this with `import public "dir1/dir2/some.proto";`)
* Class description file *class.proto* must import framework-provided *busrpc.proto* file which contains busrpc built-in protobuf types and options
* Method description file *method.proto* must import method's class description file *class.proto* (even if method or class is static)
* Service description file *service.proto* must import description files of all methods implemented or invoked by the service

### Import order

* If several files are imported, prefer to order them by their nesting level: first files from the same directory, then from the parent directory, etc.
* Prefer to order each group of imported files with the same nesting level alphabetically
* Inside the method description file *method.proto* prefer to import class description file *class.proto* first and then other files in the order specified above
* Inside the service description file *service.proto* prefer to import method description files in the following order:
  1. description files for methods, implemented by the service
  2. description files for methods, invoked by the service

## Documenting

Busrpc API should follow code as documentation principle, which implies that *proto* files should contain appropriate comments and [documentation commands](./busrpc.md#documentation-commands).

* Each class description file *class.proto* should start with a comment describing the class
* Each method description file *method.proto* should start with a comment describing the method
* Each service description file *service.proto* should start with a comment describing the service
* Each busrpc structure or enumeration (apart from self-describing `Params`, `Retval`, etc.) should be accompanied by a descriptive comment placed right before it
* Each structure field or enumeration constant should be accompanied by a descriptive comment placed right before it
* For each service method, service description file *service.proto* should have a comment right before corresponding `import` statement describing whether method is implemented, invoked or both by the service (see `\impl` and `\invk` [documentation commands](./busrpc.md#documentation-commands))

If some of the specified comments are missing, busrpc development tool will issue a warning when checking API for conformance.
