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
* Busrpc structures and enumerations correspond directly to protobuf `message` and `enum` types and should follow the rules specified by the [previous](#protobuf-entities-naming) section
* `message` types that have special meaning in busrpc API design (`ClassDesc`, `MethodDesc`, `ObjectId`, `Params`, `Retval` and `Result`) should be treated in the same way as general busrpc structures for the purpose of naming, i.e. should follow the rules specified by the [previous](#protobuf-entities-naming) section
* Methods that do not have a `Retval` usually can be considered as event sinks and may be prefixed with *on* (for example *on_signed_in*, *on_signed_out*, etc.)

## Imports

* Always use `import public` 
* Use path relative to busrpc API root directory when importing *proto* files (for example, if file *dir1/dir2/file1.proto* should be imported to any other file (even in the same directory) do this with `import public "dir1/dir2/some.proto";`)
* Files in the namespace directory contain globally visible types and can be imported from everywhere
* Files in the class directory (apart from class description file *class.proto*) contain class private types and should only be imported from other files in the class directory and/or class method directories
 * Files in the method directory (apart from method description file *method.proto*) contain method private types and should only be imported from other files in the method directory

Inside the method descritpion file *method.proto* always import class description file *class.proto* first (even for static methods) and then the following files when needed (inside each group order alphabetically):
* method private types from the method directory
* class private types from the class directory
* global types from namespace directory

Inside the service description file *service.proto* order imported method description files in the following way:
* description files for methods, implemented by the service
* description files for methods, invoked by the service

## Documenting

Busrpc API design follows code as documentation principle, which implies that *proto* files of the particular API should contain appropriate comments and [documentation commands](./busrpc.md#documentation-commands):
* Each class description file *class.proto* should start with a descriptive comment
* Each method description file *method.proto* should start with a descriptive comment
* Each service description file *service.proto* should start with a descriptive comment
* Each protobuf `message` or `enum` (apart from self-describing types `ClassDesc` and `MethodDesc` (and it's nested types)) should be accompanied by a descriptive comment placed right before it
* Each `message` field or `enum` constant should be accompanied by a descriptive comment placed right before it
* For each service method, service description file *service.proto* should have a comment right before corresponding `import` statement describing whether method is implemented, invoked or both by the service (see `\impl` and `\invk` [documentation commands](./busrpc.md#documentation-commands))

If some of the specified comments are missing, busrpc development tool will issue a warning when checking API for conformance.
