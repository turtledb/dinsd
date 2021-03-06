Relational Python
=================

Copyright 2012, 2013 by R. David Murray, Licensed under the Apache License,
Version 2.0 (http://www.apache.org/licenses/LICENSE-2.0).


Introduction
------------

About This Document
~~~~~~~~~~~~~~~~~~~

Dinsd was initially developed while I went through the book

    An Introduction to Relational Database Theory
    by Hugh Darwen

This book uses his and C. J. Date's notional ``D`` implementation, *Tutorial
D*, and gives various examples and problems based on *Tutorial D* to elucidate
the concepts of Relational Algebra.  ``D`` and *Tutorial D* are in turn
defined fully in Hugh and C. J.'s seminal work

    The Third Manifesto

This file writes out various examples in dinsd that correspond to examples
and/or problems from the book, as well as other examples that were inspired by
the book or explored during the writing of dinsd.  The examples here do not
necessarily follow the book (although that is the general trend), as writing
the implementation required hoping around a bit in the conceptual space.

In addition, while early drafts of this file represented the linear
development of dinsd, I backtracked to improve things a couple times, and then
at the point where I was working on comparisons of relations I started doing a
bunch of refactoring and backtracked to the beginning.  If you are curious
about the evolution you can check out the repository history.

This file stops at the end of Chapter 5 of AIRDT.  It covers only the
relational algebra and how dinsd brings that into Python.  The topic of
persistent databases and database constraints are covered by other documents.

Where reference is made to specific page numbers or figures in AIRTD, this is
based on the bookboon.com edition of the book as available for download on
November 1st, 2012.

This file contains blocks of tests that are meant to exercise edge cases as
well as the primary test cases and examples, because it is the test document
that I used as I was building the system.  It is, therefore, a cross between a
design document, a literate test document, a somewhat detailed discussion of the
system mechanics, and a comparison of dinsd to *Tutorial D*.  It may not be
the best possible introduction to dinsd, but if you can make your way through
it you will have a fairly deep understanding of dinsd and its relationship to
TTM and its correspondence (or occasionally lack thereof) to *Tutorial D*.

Note that the error messages shown here are not part of the API.  In theory
the exception classes that result from specific error cases are, but in
practice I have not worried about making them "right" at this point.  So while
the rest of the API is my current thinking on what the API should be, the
exception classes are not even my current thinking, they are either accidents
or relatively thoughtless implementations.  I'll fix that at some point.


About the Name
~~~~~~~~~~~~~~

When I started dinsd, I knew there was no way I could fully implement
TTM's *D* in Python.  But I wanted something very like *D*.  Given that I was
implementing *D* in Python, I figured I should use the name of some character
that had something to do with the letter D, and Dinsdale immediately came to
mind.  However, since my program was not going to be a full D, I decided I'd
shorten the name to indicate it was only a partial thing.  This also served to
make it a more uniquely searchable term.  I contemplated both 'dins' and
'dinsd', settling on the latter because it had two Ds in it.

After a bit I realized that I was developing something closer to the true
spirit of *D* than I had initially thought I'd be able to manage, but still
something that did not conform to TTM.  It occurred to me that 'dinsd' could
almost, but not quite, be interpreted as "D Is Not D", and I briefly
considered changing the name to disnd.  That's a lot harder to pronounce,
though.

And then it occurred to me that the big difference between my (non-conformant)
implementation of *D* and a real *D* is that Python does not have compile
time, static typing.  And thus the real acronym behind the name dinsd was
revealed:

    D Is Not Static D

It is otherwise as close to *D* as I understood how to make it.


About the Namespace(s)
~~~~~~~~~~~~~~~~~~~~~~

The relational algebra part of dinsd is its core.  All of the functions and
classes that are used in the algebra (and documented in this document) are
available at the top level of the dinsd namespace.  We could access all of
dinsd through this namespace, using a single import::

    >>> import dinsd

However, it will be much more convenient for this document to use the various
names without referencing them from the ``dinsd`` namespace, so we will import
them into the document namespace one by one as we cover them.  An application
program is of course free to either refer to everything though an import such
as the preceding, or to import the names that it wants to use directly into
its own namespace.

On the other hand, if you are experimenting with dinsd and relational algebra
in the Python shell, it can be very useful to do ``from dinsd import *``.  Per
best practices for Python programming this is the *only* time you should use
that form.

dinsd also defines a separate namespace that is used during expression
evaluation (much more on that below).  It is named ``expression_namespace``,
and starts out containing almost the same names and values as those imported by
``from dinsd import *``.  The exceptions are ``expression_namespace`` itself,
and ``ns``, which we'll learn more about later. ::

    >>> g = {}
    >>> exec("from dinsd import *", g)
    >>> from dinsd import expression_namespace
    >>> (g.keys() - expression_namespace.keys() - {'__builtins__'} == 
    ...                     {'expression_namespace', 'ns'})
    True

(``__builtins__`` is sometimes present and sometimes not, and I haven't
figured out why.)

The ``expression_namespace`` is a global registry, so only truly global
objects such as application created types and functions should be registered
in it.



Databases, Terminology, and Conformance
---------------------------------------

A (Very) Few Words About Databases
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The concept of a database holding a set of relations is central to *Tutorial
D*.  However, those concepts are not really required to understand the bulk of
AIRDT, which is concerned with the relational algebra.  Therefore we will defer
all discussion of the syntax related to such matters to other documents that
deal with them specifically, and here we will focus only on the syntax for
declaring relations and manipulating them with the relational algebra.


Terminology
~~~~~~~~~~~

A database is a set of relations.  Per TTM and AIRDT, a relation consists of a
header plus a set of ``tuples`` that conform to that header.  Since Python
already uses the term ``tuple`` to refer to a different concept, we will fall
back to the less precise but more widely recognized term ``row``.  I'm sure
the authors of TTM would/will be annoyed by this, but it is less confusing
than trying to reuse the world ``tuple`` in a Python context.

So, a relation consists of a name and a header.  The header consists of a list
of attributes, with each attribute having a type.  A row consists of a header
and a set of attributes, one attribute per attribute in the header; plus a
specific value, of the type defined by the header, for each attribute.


Lack of Static Typing
~~~~~~~~~~~~~~~~~~~~~

As alluded to in the ``About`` section, while TTM requires static tying
(Prescription 1), Python is not a compiled language and does not support
static typing.  Therefore dinsd can never be technically compliant with TTM.

We are also not, here, embedding a computationally complete ``D`` inside
Python and providing a way to talk to Python (one way to satisfy OO
prescription 3).  We can chose to view Python plus dinsd as being (almost)
``D``, and since Python is computationally complete, this would satisfy the
prescription.  This makes the synthetic acronym meaning of the name dinsd even
more apropos, since the dinsd module by itself cannot be ``D``, even
leaving aside static typing considerations.

Although Python does not support static typing, it is *strictly* typed.  Every
object in the system has a strict type.  It is the names that refer to those
objects that are not themselves typed, and can refer to any object.  So dinsd
does not provide "early warning" if you mix types inappropriately.  It does,
however, provide runtime warning.  I believe that dinsd observes the spirit
of *TTM*, even if it doesn't conform to the letter.

Python and dinds, then, separate the creation of a relation *object* (which has
a type) from creating a reference to that object in a database (and therefore
storing it as global, application independent state).  As indicated, we will
postpone the discussion of creating databases until later.  When we do discuss
them, however, we will see that the names referring to the persistent
relations *are* typed, bringing Python-plus-dinsd that much closer to a real ``D``.

For now, though, we will create relation objects in the namespace of this test
document, where the names that point to them will not be typed, and they will
not outlast the test run.  (This association of a name with a non-persistent
relation is in fact not dissimilar to what *Tutorial D* does with its ``WITH``
statement, just longer lasting.)



Scalars, Rows, and Relations
----------------------------

Types, User Defined Types, and Scalars
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Defining new types in Python is very simple:  you define a class.  Classes
are, in fact, of type ``type`` in Python.  The terms ``class`` and ``type``
can be used pretty much interchangeably in Python, and we will use the word
``type`` unless we are actually talking about defining a class using the
``class`` keyword, or talking about a type that we defined in that way.
You will see that Python's error messages generally use the world ``class`` to
refer to type/class objects, but we prefer type in the context of dinsd
because it matches more closely with the terminology used in the relational
literature.

To define the type for an attribute, we need a name that refers to the correct
type object (class).  For standard types (``int``, ``str``, etc), we can use
the Python built in types.  For what TTM calls "user defined types", we define
a new class.

In order to be usable in relation definitions, every user defined scaler type
(ie: non-relation type) used with dinsd must conform to the common syntax and
semantics of the Python built in types:  the type constructor at a minimum
accepts one argument, and either raises an error if the argument (the
"selector" in TTM/AIRDT terms) cannot be converted into a valid instance of
the type, or a valid instance of the type if it can be.  In Python terms, this
means that the class's ``__init__`` method must accept at least one positional
argument.  (For the purposes of dinsd, this argument should always be present,
even if it is optional in the class definition for other reasons.)

It is also critical that, like the Python built in types, the user defined
type accept an instance of itself as valid input, and return an equivalent
instance.  (Note that this requirement is not true for *Tutorial D*.  We'll
discuss why it is true for dinsd below.)

XXX: The above restriction needs to be removed!  (I've started to do so,
starting with ``row``.)

In addition to the conformance checking in the ``__init__`` method, there are
some auxiliary methods that every user defined scaler type must have.
Therefore dinsd provides a base type for such user defined scalers, named
``Scaler``.  A simple ``Scaler`` subclass must always store its value in an
attribute named ``value`` in order for these additional methods to function
correctly.

There is, however, nothing preventing an application from defining a type
without using ``Scaler`` as the superclass, or using ``Scaler`` and
implementing a more complex value store.  If an application does so it is
responsible for correctly implementing the equivalents of all of the methods
that ``Scaler`` provides.  Specifically, it must provide all the methods to
define a total ordering of the values.  (TTM does not require a total
ordering, and this constraint in dinsd may eventually be relaxed.)

AITDT uses some custom types, ``SID`` and ``CID``, as examples in section 2.8
and 2.9.  Example 2.4 in section 2.10 (page 42 of my copy of AIRDT) defines
``SID`` this way::

    TYPE SID POSSREP SID { C CHAR
                           CONSTRAINT LENGTH(C) <= 5
                           AND
                           STARTS_WITH(C, 'S')
                           AND
                           IS_DIGITS(SUBSTRING(C,1)))

We can define the ``SID`` and ``CID`` types as follows::

    >>> from dinsd import Scaler
    >>> class ID(Scaler):
    ...
    ...     def __init__(self, id):
    ...         if isinstance(id, self.__class__):
    ...             self.value = id.value
    ...             return
    ...         if not isinstance(id, str):
    ...             raise TypeError(
    ...                 "Expected str but passed {}".format(type(id)))
    ...         if (2 <= len(id) <=4 and id.startswith(self.firstchar) and
    ...                 id[1:].isdigit()):
    ...             self.value = id
    ...         else:
    ...             raise TypeError("Expected '{}' followed by one to "
    ...                             "three digits, got {!r}".format(
    ...                             self.firstchar, id))
    ...
    >>> class SID(ID):
    ...     firstchar = 'S'
    ...
    >>> class CID(ID):
    ...     firstchar = 'C'

This definition corresponds to example 2.4 on page 47 of AIRDT.  It looks
somewhat more complicated, but this is because we have a clause for accepting
an instance as a "selector", and also because we are defining the error
messages resulting from "selection" failure, whereas example 2.4 is leaving
those error messages to be automatically generated by the *Tutorial D*
constraint system.

Semantically the above is equivalent to the AIRDT example.  We are using
Python built in functions for the checks, whereas in *Tutorial D* the check
functions are user defined (and their definition is never given in the book).
The *Tutorial D* ``C CHAR`` declaration corresponds to our ``self.value =
id``, with the difference that our self.value is not typed.  The *value* has a
type, which our code is constraining to be ``str`` (Python's equivalent
of *Tutorial D*'s ``CHAR``), but as is normal in Python the name we store it
under is not itself typed.

To prove that this straightforward Python implementation is correct::

    >>> SID('S1')
    SID('S1')
    >>> CID('C1')
    CID('C1')
    >>> SID(SID('S1'))
    SID('S1')
    >>> print(SID('S2'))
    S2
    >>> SID('1')
    Traceback (most recent call last):
        ...
    TypeError: Expected 'S' followed by one to three digits, got '1'
    >>> CID(1)
    Traceback (most recent call last):
        ...
    TypeError: Expected str but passed <class 'int'>
    >>> SID('C1')
    Traceback (most recent call last):
        ...
    TypeError: Expected 'S' followed by one to three digits, got 'C1'
    >>> CID('C0003')
    Traceback (most recent call last):
        ...
    TypeError: Expected 'C' followed by one to three digits, got 'C0003'
    >>> SID(CID('C1'))
    Traceback (most recent call last):
        ...
    TypeError: Expected str but passed <class '__main__.CID'>
    >>> SID('S1') == SID('S1')
    True
    >>> SID('S1') != CID('C1')
    True
    >>> SID('S2') == SID(SID('S2'))
    True
    >>> SID('S1') < SID('S2')
    True
    >>> CID('C7') >= CID('C7')
    True
    >>> SID('S1') > CID('C2')
    Traceback (most recent call last):
        ...
    TypeError: unorderable types: SID() > CID()

In dinsd there is one more thing we need to do with user defined types: we
need to register them in the ``expression_namespace``.  This allows them to be
used in string valued expressions, about which more later. ::

    >>> from dinsd import expression_namespace
    >>> expression_namespace['SID'] = SID
    >>> expression_namespace['CID'] = CID


Defining Relations
~~~~~~~~~~~~~~~~~~

The *Tutorial D* syntax for defining a Relation type is given on page
42 of my copy of AIRDT::

    RELATION { StudentId SID, Name NAME, CourseId CID }

The braces indicate that the attributes are a set, not an ordered list.

In translating these examples to dinsd, I'm going to do two things in addition
to using the dinsd syntax: (1) I'm going to use ``str`` instead of a ``NAME``
type both for simplicity and to show how Python standard types work in a dinsd
context, and (2) I'm going to convert from *Tutorial D*s case convention to
the :pep:``8`` convention, which is used by a lot of Python software.  The
:pep:``8`` convention that classes (types) are named with CamelCase, while
variables, methods, and functions are named with
underscore_separated_lower_case.  In standard Python code we also make more
sparing use of blanks around grouping operators such as braces and parenthesis.

With those preliminaries out of the way...the dinsd equivalent to
the *Tutorial D* ``RELATION`` keyword is named ``rel``::

    >>> from dinsd import rel

``rel`` is used in a fashion very similar to the above *Tutorial D# statement,
except that it is an expression that returns a relation type definition
as its result:

    >>> x = rel({'student_id': SID, 'name': str, 'course_id': CID})
    >>> x 
    <class 'dinsd.rel({'course_id': CID, 'name': str, 'student_id': SID})'>

(Recall that types in Python are defined by classes.)

In Python, dictionaries are unordered, and the keys of a dictionary are a set
(that is, no two keys may be equal), so this declaration fully satisfies
the *TTM* requirement that the header of a relation be a set of pairs in
which order does not matter::

    >>> y = rel({'student_id': SID, 'course_id': CID, 'name': str})
    >>> x is y
    True

However, rather than indicating a syntax error if a key is repeated, Python
simply uses the last value specified::

    >>> x == rel({'student_id': SID, 'course_id': CID, 'name': int, 'name': str})
    True

The order does matter in this case::

    >>> y = rel({'student_id': SID, 'course_id': CID, 'name': str, 'name': int})
    >>> x == y
    False
    >>> y
    <class 'dinsd.rel({'course_id': CID, 'name': int, 'student_id': SID})'>

There is an alternate way to declare relations in dinsd that *will* raise an
error if keys are duplicated, and that is to use the Python keyword syntax to
specify the attribute names and values::

    >>> x == rel(student_id=SID, course_id=CID, name=str)
    True
    >>> x == rel(student_id=SID, course_id=CID, name=str, name=int)
    Traceback (most recent call last):
        ...
    SyntaxError: keyword argument repeated

To use this form the attribute names must be valid Python identifiers.  This
restriction is in fact true in the general case.  Much of the dinsd code does
not depend on relation attributes being Python identifiers, but dinsd is not
tested to see if non-identifier attributes are supported, so it is unlikely to
work in practice even if it works in specific cases (for example, some things
that happen work in CPython will fail in other Python implementations, and
possibly vice versa.)

Aside: Experienced Python users will observe that the above gives two
different ways to do the same thing, and in Python we usually try to stick to
there being "one obvious way to do it".  dinsd is primarily operating on data
types that are expressible as dictionaries and sets, and therefore it makes
sense to support that notation throughout, for consistency.  As discussed
above, however, the keyword notation has certain advantages, and so we also
support it where possible.  In this we are following another Python principle,
that of "least surprise", by supporting each of the different notations in all
of the contexts in which it makes sense.

The dictionary and keyword forms may be mixed.  This is supported in analog to
Python's support for both a dictionary and a keyword list in the ``dict``
constructor, and because it is occasionally convenient to use this form to
merge some new attributes into an existing attribute list.  (This test looks a
little odd because it is important that we make sure that the input dictionary
is not mutated.) ::

    >>> orig = {'student_id': SID, 'name': int}
    >>> test = orig.copy()
    >>> x == rel(test, course_id=CID, name=str)
    True
    >>> assert test == orig

As with ``dict``, this is an update operation, so the values specified by
the keyword arguments overwrite those in the dictionary argument.

Passing something that is not a type as the value of a keyword argument is
invalid::

    >>> rel(course_id=CID('C1'))            # doctest: +NORMALIZE_WHITESPACE
    Traceback (most recent call last):
        ...
    ValueError: Invalid value for attribute 'course_id' in relation type
        definition: "CID('C1')" is not a type

Relation types (and therefore relation objects, but we'll get to those later)
have an attribute ``degree``, which gives the number of attributes. ::

    >>> x.degree
    3

Every relation type also has a ``header`` attribute which records the list of
attributes and their types, and corresponds to the TTM ``HEADER``.  (The
attributes are not not Python attributes of the relation, but they *are*
Python attributes of the *rows*, which we haven't discussed yet.)  The header
is a plain Python dictionary, so we need to sort its items to get a consistent
representation for the doctest::

    >>> sorted(x.header.items())            # doctest: +NORMALIZE_WHITESPACE
    [('course_id', <class '__main__.CID'>), ('name', <class 'str'>),
    ('student_id', <class '__main__.SID'>)]

An important aside: in dinsd a relation type is *entirely* defined by its
header, as required by *TTM*.  This being Python, there is nothing preventing
a program from modifying the ``header`` attribute of a relation (or the
``degree`` attribute, for that matter).  This will of course completely screw
things up, so don't do that.  That means if you want to use the information
from the ``header`` attribute in a way that is not read only *make a copy
of it*::

    >>> myvar = x.header.copy()


Row Literals
~~~~~~~~~~~~

Remember, a dinsd "row" corresponds to a *Tutorial D* "tuple".  (Unlike
"attribute", which is used in Python fairly loosely, "tuple" means something
very precise in Python, and it isn't a *Tutorial D* tuple).

On page 44 of my copy of AIRDT, there is this example of a TUPLE literal::

    TUPLE { StudentId SID('S1'), CourseId CID('C1'), Name NAME('Anne') }

Considering that I'm coming from a Python dynamically typed background, it
took me an inordinately long time to understand that the above is *implicitly
typed* by the names of its attributes and the type of its concrete values.
Once I figured that out, though, it was simple to support it in dinsd::

    >>> from dinsd import row
    >>> x = row({'student_id': SID('S1'), 'course_id': CID('C1'), 'name': 'Anne'})
    >>> x
    row({'course_id': CID('C1'), 'name': 'Anne', 'student_id': SID('S1')})

A ``row`` object has Python attributes to match the relational attributes of
its header::

    >>> x.course_id
    CID('C1')
    >>> x.name
    'Anne'

Similar to ``rel``, there is a keyword form for ``row`` literals::

    >>> y = row(student_id=SID('S1'), course_id=CID('C1'), name='Anne')
    >>> x == y
    True

This is the preferred form for row literals, since unlike the dictionary
form it will object to to multiply defined keywords::

    >>> row(student_id=SID('S1'), student_id=SID('S2'))
    Traceback (most recent call last):
        ...
    SyntaxError: keyword argument repeated

All though rows are a collection of attribute/value pairs that have no
intrinsic order, we do for convenience define a total ordering for rows of
like type.  This is principally to provide a way to get default output in a
predictable order, which improves the user experience when printing relations
for debugging purposes.  (If the order might change on every print, comparing
debugging output would be much more difficult).  It also makes it possible to
test the output in doctests like this one.

The default ordering for tuples is defined by first sorting the attributes in
the row alphabetically, and then sorting the rows by comparing the values
(which must themselves have a total ordering) of each attribute in turn from
the first to the last alphabetically::

    >>> y = row(student_id=SID('S2'), course_id=CID('C1'), name='Boris')
    >>> x == y
    False
    >>> x < y
    True
    >>> x <= y
    True
    >>> x <= x
    True
    >>> x < x
    False
    >>> y > x
    True
    >>> y >= x
    True
    >>> y > y
    False
    >>> y >= y
    True

You can't order rows of unlike type, though::

    >>> x < row(student_id=SID('S2'))         # doctest: +NORMALIZE_WHITESPACE
    Traceback (most recent call last):
        ...
    TypeError: unorderable types: row({'course_id': CID, 'name': str,
        'student_id': SID})() < row({'student_id': SID})()

This default ordering of the attribute names is also used in the repr as you
can see above.  Rows also have a simpler string representation, that uses
the string representation of both the attribute names and the values, with
an ``=`` separating them.  The whole thing is enclosed in ``{}`` to indicate
that it is a set rather than an ordered tuple::

    >>> str(x)
    '{course_id=C1, name=Anne, student_id=S1}'

In Python's standard programming conventions, names that begin with a ``_`` are
usually intended to be "private" names, names that are not part of the public
API.  However, we have some attributes that we want as part of the *public*
API, but we don't want to use names without a leading underscore, because those
might conflict with relational attribute names.  In a similar situation (the
"named tuple" type), Python just ignores this dichotomy and uses names that
start with ``_`` as part of the public API.  dinsd is a little more strict
about its naming, by also *appending* a ``_`` to names that start with a ``_``
but are nonetheless intended to be part of the public API.

Rows have two public non-relational attributes of interest.  These correspond
to the ``degree`` and ``header`` attributes of relations, but named as
explained above::

    >>> x._degree_
    3
    >>> sorted(x._header_.items())      # doctest: +NORMALIZE_WHITESPACE
    [('course_id', <class '__main__.CID'>), ('name', <class 'str'>),
     ('student_id', <class '__main__.SID'>)]

As with relations, these attributes are not read-only, but a program should
not modify them.  Make a copy if you want to manipulate the header data in
some way::

    >>> myvar = x._header_.copy()

As usual, dictionary and keywords can be mixed::

    >>> row({'name': 'Anne', 'student_id': SID('S3')}, student_id=SID('S1'))
    row({'name': 'Anne', 'student_id': SID('S1')})

Types are not legal as row values, only non-type object instances. ::

    >>> row({'student_id': SID})           # doctest: +NORMALIZE_WHITESPACE
    Traceback (most recent call last):
        ...
    ValueError: Invalid value for attribute 'student_id': "<class
        '__main__.SID'>" is a type

dinsd itself places no other requirements on row values, but when we get to
the persistence layer we'll see that only objects that are either
representable natively by the persistent store or can be pickled can be
stored.


Row Types
~~~~~~~~~

Rows have a type, just like relations have a type::

    >>> xt = type(row(student_id=SID('S1'), name='Anne'))
    >>> xt
    <class 'dinsd.row({'name': str, 'student_id': SID})'>

And, like relation types, only one type object is created for each distinct
row type::

    >>> yt = type(row(name='Anne', student_id=SID('S1')))
    >>> xt is yt
    True

Row types have ``_header_`` and ``_degree_`` attributes, just like
their instances::

    >>> sorted(yt._header_.items())
    [('name', <class 'str'>), ('student_id', <class '__main__.SID'>)]
    >>> yt._degree_
    2

(In fact, the ``_header`` and ``_degree_`` attributes of the rows are
references to the class (type) attributes.)

The type can be used to create a new row of that type::

    >>> yt(student_id=SID('S1'), name='Anne')
    row({'name': 'Anne', 'student_id': SID('S1')})

For consistency with ``rel`` and ``row``, these are also valid ways to call
the row constructor::

    >>> yt({'student_id': SID('S1'), 'name': 'Anne'})
    row({'name': 'Anne', 'student_id': SID('S1')})
    >>> yt({'student_id': SID('S1')}, name='Anne')
    row({'name': 'Anne', 'student_id': SID('S1')})

This is an invalid call, though::

    >>> yt({'student_id': SID('S1')}, {'name': 'Anne'})
    Traceback (most recent call last):
        ...
    TypeError: row() takes at most one positional argument (2 given)

A useful difference between a row type constructor and ``row`` is that the
constructor knows what type the attributes are supposed to be.  It can
therefore coerce appropriate literals of other types that are passed to it
into the correct type by passing them to the attribute's type function::

    >>> yt(student_id='S1', name='Anne')
    row({'name': 'Anne', 'student_id': SID('S1')})

Passing it a literal of an inappropriate type will produce an error::

    >>> yt(student_id=1, name='Anne')       # doctest: +NORMALIZE_WHITESPACE
    Traceback (most recent call last):
        ...
    TypeError: Expected str but passed <class 'int'>; 1 invalid for
        attribute student_id
    >>> yt(student_id='C1', name='Anne')    # doctest: +NORMALIZE_WHITESPACE
    Traceback (most recent call last):
        ...
    TypeError: Expected 'S' followed by one to three digits, got 'C1'; 'C1'
        invalid for attribute student_id

Here the first requirement is that the literal be a string literal, but
if it passes that test, it also has to be something that matches the
requirements imposed by our ``SID`` user defined type.

Passing the constructor the wrong attribute names or the wrong number of
attributes will also produce an error::

    >>> yt(student_id='S1', foo='Anne')
    Traceback (most recent call last):
        ...
    TypeError: Invalid attribute name foo
    >>> yt(student_id='C1', name='Anne',               #doctest: +ELLIPSIS
    ...    course_id=CID('C1'))
    Traceback (most recent call last):
        ...
    TypeError: Expected 2 attributes, got 3 ({...})
    >>> yt(student_id='C1')                            #doctest: +ELLIPSIS
    Traceback (most recent call last):
        ...
    TypeError: Expected 2 attributes, got 1 ({...})

There is no such thing as an "empty" row, so passing no arguments is als
an error::

    >>> yt()
    Traceback (most recent call last):
        ...
    TypeError: Expected 2 attributes, got 0 ({})
    >>> yt({})
    Traceback (most recent call last):
        ...
    TypeError: Expected 2 attributes, got 0 ({})
    

Relation Literals
~~~~~~~~~~~~~~~~~

Given the preceding, the dinsd equivalent to AIRDT's example 2.3 (page 45)::

    RELATION {
        TUPLE { StudentId SID('S1'), CourseId CID('C1'), Name NAME('Anne')},
        TUPLE { StudentId SID('S1'), CourseId CID('C2'), Name NAME('Anne')},
        TUPLE { StudentId SID('S2'), CourseId CID('C1'), Name NAME('Boris')},
        TUPLE { StudentId SID('S3'), CourseId CID('C3'), Name NAME('Cindy')},
        TUPLE { StudentId SID('S4'), CourseId CID('C1'), Name NAME('Devinder')},
        }

should be reasonably obvious.  We'll do the set-of-rows form first, since
it is closest to the *Tutorial D* notation, and also because it is the
most complete and formal specification of a relation literal::

    >>> x = rel({
    ...     row({'student_id': SID('S1'), 'course_id': CID('C1'), 'name': 'Anne'}),
    ...     row({'student_id': SID('S1'), 'course_id': CID('C2'), 'name': 'Anne'}),
    ...     row({'student_id': SID('S2'), 'course_id': CID('C1'), 'name': 'Boris'}),
    ...     row({'student_id': SID('S3'), 'course_id': CID('C3'), 'name': 'Cindy'}),
    ...     row({'student_id': SID('S4'), 'course_id': CID('C1'), 'name': 'Devinder'}),
    ...     })
    >>> x                                    # doctest: +NORMALIZE_WHITESPACE
    rel({row({'course_id': CID('C1'), 'name': 'Anne',
    'student_id': SID('S1')}), row({'course_id': CID('C1'), 'name': 'Boris',
    'student_id': SID('S2')}), row({'course_id': CID('C1'), 'name': 'Devinder',
    'student_id': SID('S4')}), row({'course_id': CID('C2'), 'name': 'Anne',
    'student_id': SID('S1')}), row({'course_id': CID('C3'), 'name': 'Cindy',
    'student_id': SID('S3')})})

As mentioned above, the rows are sorted.  This gives us a consistent repr for
doctests, and makes it easier to compare relations by their repr.  The
``repr`` of a relation is exactly the most formal version of the literal that
can be used to create that relation.

The ``repr`` is a bit hard to parse visually, though, since it is so long and
ends up line wrapped at arbitrary points.  Relations (as opposed to
relation *types*) also have an ``str`` form that is much easier to parse
visually::

    >>> print(x)
    +-----------+----------+------------+
    | course_id | name     | student_id |
    +-----------+----------+------------+
    | C1        | Anne     | S1         |
    | C1        | Boris    | S2         |
    | C1        | Devinder | S4         |
    | C2        | Anne     | S1         |
    | C3        | Cindy    | S3         |
    +-----------+----------+------------+

Since a row's type is completely described by the attribute names and the
types of the values, we don't really need to first turn the dict into a row
before passing it in to ``rel``.  ``rel`` can process dictionaries to create
a relation just as easily as it can process rows.  However, Python sets
can't hold dictionaries (because Python dictionaries are mutable), so we
instead pass them as a list::

    >>> y = rel((
    ...     {'student_id': SID('S1'), 'course_id': CID('C1'), 'name': 'Anne'},
    ...     {'student_id': SID('S1'), 'course_id': CID('C2'), 'name': 'Anne'},
    ...     {'student_id': SID('S2'), 'course_id': CID('C1'), 'name': 'Boris'},
    ...     {'student_id': SID('S3'), 'course_id': CID('C3'), 'name': 'Cindy'},
    ...     {'student_id': SID('S4'), 'course_id': CID('C1'), 'name': 'Devinder'},
    ...     ))
    >>> y == x
    True

A Python function can turn a list of arguments into an iterator automatically,
and many Python functions that accept an iterator also accept an equivalent
list of arguments.  dinsd supports this call form for the ``rel`` function::

    >>> y = rel(
    ...     {'student_id': SID('S1'), 'course_id': CID('C1'), 'name': 'Anne'},
    ...     {'student_id': SID('S1'), 'course_id': CID('C2'), 'name': 'Anne'},
    ...     {'student_id': SID('S2'), 'course_id': CID('C1'), 'name': 'Boris'},
    ...     {'student_id': SID('S3'), 'course_id': CID('C3'), 'name': 'Cindy'},
    ...     {'student_id': SID('S4'), 'course_id': CID('C1'), 'name': 'Devinder'},
    ...     )
    >>> y == x
    True

We can even mix dictionaries and rows in these forms, because the dictionaries
and rows are logically interchangeable. ::

    >>> y = rel((
    ...     row({'student_id': SID('S1'), 'course_id': CID('C1'), 'name': 'Anne'}),
    ...     {'student_id': SID('S1'), 'course_id': CID('C2'), 'name': 'Anne'},
    ...     row({'student_id': SID('S2'), 'course_id': CID('C1'), 'name': 'Boris'}),
    ...     {'student_id': SID('S3'), 'course_id': CID('C3'), 'name': 'Cindy'},
    ...     row({'student_id': SID('S4'), 'course_id': CID('C1'), 'name': 'Devinder'}),
    ...     ))
    >>> y == x
    True
    >>> y = rel(
    ...     row({'student_id': SID('S1'), 'course_id': CID('C1'), 'name': 'Anne'}),
    ...     {'student_id': SID('S1'), 'course_id': CID('C2'), 'name': 'Anne'},
    ...     row({'student_id': SID('S2'), 'course_id': CID('C1'), 'name': 'Boris'}),
    ...     {'student_id': SID('S3'), 'course_id': CID('C3'), 'name': 'Cindy'},
    ...     row({'student_id': SID('S4'), 'course_id': CID('C1'), 'name': 'Devinder'}),
    ...     )
    >>> y == x
    True

You might ask why we bother with a row object at all...one answer has to do
with Python's strong typing.  If we didn't have a type for rows that was
distinct from the base dictionary type, the implementation and the error
messages would be much messier and more difficult to work with.  However,
there is also the pedagogical reason: the keyword form of ``row`` will reject
duplicate keys, whereas the dictionary form will silently use the last of a
set of duplicates. 

If you wish to hew as close as possible to the type-safety
of ``D``, you should use the keyword form of ``row`` with either the
multiple-argument or non-set iterator form of ``rel``::

    >>> y = rel(
    ...     row(student_id=SID('S1'), course_id=CID('C1'), name='Anne'),
    ...     row(student_id=SID('S1'), course_id=CID('C2'), name='Anne'),
    ...     row(student_id=SID('S2'), course_id=CID('C1'), name='Boris'),
    ...     row(student_id=SID('S3'), course_id=CID('C3'), name='Cindy'),
    ...     row(student_id=SID('S4'), course_id=CID('C1'), name='Devinder'),
    ...     )
    >>> y == x
    True

This is because, like the dict literal, the Python set literal will happily
accept duplicate entries::

    >>> y = rel({
    ...     row(student_id=SID('S1'), course_id=CID('C1'), name='Anne'),
    ...     row(student_id=SID('S1'), course_id=CID('C2'), name='Anne'),
    ...     row(student_id=SID('S2'), course_id=CID('C1'), name='Boris'),
    ...     row(student_id=SID('S3'), course_id=CID('C3'), name='Cindy'),
    ...     row(student_id=SID('S4'), course_id=CID('C1'), name='Devinder'),
    ...     row(student_id=SID('S2'), course_id=CID('C1'), name='Boris'),
    ...     })
    >>> y == x
    True

The non-set form, however, will produce an error if you try to specify two
identical rows in a relation literal::

    >>> rel(row(foo=1), row(foo=2), row(foo=1), row(foo=4))
    Traceback (most recent call last):
        ...
    ValueError: Duplicate row: row({'foo': 1}) in row 2 of input

(When looking at that error message, remember that Python list indexes
start from zero.)

At this point you might wonder why the set/dict notation is considered the
more formal one and is used for the repr.  This is because the "formal"
notation indicates that the attributes *are* a set, and the body of the
relation *is* a set, which is important information to convey.

Although it isn't technically a literal form, this is a good place to mention
that any Python iterator can be used as the argument to ``rel``.  Here is
an example using a generator::

    >>> def multiplication_table():
    ...     for a in range(10):
    ...         for b in range(10):
    ...             yield row(a=a, b=b, result=a*b)
    >>> print(rel(multiplication_table()))              # doctest: +ELLIPSIS
    +---+---+--------+
    | a | b | result |
    +---+---+--------+
    | 0 | 0 | 0      |
    | 0 | 1 | 0      |
    | 0 | 2 | 0      |
    ...
    | 2 | 8 | 16     |
    | 2 | 9 | 18     |
    | 3 | 0 | 0      |
    | 3 | 1 | 3      |
    ...
    | 9 | 7 | 63     |
    | 9 | 8 | 72     |
    | 9 | 9 | 81     |
    +---+---+--------+

We could equally well have used a generator expression, or a list
comprehension.

It is possible to define a single row literal (this may seem obvious, but it
needs to be tested since it is an edge case)::

    >>> rel({'foo': 1, 'bar': CID('C1')})
    rel({row({'bar': CID('C1'), 'foo': 1})})
    >>> rel(row(student_id=SID('S1')))
    rel({row({'student_id': SID('S1')})})

It is an error to specify unlike types in a relation literal.  The type of the
first row is assumed to be the "correct" type of the relation::

    >>> rel(row(foo=1), row(foo=1, bar=2))     # doctest: +NORMALIZE_WHITESPACE
    Traceback (most recent call last):
        ...
    TypeError: Row header does not match relation header in row 1 (got
        row({'bar': 2, 'foo': 1}) for <class 'dinsd.rel({'foo': int})'>)

    >>> rel(row(foo=1), row(foo='bar'))        # doctest: +NORMALIZE_WHITESPACE
    Traceback (most recent call last):
        ...
    TypeError: Row header does not match relation header in row 1 (got
        row({'foo': 'bar'}) for <class 'dinsd.rel({'foo': int})'>)

The errors are a bit different if we use the dictionary form::

    >>> rel({'foo': 1}, {'foo': 1, 'bar': 2})       #doctest: +ELLIPSIS
    Traceback (most recent call last):
        ...
    TypeError: Expected 1 attributes, got 2 ({...}) in row 1
    >>> rel({'foo': 1}, {'foo': 'bar'})       # doctest: +NORMALIZE_WHITESPACE
    Traceback (most recent call last):
        ...
    ValueError: invalid literal for int() with base 10: 'bar'; 'bar' invalid
        for attribute foo in row 1

This is because a ``row`` has a header, so that is compared against the
relation header, while in the dictionary case the values get passed to the
type function for the attribute, and it is that type function that raises the
type error.

It is valid to specify a relation literal with no attributes (degree zero) and
no rows (cardinality 0).  Given that we are allowing a plain dictionary to
represent a row, there is a syntactic ambiguity in the ``rel`` notation: is
``rel()`` the degree 0 cardinality 0 type, or an instance of a relation of
that type?  The same question applies to ``rel({})``.  Because our type name
notation uses ``rel({})`` to represent the degree zero cardinality zero type,
we resolve the ambiguity by declaring that ``rel({})`` returns that type::

    >>> rel({})
    <class 'dinsd.rel({})'>

while ``rel()`` returns an instance of that type::

    >>> rel()
    rel({})()

We could also resolve the ambiguity by not allowing an unadorned dictionary to
represent a row.  There are arguments both in favor and against, and I won't
know which one weighs more with me until I've actually coded an application in
dinsd, so for the moment we have the situation described above.

Note that passing any empty iterator to ``rel`` is equivalent to the second
form above::

    >>> rel() == rel(set()) == rel([]) == rel(tuple())
    True

We can also write a literal for a relation with no attributes (degree 0) and
one row with no attributes (cardinality 1)::

    >>> rel(row())
    rel({row({})})



Extending Relation Types Into Relations
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Section 2.11 of AITDT talks about declaring variables.  In Python we don't
really have variables, we just have names that are references to some object,
and which object (and which type of object) they reference can change over
time.  This is Python's dynamically typed nature.  As I've mentioned already,
the objects themselves are strongly typed.

The dinsd equivalent of example 2.5::

    VAR SN SID INIT SID ( 'S1' ) ;

is simply::

    >>> sn = SID('S1')
    >>> sn
    SID('S1')

There is no Python equivalent to an actual variable declaration, and therefore
no equivalent to a variable *without* an initializer.

The dinsd equivalent to example 2.6::

    VAR ENROLLMENT BASE RELATION { StudentId SID,
                                   Name NAME,
                                   CourseId CID )
                        KEY ( StudentId, CourseId ) ;

is::

    >>> enrollment = rel(student_id=SID, name=str, course_id=CID)()
    >>> enrollment
    rel({'course_id': CID, 'name': str, 'student_id': SID})()

(We are deferring the discussion of the dinsd equivalent of the KEY phrase
until the general discussion of database constraints.)

Note carefully the ``()`` at the end of that expression.  What is happening
here is that we are using the type declaration form of ``rel``, and
then *calling the type* to obtain an actual relation.  Since we specify no rows
for the relation, we get an empty instance of that relation type.  You can
see that a bit more clearly in table form::

    >>> print(enrollment)
    +-----------+------+------------+
    | course_id | name | student_id |
    +-----------+------+------------+
    +-----------+------+------------+

We could equally well write the above as follows::

    >>> IsEnrolledOn = rel(student_id=SID, name=str, course_id=CID)
    >>> enrollment = IsEnrolledOn()

That is, in Python we can have a name that points to the type object just as
easily as we can have a name that points to an instance of that type.

There's no point in going over the rest of the differences between Python and
the *Tutorial D* syntax covered by section 2.11 here...the above is the only
part that involves dinsd-specific concepts.

If we want to create an instance of the type now pointed to by
``IsEnrolledOn`` that does have content, we have a number of choices.  Here
is one rather wordy way of doing it::

    >>> is_enrolled_on = IsEnrolledOn(
    ...     IsEnrolledOn.row({'student_id': SID('S1'), 'course_id': CID('C1'),
    ...                       'name': 'Anne'}),
    ...     IsEnrolledOn.row({'student_id': SID('S1'), 'course_id': CID('C2'),
    ...                       'name': 'Anne'}),
    ...     IsEnrolledOn.row({'student_id': SID('S2'), 'course_id': CID('C1'),
    ...                       'name': 'Boris'}),
    ...     IsEnrolledOn.row({'student_id': SID('S3'), 'course_id': CID('C3'),
    ...                       'name': 'Cindy'}),
    ...     IsEnrolledOn.row({'student_id': SID('S4'), 'course_id': CID('C1'),
    ...                       'name': 'Devinder'}),
    ...     )
    >>> print(is_enrolled_on)
    +-----------+----------+------------+
    | course_id | name     | student_id |
    +-----------+----------+------------+
    | C1        | Anne     | S1         |
    | C1        | Boris    | S2         |
    | C1        | Devinder | S4         |
    | C2        | Anne     | S1         |
    | C3        | Cindy    | S3         |
    +-----------+----------+------------+

A relation type has an attribute ``row`` that is a reference to the type for
the rows that may be present in that relation.  That is, ``<relation>.row`` is
the ``row`` type whose ``_header_`` is equal to the ``header`` of the relation::

    >>> IsEnrolledOn.header == IsEnrolledOn.row._header_
    True

As we saw above, the type for a row can convert appropriate standard literals into
the appropriate type, so we can simplify the above::

    >>> x = IsEnrolledOn(
    ...     IsEnrolledOn.row({'student_id': 'S1', 'course_id': 'C1',
    ...                       'name': 'Anne'}),
    ...     IsEnrolledOn.row({'student_id': 'S1', 'course_id': 'C2',
    ...                       'name': 'Anne'}),
    ...     IsEnrolledOn.row({'student_id': 'S2', 'course_id': 'C1',
    ...                       'name': 'Boris'}),
    ...     IsEnrolledOn.row({'student_id': 'S3', 'course_id': 'C3',
    ...                       'name': 'Cindy'}),
    ...     IsEnrolledOn.row({'student_id': 'S4', 'course_id': 'C1',
    ...                       'name': 'Devinder'}),
    ...     )
    >>> x == is_enrolled_on
    True

We can save ourselves some typing by using a temporary name to refer to
this row type function::

    >>> r = IsEnrolledOn.row
    >>> x = IsEnrolledOn(
    ...     r({'student_id': 'S1', 'course_id': 'C1', 'name': 'Anne'}),
    ...     r({'student_id': 'S1', 'course_id': 'C2', 'name': 'Anne'}),
    ...     r({'student_id': 'S2', 'course_id': 'C1', 'name': 'Boris'}),
    ...     r({'student_id': 'S3', 'course_id': 'C3', 'name': 'Cindy'}),
    ...     r({'student_id': 'S4', 'course_id': 'C1', 'name': 'Devinder'}),
    ...     )
    >>> x == is_enrolled_on
    True

And we can get make sure we have no duplicate attributes in our literal
rows by using the keyword version::

    >>> r = IsEnrolledOn.row
    >>> x = IsEnrolledOn(
    ...     r(student_id='S1', course_id='C1', name='Anne'),
    ...     r(student_id='S1', course_id='C2', name='Anne'),
    ...     r(student_id='S2', course_id='C1', name='Boris'),
    ...     r(student_id='S3', course_id='C3', name='Cindy'),
    ...     r(student_id='S4', course_id='C1', name='Devinder'),
    ...     )
    >>> x == is_enrolled_on
    True

We can of course also just use the regular row function, although in that
case we must provide values of the correct type, since the generic
row function doesn't know what type of row we want and produces
whatever we tell it to::

    >>> x = IsEnrolledOn(
    ...     row(student_id=SID('S1'), course_id=CID('C1'), name='Anne'),
    ...     row(student_id=SID('S1'), course_id=CID('C2'), name='Anne'),
    ...     row(student_id=SID('S2'), course_id=CID('C1'), name='Boris'),
    ...     row(student_id=SID('S3'), course_id=CID('C3'), name='Cindy'),
    ...     row(student_id=SID('S4'), course_id=CID('C1'), name='Devinder'),
    ...     )
    >>> x == is_enrolled_on
    True

While it works fine to define rows just with the ``row`` function, using the
``row`` attribute of the relation gives the system opportunities to optimize
the row creation process.  But the rows created by the two methods are
entirely equivalent::

    >>> x = row({'student_id': SID('S1'), 'course_id': CID('C1'),
    ...           'name': 'Anne'})
    >>> x
    row({'course_id': CID('C1'), 'name': 'Anne', 'student_id': SID('S1')})
    >>> y = IsEnrolledOn.row({'student_id': SID('S1'), 'course_id': CID('C1'),
    ...           'name': 'Anne'})
    >>> y
    row({'course_id': CID('C1'), 'name': 'Anne', 'student_id': SID('S1')})
    >>> x == y
    True
    >>> type(x) is type(y) is IsEnrolledOn.row
    True

Just like in *Tutorial D*, if you use a generic ``row`` declaration,
you *must* use the exact type for the attribute values.  Otherwise you
end up with a row with a different type::

    >>> z = row({'course_id': 'C1', 'name': 'Anne', 'student_id': 'S1'})
    >>> z == y
    False

Trying to pass such a row into a relation of a different type will
produce an error::

    >>> x = IsEnrolledOn(               # doctest: +NORMALIZE_WHITESPACE
    ...     row(student_id='S1', course_id='C1', name='Anne'),
    ...     row(student_id='S1', course_id='C2', name='Anne'),
    ...     row(student_id='S2', course_id='C1', name='Boris'),
    ...     row(student_id='S3', course_id='C3', name='Cindy'),
    ...     row(student_id='S4', course_id='C1', name='Devinder'),
    ...     )
    Traceback (most recent call last):
        ...
    TypeError: Row header does not match relation header in row 0 (got
        row({'course_id': 'C1', 'name': 'Anne', 'student_id': 'S1'}) for <class
        'dinsd.rel({'course_id': CID, 'name': str, 'student_id': SID})'>)

We can see exactly why this is by inspecting the ``_header_`` attribute
of the respective rows (the ``_header_`` is just a ``dict``, so we have
to sort it as tuples to get something predictable to doctest::

    >>> sorted(z._header_.items())          # doctest: +NORMALIZE_WHITESPACE
    [('course_id', <class 'str'>), ('name', <class 'str'>),
         ('student_id', <class 'str'>)]
    >>> sorted(y._header_.items())          # doctest: +NORMALIZE_WHITESPACE
     [('course_id', <class '__main__.CID'>), ('name', <class 'str'>),
        ('student_id', <class '__main__.SID'>)]

As you can see, if we don't use the specific type for the value, we get
the actual type of the value in the _header_ (``str`` in this case) which
means the two rows are not the same type, and are not equal.

Since the dictionary representation of a row is logically equivalent to
the row representation, we also have the option of just passing a list
of dictionaries into the relation constructor::

    >>> x = IsEnrolledOn(
    ...     {'student_id': SID('S1'), 'course_id': CID('C1'), 'name': 'Anne'},
    ...     {'student_id': SID('S1'), 'course_id': CID('C2'), 'name': 'Anne'},
    ...     {'student_id': SID('S2'), 'course_id': CID('C1'), 'name': 'Boris'},
    ...     {'student_id': SID('S3'), 'course_id': CID('C3'), 'name': 'Cindy'},
    ...     {'student_id': SID('S4'), 'course_id': CID('C1'), 'name': 'Devinder'},
    ...     )
    >>> x == is_enrolled_on
    True

In this case, too, since the relation knows the types of the attributes, it
can automatically convert appropriate literals into the correct type for the
attribute::

    >>> x = IsEnrolledOn(
    ...     {'student_id': 'S1', 'course_id': 'C1', 'name': 'Anne'},
    ...     {'student_id': 'S1', 'course_id': 'C2', 'name': 'Anne'},
    ...     {'student_id': 'S2', 'course_id': 'C1', 'name': 'Boris'},
    ...     {'student_id': 'S3', 'course_id': 'C3', 'name': 'Cindy'},
    ...     {'student_id': 'S4', 'course_id': 'C1', 'name': 'Devinder'},
    ...     )
    >>> x == is_enrolled_on
    True

Trying to pass the wrong names, inappropriate literals, or wrong number of
attributes will result in an error::

    >>> IsEnrolledOn(
    ...     {'student_id': 'S1', 'course_id': 'C1', 'foo': 'Anne'},
    ...     )
    Traceback (most recent call last):
        ...
    TypeError: Invalid attribute name foo in row 0
    >>> IsEnrolledOn(                   # doctest: +NORMALIZE_WHITESPACE
    ...     {'student_id': 'S1', 'course_id': 'S1', 'name': 'Anne'},
    ...     )
    Traceback (most recent call last):
        ...
    TypeError: Expected 'C' followed by one to three digits, got 'S1'; 'S1'
        invalid for attribute course_id in row 0
    >>> IsEnrolledOn(                               # doctest: +ELLIPSIS
    ...     {'student_id': SID('S1'), 'name': 'Anne'},
    ...     )
    Traceback (most recent call last):
        ...
    TypeError: Expected 3 attributes, got 2 ({...}) in row 0
    >>> IsEnrolledOn(                               # doctest: +ELLIPSIS
    ...     {'student_id': SID('S1'), 'course_id': CID('C1'), 'name': 'Anne'},
    ...     {'student_id': SID('S1'), 'name': 'Anne'},
    ...     )
    Traceback (most recent call last):
        ...
    TypeError: Expected 3 attributes, got 2 ({...}) in row 1
    >>> IsEnrolledOn(                               # doctest: +ELLIPSIS
    ...     {'student_id': SID('S1'), 'course_id': CID('C1'), 'name': 'Anne'},
    ...     {'student_id': SID('S1'), 'course_id': CID('C1'), 'name': 'Anne', 'foo': 'bar'},
    ...     )
    Traceback (most recent call last):
        ...
    TypeError: Expected 3 attributes, got 4 ({...}) in row 1

All of these examples require repeating the attribute names in every row,
though.  dinsd provides an additional specialized shorthand for creating
a relation instance that eliminates this duplication::

    >>> x = IsEnrolledOn(
    ...     ('student_id', 'course_id', 'name'),
    ...     ('S1',          'C1',       'Anne'),
    ...     ('S1',          'C2',       'Anne'),
    ...     ('S2',          'C1',       'Boris'),
    ...     ('S3',          'C3',       'Cindy'),
    ...     ('S4',          'C1',       'Devinder'),
    ...     )
    >>> x == is_enrolled_on
    True

In this form, which is unique to the relation constructor and is intended only
for use in providing the body of a relation in literal form, the input must be
a sequence of arguments, the fist one providing an ordering for the
attributes, and the remaining arguments providing lists of attribute values in
that same order.

There is no loss of type-safety here, since each tuple is checked for
conformance to the relation's type, and each value  is passed through the
corresponding attribute's type constructor::

    >>> IsEnrolledOn(               # doctest: +NORMALIZE_WHITESPACE
    ...     ('student_id', 'course_id', 'foo'),
    ...     ('S1',          'C1',       'Anne'),
    ...     )
    Traceback (most recent call last):
        ...
    AttributeError: <class 'dinsd.rel({'course_id': CID, 'name': str,
        'student_id': SID})'> has no attribute 'foo'
    >>> IsEnrolledOn(               # doctest: +NORMALIZE_WHITESPACE
    ...     ('student_id', 'course_id', 'name'),
    ...     ('S1',          'C1',       'Anne', 'foo'),
    ...     )
    Traceback (most recent call last):
        ...
    TypeError: Expected 3 attributes, got 4 in row 1 for <class
        'dinsd.rel({'course_id': CID, 'name': str, 'student_id': SID})'>
    >>> IsEnrolledOn(               # doctest: +NORMALIZE_WHITESPACE
    ...     ('student_id', 'course_id', 'name'),
    ...     ('C1',          'C1',       'Anne'),
    ...     )
    Traceback (most recent call last):
        ...
    TypeError: Expected 'S' followed by one to three digits, got 'C1'; 'C1'
        invalid for attribute student_id in row 1

We can explain now why it is critical that the type constructor accept an
instance of itself as valid:  the relation constructor will always pass the
value of an attribute through the corresponding type function for validation,
and an instance of that type must be reported as valid.

It might appear as though there is another step of simplification we could do:
not require the first tuple of attribute names, but instead assume the same
ordering as that used in the relation definition.  Although it would
technically be possible to do this in Python (by using an ``OrderedDict`` for
the Relation class dictionary) it is not a natural fit for normal Python
semantics, and would (unlike, I believe, the previous simplifications)
definitely violate the spirit of TTM by making the meaning of a relation
literal dependent on the order of definition of the attributes in the relation
definition.  In contrast, the final simplification above is not dependent on
the definition order, only on the types, which cannot change.

We could also choose to accept rows in the sorted order of the attributes, but
the value (and usefulness) of specifying the order of the tuples in the
literal representation is too valuable to give up for the small reduction in
additional typing.

By the way, as with ``rel``, rows and dictionaries can be mixed::

    >>> x = IsEnrolledOn(
    ...     IsEnrolledOn.row({'student_id': SID('S1'), 'course_id': CID('C1'),
    ...          'name': 'Anne'}),
    ...     {'student_id': SID('S1'), 'course_id': CID('C2'), 'name': 'Anne'},
    ...     )
    >>> y = IsEnrolledOn(
    ...     {'student_id': SID('S1'), 'course_id': CID('C2'), 'name': 'Anne'},
    ...     IsEnrolledOn.row({'student_id': SID('S1'), 'course_id': CID('C1'),
    ...          'name': 'Anne'}),
    ...     )
    >>> x == y
    True


One Obvious Way To Do It?
~~~~~~~~~~~~~~~~~~~~~~~~~

The Python language is designed around a set of philosophical principles.
(Well, actually, the set of philosophical principles was derived from the
original language design, but then it became a guide.) ::

    >>> import this
    The Zen of Python, by Tim Peters
    <BLANKLINE>
    Beautiful is better than ugly.
    Explicit is better than implicit.
    Simple is better than complex.
    Complex is better than complicated.
    Flat is better than nested.
    Sparse is better than dense.
    Readability counts.
    Special cases aren't special enough to break the rules.
    Although practicality beats purity.
    Errors should never pass silently.
    Unless explicitly silenced.
    In the face of ambiguity, refuse the temptation to guess.
    There should be one-- and preferably only one --obvious way to do it.
    Although that way may not be obvious at first unless you're Dutch.
    Now is better than never.
    Although never is often better than *right* now.
    If the implementation is hard to explain, it's a bad idea.
    If the implementation is easy to explain, it may be a good idea.
    Namespaces are one honking great idea -- let's do more of those!

This is a Zen particularly because there are not right answers, and the "best"
design is an aesthetic balance between the principles, many of which pull
in opposite directions.

There is also an additional principle that I was surprised not to find in
the above list when I checked, the principle of least surprise.  This
principle is covered in part in AIRTD, in its discussion of the fact
that a programming language should be self consistent.  If something is
surprising, it probably indicates a questionable design decision.

You can see from the previous sections that in dinsd there are several ways
to create row and relation definitions, and several ways to extend a
relation definition into a relation instance.  So, is one of the ways
to do each of these things the one obvious way?

Perhaps.  The API resulted from following the consequences of the principle of
least surprise/self consistency in regard to the decision that (a) the obvious
representation of a row (*Tutorial D* tuple) in Python is a dictionary and the
obvious representation of a relation body is a set and (b) it is desirable
have a notation that can detect duplicate values.  Adding (b) is also
motivated by the fact that the resulting notation is more compact, and the
practicality/economy of expression principles lead us to also provide a
further simplified notation for literal representation of relation instances.

So: using dictionary/set notation is provided mostly because it would be
surprising if it didn't work.  But the one (mostly obvious) way to do it
should be the following:

Defining a relation type::

    >>> Foo = rel(foo=int, bar=SID)

Row literals::

    >>> r = row(foo=1, bar=SID('S1'))

Extending a relation type into an instance::

    >>> foo = Foo(('foo', 'bar'), (1, 'S1'), (2, 'S2'))

A single row relational literal::

    >>> baz = rel(row(baz='test', foobar=2))

A multi-row relational literal::

    >>> baz2 = rel(baz=str, foobar=int)(
    ...             ('baz',  'foobar'),
    ...             ('test',  1),
    ...             ('test2', 7),
    ...             ('test3', 42),
    ...             )



Relational Operators
--------------------

Example Relations
~~~~~~~~~~~~~~~~~

Here are the two relations used in the examples in Chapter 4 of AIRDT::

    >>> IsCalled = rel(student_id=SID, name=str)
    >>> is_called = IsCalled(
    ...     ('student_id',  'name'),
    ...     ('S1',          'Anne'),
    ...     ('S2',          'Boris'),
    ...     ('S3',          'Cindy'),
    ...     ('S4',          'Devinder'),
    ...     ('S5',          'Boris'),
    ...     )
    >>> print(is_called)
    +----------+------------+
    | name     | student_id |
    +----------+------------+
    | Anne     | S1         |
    | Boris    | S2         |
    | Boris    | S5         |
    | Cindy    | S3         |
    | Devinder | S4         |
    +----------+------------+
    >>> IsEnrolledOn = rel(student_id=SID, course_id=CID)
    >>> is_enrolled_on = IsEnrolledOn(
    ...     ('student_id',  'course_id'),
    ...     ('S1',          'C1'),
    ...     ('S1',          'C2'),
    ...     ('S2',          'C1'),
    ...     ('S3',          'C3'),
    ...     ('S4',          'C1'),
    ...     )
    >>> print(is_enrolled_on)
    +-----------+------------+
    | course_id | student_id |
    +-----------+------------+
    | C1        | S1         |
    | C1        | S2         |
    | C1        | S4         |
    | C2        | S1         |
    | C3        | S3         |
    +-----------+------------+

Here we have changed the meaning of the names IsEnrolledOn and is_enrolled_on.
We would not be allowed to do this in *Tutorial D* or *TTM* without first
declaring variables as discarded and *then* defining new types for them.
Python sees that as pointless busywork and does not require it, but it is one
of the things that makes dinsd not TTM compliant.  (We will see later,
however, that disd *does* implement semi-statically-typed relation
variables inside databases.)

Python is *strongly typed*, even though it is also *dynamically typed*.  If
you consider that it is really the object that is typed, and that there are no
real 'variables' in Python, just names that contain references to objects,
disnd might be considered to technically comply with TTM...but not with its
spirit, which requires static typing.


display
~~~~~~~

``display`` is not a relational algebra function, but we introduce it here
because it is useful in the examples, and there is no corresponding
concept in AIRDT. ::

    >>> from dinsd import display

The value returned by ``display`` is very similar to the value returned
by turning a relation in to a string, except that we can control the
order of the columns in the resulting table display.  Using ``display``
we can print the relations in the same column order that is used in AIRDT::

    >>> print(display(is_called, 'student_id', 'name'))
    +------------+----------+
    | student_id | name     |
    +------------+----------+
    | S1         | Anne     |
    | S2         | Boris    |
    | S3         | Cindy    |
    | S4         | Devinder |
    | S5         | Boris    |
    +------------+----------+
    >>> print(display(is_enrolled_on, 'student_id', 'course_id'))
    +------------+-----------+
    | student_id | course_id |
    +------------+-----------+
    | S1         | C1        |
    | S1         | C2        |
    | S2         | C1        |
    | S3         | C3        |
    | S4         | C1        |
    +------------+-----------+

Since ``display`` can only ever apply to one relation, it is available
as a method on relations::

    >>> print(is_called.display('student_id', 'name'))
    +------------+----------+
    | student_id | name     |
    +------------+----------+
    | S1         | Anne     |
    | S2         | Boris    |
    | S3         | Cindy    |
    | S4         | Devinder |
    | S5         | Boris    |
    +------------+----------+

We can also choose a sort order that is different from sorting by the
columns in the ordered displayed::

    >>> print(is_called.display('student_id', 'name',
    ...          sort=('name', 'student_id')))
    +------------+----------+
    | student_id | name     |
    +------------+----------+
    | S1         | Anne     |
    | S2         | Boris    |
    | S5         | Boris    |
    | S3         | Cindy    |
    | S4         | Devinder |
    +------------+----------+


join
~~~~

    >>> from dinsd import join

In *Tutorial D*, a join operation looks like this::

    IS_CALLED JOIN IS_ENROLLED_ON

In Python we can't define new infix operators, but we can overload existing
ones.  Since ``JOIN`` is, at base, the logical ``and`` operator, it would make
sense to override ``and`` for the infix version of join.  However, we can't do
that in Python, because ``and`` is a short circuit operator, whereas both
arguments to a function must be evaluated before the function can be called,
and we can only override an operator by defining a function.

What we can do, however, is override the arithmetic ``&`` operator.  So the
dinsd equivalent of the *Tutorial D* expression above is:

    >>> enrollment = is_enrolled_on & is_called

and produces the table from figure 4.2 (page 89 in my copy of AIRDT):

    >>> print(enrollment.display('student_id', 'name', 'course_id'))
    +------------+----------+-----------+
    | student_id | name     | course_id |
    +------------+----------+-----------+
    | S1         | Anne     | C1        |
    | S1         | Anne     | C2        |
    | S2         | Boris    | C1        |
    | S3         | Cindy    | C3        |
    | S4         | Devinder | C1        |
    +------------+----------+-----------+

In *Tutorial D* the ``JOIN`` relational operator can be used both as a prefix
function and as a infix operator::

    JOIN { r1, r2, ... }

(In dinsd, there is always a prefix (functional) form for every operator, but
only sometimes an infix form...because there are a limited number of operators
that we can overload.)

The prefix form of ``join`` in dinsd is::

    >>> j2 = join(is_enrolled_on, is_called)
    >>> print(j2.display('student_id', 'name', 'course_id'))
    +------------+----------+-----------+
    | student_id | name     | course_id |
    +------------+----------+-----------+
    | S1         | Anne     | C1        |
    | S1         | Anne     | C2        |
    | S2         | Boris    | C1        |
    | S3         | Cindy    | C3        |
    | S4         | Devinder | C1        |
    +------------+----------+-----------+
    >>> enrollment == j2
    True

On page 93 there is a discussion of cases where we *can't* perform a join.  In
particular, if two tables have columns with the same name but different types,
we cannot join them::

    >>> permissive_is_called = rel(student_id=str, name=str)(
    ...     ('student_id', 'name'),
    ...     ('S1', 'Anne'),
    ...     ('S2', 'Boris'),
    ...     )
    >>> permissive_is_called & is_enrolled_on   # doctest: +NORMALIZE_WHITESPACE
    Traceback (most recent call last):
        ...
    TypeError: Duplicate attribute name ('student_id') with different type
        (first: <class 'str'>, second: <class '__main__.SID'> found in joined
        relations with types <class 'dinsd.rel({'name': str, 'student_id':
        str})'> and <class 'dinsd.rel({'course_id': CID, 'student_id': SID})'>

The prefix form of ``join`` can take more than two arguments, joining all of
the relations so specified::

    >>> x = join(is_enrolled_on, is_called, rel(row(student_id=SID('S1'))))
    >>> print(x.display('student_id', 'name', 'course_id'))
    +------------+------+-----------+
    | student_id | name | course_id |
    +------------+------+-----------+
    | S1         | Anne | C1        |
    | S1         | Anne | C2        |
    +------------+------+-----------+

A mismatched header error in a multi-join also indicates in which argument the
error was detected::

    >>> join(is_enrolled_on, is_called, permissive_is_called)
    ...
    ... # doctest: +NORMALIZE_WHITESPACE
    Traceback (most recent call last):
        ...
    TypeError: Duplicate attribute name ('student_id') with different type
        (first: <class '__main__.SID'>, second: <class 'str'> found in joined
        relations with types <class 'dinsd.rel({'course_id': CID, 'name': str,
        'student_id': SID})'> and <class 'dinsd.rel({'name': str,
        'student_id': str})'> (error detected while processing argument 2)

If we join with an empty relation, we get an empty relation::

    >>> print(IsCalled() & is_enrolled_on)
    +-----------+------+------------+
    | course_id | name | student_id |
    +-----------+------+------------+
    +-----------+------+------------+

Join is idempotent: if we join a relation to itself, or if we join a relation
to the result of a previous join involving that relation, we get back the
original join::

    >>> j = is_enrolled_on & is_called
    >>> j == j & j
    True
    >>> j == j & is_called
    True

Join is commutative::

    >>> is_enrolled_on & is_called == is_called & is_enrolled_on
    True

Join is associative::

    >>> last_year = IsEnrolledOn(
    ...         ('student_id', 'course_id'),
    ...         ('S1',         'C3'),
    ...         ('S5',         'C3'),
    ...         ('S3',         'C3'),
    ...         )
    >>> (is_enrolled_on & is_called) & last_year == (
    ...     is_enrolled_on & (is_called & last_year))
    True

Given the above, a monadic join simply returns the argument
relation::

    >>> join(last_year) == last_year
    True

While an empty join gets us an empty relation with no attributes and one row
with no attributes::

    >>> join()
    rel({row({})})


intersect
~~~~~~~~~

    >>> from dinsd import intersect

This is a special case of ``join``: the case where all of the attributes of
the two relations being joined are the same.  It is equivalent to the
set-intersection of the two relations::

    >>> repeat_enrollment = intersect(is_enrolled_on, last_year)
    >>> print(repeat_enrollment.display('student_id', 'course_id'))
    +------------+-----------+
    | student_id | course_id |
    +------------+-----------+
    | S3         | C3        |
    +------------+-----------+
    >>> repeat_enrollment == is_enrolled_on & last_year
    True

There are two reasons to use ``intersect`` instead of ``join``.  The first is
that because the domain of operation is constrained, the implementation may be
faster.  The better reason to use it is that it declares your intention to
join two sets with the same attributes, and will raise an error if the two
sets do not have the same attributes::

    >>> intersect(is_called, is_enrolled_on)
    Traceback (most recent call last):
        ...
    TypeError: Cannot take intersection of unlike relations

Unlike *Tutorial D*, dinsd does not have an infix form of this operator.

Like ``join``, ``intersect`` may be called with more than two arguments::

    >>> two_years_ago = IsEnrolledOn()
    >>> print(intersect(is_enrolled_on, last_year, two_years_ago))
    +-----------+------------+
    | course_id | student_id |
    +-----------+------------+
    +-----------+------------+


times
~~~~~

    >>> from dinsd import times

``times`` is another special case of ``join``.  In this case, there
are *no* attributes in common between the two relations, and the result of the
join is the Cartesian product of the two sets of values.

Use cases for this are not common, so we won't even try to come up with a
sensible example::

    >>> Foo = rel(bar=str)
    >>> foo = Foo(('bar',), ('fizz',), ('gin',))
    >>> print(times(last_year, foo))
    +------+-----------+------------+
    | bar  | course_id | student_id |
    +------+-----------+------------+
    | fizz | C3        | S1         |
    | fizz | C3        | S3         |
    | fizz | C3        | S5         |
    | gin  | C3        | S1         |
    | gin  | C3        | S3         |
    | gin  | C3        | S5         |
    +------+-----------+------------+

Here there is not likely to be any performance gain by using ``times``, but
the feature of getting an error if you try to ``times`` relations that share
columns is probably even more valuable. ::

    >>> times(is_called, last_year)
    Traceback (most recent call last):
        ...
    TypeError: Cannot multiply relations that share attributes

There is no infix form of ``times``, though we may choose to assign it to
``*`` at some point.

Like ``join`` and ``intersect``, ``times`` may be called with more than two
relations::

    >>> bar = rel(foo=str)(('foo',), ('gin',), ('fiz',))
    >>> print(times(last_year, bar, foo))
    +------+-----------+-----+------------+
    | bar  | course_id | foo | student_id |
    +------+-----------+-----+------------+
    | fizz | C3        | fiz | S1         |
    | fizz | C3        | fiz | S3         |
    | fizz | C3        | fiz | S5         |
    | fizz | C3        | gin | S1         |
    | fizz | C3        | gin | S3         |
    | fizz | C3        | gin | S5         |
    | gin  | C3        | fiz | S1         |
    | gin  | C3        | fiz | S3         |
    | gin  | C3        | fiz | S5         |
    | gin  | C3        | gin | S1         |
    | gin  | C3        | gin | S3         |
    | gin  | C3        | gin | S5         |
    +------+-----------+-----+------------+


rename
~~~~~~

This is operator is not one discussed in the original literature on the
relational algebra (according to AIRDT), but is needed for certain logical
operations.  It allows us to construct a new relation that is identical to an
existing relation except that one or more of the attributes have different
names.  This is required in order to perform logical operations on relations
where we need to *not* treat the attributes as the same, even though they have
the same data in them (we'll see an example of this below).

In *Tutorial D* (example 4.2) this is written::

    IS_CALLED RENAME ( StudentId AS Sid )

Unlike the ``join`` operators, ``rename`` is a function of a single relation.
In Python, the natural notation for this is the postfix object method
invocation syntax::

    >>> r = is_called.rename(student_id='sid')
    >>> print(r.display('sid', 'name'))
    +-----+----------+
    | sid | name     |
    +-----+----------+
    | S1  | Anne     |
    | S2  | Boris    |
    | S3  | Cindy    |
    | S4  | Devinder |
    | S5  | Boris    |
    +-----+----------+

This gives us the equivalent result as that shown in figure 4.4 in AIRDT.

As is standard for dinsd, there is also a prefix form::

    >>> from dinsd import rename
    >>> r == rename(is_called, student_id='sid')
    True

We can also rename multiple attributes::

    >>> r2 = is_called.rename(student_id='sid', name='called')
    >>> print(r2.display('sid', 'called'))
    +-----+----------+
    | sid | called   |
    +-----+----------+
    | S1  | Anne     |
    | S2  | Boris    |
    | S3  | Cindy    |
    | S4  | Devinder |
    | S5  | Boris    |
    +-----+----------+

We support the revised semantics of *Tutorial D* that allows attribute names
to be swapped in one call::

    >>> rswap = is_called.rename(student_id='name', name='student_id')
    >>> print(rswap)
    +------+------------+
    | name | student_id |
    +------+------------+
    | S1   | Anne       |
    | S2   | Boris      |
    | S3   | Cindy      |
    | S4   | Devinder   |
    | S5   | Boris      |
    +------+------------+

This is the natural interpretation in Python: even though we don't use the
``{}`` notation above to indicate that the renames are an unordered set, the
Python keyword semantics are that of a set.  And in fact Python allows us to
use the dictionary notation, which makes this explicit::

    >>> rswap == rename(is_called, **{'student_id': 'name',
    ...                               'name': 'student_id'})
    True

Rename is a read-only operation, it does not modify the source relation::

    >>> r  == is_called
    False
    >>> r2 == is_called
    False
    >>> print(is_called.display('student_id', 'name'))
    +------------+----------+
    | student_id | name     |
    +------------+----------+
    | S1         | Anne     |
    | S2         | Boris    |
    | S3         | Cindy    |
    | S4         | Devinder |
    | S5         | Boris    |
    +------------+----------+

Example 4.3 on page 98 of AIRDT uses ``rename`` and ``join`` to discover all
pairs of students that share the same name::

    >>> shared_name = join(is_called.rename(student_id='sid1'),
    ...                    is_called.rename(student_id='sid2'))
    >>> print(shared_name.display('sid1', 'name', 'sid2'))
    +------+----------+------+
    | sid1 | name     | sid2 |
    +------+----------+------+
    | S1   | Anne     | S1   |
    | S2   | Boris    | S2   |
    | S2   | Boris    | S5   |
    | S3   | Cindy    | S3   |
    | S4   | Devinder | S4   |
    | S5   | Boris    | S2   |
    | S5   | Boris    | S5   |
    +------+----------+------+

This table doesn't quite look like the one on page 98.  We can fix that using
the 'sort' keyword of ``display``::

    >>> print(shared_name.display('sid1', 'name', 'sid2',
    ...                 sort=('name', 'sid1', 'sid2')))
    +------+----------+------+
    | sid1 | name     | sid2 |
    +------+----------+------+
    | S1   | Anne     | S1   |
    | S2   | Boris    | S2   |
    | S2   | Boris    | S5   |
    | S5   | Boris    | S2   |
    | S5   | Boris    | S5   |
    | S3   | Cindy    | S3   |
    | S4   | Devinder | S4   |
    +------+----------+------+

We can't rename a non-existent attribute::

    >>> is_called.rename(fred='called')
    Traceback (most recent call last):
        ...
    KeyError: 'fred'

Specifying a name as the target of more than one attribute rename is also an
error::

    >>> is_called.rename(student_id='foo', name='foo')
    Traceback (most recent call last):
        ...
    ValueError: Duplicate relational attribute name 'foo'

Rename with no attributes renamed returns a relation equal to the relation on
which rename is called::

    >>> is_called.rename() == is_called
    True


project
~~~~~~~

``project`` takes as its argument a set of attribute names, and returns a
relation containing just those attributes (and the corresponding values).

This is not an explicit named operator in *Tutorial D*, but an implicit one
arising from a particular syntax.  From example 4.4::

    IS_ENROLLED_ON { StudentId }

For the dinsd version of this we use overload the ``>>`` operator::

    >>> sids = is_enrolled_on >> {'student_id'}
    >>> print(sids)
    +------------+
    | student_id |
    +------------+
    | S1         |
    | S2         |
    | S3         |
    | S4         |
    +------------+

There is, as is standard for dinsd, a functional version of this::

    >>> from dinsd import project
    >>> sids == project(is_enrolled_on, {'student_id'})
    True

It is often more convenient to list the attribute we want to drop instead
of the attributes we want to keep.  *Tutorial D* does this by prefixing 
the keyword ``ALL BUT`` to the list of attribute names::

    IS_ENROLLED_ON { ALL BUT CourseId }

Since ``is_enrolled_on`` only has two columns, the above is equivalent to the
projection on to ``student_id``.

dinsd's equivalent to ``ALL BUT`` is a functional wrapper (that is, a function
that returns a lazy result that can later be used to compute the inverse of
the specified set of attribute names)::

    >>> from dinsd import all_but
    >>> sids == project(is_enrolled_on, all_but({'course_id'}))
    True

We could also write this as::

    >>> sids == is_enrolled_on >> all_but({'course_id'})
    True

It is a common operation, though, so dinsd also overloads the ``<<`` operator
to mean "all but"::

    >>> sids == is_enrolled_on << {'course_id'}
    True

Inverting an inversion is of questionable utility, but it is legal::

    >>> sids == is_enrolled_on << all_but({'student_id'})
    True

In AIRDT, figure 4.7 on page 101, the name matching expression is improved
using project as follows::

    ( ( IS_CALLED RENAME ( StudentId AS Sid1 ) ) JOIN
      ( IS_CALLED RENAME ( StudentId AS Sid2 ) ) ) { ALL BUT Name }

In dinsd, this becomes::

    >>> same_name = (is_called.rename(student_id='sid1') &
    ...              is_called.rename(student_id='sid2')) << {'name'}
    >>> print(same_name)
    +------+------+
    | sid1 | sid2 |
    +------+------+
    | S1   | S1   |
    | S2   | S2   |
    | S2   | S5   |
    | S3   | S3   |
    | S4   | S4   |
    | S5   | S2   |
    | S5   | S5   |
    +------+------+

(There is no way to get that table to exactly match the one from the book,
since the order from the book appears to be some artifact of the *Tutorial D*
implementation.)

Note the need for parenthesis round the join expression in both the *Tutorial
D* version and the dinsd version: the projection operator (``>>``) has a
higher precedence than the join operator (``&``), and since we want to project
the result of the join, we need to enclose the join expression in parenthesis.
(In dinsd this operator precedence is fortuitous, since although we can
overload operators in Python we cannot change their relative precedence.)

An ``all_but`` projection of the empty set produces an identical relation::

    >>> same_name << {} == same_name
    True

In dinsd this is explicitly *not* the same object that was passed in to
the operation::

    >>> same_name << {} is same_name
    False

Aside: we are being a bit loose with our Python data types here in order to
make this notation consistent.  This may or may not be a bad idea, but it
does seem natural.  Technically, ``{}`` is an empty *dictionary* not an
empty set.  Python's literal notation for an empty set is ``set()``.  But
we can easily interpret an empty dictionary as an empty set *in this
context*, and so we do.  Using ``set()`` will of course produce the same
result, if you prefer to keep your data types straight::

    >>> same_name << set() == same_name
    True

Obversely, projecting to the empty set yields a relation of no attributes
having a single row with no attributes. ::

    >>> same_name >> {}
    rel({row({})})

Trying to project using a name that is not a valid attribute is an error::

    >>> shared_name << {'foo'}
    Traceback (most recent call last):
        ...
    TypeError: Attribute list included unknown attributes: {'foo'}
    >>> shared_name >> {'foo'}
    Traceback (most recent call last):
        ...
    TypeError: Attribute list included unknown attributes: {'foo'}

On page 102 of AIRDT example 4.5 demonstrates using projection to split
the original enrollment relation into ``is_enrolled_on`` and ``is_called``
in *Tutorial D*::

    VAR IS_CALLED BASE
    INIT (ENROLMENT { StudentId, Name })
    KEY { StudentId } ;
    VAR IS_ENROLLED_ON BASE
    INIT (ENROLMENT { StudentId, CourseId })
    KEY { StudentId, CourseId } ;
    DROP VAR ENROLMENT ;

in dinsd, that looks like this::

    >>> is_enrolled_on_split = enrollment >> {'student_id', 'course_id'}
    >>> is_called_split = enrollment >> {'student_id', 'name'}
    >>> del enrollment

That is, whereas in *Tutorial D* you must go through the process of declaring
the variables and initializing them, in dinsd you just do the projection and
keep a pointer to it in the name of your choice.  This comparison isn't fair,
though, because I'm ignoring the issue of persisting the database here, which
will add a few characters and one additional statement when we get to it.  I
think *Tutorial D* does the persistence automatically.  But more, we haven't
dealt with constraints at all yet, and establishing those for a table is a bit
more complicated. ::

    >>> print(is_enrolled_on_split.display('student_id', 'course_id'))
    +------------+-----------+
    | student_id | course_id |
    +------------+-----------+
    | S1         | C1        |
    | S1         | C2        |
    | S2         | C1        |
    | S3         | C3        |
    | S4         | C1        |
    +------------+-----------+
    >>> print(is_called_split.display('student_id', 'name'))
    +------------+----------+
    | student_id | name     |
    +------------+----------+
    | S1         | Anne     |
    | S2         | Boris    |
    | S3         | Cindy    |
    | S4         | Devinder |
    +------------+----------+

``is_enrolled_on_split`` is identical to the original ``is_enrolled_on``
relation, but ``is_called`` is not, since the ``enrollment`` relation did not
include any entry for the student who was to enrolled in any courses::

    >>> is_enrolled_on_split == is_enrolled_on
    True
    >>> is_called_split == is_called
    False
    >>> len(is_called_split) == len(is_called) - 1
    True


Dum and Dee
~~~~~~~~~~~

Two relations have special meaning and are given names in *Tutorial D*:
``TABLE_DUM`` and ``TABLE_DEE``.  Because these names are a bit awkward (and
why are these called ``tables`` when everything else is a relation?), standard
practice has quickly become to refer to them as ``Dum`` (or ``DUM``) and
``Dee`` (``DEE``).  We prefer the dual case versions, in analogy to the Python
capitalization for the analogous logical constants``True`` and ``False``.

In *Tutorial D*, the literal for ``TABLE_DUM``, which is the relation with
no attributes and no rows, is::

    RELATION { } { }

and ``TABLE_DEE``, which is the relation with no attributes and one row is::

    RELATION { TUPLE { } }

We've already seen the dinsd equivalents of these, but now we note that they
are also defined as named constants::

    >>> from dinsd import Dum, Dee
    >>> Dum
    rel({})()
    >>> print(Dum)
    ++
    ||
    ++
    ++
    >>> Dee
    rel({row({})})
    >>> print(Dee)
    ++
    ||
    ++
    ||
    ++

These, especially ``Dee``, should look very familiar, as we have encountered
their reprs previous, in the results of edge cases for various operators::

    >>> is_called >> {} == Dee
    True
    >>> join() == intersect() == times() == Dee
    True

The boolean value of ``Dum`` is ``False``, while that of ``Dee`` is ``True``::

    >>> bool(Dum)
    False
    >>> bool(Dee)
    True

This follows both from the relational logic and from Python's own conventions:
``Dum`` has no rows (it is empty, and like an empty list it is ``False``),
while ``Dee`` has one row, and being non-empty is ``True``.

Note that unlike Python's ``True`` and ``False``, which are singletons, there
may be other relation instances equivalent to ``Dum`` and ``Dee`` besides the
two defined as constants.  Conceptually they are the "same" relation, but in
Python object terms they may be different objects::

    >>> is_called >> {} is Dee
    False

It is best not to depend on this fact, though, since a future optimization
might make ``Dum`` and ``Dee`` singletons.

As per the discussion on page 104 of AIRDT, ``Dee`` is an identity value for
the ``join`` operator.  And, again, in dinsd what is produced is an equivalent
relation, not an identity at the object level::

    >>> is_called & Dee == is_called
    True
    >>> is_called & Dee is is_called
    False



where
~~~~~

Example 4.6 on page 105 at the start of section 4.7 shows how to select
rows from a relation using join and projection::

    ( IS_CALLED JOIN RELATION { TUPLE { Name NAME ( 'Boris' ) } } )
    { StudentId }

In dinsd we write this::

    >>> borises = (is_called & rel(row(name='Boris'))) >> {'student_id'}
    >>> print(borises)
    +------------+
    | student_id |
    +------------+
    | S2         |
    | S5         |
    +------------+

Example 4.7 shows the same computation using the ``where`` operator::

    ( IS_CALLED WHERE Name = NAME ( 'Boris' ) )

``where``, like projection, is a function of one relation, so in dinsd it
is a method of relation objects::

    >>> b2 = is_called.where(lambda row: row.name == 'Boris')
    >>> print(b2)
    +-------+------------+
    | name  | student_id |
    +-------+------------+
    | Boris | S2         |
    | Boris | S5         |
    +-------+------------+

Here we haven't projected away the ``name`` attribute, we're just looking
at the join.

The above expression is not shorter than the formulation using a simple join.
However, ``where`` takes an arbitrary function of one argument as its
parameter (called the condition), which means that for a more complex join it
is much easier to express the join as a ``where`` expression than as a join.
We'll see an example of that in a moment.

The ``lambda`` expression we used as the condition in the example above is a
Python facility for dynamically defining a function.  A condition function is
passed a single argument consisting of a row object.  In this expression we
access the ``name`` attribute of the row, and check if it is equal to the
string ``Boris``.  Since our ``name`` attribute is string valued, this
expression will be ``True`` if and only if the row in question has an ``name``
attribute of ``Boris``.  All such rows are included in the returned relation,
and no rows that fail the test are included in the returned relation.

In addition to this fully general method for specifying a condition, dinsd
provides a less general but also less wordy method::

    >>> b2 == is_called.where("name == 'Boris'")
    True

This is just as compact as the join version.  What is happening here is that a
string-valued condition is *evaluated* in a context in which all the
attributes of the row are available as names in the evaluation namespace.
Thus, as in *Tutorial D*, if we refer to ``name``, it automatically refers to
the ``name`` attribute of the row currently being tested by the expression.

In addition to the brevity advantage, string valued expressions (and there
will be a number of additional instances of them) are more likely to be
optimized by the back end.  So although it doesn't change the meaning of the
program, it is better to use string valued expressions where possible.

IMPORTANT CAVEAT: string expressions may be ``eval``ed.  This means that you
should *never* construct a string-valued expression programmatically in order
to get values (that might have come from outside input) into the expression.
Later we'll see how to get such values into a string valued expression safely.

There is also a prefix version of `where``, though there is seldom reason
to use it::

    >>> from dinsd import where
    >>> b2 == where(is_called, "name == 'Boris'")
    True

The place where using ``join`` for restriction breaks down is when the
number of values we'd have to put in the joined relation is too large
to be practical, or may even be an infinite set.  The example from AIRDT
of this is example 4.8::

    IS_CALLED WHERE STARTS_WITH(THE_C(Name), 'B')

Enumerating all possible names that start with 'B' is not possible, but
a programmatic test is.

Here is example 4.8 written in dinsd::

    >>> print(is_called.where("name.startswith('B')"))
    +-------+------------+
    | name  | student_id |
    +-------+------------+
    | Boris | S2         |
    | Boris | S5         |
    +-------+------------+

Here we are using the Python ``startswith`` method of string objects,
which is analogous to the ``Rel`` ``STARTS_WITH`` function.

With Python, it is easy to make this a bit more interesting::

    >>> print(where(is_called, "name.startswith(('B', 'A'))"))
    +-------+------------+
    | name  | student_id |
    +-------+------------+
    | Anne  | S1         |
    | Boris | S2         |
    | Boris | S5         |
    +-------+------------+

Here are the edge cases from the bottom of page 106::

    >>> is_called.where("True") == is_called
    True
    >>> is_called.where("False") == type(is_called)()
    True

Using ``where`` we can further improve the problem of finding just those student_ids
that designate students with the same name (figure 4.8 on page 107)::

    ( ( IS_CALLED RENAME ( StudentId AS Sid1 ) )
      JOIN
      ( IS_CALLED RENAME ( StudentId AS Sid2 ) )
    WHERE NOT (Sid1 = Sid2) ) { Sid1, Sid2 }

In dinsd::

    >>> x = (is_called.rename(student_id='sid1') &
    ...      is_called.rename(student_id='sid2')).where(
    ...             "sid1 != sid2") >> {'sid1', 'sid2'}
    >>> print(x)
    +------+------+
    | sid1 | sid2 |
    +------+------+
    | S2   | S5   |
    | S5   | S2   |
    +------+------+

Our ``SID`` class is based on dinsd's ``Scaler`` class, it does
indeed have a total ordering, and thus we can write our version
of::

    ( ( IS_CALLED RENAME ( StudentId AS Sid1 ) )
      JOIN
      ( IS_CALLED RENAME ( StudentId AS Sid2 ) )
    WHERE NOT (Sid1 < Sid2) ) { Sid1, Sid2 }

as::
    
    >>> x = (is_called.rename(student_id='sid1') &
    ...      is_called.rename(student_id='sid2')).where(
    ...             "sid1 < sid2") >> {'sid1', 'sid2'}
    >>> print(x)
    +------+------+
    | sid1 | sid2 |
    +------+------+
    | S2   | S5   |
    +------+------+


extend
~~~~~~

This operator allows us to create a new relation that extends an existing
relation by performing a computation on the attributes of the initial
relation, and creating a new attribute whose values are the computed values.

Figure 4.10 of AIRDT shows the result of the following *Tutorial D*
expression::

    EXTEND IS_CALLED ADD ( FirstLetter ( Name ) AS Initial )

I'm not clear on whether or not ``FirstLetter`` is an actual ``Rel`` function,
or just a theoretical function used in the example.  Python does have an
equivalent to ``FirstLetter``: ``name[0]`` refers to the first letter of the
string pointed to by ``name``.

``extend`` is again a function of a single relation, and so in dinsd it is
a method of relations::

    >>> x = is_called.extend(initial="name[0]")
    >>> print(x.display('student_id', 'name', 'initial'))
    +------------+----------+---------+
    | student_id | name     | initial |
    +------------+----------+---------+
    | S1         | Anne     | A       |
    | S2         | Boris    | B       |
    | S3         | Cindy    | C       |
    | S4         | Devinder | D       |
    | S5         | Boris    | B       |
    +------------+----------+---------+

Here dinsd requires the keyword form, where the new column name is equated
to the expression (``lambda`` or string) the evaluation of which yields
the value for the new attribute.  There can be any number of new attributes,
including none.  In the case of no attributes, we have an identity relation,
and the resulting value is equivalent to the source relation::

    >>> is_called.extend() == is_called
    True

Here is an example with more than one new attribute, and we also demonstrate
here mixing ``lambda`` expressions and string expressions::

    >>> x2 = is_called.extend(caps="name.upper()",
    ...                       hello=lambda row: "hello")
    >>> print(x2.display('student_id', 'name', 'caps', 'hello'))
    +------------+----------+----------+-------+
    | student_id | name     | caps     | hello |
    +------------+----------+----------+-------+
    | S1         | Anne     | ANNE     | hello |
    | S2         | Boris    | BORIS    | hello |
    | S3         | Cindy    | CINDY    | hello |
    | S4         | Devinder | DEVINDER | hello |
    | S5         | Boris    | BORIS    | hello |
    +------------+----------+----------+-------+

Unlike *Tutorial D*, we do not have a left to right evaluation order for
extension expressions, so extension expressions in dinsd may only refer to
attributes from the base relation, not any newly created attributes.  This is
a minor restriction, since it is simple enough (and often clearer) to write
multiple extension expressions in such a case.  It would also be possible to
relax this restriction by using (name, func) tuples or some inspect trickery
and doing the extension one attribute at a time, but this hardly seems worth
the effort for what is probably a relatively infrequent use case.  It also has
the advantage that it preserves internal consistency in dinsd: keywords
arguments in Python are an unordered set, and to break that expectation would
be surprising.

There is, as usual, a prefix version of ``extend``::

    >>> from dinsd import extend
    >>> x == extend(is_called, initial="name[0]")
    True

``extend`` can not be used to add a duplicate attribute::

    >>> is_called.extend(name="'hello'")
    Traceback (most recent call last):
        ...
    ValueError: Duplicate relational attribute name 'name'

Python's restriction on duplicate keywords keeps us from adding two new
attributes with the same name::

    >>> is_called.extend(foo="'hello'", foo="'foobar'")
    Traceback (most recent call last):
        ...
    SyntaxError: keyword argument repeated

There is one unfortunate situation in which Python's dynamic typing leaves us
in a bit of a pickle, and we have to resort to an awkward call form to work
around it.  We normally determine the type of the extended column by checking
the type of the value computed from the first row.  But what if the relation is
empty?  In a statically typed language, one can infer the type of the
computation from the computation itself.  In Python this is not reliable; we
can do it in cases where the type function called with no arguments returns
a default value of the specified type::

    >>> rel(id=int, name=str)().extend(xxx="'hello'")
    rel({'id': int, 'name': str, 'xxx': str})()

But not all types follow this strict rule, since in some cases we do not
want there to be a default value.  ``CID`` and ``SID`` are examples of this.
In this case, trying to extend the empty relation will result in an error::

    >>> empty_is_called = is_called.where('False')
    >>> x = empty_is_called.extend(xxx="'hello'", yyy=2)
    Traceback (most recent call last):
        ...
    TypeError: Cannot extend this empty relation without a prototype

Note that the full traceback includes the initial traceback that shows why the
attempt to guess the types didn't work, and the reason won't always be a type
function that raises if given no arguments.  For instance, the computation
executed by the extend clause might raise an error when applied to the default
values (for example, division by zero).

To solve this problem if you are operating on such a relation that might be
empty, you can explictly pass a "prototype" that specifies the types of the new
columns by passing ``extend`` an initial argument that is a ``rel`` with fields
of the appropriate type.  The argument ``rel`` should contain all of the
extended fields, and no others::

    >>> x = empty_is_called.extend(rel(xxx=str, yyy=int), xxx="'hello'", yyy=2)
    >>> print(x)
    +------+------------+-----+-----+
    | name | student_id | xxx | yyy |
    +------+------------+-----+-----+
    +------+------------+-----+-----+

Alternatively, of course, you can make sure your types return a default value
when called with no arguments, and this is actually the recommended solution,
though you may still need prototypes if you use extend expressions that can
raise doing computations on the default values.

Note that if a prototype is given and the relation is *not* empty, the actual
computed values must match the type given in the prototype::

    >>> is_called.extend(rel(foo=int), foo="'hello'")
    ...
    ... # doctest: +NORMALIZE_WHITESPACE
    Traceback (most recent call last):
        ...
    ValueError: invalid literal for int() with base 10: 'hello';
        'hello' invalid for attribute foo

Specifying a prototype can thus also be useful if you are a belt-and-suspenders
type and want to make sure your extend expressions are producing the type you
are intending them to produce.


union
~~~~~

In the relational algebra, ``union`` corresponds to the ``or`` logical
operator, but is restricted to being an operator on relations whose
types are equal.  Example 4.9 on page 111 shows a legal ``union``
operation in *Tutorial D*::

    ( IS_CALLED WHERE Name = NAME ('Devinder') ) { StudentId }
    UNION
    ( IS_ENROLLED_ON WHERE CourseId = CID ('C1') ) { StudentId }

dinsd, like *Tutorial D*, provides ``union`` both as a prefix function and as
an infix operator.  In dinsd the operator is ``|``.  The dinsd version of
example 4.9 is::

    >>> ex49 = (is_called.where("name == 'Devinder'") >> {'student_id'} |
    ...       is_enrolled_on.where("course_id == CID('C1')") >> {'student_id'})
    >>> print(ex49)
    +------------+
    | student_id |
    +------------+
    | S1         |
    | S2         |
    | S4         |
    +------------+

This result matches that from figure 4.11.

Using the prefix version of ``union`` it looks like this::

    >>> from dinsd import union
    >>> ex49 == union(is_called.where("name == 'Devinder'") >> {'student_id'},
    ...       is_enrolled_on.where("course_id == CID('C1')") >> {'student_id'})
    True

It is an error to try to form the union of relations of disparate types::

    >>> is_called | is_enrolled_on
    Traceback (most recent call last):
        ...
    TypeError: Union operands must of equal types

Union is commutative and associative::

    >>> x = is_called >> {'student_id'}
    >>> y = is_enrolled_on >> {'student_id'}
    >>> x | y == y | x
    True
    >>> z = rel(row(student_id=SID('S5')))
    >>> (x | y) | z == x | (y | z) == union(x, y, z)
    True

The union of a single relation, or a relation with itself, is that relation. ::

    >>> union(is_called) == is_called | is_called == is_called
    True

The union of a relation with an empty relation is also equal to the original
relation::

    >>> is_called | type(is_called)() == is_called
    True

The union of no relations is ``Dum``::

    >>> union() == Dum
    True

*Tutorial D* has the ability to return an empty union if you specify a
heading.  The closest equivalent in dinsd is to pass an empty relation defined
via ``rel``::

    >>> print(union(rel(name=str, student_id=SID)()))
    +------+------------+
    | name | student_id |
    +------+------------+
    +------+------------+

This is just the union of a single empty relation.

On the other hand, calling ``union`` with no arguments does do the obvious
thing, by returning ``Dum``::

    >>> union()
    rel({})()


minus
~~~~~

The ``minus`` operator can (only) be applied to two relations have exactly the
same columns, as was the case for ``union``.  The result is a new relation of
the same type, whose body contains those rows from the first relation that
do not occur in the second relation.

Here is the *Tutorial D* example 4.10::

    ( IS_CALLED WHERE Name = NAME ('Devinder') ) { StudentId }
    MINUS
    ( IS_ENROLLED_ON WHERE CourseId = CID ('C1') ) { StudentId }

In dinsd we have no infix operator for ``minus`` (we'll see why in a moment), so
this is written in dinsd using ``minus`` as a prefix operator::

    >>> from dinsd import minus
    >>> minus(is_called.where("name == 'Devinder'") >> {'student_id'},
    ...       is_enrolled_on.where("course_id == CID('C1')") >> {'student_id'})
    rel({'student_id': SID})()

Hmm.  That wasn't very interesting (or a very good test).  How about this one::
    
    >>> anne = rel(row(student_id=SID('S1'), name='Anne'))
    >>> print(minus(is_called, anne))
    +----------+------------+
    | name     | student_id |
    +----------+------------+
    | Boris    | S2         |
    | Boris    | S5         |
    | Cindy    | S3         |
    | Devinder | S4         |
    +----------+------------+

As indicated, ``minus`` only works for relations of like type::

    >>> minus(is_called, rel(student_id=SID)())
    Traceback (most recent call last):
        ...
    TypeError: Relation types must match for minus operation

Subtracting an empty relation yields a relation equivalent to the first
relation::

    >>> minus(is_called, type(is_called)()) == is_called
    True

Subtracting a relation from itself yields an empty relation of the same type::

    >>> minus(is_called, is_called)
    rel({'name': str, 'student_id': SID})()

``minus`` is *only* a dyadic function, so there is no call form with no
arguments or one argument or more than two arguments::

    >>> minus()                             # doctest: +NORMALIZE_WHITESPACE
    Traceback (most recent call last):
        ...
    TypeError: minus() missing 2 required positional arguments: 'first' and
        'second'
    >>> minus(is_called)
    Traceback (most recent call last):
        ...
    TypeError: minus() missing 1 required positional argument: 'second'
    >>> minus(is_called, is_called, is_called)
    Traceback (most recent call last):
        ...
    TypeError: minus() takes 2 positional arguments but 3 were given

``minus`` is not commutative::

    >>> print(minus(anne, is_called))
    +------+------------+
    | name | student_id |
    +------+------------+
    +------+------------+

Nor is it associative::

    >>> boris = rel(row(student_id=SID('S5'), name='Boris'))
    >>> print(minus(minus(is_called, anne), boris))
    +----------+------------+
    | name     | student_id |
    +----------+------------+
    | Boris    | S2         |
    | Cindy    | S3         |
    | Devinder | S4         |
    +----------+------------+
    >>> print(minus(is_called, minus(anne, boris)))
    +----------+------------+
    | name     | student_id |
    +----------+------------+
    | Boris    | S2         |
    | Boris    | S5         |
    | Cindy    | S3         |
    | Devinder | S4         |
    +----------+------------+


notmatching
~~~~~~~~~~~
A more general "subtraction" type operator is called "semidifference" or,
in *Tutorial D*, ``NOT MATCHING``.  Here we require only that all of the
attributes that the two relations have in *common* be of the same type.  We
return all rows from the first relation that do *not* have rows in the second
relation with matching values for the common attributes.

Example 4.11 from AIRDT::

    IS_CALLED NOT MATCHING IS_ENROLLED_ON

In dinsd we assign this operator (the more general of the two "subtraction"
operators) to the ``-`` infix operator.  So in dinsd we write this::

    >>> not_enrolled = is_called - is_enrolled_on
    >>> print(not_enrolled)
    +-------+------------+
    | name  | student_id |
    +-------+------------+
    | Boris | S5         |
    +-------+------------+

This is our equivalent of Figure 4.12.  Here we have only those students who
appear in ``is_called`` but do not appear in ``is_enrolled_on``...that is all
students who are not enrolled on any class.
 
There is of course also a prefix version.  Although the name is a bit more
awkward in dinsd than *Tutorial D*, we prefer the name ``notmatching`` to
``semidifference``::

    >>> from dinsd import notmatching
    >>> notmatching(is_called, is_enrolled_on) == not_enrolled
    True

Since we are using the '-' operator and the operation is also referred to as a
"semidiffernce", we will use the imprecise but convenient terminology
"subtract" for this operation.

If we subtract any relation from an empty relation we get an empty relation.
On the other hand, if we subtract an empty relation we get the original
relation (we subtract nothing). ::

    >>> type(is_called)() - is_enrolled_on == type(is_called)()
    True
    >>> is_called - type(is_enrolled_on)() == is_called
    True

If we subtract a relation that has no common attributes, then if that
dissimilar relation is empty (no matching rows) we get back the original
relation.  If, however, it is not empty, then by the relational rules *all*
the rows match, and we get back the empty relation. ::

    >>> is_called - rel(foo=str)() == is_called
    True
    >>> is_called - rel(row(foo='bar')) == type(is_called)()
    True

Given this, we can see that subtracting ``Dum`` we'll get the original
relation, and subtracting ``Dee`` we'll get an empty relation.  You can
think of this as ``Dum`` (``False``) not matching any rows, whereas
``Dee`` (``True``) matches all rows::

    >>> is_called - Dum == is_called
    True
    >>> is_called - Dee == type(is_called)()
    True

It is an error for match columns with the same name to have different types::

    >>> is_called - rel(student_id=str)()     # doctest: +NORMALIZE_WHITESPACE
    Traceback (most recent call last):
        ...
    TypeError: Duplicate attribute name ('student_id') with different type
        (first: <class '__main__.SID'>, second: <class 'str'> found in
         match relation (relation types are <class 'dinsd.rel({'name': str,
         'student_id': SID})'> and <class 'dinsd.rel({'student_id': str})'>)

Like ``minus``, ``notmatching`` is a dyadic-only operation::

    >>> notmatching()                         # doctest: +NORMALIZE_WHITESPACE
    Traceback (most recent call last):
        ...
    TypeError: notmatching() missing 2 required positional arguments: 'first'
        and 'second'
    >>> notmatching(is_called)
    Traceback (most recent call last):
        ...
    TypeError: notmatching() missing 1 required positional argument: 'second'
    >>> notmatching(is_called, is_called, is_called)
    Traceback (most recent call last):
        ...
    TypeError: notmatching() takes 2 positional arguments but 3 were given

Also like ``minus``, ``notmatching`` is neither commutative nor associative::

    >>> print(is_enrolled_on - is_called)
    +-----------+------------+
    | course_id | student_id |
    +-----------+------------+
    +-----------+------------+
    >>> print((is_called - anne) - boris)
    +----------+------------+
    | name     | student_id |
    +----------+------------+
    | Boris    | S2         |
    | Cindy    | S3         |
    | Devinder | S4         |
    +----------+------------+
    >>> print(is_called - (anne - boris))
    +----------+------------+
    | name     | student_id |
    +----------+------------+
    | Boris    | S2         |
    | Boris    | S5         |
    | Cindy    | S3         |
    | Devinder | S4         |
    +----------+------------+

From the last two examples you will observe that ``notmatching`` can be used
in place of ``minus``, and in fact this is always the case.  We provide
``minus`` only for completeness: it was defined in the original relational
algebra by E.F. Codd, and so many people working with relational algebra will
expect it to exist.


Chapter 4 Exercises
~~~~~~~~~~~~~~~~~~~

Just for fun (well, and to improve our test coverage), we'll do some of the
exercises from the end of chapter 4 in AIRDT.  There won't be much if any
explanation here, just code.

Note that we still aren't dealing with keys or other constraints yet.

Exercise 2::

    >>> from datetime import date
    >>> class date_type(date):
    ...     def __new__(cls, value):
    ...         if isinstance(value, date):
    ...             return value
    ...         return super().__new__(cls, *value)
    >>> cust = rel(c_no=str, discount=float)(
    ...         ('c_no', 'discount'),
    ...         ('0012', 10.0),
    ...         ('1001', 5.0),
    ...         ('5', 20.0),
    ...         )
    >>> orders = rel(o_no=str, c_no=str, date=date_type)(
    ...         ('o_no', 'c_no', 'date'),
    ...         ('0001', '0012', date(2012, 11, 4)),
    ...         ('0002', '0012', (2012, 11, 5)),
    ...         ('0003', '5', date(2012, 11, 3)),
    ...         )
    >>> order_items = rel(o_no=str, p_no=str, qty=int)(
    ...         ('o_no', 'p_no', 'qty'),
    ...         ('0001', '20001', 2),
    ...         ('0001', '20002', 3),
    ...         ('0002', '20001', 7),
    ...         ('0003', '2201', 20),
    ...         )
    >>> products = rel(p_no=str, unit_price=float)(
    ...         ('p_no', 'unit_price'),
    ...         ('20001', 120.50),
    ...         ('20002', 12.75),
    ...         ('2201', 1000.00),
    ...         )
    >>> x = ((order_items & products & orders & cust).extend(
    ...         price="round(qty*unit_price*(1-discount/100), 2)")
    ...             >> {'o_no', 'p_no', 'price'})
    >>> print(x.display('o_no', 'p_no', 'price'))
    +------+-------+---------+
    | o_no | p_no  | price   |
    +------+-------+---------+
    | 0001 | 20001 | 216.9   |
    | 0001 | 20002 | 34.43   |
    | 0002 | 20001 | 759.15  |
    | 0003 | 2201  | 16000.0 |
    +------+-------+---------+

Exercise 3::

    >>> exam_marks = rel(student_id=SID, course_id=CID, mark=int)(
    ...     ('student_id', 'course_id', 'mark'),
    ...     ('S1',         'C1',        85),
    ...     ('S1',         'C2',        49),
    ...     ('S1',         'C3',        85),
    ...     ('S2',         'C1',        49),
    ...     ('S3',         'C3',        66),
    ...     ('S4',         'C1',        93),
    ...     )
    >>> x = exam_marks.rename(student_id='sid1', mark='mark1')
    >>> y = exam_marks.rename(student_id='sid2', mark='mark2')
    >>> pairs = (x & y).where("sid1 < sid2")
    >>> diff = pairs.extend(gap="abs(mark1 - mark2)")
    >>> diff_only = diff << {'mark1', 'mark2'}
    >>> print(diff_only.display('course_id', 'sid1', 'sid2', 'gap'))
    +-----------+------+------+-----+
    | course_id | sid1 | sid2 | gap |
    +-----------+------+------+-----+
    | C1        | S1   | S2   | 36  |
    | C1        | S1   | S4   | 8   |
    | C1        | S2   | S4   | 44  |
    | C3        | S1   | S3   | 19  |
    +-----------+------+------+-----+

Exercise 4::

    >>> is_called - Dee
    rel({'name': str, 'student_id': SID})()
    >>> is_called - Dum == is_called
    True
    >>> is_called - is_called
    rel({'name': str, 'student_id': SID})()
    >>> (is_called - is_called) - is_called
    rel({'name': str, 'student_id': SID})()
    >>> is_called - (is_called - is_called) == is_called
    True

Oh, and for completeness: ``rename`` can be written in terms of
``extend``::

    >>> x = rename(is_called, student_id='sid')
    >>> y = extend(is_called, sid="student_id") << {'student_id'}
    >>> x == y
    True

Or, if we want to be more general::

    >>> def rename2(relation, **kw):
    ...     temps = {'XXtemp'+str(i): old for i, old in enumerate(kw)}
    ...     relation = relation.extend(**temps) << kw.keys()
    ...     new = {n: 'XXtemp'+str(i) for i, n in enumerate(kw.values())}
    ...     relation = relation.extend(**new) << temps.keys()
    ...     return relation
    >>> x = rename(is_called, student_id='name', name='student_id')
    >>> y = rename2(is_called, student_id='name', name='student_id')
    >>> x == y
    True

The XXtemp dodge is required in order to satisfy the parallel evaluation
definition of ``rename`` in *Tutorial D*.

It is interesting to note the nice fallout of being internally consistent and
having the projection arguments be a set: the keys of a dictionary are a set,
so we can use the result of calling the ``keys()`` method directly in the
projection expression.


A Note About Infix Operators
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In defining the infix operators for relations, we have followed already
established Python precedent, both because it makes sense and to preserve
internal consistency in Python-plus-dinsd.  (Remember that relation bodies are
sets of rows.)

Python defines the ``&`` operator as set intersection, which it is for
relations as well, although our ``&`` operator also does more given that the
domain is not *just* a set.  Python defines the ``|`` operator as equivalent
to set union, as does dinsd...but our union has a constrained domain relative
to a straightforward set union of the bodies of the relations.  Finally,
Python defines ``-`` as set difference.  Set difference is pretty much exactly
what the relational ``minus`` is, but it is reasonable to extend it to the
attribute-aware ``notmatching`` operation in the same way that we extend set
union to the attribute-aware ``join``.



Extended Relational Operators
-----------------------------


matching
~~~~~~~~

Following along in Chapter 5 of AIRDT, we need to another couple of relations
to use in the subsequent examples.  We already defined the ``exam_marks``
relation instance in one of the problems above.  Now we'll add courses. ::

    >>> courses = rel(course_id=CID, title=str)(
    ...             ('course_id',   'title'),
    ...             ('C1',          'Database'),
    ...             ('C2',          'HCI'),
    ...             ('C3',          'Op systems'),
    ...             ('C4',          'Programming'),
    ...             )
    >>> print(courses)
    +-----------+-------------+
    | course_id | title       |
    +-----------+-------------+
    | C1        | Database    |
    | C2        | HCI         |
    | C3        | Op systems  |
    | C4        | Programming |
    +-----------+-------------+
    >>> print(exam_marks.display('student_id', 'course_id', 'mark'))
    +------------+-----------+------+
    | student_id | course_id | mark |
    +------------+-----------+------+
    | S1         | C1        | 85   |
    | S1         | C2        | 49   |
    | S1         | C3        | 85   |
    | S2         | C1        | 49   |
    | S3         | C3        | 66   |
    | S4         | C1        | 93   |
    +------------+-----------+------+

*Tutorial D*'s expression::

   ``( COURSE JOIN EXAM_MARK ) { ALL BUT StudentId, Mark }``

becomes the dinsd expression::

    >>> print((courses & exam_marks) << {'student_id', 'mark'})
    +-----------+------------+
    | course_id | title      |
    +-----------+------------+
    | C1        | Database   |
    | C2        | HCI        |
    | C3        | Op systems |
    +-----------+------------+

giving us the table from figure 5.2.

The ``matching`` relational operator encapsulates this (union plus projecting
away any columns obtained from the second relation)::

    >>> from dinsd import matching
    >>> print(matching(courses, exam_marks))
    +-----------+------------+
    | course_id | title      |
    +-----------+------------+
    | C1        | Database   |
    | C2        | HCI        |
    | C3        | Op systems |
    +-----------+------------+

Although this is the opposite of ``notmatching`` in a relational sense,
it is not an intuitive match for ``+`` the way ``notmatching`` is for
``-``.  Hence we do not overload any Python operators for this relational
operator.

A match against an empty relation is empty::

    >>> matching(type(courses)(), courses)
    rel({'course_id': CID, 'title': str})()

In the reverse of the case for ``notmatching``, if there are no common
attributes matching against an empty relation produces an empty relation,
while matching against a non-empty relation produces the original
relation::

    >>> matching(courses, Dum)
    rel({'course_id': CID, 'title': str})()
    >>> matching(courses, Dee) == courses
    True

We can't match relations with attributes having the same name but different
types::

    >>> matching(courses, rel(row(course_id='S1')))
    ...
    ... # doctest: +NORMALIZE_WHITESPACE
    Traceback (most recent call last):
        ...
    TypeError: Duplicate attribute name ('course_id') with different type
    (first: <class '__main__.CID'>, second: <class 'str'> found in match
    relation (relation types are <class 'dinsd.rel({'course_id': CID,
     'title': str})'> and <class 'dinsd.rel({'course_id': str})'>)

For completeness, the definition of ``matching`` in terms of the existing
operands (which is not how it is implemented) would be::

    >>> def matching2(first, second):
    ...     return (first & second) >> first.header.keys()
    >>> matching2(courses, exam_marks) == matching(courses, exam_marks)
    True


compose
~~~~~~~

``compose`` joins its operands and then projects away the attributes
that the two relations have in common, eg::

    ( COURSE JOIN EXAM_MARK ) { ALL BUT CourseId }

In *Tutorial D* this is written the ``COMPOSE`` operator as::

    COURSE COMPOSE EXAM_MARK
    
We can view the compose operation as adding together the two component
relations.  The fact that this operation elides the common attributes makes it
not directly analogous to arithmetic or set addition, but it is close enough
that it seems to make sense to overload the Python ``+`` operator as the
relational compose.  The other justification for this is that there is not
another relational operator (other than join, for which we already have ``&``)
whose meaning is anywhere near as close. ::

    >>> course_marks = (courses & exam_marks) << {'course_id'}
    >>> courses + exam_marks == course_marks
    True

which gives us our equivalent of the table from Figure 5.3::

    >>> print(course_marks.display('title', 'student_id', 'mark',
    ...                             sort=('student_id', 'title')))
    +------------+------------+------+
    | title      | student_id | mark |
    +------------+------------+------+
    | Database   | S1         | 85   |
    | HCI        | S1         | 49   |
    | Op systems | S1         | 85   |
    | Database   | S2         | 49   |
    | Op systems | S3         | 66   |
    | Database   | S4         | 93   |
    +------------+------------+------+

There is also a prefix functional version::

    >>> from dinsd import compose
    >>> compose(courses, exam_marks) == course_marks
    True

If there are no common attributes, ``compose`` is the same as ``join``::

    >>> r1 = rel(row(foo=1, bar=2), row(foo=3, bar=4))
    >>> r2 = rel(row(bing=5, bang=6), row(bing=7, bang=8))
    >>> r1 & r2 == r1 + r2
    True

If all the attributes are in common, they all get projected away and we get
``Dee``::

    >>> course_marks + course_marks == Dee
    True

Compose is commutative, but not associative (which is an argument
against using ``+``, but we're ignoring that)::

    >>> courses + exam_marks == exam_marks + courses
    True
    >>> r = rel(student_id=SID, marks=int, name=str)(
    ...             ('student_id', 'marks', 'name'),
    ...             ('S1',         30,      'Anne'),
    ...             ('S2',         20,      'Borris'))
    >>> c1 = (is_called + r) + exam_marks
    >>> c2 = is_called + (r + exam_marks)
    >>> c1 == c2
    False
    >>> print(c1)
    +-----------+------+-------+------------+
    | course_id | mark | marks | student_id |
    +-----------+------+-------+------------+
    | C1        | 49   | 30    | S2         |
    | C1        | 85   | 30    | S1         |
    | C1        | 93   | 30    | S4         |
    | C2        | 49   | 30    | S1         |
    | C3        | 66   | 30    | S3         |
    | C3        | 85   | 30    | S1         |
    +-----------+------+-------+------------+
    >>> print(c2)
    +-----------+------+-------+------------+
    | course_id | mark | marks | student_id |
    +-----------+------+-------+------------+
    | C1        | 85   | 30    | S1         |
    | C2        | 49   | 30    | S1         |
    | C3        | 85   | 30    | S1         |
    +-----------+------+-------+------------+



Aggregate Operators
-------------------

Aggregate operators map from relations to scalers.  The results of aggregate
operators are well defined (and thus the operator is a true relational
aggregate operator) only if the dyadic form of the operation (the "base
operation") is commutative and associative.  That is, the order of evaluation
must not matter, since the order in which the rows are accessed is arbitrary
and may change.

For aggregate operators dinsd departs a bit from *Tutorial D*, though the end
result is the same functionality.  In *Tutorial D* an aggregate operator takes
a relation as an argument, plus an expression over the rows of that relation
that yield the values to be aggregated.  dinsd inverts this, providing a
single operator (``compute``) that takes a relation and an expression and
returns an arbitrarily ordered list of values that are the result of applying
that expression to each row in turn.  The dinsd "aggregate" operators are then
applied to this returned list, producing the ultimate aggregated value.

In effect, dinsd *decomposes* aggregation into two steps: ``compute``, whose
part of the operation is to turn the relation into a single list of values,
and a specific "summarization" operator that reduces the list to another form,
usually a single scaler value.  So, while we call the functions that we pass
the ``compute`` results to "aggregate functions", in fact it is only the
composition of the dinsd "aggregate" function with the dinsd ``compute``
operator that really meets the definition of "aggregate operator" from AIRDT.

We will see shortly that this provides both considerable flexibility, and
better integration with the underlying Python language.

dinsd does have a couple of aggregate operators in the AIRDT sense, operators
that take relations as arguments and return an aggregated result: ``display``,
which was discussed earlier, and ``len``, which we will encounter in the next
section.  These exist as non-decomposed operators because they operate on
the *entire* row (they take no expression argument).  This means that the
corresponding ``compute`` function would trivially return the entire row as
its result.  ``display`` and ``len``, then, are shorthand notations for
aggregate operations that take no expression argument.


len (count)
~~~~~~~~~~~

The *Tutorial D* operator ``COUNT`` returns the number of rows in a relation
(Example 5.1)::

    COUNT ( EXAM_MARK )

Just as we can overload the meaning of (most of) the standard infix operators
in Python, there are also certain prefix operators that we can overload.
One of these is ``len``, which is the standard Python way to get the "count"
of the number of elements in an object.  This is clearly analogous to
``COUNT``, and so in dinsd we so overload ``len``::

    >>> len(exam_marks)
    6
    >>> len(is_called)
    5

Example 5.2 demonstrates using a relational expression as an argument
to ``COUNT``::

    COUNT ( ( EXAM_MARK WHERE Mark > 50 ) {StudentId} )

In dinsd we write this::

    >>> len(exam_marks.where("mark > 50") >> {'student_id'})
    3


compute
~~~~~~~

    >>> from dinsd import compute

As discussed in the introduction to this chapter, ``compute`` has no analog
in *Tutorial D*.  All of the aggregate functions in *Tutorial D* version 2 have
the ability to use an arbitrary expression to compute a value over each row of
the relation being aggregated.  ``compute`` is the dinsd function that
performs this computation operation.  It is an iterator (specifically a
generator) that produces the computed values one by one, which the individual
"aggregate" operators then consume.  Since the corresponding Python functions
can consume iterators, this means that most of dinsd's "aggregation" operators
are vanilla Python functions applied to a ``compute`` value.

``compute`` takes a relation and a function, which can be either a function of
one argument or a string expression.  The function/expression is passed each
row of the relation in turn, and the result of the function applied to that
row is yielded as the next value of the iterator.

For example, we can extract the scores from the ``mark`` column of the
``exam_marks`` table like this::

    >>> sorted(compute(exam_marks, "mark"))
    [49, 49, 66, 85, 85, 93]

Here ``mark`` is a very simple expression: the value of a single attribute.
The call to ``sorted`` here does two things:  it consumes all of the values
from the compute generator, and it gives the returned values a predictable
order so that the result can be used as part of the doctest.

Since ``compute`` is a function of only one relation, it is also available
as a relation method::

    >>> sorted(exam_marks.compute("mark"))
    [49, 49, 66, 85, 85, 93]

Although it is often used just to extract a column, it is named ``compute``
because it's expression argument is an arbitrary computation::

    >>> sorted(exam_marks.compute("'{}%'.format(mark)"))
    ['49%', '49%', '66%', '85%', '85%', '93%']

We can alternatively specify a full ``lambda`` function instead of a string::

    >>> sorted(exam_marks.compute(lambda r: "{}%".format(r.mark)))
    ['49%', '49%', '66%', '85%', '85%', '93%']

Although this works, ``compute``'s value is the access it gives to the
expression name space.  Anything you can do by passing a ``lambda`` function
to ``compute`` you can do more easily using a normal Python generator
expression (or list comprehension)::

    >>> sorted("{}%".format(r.mark) for r in exam_marks)
    ['49%', '49%', '66%', '85%', '85%', '93%']

This works because when a relation is iterated, it yields its rows.

The other reason for using ``compute`` instead of a generator or list
comprehension is that it may give the back end opportunities for optimization,
but again that works only if you use string expressions.  On the other hand,
if you have a computation that is too complex to express as a string
expression, it is probably too complex for the back end to optimize, and you
you may as well express it as a for loop::

    >>> for rw in exam_marks:
    ...     # Do something complicated here
    ...     pass


sum
~~~

The way to write the sum of the values of an attribute in *Tutorial D*
is shown in Example 5.3::

    SUM ( EXAM_MARK WHERE StudentId = SID ( 'S1' ), Mark )

The Python ``sum`` function can be used to "sum" any values that support
addition.  The result will be a valid relational aggregation if and only if
the addition operation is both associative and commutative.  Arithmetic
addition satisfies this criteria.  Here is AIRDT's example 5.3 in dinsd::

    >>> sum(exam_marks.where("student_id == SID('S1')").compute("mark"))
    219

Tuple concatenation does not satisfy the criteria for an aggregate operator,
so while the expression::

    >>> t = sum(course_marks.compute('(title,)'), ())
    >>> sorted(list(t)) == sorted(course_marks.compute('title'))
    True

produces a result, it is neither a useful result nor a valid relational
aggregation result.  We can't use the result ``t`` above as a doctest result,
because the titles will be in a random order every time Python is run.  For
reference, here are two examples from different runs::

    ('HCI', 'Database', 'Database', 'Database', 'Op systems', 'Op systems')
    ('Database', 'Database', 'Op systems', 'Op systems', 'HCI', 'Database')

These are *not* equivalent tuples.

If we understand Python's ``sum`` operator as being by default an arithmetic
operator, it produces the correct result when passed an attribute from an
empty relation::

    >>> sum(rel(foo=int)().compute('foo'))
    0


max and min
~~~~~~~~~~~

In *Tutorial D*, ``max`` and ``min`` look like this (from Example 5.4)::

    MAX ( EXAM_MARK WHERE StudentId = SID ( 'S1' ), Mark )
    MIN ( EXAM_MARK WHERE StudentId = SID ( 'S1' ), Mark )

Like ``sum``, the Python ``max`` and ``min`` functions accept iterators as
input, so we can use them to compute the corresponding relational max and min::

    >>> max(exam_marks.where("student_id == SID('S1')").compute('mark'))
    85
    >>> min(exam_marks.where("student_id == SID('S1')").compute('mark'))
    49

``min`` and ``max`` do not have defined results on an empty relation::

    >>> min(compute(rel(foo=int)(), 'foo'))
    Traceback (most recent call last):
        ...
    ValueError: min() arg is an empty sequence


avg
~~~

Python doesn't have a function that corresponds directly to the *Tutorial D*
``AVG`` function.  It is a bit tricky to define it as a function that takes a
single iterator as its argument if that iterator does not support ``len``, and
``compute`` does not support ``len`` since it is a generator function.  So
dinsd provides a suitable ``avg`` function::

    >>> from dinsd import avg
    >>> round(avg(exam_marks.compute('mark')), 2)
    71.17
    >>> avg(rel(row(foo=3)).compute('foo'))
    3.0

The average of an empty relation involves division by zero, and that is
exactly the result we get from trying it::

    >>> avg(rel(foo=int)().compute('foo'))
    Traceback (most recent call last):
        ...
    ZeroDivisionError: division by zero


any and all
~~~~~~~~~~~

The Python/dinsd version of the *Tutorial D* ``AND`` and ``OR`` operators are
named ``all`` and ``any``, respectively.  Because Python assigns a truth value
to almost all data types, these functions can be applied to almost any
attribute.  Whether that is useful or not depends on what is stored under
that attribute, of course. ::

    >>> any(exam_marks.compute('mark'))
    True
    >>> all(exam_marks.compute('mark'))
    True
    >>> bools = rel(bar=int, foo=bool)(('bar', 'foo'), (1, True), (2, False),
    ...                                                (3, True), (4, True))
    >>> any(bools.compute('foo'))
    True
    >>> all(bools.compute('foo'))
    False

``any`` and ``all`` have the correct logical behavior on empty relations::

    >>> any(type(bools)().compute('foo'))
    False
    >>> all(type(bools)().compute('foo'))
    True



Relations and Rows as Values
-----------------------------

with
~~~~

Here is a dinsd expression to computes the table in figure 5.4 on page 130 of
AIRDT, the number of students that have sat for an exam in each course::

    >>> x = (courses >> {'course_id'}).extend(
    ...            n=lambda r: len(exam_marks.where(
    ...                                  lambda x: x.course_id==r.course_id
    ...                                 ) >> {'student_id'}))
    >>> print(x)
    +-----------+---+
    | course_id | n |
    +-----------+---+
    | C1        | 3 |
    | C2        | 1 |
    | C3        | 2 |
    | C4        | 0 |
    +-----------+---+

Because there are two row namespaces involved here, we have to use lambda
expressions instead of string expressions.  So this is clearly not optimal.

In any case, that is getting ahead of the text.  Here is the dinsd expression
I naively came up with to compute the table in figure 5.5::

    >>> c_er = courses.extend(exam_result=lambda r:
    ...            exam_marks.where(
    ...                  lambda x: x.course_id==r.course_id)
    ...                      >> {'student_id', 'mark'}) << {'title'}
    >>> print(c_er)
    +-----------+-----------------------+
    | course_id | exam_result           |
    +-----------+-----------------------+
    | C1        | +------+------------+ |
    |           | | mark | student_id | |
    |           | +------+------------+ |
    |           | | 49   | S2         | |
    |           | | 85   | S1         | |
    |           | | 93   | S4         | |
    |           | +------+------------+ |
    |           |                       |
    | C2        | +------+------------+ |
    |           | | mark | student_id | |
    |           | +------+------------+ |
    |           | | 49   | S1         | |
    |           | +------+------------+ |
    |           |                       |
    | C3        | +------+------------+ |
    |           | | mark | student_id | |
    |           | +------+------------+ |
    |           | | 66   | S3         | |
    |           | | 85   | S1         | |
    |           | +------+------------+ |
    |           |                       |
    | C4        | +------+------------+ |
    |           | | mark | student_id | |
    |           | +------+------------+ |
    |           | +------+------------+ |
    |           |                       |
    +-----------+-----------------------+

You will note that we have no way to control the print order of the subtables,
so they come out in the default sorted order.  It would be possible to extend
display to support column and sort specifications for attributes of the main
table, but the complexity is not warranted by the minimal use that such a
facility would be likely to get.

AIRDT uses the following *Tutorial D* expression to compute the above
table in Example 5.6::

    EXTEND COURSE{CourseId} ADD
    ( RELATION { TUPLE { CourseId CourseId } } COMPOSE EXAM_MARK
      AS ExamResult )

Here is the simpler, direct equivalent of that expression in dinsd.  Here we
need to use a dinsd syntax that doesn't have a direct correspondent
in *Tutorial D* (though there is something that looks almost like it), a ``with
ns(...):`` block.  We use this to make the ``exam_marks`` relation temporarily
available under that name in the expression name space::

    >>> from dinsd import ns
    >>> with ns(exam_marks=exam_marks):
    ...     c_er == extend(courses >> {'course_id'}, exam_result=
    ...                "rel(row(course_id=course_id)) + exam_marks"
    ...                )
    True

If ``exam_marks`` and ``courses`` were both part of the same database, their
names would automatically be part of the expression name space, but we haven't
covered creating a database yet, so for now we need to use ``with``.  ``with``
can be used to add arbitrary named values to the expression name space, and is
the mechanism alluded to early that allows you to avoid creating expression
strings programmatically when you want to introduce dynamic values from
outside the database into an expression.  It is the dinsd equivalent of
SQL parameters.

While dinsd's ``with ns(...)`` shares some similarities with *Tutorial D*'s
``WITH`` expression, its purpose is subtly different.  We can achieve the
affect of *Tutorial D*'s ``WITH`` in Python simply by assigning an expression
to a name, as we have been doing all along.  But those names are only
available in the full *Python* context...to *also* make them available in the
special evaluation environment of a string-valued function expression, we use
``with ns(...)``.

Having successfully written first expression above, and successfully
translated example 5.6, I have to admit that I had no clue what AIRDT's
Example 5.7 was doing::

    WITH
        EXTEND COURSE{CourseId} ADD
        ( RELATION { TUPLE { CourseId CourseId } } COMPOSE EXAM_MARK
          AS ExamResult )
        AS C_ER :
    EXTEND C_ER ADD ( COUNT ( ExamResult ) AS n )
    { ALL BUT ExamResult }

After reading the explanation and thinking about it, I believe he uses that
more complicated expression because *Tutorial D* version 1 did not have a
general expression facility for aggregate operators.  Although we can do it in
one step as shown above, for completeness we will translate this two step
version into dinsd as well::

    >>> with ns(exam_marks=exam_marks):
    ...     c_er = (courses >> {'course_id'}).extend(exam_result=
    ...                 "rel(row(course_id=course_id)) + exam_marks"
    ...                 )
    ...     counts = c_er.extend(n="len(exam_result)") << {'exam_result'}
    ...     print(counts)
    +-----------+---+
    | course_id | n |
    +-----------+---+
    | C1        | 3 |
    | C2        | 1 |
    | C3        | 2 |
    | C4        | 0 |
    +-----------+---+

Example 5.8 shows a similar computation for the average grade::

    WITH
        EXTEND EXAM_MARK{CourseId} ADD
        ( RELATION { TUPLE { CourseId CourseId } } COMPOSE EXAM_MARK
          AS ExamResult )
     AS C_ER2 :
    EXTEND C_ER2 ADD
        ( AVG(ExamResult, Mark) AS AvgMark )
    { ALL BUT ExamResult }

Here we will translate this using the ``compose`` form, but taking advantage
of dinsd's decomposed aggregation operators to do it in one step::

    >>> with ns(exam_marks=exam_marks):
    ...     av = (exam_marks >> {'course_id'}).extend(avg_mark=
    ...               "round(avg((rel(row(course_id=course_id)) + "
    ...                          "exam_marks).compute('mark')), 2)"
    ...               )
    ...     print(av.display('course_id', 'avg_mark'))
    +-----------+----------+
    | course_id | avg_mark |
    +-----------+----------+
    | C1        | 75.67    |
    | C2        | 49.0     |
    | C3        | 75.5     |
    +-----------+----------+

The relation being extended changes to exam_marks here so that we only have
rows for exams that had marks.  AIRDT notes that if we had continued to use
courses we'd end up with a division by zero for the course with no marks, and
indeed we do::

    >>> with ns(exam_marks=exam_marks):
    ...     av = (courses >> {'course_id'}).extend(avg_mark=
    ...               "round(avg((rel(row(course_id=course_id)) + "
    ...                          "exam_marks).compute('mark')), 2)"
    ...               )
    Traceback (most recent call last):
        ...
    ZeroDivisionError: division by zero

The two step version is probably easier to parse visually, but as we will see
in the next section, there is an even simpler way to do this operation.

Namespaces can be nested.  This isn't particularly useful in a single piece of
code (nested with statements), but can be very useful if a called function
needs to add something to the namespace.  We'll demonstrate the capability with
a contrived example::

    >>> def avg_marks():
    ...     with ns(exam_marks=exam_marks):
    ...         return (exam_marks >> {'course_id'}).extend(avg_mark=
    ...                     "round(avg((rel(row(course_id=course_id)) + "
    ...                                "exam_marks).compute('mark')), 2)"
    ...                )
    >>> with ns(is_enrolled_on=is_enrolled_on):
    ...     one_student_enrolled = avg_marks().where(
    ...         "len(matching(is_enrolled_on, "
    ...                      "rel(row(course_id=course_id)))) == 1")
    >>> print(one_student_enrolled)
    +----------+-----------+
    | avg_mark | course_id |
    +----------+-----------+
    | 49.0     | C2        |
    | 75.5     | C3        |
    +----------+-----------+

The names placed into the expression namespace by the context manager are
removed at the end of the context::

    >>> av = (exam_marks >> {'course_id'}).extend(avg_mark=
    ...           "round(avg((rel(row(course_id=course_id)) + "
    ...                      "exam_marks).compute('mark')), 2)"
    ...           )
    Traceback (most recent call last):
        ...
    NameError: name 'exam_marks' is not defined

    >>> with ns():
    ...     av = (exam_marks >> {'course_id'}).extend(avg_mark=
    ...               "round(avg((rel(row(course_id=course_id)) + "
    ...                          "exam_marks).compute('mark')), 2)"
    ...               )
    Traceback (most recent call last):
        ...
    NameError: name 'exam_marks' is not defined

The namespace additions are also kept separate across threads::

    >>> import threading
    >>> start = threading.Event()
    >>> def thread_demo():
    ...     with ns(exam_marks=exam_marks, is_enrolled_on=is_enrolled_on):
    ...         start.wait(timeout=20)
    ...         av = (exam_marks >> {'course_id'}).extend(avg_mark=
    ...                   "round(avg((rel(row(course_id=course_id)) + "
    ...                              "exam_marks).compute('mark')), 2)"
    ...                   )
    ...         print(av.display('course_id', 'avg_mark'))
    >>> t = threading.Thread(target=thread_demo)
    >>> t.start()
    >>> fake_exam_marks = rel(exam_marks.header)(
    ...                       ('student_id', 'course_id', 'mark'),
    ...                       ('S1', 'C1', 87),
    ...                       ('S1', 'C2', 89),
    ...                       ('S1', 'C3', 89),
    ...                       )
    >>> with ns(exam_marks=fake_exam_marks):
    ...     start.set()
    ...     f = (exam_marks >> {'course_id'}).extend(avg_mark=
    ...               "round(avg((rel(row(course_id=course_id)) + "
    ...                          "exam_marks).compute('mark')), 2)"
    ...               )
    ...     t.join()
    ...     print(f.display('course_id', 'avg_mark'))
    +-----------+----------+
    | course_id | avg_mark |
    +-----------+----------+
    | C1        | 75.67    |
    | C2        | 49.0     |
    | C3        | 75.5     |
    +-----------+----------+
    +-----------+----------+
    | course_id | avg_mark |
    +-----------+----------+
    | C1        | 87.0     |
    | C2        | 89.0     |
    | C3        | 89.0     |
    +-----------+----------+


summarize
~~~~~~~~~
    >>> from dinsd import summarize

``summarize`` is designed to make the above complex expressions simpler and
easier to understand.  The basic idea is that the pattern of composing with a
row containing data from one table to obtain a list of matching data from
another table (which could be the same table) and then aggregating over that
extracted data is very common.  So we have an operator that expresses that
more compactly.  Here is the *Tutorial D* version from Example 5.9::

    SUMMARIZE EXAM_MARK PER ( COURSE { CourseId } )
               ADD ( COUNT ( ) AS n )

In *Tutorial D*, this expression has its own special semantics: you omit the
relation argument to the aggregation functions, as they automatically work on
the relations provided, one by one, by ``SUMMARIZE``.  In dinsd, on the other
hand, ``summarize`` works exactly as if it is a composition of ``compose`` and
``extend`` with an automatic "project away" of the temporary column at the
end.  The temporary column is named ``_summary_``, which is guaranteed not to
collide with any real attributes since it would be an invalid attribute name.

The dinsd version is::

    >>> x = summarize(exam_marks, courses >> {'course_id'},
    ...               n="len(_summary_)")
    >>> print(x)
    +-----------+---+
    | course_id | n |
    +-----------+---+
    | C1        | 3 |
    | C2        | 1 |
    | C3        | 2 |
    | C4        | 0 |
    +-----------+---+

``summarize`` will of course also take a real function as the value for a new
attribute::

    >>> x == summarize(exam_marks, courses >> {'course_id'},
    ...               n=lambda r: len(r._summary_))
    True

The above formulation corresponds to the *Tutorial D* ``SUMMARIZE PER``
construct.  The dinsd equivalent of *Tutorial D*'s ``SUMMARIZE BY`` looks
almost exactly the same, except that the second argument is a set of attribute
names, instead of a relation.  Those names are used to project the main
relation for composition with itself.  We used that pattern in the average
computation::

    >>> av == summarize(exam_marks, {'course_id'},
    ...                 avg_mark="round(avg(_summary_.compute('mark')), 2)")
    True

Definitely a worthwhile simplification.

This ``BY`` form is a function of a single relation, and so can also be used
in the form of a relation method::

    >>> av == exam_marks.summarize({'course_id'},
    ...                 avg_mark="round(avg(_summary_.compute('mark')), 2)")
    True

Because debugging a summarization can be tricky without being able to see the
data computed by the compose, ``summarize`` supports a special boolean keyword
argument ``_deubg``.  If set to ``True``, the intermediate table extended with
the composed data is printed to stdout::

    >>> x = exam_marks.summarize({'course_id'}, _debug_=True,
    ...              avg_mark="round(avg(_summary_.compute('mark')), 2)")
    +-----------------------+-----------+
    | _summary_             | course_id |
    +-----------------------+-----------+
    | +------+------------+ | C2        |
    | | mark | student_id | |           |
    | +------+------------+ |           |
    | | 49   | S1         | |           |
    | +------+------------+ |           |
    |                       |           |
    | +------+------------+ | C1        |
    | | mark | student_id | |           |
    | +------+------------+ |           |
    | | 49   | S2         | |           |
    | | 85   | S1         | |           |
    | | 93   | S4         | |           |
    | +------+------------+ |           |
    |                       |           |
    | +------+------------+ | C3        |
    | | mark | student_id | |           |
    | +------+------------+ |           |
    | | 66   | S3         | |           |
    | | 85   | S1         | |           |
    | +------+------------+ |           |
    |                       |           |
    +-----------------------+-----------+

Summarize can take any number of new column computations functions, and they
may be a mix of strings and lambda expressions::

    >>> x = exam_marks.summarize({'course_id'},
    ...               attendees=lambda r: len(r._summary_),
    ...               avg_mark="round(avg(_summary_.compute('mark')), 2)")
    >>> print(x.display('course_id', 'attendees', 'avg_mark'))
    +-----------+-----------+----------+
    | course_id | attendees | avg_mark |
    +-----------+-----------+----------+
    | C1        | 3         | 75.67    |
    | C2        | 1         | 49.0     |
    | C3        | 2         | 75.5     |
    +-----------+-----------+----------+

If no additional attributes are specified, the result is equal to the
projection::

    >>> exam_marks.summarize({'course_id'}) == exam_marks >> {'course_id'}
    True
    >>> (summarize(exam_marks, courses >> {'course_id'}) ==
    ...     courses >> {'course_id'})
    True


group
~~~~~

``group`` encapsulates the ``extend``/``compose`` sub-operation of summarize.
In Example 5.8 we had the subexpression::

    EXTEND EXAM_MARK{CourseId} ADD
    ( RELATION { TUPLE { CourseId CourseId } } COMPOSE EXAM_MARK
      AS ExamResult )

Example 5.12 shows an alternative *Tutorial D* formulation that produces the
same result for this case::

    EXAM_MARK GROUP ( { StudentId, Mark } AS ExamResult )

In dinsd we write this::

    >>> results = exam_marks.group(exam_results={'student_id', 'mark'})

Which gives us our version of Figure 5.6::

    >>> print(results)
    +-----------+-----------------------+
    | course_id | exam_results          |
    +-----------+-----------------------+
    | C1        | +------+------------+ |
    |           | | mark | student_id | |
    |           | +------+------------+ |
    |           | | 49   | S2         | |
    |           | | 85   | S1         | |
    |           | | 93   | S4         | |
    |           | +------+------------+ |
    |           |                       |
    | C2        | +------+------------+ |
    |           | | mark | student_id | |
    |           | +------+------------+ |
    |           | | 49   | S1         | |
    |           | +------+------------+ |
    |           |                       |
    | C3        | +------+------------+ |
    |           | | mark | student_id | |
    |           | +------+------------+ |
    |           | | 66   | S3         | |
    |           | | 85   | S1         | |
    |           | +------+------------+ |
    |           |                       |
    +-----------+-----------------------+

There is also a prefix version of ``group``::

    >>> from dinsd import group
    >>> results == group(exam_marks, exam_results={'student_id', 'mark'})
    True

Since the grouping specification is an attribute name set, we can also use
the ``all_but`` wrapper with it::

    >>> results == exam_marks.group(exam_results=all_but({'course_id'}))
    True

The grouping refers implicitly to both the set of columns being grouped and
the set of columns not being grouped.  This leads to an ambiguity as to which
columns are involved if there are multiple grouping specifications.  Rather
than trying to sort this out and greatly complicating the implementation for
little practical benefit, the group function only accepts one new column name::

    >>> group(exam_marks, res1={'marks'}, res2={'student_id'})
    Traceback (most recent call last):
        ...
    TypeError: Only one new attribute may be specified for group

In this we follow *Tutorial D*, which dropped support for multiple grouping
columns in Version 2.

It is an error to try to group on a non-existent column::

    >>> group(exam_marks, foo={'bar'})
    Traceback (most recent call last):
        ...
    TypeError: Attribute list included unknown attributes: {'bar'}

Grouping on no attributes produces a column consisting of instances of
``Dee``::

    >>> print(group(exam_marks, foo={}))
    +-----------+-----+------+------------+
    | course_id | foo | mark | student_id |
    +-----------+-----+------+------------+
    | C1        | ++  | 49   | S2         |
    |           | ||  |      |            |
    |           | ++  |      |            |
    |           | ||  |      |            |
    |           | ++  |      |            |
    |           |     |      |            |
    | C1        | ++  | 85   | S1         |
    |           | ||  |      |            |
    |           | ++  |      |            |
    |           | ||  |      |            |
    |           | ++  |      |            |
    |           |     |      |            |
    | C1        | ++  | 93   | S4         |
    |           | ||  |      |            |
    |           | ++  |      |            |
    |           | ||  |      |            |
    |           | ++  |      |            |
    |           |     |      |            |
    | C2        | ++  | 49   | S1         |
    |           | ||  |      |            |
    |           | ++  |      |            |
    |           | ||  |      |            |
    |           | ++  |      |            |
    |           |     |      |            |
    | C3        | ++  | 66   | S3         |
    |           | ||  |      |            |
    |           | ++  |      |            |
    |           | ||  |      |            |
    |           | ++  |      |            |
    |           |     |      |            |
    | C3        | ++  | 85   | S1         |
    |           | ||  |      |            |
    |           | ++  |      |            |
    |           | ||  |      |            |
    |           | ++  |      |            |
    |           |     |      |            |
    +-----------+-----+------+------------+

Grouping on *all* of the attributes, on the other hand, produces a relation
with a single attribute, whose single value is the original relation::

    >>> print(group(exam_marks, foo=all_but({})))
    +-----------------------------------+
    | foo                               |
    +-----------------------------------+
    | +-----------+------+------------+ |
    | | course_id | mark | student_id | |
    | +-----------+------+------------+ |
    | | C1        | 49   | S2         | |
    | | C1        | 85   | S1         | |
    | | C1        | 93   | S4         | |
    | | C2        | 49   | S1         | |
    | | C3        | 66   | S3         | |
    | | C3        | 85   | S1         | |
    | +-----------+------+------------+ |
    |                                   |
    +-----------------------------------+


ungroup
~~~~~~~

Example 5.13 shows the *Tutorial D* ``UNGROUP`` operator::

    C_ER2 UNGROUP ( ExamResult )

``ungroup`` effectively undoes what ``group`` does.  Here is the
dinsd version::

    >>> results.ungroup('exam_results') == exam_marks
    True

There is also a prefix version::

    >>> from dinsd import ungroup
    >>> ungroup(results, 'exam_results') == exam_marks
    True

AIRDT gives a formal definition for ``ungroup`` in terms of the
operators already introduced (see page 141)::

    r UNGROUP ( a ):
        UNION (
        EXTEND r ADD ( RELATION { t } TIMES a AS x ),
        x)

    where x is an attribute name arbitrarily chosen and
    t = TUPLE { b1 b1, …, bn bn }, where b1, …, bn
    are the attributes of r{ALL BUT a}.

We can implement this recipe in dinsd::

    >>> def ungroup2(r, a):
    ...     return union(r.extend(x=lambda z, typ=type(r):
    ...                                times(typ(z) << {a},
    ...                                      getattr(z, a))).compute("x"))
    >>> exam_marks == ungroup2(results, 'exam_results')
    True

This works because if ``union`` is passed a single argument consisting of a
non-``Relation`` iterator, it does the union of all the relations returned by
that iterator.  That is, it acts as an aggregation function.  The built-in
``ungroup`` is more efficient, though.

We can also do this using a string valued expression, by using the special
variable ``_row_`` to access the entire row::

    >>> def ungroup3(r, a):
    ...     with ns(typ=type(r), a=a):
    ...         return union(r.extend(x=
    ...             "times(typ(_row_) << {a}, locals()[a])").compute("x"))
    >>> exam_marks == ungroup3(results, 'exam_results')
    True

The ``locals()[a]`` is a dodge to get around the fact that what we have in
``a`` is the name that we want to *look up* in the expression namespace.  The
``_row_`` special variable is always defined when a string expression is
evaluated, but it is seldom needed.

It is an error to pass ``ungroup`` a column not in the relation::

    >>> ungroup(results, 'foo')
    Traceback (most recent call last):
        ...
    KeyError: 'foo'

The column specified must of course be a relation valued attribute::

    >>> ungroup(results, 'course_id')
    Traceback (most recent call last):
        ...
    AttributeError: 'CID' object has no attribute 'header'


wrap
~~~~

Example 5.14 shows how to turn attribute data into rows as an attribute value::

    ( EXTEND CONTACT_INFO ADD
        ( TUPLE{ House House, Street Street, City City , Zip Zip }
          AS Address ) )
        { ALL BUT House, Street, City, Zip }

Given a suitable relation instance::

    >>> contact_info = rel(name=str, house=str, street=str, city=str, zip=str)(
    ...                     ('name',  'house', 'street',  'city',   'zip'),
    ...                     ('Anne',  '120',   'Peach',   'London', '00111'),
    ...                     ('Boris', '5',     'Rose',    'Kiev',   '00112'))

So we can write the dinsd equivalent of 5.14 as follows::

    >>> addrs = contact_info.extend(address=
    ...             "row(dict(house=house, street=street, city=city, zip=zip))"
    ...             ) << {'house', 'street', 'city', 'zip'}
    >>> print(addrs.display('name', 'address'))
    +-------+---------------------------------------------------+
    | name  | address                                           |
    +-------+---------------------------------------------------+
    | Anne  | {city=London, house=120, street=Peach, zip=00111} |
    | Boris | {city=Kiev, house=5, street=Rose, zip=00112}      |
    +-------+---------------------------------------------------+

Example 5.5 shows how the *Tutorial D* ``WRAP`` operator does the same thing
more compactly::

    CONTACT_INFO WRAP ( { House, Street, City, Zip } AS Address )

``wrap`` is a function of a single relation, and so is available as a method
in dinsd::

    >>> addrs == contact_info.wrap(address={'house', 'street', 'city', 'zip'})
    True

as well as a prefix function::

    >>> from dinsd import wrap
    >>> addrs == wrap(contact_info, address={'house', 'street', 'city', 'zip'})
    True

It is possible (though usually pointless) to wrap all the attributes::

    >>> print(contact_info.wrap(all=all_but({})))
    +--------------------------------------------------------------+
    | all                                                          |
    +--------------------------------------------------------------+
    | {city=Kiev, house=5, name=Boris, street=Rose, zip=00112}     |
    | {city=London, house=120, name=Anne, street=Peach, zip=00111} |
    +--------------------------------------------------------------+

Wrapping no attributes results in the new attribute having empty rows as its
values::

    >>> print(contact_info.wrap(none={}))
    +--------+-------+-------+------+--------+-------+
    | city   | house | name  | none | street | zip   |
    +--------+-------+-------+------+--------+-------+
    | Kiev   | 5     | Boris | {}   | Rose   | 00112 |
    | London | 120   | Anne  | {}   | Peach  | 00111 |
    +--------+-------+-------+------+--------+-------+

You can't wrap a non-existent attribute::

    >>> contact_info.wrap(bad={'foo'})
    Traceback (most recent call last):
        ...
    TypeError: Attribute list included unknown attributes: {'foo'}

And for the same reasons that applied in the ``compose`` case, you can only
specify a single new attribute in a ``wrap``::

    >>> contact_info.wrap(one={'home'}, two={'zip'})
    Traceback (most recent call last):
        ...
    TypeError: Only one new attribute may be specified for wrap


unwrap
~~~~~~

``unwrap`` is the opposite of ``wrap``.  Expressing this in *Tutorial D*
introduces a new expression (``FROM``)::

    r UNWRAP ( a )

    ( EXTEND r ADD ( b1 FROM a AS b1, …, bn FROM a AS bn ) )
    { ALL BUT a }

    where b1, …, bn are the attributes of a.

The dinsd equivalent of ``FROM`` is Python's attribute access syntax::

    >>> contact_info == extend(addrs,
    ...                        house="address.house",
    ...                        street="address.street",
    ...                        city="address.city",
    ...                        zip="address.zip") << {'address'}
    True

``unrwap`` is of course much simpler::

    CONTACT_INFO_WRAPPED UNWRAP ( Address )

and, in dinsd::

    >>> contact_info == addrs.unwrap('address')
    True

or, in prefix form::

    >>> from dinsd import unwrap
    >>> contact_info == unwrap(addrs, 'address')
    True

And here are the errors and edge cases::

    >>> unwrap(addrs, 'bad')
    Traceback (most recent call last):
        ...
    KeyError: 'bad'
    >>> unwrap(addrs, 'name')
    Traceback (most recent call last):
        ...
    AttributeError: 'str' object has no attribute '_header_'
    >>> unwrap(Dum, 'name')
    Traceback (most recent call last):
        ...
    ValueError: Cannot unwrap an empty relation
    >>> contact_info.wrap(none={}).unwrap('none') == contact_info
    True
    >>> contact_info.wrap(all=all_but({})).unwrap('all') == contact_info
    True


Aside: join operators as aggregators
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

We already saw an example of ``union`` being used as an aggregator (which
means, in dinsd, taking an iterator as input).  This also works for the
join-type operators, which we will now prove for testing completeness. ::

    >>> R = rel(a=int, b=int)
    >>> a = R(('a', 'b'), (1, 2), (3, 4))
    >>> b = R(('a', 'b'), (1, 2), (7, 8))
    >>> c = rel(row(c=5, d=6))
    >>> join((a, b))
    rel({row({'a': 1, 'b': 2})})
    >>> intersect((a, b))
    rel({row({'a': 1, 'b': 2})})
    >>> times((a, c))               # doctest: +NORMALIZE_WHITESPACE
    rel({row({'a': 1, 'b': 2, 'c': 5, 'd': 6}),
         row({'a': 3, 'b': 4, 'c': 5, 'd': 6})})



Relation Comparison
-------------------

Python meets the requirements on the equality relation expounded upon in
section 5.9 or AIRDT.  In particular, string equality is a strict test::

    >>> 'this ' == 'this'
    False

It is possible to define Python types that do not support equality testing,
but you have to go out of your way to do it.  Instances of objects are
compared by identity if no other equality test is defined::

    >>> object() == object()
    False

A large part of the purpose of the dinsd ``Scalar`` class is to provide an
easy way to support both a sensible equality function and a total ordering for
user defined types.

dinsd supports equality and total ordering for all relations.  These operators
work with relations of like type, and except for equality do not work with
relations of unlike type.

Like ``Rel``, Python uses ``<=`` and ``>=`` for the subset and superset
(respectively) comparisons on relations. ::

    >>> R = rel(foo=int, bar=int)
    >>> r1 = R(('foo', 'bar'), (1, 2), (3, 4), (5, 6))
    >>> r2 = R(('foo', 'bar'), (1, 2), (3, 4))
    >>> r3 = R(('foo', 'bar'), (3, 4))
    >>> r4 = R(('foo', 'bar'), (3, 4))

Observe that r3 and r4 are different objects::

    >>> r3 is not r4
    True

So, the expected relationships hold::

    >>> r1 <= r1
    True
    >>> r1 >= r1
    True
    >>> r1 >= r2
    True
    >>> r2 >= r1
    False
    >>> r3 <= r4
    True
    >>> r3 == r4
    True
    >>> r1 == r2
    False
    >>> r1 >= r2 >= r3 >= r4
    True

Python also supports proper subset and proper superset via ``<`` and ``>``::


    >>> r1 < r1
    False
    >>> r1 > r1
    False
    >>> r1 > r2
    True
    >>> r2 > r1
    False
    >>> r3 < r4
    False
    >>> r3 > r4
    False

dinsd does not define an equivalent for the *Tutorial D* ``IS_EMPTY``,
because it would be redundant, as we'll see in a moment.

We could define ``IS_EMTPY`` using the ``TABLE_DUM`` test from AIRDT::

    >>> def is_empty(r):
    ...     return r >> {} == Dum
    >>> is_empty(R())
    True
    >>> is_empty(r1)
    False

Or using the dinsd equivalent of the ``COUNT`` test::

    >>> def is_empty(r):
    ...     return len(r)==0
    >>> is_empty(R())
    True
    >>> is_empty(r1)
    False

However, python objects usually have a boolean value (all the built in types
do).  For relations, dinsd makes the boolean value ``True`` if the relation
has at least one row, and ``False`` if it has no rows::

    >>> bool(R())
    False
    >>> bool(r1)
    True

This means you can use a relation in an ``if`` statement directly to check if
it is empty::

    >>> if r1:
    ...     print("not empty")
    not empty
    >>> if not R():
    ...     print("empty")
    empty

Relations of different types can be tested for equality (if they are different
types they are by Python convention unequal), but are otherwise not
comparable::

    >>> r1 == is_enrolled_on
    False
    >>> r1 < is_enrolled_on                 # doctest: +NORMALIZE_WHITESPACE
    Traceback (most recent call last):
        ...
    TypeError: unorderable types: rel({'bar': int, 'foo': int})() <
        rel({'course_id': CID, 'student_id': SID})()
    >>> r1 <= is_enrolled_on                # doctest: +NORMALIZE_WHITESPACE
    Traceback (most recent call last):
        ...
    TypeError: unorderable types: rel({'bar': int, 'foo': int})() <=
        rel({'course_id': CID, 'student_id': SID})()
    >>> r1 > is_enrolled_on                 # doctest: +NORMALIZE_WHITESPACE
    Traceback (most recent call last):
        ...
    TypeError: unorderable types: rel({'bar': int, 'foo': int})() >
        rel({'course_id': CID, 'student_id': SID})()
    >>> r1 >= is_enrolled_on                # doctest: +NORMALIZE_WHITESPACE
    Traceback (most recent call last):
        ...
    TypeError: unorderable types: rel({'bar': int, 'foo': int})() >=
        rel({'course_id': CID, 'student_id': SID})()

Example 5.16 in AIRDT demonstrates computing a relation listing the names of
all the students that have taken an exam for every course in which they are
enrolled.  To make this interesting, we need at least one student enrolled on
a course where they do not have an exam mark, so let's update
``is_enrolled_on`` with an entry for which that is true::

    >>> is_enrolled_on = union(is_enrolled_on,
    ...                        rel(row(student_id=SID('S2'),
    ...                              course_id=CID('C3'))))

I tried working the "problem", using relational comparison somehow, without
looking at the solution, and came up with::

    >>> m = join(is_enrolled_on, is_called).group(courses={'course_id', 'name'})
    >>> c = join(is_enrolled_on, exam_marks).group(marks={'course_id', 'mark'})
    >>> j = join(m, c)
    >>> r = extend(j, completed=
    ...                 "courses >> {'course_id'} == marks >> {'course_id'}"
    ...           ) >> {'student_id', 'completed'}
    >>> done = join(r, is_called)
    >>> print(done.display('name', 'student_id', 'completed'))
    +----------+------------+-----------+
    | name     | student_id | completed |
    +----------+------------+-----------+
    | Anne     | S1         | True      |
    | Boris    | S2         | False     |
    | Cindy    | S3         | True      |
    | Devinder | S4         | True      |
    +----------+------------+-----------+

Well, that was much more complicated than AIRDT's solution...and I seem to
have misread the problem statement.  In *Tutorial D* he solved this via::

    IS_CALLED NOT MATCHING ( ENROLMENT NOT MATCHING EXAM_MARK )

On the other hand, this solution doesn't use relational comparison.  In dinsd
translates to::

    >>> print(is_called - ((is_enrolled_on & is_called) - exam_marks))
    +----------+------------+
    | name     | student_id |
    +----------+------------+
    | Anne     | S1         |
    | Boris    | S5         |
    | Cindy    | S3         |
    | Devinder | S4         |
    +----------+------------+

His book obviously isn't doctested, since both he and we earlier deleted the
``enrollment`` variable, which I discovered when trying to use it, so
I recreated it via the join above.

My misunderstanding is revealed by the presence of ``S5`` in this table: he is
enrolled on no courses and therefore has trivially sat all his exams.  Mine
lost him because of the joins, making mine the answer to the different
question: "which students who have sat at least one exam have sat the exams
for all of the courses on which they are enrolled?".

Then I noticed that this gives the same result::

    >>> print(is_called - (is_enrolled_on - exam_marks))
    +----------+------------+
    | name     | student_id |
    +----------+------------+
    | Anne     | S1         |
    | Boris    | S5         |
    | Cindy    | S3         |
    | Devinder | S4         |
    +----------+------------+

I wondered why he didn't use that formulation, but in fact I think he intended
to, as we'll see below.

Example 5.17 shows how to do this computation with logic that is easier to
derive from the problem statement, by using relational comparision::

    IS_CALLED WHERE
    ENROLMENT COMPOSE RELATION {TUPLE {StudentId StudentId}}
    ⊆
    ( EXAM_MARK COMPOSE RELATION {TUPLE {StudentId StudentId}} )
     {ALL BUT Mark}

Here is the dinsd equivalent, in which we use a slightly different form of the
``ns`` context manager to get our two component relations into the expression
namespace::

    >>> with ns() as n:                     # doctest: +NORMALIZE_WHITESPACE
    ...     n['exam_marks'] = exam_marks
    ...     n['enrollment'] = is_called & is_enrolled_on
    ...     print(is_called.where(          
    ...            "enrollment + rel(row(student_id=student_id)) <= "
    ...            "exam_marks + rel(row(student_id=student_id)) "
    ...                "<< {'mark'}"
    ...          ))
    Traceback (most recent call last):
        ...
    TypeError: unorderable types: rel({'course_id': CID, 'name': str})() <=
        rel({'course_id': CID})()

Hmm.  Composing enrollment gives us ``course_id`` and ``name``.  Composing
``exam_marks`` gives us ``course_id`` and ``marks``... and we then get
rid of ``marks``, leaving us with ``course_id`` (as we can see from the
projection relation names in the error message).  So, no, those two
aren't comparable.  We can do::

    >>> with ns() as n:
    ...     n['exam_marks'] = exam_marks
    ...     n['enrollment'] = is_called & is_enrolled_on
    ...     print(is_called.where(
    ...         "(enrollment + rel(row(student_id=student_id))) "
    ...             "<< {'name'}  <= "
    ...         "(exam_marks + rel(row(student_id=student_id))) "
    ...             "<< {'mark'}"
    ...      ))
    +----------+------------+
    | name     | student_id |
    +----------+------------+
    | Anne     | S1         |
    | Boris    | S5         |
    | Cindy    | S3         |
    | Devinder | S4         |
    +----------+------------+

    XXX: The above is a bug in ns...the names aren't supposed to remain in
         the expression namespace.

Which gets us the expected result.  So does::

    >>> with ns(exam_marks=exam_marks, is_enrolled_on=is_enrolled_on):
    ...    print(is_called.where(
    ...         "is_enrolled_on + rel(row(student_id=student_id)) <= "
    ...         "(exam_marks + rel(row(student_id=student_id))) "
    ...             "<< {'mark'}"
    ...         ))
    +----------+------------+
    | name     | student_id |
    +----------+------------+
    | Anne     | S1         |
    | Boris    | S5         |
    | Cindy    | S3         |
    | Devinder | S4         |
    +----------+------------+

And this clears up the mystery about ``enrollment`` in the AIRDT examples:
he's using ``enrollment`` in those examples to refer to ``is_enrolled_on``.



Other Operators on Relations and Rows
-------------------------------------


in and ~ (extract_only_row)
~~~~~~~~~~~~~~~~~~~~~~~~~~~

In section 5.10, AIRDT talks about the tuple membership test::

    t ∈ r

In ``Rel`` this is spelled ``IN``, and in dinsd it is spelled ``in`` (which
is a standard Python operator)::

    >>> row(student_id=SID('S1'), name='Anne') in is_called
    True
    >>> row(student_id=SID('S2'), name='Anne') in is_called
    False
    >>> row(name='Anne') in is_called
    False

The membership test depends on the equality test, and so in dinsd it is
defined for unlike types.  (It is categorically true that a row of type ``t1``
type is not an element of a relation of type ``t2``.)

We've already mentioned the dinsd equivalent of the *Tutorial D* ``FROM``
operation in the context of extracting attributes from tuples.  *Tutorial D*
also has a ``TUPLE FROM`` operation it uses to extra the row from a relation
that only contains one row::

    TUPLE FROM COURSE WHERE CourseId = CID('C1')

For the dinsd equivalent we follow a convention that exists in other Python's
database libraries, and use the operator ``~``::

    >>> c1 = ~courses.where("course_id == CID('C1')")
    >>> print(c1)
    {course_id=C1, title=Database}

``~`` is Python's "binary not" operation.  There is no strong connection
between that fact and dinsd use of it, but since it is a unary prefix operator
it isn't useful for anything else in the relational algebra context.  We can
rationalize its use a small bit by noting that the name of the Python operator
is "invert", and there is a (colloquial) sense in which the inversion of a
relation of a single row is the row.

It is also available as a prefix function, if using the operator bothers you::

    >>> from dinsd import extract_only_row
    >>> extract_only_row(courses.where("course_id == CID('C1')")) == c1
    True

The name is explicit, if not very convenient to type.

As you might gather from the function name, it is an error to use this
operator on a relation that has more than one row::

    >>> ~rel(courses)                       # doctest: +NORMALIZE_WHITESPACE
    Traceback (most recent call last):
        ...
    ValueError: <class 'dinsd.rel({'course_id': CID, 'title': str})'> object
        has more than one row

Likewise it cannot be used on an empty relation::

    >>> ~(rel(name=str)())
    Traceback (most recent call last):
        ...
    StopIteration

For completeness, the example of compound use of ``FROM``::

    Title FROM TUPLE FROM COURSE WHERE CourseId = CID('C1')

is written in dinsd as follows::

    >>> (~courses.where("course_id == CID('C1')")).title
    'Database'


Relational Operations on Rows
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

All of these operations are easily defined in terms of the relational concepts
we already have: turn the row or rows into a single row relation with the same
header, perform the operation, and extract the single row from the result
relation.

And in fact, in the pure-Python version of dinsd they are implemented exactly
that way, which is why there aren't tests for the edge cases (we've already
tested them above).

In dinsd these operations on rows are only available as infix operators.
Otherwise we'd have to have functions with names like ``row_join``, which
doesn't really seem worthwhile.  This means we don't have an n-adic version of
row join, but if you need it you can use the Python ``accumulate`` function
from ``itertools``.

We'll use the following rows in the test examples::

    >>> rw1 = row(student_id=SID('S1'), name='Anne')
    >>> rw2 = row(student_id=SID('S1'), course_id=CID('C1'))

row rename::

    >>> rw1.rename(student_id='SID')
    row({'SID': SID('S1'), 'name': 'Anne'})

row projection::

    >>> rw1 >> {'name'}
    row({'name': 'Anne'})
    >>> rw1 << {'name'}
    row({'student_id': SID('S1')})

row extension::

    >>> rw1.extend(capname="name.upper()")
    row({'capname': 'ANNE', 'name': 'Anne', 'student_id': SID('S1')})

Extending by a constant expression is much more useful with rows, but
keep in mind that the value is a *string* expression or *function*, not
a Python expression::

    >>> rw1.extend(foo=1)
    Traceback (most recent call last):
        ...
    TypeError: 'int' object is not callable
    >>> rw1.extend(foo="1")
    row({'foo': 1, 'name': 'Anne', 'student_id': SID('S1')})

row join::

    >>> rw1 & rw2
    row({'course_id': CID('C1'), 'name': 'Anne', 'student_id': SID('S1')})

row compose::

    >>> rw1 + rw2
    row({'course_id': CID('C1'), 'name': 'Anne'})

We also have one function that *Tutorial D* does not.  The ``invert``
operator, ``~``, can also be applied to a row, producing a single
relation with the same header::

    >>> ~rw1
    rel({row({'name': 'Anne', 'student_id': SID('S1')})})

This gives a little more strength to the operator choice, since the operation
is functionally an inverse::

    >>> ~~rw1 == rw1
    True



Extended Example
----------------


Now that we've got all the basics under out belts, let's take a look at an
extended example of actually using the relational algebra in dinsd, by
working the exercises from the end of chapter 5 of AIRDT.  To do this
we'll need the relations described in figure 4.13::

    >>> S = rel(sn=str, sname=str, status=str, city=str)(
    ...        ('sn',  'sname',    'status',   'city'),
    ...        ('S1',  'Smith',     20,        'London'),
    ...        ('S2',  'Jones',     10,        'Paris'),
    ...        ('S3',  'Blake',     30,        'Paris'),
    ...        ('S4',  'Clark',     20,        'London'),
    ...        ('S5',  'Adams',     30,        'Athens'),
    ...        )
    >>> print(S.display('sn', 'sname', 'status', 'city'))
    +----+-------+--------+--------+
    | sn | sname | status | city   |
    +----+-------+--------+--------+
    | S1 | Smith | 20     | London |
    | S2 | Jones | 10     | Paris  |
    | S3 | Blake | 30     | Paris  |
    | S4 | Clark | 20     | London |
    | S5 | Adams | 30     | Athens |
    +----+-------+--------+--------+
    >>> SP = rel(sn= str, pn=str, qty=int)(
    ...          ('sn', 'pn', 'qty'),
    ...          ('S1', 'P1', 300),
    ...          ('S1', 'P2', 200),
    ...          ('S1', 'P3', 400),
    ...          ('S1', 'P4', 200),
    ...          ('S1', 'P5', 100),
    ...          ('S1', 'P6', 100),
    ...          ('S2', 'P1', 300),
    ...          ('S2', 'P2', 400),
    ...          ('S3', 'P2', 200),
    ...          ('S4', 'P2', 200),
    ...          ('S4', 'P4', 300),
    ...          ('S4', 'P5', 400),
    ...          ('S5', 'P2', 100),
    ...          )
    >>> print(SP.display('sn', 'pn', 'qty'))
    +----+----+-----+
    | sn | pn | qty |
    +----+----+-----+
    | S1 | P1 | 300 |
    | S1 | P2 | 200 |
    | S1 | P3 | 400 |
    | S1 | P4 | 200 |
    | S1 | P5 | 100 |
    | S1 | P6 | 100 |
    | S2 | P1 | 300 |
    | S2 | P2 | 400 |
    | S3 | P2 | 200 |
    | S4 | P2 | 200 |
    | S4 | P4 | 300 |
    | S4 | P5 | 400 |
    | S5 | P2 | 100 |
    +----+----+-----+
    >>> P = rel(pn=str, pname=str, color=str, weight=float, city=str)(
    ...         ('pn', 'pname',   'color',    'weight',   'city'),
    ...         ('P1', 'Nut',     'Red',       12.0,      'London'),
    ...         ('P2', 'Bolt',    'Green',     17.0,      'Paris'),
    ...         ('P3', 'Screw',   'Blue',      17.0,      'Oslo',),
    ...         ('P4', 'Screw',   'Red',       14.0,      'London'),
    ...         ('P5', 'Cam',     'Blue',      12.0,      'Paris'),
    ...         ('P6', 'Cog',     'Red',       19.0,      'London'),
    ...         )
    >>> print(P.display('pn', 'pname', 'color', 'weight', 'city'))
    +----+-------+-------+--------+--------+
    | pn | pname | color | weight | city   |
    +----+-------+-------+--------+--------+
    | P1 | Nut   | Red   | 12.0   | London |
    | P2 | Bolt  | Green | 17.0   | Paris  |
    | P3 | Screw | Blue  | 17.0   | Oslo   |
    | P4 | Screw | Red   | 14.0   | London |
    | P5 | Cam   | Blue  | 12.0   | Paris  |
    | P6 | Cog   | Red   | 19.0   | London |
    +----+-------+-------+--------+--------+

I added ('S5', 'P2') in the SP table so that exercise (e) below would have at
least one row in the result.

(a) Get the total number of parts supplied by supplier S1::

    >>> sum(SP.where("sn == 'S1'").compute('qty'))
    1300

(b) Get supplier numbers for suppliers whose city is first in the alphabetic
list of such cities::

    >>> with ns(S=S):
    ...     r = S.where("city == min(S.compute('city'))") >> {'sn'}
    >>> print(r)
    +----+
    | sn |
    +----+
    | S5 |
    +----+

(c) Get part numbers for parts supplied by all suppliers in London::

    >>> print((S.where("city == 'London'") & SP) >> {'pn'})
    +----+
    | pn |
    +----+
    | P1 |
    | P2 |
    | P3 |
    | P4 |
    | P5 |
    | P6 |
    +----+

(d) Get supplier numbers and names for suppliers who supply all the Red parts
(I made it Red instead of purple so there would be some results)::

    >>> print((P.where("color=='Red'") >> {'pn'} & SP & S) >> {'sn', 'sname'})
    +----+-------+
    | sn | sname |
    +----+-------+
    | S1 | Smith |
    | S2 | Jones |
    | S4 | Clark |
    +----+-------+

(e) Get all pairs of supplier numbers Sx and Sy such that Sx and Sy supply
exactly the same set of parts each::

    >>> sns = S >> {'sn'}
    >>> with ns() as names:
    ...     names['pSP'] = SP >> {'sn', 'pn'}
    ...     pairs = (sns & sns.rename(sn='sn2')).where(
    ...                 "pSP + ~row(sn=sn) == pSP + ~row(sn=sn2) and sn < sn2")
    >>> print(pairs)
    +----+-----+
    | sn | sn2 |
    +----+-----+
    | S3 | S5  |
    +----+-----+

(f) Write a truth-valued expression to determine whether all supplier names
are unique in S::

    >>> len(S) == len(S >> {'sname'})
    True

(g) Write a truth-valued expression to determine whether all part numbers
appearing in SP also appear in P::

    >>> SP >> {'pn'} == P >> {'pn'}
    True
