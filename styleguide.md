# L4Re Coding Style

Here we describe our preferred coding style for L4Re and the microkernel. The
structure and some of the rules are derived from the Linux kernel coding style.
In general a common coding style is about maintaining readability and
maintainability of a shared code base.

## Indentation

We don't use tabs but spaces for indentation. A new block of control is
indented by 2 spaces. To avoid double-indentation in switch statements it is
allowed to align the ***{*** and the ***case*** labels in the same column.
```
switch (action)
  {
  case 1:
    do_something();
    break;

  case 2:
    {
      do_another_thing();
      break;
    }
  default:
    break;
  }
```

### Namespaces

Namespaces must be started using:
```
namespace Xyz {

or

namespace Xyz
{
```

Multiple namespaces shall be started on a single line if they are closed
on a single line:
```
namespace Xyz { namespace Abc {
  ...
}} // namespace Xyz::Abc
```

The contents of a namespace must be indented by two spaces (except when
covering a whole file):
```
namespace Xyz { namespace Abc {
  This_is_some_statement;
}} // namespace Xzy::Abc
```

The contents of a namespace shall not be indented when the namespace covers a
complete file, this means the file may contain only preprocessor directives
before and after the namespace declaration:
```
#pragma once

#include <cstdio>

namespace X {

Code;
...

}
```

## Line Breaks

Our line length limit is 80 columns. Statements longer than 80 columns should
be broken into sensible chunks. One exception to this rule is to never break
user-visible strings because that breaks the ability to grep for them.

It is not allowed to put multiple statements on a single line. It is not
allowed to put multiple assignments on a single line.  The only exceptions is
for switch statements where case, statement and break may go on a single line
provided the line remains shorter than 80 characters.

```
switch (foo)
  {
  case 1: bar = "spring"; break;
  case 2: bar = "summer"; break;
  case 3: bar = "fall"; break;
  default: bar = "winter"; break
  }
```

A single statement that follows an if-condition is put on its own line.
```
if (condition)
  break;

if (condition2)
  return 0;
```

Don't leave blanks at the end of lines.

## Function Definitions

In function definitions outside classes the return type should go on its own
line

```
static int
Magic::create_something_wonderful(int random)
{
   return magic_happens(random);
}
```

## Braces

Opening and closing braces are placed on individual lines and they are indented
by 2 spaces.
```
if (foo)
  {
    bar();
    foo();
  }
else
  {
    baz();
    biz();
  }
```

One exception to this rule is when a new function, class, namespace or enum is
declared. Then the indentation is omitted.

The body of very short inline functions inside class definitions may go on a
single line. Add spaces around the braces:

```
class Foo
{
  int is_a_foo()
  { return true; }

};
```

The braces of an empty class body go on a new line below the class declaration
```
class Foo
{};
```

## Spaces

A space is inserted after most keywords. Exceptions to this rule are keywords
which are used like functions such as `sizeof` and `typeof`.

Use one space around the binary and ternary operators:
```=  +  -  <  >  *  /  %  |  &  ^  <=  >=  ==  !=  ?  :```

Don't put space after the unary operators:
```&  *  +  -  ~  !```

Don't put space before the postfix increment and decrement unary operators:
```++ --```

Don't put space after the prefix increment and decrement unary operators:
```++ --```

## Naming

We don't use camel-case. Names, types and constants start with a capital letter
and concatenate multiple words using ***_***.
```
namespace Util
{
class Br_manager
{
  enum Some_enum
  {
    Item_1,
    Item_2
  };
};
}
```

Local variable names should be descriptive and short. Using one-letter names is
considered bad as it hinders grep'ing.

Public members start with a lowercase letter. Private members start with an
underscore.
```
class Rect
{
public:
  l4_uint32_t x;
  l4_uint32_t y;

private:
  l4_uint32_t _area;
};
```

Usually you should use `class` for class definitions.  `struct` may be only
used for classes when all members of a class are public.

## Loops

```
while (condition)
  {
    multiple;
    statements;
  }

while (cond)
  single_statement;

for (unsigned i = 0; i < 10; ++i)
  {
    multiple;
    statements;
  }

for (...)
  single_statement;

do
  {
    code;
  }
while (condition);
```

## Endless Loops
```
for (;;)
  {
    code...;
  }
```

## C/C++ Integral Constants

Use `enum` types to define integral constants wherever possible. Avoid `static
const` and `static constexpr`.

## Commenting

In C files and C compatible header files we use `/* ... */`, in C++ code we use
`// ...` for comments.

## 3rd party code

If we modify 3rd party code e.g. a library or Linux we apply the coding style
rules from the 3rd party project.

# Doxygen Styles

* we use the `\command` Syntax
* we use doxygen's Markdown instead of HTML or doxygen commands

## Brief and Detailed Documentation

* we have autobrief enabled, so don't use `\brief`
* place an empty line after the brief description
* The detailed description is placed after the documentation of the return
  values. Put an emtpy line before the detailed description.

## Parameter Documentation

* parameter names must be left-aligned, as well as parameter descriptions
* at least two spaces between parameter name and description
* Parameter directions:
    * the default is *Input*, just use plain `\param`
    * `\param[in,out]` is used if the parameter is used to pass arguments into
      the function and the function also returns values through this parameter
    * `\param[out]` is used if the called function stores or returns data
      through this parameter

## Return Value Documentation

* place an empty line between parameter and return value documentation
* use `\retval` in case the function may return different return values and if
  possible describe each of them

## Example

```
/**
 * A brief description of the function
 *
 * \param         par1  Description of parameter 1.
 * \param[out]    par2  Description of parameter 2. Caller must allocate
 *                      memory.
 * \param[in,out] par3  Description of parameter 3.
 *
 * \retval 0           The function returned successfully.
 * \retval -L4_ENOMEM  Not enough memory to create the new kernel object.
 *
 * \pre Add expected conditions here. Expectations for specific parameters
 *      may also be added in the respective param section.
 *
 * \post Describe conditions here that the caller can expect after
 *       execution. Don't forget to mention when they are only valid
 *       if execution was successful.
 *
 * Longer detailed description of the function. This may span multiple
 * paragraphs. But the line length is limited to 80 characters.
 */
```

# L4Linux Coding Style

The L4Linux coding style mirrors the Linux coding style. However there are a
few inaccuracies that need to be made clear:

## Indentation

* When indenting parameters, you shall use *tabs* to reach the indent level of
  the function, and then use *spaces* to align the parameter to the first one.
  The reason is that if you used a mixture of tabs and spaces, the indenting of
  the function parameter breaks.