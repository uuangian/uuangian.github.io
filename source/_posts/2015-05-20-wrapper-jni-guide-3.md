title: "Wrapper JNI Program Guide 3"
date: 2015-05-20 14:00:08
tags: [Java, JNI]
---

Originally From [Wrapping a C++ library with JNI Part 3](http://thebreakfastpost.com/2012/02/09/wrapping-a-c-library-with-jni-part-3/)


## **Wrapping a C++ library with JNI <br/> - part 3**


### In this series…

- [Introduction](/2015/05/20/wrapper-jni-guide-0), outlining the general steps from starting with a C++ library to being able to build and run simple tests on some JNI wrappers;
- [Part 1](/2015/05/20/wrapper-jni-guide-1), in which I design some simple Java classes and generate the stub wrapper code;
- [Part 2](/2015/05/20/wrapper-jni-guide-2), in which I add just enough of the implementation to be able to do a test build;
- [Part 3](/2015/05/20/wrapper-jni-guide-3), discussing object lifecycles in C++ and Java;
- [Part 4](/2015/05/21/wrapper-jni-guide-4), the final episode covering a few remaining points of interest.

My [previous post on this topic](/2015/05/20/wrapper-jni-guide-2) ended with a very simple test program built and run successfully, wrapping a small subset of the C++ library I was referring to.

That’s all I want to go through step-by-step, but there are still a couple more things to explain. The first one is…

### **Disposing of C++ objects**

Java objects are garbage-collected: you don’t have to delete them when you’ve finished with them.

C++ objects are not. Unless they’re locally scoped objects allocated on the stack (i.e. created without a call to new), you have to delete them explicitly when you’ve finished with them, or else you’re creating a resource leak.

The wrapper code I’ve been describing uses Java objects that “own” their corresponding C++ objects. My Plugin object for example contains an opaque nativeObject handle which, on the C++ side, gets interpreted as a pointer to the C++ object that provides the plugin implementation. This object belongs to the Java Plugin wrapper, and it should have a corresponding lifetime which we need to manage. Nothing else is going to delete it if we forget to.

This means we need the Java object to delete its C++ object when it is itself deleted. But a Java object isn’t deleted explicitly, and it has no C++-style destructor function we could make this happen from.

We have two options:

1. Add a method to the Java class that deletes the C++ object, and insist that users of our library remember to call it;
2. Delete the C++ object from the Java class’s finalize method, which is called by the JVM when the Java object is garbage collected.

The second option, using finalize, is attractive because it reduces the burden on our users. Deleting the object would be automatic. If we take the first option, any method we add to delete our object will be one that isn’t required for most Java classes, and that also doesn’t exist in the same form in the C++ library we’re wrapping—so developers coming from either Java or C++ will quite likely overlook it.

But there are several problems with using finalize for this.

One problem is that, because the JVM only controls memory and resources within the Java heap, it can’t take into account any resources allocated in the native layer when determining which Java objects it needs to garbage-collect: our Java objects will seem smaller than they effectively are. So it might not finalize our objects quickly enough to prevent memory pressure.

A second problem is that we can’t guarantee which thread the finalize method will be called from. It’s quite important that our C++ object is deleted from the same thread that allocated it.

Finally, it’s possible that the C++ library we are wrapping might expect objects to be deleted in some particular order relative to one another, and we can’t enforce that if we leave the garbage collector to do it. When in C++, we should do as C++ programmers do.

And that means we need to get the user of our library to tell us when to delete objects: we must at least take our option 1. That is, we add a method called `dispose` to the Java class: when it’s called, it deletes the underlying native object. But, of course, our user—the programmer writing to our library—must remember to call it.

(Why “dispose”? Well, it’s a good name, and it’s the name used by some other libraries that rely on native code, such as [the SWT widget toolkit](http://www.eclipse.org/swt/).)

In our Plugin class, the `dispose` call would be simply
```
public native void dispose();
```
and the implementation in Plugin.cpp, using the native handle helper method referred to in my earlier post, would be along the lines of
```
void
Java_org_vamp_1plugins_Plugin_dispose(JNIEnv *env, jobject obj)
{
    Plugin *p = getHandle(env, obj);
    setHandle(env, obj, 0);
    delete p;
}
```
In principle we could also make the `finalize` method call `dispose` in case the user forgets. But that’s probably unwise, as the library’s behaviour could become quite unpredictable in subtle ways if the pattern and order of deallocations depended on whether the user had remembered to dispose an object or had left it to the garbage collector.

Another possibility might be to test within `finalize` whether the object has been disposed, and print a warning to `System.err` if it hadn’t.

### **Coming next**

In my [next (and final) post](/2015/05/20/wrapper-jni-guide-4) on this subject, I’ll give an illustration of a function with rather more complex argument and return types than the ones we’ve seen so far.

