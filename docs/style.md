# Style guide

Busrpc *proto* file style is very similar to the one suggested in the protocol buffer's official [documentation](https://developers.google.com/protocol-buffers/docs/style) (with some additions related to busrpc specifics). All places where busprc style guide violates the official one are explicitly stated.

- [Basic rules](#basic-rules)
- [File structure](#file-structure)
- [Naming](#naming)
- [Enums naming](#enums-naming)
- [Busrpc entities naming](#busrpc-entities-naming)

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

## Naming

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

## Enums naming

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

* Use lower case underscore-separated names for busrpc services, classes and methods
* Methods that do not have a `Retval` usually can be considered as event sinks and may be prefixed with *on* (for example *on_signed_in*, *on_signed_out*, etc.)

## Imports

* Always use `import public` 
* Use path relative to repository root when importing *proto* files (for example, if file *dir1/dir2/some.proto* should be imported to any other file (even in the same directory) do this with `import public "dir1/dir2/some.proto"`)
* Files in *csmq* directory **can** be imported from everywhere
* Files in *csmq/<class_name>* (except for *class.proto*) **must not** be imported from files outside class directory
* Files in *csmq/<class_name>/<method_name>* (except for *method.proto*) **must not** be imported from files outside method directory

Imports order for *method.proto* file (inside each group order alphabetically):
* Import method's class *class.proto* first
* Import class-level types from *csmq/<class_name>* directory
* Import globally-visible types from *csmq* directory

Imports order for *service.proto* file:
* Import *method.proto* files for implemented methods
* Import *method.proto* files for invoked methods

# Documenting

Our [documentation generator](https://gitlab-595988116.camfrog.com/camshare/common/mq-docs) supports several [doxygen](https://www.doxygen.nl/)-like commands to facilitate documenting.

Method documentation commands (can be specified in *method.proto"):
* *\throws \<errc\> \<condition\>* - allows to explicitly describe condition when `csmq.Exception` with specified *errc* is thrown by the method (note that even in the absence of this command method still **may** throw (for example, because of network or database errors), *\throws* only used to document important parts of method contract)

Service documentation commands (can be specified in *service.proto*):
* *\author \<name\>* - service author
* *\email \<email\>* - service author email
* *\url \<url\>* - service repository URL
* *\docs \<url\>* - additional service documentation URL
* *\impl* - indicates that method is implemented/overloaded by the service
* *\invk* - indicates that method is invoked by the service
* *\qgroup \<name\>* - name of the NATS queue group (only for implemented methods)
* *\accepts \<extparam-name\> \<extparam-value\>* - specifies which value should have method's external parameter during the call to be accepted by the service (only for implemented methods); *extparam-value* can be expressed in an arbitrary way

## Guide

* The basic rule is that all protobuf messages (except for predefined types like `ClassInfo`, `Method`, `Params` etc., which are self-describing), their fields (except for predefined `Result` type fields), enumerations and enumeration values **must** be documented
* Use `//`-style comments
* Documentation comment is placed right before the entity to which it relates (note that documentation generator treats the firts line of multiline comment as brief description)
* Links to external resources are created with markdown notation: \[*link text*\]\(*link address*\)
* Links to CMAPI entities are created automatically for fully-qualified names (for example, *csmq.profile.update* string will be converted to a link that points to corresponding method documentation)
* Each *method.proto* file **must** contain a comment with method description at the top
* Each *service.proto* file **must** contain the following information about the service:
  * Description at the top of the file
  * Repository url specified in *\url* command
  * Author name specified in *\author* command
  * Author email specified in *\email* command
  * *\impl* command before `import` of each method implemented by the service
  * *\invk* command before `import` of each method invoked by the service
  * *\qgroup* command if method is implemented by a queue group
  * *\accepts* command(s) is service accepts only specific value of an external parameter(s)

# Testing

We provide [mq-client](https://gitlab-595988116.camfrog.com/camshare/common/mq-client) utility which allows developers to quickly test their services:
* call any CMAPI method
* accept any CMAPI method call and answer it with fixed reply
* monitor arbitrary parts of CMAPI

*Mq-client* utility uses JSON for input/output and encodes/decodes it to/form binary protobuf format automatically.

# Versioning

* CMAPI version uses *major.minor.patch* format of [semantic versioning](https://semver.org/)
* Change major version when you make incompatible changes
* Change minor version when you add functionality in a backwards compatible manner
* Change patch number when you make backwards compatible bug fixes

