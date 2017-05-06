==============================
What you can structure and how
==============================

The philosophy of ``cattrs`` structuring is simple: give it an instance of Python
built-in types and collections, and a type describing the data you want out.
``cattrs`` will convert the input data into the type you want, or throw an
exception.

All loading conversions are composable, where applicable. This is
demonstrated further in the examples.

.. warning::

    When using Python 3.5 earlier or equal to 3.5.3 or Python 3.6.0, please use
    the bundled ``cattr.vendor.typing`` module instead of Python's standard ``typing``
    module. These versions of ``typing`` are incompatible with ``cattrs``. If
    your Python version is a later one, please use Python's ``typing`` instead.

Primitive values
----------------

``typing.Any``
~~~~~~~~~~~~~~

Use ``typing.Any`` to avoid applying any conversions to the object you're
loading; it will simply be passed through.

.. doctest::

    >>> cattr.structure(1, Any)
    1
    >>> d = {1: 1}
    >>> cattr.structure(d, Any) is d
    True

``int``, ``float``, ``str``, ``bytes``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Use any of these primitive types to convert the object to the type.

.. doctest::

    >>> cattr.structure(1, str)
    '1'
    >>> cattr.structure("1", float)
    1.0

In case the conversion isn't possible, the expected exceptions will be
propagated out. The particular exceptions are the same as if you'd tried to
do the conversion yourself, directly.

.. code-block:: python

    >>> cattr.structure("not-an-int", int)
    Traceback (most recent call last):
    ...
    ValueError: invalid literal for int() with base 10: 'not-an-int'

Enums
~~~~~

Enums will be structured by their values. This works even for complex values,
like tuples.

.. doctest::

    >>> @unique
    ... class CatBreed(Enum):
    ...    SIAMESE = "siamese"
    ...    MAINE_COON = "maine_coon"
    ...    SACRED_BIRMAN = "birman"
    ...
    >>> cattr.structure("siamese", CatBreed)
    <CatBreed.SIAMESE: 'siamese'>

Again, in case of errors, the expected exceptions will fly out.

.. code-block:: python

    >>> cattr.structure("alsatian", CatBreed)
    Traceback (most recent call last):
    ...
    ValueError: 'alsatian' is not a valid CatBreed

Collections and other generics
------------------------------

Optionals
~~~~~~~~~

``Optional`` primitives and collections are supported out of the box.

.. doctest::

    >>> cattr.structure(None, int)
    Traceback (most recent call last):
    ...
    TypeError: int() argument must be a string, a bytes-like object or a number, not 'NoneType'
    >>> cattr.structure(None, Optional[int])
    >>> # None was returned.

Bare ``Optional`` s (non-parameterized, just ``Optional``, as opposed to
``Optional[str]``) aren't supported, use ``Optional[Any]`` instead.

This generic type is composable with all other converters.

.. doctest::

    >>> cattr.structure(1, Optional[float])
    1.0

Lists
~~~~~

Lists can be produced from any iterable object. Types converting to lists are:

* ``Sequence[T]``
* ``MutableSequence[T]``
* ``List[T]``

In all cases, a new list will be returned, so this operation can be used to
copy an iterable into a list. A bare type, for example ``Sequence`` instead of
``Sequence[int]``, is equivalent to ``Sequence[Any]``.

.. doctest::

    >>> cattr.structure((1, 2, 3), MutableSequence[int])
    [1, 2, 3]

These generic types are composable with all other converters.

.. doctest::

    >>> cattr.structure((1, None, 3), List[Optional[str]])
    ['1', None, '3']

Sets and frozensets
~~~~~~~~~~~~~~~~~~~

Sets and frozensets can be produced from any iterable object. Types converting
to sets are:

* ``Set[T]``
* ``MutableSet[T]``

Types converting to frozensets are:

* ``FrozenSet[T]``

In all cases, a new set or frozenset will be returned, so this operation can be
used to copy an iterable into a set. A bare type, for example ``MutableSet``
instead of ``MutableSet[int]``, is equivalent to ``MutableSet[Any]``.

.. doctest::

    >>> cattr.structure([1, 2, 3, 4], Set)
    {1, 2, 3, 4}

These generic types are composable with all other converters.

.. doctest::

    >>> cattr.structure([[1, 2], [3, 4]], Set[FrozenSet[str]])
    {frozenset({'4', '3'}), frozenset({'1', '2'})}

Dictionaries
~~~~~~~~~~~~

Dicts can be produced from other mapping objects. To be more precise, the
object being converted must expose an ``items()`` method producing an iterable
key-value tuples, and be able to be passed to the ``dict`` constructor as an
argument. Types converting to dictionaries are:

* ``Dict[K, V]``
* ``MutableMapping[K, V]``
* ``Mapping[K, V]``

In all cases, a new dict will be returned, so this operation can be
used to copy a mapping into a dict. Any type parameters set to ``typing.Any``
will be passed through unconverted. If both type parameters are absent,
they will be treated as ``Any`` too.

.. doctest::

    >>> from collections import OrderedDict
    >>> cattr.structure(OrderedDict([(1, 2), (3, 4)]), Dict)
    {1: 2, 3: 4}

These generic types are composable with all other converters. Note both keys
and values can be converted.

.. doctest::

    >>> cattr.structure({1: None, 2: 2.0}, Dict[str, Optional[int]])
    {'1': None, '2': 2}

Homogeneous and heterogeneous tuples
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Homogeneous and heterogeneous tuples can be produced from iterable objects.
Heterogeneous tuples require an iterable with the number of elements matching
the number of type parameters exactly. Use:

* ``Tuple[A, B, C, D]``

Homogeneous tuples use:

* ``Tuple[T, ...]``

In all cases a tuple will be returned. Any type parameters set to
``typing.Any`` will be passed through unconverted.

.. doctest::

    >>> cattr.structure([1, 2, 3], Tuple[int, str, float])
    (1, '2', 3.0)

The tuple conversion is composable with all other converters.

.. doctest::

    >>> cattr.structure([{1: 1}, {2: 2}], Tuple[Dict[str, float], ...])
    ({'1': 1.0}, {'2': 2.0})

Unions
~~~~~~

Unions of ``NoneType`` and a single other type are supported (also known as
``Optional`` s). All other unions a require a disambiguation function.

In the case of a union consisting exclusively of ``attrs`` classes, ``cattrs``
will attempt to generate a disambiguation function automatically; this will
succeed only if each class has a unique, required field. Given the following
classes:

.. code-block:: python

    >>> @attr.s
    ... class A:
    ...     a = attr.ib()
    ...     x = attr.ib()
    ...
    >>> @attr.s
    ... class B:
    ...     a = attr.ib()
    ...     y = attr.ib()
    ...
    >>> @attr.s
    ... class C:
    ...     a = attr.ib()
    ...     z = attr.ib()
    ...

``cattrs`` can deduce only instances of ``A`` will contain `x`, only instances
of ``B`` will contain ``y``, etc. A disambiguation function using this
information will then be generated and cached. This will happen automatically,
the first time an appropriate union is structured.


``attrs`` classes
-------------------------

Simple ``attrs`` classes
~~~~~~~~~~~~~~~~~~~~~~~~

``attrs`` classes using primitives, collections of primitives and their own
converters work out of the box. Given a mapping ``d`` and class ``A``,
``cattrs`` will simply instantiate ``A`` with ``d`` unpacked.

.. doctest::

    >>> @attr.s
    ... class A:
    ...     a = attr.ib()
    ...     b = attr.ib(convert=int)
    ...
    >>> cattr.structure({'a': 1, 'b': '2'}, A)
    A(a=1, b=2)

``attrs`` classes deconstructed into tuples can be structured using
``cattr.structure_attrs_fromtuple`` (``fromtuple`` as in the opposite of
``attr.astuple`` and ``converter.unstructure_attrs_astuple``).

.. doctest::

    >>> @attr.s
    ... class A:
    ...     a = attr.ib()
    ...     b = attr.ib(convert=int)
    ...
    >>> cattr.structure_attrs_fromtuple(['string', '2'], A)
    A(a='string', b=2)

Loading from tuples can be made the default by assigning to the ``structure_attr``
property of ``Converter`` objects.

.. doctest::

    >>> converter = cattr.Converter()
    >>> converter.structure_attrs = converter.structure_attrs_fromtuple
    >>> @attr.s
    ... class A:
    ...     a = attr.ib()
    ...     b = attr.ib(convert=int)
    ...
    >>> converter.structure(['string', '2'], A)
    A(a='string', b=2)

Structuring from tuples can also be made the default for specific classes only;
see registering custom structure hooks below.

Complex ``attrs`` classes
~~~~~~~~~~~~~~~~~~~~~~~~~

Complex ``attrs`` classes are classes with type information available for some
or all attributes. These classes support almost arbitrary nesting.

Registering custom loading hooks
--------------------------------