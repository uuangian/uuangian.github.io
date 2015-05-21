title: "Wrapper JNI Program Guide 4"
date: 2015-05-21 08:45:27
tags: [Java, JNI]
---

Originally From [Wrapping a C++ library with JNI Part 4](http://thebreakfastpost.com/2012/03/06/wrapping-a-c-library-with-jni-part-4/)


## **Wrapping a C++ library with JNI <br/> - part 4**


### In this series…

- [Introduction](/2015/05/20/wrapper-jni-guide-0), outlining the general steps from starting with a C++ library to being able to build and run simple tests on some JNI wrappers;
- [Part 1](/2015/05/20/wrapper-jni-guide-1), in which I design some simple Java classes and generate the stub wrapper code;
- [Part 2](/2015/05/20/wrapper-jni-guide-2), in which I add just enough of the implementation to be able to do a test build;
- [Part 3](/2015/05/20/wrapper-jni-guide-3), discussing object lifecycles in C++ and Java;
- [Part 4](/2015/05/21/wrapper-jni-guide-4), the final episode covering a few remaining points of interest.


### **A More Complex Method**

The most complex function in our API is the process method in the `Plugin` class. In C++, this is
```
typedef std::vector<Feature> FeatureList;
typedef std::map<int, FeatureList> FeatureSet;

virtual FeatureSet process(const float *const *inputBuffers,
                           RealTime timestamp) = 0;
```
That is, the process method of a `Plugin` object takes a two-dimensional array of floats and a `RealTime` object as arguments, and returns a map from an integer to a `FeatureList`, which is a sequence of Feature objects stored in a vector. That's fairly complicated.

Java lacks typedef or [any very satisfactory alternative](http://www.ibm.com/developerworks/java/library/j-jtp02216/index.html). So, we render this as
```
public native Map<Integer, ArrayList<Feature>>
    process(float[][] inputBuffers, RealTime timestamp);
```
where `Feature` and `RealTime` are classes we must provide separately. (It's good form to declare the return type using an interface such as `Map` in cases where the caller doesn't need to care which specific container implementation is used. In this case our concrete return type should probably be a `TreeMap`.)

I'm not going to explain every detail of the implementation of this in the JNI wrapper, but I want to illustrate a couple of aspects:

### **Constructing Java objects from JNI code**
In this particular case, the objects we want to return take rather a lot of work to construct. However, we can illustrate a simple example. At the innermost level, all of our returned objects have type Feature, a Java class we define which has a constructor that needs no arguments.

To construct one of these, first we need to look up its class:
```
jclass featClass = env->FindClass("org/vamp_plugins/Feature");
```
Then we find the method in the class that corresponds to the constructor. The `GetMethodID` function is used for looking up all non-static methods, with the method name supplied as the first string argument. For a constructor, the method name is `<init>`. The final argument gives the [signature](http://docs.oracle.com/javase/1.5.0/docs/guide/jni/spec/types.html#wp16432) of the method to look up; in this case ()V means a method taking no arguments with a void return type.
```
jmethodID ctor = env->GetMethodID(featClass, "<init>", "()V");
```
Then, to construct the object we simply call the constructor:
```
jobject feature = env->NewObject(featClass, ctor);
```
### **Handling Generics through JNI**
This is dead easy. Generic types are there only for the purpose of compiler type-checking: they don't exist in the virtual machine or in the .class file.
In terms of Java objects, including from the perspective of any JNI code, our ArrayList<Feature> is simply ArrayList.
So, to construct the `TreeMap<Integer, ArrayList<Feature>>` we intend to return from our function, we only need to do this:
```
jclass treeMapClass = env->FindClass("java/util/TreeMap");
jmethodID treeMapCtor = env->GetMethodID(treeMapClass, "<init>", "()V");
jobject map = env->NewObject(treeMapClass, treeMapCtor);
```
Of course we then need to make sure the objects we put in it are of the right type, as the Java compiler is no longer there to help us with type-checking.

Similarly, if a generic container gets passed to a native function, we can (and must) just ignore the type specialisation when unpacking it.

### **Getting Data from Multi-Dimensional Arrays**

My process function takes a two-dimensional array of floats as one of its arguments. (This actually represents multi-channel audio sample data.)

Multi-dimensional arrays in Java are easier to deal with reliably than in C++. An array is an object, so a two-dimensional array is an array of array objects. It appears in the JNI implementation as a `jobjectArray`:
```
jobject
Java_org_vamp_1plugins_Plugin_process
    (JNIEnv *env, jobject obj,
     jobjectArray inputBuffers, jobject timestamp);
```
JNI provides simple accessor methods for getting at array elements: to pull out an object from an array of objects, we use `GetObjectArrayElement`. That will enable us to get hold of the second dimension of our arrays.

Then, to access a sequence of non-object type elements such as our float values all at once, we need to use a symmetrical pair of calls: one to lock the elements in place so the garbage collector can't get at them, and the other to release them again. To wit, `GetFloatArrayElements` and `ReleaseFloatArrayElements`.

Putting these together to retrieve our two-dimensional float array in C++, we have as the body of our `process` function:

```
int channels = env->GetArrayLength(data);
float **input = new float *[channels];

for (int c = 0; c < channels; ++c) {
    jfloatArray cdata =
        (jfloatArray)env->GetObjectArrayElement(data, c);
    input[c] = env->GetFloatArrayElements(cdata, 0);
}
```
Then we do something with the C array that is now stored in `input`, hang on to the output for a moment, and tidy up:
```
for (int c = 0; c < channels; ++c) {
    jfloatArray cdata =
        (jfloatArray)env->GetObjectArrayElement(data, c);
    env->ReleaseFloatArrayElements(cdata, input[c], 0);
}

delete[] input;
```
and we're done. It remains only to construct and return our rather complex return value based on whatever we calculated with the input array earlier.

### **That's all**

That's all for this series—thanks for reading, and I hope it's been useful.