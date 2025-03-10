# JSON3.jl

*Yet another JSON package for Julia; this one is for speed and slick struct mapping*

[![Build Status](https://travis-ci.org/quinnj/JSON3.jl.svg?branch=master)](https://travis-ci.org/quinnj/JSON3.jl)
[![Codecov](https://codecov.io/gh/quinnj/JSON3.jl/branch/master/graph/badge.svg)](https://codecov.io/gh/quinnj/JSON3.jl)

**TL;DR**
```julia
# builtin reading/writing
JSON3.read(json_string)
JSON3.write(x)

# custom types
JSON3.read(json_string, T)
JSON3.write(x)
```

## Builtin types

The JSON format is made up of just a few types: Object, Array, String, Number, Bool, and Null.
In the JSON3 package, there are two main interfaces for interacting with these JSON types: 1) builtin and 2) struct mapping.
For builtin reading, called like `JSON3.read(json_string)`, the JSON3 package will parse a string or `Vector{UInt8}`,
returning a default object that maps to the type of the JSON. For a JSON Object, it will return a `JSON3.Object` type, which
acts like a Julia `Dict`, but has a more efficient representation. For a JSON Array, it will return a `JSON3.Array` type,
which acts like a Julia `Vector`, but also has a more efficient representation. If the JSON Array has homogenous elements,
the resulting `JSON3.Array` will be strongly typed accordingly. For the other JSON types (string, number, bool, and null),
they are returned as Julia equivalents (`String`, `Int64` or `Float64`, `Bool`, and `nothing`).

One might wonder why custom `JSON3.Object` and `JSON3.Array` types exist instead of just returning `Dict` and `Vector` directly. JSON3 employs a novel technique [inspired by the simdjson project](https://github.com/lemire/simdjson), that is a 
semi-lazy parsing of JSON to the `JSON3.Object` or `JSON3.Array` types. The technique involves using a type-less "tape" to note
the ***positions*** of objects, arrays, and strings in a JSON structure, while avoiding the cost of ***materializing*** such
objects. For "scalar" types (number, bool, and null), the values are parsed immediately and stored inline in the "tape". This can result in best of both worlds performance: very fast initial parsing of a JSON input, and very cheap access afterwards. It also enables efficiencies in workflows where only small pieces of a JSON structure are needed, because expensive objects, arrays, and strings aren't materialized unless accessed. One additional advantage this technique allows is strong typing of `JSON3.Array{T}`; because the type of each element is noted while parsing, the `JSON3.Array` object can then be constructed with the most narrow type possible without having to reallocate any underlying data (since all data is stored in a type-less "tape" anyway).

The `JSON3.Object` supports the `AbstactDict` interface, but is read-only (it represents a ***view*** into the JSON string input), thus it supports `obj[:x]` and `obj["x"]`, as well as `obj.x` for accessing fields. It supports `keys(obj)` to see available keys in the object structure. You can call `length(obj)` to see how many key-value pairs there are, and it iterates `(k, v)` pairs like a normal `Dict`. It also supports the regular `get(obj, key, default)` family of methods.

The `JSON3.Array{T}` supports the `AbstractArray` interface, but like `JSON3.Object` is a ***view*** into the input JSON, hence is read-only. It supports normal array methods like `length(A)`, `size(A)`, iteration, and `A[i]` `getindex` methods.

## Struct API

The builtin JSON API in JSON3 is efficient and simple, but sometimes a direct mapping to a Julia structure is desirable. JSON3 provides simple, yet powerful "struct mapping" techniques for a few different use-cases, using the `JSON3.StructType` trait specification.

In general, custom Julia types tend to be one of: 1) "data types", 2) "interface types" and sometimes 3) "abstract types" with a known set of concrete subtypes. Data types tend to be "collection of fields" kind of types; fields are generally public and directly accessible, they might also be made to model "objects" in the object-oriented sense. In any case, the type is "nominal" in the sense that it's "made up" of the fields it has, sometimes even if just for making it more convenient to pass them around together in functions.

Interface types, on the other hand, are characterized by ***private*** fields; they contain optimized representations "under the hood" to provide various features/functionality and are useful via interface methods implemented: iteration, `getindex`, accessor methods, etc. Many package-provided libraries or Base-provided structures are like this: `Dict`, `Array`, `Socket`, etc. For these types, their underlying fields are mostly cryptic and provide little value to users directly, and are often explictly documented as being implementation details and not to be accessed under warning of breakage.

What does all this have to do with mapping Julia structures to JSON? A lot! For data types, the most typical JSON representation is for each field name to be a JSON key, and each field value to be a JSON value. And when ***reading*** data types from JSON, we need to specify how to construct the Julia structure for the key-value pairs encountered while parsing. This can be considered a "direct" mapping of Julia struct types to JSON objects in that we try to map field to key directly.

For interface types, however, we don't want to consider the type's fields at all, since they're "private" and not very meaningful. For these types, an alternative API is provided where a user can specify the `JSON3.JSONType` their type most closely maps to, one of `JSON3.ObjectType()`, `JSON3.ArrayType()`, `JSON3.StringType()`, `JSON3.NumberType()`, `JSON3.BoolType()`, or `JSON3.NullType()`.

For abstract types, it can sometimes be useful when reading a JSON structure to say that it will be one of a limited set of related types, with a specific JSON key in the structure signaling which concrete type the rest of the structure represents. JSON3 provides functionality to specify a `JSON3.AbstractType()` for a type, along with a mapping of JSON key-type values to Julia subtypes that can be used to identify the concrete type while parsing.

### DataTypes

For "data types", we aim to directly specify the JSON reading/writing behavior with respect to a Julia type's fields. This kind of data type can signal its struct type in one of two ways:
```julia
JSON3.StructType(::Type{MyType}) = JSON3.Struct()
# or
JSON3.StructType(::Type{MyType}) = JSON3.Mutable()
```
`JSON3.Struct()` is less flexible, yet more performant. For reading a `JSON3.Struct()` from a JSON string input, each key-value pair is read in the order it is encountered in the JSON input, the keys are ignored, and the values are directly passed to the type at the end of the object parsing like `MyType(val1, val2, val3)`. Yes, the JSON specification says that Objects are specifically ***un-ordered*** collections of key-value pairs, but the truth is that many JSON libraries provide ways to maintain JSON Object key-value pair order when reading/writing. Because of the minimal processing done while parsing, and the "trusting" that the Julia type constructor will be able to handle fields being present, missing, or even extra fields that should be ignored, this is the fastest possible method for mapping a JSON input to a Julia structure. If your workflow interacts with non-Julia APIs for sending/receiving JSON, you should take care to test and confirm the use of `JSON3.Struct()` in the cases mentioned above: what if a field is missing when parsing? what if the key-value pairs are out of order? what if there extra fields get included that weren't anticipated? If your workflow is questionable on these points, or it would be too difficult to account for these scenarios in your type constructor, it would be better to consider the `JSON3.Mutable()` option.

The slightly less performant, yet much more robust method for directly mapping Julia struct fields to JSON objects is via `JSON3.Mutable()`. This technique requires your Julia type to be defined, ***at a minimum***, like:
```julia
mutable struct MyType
    field1
    field2
    field3
    # etc.

    MyType() = new()
end
```
Note specifically that we're defining a `mutable struct` to allow field mutation, and providing a `MyType() = new()` inner constructor which constructs an "empty" `MyType` where isbits fields will be randomly initialied, and reference fields will be `#undef`. (Note that the inner constructor doesn't need to be ***exactly*** this, but at least needs to be callable like `MyType()`. If certain fields need to be intialized or zeroed out for security, then this should be accounted for in the inner constructor). For these mutable types, the type will first be initizlied like `MyType()`, then JSON parsing will parse each key-value pair in a JSON object, setting the field as the key is encountered, and converting the JSON value to the appropriate field value. This flow has the nice properties of: allowing parsing success even if fields are missing in the JSON structure, and if "extra" fields exist in the JSON structure that aren't apart of the Julia struct's fields, it will automatically be ignored. This allows for maximum robustness when mapping Julia types to arbitrary JSON objects that may be generated via web services, other language JSON libraries, etc.

There are a few additional helper methods that can be utilized by `JSON3.Mutable()` types to hand-tune field reading/writing behavior:

* `JSON3.names(::Type{MyType}) = ((:field1, :json1), (:field2, :json2))`: provides a mapping of Julia field name to expected JSON object key name. This affects both reading and writing. When reading the `json1` key, the `field1` field of `MyType` will be set. When writing the `field2` field of `MyType`, the JSON key will be `json2`.
* `JSON3.excludes(::Type{MyType}) = (:field1, :field2)`: specify fields of `MyType` to ignore when reading and writing, provided as a `Tuple` of `Symbol`s. When reading, if `field1` is encountered as a JSON key, it's value will be read, but the field will not be set in `MyType`. When writing, `field1` will be skipped when writing out `MyType` fields as key-value pairs.
* `JSON3.omitempties(::Type{MyType}) = (:field1, :field2)`: specify fields of `MyType` that shouldn't be written if they are "empty", provided as a `Tuple` of `Symbol`s. This only affects writing. If a field is a collection (AbstractDict, AbstractArray, etc.) and `isempty(x) === true`, then it will not be written. If a field is `#undef`, it will not be written. If a field is `nothing`, it will not be written. 

### JSONTypes

For interface types, we don't want the internal fields of a type exposed, so an alternative API is to define the closest JSON type that our custom type should map to. This is done by choosing one of the following definitions:
```julia
JSON3.StructType(::Type{MyType}) = JSON3.ObjectType()
JSON3.StructType(::Type{MyType}) = JSON3.ArrayType()
JSON3.StructType(::Type{MyType}) = JSON3.StringType()
JSON3.StructType(::Type{MyType}) = JSON3.NumberType()
JSON3.StructType(::Type{MyType}) = JSON3.BoolType()
JSON3.StructType(::Type{MyType}) = JSON3.NullType()
```

Now we'll walk through each of these and what it means to map my custom Julia type to a JSON type.

#### JSON3.ObjectType

    JSON3.StructType(::Type{MyType}) = JSON3.ObjectType()

Declaring my type is `JSON3.ObjectType()` means it should map to a JSON object of unordered key-value pairs, where keys are `Symbol` or `String`, and values are any other type (or `Any`).

Types already declared as `JSON3.ObjectType()` include:
  * Any subtype of `AbstractDict`
  * Any `NamedTuple` type
  * Any `Pair` type

So if your type subtypes `AbstractDict` and implements its interface, then JSON reading/writing should just work!

Otherwise, the interface to satisfy `JSON3.ObjectType()` for reading is:

  * `MyType(x::Dict{Symbol, Any})`: implement a constructor that takes a `Dict{Symbol, Any}` of key-value pairs parsed from JSOn
  * `JSON3.construct(::Type{MyType}, x::Dict)`: alternatively, you may overload the `JSON3.construct` method for your type if defining a constructor is undesirable (or would cause other clashes or ambiguities)

The interface to satisfy for writing is:

  * `pairs(x)`: implement the `pairs` iteration function (from Base) to iterate key-value pairs to be written out to JSON
  * `JSON3.keyvaluepairs(x::MyType)`: alternatively, you can overload the `JSON3.keyvaluepairs` function if overloading `pairs` isn't possible for whatever reason

#### JSON3.ArrayType

    JSON3.StructType(::Type{MyType}) = JSON3.ArrayType()

Declaring my type is `JSON3.ArrayType()` means it should map to a JSON array of ordered elements, homogenous or otherwise.

Types already declared as `JSON3.ArrayType()` include:
  * Any subtype of `AbstractArray`
  * Any subtype of `AbstractSet`
  * Any `Tuple` type

So if your type already subtypes these and satifies the interface, things should just work.

Otherwise, the interface to satisfy `JSON3.ArrayType()` for reading is:

  * `MyType(x::Vector)`: implement a constructo that takes a `Vector` argument of values and constructs a `MyType`
  * `JSON3.construct(::Type{MyType}, x::Vector)`: alternatively, you may overload the `JSON3.construct` method for your type if defining a constructor isn't possible
  * Optional: `Base.IteratorEltype(::Type{MyType})` and `Base.eltype(x::MyType)`: this can be used to signal to JSON3 that elements for your type are expected to be a single type and JSON3 will attempt to parse as such

The interface to satisfy for writing is:

  * `iterate(x::MyType)`: just iteration over each element is required; note if you subtype `AbstractArray` and define `getindex(x::MyType, i::Int)`, then iteration is already defined for your type

#### JSON3.StringType

    JSON3.StructType(::Type{MyType}) = JSON3.StringType()

Declaring my type is `JSON3.StringType()` means it should map to a JSON string value.

Types already declared as `JSON3.StringType()` include:
  * Any subtype of `AbstractString`
  * The `Symbol` type
  * Any subtype of `Enum` (values are written with their symbolic name)
  * The `Char` type

So if your type is an `AbstractString` or `Enum`, then things should already work.

Otherwise, the interface to satisfy `JSON3.StringType()` for reading is:

  * `MyType(x::String)`: define a constructor for your type that takes a single String argument
  * `JSON3.construct(::Type{MyType}, x::String)`: alternatively, you may overload `JSON3.construct` for your type
  * `JSON3.construct(::Type{MyType}, ptr::Ptr{UInt8}, len::Int)`: another option is to overload `JSON3.construct` with pointer and length arguments, if it's possible for your custom type to take advantage of avoiding the full string materialization; note that your type should implement both `JSON3.construct` methods, since JSON strings with escape characters in them will be fully unescaped before calling `JSON3.construct(::Type{MyType}, x::String)`, i.e. there is no direct pointer/length method for escaped strings

The interface to satisfy for writing is:

  * `Base.string(x::MyType)`: overload `Base.string` for your type to return a "stringified" value

#### JSON3.NumberType

    JSON3.StructType(::Type{MyType}) = JSON3.NumberType()

Declaring my type is `JSON3.NumberType()` means it should map to a JSON number value.

Types already declared as `JSON3.NumberType()` include:
  * Any subtype of `Signed`
  * Any subtype of `Unsigned`
  * Any subtype of `AbstractFloat`

In addition to declaring `JSON3.NumberType()`, custom types can also specify a specific, ***existing*** number type it should map to. It does this like:
```julia
JSON3.numbertype(::Type{MyType}) = Float64
```

In this case, I'm declaring the `MyType` should map to an already-supported number type `Float64`. This means that when reading, JSON3 will first parse a `Float64` value, and then call `MyType(x::Float64)`. Note that custom types may also overload `JSON3.construct(::Type{MyType}, x::Float64)` if using a constructor isn't possible. Also note that the default for any type declared as `JSON3.NumberType()` is `Float64`.

Similarly for writing, JSON3 will first call `Float64(x::MyType)` before writing the resulting `Float64` value out as a JSON number.

#### JSON3.BoolType

    JSON3.StructType(::Type{MyType}) = JSON3.BoolType()

Declaring my type is `JSON3.BoolType()` means it should map to a JSON boolean value.

Types already declared as `JSON3.BoolType()` include:
  * `Bool`

The interface to satisfy for reading is:
  * `MyType(x::Bool)`: define a constructor that takes a single `Bool` value
  * `JSON3.construct(::Type{MyType}, x::Bool)`: alternatively, you may overload `JSON3.construct`

The interface to satisfy for writing is:
  * `Bool(x::MyType)`: define a conversion to `Bool` method

#### JSON3.NullType

    JSON3.StructType(::Type{MyType}) = JSON3.NullType()

Declaring my type is `JSON3.NullType()` means it should map to the JSON value `null`.

Types already declared as `JSON3.NullType()` include:
  * `nothing`
  * `missing`

The interface to satisfy for reading is:
  * `MyType()`: an empty constructor for `MyType`
  * `JSON3.construct(::Type{MyType}, x::Nothing)`: alternatively, you may overload `JSON3.construct`

There is no interface for writing; if a custom type is declared as `JSON3.NullType()`, then the JSON value `null` will be written.

### AbstractTypes

A final, uncommon option for struct mapping is declaring:
```julia
JSON3.StructType(::Type{MyType}) = JSON3.AbstractType()
```
When declaring my type as `JSON3.AbstractType()`, you must also define `JSON3.subtypes`, which should be a NamedTuple with subtype keys mapping to Julia subtype Type values. You may optionally define `JSON3.subtypekey` that indicates which JSON key should be used for identifying the appropriate concrete subtype. A quick example should help illustrate proper use of this `StructType`:
```julia
abstract type Vehicle end

struct Car <: Vehicle
    type::String
    make::String
    model::String
    seatingCapacity::Int
    topSpeed::Float64
end

struct Truck <: Vehicle
    type::String
    make::String
    model::String
    payloadCapacity::Float64
end

JSON3.StructType(::Type{Vehicle}) = JSON3.AbstractType()
JSON3.subtypekey(::Type{Vehicle}) = :type
JSON3.subtypes(::Type{Vehicle}) = (car=Car, truck=Truck)

car = JSON3.read("""
{
    "type": "car",
    "make": "Mercedes-Benz",
    "model": "S500",
    "seatingCapacity": 5,
    "topSpeed": 250.1
}""", Vehicle)
```
Here we have a `Vehicle` type that is defined as a `JSON3.AbstractType()`. We also have two concrete subtypes, `Car` and `Truck`. In addition to the `StructType` definition, we also define `JSON3.subtypekey(::Type{Vehicle}) = :type`, which signals to JSON3 that, when parsing a JSON structure, when it encounters the `type` key, it should use the value, in our example it's `car`, to discover the appropriate concrete subtype to parse the structure as, in this case `Car`. The mapping of JSON subtype key value to Julia Type is defined in our example via `JSON3.subtypes(::Type{Vehicle}) = (car=Car, truck=Truck)`. Thus, `JSON3.AbstractType` is useful when the JSON structure to read includes a "subtype" key-value pair that can be used to parse a specific, concrete type; in our example, parsing the structure as a `Car` instead of a `Truck`.