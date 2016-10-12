---
date: 2014-01-15T00:00:00Z
description: Examples of lambda usage in Java 8
tags:
- java
- functional
- lambda
title: Java 8 Features - Lambdas and Method References
url: /2014/01/15/java-8-lambdas/
---

After a long wait, Java 8 is finally close to release. I am going to dig through the feature list and learn about the upcoming features. First, the headliner of Java 8 - lambda expressions.

### Lambda Expressions

Lambda expression is a concise way to express an anonymous function. The existing way for Java to pass functions around is using an anonymous inner class. This can be pretty wordy.

{{< highlight java >}}
        Callable<String> callable = new Callable<String>() {
            @Override
            public String call() throws Exception {
                return "I am called";
            }
        };
        callable.call();
{{< / highlight >}}

In Java 8, this becomes

{{< highlight java >}}
        Callable<String> callable = () -> "I am called";
        callable.call();
{{< / highlight >}}

Nice and short. On the right hand side, there is no need to specify you are implementing the `call()` method on `Callable`. The compiler figured that out by inferring the type (`Callable`) from the left hand side, which provides the context. There is no need to specify `call()` method because it only support interfaces with single public method. The new name for these types of interfaces is *functional interfaces*.

In the simplest case, you don't even need to provide a return statement since the result of the single expression will be used as the value for the return statement. In more complex cases, a block of statements can be provided instead of the expression.

When there is no left hand side and thus no context to infer the type from, a cast is needed to give the compiler a hint.

{{< highlight java >}}
        ((Runnable) () -> {
            System.out.println("Hi!");
        }).run();
{{< / highlight >}}

#### Method Parameters

When the implementable method has parameters, they typically don't need to be provided either. The compiler will determine them from the interface method declaration.

{{< highlight java >}}
        Comparator<String> comparator = (s1, s2) -> s1.compareToIgnoreCase(s2);
{{< / highlight >}}

#### Where is this pointing to?

`this` is a bit tricky inside lambdas. The `this` object basically references the enclosing instance. This is different from anonymous inner classes, but it makes perfect sense when you see it.

{{< highlight java >}}
public class LambdaTest {
    @Test
    public void simple_lambda() throws Exception {
        // in lambda,
        // scoping causes "this" to point to the instance of LambdaTest instead of the lambda expression itself
        ((Runnable) () -> {
            assertEquals("LambdaTest", this.toString());
        }).run();
        // in anonymous inner class, this is different
        // practically, this makes sense, as I can make actual use of "this" in an anon-inner class
        (new Runnable() {
            @Override
            public void run() {
                assertEquals("Inner", this.toString());
            }
            @Override
            public String toString() {
                return "Inner";
            }
        }).run();
    }
    @Override
    public String toString() {
        return LambdaTest.class.getSimpleName();
    }
}
{{< / highlight >}}

#### Variable capture

Just like anonymous inner class, using variable from outside the lambda has restrictions. The compiler is smart enough to know if you are changing the capture variable.

{{< highlight java >}}
        int size = 100;
        ((Runnable) () -> System.out.println(size)).run();
        //size++; //this causes a compilation error when uncommented - size needs to be effectively final
{{< / highlight >}}

Interestingly, you can get around this by using a method reference. I will show an example in the next section.

### Method References

Once you can create lambdas, method references make a lot sense. This allows you use an existing method in a lambda like way.

For example, you can write a lambda like this:

{{< highlight java >}}
        Comparator<String> comparator1 = (s1, s2) -> s1.compareToIgnoreCase(s2);
{{< / highlight >}}

And you can rewrite it using a method reference like this:

{{< highlight java >}}
        Comparator<String> comparator2 = String::compareToIgnoreCase;
{{< / highlight >}}

I was a bit surprised that it figured out Comparator's compare(o1, o2) method maps to o1.compareToIgnoreCase(o2). Based on the spec, the first parameter will be used as the receiver of the method reference and the rest of the parameters will be passed as parameters to the method in the method reference.

In addition to this format, there are additional syntax to satisfy calling a method reference for an instance, a static method, super class method, constructor and array constructor.

Calling an instance method on an object has an interesting twist.

{{< highlight java >}}
        Set<String> items = Collections.singleton("OK");
        Predicate<String> doesContain = items::contains;
        items = Collections.singleton("NOT OK");
        assertFalse(items.contains("OK"));
        assertTrue(doesContain.test("OK"));
{{< / highlight >}}

The code above works. This surprised me. I was expecting a compilation error since I made a modification to items. But method reference must be doing something more complex than re-writing as a lambda behind the scene. Very likely, it stored `items` object in another reference in the object that represented the method reference.

It probably looked similar to this:
{{< highlight java >}}
        Set<String> items = Collections.singleton("OK");
        // start of method reference 
        Set<String> reference = items;
        Predicate<String> doesContain = (s) -> reference.contains(s);
        // end of method reference 
        items = Collections.singleton("NOT OK");
        assertFalse(items.contains("OK"));
        assertTrue(doesContain.test("OK"));
{{< / highlight >}}

### Closing

I am looking forward to using these once Java 8 comes out. Many times, I shun the use of anonymous inner classes for functional logic due to the verbosity (and the ugliness) of the result code. Being able to use lambda and method reference will go a long way to make function style logic clear and concise in Java!

