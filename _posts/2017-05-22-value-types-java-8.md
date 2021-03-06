---
layout: post
title: "Value Types in Java 8"
author: karl
date: 2017-05-22
comment: true
categories: [Articles]
published: true
noindex: false
---
This is a blog post based on the lightning talk I gave at the [Kats Conf 2](http://www.katsconf.com/) conference in Dublin. The inspiration came from the book [Functional Programming in Java](https://www.manning.com/books/functional-programming-in-java) by Pierre-Yves Saumont. Value types and how to implement them in Java is introduced early in the book and for good reason. value types are great for increasing type safety and getting the compiler to detect errors for you while you write code.
 
You can find the source code for this blog post [here](https://github.com/karlkyck/valuetypes-in-java8).

Let's begin with a simplified problem that has been solved using standard types and show how that can get you into trouble:

```java
public class InvoiceItem {
    public final String description;
    public final double amount;
    public final double vatAmount;
   
   public InvoiceItem(String description, double amount, double vatAmount) {
       this.description = description;
       this.amount = amount;
       this.vatAmount = vatAmount;
   }
}
```

## One possible solution

Imagine we have invoice items, each representing a piece of work that has been completed. There’s a description, the invoiceable amount, and the vat amount. Say we want to create an overall invoice with a total amount and a total VAT amount. Well here is one possible solution:

```java
InvoiceItem hardWork = new InvoiceItem(
        "Hard work!",
        40.1,
        5.0125
);
InvoiceItem evenHarderWork = new InvoiceItem(
        "Even Harder work!",
        60.2,
        10.535
);
InvoiceItem superHardWork = new InvoiceItem(
        "Super Harder work!",
        100.3,
        23.069
);

List<InvoiceItem> invoiceItems = Collections.unmodifiableList(
        Arrays.asList(
                hardWork,
                evenHarderWork,
                superHardWork
        ));

double totalAmount = invoiceItems
        .stream()
        .mapToDouble(value -> value.vatAmount)
        .sum();

double totalVatAmount = invoiceItems
        .stream()
        .mapToDouble(value -> value.amount)
        .sum();

System.out.println(String.format("Total invoice amount is %s", totalAmount));
System.out.println(String.format("Total invoice VAT amount is %s", totalVatAmount));
```

First we instantiate our `InvoiceItem` objects representing the work that has been completed:

```java
InvoiceItem hardWork = new InvoiceItem(
        "Hard work!",
        40.1,
        5.0125
);
```

Then we create an immutable list with all the `InvoiceItem` objects we want to use to calculate our totals:

```java
List<InvoiceItem> invoiceItems = Collections.unmodifiableList(
        Arrays.asList(
                hardWork,
                evenHarderWork,
                superHardWork
        ));
```

Next we sum our values using the [Stream API in Java 8](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html) by mapping our `Stream` of `InvoiceItem` objects to a `DoubleStream` upon which we can then call the method `sum()`:

```java
double totalAmount = invoiceItems
        .stream()
        .mapToDouble(value -> value.vatAmount)
        .sum();
```

The code compiles and runs OK and produces the following output:

```
Total invoice amount is 38.6165
Total invoice VAT amount is 200.6
```

This isn't the result we are expecting! 
The bug is subtle and the compiler cannot help us with these kinds of errors. 
We went wrong by mixing up the `amount` and `vatAmount` values in our calculation:
 
```java
double totalAmount = invoiceItems
        .stream()
        .mapToDouble(value -> value.vatAmount)
        .sum();

double totalVatAmount = invoiceItems
        .stream()
        .mapToDouble(value -> value.amount)
        .sum();
```

The only way to catch these kinds of bugs is to test our code. 
Tests are great, essential even, and remain the single most effective way to ensure that your code meets business and functional requirements. 
I cannot recommend enough adopting such practices as TDD and BDD. 
Tests are also prone to error. 
And other than [mutation testing](http://pitest.org/) who tests the tests?

Instead let's get the compiler to catch these kinds of errors for us. 
That way we can speed up development and get feedback while we write our code.

The problem is we are using variables of the same type for `amount`, and `vatAmount`. Let’s define the value types `Amount` and `VatAmount`.

## Defining our value types

```java
public class Amount {

    public final double value;

    public Amount(double value) {
        this.value = value;
    }
}
```

```java
public class VatAmount {

    public final double value;

    public VatAmount(double value) {
        this.value = value;
    }
}
```

Oops the problem remains as we can still mix up our types so we're not quite there yet:

```java
double totalAmount = invoiceItems
    .stream()
    .mapToDouble(item -> item.vatAmount.value)
    .sum();
    
double totalVatAmount = invoiceItems
    .stream()
    .mapToDouble(item -> item.amount.value)
    .sum();
```

## Strict value types
Let's tighten up our `Amount` and `VatAmount` value types by including a method for addition. 
We should make our `value` member variable private while we are at. 
The `ZERO` constant will come in useful for a [reduction](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html#reduce-U-java.util.function.BiFunction-java.util.function.BinaryOperator-) operation later. And `toString()` helps us with debugging:

```java
public class Amount {

    public static final Amount ZERO = new Amount(0);

    private final double value;

    public Amount(double value) {
        this.value = value;
    }

    public Amount add(Amount that) {
        return new Amount(value + that.value);
    }

    public String toString() {
        return Double.toString(value);
    }
}
```

```java
public class VatAmount {

    public static final VatAmount ZERO = new VatAmount(0);

    private final double value;

    public VatAmount(double value) {
        this.value = value;
    }

    public VatAmount add(VatAmount that) {
        return new VatAmount(value + that.value);
    }

    @Override
    public String toString() {
        return Double.toString(value);
    }
}
```

Now we refactor our `InvoiceItem` class to use our new value types:

```java
public class InvoiceItem {

    public final String description;
    public final Amount amount;
    public final VatAmount vatAmount;

    public InvoiceItem(String description, Amount amount, VatAmount vatAmount) {
        this.description = description;
        this.amount = amount;
        this.vatAmount = vatAmount;
    }
}
```

## Applying our changes to our solution
Let's revisit our solution to calculate the total amount and VAT amount and apply our changes:

```java
InvoiceItem hardWork = new InvoiceItem(
        "Hard work!",
        new Amount(40.1),
        new VatAmount(5.0125)
);
InvoiceItem evenHarderWork = new InvoiceItem(
        "Even Harder work!",
        new Amount(60.2),
        new VatAmount(10.535)
);
InvoiceItem superHardWork = new InvoiceItem(
        "Super Harder work!",
        new Amount(100.3),
        new VatAmount(23.069)
);

List<InvoiceItem> invoiceItems = Collections.unmodifiableList(
        Arrays.asList(
                hardWork,
                evenHarderWork,
                superHardWork
        ));

Amount totalAmount = invoiceItems
        .stream()
        .reduce(
                Amount.ZERO,
                (amount, invoiceItem) -> amount.add(invoiceItem.amount),
                Amount::add
        );

VatAmount totalVatAmount = invoiceItems
        .stream()
        .reduce(
                VatAmount.ZERO,
                (amount, invoiceItem) -> amount.add(invoiceItem.vatAmount),
                VatAmount::add
        );
```

There's no longer any ambiguity when applying our value types. 
The compiler will complain if you don't pass the right type during construction of the `InvoiceItem` objects:
  
```java
InvoiceItem hardWork = new InvoiceItem(
        "Hard work!",
        new Amount(40.1),
        new VatAmount(5.0125)
);
```

Now when we go to sum the amount we can no longer mix up our types. 
The `reduce()` method is Java 8's equivalent to a foldLeft operation. 
We are expecting an `Amount` type to be returned from the calculation and the compiler will complain if you don't use the correct type:  

```java
Amount totalAmount = invoiceItems
    .stream()
    .reduce(
        Amount.ZERO, 
        (amount, invoiceItem) -> amount.add(invoiceItem.amount), 
        Amount::add
    );
```

The `reduce()` method takes a starting value of type `Amount`, in this case starting at zero with `Amount.ZERO`. 
The method also takes a [bi-function](https://docs.oracle.com/javase/8/docs/api/java/util/function/BiFunction.html) that takes the accumulator value of type `Amount` and the next `InvoiceItem` object in the `Stream` and returns an object of type `Amount`. 

Inside this function we add the accumulator `Amount` parameter to the `Amount` field of the `InvoiceItem` parameter thus producing and returning an `Amount` type. 
The final parameter to the `reduce()` method takes a `bi-function` that takes two parameters of type `Amount` and produces a result of type `Amount`. 

This final `bi-function` is used when using operations on Streams in parallel. 
In this case we merely use the `add()` method to sum `Amount` objects.

## We can do better still 
We have the compiler working for us and we have improved our code by making it more type safe. We can do better still. 
Let's add some validation to our new value types and create a convenient method for their construction:

```java
private Amount(double value) {
    this.value = value;
}

public static Amount of(double value) {
    if (value <= 0) {
        throw new IllegalArgumentException("Amount must be greater than 0");
    } else {
        return new Amount(value);
    }
}
```

We've locked down our constructor and introduced a static factory method `of()` that contains our validation logic. 
Now when we go to instantiate an `Amount` object it will look like `Amount.of(40.1);`. Nice and neat.

## Immutables.io library

Before wrapping up let’s have a quick look at [Immutables](https://immutables.github.io/), a handy library for generating immutable objects in Java at compile time. 
Here is what our `Amount` value type looks like when using Immutables:

```java
@Value.Immutable
@Value.Style(visibility = Value.Style.ImplementationVisibility.PACKAGE)
public abstract class Amount {

    @Value.Parameter
    abstract double value();

    public static Amount of(double value) {
        if (value <= 0) throw new IllegalArgumentException("Amount must be greater than 0");
        else return ImmutableAmount.of(value);
    }

    public static Amount zero() {
        return ImmutableAmount.of(0);
    }

    public Amount add(Amount that) {
        return Amount.of(value() + that.value());
    }

    public String toString() {
        return Double.toString(value());
    }
}
```

There's still quite a bit of boilerplate when defining your value types but check out some of the neat extras you get with Immutables:
 
```java
@Override
public boolean equals(Object another) {
    if (this == another) return true;
    return another instanceof ImmutableAmount && equalTo((ImmutableAmount) another);
}

private boolean equalTo(ImmutableAmount another) {
    return Double.doubleToLongBits(value) == Double.doubleToLongBits(another.value);
}

@Override
public int hashCode() {
    int h = 5381;
    h += (h << 5) + Double.hashCode(value);
    return h;
}

public static ImmutableAmount copyOf(Amount instance) {
    if (instance instanceof ImmutableAmount) return (ImmutableAmount) instance;

    return ImmutableAmount.builder().from(instance).build();
}
```

It auto generates an equality method along with a `hashCode()` method that uses the standard type that's being wrapped. 
Along with that you get a copyOf method. 
Pretty handy.

Hopefully this blog post has demonstrated how useful value types are and how they can improve your code.
