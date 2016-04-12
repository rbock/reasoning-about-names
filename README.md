# Introduction
Names are important in programming. Probably every programmer learns this rather early in his or her career.

Still, when we have a `std::pair`, its members are called `first` and `second`, regardless of what they represent.
The members of a tuple do not even have a specified name. They can only be accessed via `std::get<>`. This is nice for generic code, but pretty bad for humans.

There are quite a few examples showing the wish for constructing classes/structs with compile-time configurable names, e.g.

  - https://github.com/ericniebler/range-v3 (tagged pair, tagged tuple)
    Uses pre-processor macro to create tag classes which allow to access members via member functions with the name given to the macro. Functions are pulled into the tagged pair/tuple via variadic inheritance.
  - https://github.com/rbock/sqlpp11 (named table columns, named result fields)
    Uses pre-processor macro to create template base classes that contain data members of a certain type and the name given to the macro. Data members are pulled into the table and row structs via variadic inheritance.
  - http://www.boost.org/doc/libs/1_60_0/libs/fusion/doc/html/index.html (BOOST_FUSION_ADAPT_STRUCT)
    Uses pre-processor macro to create compile-time iteratable structures that correspond to structs, allowing transformations back and forth.
  - https://github.com/duckie/named_types (named tuple elements)
    Uses string literals from N3599 to create types from ""names"", this way string literals can be used to create named tuples, whose elements can then be accessed via string literals. This does not require pre-processor macros, but results in a more awkward syntax for developers. It also requires the literal operator to be known in the current namespace.

This proposal presents the idea of name literals that could be used to create

  - variadic composition
  - named tuples
  - named function arguments

Assuming compile-time reflection being in place, it could also be used for the following use cases from N3814 (Call for Compile-Time Reflection Proposals):

  - compiler-generated comparison operators for classes/structs
  - type converson, e.g. struct of arrays <-> array of structs

# Language additions
## std::name
std::name is an implementation-defined type that represents C++ names (see [basic]).

A constexpr object of std::name preceded by a $ is to interpreted as the name it represents.

A C++ name, enclosed by $"..." is a constexpr literal of type std::name.

Name literals are evaluated into names in the context in which they defined, e.g.

```C++
using T = int;
constexpr auto A = $"int";
constexpr auto B = $"T";

static_assert(A == B);
```

### Synopsis

#### Constructor
```C++
constexpr name(const name&);
constexpr name(std::name&&);
```

#### Member functions
```C++
constexpr auto char_array() const -> std::array<char, N>;
```

N being the number of characters required to represent the name in utf-8.

#### Non-member functions
constexpr auto operator==(const std::name&, const std::name&) -> bool;
constexpr auto operator<(const std::name&, const std::name&) -> bool;
constexpr auto operator+(const std::name&, const std::name&) -> std::name;

### Hello World
With names being available as static constexpr values, it is possible to parameterize names of data members. E.g.

```C++
template<std::name x>
struct sample
{
	int $x; // use the name that x represents
};

using my_sample = sample<$"hello_world">;

static_assert(std::is_same_v(my_sample::hello_world, int), "");
```

## Variadic Composition
C++ Gurus tell us things like

  - Prefer composition to inheritance (Herb Sutter)
  - Inheritance is the base class of evil (Sean Parent)

But while we have single variadic inheritance (used for inside of tuples, for instance), we have no variadic composition.
The reason is rather obvious: Every member of a class/struct needs to have a name. If we had name literals as sketched above, we could easily add variadic composition to the language, e.g. like this

```C++
template<typename Type, std::name Name>
struct named_type
{
	using type = Type;
	static constexpr auto name = Name
};

template<typename... NamedTypes>
struct named_tuple
{
	typename NamedTypes::type $NamedTypes::name...;
};

using my_named_tuple = named_tuple<named_type<std::string, $"first_name">,
                                   named_type<std::string, $"last_name">,
                                   named_type<int, $"birth_year">>;

static_assert(std::is_same_v(my_named_tuple::fist_name, std::string), "");
static_assert(std::is_same_v(my_named_tuple::last_name, std::string), "");
static_assert(std::is_same_v(my_named_tuple::birth_year, int), "");
```

# Usage examples
## Generating comparison operators
The named tuple requires a less operator, of course. Here we go

```C++
template <typename T, std::name... X>
bool named_less(const T& lhs, const T& rhs)
{
  return ((lhs.$X != rhs.$X && lhs.$X < rhs.$X) || ...);
}

template<typename... T, typename... U>
bool operator<(const named_tuple<T...>& lhs,
               const named_tuple<U...>& rhs)
{
  return named_less<T::name...>(lhs, rhs);
}
```

The same `named_less` could be used for structs if reflection would yield the member names at compile time, which is one of the goals for reflection, as given in N3418.

## Array of structs <-> struct of arrays
If reflection would give us the types and names of the data members of a struct, for instance as a `named_tuple`,
then it would be easy to transform an array of structs to a struct of arrays, which one of the goals of reflection (see N3418).

```C++
template <T, typename... NamedTypes>
struct struct_of_arrays;

template <typename T, typename NamedTuple>
struct as_struct_of_arrays_impl;

template <typename T, typename... NamedTypes>
struct as_struct_of_arrays_impl<T, named_tuple<NamedTypes>>
{
    using type = struct_of_arrays<T, NamedTypes>;
};

template <typename T>
using as_struct_of_arrays = typename as_struct_of_arrays_impl<reflect_as_named_tuple<T>>::type;
```

And of course it would be rather straight forward to implement the `struct_of_arrays` with an interface similar to `vector<>`:

```C++
template <typename T, typename... NamedTypes>
struct struct_of_arrays
{
    typename NamedTypes::type $NamedTypes::name...;

    // push_back for example
    void push_back(const T& t)
    {
        ($NamedTypes::name.push_back(t.$NamedTypes::name),...);
    }
};
```

## Get rid of certain types of macros
Who likes preprocessor macros? In quite a few cases, macros could be made obsolete using names.

```C++
#define CHECK_TRAIT_TYPE(name)                           \
  namespace detail                                       \
  {                                                      \
    template <typename T, typename Enable = void>        \
    struct name##_impl                                   \
    {                                                    \
      using type = std::false_type;                      \
    };                                                   \
    template <typename T>                                \
    struct name##_impl<T, std::void_t<typename T::name>> \
    {                                                    \
      using type = std::true_type;                       \
    };                                                   \
  }                                                      \
  template <typename T>                                  \
  using name = typename detail::name##_impl<T>::type;

  SAMPLE_HAS_TRAIT_TYPE(is_expression);
```

This could be replaced by

```C++
namespace detail
{
	template <typename T, std::name Name, typename Enable = void>
	struct has_tag_type
	{
		using type = std::false_type;
	};
	template <typename T, std::name Name>
	struct has_tag_type<T, Name, void_t<typename T::$Name>>
	{
		using type = typename T::$Name;
	};
}

template <typename T>
using is_expression = typename detail::has_tag_type<T, $"is_expression">::type;
```

## Named function arguments
Let's add another operator to `std::name` that captures the name and the operator's (other) argument, e.g.

```C++
template<typename >
struct named_value;

template<typename T>
constexpr auto operator|(std::name name, T&& arg)
{
    return named_value<T, name>(std::forward<T>(arg)); // name has to be constepxr
}
```

We could then write functions that take named values as parameters, e.g.

```C++
struct my_struct
{
    template<typename... T>
    my_struct(T&& t):
        $(T::name){std::forward<typename T::type>(t.value)}
    {
    }

    long id;
    long parent_id = 0;
    long type = 0;
    long timestamp = 0;
    long value = 0;
};

auto x = my_struct{$"id"|17, $"value"|42};
```

## Replace CRTP in some situations
In [sqlpp11](https://github.com/rbock/sqlpp11), variadic CRTP is used in several places to add configurable data members or functions to structs. Here is a simplified example and a way it could be rewritten with names and variadic composition.

### Table definitions
SQL tables have one or more columns, each with its own name and type. sqlpp11 mimics this in its table structs:

```C++
template <typename Table, typename ColumnSpec>
struct column;

template <typename Table, typename... ColumnSpec>
struct table : ColumnSpec::template _member_t<column<Table, ColumnSpec>>... // variadic CRTP
{
    //...
};

// Column specification
struct ColAlpha
{
    static constexpr const char _literal[] = "alpha";

    template <typename T>
    struct _member_t
    {
        T alpha;
    };
};

// Table struct
struct TabSample : table<TabSample, ColAlpha, ColBeta, ColGamma>
{
    static constexpr const char _literal[] = "tab_sample";
};
```

Through variadic inheritance, `TabSample` is imbued with reasonably named data members with respective column types.
The column types know which table they belong to via CRTP. This is important for compile time checks of SQL statements and for serialization (that's what the `_literal` is for).

It should be noted though, that the variadic inheritance is just a tool here to add data members with appropriate names. It does not model anything like "isA".

If we had names and variadic composition, we could of course get rid of the whole variadic CRTP here with ease:

```C++
template <typename Table, typename ColumnSpec>
struct column;

template <typename Table, typename... ColumnSpec>
struct table
{
    column<Table, ColumnSpec> $ColumnSpec::name...; // variadic composition
    //...
};

// Column specification
struct ColAlpha
{
    static constexpr auto name = $"alpha";
};

// Table struct
struct TabSample : table<TabSample, ColAlpha, ColBeta, ColGamma>
{
    static constexpr auto name = $"tab_sample";
};
```

# Other languages
Other languages have features similar to what is proposed here.

## D
In D, you can use strings as template parameters and use them in places where names would be used, resulting in a tuple with optional names, see http://dlang.org/phobos/std_typecons.html#.Tuple

```D
alias Entry = Tuple!(int, "index", string, "value");
Entry e;
e.index = 4;
e.value = "Hello";
assert(e[1] == "Hello");
assert(e[0] == 4);
```

## Python
Python also has a named tuple, see https://docs.python.org/3/library/collections.html#collections.namedtuple

```Python
Point = namedtuple('Point', ['x', 'y'])
p = Point(11, y=22)

```
