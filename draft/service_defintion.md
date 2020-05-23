<p align="center"><img src="../images/RRheader2.jpg"></p>

# Robot Raconteur Service Definition Standard

Version 0.9.2

http://robotraconteur.com

http://github.com/robotraconteur

Copyright &copy; 2020 Wason Technology, LLC

*Robot Raconteur is a communication framework designed for use with robotics, automation, building control, and internet of things applications.*

## Abstract

*Service Definitions* specify value, object, and other types for use with Robot Raconteur services using a simple Interface Definition Language (IDL). This IDL has been designed to be easy to parse while providing enough flexibility for sophisticated applications.

## Introduction

Robot Raconteur uses a highly specialized object and data model to implement a Remote Procedure Call (RPC) framework. (The object and data models are discussed in [Robot Raconteur Object Protocol Specification](object_protocol.md) and [Robot Raconteur Value Types Specification](value_types.md).) The framework allows for a service to define object and data types, extending the capabilities of Robot Raconteur. It also allows for these object and data types to be shared between services by `importing` types. These types are defined in plain text *Service Definition* files using a simple Interface Definition Language (IDL). This IDL allows for defining variations of the following types:

* `constant`
* `enum`
* `exception`
* `struct`
* `pod`
* `namedarray`
* `object`

*Service Definition* files stored in a filesystem shall have the extension `.robdef`.

The remainder of this document will define the contents and syntax of a *Service Definition* file.


### Regex

Regex are used in this document to specify the syntax used in *Service Definition* files. These regex use the **PCRE* standard. Named groups are used for clarity. If a named group is not defined in a regex, it is assumed that the previous definition is included in the regex using `(?(DEFINE)` ... `)` to include the subroutine.

## Character Set

Service Definition files shall consist of the following characters as defined by the following regex character class:

    [A-Za-z0-9\r\n\t!"#$%&'\(\)*+,\-\.\/\:;<=>\?@\[\]\^_`,\{\|\}~ \\]

Either UNIX or Windows line ending styles may be used. Line ending style must be consistent within a file.

## Lines

Each line is a statement. Statements must be contained in one line, and more than one statement may not exist on the same line.

Lines may be continued with a backslash `\` followed by a newline.

Whitespace at the beginning and end of a line is ignored.

The following regex substitution command can be used to expand line continuations:

    s/\\\r?\n//gm

## Comments

Lines beginning with `#` shall be considered a comment. Comments must begin at the start of the line or after whitespace. They shall not begin after a statement. Comment regex:

    s/^[ \t]*#[ -~\t]*$//g

### Documentation Comments

All declarations in service definitions except for `stdver`, `import`, `using`, and `implements` may be preceded with documentation comment blocks. These blocks start with a double pound sign, `##`, and consists of multiple lines with the double pound sign. A single pound sign comment or a blank line will reset the comment block. (Blank lines may exist between the documentation block and the declaration, but comment lines clear the documentation.) Documentation comment regex:

    ^[ \t]*##([ -~\t]*)$


## Keywords

The following keywords are reserved:

`object` `end` `option` `service` `struct` `import` `implements` `field` `property` `function` `event` `objref` `pipe` `callback` `wire` `memory` `void` `int8` `uint8` `int16` `uint16` `int32` `uint32` `int64` `uint64` `single` `double` `string` `varvalue` `varobject` `exception` `using` `constant` `enum` `pod` `namedarray` `cdouble` `csingle` `bool` `stdver`

## Numeric Literals

Numeric literals are used in *Service Definition* files in specific places, such as constant declarations. Integers, positive integers, and floating point rationals are available for use. Integers may sometimes be in hexadecimal form. The following regex patterns are defined for use in later declaration patters:

Decimal Integer:

    (?'int'[+\-]?(?:0|[1-9]\d*))

Positive Decimal Integer:

    (?'pos_int'(?:[1-9]\d*))

Hexadecimal Integer:

    (?'hex_int'[+\-]?0x[\da-fA-F]+)

Hexadecimal Integer or Decimal Integer:

    (?'hex_or_int'(?&hex_int)|(?&int))

Floating Point Rational:

    (?'float'[+\-]?(?:(?:0|[1-9]\d*)(?:\.\d*)?|(?:\.\d+))(?:[eE][+\-]?\d+)?)

Any supported numeric literal:

    (?'number'(?&hex_int)|(?&float)|(?&int))

**All overflows or underflows shall be treated as errors.**

## Whitespace

Spaces and tabs are considered "blank" characters and match the following regex:

    (?'blank'[ \t])

Newlines may either be UNIX or Windows style. Blanks before and after newlines are ignored:

    (?'newline'(?&blank)*(\r?\n)(?&blank)*)

## Names

User-defined names are used throughout *Service Definition* files. These names must:

* Contain only letters, numbers, and underscores
* Not begin with a number
* Not begin or end with an underscore
* Not begin with "rr" or "robotraconteur" in any upper and lower case combination
  * Service name segments are exempt from this rule
* Not begin with:
  * `get_`
  * `set_`
  * `async_`
* Not be a keyword

Name Regex:

    (?'name'[a-zA-Z](?:\w*[a-zA-Z0-9])?)

## Service Definition Name

*Service Definitions* names allow multiple Robot Raconteur style names separated by periods. This allows for namespacing. An example of a service name is `com.robotraconteur.geometry`, which defines geometry data types necessary for robotics computations.

Service Definition name regex:

    (?'service_name'(?:(?&name))(?:\.(?&name))*)

## Fully Qualified Names

Names that are imported from other *Service Definitions* are qualified by prefixing the name with the *Service Definition* name separated by a dot. An example of a fully qualified name is `com.robotraconteur.geometry.Vector3`.

    (?'qualified_name'(?:(?&name))(?:\.(?&name))+)

## Types

Robot Raconteur uses *value* and *reference* types. *Value* types are always passed by value. This means that the contents of the value type are *copied* to another node when transmitted to another node. Objects in Robot Raconteur are *reference* types, meaning they are always passed by *reference* and are not copied between nodes. The *Object Reference* (or *Proxy*) is used to communicate with the object using RPC from other nodes.

### Value Types

*Value* types are used to store and communicate data. Robot Raconteur supports the following data types:

#### Primitives

Primitives are provided by Robot Raconteur and are used by themselves or combined to create other more complex types. They typically directly correspond to primitive types in other computer languages.

The following primitive types are supported:

| Type  |  Bytes/Element | Description |
| ----- | -------------- | ----------- |
| `void` | 0 | Void |
| `double` | 8 | Double precision floating point |
| `single` | 4 | Single precision floating point |
| `int8` | 1 | Signed 8-bit integer |
| `uint8` | 1 | Unsigned 8-bit integer |
| `int16` | 2 | Signed 16-bit integer |
| `uint16` | 2 | Unsigned 16-bit integer |
| `int32` | 4 | Signed 32-bit integer |
| `uint32` | 4 | Unsigned 32-bit integer |
| `int64` | 8 | Signed 64-bit integer |
| `uint64` | 8 | Unsigned 64-bit integer |
| `string` | 1 | UTF-8 string |
| `cdouble` | 16 | Complex double precision floating point |
| `csingle` | 8 | Complex single precision floating point |
| `bool` | 1 | Logical boolean |

*Primitives are non-nullable*
*`void` can only be used for returns from functions*

#### Structures

Structures are user-defined data types specified in *Service Definition* files. Structures vary in size, and contain named fields of any valid Robot Raconteur data type.

Structure types are assigned a name in *Service Definition* files.

*Structures are nullable*

#### Pods

Pods are user-defined data types similar to structures, but are more restricted to ensure that they have constant (or maximum) size. "Pod" is short for "Plain-old-data". Pods are can be thought of as data that can be stored in contiguous memory, such as a C structure array.

Pods may contain:

* numeric primitives
* numeric primitive fixed-length arrays
* numeric primitive limited-size arrays
* numeric multi-dimensional fixed-size arrays
* pods
* pods fixed-length arrays
* pods limited-size arrays
* pod multi-dimensional fixed-size arrays
* named arrays
* named array fixed-length arrays
* named array limited-size arrays
* named array multi-dimensional fixed-size arrays

Recursive types are not allowed.

Pod types are assigned a name in *Service Definition* files.

*Pods are non-nullable*

*All fields in pods are non-nullable*

#### Named Array

Named arrays are user-defined numeric arrays where each position of the array has a special meaning. For instance, A three element vector may be thought of a on array with three elements, or as a structure with fields (x,y,z). Named arrays are a union type allowing an array to be viewed as either an array or a structure.

Named arrays may contain:

* numeric primitives
* numeric primitive fixed-length arrays
* named arrays
* named arrays fix-length arrays

Recursive types are not allowed.

Named array types are assigned a name in *Service Definition* files.

*Named arrays are non-nullable.*

*All fields in named arrays are non-nullable.*

*All elements in a named array must have the same primitive type.*

#### Arrays

Arrays are a single-dimensional indexed sequence of values. *Numeric primitives*, *Pods*, and *Named Arrays* can be arrays. *strings*, *structures*, and *containers* cannot be arrays.

Arrays can be variable length, variable length with a maximum length, or have a fixed length.

#### Multi-dimensional Arrays

*Multi-dimensional arrays* are arrays that have *n* dimensions instead of one dimension as in standard arrays. *Multi-dimensional* arrays are typically stored in *Fortran*  (column-major) order, with the dimensions stored separately from the array. *Multi-dimensional arrays* can either be variable sized or fixed sized.

#### Container Types

*Container Types* allow for a collection of values to be stored. Robot Raconteur supports *Maps* with `int32` or `string` keys, and *Lists*. Any valid *primitive*, *structure*, *pod*, *namedarray*, and *array* can be stored in a container type. Containers cannot directly contain another container type.

#### Variant Types

A *variant* is a wildcard type that can take any valid Robot Raconteur type. The `varvalue` keyword is used to specify a variant value type.

### Object Types

Objects are user-defined types that contain members. Supported member types are `property`, `function`, `event`, `objref`, `pipe`, `callback`, `wire`, and memory. See [Robot Raconteur Object Protocol](object_protocol.md) for more details. Objects may also *implement* other objects to allow for polymorphism through inheritance.

Objects allow *polymorphism* using inheritance, meaning that any object that implements a given type can be treated as that type. While objects are typically strongly typed, the `varobject` keyword can be used to specify a wildcard type.

Object types can only be used with `objref` member declarations. When used with an `objref`, object types can be used as variable length arrays and maps.

### Type Syntax

The following syntaxes are valid for specifying a data type in a declaration:

Type with no array or container:

    type_name

Type with array of unspecified length:

    type_name[]

Type with array of specific length, where `array_length` must be a positive integer:

    type_name[array_length]

Type with array of maximum length, where `array_max_length` must be a positive integer:

    type_name[array_max_length-]

Type with list container:

    type_name{list}

Type with map container using `int32` keys:

    type_name{int32}

Type with map container using `string` keys:

    type_name{string}

Containers may also be combined with arrays, for example:

    type_name[]{list}

If a `struct`, `object`, `pod` or `namedarray` is imported (and does not have a `using` declaration), the fully qualified name must be used for `type_name`. For example, `com.robotraconteur.geometry.Vector3` is the fully qualified `type_name` for `Vector3`. Fully qualified names follow all of the above syntax rules for arrays and containers.

Type matching uses the following regex:

    (?'type'(?=(?!void))(?&name)(?:\.(?&name))*(?:\[(?:(?&pos_int)\-?|\*|(?&pos_int)(?:,(?&pos_int))|)\])?(?:\{(?:list|int32|string)\})?)

Voidable types match the following regex:

    (?'type_voidable'(?&name)(?:\.(?&name))*(?:\[(?:(?&pos_int)\-?|\*|(?&pos_int)(?:,(?&pos_int))|)\])?(?:\{(?:list|int32|string)\})?)

Types that can have generator containers match the following regex:

    (?'type_generator'(?=(?!void))(?&name)(?:\.(?&name))*(?:\[(?:(?&pos_int)\-?|\*|(?&pos_int)(?:,(?&pos_int))|)\])?(?:\{(?:list|int32|string|generator)\})?)

Types that can have generator containers and are voidable match the following regex:

    (?'type_generator_voidable'(?&name)(?:\.(?&name))*(?:\[(?:(?&pos_int)\-?|\*|(?&pos_int)(?:,(?&pos_int))|)\])?(?:\{(?:list|int32|string|generator)\})?)

## Service Definition Contents

*Service Definitions* contain a list declarations, with each declaration on a separate line. For the user-defined types `enum`, `struct`, `pod`, `namedarray`, and `object`, declarations can be grouped together to form *block* declarations. *Block* declarations are begun with one of the block type keywords, and terminated with the `end` keyword.

*Comments are ignored by the parser and do not affect the following rules on service definition contents.*

All high level names must be unique. All names within a block declaration must be unique.

### Service Name

The first declaration in a *Service Definition* must be the name of the service. The declaration line must match the following regex:

    ^(?&blank)*(?'service'service(?&blank)+(?&service_name))(?&blank)*$

This declaration must only appear once.

### Standard Version

The *Service Definition* standard version should follow the *Service Name*. (The version of this document.). The declaration line must match the following regex:

    ^(?&blank)*(?'stdver'stdver(?&blank)+\d+\.\d+(?:\.\d+)?)(?&blank)*$

This declaration must only appear once.

### Imports

*Service definitions* can *import* other service definitions. The contents of the other *Service Definitions* can be accessed using fully qualified names or with `using` statements.

The declaration line for imports must match the following regex:

    ^(?&blank)*(?'import'import(?&blank)+(?:(?&name))(?:\.(?&name))*)(?&blank)+$

This declaration can appear multiple times with unique arguments.

### Using

Using declarations allow for imported types to be used without fully qualifying the type name. Using declarations can either use the unqualified name of the type, or use `as` to rename the type locally (create an alias).

Using declarations must match the following regex:

    ^(?&blank)*(?'using'using(?&blank)+(?&qualified_name)(?:(?&blank)+as(?&blank)(?&name))?)(?&blank)*$

This declaration can appear multiple times with unique types. A single type cannot be aliased more than once with different local names.

### Constants

Constants are real numbers, real numeric arrays (single dimensional), or strings that have a fixed value and an assigned name.

Strings allow JSON character escape sequences.

Constant structures can be declared with fields pointing to other constants defined in the current service definition (not imported).

Constant declaration lines must match one of the following regex:

Scalar integer:

    ^(?&blank)*(?'const_int'constant(?&blank)+(u?int(?:8|16|32|64))(?&blank)+(?&name)(?&blank)+(?&hex_or_int))(?&blank)*$

Scalar floating point:

    ^(?&blank)*(?'const_float'constant(?&blank)+(single|double)(?&blank)+(?&name)(?&blank)+(?&float))(?&blank)*$

Integer arrays:

    ^(?&blank)*(?'const_int_array'constant(?&blank)+(u?int(?:8|16|32|64))\[\](?&blank)+(?&name)(?&blank)+\{((?&blank)*(?&hex_or_int)(?:(?&blank)*,(?&blank)*(?&hex_or_int))*)?(?&blank)*\})(?&blank)*$

Floating point array:

    ^(?&blank)*(?'const_float_array'constant(?&blank)+(single|double)\[\](?&blank)+(?&name)(?&blank)+\{((?&blank)*(?&float)(?:(?&blank)*,(?&blank)*(?&float))*)?(?&blank)*\})(?&blank)*$

Strings:

    ^(?&blank)*(?'const_string'constant(?&blank)+string(?&blank)+(?&name)(?&blank)+"(?:(?:\\"|\\\\|\\\/|\\b|\\f|\\n|\\r|\\t|\\u[\da-fA-F]{4})|[^"\\])*")(?&blank)*$

Structure:

    ^(?&blank)*(?'const_struct'constant(?&blank)struct(?&blank)+(?&name)(?&blank)+\{(?&blank)*(?:(?&name)(?&blank)*\:(?&blank)+(?&name)(?:(?&blank)*,(?&blank)*(?&name)(?&blank)*\:(?&blank)+(?&name))*(?&blank)*)?\})(?&blank)*$

All constants:

    ^(?&blank)*(?'constant'(?:(?&const_int)|(?&const_float)|(?&const_int_array)|(?&const_float_array)|(?&const_string)|(?&const_struct)))

### Exceptions

User-defined *exception* types can be defined in *service definitions*. User-defined exceptions are aliases to `RobotRaconteurRemoteException`.

Exception declaration lines must match the following regex:

    ^(?&blank)*(?'exception'exception(?&blank)+(?&name))(?&blank)*$

### Enumerations

Enumerations are aliases to `int32`. They may be used with containers but may not be used in arrays.

Enumerations declarations are block types, meaning they combine multiple declarations. Enumerations begin with the keyword `enum` followed by the name of the enumeration. Next, the elements defined by the enumeration. Finally, the enumeration is closed using the `end` keyword on the last line by itself. Blank lines and comment lines are ignored.

The first declaration line of an enumeration must match the following regex:

    ^(?&blank)*enum(?&blank)+(?&name)(?&blank)*$

The next lines contain the elements defined in the enum. Enumeration values must satisfy the following rules:

* Value must be within the valid range for an `int32`
* Value must be specified as a decimal or hexadecimal literal (using `0x` prefix notation and [0-9a-fA-F] characters)
* Decimal and hexadecimal may be specified as negative using dash symbol as prefix
* Elements may be directly specified using an equal symbol, or will be implicitly set by incrementing the previous element by one
* All elements must be separated by commas
* Elements may be on one or more lines

Enumeration elements must match the following regex:

    (?'enum_value'(?&name)(?:(?&blank)*=(?&blank)*(?&hex_or_int))?)

The first enumeration element must match the following regex:

    (?'enum_first_value'(?&name)(?:(?&blank)*=(?&blank)*(?&hex_or_int)))

The closing line of the enumeration must match the following regex:

    ^(?&blank)*end(?&blank)*$

The full multi-line enumeration must match the following regex:

    ^(?&blank)*(?'enum'enum(?&blank)+(?&name)(?&blank)*(?&newline)+(?&blank)*(?&enum_first_value)(?:(?&blank)*,(?&blank)*(?&newline)*(?&blank)*(?&enum_value))*(?&blank)*(?&newline)+(?&blank)*end)(?&blank)*$

### Modifiers

*Modifiers* may optionally be applied to *structure* fields, *pod* fields, *namedarray* fields, and *object* members. A list of modifiers is declared between square brackets, separated by commas, on the same line after the declaration of the *field* or *member*. Modifiers may either be a single word, such as `readonly`, or may contain parameters between parentheses, such as `mymodifier(10,34.4,myconstant)`. Parameters must be a constant integer scalar, a constant float scalar, or the name of a constant declared in the same scope. Currently used modifier names include `readonly`, `writeonly`, `unreliable`, `urgent`, `perclient`, `nolock`, and `nolockread`.

Modifier parameters must match the following regex:

    (?'modifier_param'(?:(?&number)|(?&name)))

Modifiers must match the following regex:

    (?'modifier'(?&name)(?:\((?&blank)*(?&modifier_param)(?:(?&blank)*,(?&blank)*(?&modifier_param))*(?&blank)*\))?)

Modifier lists must match the following regex:

    (?'modifier_list'\[(?&blank)*(?&modifier)(?:(?&blank)*,(?&blank)*(?&modifier))*(?&blank)*\])

### Structures

Structure declarations are block types, meaning they combine multiple declarations. Structures begin with the keyword `struct` followed by the name of the structure type. The next lines define the *fields* of the structure. Finally, the struct is closed using the `end` keyword on its own line. Blank lines and comment lines are ignored.

The first declaration line of a struct must match the following regex:

    ^(?&blank)*struct(?&blank)+(?&name)(?&blank)*$

The following lines contain one or more *field* declarations. *Field* declarations begin with the keyword `field`, followed by the data type of the field, and finally the name of the *field*. *Fields* may optionally have modifiers following the name contained between in square brackets, with each modifier separated by commas.

*Field* declaration lines must match the following regex:

    ^(?&blank)*(?'field'field(?&blank)+(?&type)(?&blank)+(?&name)(?:(?&blank)+(?&modifier_list))?)(?&blank)*$

The closing line of the struct must match the following regex:

    ^(?&blank)*end(?&blank)*$

Constants may be declared before fields using the same syntax as at *service definition* scope.

The full multi-line struct must match the following regex:

    ^(?&blank)*(?'struct'struct(?&blank)+(?&name)(?&blank)*(?&newline)+(?:(?&blank)*(?&constant)(?&blank)*(?&newline)+)*(?:(?&blank)*(?&field)(?&blank)*(?&newline)+)+(?&blank)*end)(?&blank)*$

### Pods

Pod declarations are block types, meaning they combine multiple declarations. Pods begin with the keyword `pod` followed by the name of the pod type. The next lines define the *fields* of the pod. Finally, the pod is closed using the `end` keyword on its own line. Blank lines and comment lines are ignored.

The first declaration line of a pod must match the following regex:

    ^(?&blank)*pod(?&blank)+(?&name)(?&blank)*$

The following lines contain one or more *field* declarations. *Field* declarations begin with the keyword `field`, followed by the data type of the field, and finally the name of the *field*. *Fields* may optionally have modifiers following the name contained between in square brackets, with each modifier separated by commas.

*Pods* have constrained type requirements. *Pod* field types must match the following regex:

    (?'type_pod_field'(?=(?!void))(?&name)(?:\.(?&name))*(?:\[(?:(?&pos_int)\-?|(?&pos_int)(?:,(?&pos_int)))\])?)

*Field* declaration lines must match the following regex:

    ^(?&blank)*(?'field_pod'field(?&blank)+(?&type_pod_field)(?&blank)+(?&name)(?:(?&blank)+(?&modifier_list))?)(?&blank)*$

The closing line of the pod must match the following regex:

    ^(?&blank)*end(?&blank)*$

Constants may be declared before fields using the same syntax as at *service definition* scope.

The full multi-line pod must match the following regex:

    ^(?&blank)*(?'pod'pod(?&blank)+(?&name)(?&blank)*(?&newline)+(?:(?&blank)*(?&constant)(?&blank)*(?&newline)+)*(?:(?&blank)*(?&field_pod)(?&blank)*(?&newline)+)+(?&blank)*end)(?&blank)*$

### Named array

Named array declarations are block types, meaning they combine multiple declarations. Named arrays begin with the keyword `namedarray` followed by the name of the named array type. The next lines define the *fields* of the named array. Finally, the namedarray is closed using the `end` keyword on its own line. Blank lines and comment lines are ignored.

The first declaration line of a namedarray must match the following regex:

    ^(?&blank)*namedarray(?&blank)+(?&name)(?&blank)*$

The following lines contain one or more *field* declarations. *Field* declarations begin with the keyword `field`, followed by the data type of the field, and finally the name of the *field*. *Fields* may optionally have modifiers following the name contained between in square brackets, with each modifier separated by commas.

*Named arrays* have constrained type requirements. *Named array* field types must match the following regex:

    (?'type_namedarray_field'(?=(?!void))(?&name)(?:\.(?&name))*(?:\[(?&pos_int)\])?)

*Field* declaration lines must match the following regex:

    ^(?&blank)*(?'field_namedarray'field(?&blank)+(?&type_namedarray_field)(?&blank)+(?&name)(?:(?&blank)+(?&modifier_list))?)(?&blank)*$

The closing line of the namedarray must match the following regex:

    ^(?&blank)*end(?&blank)*$

Constants may be declared before fields using the same syntax as at *service definition* scope.

The full multi-line namedarray must match the following regex:

    ^(?&blank)*(?'namedarray'namedarray(?&blank)+(?&name)(?&blank)*(?&newline)+(?:(?&blank)*(?&constant)(?&blank)*(?&newline)+)*(?:(?&blank)*(?&field_namedarray)(?&blank)*(?&newline)+)+(?&blank)*end)(?&blank)*$

### Objects

Object declarations are block types, meaning they combine multiple declarations. Objects begin with the keyword `object` followed by the name of the object type. The next lines define the *members* of the named array. Finally, the object is closed using the `end` keyword on its own line. Blank lines and comment lines are ignored.

The first declaration line of an object must match the following regex:

    ^(?&blank)*object(?&blank)+(?&name)(?&blank)*$

The following lines contain one or more *member* declarations. *Member* declarations may be one of eight member types. each type has its own syntax. The following subsections discuss the available member types.

After members are declared, the `end` statement is used to close the object.

    ^(?&blank)*end(?&blank)*$

#### Properties

Property declarations must match the following regex:

    ^(?&blank)*(?'property'property(?&blank)+(?&type)(?&blank)+(?&name)(?:(?&blank)+(?&modifier_list))?)(?&blank)*$

Currently used modifiers for properties are `readonly`, `writeonly`, `perclient`, `urgent`, `nolock`, and `nolockread`.

#### Functions

Function declarations may be non-generator or generator functions. A function is a generator function if the last parameter or the return type has a `{generator}` container type.

Function *parameters* must match the following regex:

    (?'function_param'(?&type)(?&blank)+(?&name))

A generator param must match the following regex:

    (?'function_param_generator'(?&type_generator)(?&blank)+(?&name))

Functions declarations must match the following regex:

    ^(?&blank)*(?'function'function(?&blank)+(?&type_generator_voidable)(?&blank)+(?&name)\((?&blank)*(?:(?:(?&function_param)(?&blank)*,(?&blank)*)*(?&function_param_generator))?(?&blank)*\)(?:(?&blank)+(?&modifier_list))?)(?&blank)*$

Currently used modifiers for properties are `urgent` and `nolock`.

#### Events

Event declarations must match the following regex:

    ^(?&blank)*(?'event'event(?&blank)+(?&name)\((?&blank)*(?:(?:(?&function_param)(?&blank)*,(?&blank)*)*(?&function_param))?(?&blank)*\)(?:(?&blank)+(?&modifier_list))?)(?&blank)*$

Currently the `urgent` modifier may be used with events.

#### Objref

Objref declarations use a subset of standard *type* syntax. The *type* must match the following regex:

    (?'type_objref'(?=(?!void))(?&name)(?:\.(?&name))*(?:\[\]|(?:\{(?:int32|string)\}))?)

Objref declarations must match the following regex:

    ^(?&blank)*(?'objref'objref(?&blank)+(?&type_objref)(?&blank)+(?&name)(?:(?&blank)+(?&modifier_list))?)(?&blank)*$

Currently no modifiers are used with objrefs.

#### Pipe

Pipe declarations must match the following regex:

    ^(?&blank)*(?'pipe'pipe(?&blank)+(?&type)(?&blank)+(?&name)(?:(?&blank)+(?&modifier_list))?)(?&blank)*$

Currently used modifiers for pipes are `readonly`, `writeonly`, `unreliable`, and `nolock`.

#### Callback

Callback declarations must match the following regex:

    ^(?&blank)*(?'callback'callback(?&blank)+(?&type_voidable)(?&blank)+(?&name)\((?&blank)*(?:(?:(?&function_param)(?&blank)*,(?&blank)*)*(?&function_param))?(?&blank)*\)(?:(?&blank)+(?&modifier_list))?)(?&blank)*$

Currently used modifiers for callbacks are `urgent`.

Callbacks may not be generator functions.

#### Wire

Wire declarations must match the following regex:

    ^(?&blank)*(?'wire'wire(?&blank)+(?&type)(?&blank)+(?&name)(?:(?&blank)+(?&modifier_list))?)(?&blank)*$

Currently used modifiers for wires are `readonly`, `writeonly`, and `nolock`.

#### Memory

Memory declarations use a subset of standard *type* syntax. The *type* must match the following regex:

    (?'type_memory'(?=(?!void))(?&name)(?:\.(?&name))*\[(?:|\*)\])

The base type must be a number, pod, or named array.

Memory declarations must match the following regex:

    ^(?&blank)*(?'memory'memory(?&blank)+(?&type_memory)(?&blank)+(?&name)(?:(?&blank)+(?&modifier_list))?)(?&blank)*$

Currently used modifiers for memories are `readonly`, `writeonly`, `nolock`, and `nolockread`.

#### Implements

Implements declarations must match the following syntax:

    ^(?&blank)*(?'implements'implements(?&blank)+(?:(?&name))(?:\.(?&name))*)(?&blank)*$

#### Constant

Constants may appear using the same syntax as at *service definition* scope.

#### Full Structure Syntax

The following regex will match any member type:

    (?'member'(?:(?&property)|(?&function)|(?&event)|(?&objref)|(?&pipe)|(?&callback)|(?&wire)|(?&memory)))

The full multi-line object declaration must match the following regex:

    ^(?&blank)*(?'object'object(?&blank)+(?&name)(?&blank)*(?&newline)+(?:(?&blank)*(?:(?&constant)|(?&implements))(?&blank)*(?&newline)+)*(?:(?&blank)*(?&member)(?&blank)*(?&newline)+)+(?&blank)*end)(?&blank)*$

### Deprecated syntax

* Older versions of Robot Raconteur used the `option` keyword. This is now deprecated, and should generate a warning and be ignored.
* Older versions of Robot Raconteur include the type of block declaration with end, for example the `end` declaration for a structure would be `end struct` and the `end` declaration for `end object`. This is now deprecated and should be rejected for service definitions version 0.9 and higher.

### Full Service Definition Syntax

The full *service definition* (containing no deprecated syntax) must match the following regex after line continuations are expanded and comments removed:

    ^(?&newline)*(?'robdef'(?&service)(?&newline)+(?&stdver)(?:(?&newline)+(?&import))*(?:(?&newline)+(?&using))*(?:(?&newline)+(?:(?&constant)|(?&exception)|(?&enum)))*(?:(?&newline)+(?:(?&struct)|(?&pod)|(?&namedarray)))*(?:(?&newline)+(?&object))*)(?&newline)*$

## Service Definition Verification

The following rules must be verified after a *Service Definition* has been parsed:

* Names must be valid as described in the [Names](#Names) section
* Service definition scope level names including the names of structs, pods, namedarrays, objects, exceptions, constants, enums, and using aliases, must be unique
* Block scope names including the names of constants, fields, members, and enumeration values must be unique within the block. They are not required to be unique against service definition scope names.
* Modifiers are valid and are unique, or have unique parameters if the modifier name is repeated. Unknown modifiers should be ignored and a warning generated.
* All imported service definitions and service definition types are available
* Imported service definitions do not have greater `stdver`
* All types are valid and follow the rules for each type and usage
* All implemented object types exist, and the implementing object exactly implements each member and constant of the implemented object. No implicit inheritance is assumed.
* All members and fields are valid

## Verification

It shall be assumed that `RobotRaconteurGen --verify-robdef` accurately interprets the *Service Definition* standard.

## Conventions

Some conventions are recommended for *Service Definition* file formatting:

* Service names should use Java style package names, using reverse domain name order. All letters should be lowercase.
* Enumeration, structure, pod, namedarray, and object names should be nouns with each internal word capitalized (UpperCamelCase)
* All letters in constant names should be capitalized with internal words separated with underscores (ALL_CAPS)
* Field names, member names, parameter names, modifier names, and enumeration values should use lowercase for letters, and separate each internal word with underscores (snake_case)
* *Service Definition* scope declaration should not be indented. Fields, members, and enum values should be indented four spaces. Tabs should not be used for indentation.
* It is suggested that lines be split after 79 characters using the line continuation character `\`
* Line continuations should be indented four spaces more than the line before the continuation. Additional line continuations should match the indentation of the first continuation.
* Comments should match the indentation of the relevant declaration

These conventions are loosely based on Python PEP 8
