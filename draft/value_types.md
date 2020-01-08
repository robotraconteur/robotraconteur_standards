<p align="center"><img src="../images/RRheader2.jpg"></p>

# Robot Raconteur Value Types Standard

http://robotraconteur.com

http://github.com/robotraconteur

Copyright &copy; 2020 Wason Technology, LLC

*Robot Raconteur is a communication framework designed for use with robotics, automation, building control, and internet of things applications.*

## Abstract

Robot Raconteur separates *value* from *reference* data types. See [Robot Raconteur Framework Architecture Standard](framework_architecture.md) for more details on the distinction between *value* and *reference* types. Value types are used to store and communication *data*, and are always passed *by value*. This means that the information stored in a value type is *copied* between nodes when it is transmitted. (This is in contrast to *reference* types, where a handle to the object is passed to the client, and is interacted through members.) The value types supported by Robot Raconteur are specialized for use in robotics, automation, and other engineering/scientific applications. This is in contrast to the more common frameworks which are intended for business and web applications. Value types fall into four categories: primitive, multi-dim array, structure, and container. This standard specifies what types are supported, and how they are stored in messages.

## Messages

Robot Raconteur transmits commands and data through *messages*. See [Robot Raconteur Framework Architecture Standard](framework_architecture.md) for an overview of messages, and how they are used by Robot Raconteur. This document will discuss how data types are represented in messages, but *not* how they are serialized to a binary stream. Converting a value type to a binary representation involves a two step process. The first step involves "packing" the data from the native data structure into a message. This generally involves creating a shallow copy of the data (and structure of data) into message element(s). The result is either a single message element for a primitive type, or a tree of message elements for more complicated types. This message is then passed to a transport, which will serialize the message to the appropriate format. On the receiving end, the transport will deserialize the incoming data into message(s). These message(s) can then be "unpacked" into the native data format. The separation of these two concepts provides additional flexibility to the architecture of Robot Raconteur.

*Message elements* have, at minimum, the following contents:

* ElementName or ElementNumber: A name string or int32 number for the element. Numbers may be used lists and int32 keyed maps for efficiency
* ElementType: A numeric code of the contained data's type
* ElementTypeName: A string of the data type for structure, pod, or named array
* Data: The data as either a numeric array or nested "Message Elements"

Different implementations or transports may have more fields to hold implementation specific metadata. These extra fields are not considered in this document.

## Primitive Types

The following primitive types are supported by Robot Raconteur message elements:

| Type | Bytes/Element | Description | Data Type Code |
| --- | --- | --- | --- |
| void | 0 | void/null | 0 |
| double | 8 | 64-bit floating point | 1 |
| single | 4 | 32-bit floating point | 2 |
| int8 | 1 | Signed 8-bit integer | 3 |
| uint8 | 1 | Unsigned 8-bit integer | 4 |
| int16 | 2 | Signed 16-bit integer | 5 |
| uint16 | 2 | Unsigned 16-bit integer | 6 |
| int32 | 4 | Signed 32-bit integer | 7 |
| uint32 | 4 | Unsigned 32-bit integer | 8 |
| int64 | 8 | Signed 64-bit integer | 9 |
| uint64 | 8 | Unsigned 64-bit integer | 10 |
| string | 1 | utf-8 encoded string | 11 |
| cdouble | 16 | 128-bit complex floating point | 12 |
| csingle | 8 | 64-bit complex floating point | 13 |
| bool | 1 | Logical boolean | 14 |

Primitive types are stored in *message elements* using the *Data* field.

Primitive types are always single-dimensional arrays when stored in *message elements*. Scalar numeric types are treated as arrays of length 1.

Strings are *always stored as arrays*. For this reason, the array notation cannot be used in service definitions for strings, since they are already stored as arrays. The array stored in a *message element* for *string* type contains the utf-8 encoded bytes. The length of the array is the number of bytes after encoding the string.  *void* (aka null) is treated as a zero length array.

Complex numbers are stored in interleaved format with the real component first, followed by the imaginary component.

*Message elements* containing primitive types always have the following format:

* ElementName or ElementNumber: Assigned by the parent *message element* or *message entry*
* ElementType: 1, 2, 3, 4, 5, 6, 7, 8, 10, 11, 12, 13, or 14 (The numeric *data type code* as specified in the above table.)
* ElementTypeName: Always empty
* Data: An array of numbers or bytes containing a utf-8 encoded string. This array must be empty for *void* data type.

Because primitives are always stored as an array, the message element cannot distinguish between scalars and single element arrays. The thunk code will convert appropriately using information from the service definitions.

## Multi-dim Array Types

Numeric types are always single-dimensional arrays when stored in a *message element*. *Multi-dim arrays* allow for arbitrary array shapes with greater than one dimension. This is accomplished by assuming that the single-dimensional array is a flattened multi-dimensional array stored in column-major order. The unflatenned dimensions are transmitted separately, allowing the receiver to reshape the array appropriately. Column-major order used is often referred to as "Fortran order". This is different than "C order", which flattens in the opposite index order.

Multi-dim arrays use a nested *message element* format to store the dimensions and flattened array.

* ElementName or ElementNumber: Assigned by the parent *message element* or *message entry*
* ElementType: 117
* ElementTypeName: Always empty
* Data: Nested message element list:
  * Nested Element 1:
    * ElementName: "dims"
    * ElementType: 8 (uint32[])
    * Data: The dimensions of the multi-dim array as a uint32[]. The "dims" array must have at least one entry.
  * Nested Element 2:
    * ElementName: "array"
    * ElementType: 1, 2, 3, 4, 5, 6, 7, 8, 10, 12, 13, or 14 (Any numeric primitive)
    * Data: Flattened numeric array. Length must match the product of all elements of "dims".

## Structure Types

Structures are user defined types specified in *service definitions*. See [Robot Raconteur Service Definitions](service_definition.md) for more details on how structures are specified. Structures are types that group items of any type into a single type using *fields* into a record. Each *field* stores a *value*. The type of the value stored in each field can be any valid Robot Raconteur data type.

Structures must know the name of the type as specified in the service definition. The type code is not enough enough information to reconstruct the structure. For this reason, the "ElementTypeName" contains the fully qualified name of the structure. For instance, for a camera image, the "ElementTypeName" would be "com.robotraconteur.image.Image". Note that the fully qualified name contains both the name of the service definition and the name of the structure. Passing an unqualified name like "Image" is forbidden.

Structures must never be stored in arrays.

Fields are stored a list of nested message elements, with each nested element name set to the corresponding field name as specified in the service definition. Fields *should* be in the order specified in the service definition, but are not required to be in the same order.

* ElementName or ElementNumber: Assigned by the parent *message element* or *message entry*
* ElementType: 101
* ElementTypeName: Fully qualified structure type name encoded as a utf-8 string
* Data: Nested message element list:
  * Nested Element 1..i..N:
    * ElementName: Name of field i encoded as a utf-8 string
    * ElementType: Any valid value type code
    * Data: Any valid value

## Pod Types

Pod types are user defined types specified in *service definitions*. See [Robot Raconteur Service Definitions](service_definition.md) for more details on how structures are specified. Pods are very similar to *structures*, but are intended to represent "flat" or "c-style" structures, where the entire structure is stored contiguously in memory and has a fixed byte length. These structures can be stored concurrently and memory, and treated as arrays. In C++, these are called "Plain Old Data" structures, and that is where the *POD* name comes from.

Pods may only contain the following data types:

* Numeric scalars
* Numeric arrays with fixed or maximum length
* Multi-dim numeric arrays with fixed dimensions
* Pod scalars
* Pod arrays with fixed or maximum length
* Pod multi-dim arrays with fixed dimensions
* Named array scalars
* Named arrays with fixed or maximum length
* Named multi-dim arrays with fixed or maximum length

Recursive use of pod or namedarray types is forbidden. It must be possible to determine the fixed or maximum size of a pod.

Pods are transmitted as arrays, similar to primitive types. Each array entry is stored as a nested element of the pod array message element, using the same format a structure. Unlike structures, pod fields must be in the order specified in the service definition. For each element, the "ElementTypeName" is left blank to save memory. The type name is determined from the parent message element. Array indices are zero-indexed.

* ElementName or ElementNumber: Assigned by the parent *message element* or *message entry*
* ElementType: 110
* ElementTypeName: Fully qualified pod type name encoded as a utf-8 string
* Data: Nested array of pod elements
  * Nested Element 1..i..M
    * ElementName or ElementNumber: Array index (i-1) as string in decimal format or int32
    * ElementType: 109
    * ElementTypeName: *[empty]*
    * Data: Nested message element list:
      * Nested Element 1..j..N:
        * ElementName: Name of field j encoded as a utf-8 string
        * ElementType: Any valid pod field value type code
        * Data: Any valid pod field value


Pods can also be stored as multi-dim arrays using a format similar to numeric multi-dim arrays.

* ElementName or ElementNumber: Assigned by the parent *message element* or *message entry*
* ElementType: 111
* ElementTypeName: Fully qualified pod type name encoded as a utf-8 string
* Data: Nested message element list:
  * Nested Element 1:
    * ElementName: "dims"
    * ElementType: 8 (uint32[])
    * Data: The dimensions of the multi-dim array as a uint32[]. The "dims" array must have at least one entry.
  * Nested Element 2:
    * ElementName: "array"
    * ElementType: 110
    * ElementTypeName: Fully qualified pod type name encoded as a utf-8 string
    * Data: Flattened elements of pod array. Length must match the product of all elements of "dims".
      * Nested Element 1..i..M
        * ElementName or ElementNumber: Array index (i-1) as string in decimal format or int32
        * ElementType: 109
        * ElementTypeName: *[empty]*
        * Data: Nested message element list:
          * Nested Element 1..j..N:
            * ElementName: Name of field j encoded as a utf-8 string
            * ElementType: Any valid pod field value type code
            * Data: Any valid pod field value

## Named Array Types

Named arrays are a union type between a structure and a numeric array. They are intended for use with data types which can be interpreted as an indexed vector or as a structure. An example is a three dimensional vector. A vector can be interpreted as either a 3x1 vector, or as three components (x,y,z). Named arrays formalize this concept, and have the advantage of significantly more efficient storage, since the data is stored as a plain numeric array.

Elements of a named array must all have the same underlying numeric type, since the data is stored as a plain numeric array. An array of named arrays will be a plain numeric array, with a length of N*k, where k is the number of numeric elements in each scalar named array. The numeric arrays are essentially "stacked".

Pods may only contain the following data types:

* Numeric scalars
* Numeric arrays with fixed length of the same type
* Named array scalars
* Named arrays with fixed length of the same underlying numeric type

Named arrays are defined in *service definitions*. Fully qualified names are stored with the named array for reconstruction.

Like numeric primitives and pods, named arrays are always transmitted as arrays of named arrays.

Named arrays are stored in messages using the following format:

* ElementName or ElementNumber: Assigned by the parent *message element* or *message entry*
* ElementType: 115
* ElementTypeName: Fully qualified named array type name encoded as a utf-8 string
* Data: Array data
  * Nested Element 1
    * ElementName: "array"
    * ElementType: 1, 2, 3, 4, 5, 6, 7, 8, 10, 12, 13, or 14 (Any numeric primitive)
    * Data: Array of named arrays as flattened numeric array. Length must be divisible number of elements in scalar named array.

Multi-dimensional named arrays can also be stored:

* ElementName or ElementNumber: Assigned by the parent *message element* or *message entry*
* ElementType: 116
* ElementTypeName: Fully qualified named array type name encoded as a utf-8 string
* Data: Nested message element list:
  * Nested Element 1:
    * ElementName: "dims"
    * ElementType: 8 (uint32[])
    * Data: The dimensions of the multi-dim array as a uint32[]. The "dims" array must have at least one entry.
  * Nested Element 2:
    * ElementName: "array"
    * ElementType: 115
    * ElementTypeName: Fully qualified named array type name encoded as a utf-8 string
    * Data: Array data
      * Nested Element 1
        * ElementName: "array"
        * ElementType: 1, 2, 3, 4, 5, 6, 7, 8, 10, 12, 13, or 14 (Any numeric primitive)
        * Data: Array of named arrays as flattened numeric array. Length must be divisible number of elements in scalar named array.

## Container Types

Robot Raconteur supports *map* and *list* value types. Maps are associative container, where each element in the container has a "key". Maps can either have key type "int32" or "string". Lists are similar to "int32" keyed maps, but the keys must begin at zero and increase by one for each entry. The list is presented to the user as a standard list rather than a map.

For maps with string keys, the elements are stored as nested elements, with the "ElementName" being the key.

* ElementName or ElementNumber: Assigned by the parent *message element* or *message entry*
* ElementType: 103
* Data: Nested message element list of map elements:
  * Nested Element 1..i..N:
    * ElementName: Map element key
    * ElementType: Any valid value type code
    * Data: Any valid value

For maps with int32 keys, the elements are stored as nested elements, with the "ElementName" being the key converted to decimal string format, or as an int32 "ElementNumber".

* ElementName or ElementNumber: Assigned by the parent *message element* or *message entry*
* ElementType: 102
* Data: Nested message element list of map elements:
  * Nested Element 1..i..N:
    * ElementName or Element Number: Map element key as decimal string or int32
    * ElementType: Any valid value type code
    * Data: Any valid value

For lists, the elements are stored as nested elements, with the "ElementName" being the index of the element converted to decimal string format, or as an int32 "ElementNumber". Lists are always zero-indexed.

* ElementName or ElementNumber: Assigned by the parent *message element* or *message entry*
* ElementType: 108
* Data: Nested message element list of map elements:
  * Nested Element 1..i..N:
    * ElementName or Element Number: (i-1) as decimal string or int32
    * ElementType: Any valid value type code
    * Data: Any valid value

## Enum Types

Enums are always stored as int32. There is no specialized message format for enums.

Enums are always stored as single element arrays. They can be stored in containers and structures, but may not be arrays, used in pods, or used in namedarray.

## `varvalue` Variant Type

`varvalue` is a wildcard variant value type that can be any valid Robot Raconteur value type. `variants` must be packed to a concrete type to be stored in messages, using the available valid types in Robot Raconteur messages. When unpacked, the type of the `variant` is determined from the "ElementType" and "ElementTypeName" information stored in the message elements. `varvalue` is never stored in messages, since any `varvalue` value must by definition be packed to a message element with concrete types.

## Element Type Code Table

Reference table for element type codes:

| Type | Element Type Code | Description
| --- | --- | --- |
| void | 0 | void/null |
| double | 1 | 64-bit floating point |
| single | 2 | 32-bit floating point
| int8 | 3 | Signed 8-bit integer |
| uint8 | 4 | unsigned 8-bit integer |
| int16 | 5 | Signed 16-bit integer |
| uint16 | 6 | Unsigned 16-bit integer |
| int32 | 7 | Signed 32-bit integer |
| uint32 | 8 | Unsigned 32-bit integer |
| int64 | 9 | Signed 64-bit integer |
| uint64 | 10 | Unsigned 64-bit integer |
| string | 11 | utf-8 encoded string |
| cdouble | 12 | 128-bit complex floating point |
| csingle | 13 | 64-bit complex floating point |
| bool | 14 | Logical boolean |
| struct | 101 | Structure |
| map{int32} | 102 | Map with int32 keys |
| map{string} | 103 | Map with string keys |
| list | 108 | List |
| pod | 109 | pod element (must be part of pod[]) |
| pod[] | 110 | pod array |
| pod[*] | 111 | pod multi-dim array |
| namedarray[] | 115 | array of named array |
| namedarray[*] | 116 | multi-dim array of named array |
| multidimarray | 117 | numeric multi-dim array |
