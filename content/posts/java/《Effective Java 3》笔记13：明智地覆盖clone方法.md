---
title: "《Effective Java 3》笔记13：明智地覆盖 clone 方法"
date: 2023-05-26
type: post
slug: override-clone-judiciously
categories: ["Java"]
tags: [java]
---

本文是 《Effective Java 3》第三章《对象的通用方法》的学习笔记：明智地覆盖 clone 方法。

## 介绍

Cloneable 接口的目的是作为 mixin 接口，用于让类来宣称它们允许克隆。不幸的是，它没有达到这个目的。它的主要缺点是缺少 clone 方法，并且 Object 类的 clone 方法是受保护的。如果不求助于反射，就不能仅仅因为对象实现了 Cloneable 接口就能调用 clone 方法。即使反射调用也可能失败，因为不能保证对象具有可访问的 clone 方法。尽管存在多种缺陷，但该机制的使用范围相当广泛，因此理解它是值得的。本条目将告诉你如何实现行为良好的 clone 方法，讨论什么时候应该这样做，并提供替代方案。

既然 Cloneable 接口不包含任何方法，用它来做什么呢？它决定了 Object 类受保护的 clone 实现的行为：如果一个类实现了 Cloneable 接口，Object 类的 clone 方法则返回该类实例的逐字段拷贝；否则它会抛出 CloneNotSupportedException。这是接口非常不典型的一种使用方式，不应该效仿。通常，类实现接口可以表明类能够为其客户端做些什么。在本例中，它修改了超类上受保护的方法的行为。

虽然规范没有说明，但是在实践中，实现 Cloneable 接口的类应该提供一个功能正常的公共 clone 方法。为了实现这一点，类及其所有超类必须遵守复杂的、不可强制执行的、文档很少的协议。产生的机制是脆弱的、危险的和非语言的：即它创建对象而不调用构造函数。

clone 方法的一般约定很薄弱。下面的内容是从 Object 规范复制过来的：

创建并返回此对象的副本。"副本" 的确切含义可能取决于对象的类。通常的意图是，对于任何对象 x，表达式

```java
x.clone() != x
```

将为 true，并且表达式

```java
x.clone().getClass() == x.getClass()
```

将为 true，但这并不是绝对要求。一般来说，对于任何对象 x 和 y，如果它们的 equals 方法返回 true，则表达式

```java
x.clone().equals(x)
```

也应返回 true。

clone 方法创建并返回对象的副本。「副本」的确切含义可能取决于对象的类别。通常，对于任何对象 x，表达式 `x.clone() != x`、`x.clone().getClass() == x.getClass()` 以及 `x.clone().equals(x)` 的值都将为 true，但都不是绝对的。

按照惯例，此方法返回的对象应通过调用 super.clone() 来获取。如果一个类及其所有父类（除了 Object）都遵循这个惯例，那么就会有以下情况：

```java
x.clone().getClass() == x.getClass().
```

按照约定，clone 方法返回的对象应该通过调用 super.clone() 来获得。如果一个类和它的所有超类（Object 类除外）都遵守这个约定，在这种情况下，表达式 `x.clone().getClass() == x.getClass()` 则为 true。

按照约定，返回的对象应该独立于被克隆的对象。为了实现这种独立性，可能需要在 super.clone() 返回前，修改对象的一个或多个字段。

这种机制有点类似于构造方法链，只是没有强制执行：

- （1）如果一个类的 clone 方法返回的实例不是通过调用 super.clone() 而是通过调用构造函数获得的，编译器不会报错，但是如果这个类的一个子类调用 super.clone()，由此产生的对象类型将是错误的，影响子类 clone 方法正常工作。
- （2）如果覆盖 clone 方法的类是 final 修饰的，那么可以安全地忽略这个约定，因为没有子类需要担心。
- （3）如果一个 final 修饰的类不调用 super.clone() 的 clone 方法。类没有理由实现 Cloneable 接口，因为它不依赖于 Object 类的 clone 实现的行为。

```java
class Base {
    @Override protected Object clone() throws CloneNotSupportedException {
        return new Base(); // ①
    }
}

class BasePro extends Base implements Cloneable {
    @Override protected Object clone() throws CloneNotSupportedException {
        return super.clone();
    }

    public static void main(String[] args) throws Exception {
        BasePro basePro = new BasePro();
        System.out.println(basePro.clone().getClass()); // 输出 class com.example.demo.Base
        System.out.println(basePro.getClass()); // 输出 class com.example.demo.BasePro
    }
}
```

可采用两种方式修复

- ① 处改用 super.clone()
- 移除 Base 类整个 clone() 实现

假设您想在一个类中实现 Cloneable 接口，而其超类提供了一个良好的 clone 方法。首先调用 super.clone() 方法。您得到的对象将是原始对象的一个完全功能的副本。在您的类中声明的任何字段都将具有与原始对象相同的值。如果每个字段都包含一个基本类型的值或一个不可变对象的引用，则返回的对象可能正是您所需要的，此时不需要进一步处理。例如，对于 PhoneNumber 类，就是这种情况，但请注意，不可变类不应提供 clone 方法，因为这只会鼓励浪费性的复制。在这个前提下，下面是一个 PhoneNumber 的 clone 方法的实现示例：

```java
// Clone method for class with no references to mutable state
@Override public PhoneNumber clone() {
    try {
        return (PhoneNumber) super.clone();
    } catch (CloneNotSupportedException e) {
        throw new AssertionError(); // Can't happen
    }
}
```

为了让这个方法工作，必须修改 PhoneNumber 类的声明，使之实现 Cloneable 接口。虽然 Object 的 clone 方法返回 Object 类型，但是这个 clone 方法返回 PhoneNumber 类型。这样做是合法的，也是可取的，因为 Java 的返回值类型支持协变。换句话说，覆盖方法的返回类型可以是被覆盖方法的返回类型的子类。这样就不需要在客户端中进行强制转换。我们必须把源自 Object 类的 super.clone() 方法在返回前将结果转换为 PhoneNumber 类型，这类强制转换肯定会成功。

将 super.clone() 包含在 try-catch 块中。这是因为 Object 类声明的 clone 方法会抛出 CloneNotSupportedException，这是一种 checked exception。因为 PhoneNumber 类实现了 Cloneable 接口，所以我们知道对 super.clone() 的调用将会成功。这个样板文件的需求表明 CloneNotSupportedException 应该是 unchecked exception。

如果对象的字段包含可变对象的引用，前面所示 clone 方法的这种简易实现可能引发灾难。例如，考虑 Stack 类：

```java
public class Stack {
    private Object[] elements;
    private int size = 0;
    private static final int DEFAULT_INITIAL_CAPACITY = 16;

    public Stack() {
      this.elements = new Object[DEFAULT_INITIAL_CAPACITY];
    }

    public void push(Object e) {
        ensureCapacity();
        elements[size++] = e;
    }

    public Object pop() {
        if (size == 0)
            throw new EmptyStackException();
        Object result = elements[--size];
        elements[size] = null; // Eliminate obsolete reference
        return result;
    }

    // Ensure space for at least one more element.
    private void ensureCapacity() {
        if (elements.length == size)
            elements = Arrays.copyOf(elements, 2 * size + 1);
    }
}
```

假设你想让这个类可克隆。如果 clone 方法只返回 super.clone()，得到的 Stack 实例在其 size 字段中会有正确的值，但其 elements 字段将引用与原始 Stack 实例相同的数组。修改初始值将破坏克隆的不变性，反之亦然。你将很快发现你的程序产生了无意义的结果或抛出 NullPointerException。

调用 Stack 类中唯一构造函数的情况永远不会发生。实际上，clone 方法将充当构造函数；你必须确保它不会对原始对象造成伤害，并且 clone 方法正确地实现了不变性。为了使 Stack 类上的 clone 方法正常工作，它必须复制 Stack 类实例的内部。最简单的做法是在 elements 字段对应的数组递归调用 clone 方法：

```java
// Clone method for class with references to mutable state
@Override
public Stack clone() {
    try {
        Stack result = (Stack) super.clone();
        result.elements = elements.clone();
        return result;
    } catch (CloneNotSupportedException e) {
        throw new AssertionError();
    }
}
```

注意，我们不需要将 `elements.clone` 的结果强制转换到 `Object[]`。在数组上调用 clone 方法将返回一个数组，该数组的运行时和编译时类型与被克隆的数组相同。这是复制数组的首选习惯用法。实际上，复制数组是 clone 机制唯一令人信服的使用场景。

还要注意，如果 elements 字段是 final 修饰的，上述解决方案就无法工作，因为 clone 方法将被禁止为字段分配新值。这是一个基础问题：与序列化一样，可克隆体系结构与使用 final 修饰可变对象引用的常用方式不兼容，除非在对象与其克隆对象之间可以安全地共享可变对象。为了使类可克隆，可能需要从某些字段中删除 final 修饰符。

仅仅递归调用 clone 方法并不总是足够的。例如，假设你正在为 HashTable 编写一个 clone 方法，HashTable 的内部由一组 bucket 组成，每个 bucket 引用键-值对链表中的第一个条目。为了提高性能，类实现了自己的轻量级单链表，而不是在内部使用 `java.util.LinkedList`：

```java
public class HashTable implements Cloneable {
    private Entry[] buckets = ...;

    private static class Entry {
        final Object key;
        Object value;
        Entry next;

        Entry(Object key, Object value, Entry next) {
            this.key = key;
            this.value = value;
            this.next = next;
        }
    } ... // Remainder omitted
}
```

假设你只是像对 Stack 所做的那样，递归克隆 bucket 数组，如下所示：

```java
// Broken clone method - results in shared mutable state!
@Override
public HashTable clone() {
    try {
        HashTable result = (HashTable) super.clone();
        result.buckets = buckets.clone();
        return result;
    } catch (CloneNotSupportedException e) {
        throw new AssertionError();
    }
}
```

尽管 clone 方法有自己的 bucket 数组，但该数组引用的链接列表与原始链表相同，这很容易导致克隆和原始的不确定性行为。要解决这个问题，你必须复制包含每个 bucket 的链表。这里有一个常见的方法：

```java
// Recursive clone method for class with complex mutable state
public class HashTable implements Cloneable {
    private Entry[] buckets = ...;

    private static class Entry {
        final Object key;
        Object value;
        Entry next;

        Entry(Object key, Object value, Entry next) {
            this.key = key;
            this.value = value;
            this.next = next;
        }

        // Recursively copy the linked list headed by this Entry
        Entry deepCopy() {
            return new Entry(key, value,next == null ? null : next.deepCopy());
        }
    }

    @Override
    public HashTable clone() {
        try {
            HashTable result = (HashTable) super.clone();
            result.buckets = new Entry[buckets.length];

            for (int i = 0; i < buckets.length; i++)
                if (buckets[i] != null)
                    result.buckets[i] = buckets[i].deepCopy();

            return result;
        } catch (CloneNotSupportedException e) {
            throw new AssertionError();
        }
    } ... // Remainder omitted
}
```

私有内部类 HashTable.Entry 已经被增强，它提供了进行「深拷贝」的方法。HashTable 上的 clone 方法分配一个大小合适的新 buckets 数组，并遍历原始 buckets 数组，对每个非空 buckets 元素进行深拷贝。Entry 类的 deepCopy() 方法会被递归调用直至复制完整个链表（该链表以 Entry 类的实例作为头节点）。这种方法虽然很灵活，而且在 buckets 不太长的情况下可以很好地工作，但是克隆链表并不是一个好方法，因为它为链表中的每个元素消耗一个堆栈帧。如果列表很长，很容易导致堆栈溢出。为了防止这种情况的发生，你可以用迭代替换 deepCopy() 方法的递归调用：

```java
// Iteratively copy the linked list headed by this Entry
Entry deepCopy() {
    Entry result = new Entry(key, value, next);
    for (Entry p = result; p.next != null; p = p.next)
        p.next = new Entry(p.next.key, p.next.value, p.next.next);
    return result;
}
```

克隆复杂可变对象的最后一种方法是调用 super.clone()，将结果对象中的所有字段设置为初始状态，然后调用更高级别的方法重新生成原始对象的状态。在我们的 HashTable 示例中，buckets 字段将初始化为一个新的 bucket 数组，并且对于克隆的 hash 表中的每个键值映射将调用 put(key, value) 方法（未显示）。这种方法通常产生一个简单、相当优雅的 clone 方法，它的运行速度不如直接操作克隆的内部的方法快。虽然这种方法很简洁，但它与整个可克隆体系结构是对立的，因为它盲目地覆盖了构成体系结构基础的逐字段对象副本。

与构造函数一样，clone 方法绝不能在正在构建的克隆上调用可覆盖方法。如果 clone 调用一个在子类中被重写的方法，这个方法将在子类有机会修复其在克隆中的状态之前执行，很可能导致克隆和原始的破坏。因此，前一段中讨论的 put(key, value) 方法应该是 final 修饰或 private 修饰的方法。（如果它是私有的，那么它可能是没有 final 修饰的公共「助手方法」。)

对象的 clone 方法被声明为抛出 CloneNotSupportedException，但是重写方法不需要。**公共克隆方法应该省略 throw 子句，** 作为不抛出受控异常的方法更容易使用。

用继承方式设计一个类时，你有两种选择，但是无论你选择哪一种，都不应该实现 Cloneable 接口。你可以选择通过实现一个功能正常的受保护克隆方法来模拟 Object 的行为，该方法声明为抛出 CloneNotSupportedException。这给子类实现 Cloneable 或不实现 Cloneable 的自由，就像它们直接扩展对象一样。或者，你可以选择不实现一个有效的克隆方法，并通过提供以下退化的克隆实现来防止子类实现它：

```java
// clone method for extendable class not supporting Cloneable
@Override
protected final Object clone() throws CloneNotSupportedException {
    throw new CloneNotSupportedException();
}
```

还有一个细节需要注意。如果你编写了一个实现了 Cloneable 接口的线程安全类，请记住它的 clone 方法必须正确同步，就像其他任何方法一样。Object 类的 clone 方法不是同步的，因此即使它的实现在其他方面是令人满意的，你也可能需要编写一个返回 super.clone() 的同步 clone 方法。

回顾一下，所有实现 Cloneable 接口的类都应该使用一个返回类型为类本身的公共方法覆盖 clone。这个方法应该首先调用 super.clone()，然后「修复」任何需要「修复」的字段。通常，这意味着复制任何包含对象内部「深层结构」的可变对象，并将克隆对象对这些对象的引用替换为对其副本的引用。虽然这些内部副本通常可以通过递归调用 clone 来实现，但这并不总是最好的方法。如果类只包含基本数据类型的字段或对不可变对象的引用，那么很可能不需要修复任何字段。这条规则也有例外。例如，表示序列号或其他唯一 ID 的字段需要修复，即使它是基本数据类型或不可变的。

搞这么复杂真的有必要吗？答案是否定的。如果你扩展了一个已经实现了 Cloneable 接口的类，那么除了实现行为良好的 clone 方法之外，你别无选择。否则，最好提供对象复制的替代方法。一个更好的对象复制方法是提供一个复制构造函数或复制工厂。复制构造函数是一个简单的构造函数，它接受单个参数，其类型是包含构造函数的类，例如：

```java
// Copy constructor
public Yum(Yum yum) { ... };
```

复制工厂与复制构造函数的静态工厂类似：

```java
// Copy factory
public static Yum newInstance(Yum yum) { ... };
```

复制构造函数方法及其静态工厂变体与克隆/克隆相比有许多优点：它们不依赖于易发生风险的语言外对象创建机制；他们不要求无法强制执行的约定；它们与最终字段的正确使用不冲突；它们不会抛出不必要的检查异常；而且不需要强制类型转换。

此外，复制构造函数或工厂可以接受类型为类实现的接口的参数。例如，按照约定，所有通用集合实现都提供一个构造函数，其参数为 collection 或 Map 类型。基于接口的复制构造函数和工厂（更确切地称为转换构造函数和转换工厂）允许客户端选择副本的实现类型，而不是强迫客户端接受原始的实现类型。例如，假设你有一个 HashSet s，并且希望将它复制为 TreeSet。克隆方法不能提供这种功能，但是使用转换构造函数很容易：new TreeSet<>(s)。

考虑到与 Cloneable 相关的所有问题，新的接口不应该扩展它，新的可扩展类不应该实现它。虽然 final 类实现 Cloneable 接口的危害要小一些，但这应该被视为一种性能优化，仅在极少数情况下是合理的。通常，复制功能最好由构造函数或工厂提供。这个规则的一个明显的例外是数组，最好使用 clone 方法来复制数组。

## 总结

建议慎重覆盖 `clone()` 方法。`clone()` 方法是一个用于对象复制的方法，它可以创建一个新的对象，并将原始对象的状态复制到新的对象中。但是，在实现 `clone()` 方法时，需要注意一些问题，否则使用 `clone()` 方法可能会导致一些潜在的问题。

以下是在实现 `clone()` 方法时需要注意的一些问题：

1. 首先，需要确保类实现了 `Cloneable` 接口。这个接口是一个标记接口，没有任何方法，但是如果一个类没有实现 `Cloneable` 接口，调用其 `clone()` 方法会抛出 `CloneNotSupportedException` 异常。
2. 在实现 `clone()` 方法时，需要调用 `super.clone()` 方法来创建一个新的对象，并将原始对象的状态复制到新的对象中。
3. 如果类的成员变量包含可变对象，那么需要对这些成员变量进行深度复制，以确保新对象与原始对象具有不同的状态。
4. `clone()` 方法返回的对象类型是 `Object`，需要进行类型转换，这可能会导致一些类型安全问题。因此，建议在覆盖 `clone()` 方法时，将返回类型声明为类本身的类型，并在方法中进行类型转换。
5. 在实现 `clone()` 方法时，需要注意对异常的处理。如果一个类没有实现 `Cloneable` 接口，调用其 `clone()` 方法会抛出 `CloneNotSupportedException` 异常。在实现 `clone()` 方法时，需要考虑这个异常，并在方法中进行适当的处理。

总之，在覆盖 `clone()` 方法时，需要考虑到上述问题，并根据实际情况进行处理。在某些情况下，可能需要采用其他的复制方式，例如序列化和反序列化。因此，在覆盖 `clone()` 方法之前，需要仔细考虑是否真正需要使用该方法，并根据实际情况选择最合适的复制方式。

## 扩展

### 1、深度拷贝和浅度拷贝

深度拷贝和浅度拷贝是指在进行对象复制时，对于对象中包含的成员变量，如何进行复制的问题。

浅度拷贝是指只复制对象中的基本数据类型和引用类型的地址，而不是引用类型所指向的对象本身。这就意味着，在进行浅度拷贝时，原始对象和复制对象中的引用类型成员变量将指向同一个对象，这可能会导致一些潜在的问题，例如一个对象的状态的改变会影响到另一个对象。

深度拷贝是指将对象中的基本数据类型和引用类型所指向的对象都进行复制。这就意味着，在进行深度拷贝时，原始对象和复制对象中的引用类型成员变量将指向不同的对象，这可以避免上述问题。

例如，考虑一个包含一个引用类型成员变量的类 `Person`，其中引用类型成员变量是一个 `Address` 对象。如果进行浅度拷贝，那么复制对象中的 `address`成员变量将指向原始对象中的 `address` 成员变量指向的同一个 `Address` 对象。这就意味着，如果修改复制对象中的 `address` 成员变量，原始对象中的 `address` 成员变量也会发生改变。

如果进行深度拷贝，那么复制对象中的 `address` 成员变量将指向一个新的 `Address` 对象，这样就可以避免修改一个对象对另一个对象造成的影响。

需要注意的是，进行深度拷贝可能会导致性能问题，因为需要递归地复制对象中的所有成员变量和引用类型所指向的对象。因此，在进行对象复制时，需要根据实际情况选择适当的复制方式，以权衡性能和正确性。

在 Java 中，通过实现 `Cloneable` 接口和覆盖 `clone()` 方法，可以实现对象的浅度拷贝。如果需要进行深度拷贝，可以通过序列化和反序列化实现，或者使用第三方库进行对象复制。

### 2、请问在 Java 中如何实现深度拷贝？

在 Java 中，可以通过以下几种方式实现深度拷贝：

1. 通过实现 `Serializable` 接口实现深度拷贝：在需要进行深度拷贝的类中实现 `Serializable` 接口，并通过序列化和反序列化实现深度拷贝。具体来说，可以将对象序列化为字节数组，然后再将字节数组反序列化为一个新的对象。需要注意的是，对象中的所有成员变量都必须是可序列化的，否则将会抛出 `NotSerializableException` 异常。

   下面是一个通过实现 `Serializable` 接口实现深度拷贝的示例代码：

   ```java
   import java.io.*;

   public class Person implements Serializable {
       private String name;
       private Address address;

       // 构造方法和 getter/setter 方法省略

       public Person deepCopy() throws IOException, ClassNotFoundException {
           // 将对象序列化为字节数组
           ByteArrayOutputStream bos = new ByteArrayOutputStream();
           ObjectOutputStream oos = new ObjectOutputStream(bos);
           oos.writeObject(this);
           oos.flush();

           // 将字节数组反序列化为一个新的对象
           ByteArrayInputStream bis = new ByteArrayInputStream(bos.toByteArray());
           ObjectInputStream ois = new ObjectInputStream(bis);
           return (Person) ois.readObject();
       }
   }
   ```

   在上面的示例代码中，通过实现 `Serializable` 接口，并在 `deepCopy()` 方法中将对象序列化为字节数组，然后再将字节数组反序列化为一个新的对象，从而实现了深度拷贝。需要注意的是，对象中的所有成员变量都必须是可序列化的，否则将会抛出 `NotSerializableException` 异常。

2. 通过实现自定义深度拷贝方法实现深度拷贝：在需要进行深度拷贝的类中实现自定义的深度拷贝方法，递归地复制对象中的所有成员变量和引用类型所指向的对象。需要注意的是，如果对象中的成员变量包含循环引用，那么需要对循环引用进行特殊处理，以避免无限递归。

3. 使用第三方库实现深度拷贝：Java 中有许多第三方库可以用于实现深度拷贝，例如 Apache Commons 的 `SerializationUtils`、Google 的 `Gson`、Jackson 等。

   下面是一个使用 Jackson 库实现深度拷贝的示例代码：

   ```java
   import com.fasterxml.jackson.databind.ObjectMapper;

   public class Person {
       private String name;
       private Address address;

       // 构造方法和 getter/setter 方法省略

       public Person deepCopy() throws IOException {
           ObjectMapper objectMapper = new ObjectMapper();
           String json = objectMapper.writeValueAsString(this);
           return objectMapper.readValue(json, Person.class);
       }
   }
   ```

   在上面的示例代码中，通过使用 Jackson 库将对象转换为 JSON 字符串，然后再将 JSON 字符串转换为新的对象，从而实现了深度拷贝。需要注意的是，如果对象中的成员变量包含循环引用，那么需要对循环引用进行特殊处理，以避免无限递归。

4. 使用流式 API：Java 8 引入了流式 API，可以使用流式 API 来实现对象的深度拷贝。具体来说，可以使用 `map()` 方法将对象流中的每个元素复制为一个新的对象，然后使用 `collect()` 方法将新对象收集到一个集合中。需要注意的是，如果对象中的成员变量包含循环引用，那么需要对循环引用进行特殊处理，以避免无限递归。

   当使用流式 API 实现深度拷贝时，可以使用 `map()` 方法将对象流中的每个元素复制为一个新的对象，然后使用 `collect()` 方法将新对象收集到一个集合中。需要注意的是，如果对象中的成员变量包含循环引用，那么需要对循环引用进行特殊处理，以避免无限递归。

   以下是一个使用流式 API 实现深度拷贝的示例代码：

   ```java
   import java.util.List;
   import java.util.stream.Collectors;

   public class Person {
       private String name;
       private Address address;

       // 构造方法和 getter/setter 方法省略

       public Person deepCopy() {
           List<Address> addresses = this.addresses.stream()
             .map(Address::deepCopy) // 使用 map() 方法将每个 Address 对象复制为一个新的对象
             .collect(Collectors.toList()); // 使用 collect() 方法将新对象收集到一个集合中
           return new Person(this.name, addresses);
       }
   }

   public class Address {
       private String street;
       private String city;

       // 构造方法和 getter/setter 方法省略

       public Address deepCopy() {
           return new Address(this.street, this.city);
         // 直接复制 Address 对象即可，因为 Address 类中没有引用类型的成员变量
       }
   }
   ```

   在上面的示例代码中，通过使用流式 API 将 `addresses` 集合中的每个 `Address` 对象复制为一个新的对象，并将新对象收集到一个集合中，从而实现了 `Person` 对象的深度拷贝。需要注意的是，在实现 `Address` 的 `deepCopy()` 方法时，因为 `Address` 类中没有引用类型的成员变量，因此可以直接复制一个新的 `Address` 对象即可。

   需要注意的是，如果对象中的成员变量包含循环引用，那么需要进行特殊处理，以避免无限递归。例如，在上面的示例代码中，如果 `Person` 类中包含一个 `List<Person>` 类型的成员变量，那么在进行深度拷贝时就需要对 `List<Person>` 进行特殊处理。
