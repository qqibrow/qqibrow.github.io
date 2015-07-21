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
![regex parse class]({{ site.baseurl }}/images/regex_parse_class.png)
       
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

Clang AST provide compiler level APIs to explore C++ classes and functions. Finally I choose this solution mainly because it is similar with Java reflection solution: pure analysis, not a single line change required to the codebase. However, it is much much more difficult than I expected, but I learnt a lot in this journey.

### A simple taste
The idea is pretty simple. Using clang AST to analyze a class, aggregate all fields and print out the class definition in protobuf message format. The first thing is to get familiar with clang AST.  clang provide a tool called `clang-check` which could be used to print out the AST tree. This tool is helpful for both learning and debugging.  We could try to use clang-check to dump the AST  of a simple class.

{% highlight c++ %}
#include <string>
using std::string;
namespace example {
class Foo {
    public:
        Foo();
        explicit Foo(string);
        ~Foo();
    private:
        string name_;
};
}  // namespace example

using namespace std;
using namespace example;

Foo::Foo() {}

Foo::Foo(string name): name_(name) {}

Foo::~Foo() {}
{% endhighlight %}

The output is:
![clang_check_simple_result]({{ site.baseurl }}/images/clang_check_simple_result.png)

From the result, the only thing needs to take care of is the `CXXRecordDecl` cause all the fields are under this declaration. Obversely we need to find out the API to do that.
### Output all fields of one class
code example
### recursive type def


