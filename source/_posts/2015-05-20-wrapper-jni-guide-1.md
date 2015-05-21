title: "Wrapper JNI Program Guide 1"
date: 2015-05-20 13:59:53
tags:  [Java, JNI]
---

Originally From [Wrapping a C++ library with JNI Part 1 ](http://thebreakfastpost.com/2012/01/23/wrapping-a-c-library-with-jni-part-1/)


## **Wrapping a C++ library with JNI <br/> - part 1**


### In this series…

- [Introduction](/2015/05/20/wrapper-jni-guide-0), outlining the general steps from starting with a C++ library to being able to build and run simple tests on some JNI wrappers;
- [Part 1](/2015/05/20/wrapper-jni-guide-1), in which I design some simple Java classes and generate the stub wrapper code;
- [Part 2](/2015/05/20/wrapper-jni-guide-2), in which I add just enough of the implementation to be able to do a test build;
- [Part 3](/2015/05/20/wrapper-jni-guide-3), discussing object lifecycles in C++ and Java;
- [Part 4](/2015/05/21/wrapper-jni-guide-4), the final episode covering a few remaining points of interest.

In my introductory post I outlined the steps I’d need to take in order to get a JNI wrapper working for a simple subset of a C++ library. Time to get started.

### **1. Identify the C++ classes and methods**

The C++ library I’m wrapping is documented [here](https://code.soundsoftware.ac.uk/embedded/vamp-plugin-sdk/namespaceVamp.html).

I’ll initially need to wrap parts of two classes: PluginLoader, in the Vamp::HostExt namespace, and Plugin, in the Vamp namespace.

PluginLoader is a singleton class which loads and returns an instance of a Vamp plugin on request, given the name (or “key”) of the plugin. It’s essentially an object factory.

Then Plugin has both methods that return information about the specific plugin that has been loaded, and methods that are called with blocks of audio data in order to do the actual analysis.

In my first draft, I want to make a singleton Java class to correspond to PluginLoader and implement in it the one method that actually loads the plugin; and I want to make a Plugin class with a couple of the methods that return simple metadata about the plugin.

The methods that do real work involve some complicated return types, so I’ll leave those until later.

So the basic library API I’m starting with looks like this, in the existing C++ API:

```
class PluginLoader
{
public:
    static PluginLoader *getInstance();
    Plugin *loadPlugin(PluginKey key,
                       float inputSampleRate,
                       int adapterFlags = 0);
    // ... other methods for later
};

class PluginBase
{
public:
    virtual std::string getIdentifier() const = 0;
    virtual std::string getName() const = 0;
    virtual std::string getDescription() const = 0;
    virtual int getPluginVersion() const = 0;
    // ... other methods for later
};
```

That’s plenty to be going on with.

Note that the `PluginBase` with its pure virtual methods is somewhat like a C++ equivalent of a Java interface; the plugins themselves have a subclass type, namely Plugin.

### **2. Design the Java classes**

There may be several ways to render a native library into Java classes, even where the native library API is already written as a set of C++ classes.

In this case, `PluginBase` is similar to a Java interface, so I could turn it into one, or I could just put everything straight into the `Plugin` class in the Java version.

I’ll start with the latter, because it’s simpler. At this stage my only goal is to get something built that I can run and test. I can refactor later.

So, my first draft of `Plugin` is:

```
package org.vamp_plugins;

public class Plugin
{
    public native String getIdentifier();
    public native String getName();
    public native String getDescription();
    public native int getPluginVersion();
}
```

The methods are all declared native and left unimplemented in the Java; they’ll be implemented in our C++ JNI wrapper later on.

Meanwhile I want my `PluginLoader` to look a bit like this:
```
package org.vamp_plugins;

public class PluginLoader
{
    public class LoadFailedException
        extends Exception { };
    public static synchronized PluginLoader getInstance() {
        // some magic here
    }
    public Plugin loadPlugin(String key, float inputSampleRate)
        throws LoadFailedException {
        // and here
    }
}
```

The C++ `PluginLoader` returns a null pointer if it can’t load a plugin. I want to throw an exception instead, and for this reason I haven’t marked loadPlugin as native—instead it will call a native method that returns a null or non-null value, throwing the exception if the returned value is null.

I’ve made other simplifications too. I happen to know that the PluginKey type referred to in the C++ API is just a typedef for std::string, so I’ve translated it to a Java string type for the moment. I’ve omitted the options argument for now as well.

### Native object handles

I have two Java classes, each instance of which should “manage” a C++ object of a corresponding class from the native library. Every time a method is called on a given Java instance, it should call through to the same instance of the underlying C++ class.

To make this happen, we need to make the Java object remember which C++ object it is managing.

For this purpose we’ll have both classes simply contain a “handle” field, which will hold a value that (on the native side of the fence) will be interpreted as a pointer to the C++ object on the heap, and (on the Java side) will be an opaque integer value.

We just need to make sure the handle is stored in an integer type big enough for a pointer; the Java long type will do.

Also, we need a way to construct Plugin objects. The PluginLoader will make plugins by calling its native-code implementation, getting back a C++ plugin pointer, and using that to construct our Java Plugin, so our plugin class needs a constructor that works from our opaque handle type.

Adding these details and filling in our singleton implementation gives us:

```
public class PluginLoader
{
    public class LoadFailedException
        extends Exception { };

    public static synchronized PluginLoader getInstance() {
        if (inst == null) {
            inst = new PluginLoader();
            inst.initialise();
        }
        return inst;
    }

    public Plugin loadPlugin(String key, float inputSampleRate)
        throws LoadFailedException {
        long handle = loadPluginNative(key, inputSampleRate);
        if (handle != 0) return new Plugin(handle);
        else throw new LoadFailedException();
    }

    private PluginLoader() { initialise(); }
    private native long loadPluginNative(String key, float inputSampleRate);
    private native void initialise();
    private static PluginLoader inst;
    private long nativeHandle;
}
```
and
```
public class Plugin
{
    private long nativeHandle;
    protected Plugin(long handle) {
        nativeHandle = handle;
    }

    public native String getIdentifier();
    public native String getName();
    public native String getDescription();
    public native int getPluginVersion();
}
```

The initialise method in `PluginLoader` is there to construct a native object and set up its `nativeHandle` field. `Plugin` doesn’t need a similar method, because its handle is given to it in the constructor.

*Hang on! How does the native object get deleted?*  — I give over a later post entirely to discussion of this question, so skip there if you’re interested now.

### **3. Generate JNI function declarations**

For each of the native methods in our Java classes, we need an implementation in the JNI wrapper code. This will be a function in C or C++, with a name encoding the Java class and method name and an appropriate signature to receive the JNI mappings for the Java argument types, and with C linkage.

It’s perfectly possible to write these from scratch, but the JDK includes a tool called javah that can help by producing the function prototypes. It requires compiled class files rather than Java files as input:

```
$ javac org/vamp_plugins/*.java
$ javah -jni org.vamp_plugins.Plugin org.vamp_plugins.PluginLoader
$ ls -1
org
org_vamp_plugins_Plugin.h
org_vamp_plugins_PluginLoader.h
sample.h
$
```
As you see, javah generates a header file for each Java class; they contain declarations like
```
JNIEXPORT jstring JNICALL
Java_org_vamp_1plugins_Plugin_getIdentifier
 (JNIEnv *, jobject);
```

The `JNIEXPORT` and `JNICALL` macros simply ensure traditional C linkage regardless of the build platform.

The method’s return type, `String` in the Java code, has been converted to the opaque JNI type `jstring`.

The function name is an encoding of the package, class, and method name. These must appear exactly as generated by javah, as the JVM relies on them to find the implementation code.

And the two arguments, which are common to all JNI implementation functions, provide the environment object which rolls up the JNI function namespace, and an opaque jobject referring to the this object on which the method was called.

If we leave these headers untouched and simply include them in our implementation files, then we’ll be able to regenerate them when we extend our Java API without losing any of our work.

> *Administrative note: If you haven’t already, this is a good moment to put the code directory under version control, so as to be able to mess around without fear of losing any working code.*
> 
>  *I like [Mercurial](http://mercurial.selenic.com/), so I would either `hg init; hg add org/vamp_plugins/*.java` and `hg commit`, or else run up [EasyMercurial](http://easyhg.org/) and follow roughly [the explanation in this video.](http://vimeo.com/29779641)*

## **Next**

In my [next article](/2015/05/20/wrapper-jni-guide-2) I talk about writing the JNI wrapper implementations for these functions, and testing them from Java.