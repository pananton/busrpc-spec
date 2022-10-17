# Style guide

Busrpc *proto* file style is very similar to the one suggested in the protocol buffer's official [documentation](https://developers.google.com/protocol-buffers/docs/style) (with some additions related to busrpc specifics). All places where busprc style guide violates the official one are explicitly stated.

* [Basic rules](#basic-rules)
* [File structure](#file-structure)
* [Protobuf entities naming](#protobuf-entities-naming)
  * [Enums naming](#enums-naming)
* [Busrpc entities naming](#busrpc-entities-naming)
* [Imports](#imports)
* [Documenting](#documenting)

## Basic rules

* Use proto3 syntax
* Keep the line length to 120 characters (80 in the official style guide)
* Use 2 spaces for indentation
* Prefer to place each protobuf `message` and `enum` in a separate file unless they are tightly coupled and not intended to be used separately

## File structure

All *proto* files must be structured in the following way:
* Line `syntax  =  "proto3";`
* Package name
* Imports (ordered, see [this](#imports) section)
* [File options](https://developers.google.com/protocol-buffers/docs/proto3#options)
* Type definitions

## Protobuf entities naming

* Name can only contain letters (a-zA-Z), digits (0-9) and underscores and should start with letter or underscore
* Package names must correspond to the directory hierarchy, for example file *dir1/dir2/file.proto* content must be placed to *dir1.dir2* package
* Use CamelCase for protobuf `message` name
* Use lower case underscore-separated names for protobuf `message` fields
* Use pluralized names for `repeated` fields

Example:
```
syntax = "proto3";
package dir1.dir2;

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

* Use lower case underscore-separated names for busrpc services, namespaces, classes and methods
* Busrpc structures, enumerations and descriptors correspond directly to protobuf `message` and `enum` types and should follow the rules specified by the [previous](#protobuf-entities-naming) section
* Methods that do not have a `Retval` usually can be considered as event sinks and may be prefixed with *on* for uniformity (for example *on_signed_in*, *on_signed_out*, etc.)

## Imports

* Always use `import public` 
* Use path relative to busrpc API root directory when importing *proto* files (for example, if file *dir1/dir2/file1.proto* should be imported to any other file (even in the same directory) do this with `import public "dir1/dir2/some.proto";`)
* If several files are imorted, they should be ordered by their scope (from narrowest to widest), unless explicitly stated otherwise (see next points):
  * method-scoped files
  * class-scoped files
  * namespace-scoped files
  * global files
* Inside the method description file *method.proto* import class description file *class.proto* first (even if method is static) and then all remainding files in the order specified above
* Inside the service description file *service.proto* import method description files after any other files; imported method description files should be ordered in the following way:
  * description files for methods, implemented by the service
  * description files for methods, invoked by the service

## Documenting

Busrpc API design follows code as documentation principle, which implies that *proto* files of the particular API should contain appropriate comments and [documentation commands](./busrpc.md#documentation-commands):
* Each class description file *class.proto* should start with a comment describing the class
* Each method description file *method.proto* should start with a comment describing the method
* Each service description file *service.proto* should start with a comment describing the service
* Each busrpc structure or enumeration (apart from self-describing `Params`, `Retval`, etc.) should be accompanied by a descriptive comment placed right before it
* Each structure field or enumeration constant should be accompanied by a descriptive comment placed right before it
* For each service method, service description file *service.proto* should have a comment right before corresponding `import` statement describing whether method is implemented, invoked or both by the service (see `\impl` and `\invk` [documentation commands](./busrpc.md#documentation-commands))

If some of the specified comments are missing, busrpc development tool will issue a warning when checking API for conformance.
