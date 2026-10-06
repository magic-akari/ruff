# `typing.ClassVar`

[`typing.ClassVar`] is a type qualifier that is used to indicate that a class variable may not be
written to from instances of that class.

This test makes sure that we discover the type qualifier while inferring types from an annotation.
For more details on the semantics of pure class variables, see [this test](../attributes.md).

## Basic

```py
import typing
from typing import ClassVar, Annotated

class C:
    a: ClassVar[int] = 1
    b: Annotated[ClassVar[int], "the annotation for b"] = 1
    c: ClassVar[Annotated[int, "the annotation for c"]] = 1
    d: ClassVar = 1
    e: "ClassVar[int]" = 1
    f: typing.ClassVar = 1

reveal_type(C.a)  # revealed: int
reveal_type(C.b)  # revealed: int
reveal_type(C.c)  # revealed: int
reveal_type(C.d)  # revealed: Unknown | Literal[1]
reveal_type(C.e)  # revealed: int
reveal_type(C.f)  # revealed: Unknown | Literal[1]

c = C()

# error: [invalid-attribute-access]
c.a = 2
# error: [invalid-attribute-access]
c.b = 2
# error: [invalid-attribute-access]
c.c = 2
# error: [invalid-attribute-access]
c.d = 2
# error: [invalid-attribute-access]
c.e = 2
# error: [invalid-attribute-access]
c.f = 3
```

## From stubs

This is a regression test for a bug where we did not properly keep track of type qualifiers when
accessed from stub files.

`module.pyi`:

```pyi
from typing import ClassVar

class C:
    a: ClassVar[int]
```

`main.py`:

```py
from module import C

c = C()
c.a = 2  # error: [invalid-attribute-access]
```

## Conflicting type qualifiers

We currently ignore conflicting qualifiers and simply union them, which is more conservative than
intersecting them. This means that we consider `a` to be a `ClassVar` here:

```py
from typing import ClassVar

def flag() -> bool:
    return True

class C:
    if flag():
        a: ClassVar[int] = 1
    else:
        a: str

reveal_type(C.a)  # revealed: int | str

c = C()

# error: [invalid-attribute-access]
c.a = 2
```

## Too many arguments

```py
from typing import ClassVar

class C:
    # error: [invalid-type-form] "Type qualifier `typing.ClassVar` expected exactly 1 argument, got 2"
    x: ClassVar[int, str] = 1
```

## Trailing comma creates a tuple

A trailing comma in a subscript creates a single-element tuple. We need to handle this gracefully
and emit a proper error rather than crashing (see
[ty#1793](https://github.com/astral-sh/ty/issues/1793)).

```py
from typing import ClassVar

class C:
    # error: [invalid-type-form] "Tuple literals are not allowed in this context in a type expression: Did you mean `tuple[()]`?"
    x: ClassVar[(),]

# error: [invalid-attribute-access] "Cannot assign to ClassVar `x` from an instance of type `C`"
C().x = 42
reveal_type(C.x)  # revealed: Unknown
```

This also applies when the trailing comma is inside the brackets (see
[ty#1768](https://github.com/astral-sh/ty/issues/1768)):

```py
from typing import ClassVar

class D:
    # A trailing comma here doesn't change the meaning; it's still one argument.
    a: ClassVar[int,] = 1

reveal_type(D.a)  # revealed: int
```

## `ClassVar` cannot contain non-self type variables

`ClassVar` cannot include type variables at any level of nesting.

```toml
[environment]
python-version = "3.12"
```

```py
from typing import ClassVar, TypeVar, ParamSpec, Generic

T = TypeVar("T")
P = ParamSpec("P")

class C(Generic[T, P]):
    # error: [invalid-type-form] "`ClassVar` cannot contain type variables"
    a: ClassVar[T]

    # error: [invalid-type-form] "`ClassVar` cannot contain type variables"
    b: ClassVar[list[T]]

    # error: [invalid-type-form] "`ClassVar` cannot contain type variables"
    c: ClassVar[int | T]

    # error: [invalid-type-form] "Bare ParamSpec `P` is not valid in this context"
    d: ClassVar[P]

    # No error: no type variables
    e: ClassVar[int] = 1

# PEP 695 syntax
class D[T]:
    # error: [invalid-type-form] "`ClassVar` cannot contain type variables"
    x: ClassVar[T]

    # error: [invalid-type-form] "`ClassVar` cannot contain type variables"
    y: ClassVar[dict[str, T]]
```

## Type variables inside aliases

An alias does not hide a type variable from `ClassVar` validation. Arguments that the alias discards
are allowed, including arguments removed when the alias's value simplifies to a concrete type.

```toml
[environment]
python-version = "3.12"
```

```py
from typing import ClassVar, Generic, TypeVar

type DirectAlias[T] = T
type Alias[T] = list[T]
type Ignored[T] = int
type Either[T, U] = T | U

class Holder[T]:
    # error: [invalid-type-form] "`ClassVar` cannot contain type variables"
    direct: ClassVar[DirectAlias[T]]
    # error: [invalid-type-form] "`ClassVar` cannot contain type variables"
    exposed: ClassVar[Alias[T]]
    ignored: ClassVar[Ignored[T]]
    simplified: ClassVar[Either[object, T]]

T = TypeVar("T")

class LegacyHolder(Generic[T]):
    # error: [invalid-type-form] "`ClassVar` cannot contain type variables"
    exposed: ClassVar[Alias[T]]
    ignored: ClassVar[Ignored[T]]
    simplified: ClassVar[Either[object, T]]
```

Recursive aliases can expose an argument only after passing it to a different parameter. Parameters
that remain unused throughout the recursion are allowed.

```py
type Shift[A, B] = A | list[Shift[B, int]]
type RecursiveIgnored[A, B] = B | list[RecursiveIgnored[A, int]]

class RecursiveHolder[T]:
    # error: [invalid-type-form] "`ClassVar` cannot contain type variables"
    exposed: ClassVar[Shift[int, T]]
    ignored: ClassVar[RecursiveIgnored[T, int]]
    simplified: ClassVar[Shift[object, T]]
```

The search also preserves simplification inside growing recursive aliases. It does not yet detect an
occurrence that becomes visible only after a growing recursive reference.

```py
type Absorbed[A, B] = tuple[A | B, Absorbed[A, list[B]]]
type GrowingShift[A, B] = tuple[A, GrowingShift[B, list[A]]]
type Recursive[T] = list[Recursive[list[T]]]

class GrowingHolder[T]:
    simplified: ClassVar[Absorbed[object, T]]
    recursive: ClassVar[Recursive[T]]
    # TODO: Reject T when it becomes the first argument of GrowingShift.
    exposed: ClassVar[GrowingShift[int, T]]
```

## `ClassVar` can contain `Self`

`Self` is allowed inside `ClassVar`.

```toml
[environment]
python-version = "3.11"
```

```py
from typing import ClassVar, Self

class Base:
    all_instances: ClassVar[list[Self]]

    def method(self):
        reveal_type(self.all_instances)  # revealed: list[Self@method]

    @classmethod
    def cls_method(cls):
        reveal_type(cls.all_instances)  # revealed: list[Self@cls_method]

reveal_type(Base.all_instances)  # revealed: list[Base]

class Sub(Base): ...

reveal_type(Sub.all_instances)  # revealed: list[Sub]
```

The type parameters in `Self`'s bound do not make `Self` invalid in a generic class.

```py
from typing import Generic, TypeVar

U = TypeVar("U")

class GenericBase(Generic[U]):
    direct: ClassVar[Self]
    nested: ClassVar[list[Self]]
```

Assignments through class objects should bind `Self` when writing a `ClassVar`, matching read-side
behavior. This remains permissive for `type[Base]` values even though `ClassVar[Self]` in non-final
classes is unsound.

```py
from typing import ClassVar, Self, TypeVar

class Saved:
    latest: ClassVar[Self]

    def save(self) -> None:
        type(self).latest = self

Saved.latest = Saved()

reveal_type(Saved.latest)  # revealed: Saved

class SavedSub(Saved): ...

reveal_type(SavedSub.latest)  # revealed: SavedSub

SavedSub.latest = SavedSub()

SavedSub.latest = Saved()  # error: [invalid-assignment]

def store_saved(cls: type[Saved]) -> None:
    cls.latest = Saved()

T = TypeVar("T", bound=Saved)

def store_generic(cls: type[T], value: T) -> None:
    cls.latest = value
```

Assignments through gradual class objects remain permissive.

```py
from typing import Any, ClassVar, reveal_type

class DynamicSaved:
    count: ClassVar[int]

def store_any(cls: type[Any], value: Any) -> None:
    cls.count = value
    reveal_type(cls.count)  # revealed: Any
```

## `Self` in PEP 695 generic classes

```toml
[environment]
python-version = "3.12"
```

```py
from typing import ClassVar, Self

class GenericBase[U]:
    direct: ClassVar[Self]
    nested: ClassVar[list[Self]]
```

## Generic callable signatures

Type variables bound by a callable's own signature are allowed in `ClassVar`. `CallableTypeOf`
preserves a function's generic signature without specializing it.

```toml
[environment]
python-version = "3.12"
```

```py
from typing import ClassVar, Protocol, TypeVar
from ty_extensions._internal import CallableTypeOf, TypeOf

def identity[T](value: T) -> T:
    return value

U = TypeVar("U")

def legacy_identity(value: U) -> U:
    return value

class Callback(Protocol):
    def __call__[T](self, value: T) -> T: ...

class Holder:
    callback: ClassVar[CallableTypeOf[identity]]
    legacy_callback: ClassVar[CallableTypeOf[legacy_identity]]
    protocol: ClassVar[Callback]
    function: ClassVar[TypeOf[identity]]
```

A protocol's generic method also binds its own parameter when the protocol is local to a function.

```py
def outer():
    class Callback(Protocol):
        def __call__[T](self, value: T) -> T: ...

    class Holder:
        callback: ClassVar[Callback]
```

A callable can still capture a type variable from an enclosing scope. Its own type parameters do not
bind that captured variable.

```py
def outer[T]():
    def callback[U](value: T, other: U) -> U:
        return other

    class Holder:
        # error: [invalid-type-form] "`ClassVar` cannot contain type variables"
        value: ClassVar[CallableTypeOf[callback]]
        # error: [invalid-type-form] "`ClassVar` cannot contain type variables"
        function: ClassVar[TypeOf[callback]]
```

## Generic property accessors

A property's getter and setter bind their own type parameters, just like other methods. Those
parameters do not make the protocol invalid in `ClassVar`.

```toml
[environment]
python-version = "3.12"
```

```py
from typing import ClassVar, Protocol, TypeVar

class GenericProperties(Protocol):
    @property
    def getter[T](self) -> tuple[T, T]: ...
    @property
    def setter(self) -> object: ...
    @setter.setter
    def setter[T](self, value: tuple[T, T]) -> None: ...

class Holder:
    value: ClassVar[GenericProperties]  # no diagnostic
```

The same applies to accessors that use legacy type variables.

```py
T = TypeVar("T")

class LegacyProperties(Protocol):
    @property
    def getter(self) -> tuple[T, T]: ...
    @property
    def setter(self) -> object: ...
    @setter.setter
    def setter(self, value: tuple[T, T]) -> None: ...

class LegacyHolder:
    value: ClassVar[LegacyProperties]  # no diagnostic
```

## Captured type variables in property accessors

An accessor's own type parameters do not bind variables captured from an enclosing function.

```toml
[environment]
python-version = "3.12"
```

```py
from typing import ClassVar, Protocol, TypeVar

def outer[T]():
    class Getter(Protocol):
        @property
        def value[U](self) -> tuple[U, T]: ...

    class Setter(Protocol):
        @property
        def value(self) -> object: ...
        @value.setter
        def value[U](self, value: tuple[U, T]) -> None: ...

    class Holder:
        # error: [invalid-type-form] "`ClassVar` cannot contain type variables"
        getter: ClassVar[Getter]
        # error: [invalid-type-form] "`ClassVar` cannot contain type variables"
        setter: ClassVar[Setter]
```

Legacy accessors also retain the distinction between their own type variables and captured
variables.

```py
T = TypeVar("T")
U = TypeVar("U")

def legacy_outer(value: T):
    class Getter(Protocol):
        @property
        def value(self) -> tuple[U, T]: ...

    class Setter(Protocol):
        @property
        def value(self) -> object: ...
        @value.setter
        def value(self, value: tuple[U, T]) -> None: ...

    class Holder:
        # error: [invalid-type-form] "`ClassVar` cannot contain type variables"
        getter: ClassVar[Getter]
        # error: [invalid-type-form] "`ClassVar` cannot contain type variables"
        setter: ClassVar[Setter]
```

Only the getter's return type and the setter's value type are exposed by a property. Captures in
optional extra parameters do not affect its use in `ClassVar`.

```py
def extra_parameters[T]():
    class Property(Protocol):
        @property
        def value(self, fallback: T | None = None) -> int: ...
        @value.setter
        def value(self, value: int, fallback: T | None = None) -> None: ...

    class Holder:
        value: ClassVar[Property]  # no diagnostic
```

## Captured type variables in structural types

Local protocols and `TypedDict`s can capture an enclosing type variable. `ClassVar` rejects these
captures, including those exposed by a protocol method.

```toml
[environment]
python-version = "3.12"
```

```py
from __future__ import annotations

from typing import ClassVar, Protocol, TypedDict

def outer[T]():
    class Captured(Protocol):
        value: T

    class Callback(Protocol):
        def method[U](self, value: U) -> tuple[T, U]: ...

    class Payload(TypedDict):
        value: T

    class Holder:
        # error: [invalid-type-form] "`ClassVar` cannot contain type variables"
        attribute: ClassVar[Captured]
        # error: [invalid-type-form] "`ClassVar` cannot contain type variables"
        method: ClassVar[Callback]
        # error: [invalid-type-form] "`ClassVar` cannot contain type variables"
        payload: ClassVar[Payload]
```

## Recursive structural types

Concrete recursive structural types remain valid. A protocol's implicit receiver and a method's own
type parameters do not introduce a free type variable.

```toml
[environment]
python-version = "3.12"
```

```py
from __future__ import annotations

from typing import ClassVar, Protocol, TypedDict

class Recursive[T](Protocol):
    def method[U](self, value: U) -> tuple[Recursive[int], U]: ...

class RecursivePayload(TypedDict):
    next: RecursivePayload

class Holder:
    protocol: ClassVar[Recursive[int]]
    payload: ClassVar[RecursivePayload]
```

This also holds when the recursive references keep changing specialization.

```py
def outer():
    class Recursive[T](Protocol):
        next: Recursive[list[T]]

    class Payload[T](TypedDict):
        next: Payload[list[T]]

    class Holder:
        protocol: ClassVar[Recursive[int]]
        payload: ClassVar[Payload[int]]
```

## Assignments through generic aliases

Assignments through generic aliases still resolve class variables.

```py
from typing import ClassVar, Generic, TypeVar
from typing_extensions import reveal_type

T = TypeVar("T")

class Box(Generic[T]):
    count: ClassVar[int]

Box[int].count = 1
reveal_type(Box[int].count)  # revealed: int
```

## Combining `ClassVar` and `Final` in normal classes

An attribute on a class body that is annotated as `Final` is implicitly treated as a class variable.
The error message is different, but these attributes cannot be written to from instances of the
class:

```py
from typing import Final

class C:
    a: Final[int] = 1

reveal_type(C.a)  # revealed: int

c = C()
c.a = 2  # error: [invalid-assignment] "Cannot assign to final attribute `a` on type `C`"
```

In this sense, it is redundant to combine `ClassVar` and `Final`. We issue a warning in these cases:

```py
from typing import Annotated, ClassVar

class D:
    # error: [redundant-final-classvar] "Combining `ClassVar` and `Final` is redundant"
    a: ClassVar[Final[int]] = 1

    # error: [redundant-final-classvar] "Combining `ClassVar` and `Final` is redundant"
    b: Final[ClassVar[int]] = 1

    # error: [redundant-final-classvar] "Combining `ClassVar` and `Final` is redundant"
    c: Final[ClassVar] = 1

    # error: [redundant-final-classvar] "Combining `ClassVar` and `Final` is redundant"
    d: Annotated[Final[ClassVar[int]], "metadata"] = 1

    # error: [redundant-final-classvar] "Combining `ClassVar` and `Final` is redundant"
    e: ClassVar[Final] = 1

    # error: [redundant-final-classvar] "Combining `ClassVar` and `Final` is redundant"
    f: Annotated[Final[Annotated[Annotated[ClassVar[int], "a"], "b"]], "c"] = 1

reveal_type(D.a)  # revealed: int
reveal_type(D.b)  # revealed: int
reveal_type(D.c)  # revealed: Literal[1]
reveal_type(D.d)  # revealed: int
reveal_type(D.e)  # revealed: Literal[1]
reveal_type(D.f)  # revealed: int

d = D()
d.a = 2  # error: [invalid-attribute-access] "Cannot assign to ClassVar `a` from an instance of type `D`"
d.b = 2  # error: [invalid-attribute-access] "Cannot assign to ClassVar `b` from an instance of type `D`"
d.c = 2  # error: [invalid-attribute-access] "Cannot assign to ClassVar `c` from an instance of type `D`"
d.d = 2  # error: [invalid-attribute-access] "Cannot assign to ClassVar `d` from an instance of type `D`"
d.e = 2  # error: [invalid-attribute-access] "Cannot assign to ClassVar `e` from an instance of type `D`"
d.f = 2  # error: [invalid-attribute-access] "Cannot assign to ClassVar `f` from an instance of type `D`"
```

## Combining `ClassVar` and `Final` in dataclasses

In dataclasses, `ClassVar[Final[int]]` has a distinct meaning from `Final[int]`. The former is a
final class variable, the latter is a final instance attribute. The warning is therefore not emitted
when combining `ClassVar[Final[...]]` in dataclasses:

```py
from dataclasses import dataclass
from typing import ClassVar, Final

@dataclass
class D:
    # No warning:
    class_attr: ClassVar[Final[int]] = 1

    instance_attr: Final[int] = 1
```

Note that `class_attr` does not appear in the signature of `__init__`:

```py
# revealed: (self: D, instance_attr: int = 1) -> None
reveal_type(D.__init__)
```

```py
def _(d: D):
    reveal_type(d.class_attr)  # revealed: int
    reveal_type(d.instance_attr)  # revealed: int

    d.class_attr = 2  # error: [invalid-attribute-access]
```

The reverse direction `Final[ClassVar[...]]` is not recognized by the runtime implementation of
dataclasses. We could consider emitting a warning in these cases, but for now, we treat is just like
`ClassVar[Final[...]]` and allow it in dataclasses:

```py
from dataclasses import dataclass

@dataclass
class E:
    class_attr: Final[ClassVar[int]] = 1

# revealed: (self: E) -> None
reveal_type(E.__init__)

def _(e: E):
    reveal_type(e.class_attr)  # revealed: int

    e.class_attr = 2  # error: [invalid-attribute-access]
```

## Illegal `ClassVar` in type expression

```py
from typing import ClassVar

class C:
    # error: [invalid-type-form] "Type qualifier `typing.ClassVar` is not allowed in type expressions (only in annotation expressions)"
    x: ClassVar | int

    # error: [invalid-type-form] "Type qualifier `typing.ClassVar` is not allowed in type expressions (only in annotation expressions)"
    y: int | ClassVar[str]
```

## Illegal positions

```toml
[environment]
python-version = "3.12"
```

```py
from typing import ClassVar, TypedDict
from ty_extensions._internal import reveal_mro

# error: [invalid-type-form] "`ClassVar` is only allowed in class bodies"
x: ClassVar[int] = 1

class C:
    def __init__(self) -> None:
        # error: [invalid-type-form] "`ClassVar` annotations are not allowed for non-name targets"
        self.x: ClassVar[int] = 1

        # error: [invalid-type-form] "`ClassVar` is only allowed in class bodies"
        y: ClassVar[int] = 1

# error: [invalid-type-form] "Type qualifier `typing.ClassVar` is not allowed in parameter annotations"
def f(x: ClassVar[int]) -> None:
    pass

# error: [invalid-type-form] "Type qualifier `typing.ClassVar` is not allowed in parameter annotations"
def f[T](x: ClassVar[T]) -> T:
    return x

# error: [invalid-type-form] "Type qualifier `typing.ClassVar` is not allowed in return type annotations"
def f() -> ClassVar[int]:
    return 1

# error: [invalid-type-form] "Type qualifier `typing.ClassVar` is not allowed in return type annotations"
def f[T](x: T) -> ClassVar[T]:
    return x

# TODO: this should be an error
class Foo(ClassVar[tuple[int]]): ...

# TODO: Show `Unknown` instead of `@Todo` type in the MRO; or ignore `ClassVar` and show the MRO as if `ClassVar` was not there
# revealed: (<class 'Foo'>, @Todo(Inference of subscript on special form), <class 'object'>)
reveal_mro(Foo)

class Foo(TypedDict):
    # error: [invalid-type-form] "`ClassVar` is not allowed in TypedDict fields"
    x: ClassVar[int]
    # error: [invalid-type-form] "`ClassVar` is not allowed in TypedDict fields"
    y: ClassVar
```

[`typing.classvar`]: https://docs.python.org/3/library/typing.html#typing.ClassVar
