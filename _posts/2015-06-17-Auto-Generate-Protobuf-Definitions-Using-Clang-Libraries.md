---
layout: post
title: Generate Protobuf Message definitions using Clang libraries(TODO)
---

## Introduction
Recently we are doing a major refactoring of the basic infrastructure. One goal is to use [Google Protobuf](https://github.com/google/protobuf) during messing passing, which requires adding protobuf definition for all of our fundamental data format. Here is an example. Suppose we have a struct:

{% highlight bash %}
typedef double double32;
typedef long int64;

struct Stock {
char symbol[10];
double32 price;
double32 open_price;
int64 volume;
};
{% endhighlight %}

And what we want is:

{% highlight bash %}
message Stock {
required string symbol = 1;
required double price = 2;
required double open_price = 3;
required long volume = 4;
};
{% endhighlight %}

This brings up a very interesting question: is there a way to auto generate these protobuf definitions?

## Regular Expression
The first method came to my mind is regular expression. Suppose in the code there are two kinds of syntax to define a struct.
`struct Stock {}` or `typedef struct {} Stock`, we could use regex to extract the struct definition and then do a simple replacement. [demo link](https://regex101.com/r/zP0nB8/2)
![my-dev-screenshot]({{ site.baseurl }}/images/regex_parse_class.png)
       
However, using regex is just like doing a part what compiler is doing and it’s not extensible. If we have recursive typedef `typedef xxx int; typedef xxx yyy;` or we want to use constumize definitions in protobuf message  `Message Stock {required message Company;}`, it will be more difficult to analyze the code. 

## Reflection
Another way to go is reflection. [Reflection is the ability to inspect classes and dynamically call classes and functions.](http://stackoverflow.com/questions/37628/what-is-reflection-and-why-is-it-useful) it could help with questions like “what are all the fields class Foo has", "is their a method doFoo in class Foo" or "what is the root class of class Foo". Here is an toy example using java:

{% highlight java %}
package com.example;
import java.lang.reflect.Field;

public class Main {
    private class Foo {
        public int a;
        public double b;
        public String c;
    }
    public static void printAllFields(Class targetClass) {
        Field[] fields = targetClass.getFields();
        for(Field field : fields) {
            System.out.printf("%s, %s\n", field.getType().getSimpleName(), field.getName());
        }
    }
    public static void main(String[] args) {
        printAllFields(Foo.class);
    }
}
// output:
// int, a
// double, b
// String, c
{% endhighlight %}
The whole thing could be suprisingly simple in Java, but unfortunately, I am using C++. Unlike java or python, C++ doesn’t provide native support for reflection(why? short answer: it requires big change and is not top priority. [checkout this](http://stackoverflow.com/questions/359237/why-does-c-not-have-reflection) ) There are [ ways to use reflection](http://stackoverflow.com/questions/41453/how-can-i-add-reflection-to-a-c-application). One of them is boost::type_traits and it requires a big refactor of code using this feature. That’s doable but personally I think that’s even difficulty than the regex solution. You have to learn one additional libraries and also have to be very careful when it comes to dependencies between classes. 
## Clang AST

Clang AST provide compiler level APIs to explore C++ classes and functions. It is much much more difficult than I expected, but I learnt a lot in this journey.

### A simple taste
simple example using clang-check
### Output all fields of one class
code example
### recursive type def


