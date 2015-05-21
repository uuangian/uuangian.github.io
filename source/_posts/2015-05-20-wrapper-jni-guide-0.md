title: "Wrapper JNI Program Guide"
date: 2015-05-20 13:59:46
tags: [Java, JNI]
---


Originally From [Wrapping a C++ library with JNI ](http://thebreakfastpost.com/2012/01/21/wrapping-a-c-library-with-jni-introduction/)


## **Wrapping a C++ library with JNI <br/> - introduction**


### In this series…

- [Introduction](/2015/05/20/wrapper-jni-guide-0), outlining the general steps from starting with a C++ library to being able to build and run simple tests on some JNI wrappers;
- [Part 1](/2015/05/20/wrapper-jni-guide-1), in which I design some simple Java classes and generate the stub wrapper code;
- [Part 2](/2015/05/20/wrapper-jni-guide-2), in which I add just enough of the implementation to be able to do a test build;
- [Part 3](/2015/05/20/wrapper-jni-guide-3), discussing object lifecycles in C++ and Java;
- [Part 4](/2015/05/21/wrapper-jni-guide-4), the final episode covering a few remaining points of interest.

Java Native Interface, or JNI, is a standard mechanism for calling “native” code—that is, code compiled from a language such as C or C++—from within the Java Virtual Machine.

It allows a programmer working in Java, or another language running on the JVM such as Scala, Clojure, Groovy, or Yeti, to use existing C and C++ library code, for reasons of performance or for compatibility with old or third-party code.

JNI is a bit of an old-school technology, though it’s gained more attention through its use in the Android platform as the glue between managed Dalvik applications and native NDK libraries.

(Later note: Java developers with no C or C++ experience may be happier using [JNA, or Java Native Access](https://github.com/twall/jna#java-native-access-jna). This more recent technology uses a single C library for each platform, which is supplied with JNA and which can load any native library. Using JNA still involves writing some boilerplate code, but that code is all in Java. I don’t cover JNA at all in this tutorial.)

### <div style="color: #228897;">**What I’m doing**</div>

I’m about to make a wrapper for an existing C++ library using JNI, so I’m going to explain the process as I go along.

The library I’m planning to wrap up is the host SDK for [Vamp audio analysis plugins](http://vamp-plugins.org/) [^1]. I hope the result will enable Java applications to use these plugins to analyse audio signals.

From a technical perspective, though, it doesn’t really matter what the library is. What matters is that I have some header files that describe the library’s API, and either a DLL or a static library that provides the implementation. In this case I also happen to have the library’s source code, but that isn’t essential to calling it through JNI.

Most JNI tutorials use C as the native implementation language, but here I’m assuming the native code is entirely in C++. (If you’d like to read a simpler introduction that uses C, try the JNI Programmer’s Guide [Getting Started](http://java.sun.com/docs/books/jni/html/start.html) chapter.)

This explanation uses desktop Java: although the principles are much the same, I won’t be referring the Android toolchain here.

### <div style="color: #228897;">**The steps I need to take**</div>

To get my C++ library working from Java code, I will need to:

  1. Identify the C++ classes and methods that I want to make available from those in the library’s API, starting with the smallest subset I can make a meaningful test case from;
  2. Design an appropriate set of equivalent Java classes, in which the methods that will be implemented by the library are declared using the native keyword and left unimplemented in the Java code;
  3. Generate stub function declarations for the JNI wrapper. The wrapper is a chunk of C code that contains a specially named function to implement each of the classes’ native methods, and the generator is what gives us the names of these functions;
  4. Write the body of the JNI wrapper, implementing each function by converting its Java type arguments into appropriate C or C++ types, calling the native library, and converting the results back again;
  5. Make a little test program in Java;
  6. Build everything and test it.
Then repeat until enough of the library has been wrapped.

### <div style="color: #228897;">**Next**</div>

- In the first tutorial post I’ll cover steps 1 to 3, getting to the point where we’re about to have to write some C++ for the JNI wrapper code.
The subsequent post will cover steps 4 to 6, to the point of having a working interface and test program for a simple subset of the C++ library.

And I’ll finish with a couple of posts that discuss management of object disposal and a few other details of handling a more complex method.

------

[^1]: To find out more about Vamp plugins, see [the programmers’ guide](http://vamp-plugins.org/guide.pdf) and [these tutorials](http://vamp-plugins.org/wiki/). But although they explain the concepts, note that these describe the process of writing plugins, while we are providing support for hosts— so the APIs are not quite the same.



***Process Map***
![](/img/traditional-way.png)
