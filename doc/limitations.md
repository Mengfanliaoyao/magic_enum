# Limitations

* This library uses a compiler-specific hack based on `__PRETTY_FUNCTION__` / `__FUNCSIG__`.

* To check if magic_enum is supported with your compiler, use macro `MAGIC_ENUM_SUPPORTED` or constexpr constant `magic_enum::is_magic_enum_supported`.
  If magic_enum is used on an unsupported compiler, a compilation error will occur.
  To suppress the error, define macro `MAGIC_ENUM_NO_CHECK_SUPPORT`.

* magic_enum can't reflect if the enum is a forward declaration.

## Enum Flags

* For enum flags, add `is_flags` to specialization `enum_range` for necessary enum type. Specializations of `enum_range` must be injected in `namespace magic_enum::customize`.
  ```cpp
  enum class Directions { Up = 1 << 0, Down = 1 << 1, Right = 1 << 2, Left = 1 << 3 };
  template <>
  struct magic_enum::customize::enum_range<Directions> {
    static constexpr bool is_flags = true;
  };
  ```

* `MAGIC_ENUM_RANGE_MAX` / `MAGIC_ENUM_RANGE_MIN` does not affect the maximum number of enum flags.

* If an enum is declared as a flag enum, its zero value will not be reflected.

* Or, for enum types that are deeply nested in classes and/or namespaces, declare a function called `my_adl_info_struct adl_magic_enum_define_range(my_enum_type)` in the same namespace as `my_enum_type`, which magic_enum will find by ADL (because the function is in the same class/namespace as `my_enum_type`), and whose return type is a struct with `static constexpr` data members containing the same parameters as `magic_enum::customize::enum_range<my_enum_type>`
  ```cpp
  namespace Deeply::Nested::Namespace {
  enum class my_enum_type { ... };
  struct my_adl_info_struct {
    static constexpr bool is_flags = true;
    // you can also set min and max here (see Enum Range below)
    // static constexpr int min = ...;
    // static constexpr int max = ...;
  };
  // - magic_enum will find this function by ADL
  // - no need to ever define this function
  my_adl_info_struct adl_magic_enum_define_range(my_enum_type); 
  }
  ```

* As a shorthand, if you only want to set `is_flags` and not `min` or `max`, you can also use `magic_enum::customize::adl_info<is_flags_bool>` to avoid having to define `my_adl_info_struct` in your code:
  ```cpp
  namespace Deeply::Nested::Namespace {
  enum class my_enum_type { ... };
  // - magic_enum will find this function by ADL
  // - no need to ever define this function
  magic_enum::customize::adl_info<true> adl_magic_enum_define_range(my_enum_type); 
  }
  ```

## Enum Range

* Enum values must be in the range `[MAGIC_ENUM_RANGE_MIN, MAGIC_ENUM_RANGE_MAX]`.

* By default, `MAGIC_ENUM_RANGE_MIN = -128`, `MAGIC_ENUM_RANGE_MAX = 127`.

* If you need a different range for all enum types by default, redefine the macro `MAGIC_ENUM_RANGE_MIN` and `MAGIC_ENUM_RANGE_MAX`:

    ```cpp
    #define MAGIC_ENUM_RANGE_MIN 0
    #define MAGIC_ENUM_RANGE_MAX 256
    #include <magic_enum/magic_enum.hpp>
    ```

* If you need a different range for a specific enum type, add the specialization `enum_range` for the enum type. Specializations of `enum_range` must be injected in `namespace magic_enum::customize`.

  ```cpp
  #include <magic_enum/magic_enum.hpp>

  enum class number { one = 100, two = 200, three = 300 };

  template <>
  struct magic_enum::customize::enum_range<number> {
    static constexpr int min = 100;
    static constexpr int max = 300;
    // (max - min) must be less than UINT16_MAX.
  };
  ```

* Or, for enum types that are deeply nested in classes and/or namespaces, declare a function called `my_adl_info_struct adl_magic_enum_define_range(my_enum_type)` in the same namespace as `my_enum_type`, which magic_enum will find by ADL (because the function is in the same class/namespace as `my_enum_type`), and whose return type is a struct with `static constexpr` data members containing the same parameters as `magic_enum::customize::enum_range<my_enum_type>`
  ```cpp
  namespace Deeply::Nested::Namespace {
  enum class my_enum_type { ... };
  struct my_adl_info_struct {
    static constexpr int min = 100;
    static constexpr int max = 300;
    // you can also set is_flags here
    // static constexpr bool is_flags = true;
  };
  // - magic_enum will find this function by ADL
  // - no need to ever define this function
  my_adl_info_struct adl_magic_enum_define_range(my_enum_type); 
  }
  ```

* As a shorthand, if you only want to set `min` and `max` and not `is_flags`, you can also use `magic_enum::customize::adl_info<min_int, max_int>` to avoid having to define `my_adl_info_struct` in your code:
  ```cpp
  namespace Deeply::Nested::Namespace {
  enum class my_enum_type { ... };
  // - magic_enum will find this function by ADL
  // - no need to ever define this function
  magic_enum::customize::adl_info<100 /*min*/, 300 /*max*/> adl_magic_enum_define_range(my_enum_type); 
  }
  ```

## Aliasing

magic_enum [won't work if a value is aliased](https://github.com/Neargye/magic_enum/issues/68). How magic_enum works with aliases is compiler-implementation-defined.

```cpp
enum ShapeKind {
  ConvexBegin = 0,
  Box = 0, // Won't work.
  Sphere = 1,
  ConvexEnd = 2,
  Donut = 2, // Won't work too.
  Banana = 3,
  COUNT = 4
};
// magic_enum::enum_cast<ShapeKind>("Box") -> nullopt
// magic_enum::enum_name(ShapeKind::Box) -> "ConvexBegin"
```

One possible workaround for the issue is to define the enum values you want reflected before their aliases:

```cpp
enum ShapeKind {
  // Convex shapes, see ConvexBegin and ConvexEnd below.
  Box = 0,
  Sphere = 1,

  // Non-convex shapes.
  Donut = 2,
  Banana = 3,

  COUNT = Banana + 1,

  // Non-reflected aliases.
  ConvexBegin = Box,
  ConvexEnd = Sphere + 1
};
// magic_enum::enum_cast<ShapeKind>("Box") -> ShapeKind::Box
// magic_enum::enum_name(ShapeKind::Box) -> "Box"

// Non-reflected aliases.
// magic_enum::enum_cast<ShapeKind>("ConvexBegin") -> nullopt
// magic_enum::enum_name(ShapeKind::ConvexBegin) -> "Box"
```

On some compilers, enum aliases are not supported, [for example Visual Studio 2017](https://github.com/Neargye/magic_enum/issues/36), macro `MAGIC_ENUM_SUPPORTED_ALIASES` will be undefined.

```cpp
enum Number {
  one = 1,
  ONE = 1
};
// magic_enum::enum_cast<Number>("one") -> nullopt
// magic_enum::enum_name(Number::one) -> ""
// magic_enum::enum_cast<Number>("ONE") -> nullopt
// magic_enum::enum_name(Number::ONE) -> ""
```

## Other Compiler Issues

* If you hit a message like this:

  ```text
  [...]
  note: constexpr evaluation hit maximum step limit; possible infinite loop?
  ```

  Change the limit for the number of constexpr evaluated:
  * MSVC: `/constexpr:depthN`, `/constexpr:stepsN` <https://docs.microsoft.com/en-us/cpp/build/reference/constexpr-control-constexpr-evaluation>
  * Clang: `-fconstexpr-depth=N`, `-fconstexpr-steps=N` <https://clang.llvm.org/docs/UsersManual.html#controlling-implementation-limits>
  * GCC: `-fconstexpr-depth=N`, `-fconstexpr-loop-limit=N`, `-fconstexpr-ops-limit=N` <https://gcc.gnu.org/onlinedocs/gcc-9.2.0/gcc/C_002b_002b-Dialect-Options.html>

* Visual Studio's Intellisense may have some problems analyzing magic_enum.

* Enums in templates may not work correctly (especially on Сlang).
  See [#164](https://github.com/Neargye/magic_enum/issues/164), [#65](https://github.com/Neargye/magic_enum/issues/65)
