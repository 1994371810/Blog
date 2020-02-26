---
title: HashMap的实现原理
date: 2020-02-26 09:55:00
categories:
- java
---
**简单说一下HashMap的实现：**

1. hashMap底层通过数组加链表加红黑树实现 

2. 默认初始容量为16(必须为2的整数次幂)，增长因子默认为0.75(容量到0.75时自动扩容) 自动扩容为原容量的2倍

3. 存值时通过**当前容量-1**(二进制)与 **key的hashCode值 **(二进制)做按位与(&)操作 来确认存放在数组的下标位置 

   因此 hashMap的容量必须为2的倍数 如果容量为奇数 二进制最后一位为0  这样任何hash值都只会被散列到数组的偶数下标位置上，这便浪费了近一半的空间。所以，**容量取2的整数次幂，是为了使不同hash值发生碰撞的概率较小，这样就能使元素在哈希表中均匀地散列。**
   
   假设当前HashMap的容量为 16 这时调用put方法存值  key的hashCode方法返回20 那么这个值因该存放在数组的位子：
   
   ​                                    0  1  1  1  1        (16-1 的二进制)
   
   ​                                    1  0  1  0  0        (20 的二进制)
   
   ​									进行&操作后为 4则存放在下标为4的位子
   
   
   
4. 取值时 同样先调用key的hashCode方法 然后算出存放位置 然后由于是链表结构需要遍历链表与每一个Node节点的key值进行equahs比较 最终获取对应的值





# **一，JDK1.8中的涉及到的数据结构**

1 数组元素Node<K,V>实现了Entry接口

```java
//Node是单向链表，它实现了Map.Entry接口  
static class Node<k,v> implements Map.Entry<k,v> {  
    final int hash;  
    final K key;  
    V value;  
    Node<k,v> next;  
    //构造函数Hash值 键 值 下一个节点  
    Node(int hash, K key, V value, Node<k,v> next) {  
        this.hash = hash;  
        this.key = key;  
        this.value = value;  
        this.next = next;  
    }  
   
    public final K getKey()        { return key; }  
    public final V getValue()      { return value; }  
    public final String toString() { return key + = + value; }  
   
    public final int hashCode() {  
        return Objects.hashCode(key) ^ Objects.hashCode(value);  
    }  
   
    public final V setValue(V newValue) {  
        V oldValue = value;  
        value = newValue;  
        return oldValue;  
    }  
    //判断两个node是否相等,若key和value都相等，返回true。可以与自身比较为true  
    public final boolean equals(Object o) {  
        if (o == this)  
            return true;  
        if (o instanceof Map.Entry) {  
            Map.Entry<!--?,?--> e = (Map.Entry<!--?,?-->)o;  
            if (Objects.equals(key, e.getKey()) &&  
                Objects.equals(value, e.getValue()))  
                return true;  
        }  
        return false;  
    }
```



2 红黑树

```java
//红黑树  
static final class TreeNode<k,v> extends LinkedHashMap.Entry<k,v> {  
    TreeNode<k,v> parent;  // 父节点  
    TreeNode<k,v> left; //左子树  
    TreeNode<k,v> right;//右子树  
    TreeNode<k,v> prev;    // needed to unlink next upon deletion  
    boolean red;    //颜色属性  
    TreeNode(int hash, K key, V val, Node<k,v> next) {  
        super(hash, key, val, next);  
    }  
   
    //返回当前节点的根节点  
    final TreeNode<k,v> root() {  
        for (TreeNode<k,v> r = this, p;;) {  
            if ((p = r.parent) == null)  
                return r;  
            r = p;  
        }  
    }
```



# 二 HashMap的属性



```java
public class HashMap<k,v> extends AbstractMap<k,v> implements Map<k,v>, Cloneable, Serializable {  
    private static final long serialVersionUID = 362498820763181265L;  
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; //  
    static final int MAXIMUM_CAPACITY = 1 << 30;//最大容量  
    static final float DEFAULT_LOAD_FACTOR = 0.75f;//默认增长因子  
    //当add一个元素到某个位桶，其链表长度达到8时将链表转换为红黑树  
    static final int TREEIFY_THRESHOLD = 8;  
    static final int UNTREEIFY_THRESHOLD = 6;  
    static final int MIN_TREEIFY_CAPACITY = 64;  
    transient Node<k,v>[] table;//存储元素的数组  
    transient Set<map.entry<k,v>> entrySet;  
    transient int size;//存放元素的个数  
    transient int modCount;//被修改的次数fast-fail机制  
    int threshold;//临界值 当实际大小(容量*填充比)超过临界值时，会进行扩容   
    final float loadFactor;//增长因子
```



# 三 HashMap的构造函数

HashMap的构造方法有4种，主要涉及到的参数有，指定初始容量，指定填充比和用来初始化的Map



```java
//构造函数1  
public HashMap(int initialCapacity, float loadFactor) {  
    //指定的初始容量非负  
    if (initialCapacity < 0)  
        throw new IllegalArgumentException(Illegal initial capacity:  +  
                                           initialCapacity);  
    //如果指定的初始容量大于最大容量,置为最大容量  
    if (initialCapacity > MAXIMUM_CAPACITY)  
        initialCapacity = MAXIMUM_CAPACITY;  
    //填充比为正  
    if (loadFactor <= 0 || Float.isNaN(loadFactor))  
        throw new IllegalArgumentException(Illegal load factor:  +  
                                           loadFactor);  
    this.loadFactor = loadFactor;  
    this.threshold = tableSizeFor(initialCapacity);//新的扩容临界值  
}  
   
//构造函数2  
public HashMap(int initialCapacity) {  
    this(initialCapacity, DEFAULT_LOAD_FACTOR);  
}  
   
//构造函数3  
public HashMap() {  
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted  
}  
   
//构造函数4用m的元素初始化散列映射  
public HashMap(Map<!--? extends K, ? extends V--> m) {  
    this.loadFactor = DEFAULT_LOAD_FACTOR;  
    putMapEntries(m, false);  
}
```



# HashMap的存值和取值机制

HashMap如何getValue值，看源码

```java
public V get(Object key) {  
        Node<K,V> e;  
        return (e = getNode(hash(key), key)) == null ? null : e.value;  
    }  
      /** 
     * Implements Map.get and related methods 
     * 
     * @param hash hash for key 
     * @param key the key 
     * @return the node, or null if none 
     */  
    final Node<K,V> getNode(int hash, Object key) {  
        Node<K,V>[] tab;//Entry对象数组  
    Node<K,V> first,e; //在tab数组中经过散列的第一个位置  
    int n;  
    K k;  
    /*找到插入的第一个Node，方法是hash值和n-1相与，tab[(n - 1) & hash]*/  
    //也就是说在一条链上的hash值相同的  
        if ((tab = table) != null && (n = tab.length) > 0 &&(first = tab[(n - 1) & hash]) != null) {  
    /*检查第一个Node是不是要找的Node*/  
            if (first.hash == hash && // always check first node  
                ((k = first.key) == key || (key != null && key.equals(k))))//判断条件是hash值要相同，key值要相同  
                return first;  
      /*检查first后面的node*/  
            if ((e = first.next) != null) {  
                if (first instanceof TreeNode)  
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);  
                /*遍历后面的链表，找到key值和hash值都相同的Node*/  
                do {  
                    if (e.hash == hash &&  
                        ((k = e.key) == key || (key != null && key.equals(k))))  
                        return e;  
                } while ((e = e.next) != null);  
            }  
        }  
        return null;  
    }
```



HashMap如何put(key，value);看源码

```java
public V put(K key, V value) {  
        return putVal(hash(key), key, value, false, true);  
    }  
     /** 
     * Implements Map.put and related methods 
     * 
     * @param hash hash for key 
     * @param key the key 
     * @param value the value to put 
     * @param onlyIfAbsent if true, don't change existing value 
     * @param evict if false, the table is in creation mode. 
     * @return previous value, or null if none 
     */  
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,  
                   boolean evict) {  
        Node<K,V>[] tab;   
    Node<K,V> p;   
    int n, i;  
        if ((tab = table) == null || (n = tab.length) == 0)  
            n = (tab = resize()).length;  
    /*如果table的在（n-1）&hash的值是空，就新建一个节点插入在该位置*/  
        if ((p = tab[i = (n - 1) & hash]) == null)  
            tab[i] = newNode(hash, key, value, null);  
    /*表示有冲突,开始处理冲突*/  
        else {  
            Node<K,V> e;   
        K k;  
    /*检查第一个Node，p是不是要找的值*/  
            if (p.hash == hash &&((k = p.key) == key || (key != null && key.equals(k))))  
                e = p;  
            else if (p instanceof TreeNode)  
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);  
            else {  
                for (int binCount = 0; ; ++binCount) {  
        /*指针为空就挂在后面*/  
                    if ((e = p.next) == null) {  
                        p.next = newNode(hash, key, value, null);  
               //如果冲突的节点数已经达到8个，看是否需要改变冲突节点的存储结构，　　　　　　　　　　　　　  
　　　　　　　　　　　　//treeifyBin首先判断当前hashMap的长度，如果不足64，只进行  
                        //resize，扩容table，如果达到64，那么将冲突的存储结构为红黑树  
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st  
                            treeifyBin(tab, hash);  
                        break;  
                    }  
        /*如果有相同的key值就结束遍历*/  
                    if (e.hash == hash &&((k = e.key) == key || (key != null && key.equals(k))))  
                        break;  
                    p = e;  
                }  
            }  
    /*就是链表上有相同的key值*/  
            if (e != null) { // existing mapping for key，就是key的Value存在  
                V oldValue = e.value;  
                if (!onlyIfAbsent || oldValue == null)  
                    e.value = value;  
                afterNodeAccess(e);  
                return oldValue;//返回存在的Value值  
            }  
        }  
        ++modCount;  
     /*如果当前大小大于门限，门限原本是初始容量*0.75*/  
        if (++size > threshold)  
            resize();//扩容两倍  
        afterNodeInsertion(evict);  
        return null;  
    }
```

# HashMap的扩容机制

构造hash表时，如果不指明初始大小，默认大小为16（即Node数组大小16），如果Node[]数组中的元素达到（填充比\*Node.length）重新调整HashMap大小 变为原来2倍大小,扩容很耗时

```java
/** 
    * Initializes or doubles table size.  If null, allocates in 
    * accord with initial capacity target held in field threshold. 
    * Otherwise, because we are using power-of-two expansion, the 
    * elements from each bin must either stay at same index, or move 
    * with a power of two offset in the new table. 
    * 
    * @return the table 
    */  
   final Node<K,V>[] resize() {  
       Node<K,V>[] oldTab = table;  
       int oldCap = (oldTab == null) ? 0 : oldTab.length;  
       int oldThr = threshold;  
       int newCap, newThr = 0;  
      
/*如果旧表的长度不是空*/  
       if (oldCap > 0) {  
           if (oldCap >= MAXIMUM_CAPACITY) {  
               threshold = Integer.MAX_VALUE;  
               return oldTab;  
           }  
/*把新表的长度设置为旧表长度的两倍，newCap=2*oldCap*/  
           else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&  
                    oldCap >= DEFAULT_INITIAL_CAPACITY)  
      /*把新表的门限设置为旧表门限的两倍，newThr=oldThr*2*/  
               newThr = oldThr << 1; // double threshold  
       }  
    /*如果旧表的长度的是0，就是说第一次初始化表*/  
       else if (oldThr > 0) // initial capacity was placed in threshold  
           newCap = oldThr;  
       else {               // zero initial threshold signifies using defaults  
           newCap = DEFAULT_INITIAL_CAPACITY;  
           newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);  
       }  
      
      
      
       if (newThr == 0) {  
           float ft = (float)newCap * loadFactor;//新表长度乘以加载因子  
           newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?  
                     (int)ft : Integer.MAX_VALUE);  
       }  
       threshold = newThr;  
       @SuppressWarnings({"rawtypes","unchecked"})  
/*下面开始构造新表，初始化表中的数据*/  
       Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];  
       table = newTab;//把新表赋值给table  
       if (oldTab != null) {//原表不是空要把原表中数据移动到新表中      
           /*遍历原来的旧表*/        
           for (int j = 0; j < oldCap; ++j) {  
               Node<K,V> e;  
               if ((e = oldTab[j]) != null) {  
                   oldTab[j] = null;  
                   if (e.next == null)//说明这个node没有链表直接放在新表的e.hash & (newCap - 1)位置  
                       newTab[e.hash & (newCap - 1)] = e;  
                   else if (e instanceof TreeNode)  
                       ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);  
/*如果e后边有链表,到这里表示e后面带着个单链表，需要遍历单链表，将每个结点重*/  
                   else { // preserve order保证顺序  
                ////新计算在新表的位置，并进行搬运  
                       Node<K,V> loHead = null, loTail = null;  
                       Node<K,V> hiHead = null, hiTail = null;  
                       Node<K,V> next;  
                      
                       do {  
                           next = e.next;//记录下一个结点  
          //新表是旧表的两倍容量，实例上就把单链表拆分为两队，  
　　　　　　　　　　　　　//e.hash&oldCap为偶数一队，e.hash&oldCap为奇数一对  
                           if ((e.hash & oldCap) == 0) {  
                               if (loTail == null)  
                                   loHead = e;  
                               else  
                                   loTail.next = e;  
                               loTail = e;  
                           }  
                           else {  
                               if (hiTail == null)  
                                   hiHead = e;  
                               else  
                                   hiTail.next = e;  
                               hiTail = e;  
                           }  
                       } while ((e = next) != null);  
                      
                       if (loTail != null) {//lo队不为null，放在新表原位置  
                           loTail.next = null;  
                           newTab[j] = loHead;  
                       }  
                       if (hiTail != null) {//hi队不为null，放在新表j+oldCap位置  
                           hiTail.next = null;  
                           newTab[j + oldCap] = hiHead;  
                       }  
                   }  
               }  
           }  
       }  
       return newTab;  
   }
```