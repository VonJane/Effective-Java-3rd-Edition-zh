# 第 11 条：覆盖 equals 时总要覆盖 hashCode

在每次重写equals方法时你都必须重写hashCode。如果你没有这样做，你就会违反hashCode的通用协议。
这将会阻止它在HashMap和HashSet等集合中正常运行。下面是协议，改编自Object规范：

   + 当hashCode方法在应用程序执行期间重复的调用，它必须一致的返回同样的值，如果在equals比较中没有使用信息，则修改。
     从应用程序的一个执行到另一个这个信息没必要保持一致。
   + 如果两个对象通过equal(Object)方法对比是相等的，那么在两个对象上调用hashCode必须产生一致的整数结果。
   + 如果两个对象通过equal(Object)方法对比是不相等的，在每个对象上调用hashCode不一定要产生不同的结果。然而，
     程序员应该意识到生产出不同结果的不同的对象有可能提高hash表的性能。
 
当您未能重写hashCode时违反的关键条款是第二个，比对对象必须比对他们的hashCode。不同的实例可能会根据类的equal方法在逻辑上相等。
但是对于类的equals方法而言，他们仅仅是两个没有什么共同之处的对象而已。
因此，对象的hashCode方法返回了两个看似随机的数字代替了两个同样的数字作为协议要求。

例如，假设您将第十条里的PhoneNumber类的实例用作hashMap的键：
```java
    Map<PhoneNumber, String> m = new HashMap<>();
    m.put(new PhoneNumber(707, 867, 5309), "Jenny");
```

此时，或许你指望m.get(new PhoneNumber(707, 867, 5309))会返回给你"Jenny",恰恰相反，它返回了null。
请注意其中涉及到了两个PhoneNumber实例：其中一个被用于插入到HashMap，另一个，对比实例被用于检索。PhoneNumber类未能重写hashCode，导致两个
相等的实例具有不相等的hashCode，这违背了hashCode协议.因此,get方法就好像在put进hash桶的另一个桶里寻找这个hash值.
即使这两个碰巧散列到同一个hash桶里，get方法在大多数情况下也会返回空，因为hashMap有一个优化，它缓存与每一个条目相关联的hashCode,
并且如果hashCode不匹配，它不会检查对象是否相等。

解决这个问题很简单。只需为PhoneNumber编写合适的hashCode方法即可。所以什么样的hashCode方法是合适的呢？写一篇糟糕的文章是微不足道的。
这一个，总是合法的，但是不应该被使用：

```java
    // The worst [possible legal hashCode implementation - never use!]
    @Override 
    public int hashCode(){ return 42; }
```
    
它是合法的是因为它保证对比对象时会返回同样的hashCode。它是残暴的因为它保证每个对象都会有一样的hashCode。因此，
每个对象都散列到同一个桶,哈希表退化为链表.程序应该在线性时间运行而不是在二次时间运行.对于大型哈希表而言,这是工作与不工作的区别。

一个好的哈希函数应该为不相等的实例产生不相等的hashCode。这正是hashCode协议第三部分的含义.理想上,哈希函数应该在所有int值上统一分配任何合理的不等实例集合.
实现这个理想可能是困难的.幸运的是实现公平的分配并不太难.这里有一些简单的秘诀:
    
    1. 声明一个名为result的int变量，然后将其初始化为对象中第一个重要字段的hash码c,就如2.a.中计算的那样.（从
       第十节回顾，重要的领域是影响等于比较的领域.）
    
    2. 对于对象中每个剩余的重要字段f,请执行下面的操作:
        a. 计算字段的hash码c:
            i.   如果这个字段是一个原始类型,当Type是对应于f的类型的盒装原始类,计算Type.hashCode(f).  
            ii.  如果该字段是对象引用,则此类equals方法通过递归调用equals来比较该字段,在字段上递归调用hashCode.
                 如果需要更复杂的比较,计算该字段的"规范表示"并且在规范表示上调用HashCode。如果该字段的值是Null,
                 使用0(或者其他一些常数，但0是传统).
            iii. 如果该字段是一个数组,将其视为每个重要元素都是一个单独的字段.换言之，为每一个重要元素通过递归的
                 应用这些原则来计算一个hash值,并结合2.b.如果这个数组没有重要元素，使用常量，最好不要用0.如果所有
                 元素都是重要的，用Arrays.hashCode.
        b. 将步骤2.1中计算的哈希码c组合成如下结果:
                result = 31 * result + c;
    
    3. Return result.