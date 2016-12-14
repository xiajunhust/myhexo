---
title: 为什么Java HashMap、CopyOnWriteArrayList等集合自己实现readObject和writeObject方法
date: 2016-08-06 21:48:23
tags: Java
---

PS:本文源码参考的是JDK 1.8.

# 1 readObject、writeObject方法是什么？作用是什么？

当一个class实现了Serializable接口，那么意味着这个类可以被序列化。如果类不实现readObject、writeObject方法，那么会执行默认的序列化和反序列化逻辑，否则执行自定义的序列化和反序列化逻辑，即readObject、writeObject方法的逻辑。

<!-- more -->

JDK提供的对于Java对象序列化操作的类是ObjectOutputStream，反序列化的类是ObjectInputStream。下面我们来看序列化的实现（ObjectOutputStream.writeObject）。

```
/**
 * Write the specified object to the ObjectOutputStream.  The class of the
 * object, the signature of the class, and the values of the non-transient
 * and non-static fields of the class and all of its supertypes are
 * written.  Default serialization for a class can be overridden using the
 * writeObject and the readObject methods.  Objects referenced by this
 * object are written transitively so that a complete equivalent graph of
 * objects can be reconstructed by an ObjectInputStream.
 *
 * <p>Exceptions are thrown for problems with the OutputStream and for
 * classes that should not be serialized.  All exceptions are fatal to the
 * OutputStream, which is left in an indeterminate state, and it is up to
 * the caller to ignore or recover the stream state.
 *
 * @throws  InvalidClassException Something is wrong with a class used by
 *          serialization.
 * @throws  NotSerializableException Some object to be serialized does not
 *          implement the java.io.Serializable interface.
 * @throws  IOException Any exception thrown by the underlying
 *          OutputStream.
 */
public final void writeObject(Object obj) throws IOException {
    if (enableOverride) {
        writeObjectOverride(obj);
        return;
    }
    try {
        writeObject0(obj, false);
    } catch (IOException ex) {
        if (depth == 0) {
            writeFatalException(ex);
        }
        throw ex;
    }
}
```

从方法注释可以看到，此方法正是执行了将对象序列化的操作。并且默认的序列化机制可以通过重写readObject、writeObject方法实现。实际调用的方法writeObject0最终会调到writeSerialData：  

```
/**
 * Writes instance data for each serializable class of given object, from
 * superclass to subclass.
 */
private void writeSerialData(Object obj, ObjectStreamClass desc)
    throws IOException
{
    ObjectStreamClass.ClassDataSlot[] slots = desc.getClassDataLayout();
    for (int i = 0; i < slots.length; i++) {
        ObjectStreamClass slotDesc = slots[i].desc;
        //如果类重写了writeObject方法
        if (slotDesc.hasWriteObjectMethod()) {
            PutFieldImpl oldPut = curPut;
            curPut = null;
            SerialCallbackContext oldContext = curContext;

            if (extendedDebugInfo) {
                debugInfoStack.push(
                    "custom writeObject data (class \"" +
                    slotDesc.getName() + "\")");
            }
            try {
                curContext = new SerialCallbackContext(obj, slotDesc);
                bout.setBlockDataMode(true);
                //调用实现类自己的writeobject方法
                slotDesc.invokeWriteObject(obj, this);
                bout.setBlockDataMode(false);
                bout.writeByte(TC_ENDBLOCKDATA);
            } finally {
                curContext.setUsed();
                curContext = oldContext;
                if (extendedDebugInfo) {
                    debugInfoStack.pop();
                }
            }

            curPut = oldPut;
        } else {
            defaultWriteFields(obj, slotDesc);
        }
    }
}
```

# 2 为什么是private方法？

javadoc上没有明确说明声明为private的原因，一个可能的原因是，除了子类以外没有其他类会使用它，这样不会被滥用。

另一个原因是，不希望这些方法被子类override。每个类都可以有自己的writeObject方法，序列化引擎会逐一调用。readObject相同。


# 3 HashMap中对readObject、writeObject方法的实现

## 3.1 为什么HashMap要自定义序列化逻辑

下文是摘自《Effective Java》：

<i>For example, consider the case of a hash table. The physical representation is a sequence of hash buckets containing key-value entries. The bucket that an entry resides in is a function of the hash code of its key, which is not, in general, guaranteed to be the same from JVM implementation to JVM implementation. In fact, it isn't even guaranteed to be the same from run to run. Therefore, accepting the default serialized form for a hash table would constitute a serious bug. Serializing and deserializing the hash table could yield an object whose invariants were seriously corrupt.</i>

大概意思是：对于同一个key，在不同的JVM平台上计算出来的hash值可能不同，导致的结果就是，同一个hashmap反序列化之后和序列化之前不同，导致同一个key取出来的值不同。

## 3.1 HashMap是如何解决的

- 将可能造成数据不一致的元素使用transient修饰，在序列化的时候忽略这些元素：  
<i>
Entry[] table  
size  
modCount
</i>

- HashMap中对writeObject的实现：

```
/**
 * Save the state of the <tt>HashMap</tt> instance to a stream (i.e.,
 * serialize it).
 *
 * @serialData The <i>capacity</i> of the HashMap (the length of the
 *             bucket array) is emitted (int), followed by the
 *             <i>size</i> (an int, the number of key-value
 *             mappings), followed by the key (Object) and value (Object)
 *             for each key-value mapping.  The key-value mappings are
 *             emitted in no particular order.
 */
private void writeObject(java.io.ObjectOutputStream s)
    throws IOException {
    int buckets = capacity();
    // Write out the threshold, loadfactor, and any hidden stuff
    s.defaultWriteObject();
    s.writeInt(buckets);
    s.writeInt(size);
    internalWriteEntries(s);
}

// Called only from writeObject, to ensure compatible ordering.
void internalWriteEntries(java.io.ObjectOutputStream s) throws IOException {
    Node<K,V>[] tab;
    if (size > 0 && (tab = table) != null) {
        for (int i = 0; i < tab.length; ++i) {
            for (Node<K,V> e = tab[i]; e != null; e = e.next) {
                s.writeObject(e.key);
                s.writeObject(e.value);
            }
        }
    }
}
```

HashMap不会将保存数据的数组序列化，而是将元素个数以及每个元素的key、value序列化。而在反序列化的时候，重新计算，填充hashmap：

readObject的实现：

```
/**
 * Reconstitute the {@code HashMap} instance from a stream (i.e.,
 * deserialize it).
 */
private void readObject(java.io.ObjectInputStream s)
    throws IOException, ClassNotFoundException {
    // Read in the threshold (ignored), loadfactor, and any hidden stuff
    s.defaultReadObject();
    reinitialize();
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new InvalidObjectException("Illegal load factor: " +
                                         loadFactor);
    s.readInt();                // Read and ignore number of buckets
    int mappings = s.readInt(); // Read number of mappings (size)
    if (mappings < 0)
        throw new InvalidObjectException("Illegal mappings count: " +
                                         mappings);
    else if (mappings > 0) { // (if zero, use defaults)
        // Size the table using given load factor only if within
        // range of 0.25...4.0
        float lf = Math.min(Math.max(0.25f, loadFactor), 4.0f);
        float fc = (float)mappings / lf + 1.0f;
        int cap = ((fc < DEFAULT_INITIAL_CAPACITY) ?
                   DEFAULT_INITIAL_CAPACITY :
                   (fc >= MAXIMUM_CAPACITY) ?
                   MAXIMUM_CAPACITY :
                   tableSizeFor((int)fc));
        float ft = (float)cap * lf;
        threshold = ((cap < MAXIMUM_CAPACITY && ft < MAXIMUM_CAPACITY) ?
                     (int)ft : Integer.MAX_VALUE);
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] tab = (Node<K,V>[])new Node[cap];
        table = tab;

        // Read the keys and values, and put the mappings in the HashMap
        for (int i = 0; i < mappings; i++) {
            @SuppressWarnings("unchecked")
                K key = (K) s.readObject();
            @SuppressWarnings("unchecked")
                V value = (V) s.readObject();
            putVal(hash(key), key, value, false, false);
        }
    }
}
```

这样就避免了反序列化之后根据Key获取到的元素与序列化之前获取到的元素不同。

# 4 为什么CopyOnWriteArrayList也需要自定义序列化逻辑？

writeObject、readObject实现：

```
   /**
     * Saves this list to a stream (that is, serializes it).
     *
     * @param s the stream
     * @throws java.io.IOException if an I/O error occurs
     * @serialData The length of the array backing the list is emitted
     *               (int), followed by all of its elements (each an Object)
     *               in the proper order.
     */
    private void writeObject(java.io.ObjectOutputStream s)
        throws java.io.IOException {

        s.defaultWriteObject();

        Object[] elements = getArray();
        // Write out array length
        s.writeInt(elements.length);

        // Write out all elements in the proper order.
        for (Object element : elements)
            s.writeObject(element);
    }

    /**
     * Reconstitutes this list from a stream (that is, deserializes it).
     * @param s the stream
     * @throws ClassNotFoundException if the class of a serialized object
     *         could not be found
     * @throws java.io.IOException if an I/O error occurs
     */
    private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {

        s.defaultReadObject();

        // bind to new lock
        resetLock();

        // Read in array length and allocate array
        int len = s.readInt();
        Object[] elements = new Object[len];

        // Read in all elements in the proper order.
        for (int i = 0; i < len; i++)
            elements[i] = s.readObject();
        setArray(elements);
    }
```

而数组被声明为transient：

```
/** The array, accessed only via getArray/setArray. */
private transient volatile Object[] array;
```

可以看出其逻辑和ArrayList相同：是将数组长度以及所有元素序列化，在反序列化的时候新建数组，填充元素。

如果采用默认的序列化机制会有如下问题：存储数据的数组实际上是动态数组，每次在放满以后自动增长设定的长度值，如果数组自动增长长度设为100，而实际只放了一个元素，那就会序列化很多null元素，所以ArrayList把元素数组设置为transient。 

# 5 参考资料

- [Why are readObject and writeObject private, and why would I write transient variables explicitly?](http://stackoverflow.com/questions/7467313/why-are-readobject-and-writeobject-private-and-why-would-i-write-transient-vari)
- [http://www.a-site.cn/article/140346.html](http://www.a-site.cn/article/140346.html)

