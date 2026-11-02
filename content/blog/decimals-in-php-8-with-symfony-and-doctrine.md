+++
draft = false
title = "Decimals in PHP 8 with Symfony and Doctrine"
date = '2026-11-02'
description = "PHP 8.4's BcMath\\Number finally gives exact decimals a real type: an immutable object with arithmetic operators that can travel intact from the database to the form, without ever becoming a float."
tags = [
  "php",
  "symfony",
  "doctrine",
  "mysql",
  "decimal",
]
+++

PHP has never had a true decimal type. You could use a `float` and accept rounding errors, or keep the value in a
numeric string and work with it using `bcadd()` and similar functions. PHP 8.4 adds a third option: **BcMath\Number**,
an immutable object that holds an exact decimal and works with the regular arithmetic operators.

A type is only useful if it reaches every layer. Doctrine can map a `DECIMAL` database column straight to a `Number`,
and Symfony components are starting to accept it too. This article starts with why floats fail, then follows a
`Number` through the database, the presentation layer and Symfony.

## Part 1: The root of the problem

```php
var_dump(0.1 + 0.2); // 0.30000000000000004
```

The logical answer would be `0.3`, but in computer science (and in PHP as well), the result is not exactly `0.3`, but
`0.30000000000000004`.

### Why does this happen?

> [!NOTE]
> Radix is a Latin word for "root". Root can be considered a synonym for base, in the arithmetical sense.
> \- [Wikipedia](https://en.wikipedia.org/wiki/Radix)

- **Numeral system:**
  The most common numeral system used by humans today, the
  [Hindu–Arabic numeral system](https://en.wikipedia.org/wiki/Hindu–Arabic_numeral_system), is a **radix 10**
  ([decimal](https://en.wikipedia.org/wiki/Decimal)) positional numeral system. Computers, on the other hand, natively
  support [binary](https://en.wikipedia.org/wiki/Binary_number) arithmetic - based on the **radix 2**.
- **Infinite fractions:**
  The root of the problem is that conversion between number systems is not always perfect... so [hidden inaccuracies
  appear](https://martinfowler.com/bliki/HiddenPrecision.html). Just as `1/3` in radix 10 results in an infinite
  repeating decimal (`0.333...`), many numbers that appear finite in radix 10 (like `0.1` or `0.7`) become infinite
  repeating fractions when converted to radix 2. They have **no exact binary equivalent**, so the CPU stores the nearest
  representable value instead.
- **Rounding error:**
  Memory is finite, so the CPU cannot store an infinite sequence. It rounds the number, and some precision is lost.
  When we add two rounded values, the tiny errors combine and become visible.

> [!NOTE]
> The `PHP_FLOAT_DIG` constant holds the number of decimal digits that can be rounded into a float and back without
> loss of precision.

PHP, like many languages, uses the **IEEE 754** standard for its `float` type (and its alias, `double`). Almost all CPUs
use this format for non-integer numbers. The hardware does the arithmetic, so it is fast, but the format is binary.
[The PHP documentation](https://www.php.net/manual/en/language.types.float.php) warns about this:

> Floating point numbers have limited precision. Although it depends on the system, PHP typically uses the IEEE 754
> double precision format, which will give a maximum relative error due to rounding in the order of `1.11e-16`. Non
> elementary arithmetic operations may give larger errors, and, of course, error propagation must be considered when
> several operations are compounded.
>
> Additionally, rational numbers that are exactly representable as floating point numbers in base 10, like `0.1` or
> `0.7`, do not have an exact representation as floating point numbers in base 2, which is used internally, no matter
> the size of the mantissa. Hence, they cannot be converted into their internal binary counterparts without a small loss
> of precision. This can lead to confusing results: for example, `floor((0.1+0.7)*10)` will usually return `7` instead
> of the expected `8`, since the internal representation will be something like `7.9999999999999991118...`.
>
> **So never trust floating number results to the last digit, and do not compare floating point numbers directly for
> equality.**

```php
var_dump(0.1 + 0.7 === 0.8);        // bool(false)
var_dump((0.1 + 0.7) * 10);         // float(7.999999999999999)
var_dump((int) ((0.1 + 0.7) * 10)); // int(7) <- The classic!

// It hides well, too:
echo 0.1 + 0.2;               // 0.3 <- The float value will be coerced to string.
printf("%.20f\n", 0.1 + 0.2); // 0.30000000000000004441
```

> **Accuracy problems**
>
> The fact that floating-point numbers cannot accurately represent all real numbers, and that floating-point operations
> cannot accurately represent true arithmetic operations, leads to many surprising situations. This is related to the
> finite [precision](https://en.wikipedia.org/wiki/Precision_(computer_science)) with which computers generally
> represent
> numbers.
> \- [Wikipedia](https://en.wikipedia.org/wiki/Floating-point_arithmetic#Accuracy_problems)

For general statistics, this error might not matter. **For financial data, scientific calculations, or any domain that
needs exact results, it does.** Even when the inputs are exact floats, the arithmetic can still introduce
[round-off error](https://en.wikipedia.org/wiki/Round-off_error) into the result.

### Floating-point error mitigation

> Floating-point error mitigation is the minimization of errors caused by the fact that
> [real numbers](https://en.wikipedia.org/wiki/Real_numbers) cannot, in general, be accurately represented in a fixed
> space. By definition, [floating-point error](https://en.wikipedia.org/wiki/Floating-point_error) cannot be eliminated,
> and, at best, can only be managed.
> \- [Wikipedia](https://en.wikipedia.org/wiki/Floating-point_error_mitigation)

Two of the mitigation techniques matter here:

> **Variable-length arithmetic**
>
> [Variable length arithmetic](https://en.wikipedia.org/wiki/Arbitrary-precision_arithmetic) represents numbers as a
> string of digits of a variable's length limited only by the memory available. Variable-length arithmetic operations
> are considerably slower than fixed-length format floating-point instructions. When high performance is not a
> requirement, but high precision is, variable length arithmetic can prove useful, though the actual accuracy of the
> result may not be known. [...]
>
> **Choice of a different radix**
>
> Changing the radix, in particular from binary to decimal, can help to reduce the error and better control the rounding
> in some applications, such as financial applications.
> \- [Wikipedia](https://en.wikipedia.org/wiki/Floating-point_error_mitigation)

The [IEEE 754-1985](https://en.wikipedia.org/wiki/IEEE_754-1985) is a historic industry standard for representing
floating-point numbers in computers, officially adopted in 1985 and superseded in 2008
by [IEEE 754-2008](https://en.wikipedia.org/wiki/IEEE_754-2008_revision).

- The **IEEE 754-1985** standard represents numbers in binary, providing definitions for four levels of precision, of
  which the two most commonly used are:
    - **Single-precision floating-point format (`float32`/`float`)**: The 32-bit base-2 format is officially referred to
      as `binary32`.
    - **Double-precision floating-point format (`float64`/`double`)**: The 64-bit base-2 format is officially referred
      to as `binary64`.
- And the **IEEE 754-2008** standard defines 32-, 64- and
  128-bit [Decimal floating-point (DFP)](https://en.wikipedia.org/wiki/Decimal_floating_point) representations.

| Radix   | Fixed length                   | Variable length              |
|---------|--------------------------------|------------------------------|
| Binary  | binary32/binary64              | GNU Multiple Precision (GMP) |
| Decimal | decimal32/decimal64/decimal128 | BCMath                       |

**Variable length alone is not a solution.** Financial applications need the decimal number system.
[GMP](https://en.wikipedia.org/wiki/GNU_Multiple_Precision_Arithmetic_Library) is arbitrary-precision and fast, and...
it will not help you with money, because it's still binary (and integer-only besides). "More digits" is a mitigation,
not a fix.

> [!WARNING]
> Extra precision does not fix the radix: `1/3` doesn't become exact in decimal if you write more digits, and `0.1`
> doesn't become exact in binary if you allocate more bits.

Most floating-point systems use base 2, and some use base 10 (DFP). Most hardware has no native base-10 arithmetic, so
decimal math runs in software. **You trade performance for accuracy.** A DFP library would work, but in PHP the
[BCMath extension](https://www.php.net/manual/en/book.bc.php) is the obvious, pragmatic choice.

### Fixed-point arithmetic

Another approach is **fixed-point arithmetic**:

> In [computing](https://en.wikipedia.org/wiki/Computing), **fixed-point** is a method of representing
> [fractional](https://en.wikipedia.org/wiki/Fraction_(mathematics)) (non-integer) numbers by storing a fixed number of
> digits of their fractional part. [Dollar](https://en.wikipedia.org/wiki/US_dollar) amounts, for example, are often
> stored with exactly two fractional digits, representing the [cents](https://en.wikipedia.org/wiki/Cent_(currency))
> (1/100 of a dollar). More generally, the term may refer to representing fractional values as integer multiples of some
> fixed small unit, e.g., a fractional amount of hours as an integer multiple of ten-minute intervals. Fixed-point
> number representation is often contrasted to the more complicated and computationally demanding
> [floating-point representation](https://en.wikipedia.org/wiki/Floating-point_representation).
> \- [Wikipedia](https://en.wikipedia.org/wiki/Fixed-point_arithmetic)

For example, `10.95` dollars becomes `1095` cents. Amounts in zero-decimal currencies, such as the Japanese yen, stay
unchanged.

Here's a naive PHP implementation:

```php
final readonly class Decimal
{
    public function __construct(
        private int $unscaled, // 1095
        private int $scale,    // 2 -> 10.95
    ) {}
    // ...
}
```

Note that a PHP `int` is bounded by
[`PHP_INT_MAX`](https://www.php.net/manual/en/reserved.constants.php#constant.php-int-max), which brings
[a new problem: overflow](https://en.wikipedia.org/wiki/Integer_overflow). For this reason, libraries that use minor
units still reach for big number backends.

> [!TIP]
> Java's `BigDecimal` is this exact design, without the overflow. The
> [OpenJDK implementation](https://github.com/openjdk/jdk/blob/master/src/java.base/share/classes/java/math/BigDecimal.java)
> keeps the unscaled value in a plain `long` while it fits, and falls back to a `BigInteger` only when it doesn't -
> a technique often referred to as the "compact representation".

## Part 2: BCMath and the new `BcMath\Number` object

BCMath is PHP's binding to the algorithms of
[`bc`, the "Basic Calculator"](https://en.wikipedia.org/wiki/Bc_(programming_language)) - an **arbitrary-precision**
calculator language that dates back to the 1970s and is part of **POSIX**. PHP ships it as a C extension.

> [!NOTE]
> ["Arbitrary precision"](https://en.wikipedia.org/wiki/Arbitrary-precision_arithmetic) means a fixed-length register
> does not limit the numbers. Only the available memory limits the number of digits.

### The old procedural API

Until PHP 8.4, BCMath was purely procedural, so the code looked like this:

```php
$price    = '19.99';
$quantity = '3';
$taxRate  = '0.27'; // Hungarian VAT

$net   = bcmul($price, $quantity, 2); // '59.97'
$tax   = bcmul($net, $taxRate, 2);    // '16.19'
$gross = bcadd($net, $tax, 2);        // '76.16'
```

It works, but it is painful to read. Every value is a bare string with no type safety, and you repeat the scale
argument on every call.

### PHP 8.4 turns the functions into an object: `BcMath\Number`

[The "Support object type in BCMath" RFC](https://wiki.php.net/rfc/support_object_type_in_bcmath) by Saki Takamachi
introduced [`BcMath\Number`](https://www.php.net/manual/en/class.bcmath-number.php) in 2024, which wraps that procedural
functionality in an object. PHP arrived late, but `BcMath\Number` fits the standard definition of a
**[Decimal Data Type](https://en.wikipedia.org/wiki/Decimal_data_type)**. Similar to Java's `java.math.BigDecimal` or
Python's `decimal.Decimal`, it represents non-repeating decimal fractions (like `0.1` and `−1.17`) without rounding,
and allows arithmetic calculations with them.

Its signature tells you most of what you need to know:

```php
namespace BcMath;

final readonly class Number implements \Stringable
{
    public string $value;
    public int $scale;

    public function __construct(string|int $num)
    public function add(Number|string|int $num, ?int $scale = null): Number
    public function ceil(): Number
    public function compare(Number|string|int $num, ?int $scale = null): int
    public function div(Number|string|int $num, ?int $scale = null): Number
    public function divmod(Number|string|int $num, ?int $scale = null): array
    public function floor(): Number
    public function mod(Number|string|int $num, ?int $scale = null): Number
    public function mul(Number|string|int $num, ?int $scale = null): Number
    public function pow(Number|string|int $exponent, ?int $scale = null): Number
    public function powmod(Number|string|int $exponent, Number|string|int $modulus, ?int $scale = null): Number
    public function round(int $precision = 0, \RoundingMode $mode = \RoundingMode::HalfAwayFromZero): Number
    public function sqrt(?int $scale = null): Number
    public function sub(Number|string|int $num, ?int $scale = null): Number
    public function __toString(): string
    public function __serialize(): array
    public function __unserialize(array $data): void
}
```

Four design decisions matter here:

1. **It is OOP:** The procedural functions are wrapped in an object, so the number carries its own scale and behaviour
   instead of being an anonymous string passed around between global functions.
2. **It is a [Value Object](https://martinfowler.com/bliki/ValueObject.html):** The class is `final readonly`. Every
   operation returns a **new** instance, so there are no side effects. That is exactly what you want for money: two
   amounts of `19.99` are interchangeable, and no code path can reach into your invoice and change its total from
   underneath you.
3. **Values are stored as strings:** The `$value` is a `string`, not a `float`. Precision survives construction,
   serialization, and the round trip to the database.
4. **It supports operator overloading:** You can use operators directly on the objects. Userland PHP classes can't do
   this - the extension implements it in the engine.

```php
use BcMath\Number;

$price    = new Number('19.99');
$quantity = new Number('3');
$taxRate  = new Number('0.27');

$net   = $price * $quantity;
$gross = $net + ($net * $taxRate);

echo $gross;           // 76.1619 <- Stringable
echo $gross->value;    // '76.1619'
echo $gross->scale;    // 4
echo $gross->round(2); // 76.16
```

> [!NOTE]
> Scientific notation is not supported yet. See [php/php-src#18068](https://github.com/php/php-src/pull/18068).

Two behaviors are worth noting:

1. **Scale value:** Unlike the procedural functions, [the class ignores the `bcmath.scale` INI
   directive](https://www.php.net/manual/en/bc.configuration.php#ini.bcmath.scale). It computes the scale itself,
   unless you pass the `scale` parameter to the calculation methods.
2. **Identity comparison:** The comparison operators are overloaded, but the
   [identity comparison (`===`)](https://www.php.net/manual/en/language.oop5.object-comparison.php) works as it does for
   any other object. It returns `false` for two distinct `Number` instances that hold the same value. For value equality,
   use the `==` operator, or better yet,
   [the spaceship operator (`<=>`)](https://www.php.net/manual/en/migration70.new-features.php#migration70.new-features.spaceship-op),
   which calls the `compare()` method. If you need an exact string comparison that also takes the scale into account,
   compare the results of `__toString()`:

```php
use BcMath\Number;

$a = new Number('34.93');
$b = new Number('23.93') + new Number('11');

var_dump($a == $b);        // true
var_dump($a === $b);       // false <- Different objects with different identity
var_dump($a <=> $b);       // int(0)
var_dump($a->compare($b)); // int(0)

$c = new Number('34.9300');

var_dump($a->__toString() === $b->__toString()); // true
var_dump($a->__toString() === $c->__toString()); // false
```

Keep that identity comparison detail in mind, because it comes back in Part 3, from a different direction.

### Decimal is not [rational](https://en.wikipedia.org/wiki/Rational_data_type)

A Decimal type is exact for fractions whose denominator is a power of ten, and approximate for everything else:

```php
var_dump((new Number('1'))->div(3, 10)); // '0.3333333333' <- Exactness ends here...
```

Division is where every Decimal type stops being exact, because the result may not be a decimal fraction at all. This is
why **splitting a bill is a policy question and not an arithmetic one**, and why
[Money Pattern](https://martinfowler.com/eaaCatalog/money.html) ships `allocate()` rather than expecting you to divide.

### If you're not on PHP 8.4 yet

The RFC author also published a **polyfill**:
[`saki/bcmath-number-polyfill`](https://github.com/SakiTakamachi/BcMath-Number-Polyfill).

```bash
composer require saki/bcmath-number-polyfill
```

It backports the class to PHP 8.2 and 8.3 (the BCMath extension is still required). It does not define the class if a
native `BcMath\Number` already exists, so you can keep it in `composer.json` across an upgrade. The one thing it
**cannot** provide is operator overloading, because that is an engine feature. Polyfilled code has to use method
chaining (`->add(...)->mul(...)`).

> [!TIP]
> Once you've switched to `Number`, make sure no `float` sneaks back in. The
> [`roave/no-floaters`](https://github.com/Roave/no-floaters) PHPStan plugin reports `float` properties, parameters,
> return types and assignments, and with `disallowFloatsEverywhere: true` it reports every expression that produces a
> `float`.

## Part 3: The database layer

This part covers only MySQL, because that is the database I work with daily.

### `DECIMAL` is fixed-point, `FLOAT` and `DOUBLE` are not

MySQL works like PHP here. `FLOAT` and `DOUBLE` are approximate binary types with the same rounding behavior as PHP's
`float`, because they use the same **IEEE 754 representation**:

> Floating-point numbers sometimes cause confusion because they are approximate and not stored as exact values.
> A floating-point value as written in an SQL statement may not be the same as the value represented internally.
> Attempts to treat floating-point values as exact in comparisons may lead to problems. They are also subject to
> platform or implementation dependencies. The `FLOAT` and `DOUBLE` data types are subject to these issues. For
> `DECIMAL` columns, MySQL performs operations with a precision of 65 decimal digits, which should solve most common
> inaccuracy problems.
> \- [Problems with Floating-Point Values, MySQL Reference Manual](https://dev.mysql.com/doc/refman/9.7/en/problems-with-float.html)

[MySQL's `DECIMAL` (and its synonym `NUMERIC`)](https://dev.mysql.com/doc/refman/9.7/en/fixed-point-types.html) uses
**fixed-point** storage, so the digits are stored exactly. In `DECIMAL(5,2)`, `5` is the precision and `2` is the
scale. The precision is the number of significant digits stored, and the scale is the number of digits after the
decimal point.

```sql
CREATE TABLE `employee`
(
    `id`     INT PRIMARY KEY AUTO_INCREMENT,
    `salary` DECIMAL(5, 2) NOT NULL -- Exact value
    -- `salary` DOUBLE NOT NULL     -- Approximate value
);
```

The MySQL documentation specifies the limits:

> The declaration syntax for a `DECIMAL` column is `DECIMAL(M,D)`.
> The ranges of values for the arguments are as follows:
> - `M` is the maximum number of digits (the precision). It has a range of 1 to 65.
> - `D` is the number of digits to the right of the decimal point (the scale). It has a range of 0 to 30 and must be no
>   larger than `M`.
>
> If `D` is omitted, the default is 0. If `M` is omitted, the default is 10.
>
> The maximum value of 65 for `M` means that calculations on `DECIMAL` values are accurate up to 65 digits. This limit
> of 65 digits of precision also applies to exact-value numeric literals, so the maximum range of such literals
> differs from before. (There is also a limit on how long the text of `DECIMAL` literals can be; see [Section 14.26.3,
> "Expression Handling"](https://dev.mysql.com/doc/refman/9.7/en/precision-math-expressions.html).)
> \- [DECIMAL Data Type Characteristics, MySQL Reference Manual](https://dev.mysql.com/doc/refman/9.7/en/precision-math-decimal-characteristics.html)

A good practical comparison of the three is [this write-up on MySQL `DECIMAL` vs `DOUBLE` vs
`FLOAT`](https://laraveldaily.com/post/mysql-decimal-vs-double-vs-float).

### Doctrine maps `decimal` to `string`, on purpose

Doctrine DBAL's `decimal` type returns a PHP **`string`**, not a `float`,
and [the documentation explains why](https://www.doctrine-project.org/projects/doctrine-dbal/en/current/reference/types.html#decimal):
converting to a `float` would only approximate the precision, and applications would then make false assumptions based
on that approximation.

This is the right call, and it's the source of the single most common decimal bug in Symfony/Doctrine applications.

### The type-hint trap: phantom `UPDATE` statements

Consider this entity, taken
from [d1823's excellent article on type hints and Doctrine's change detection](https://1823.pl/articles/importance-of-type-hints-with-doctrines-change-detection):

```php
#[ORM\Entity]
final class Book
{
    #[ORM\Id]
    #[ORM\GeneratedValue]
    #[ORM\Column]
    public int $id;

    #[ORM\Column(type: Types::DECIMAL)]
    public float $price; // <- The bug
}
```

The column is `decimal`, and the class property is `float`. Doctrine boots, maps the entity, inserts rows, and reads
them back. Nothing warns you at runtime.

Now persist a `Book`, clear the `EntityManager`, `find()` it again, and `flush()` without touching anything. Doctrine
issues an `UPDATE` statement, because when you `flush()`, the
[Unit of Work](https://www.doctrine-project.org/projects/doctrine-orm/en/current/reference/unitofwork.html) iterates its
[identity map](https://www.doctrine-project.org/projects/doctrine-orm/en/current/reference/working-with-objects.html#entities-and-the-identity-map)
and compares the current state of each entity against the snapshot taken when it was loaded.
That comparison is an **identity comparison**:

```php
// https://github.com/doctrine/orm/blob/3.7.x/src/UnitOfWork.php#L600 - simplified:
public function computeChangeSet(ClassMetadata $class, object $entity): void
    // ...
    foreach ($actualData as $propName => $actualValue) {
        // ...
        $orgValue = $originalData[$propName];
        // ...
    
        // skip if value haven't changed
        if ($orgValue === $actualValue) {
            continue;
        }
    
        // ...otherwise it's a change!
    }
    // ...
}
```

Here is what happens during hydration: the `decimal` type gives Doctrine the string `'5.00'`, and Doctrine assigns it
to the property. The `float` type hint coerces it to `5.0`, but the snapshot still stores the original string. The
expression `'5.00' === 5.0` is `false`, so every `flush()` sends an `UPDATE` statement for an unchanged row.

These redundant `UPDATE` statements take write locks you don't need, raise the risk of deadlocks, and make the
database work harder (and cost more) than it has to.

The same behaviour has been reported repeatedly, e.g.
in [doctrine/orm#8867 - "Field with type float and orm type decimal is wrongly detected as changed on update"](https://github.com/doctrine/orm/issues/8867).

### How to catch them

Doctrine's schema validator was **extended to detect exactly this mismatch**
in [doctrine/orm#10946](https://github.com/doctrine/orm/pull/10946). On a reasonably current version you get:

```shell
$ bin/console doctrine:schema:validate

Mapping
-------

 [FAIL] The entity-class App\Entity\Book mapping is invalid:
 * The field 'App\Entity\Book#price' has the property type 'float' that differs
   from the metadata field type 'string' returned by the 'decimal' DBAL type.
```

> [!TIP]
> Run `doctrine:schema:validate` in your CI workflow. It is the cheapest guard against this class of bug.

There are three ways to resolve the mismatch:

1. Keep the column type and the entity's property type in sync (declare the property as `string`).
2. Drop the column type and let it be inferred from the property type.
3. Drop the property type and let it be inferred from the column type.

Each has trade-offs, so the honest answer is "it depends", but the first option is the one that survives code review:

```php
#[ORM\Column(type: Types::DECIMAL, precision: 10, scale: 2)]
public string $price;
```

Which is *correct*, but *miserable to work with*. You just can't add two prices. You're back to `bcadd()` on raw
strings, in your domain model, in 2026!

### DBAL 4.3: the `number` type

[doctrine/dbal#6686](https://github.com/doctrine/dbal/pull/6686) closes this gap. It implements the feature request in
[doctrine/dbal#6644](https://github.com/doctrine/dbal/issues/6644). Since **DBAL 4.3**, there is a [
`number` type](https://www.doctrine-project.org/projects/doctrine-dbal/en/current/reference/types.html#number):

> Maps and converts numeric data with fixed-point precision. This type behaves like `decimal`, but maps to
> `BCMath\Number` value objects instead of strings. Requires PHP 8.4 or newer and the bcmath extension.

So the entity finally becomes something you'd want to write:

```php
<?php

declare(strict_types=1);

namespace App\Entity;

use BcMath\Number;
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
final class Invoice
{
    #[ORM\Id]
    #[ORM\GeneratedValue]
    #[ORM\Column]
    private ?int $id = null;

    public function __construct(
        #[ORM\Column(type: Types::NUMBER, precision: 12, scale: 2)]
        private Number $netAmount,
        #[ORM\Column(type: Types::NUMBER, precision: 5, scale: 4)]
        private Number $vatRate,
    ) {
    }

    public function getVatAmount(): Number
    {
        return ($this->netAmount * $this->vatRate)->round(2);
    }

    public function getGrossAmount(): Number
    {
        return $this->netAmount + $this->getVatAmount();
    }
}
```

Exact arithmetic, exact storage, no strings, no procedural BCMath API, and a property type that matches the column type
so the schema validator stays quiet.

### A note on Doctrine's changeset computation

`BcMath\Number` is an **immutable Value Object**, so it avoids the classic mutation trap of `DateTime`. When you modify
a `DateTime` in place, the change slips past Doctrine's
[reference-based tracking](https://www.doctrine-project.org/projects/doctrine-orm/en/current/cookbook/working-with-datetime.html#datetime-changes-are-detected-by-reference):

```php
$orgValue = new \DateTime();
$actualValue = $orgValue;
$actualValue->modify('+1 day');

var_dump($orgValue === $actualValue); // true
```

Immutability brings its own changeset quirk, as a discussion on
[doctrine/orm#11781](https://github.com/doctrine/orm/pull/11781) points out. `DateTime` has behaved the same way for
years, as reported in [doctrine/orm#7583](https://github.com/doctrine/orm/issues/7583):

> We kind of have the same problem here as we already do with `DateTime` objects. If you don't modify a property that
> stores a number, it's fine. But if you set that property to a new number object, it will be detected as changed even
> if the new number object is equal to the old one.

Change detection for object types is [a known weak spot](https://github.com/doctrine/orm/issues/11910), because it
uses the **identity operator**. To prevent redundant database writes, check whether the new `BcMath\Number` value
differs from the current value before assigning it.

```php
public function changeNetAmount(Number $netAmount): void
{
    // Compares by value, so `10.5` and `10.50` are equal.
    if (0 === $this->netAmount->compare($netAmount)) {
        return;
    }

    $this->netAmount = $netAmount;
}
```

## Part 4: The presentation layer

You have an exact `BcMath\Number` in your domain and an exact `DECIMAL` column in your database. Now you have to show
`76.16` to a European user as `76,16 €` and to an American one as `$76.16`.

`BcMath\Number` has no `format()` method of its own. An early draft of the RFC had one, but
[the follow-up "Fix up BCMath Number Class" RFC](https://wiki.php.net/rfc/fix_up_bcmath_number_class) removed it before
PHP 8.4 shipped. One reason was that it handled only a limited set of notation conventions.

### `money_format()` is gone

The `money_format()` function
was [deprecated in 7.4 and removed in PHP 8.0](https://www.php.net/manual/en/function.money-format.php).
It was never a good tool anyway. It depended on the C-level locale set by `setlocale()`, which is process-wide and not
thread-safe.

### Use `NumberFormatter` instead

The replacement is [`NumberFormatter`](https://www.php.net/manual/en/class.numberformatter.php) from the `intl`
extension:

```php
$amount = 76.16;

$hu = new \NumberFormatter('hu_HU', \NumberFormatter::CURRENCY);
echo $hu->formatCurrency($amount, 'HUF'); // 76,16 Ft

$us = new \NumberFormatter('en_US', \NumberFormatter::CURRENCY);
echo $us->formatCurrency($amount, 'USD'); // $76.16

$de = new \NumberFormatter('de_DE', \NumberFormatter::CURRENCY);
echo $de->formatCurrency($amount, 'EUR'); // 76,16 €

$ja = new \NumberFormatter('ja_JP', \NumberFormatter::CURRENCY);
echo $ja->formatCurrency(7616, 'JPY'); // ￥7,616
```

`NumberFormatter` is a thin wrapper over the [ICU](https://icu.unicode.org/) library, and ICU holds all of these rules.
There's a good practical introduction in [this article on using
`NumberFormatter` for localization and formatting](https://bmehul.com/articles/using-phps-numberformatter-for-localization-and-formatting).

The locale decides the decimal separator, the group separator, the symbol and its position, the spacing, and the number
of fraction digits. **Always set the locale explicitly.** The default comes from the ICU environment and differs
between your localhost, CI, and production.

### Cross the float boundary only after rounding

Here is the catch nobody mentions: `formatCurrency()` accepts a `float`.

```php
public function formatCurrency(float $amount, string $currency): string|false
```

So there is a `float` in your pipeline after all, right at the end. That is acceptable, but only if you handle it
deliberately:

```php
use BcMath\Number;

function format_number(Number $amount, string $currency, string $locale): string {
    $formatter = new \NumberFormatter($locale, \NumberFormatter::CURRENCY);
    // 1. Get the scale value.
    $scale = $formatter->getAttribute(\NumberFormatter::FRACTION_DIGITS);
    // 2. Round to the currency's precision exactly, in decimal.
    $rounded = $amount->round($scale);

    // 3. Only then cross into float, for display only.
    return $formatter->formatCurrency((float) $rounded->value, $currency);
}

echo format_number(new Number('76.1619'), 'HUF', 'hu_HU'); // 76,16 Ft
```

Rounding is the one place where money is created or destroyed, so it happens in exact decimal arithmetic with an
explicit rounding mode. The float conversion comes afterward, on a value with two decimal places, only to satisfy the
signature. A float is close enough to that value for display.

> [!NOTE]
> Rule of thumb: never round in float, and never let a float value flow backwards into your domain or your database.
> Formatting is a one-way door.

### PHP 8.5 exposes compact notation

For dashboards and analytics, you often want `1.2K` rather than `1,200`. JavaScript has had
[`Intl.NumberFormat`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/NumberFormat/NumberFormat)
with `notation: 'compact'` for years. PHP's `NumberFormatter` appeared not to support it.

As [Bogdan Ungureanu discovered](https://ungureanu.blog/2025/04/15/locale-aware-compact-number-formatting-in-php-with-numberformatter/),
ICU supported it all along, and PHP never exposed the constants. Passing the "magic" integer `14` worked as far back as
PHP 5. He opened [php/php-src#17975](https://github.com/php/php-src/pull/17975), it was merged, and **PHP 8.5** now
exposes two new constants:

```php
// PHP 8.5+
$short = new \NumberFormatter('en_US', \NumberFormatter::DECIMAL_COMPACT_SHORT);
echo $short->format(1200); // 1.2K

$long = new \NumberFormatter('en_US', \NumberFormatter::DECIMAL_COMPACT_LONG);
echo $long->format(1200); // 1.2 thousand
```

## Part 5: Symfony integrations

### Write your own form type

Symfony has no form type for `BcMath\Number`... yet. The obvious workaround is to put a `CallbackTransformer` on a
`NumberType` field, but [symfony/symfony#64562](https://github.com/symfony/symfony/issues/64562) shows why that is not
enough:

- The [`NumberType`](https://symfony.com/doc/current/reference/forms/types/number.html) works with floats internally,
  [even with `input: 'string'`](https://github.com/symfony/symfony/blob/8.1/src/Symfony/Component/Form/Extension/Core/Type/NumberType.php#L28-L37).
  Its `StringToFloatTransformer` and `NumberToLocalizedStringTransformer` turn the value into a `float` before your
  transformer runs.
- The [`MoneyType`](https://symfony.com/doc/current/reference/forms/types/money.html) has
  [the same problem](https://github.com/symfony/symfony/blob/8.1/src/Symfony/Component/Form/Extension/Core/Type/MoneyType.php#L32-L45),
  because its transformer extends the same `float`-based one.

The issue was closed by adding `BcMath\Number` support to the Validator component, not to the Form component. So for now
you need your own type. It must not extend `NumberType`, and it must convert directly between `string` and `Number`,
with no `float` in between.

> [!WARNING]
> The [`TextType`](https://symfony.com/doc/current/reference/forms/types/text.html) is not a good parent either. When
> the field is left empty, its own view transformer turns `null` back into an empty string, and that empty string ends
> up in a `?Number` property.

A minimal, illustrative version:

```php
use BcMath\Number;
use Symfony\Component\Form\AbstractType;
use Symfony\Component\Form\CallbackTransformer;
use Symfony\Component\Form\Exception\TransformationFailedException;
use Symfony\Component\Form\FormBuilderInterface;
use Symfony\Component\Form\FormInterface;
use Symfony\Component\Form\FormView;
use Symfony\Component\OptionsResolver\OptionsResolver;

// No `getParent()` method: so none of the float transformers are inherited.
final class BcMathNumberType extends AbstractType
{
    public function buildForm(FormBuilderInterface $builder, array $options): void
    {
        // A view transformer: the submitted string becomes a Number in one step, with no float in between.
        $builder->addViewTransformer(new CallbackTransformer(
            // Transform the Number to a string.
            static function (mixed $value): string {
                if (null === $value) {
                    return '';
                }

                if (!$value instanceof Number) {
                    throw new TransformationFailedException(\sprintf('Expected %s, got %s.', Number::class, \get_debug_type($value)));
                }

                return (string) $value;
            },
            // Transform the string back to a Number.
            static function (mixed $value) use ($options): ?Number {
                if (null === $value || '' === $value) {
                    return null;
                }

                if (!\is_string($value)) {
                    throw new TransformationFailedException(\sprintf('Expected a string, got %s.', \get_debug_type($value)));
                }

                // Weak guard: `is_numeric()` accepts '1e5' and ' 12', which the Number constructor rejects with a ValueError.
                // If the error message matters to your users, filter the input with your own RegEx first.
                if (!\is_numeric($value)) {
                    throw new TransformationFailedException(\sprintf('"%s" is not a valid number.', $value));
                }

                try {
                    $scale = $options['scale'] ?? null;
                    // Constructor throws a ValueError if the input is string and not a well-formed numeric string.
                    $number = new Number($value);

                    if (null !== $scale) {
                        $number = $number->round($scale);
                    }

                    return $number;
                } catch (\ValueError $e) {
                    throw new TransformationFailedException(\sprintf('"%s" is not a valid number.', $value), 0, $e);
                }
            },
        ));
    }

    public function configureOptions(OptionsResolver $resolver): void
    {
        $resolver->setDefaults([
            'compound' => false,
            'html5' => true,
            'scale' => null,
            'invalid_message' => 'Please enter a valid number.',
        ]);

        $resolver->setAllowedTypes('html5', 'bool');
        $resolver->setAllowedTypes('scale', ['null', 'int']);
    }

    public function buildView(FormView $view, FormInterface $form, array $options): void
    {
        if ($options['html5']) {
            // Like NumberType: the browser still submits the typed string, not a float.
            // The default `step="1"` would reject decimals.
            $view->vars['type'] = 'number';
            $view->vars['attr']['step'] ??= 'any';
        } else {
            // A plain text input.
            $view->vars['attr']['inputmode'] ??= 0 === $options['scale'] ? 'numeric' : 'decimal';
        }
    }
}
```

### Native `BcMath\Number` support across Symfony components

Three components already accept `BcMath\Number`:

- [Symfony 7.3 Serializer Improvements](https://symfony.com/blog/new-in-symfony-7-3-serializer-improvements): The
  **Serializer component** added a `NumberNormalizer` that converts `BcMath\Number` and `GMP` objects to and from
  strings or integers.
- [A Week of Symfony #1017](https://symfony.com/blog/a-week-of-symfony-1017-june-22-28-2026): The **JsonStreamer
  component** added support for `BcMath\Number` and `GMP` objects.
- [A Week of Symfony #1025](https://symfony.com/blog/a-week-of-symfony-1025-august-17-23-2026): The **Validator
  component** accepts `BcMath\Number` in comparison constraints, such as `EqualTo` and `GreaterThan`, and in the `Range`
  constraint.

### Currencies: the other half of money

An exact amount is only half of a price: it also needs a currency. Symfony has long shipped tools for that half, too:

- **Intl component:**
    - [Symfony 4.3 introduced](https://symfony.com/blog/new-in-symfony-4-3-simpler-access-to-intl-data) the
      [`Currencies`](https://symfony.com/doc/current/components/intl.html#currencies) class for
      [3-letter ISO 4217 codes](https://en.wikipedia.org/wiki/ISO_4217). PHP's
      [`ResourceBundle`](https://www.php.net/manual/en/class.resourcebundle.php) in the `intl` extension reads the same
      ICU data, but `Currencies` wraps it in static methods such as `getName()` and `getFractionDigits()`.
    - [As announced on the Symfony blog](https://symfony.com/blog/new-in-symfony-7-4-better-currency-filtering), Symfony
      7.4 filters out legacy currencies by default. It also adds ICU-backed methods that tell you whether a currency is
      legal tender and when it is valid.
- **Form component:**
    - **[`CurrencyType`](https://symfony.com/doc/current/reference/forms/types/currency.html):** A choice field for
      currencies.
    - **[`MoneyType`](https://symfony.com/doc/current/reference/forms/types/money.html):** An input for amounts. As
      shown above, it converts the value to a `float`.
- **Validator component:** The
  **[`Currency` constraint](https://symfony.com/doc/current/reference/constraints/Currency.html)** checks that a value is
  a valid ISO 4217 code.
