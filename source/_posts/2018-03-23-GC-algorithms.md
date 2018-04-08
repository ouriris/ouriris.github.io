---
title: Java 垃圾回收机制之列举几种垃圾收集算法
date: 2018-03-23 16:15:04
tags:
- Java
- 进击的Java新人
---

Author: Mike Xie


---
# 垃圾收集算法

目前最基本的垃圾收集算法有四种,    

1. 标记-清除算法(mark-sweep),    
2. 标记-压缩算法(mark-compact),    
3. 复制算法(copying)   
4. 引用计数算法(reference counting).    

其中标志-清除算法、标记-压缩算法、复制算法判断对象存活与否基于 可达性分析算法。

而现代流行的垃圾收集算法一般是由这四种中的其中几种算法**相互组合**而成，比如说，对堆(heap)的一部分采用标记-清除算法，对堆(heap)的另外一部分则采用复制算法等等


---

## 标记清除算法


### 基本概念
#### mutator && collector
collector指的就是垃圾收集器，而mutator是指除了垃圾收集器之外的部分(应用程序本身)

比如说我们应用程序本身。mutator的职责一般是NEW(分配内存),READ(从内存中读取内容),WRITE(将内容写入内存)，而collector则就是回收不再使用的内存来供mutator进行NEW操作的使用

#### 可达性
[从mutator根对象开始进行遍历，可以被访问到的对象都称为是可达对象。这些对象也是mutator(你的应用程序)正在使用的对象](https://www.jianshu.com/p/b0f5d21fe031)


### 原理
[顾名思义，标记-清除算法分为两个阶段，标记(mark)和清除(sweep).](https://www.jianshu.com/p/b0f5d21fe031)
> 基于可达性分析，回头看看可达性分析存在那几个依据


#### 标记阶段
在标记阶段，collector从mutator根对象开始进行遍历，对从mutator根对象 **可以访问到的对象** 都打上一个标识，一般是在对象的header中，将其记录为可达对象。

![image](https://note.youdao.com/yws/public/resource/abc19d0c87c72c545ff1038c680f3564/xmlnote/6EFDD88D2C7E46B9B73DDC87D0755577/17933)

从上图我们可以看到
1. 在Mark阶段，从根对象1可以访问到B对象，从B对象又可以访问到E对象，所以B,E对象都是可达的。
2. 同理，F,G,J,K也都是可达对象。


#### 清除阶段
collector对堆内存(heap memory)**从头到尾进行线性的遍历**，如果发现某个对象没有标记为可达对象-通过读取对象的header信息，则就将其回收


![image](https://note.youdao.com/yws/public/resource/abc19d0c87c72c545ff1038c680f3564/xmlnote/2F5CF8DE4DE046D3BF47EA113EE6608E/17935)

到了Sweep阶段，所有非可达对象都会被collector回收。

同时，**Collector在进行标记和清除阶段时会将整个应用程序暂停(mutator)，等待标记清除结束后才会恢复应用程序的运行，这也是Stop-The-World这个单词的来历**。


---

#### 垃圾收集动作是怎么被触发的?

下面是mutator进行NEW操作的伪代码：

```
New():
    ref <- allocate()  //分配新的内存到ref指针
    if ref == null
       collect()  //内存不足，则触发垃圾收集
       ref <- allocate() //再次分配 (存在 OutOfMemory 的隐患，下面会分析)
       if ref == null
          throw "Out of Memory"   //垃圾收集后仍然内存不足，则抛出Out of Memory错误
          return ref

atomic collect():
    markFromRoots()
    sweep(HeapStart,HeapEnd)

```

标记算法
```
markFromRoots():
    worklist <- empty
    for each fld in Roots  //遍历所有mutator根对象
        ref <- *fld
        if ref != null && isNotMarked(ref)  //如果它是可达的而且没有被标记的，直接标记该对象并将其加到worklist中
           setMarked(ref)
           add(worklist,ref)
           mark()
mark(): 
    while not isEmpty(worklist)
          ref <- remove(worklist)  //将worklist的最后一个元素弹出，赋值给ref
          for each fld in Pointers(ref)  //遍历ref对象的所有指针域，如果其指针域(child)是可达的，直接标记其为可达对象并且将其加入worklist中
          //通过这样的方式来实现深度遍历，直到将该对象下面所有可以访问到的对象都标记为可达对象。
                child <- *fld
                if child != null && isNotMarked(child)
                   setMarked(child)
                   add(worklist,child)

```
在mark阶段结束后，sweep算法就比较简单了，它就是从堆内存起始位置开始，线性遍历所有对象直到堆内存末尾，如果该对象是可达对象的（在mark阶段被标记过的），那就直接去除标记位（为下一次的mark做准备），如果该对象是不可达的，直接释放内存。

```java
sweep(start,end):
    scan <- start   //从堆内存起始位置开始
   while scan < end
       if isMarked(scan)   //如果该对象是可达对象的（在mark阶段被标记过的）
          setUnMarked(scan)   //那就直接去除标记位（为下一次的mark做准备）
      else
          free(scan)   // 如果该对象是不可达的，直接释放内存。
      scan <- nextObject(scan)
```

#### 缺点
1. 效率问题： **标记 清除过程效率都不高**，主要是频繁遍历堆内存
2. 空间问题： **产生大量碎片**

标记-清除算法的比较大的缺点就是垃圾收集后有可能会造成大量的内存碎片，像上面的图片所示，垃圾收集后内存中存在三个内存碎片，假设一个方格代表1个单位的内存，如果有一个对象需要占用3个内存单位的话，**那么就会导致Mutator一直处于暂停状态，而Collector一直在尝试进行垃圾收集，直到Out of Memory**。




---
## 标记-压缩算法

### 前言

解决内部碎片的问题

内存碎片一直是 **非移动垃圾回收器** [(指在垃圾回收时不进行对象的移动)的一个问题，比如说在前面的标记-清除垃圾回收器就有这样的问题。而标记-压缩垃圾回收算法能够有效的缓解这一问题。](https://www.jianshu.com/p/698eb5e1ccb9)


### 算法原理

#### 标记阶段
也分为两个阶段，一个是标记(mark)，一个是压缩(compact). 其中标记阶段跟标记-清除算法中的 **标记阶段是一样的**


#### 压缩阶段
它的工作就是移动所有的可达对象到堆内存的**同一个区域**中，使他们紧凑的排列在一起

从而**将所有非可达对象释放出来的空闲内存都集中在一起**，通过这样的方式来达到减少内存碎片的目的

##### 移动对象的顺序
在压缩阶段，由于要移动 **可达对象**，那么需要考虑移动对象时的顺序，一般分为下面三种

**1 任意顺序**

即不考虑原先对象的排列顺序，也不考虑对象间的引用关系，随意的移动可达对象，这样可能会有内存访问的局部性问题。

前：![image](https://note.youdao.com/yws/public/resource/abc19d0c87c72c545ff1038c680f3564/xmlnote/7FE7BEDA1BBC48B591EA0F3598362272/17989)

后：![image](https://note.youdao.com/yws/public/resource/abc19d0c87c72c545ff1038c680f3564/xmlnote/AE9B31F630324DE6B8DD8F7D2A0385C7/17991)

**2 线性顺序**

在重新排列对象时，会考虑对象间的引用关系，比如A对象引用了B对象，那么就会尽可能的将A，B对象排列在一起。

**3 滑动顺序**

顾名思义，就是在重新排列对象时，将对象按照原先堆内存中的排列顺序滑动到堆的一端。

前：![image](https://note.youdao.com/yws/public/resource/abc19d0c87c72c545ff1038c680f3564/xmlnote/6A6C9A4879A14945A885A67E53C2E0BD/17985)

后：  ![image](https://note.youdao.com/yws/public/resource/abc19d0c87c72c545ff1038c680f3564/xmlnote/988888D3F77640399F088A645EC42387/17987)



---
### Two-Finger 算法
#### 前言
Two-Finger算法来自Edwards, 它在**压缩阶段移动对象时是任意顺序移动的**

它最适用于处理包含**固定大小对象**的内存区域。由于Mark阶段都是跟标记-清除算法一致的，这里我们只关注Compact阶段。

#### 原理

Two-Finger算法是一个Two Passes算法，即需要遍历堆内存两次   
**第一次遍历是将堆末尾的可达对象移动到堆开始的空闲内存单元去**，    
**第二次遍历则需要修改可达对象的引用**，因为一些可达对象已经被移动到别的地址，而原先引用它们的对象还指向着它们移动前的地址。

在这两次遍历过程中，首尾两个指针分别从堆的头尾两个位置向中间移动，直至两个指针相遇，由于它们的运动轨迹酷似两根手指向中间移动的轨迹，因此称为Two Finger算法。

![image](http://www.processon.com/chart_image/530043370cf2a3dc99dd935b.png)

##### 第一次遍历
###### 代码
```java
compact():
    relocate(HeapStart,HeapEnd) //重新定位可达对象的位置
    updateReferences(HeapStart,free) //修改可达对象的引用

relocate(start,end)
    free <- start
    scan <- end
    
    while free < scan 
        //找到一个可以被释放的空间
        while isMarked(free)
            unsetMarked(free)
            free <- free + size(free)
        
        //找到一个可以移动的可达对象
        while not isMarked(scan) && scan > free
            scan <- scan - size(scan)
            
        if scan > free
            unsetMarked(scan)
            move(scan, free) //将scan位置的可达对象移动到free位置上
            *scan <- free //将可达对象移动后的位置写到原先可达对象处于的位置
            free <- free + size(free)
            scan <- scan - size(scan)

```
第一次遍历的原理是

![image](http://www.processon.com/chart_image/530043370cf2a3dc99dd935b.png)
1. 头指针(free)沿着堆头向堆尾前进，直到找到一个空闲的内存单元（即没有被标记为可达对象的内存单元），如遇到可达对象，则清除其标记。
2. 接着尾指针(scan)从堆尾向堆头方向前进，直到找到一个被标记为可达的内存单元。
3. 最后，collector将可达对象从尾指针(scan)指向的位置移动到头指针(free)指向的位置，**最后将可达对象移动后的位置(当前free指针指向的位置)写到原先可达对象处于的位置(当前尾指针scan指向的位置), 为下一次的遍历 - 更新对象相互间的引用做好准备**。
4. 注：当移动可达对象时，其引用的对象在可达对象移动后保持不变，如下图中的G对象移动后依然指向位置5和位置10。

###### 示例图

![image](http://www.processon.com/chart_image/530175170cf2a3dc99de59d9.png)


---
##### 第二次遍历

**为了更新引用关系**
> 可见是需要STW的

###### 目的
一个可达对象可以被其他对象引用，比如上图中的K对象，如果其被移动后，引用它的对象比如说G并不知道它被移动了，那么这第二次的遍历就是为了告诉G它所引用的对象K已经被移动到新的位置上去了，它需要更新它对K的引用。

###### 代码

```
updateReferences(start,end)
    for each fld in Roots //先更新mutator根对象所引用的对象关系
        ref <- *fld
        if ref >= end
            *fld <- *ref 
    scan <- start
    while scan < ned
        for each fld in Pointers(scan)
            ref <- * fld
            if ref >= end 
                *fld <- *ref
        scan <- scan + size(scan)

```

###### 分析
1. 第二次遍历，collector先会对根对象进行遍历   

2. 比如根对象2引用着位置6的内存单元，根据算法，该位置**大于等于end指针所指向的位置** - 即第一次遍历free指针和scan指针相遇的位置，那么我们就认为这个位置的对象已经被移动，需要更新根对象2的引用关系，即从引用位置6改为引用位置2(位置6的内存单元中记录着该对象被移动后的新位置)。

3. 同理，在移动G对象的时候，也是要判断看G所引用的内存单元位置是否大于end指针指向的位置，如果小于，则不处理。否则则修改G的引用关系。

###### 示例图

![image](http://www.processon.com/chart_image/53006f070cf2a3dc99ddbc95.png)





==还有另外一种算法的==

---

## 复制算法

### 前言
1. [半区复制算法](https://www.jianshu.com/p/74659de07264)的目的也是为了更好的 **缓解内存碎片问题**。
2. 对比于标记-压缩算法, 它不需要遍历堆内存那么多次，**节约了时间**，但是它也带来了一个主要的缺点，那就是相比于标记-清除和标记-压缩垃圾回收器，它的**可用堆内存减少了一半**。
3. 同时对于大对象，复制比标记的代价更大。所以**半区复制算法更一般适合回收小的，存活期短的对象**。

### 三色抽象法
在我们深入半区复制算法原理前，我们需要了解下什么是三色抽象法。对于一个对象，垃圾收集器可以将其标记为灰色，黑色和白色中的一种，每种颜色代表不同的含义，

1. 灰色 - 表示垃圾收集器已经**访问过该对象**，但是还**没有访问过它的所有孩子节点**。
2. 黑色 - 表示**该对象以及它的所有孩子节点都已经被垃圾收集器访问过**了。
3. 白色 - 表示该对象**从来没有被垃圾收集器访问过**，这就是非可达对象。

三色抽象法也可以用在标记-清除算法和标记-压缩算法。当垃圾收集结束后，可达对象都被标记为黑色，非可达对象都被标记为白色，不会有灰色对象存在。在半区复制算法里，我们也采用了三色抽象法来标记对象。

### 算法原理
之所以叫半区复制，是因为它将堆内存对半分为两个半区，只用其中一个半区来进行对象内存的分配，   
如果在这个半区内存不够给新的对象分配了，那么就开始进行垃圾收集，      
将这个半区中的所有可达对象都拷贝到另外一个半区中去，    
然后继续在另外那个半区进行新对象的内存分配。半区复制算法中比较常见是Cheney算法

### 代码
结合图形去分析
```java
atomic collect():
    flip()
    scan <- free
    for each fld in Roots  //遍历处理根对象A, B
        process(fld)
    while not isEmpty(worklist)
        ref <- remove(worklist)
        scan(ref)

flip():
    fromspace, tospace <- tospace, fromspace  //交换左右半区
    top <- tospace+extent  //界定堆内存的边界
    free <- tospace

scan(ref):
    for each fld in Pointers(ref)
        process(fld)

process(fld):  //处理根对象
    fromRef <- *fld
    if fromRef != null
        *fld <- forward(fromRef) //将可达对象复制后的地址写到原对象的位置上，当作迁移地址
    
forward(fromRef):
    toRef <- forwardingAddress(fromRef) //读取该位置上对象的迁移地址
    if toRef == null //如果该位置上的对象没有迁移地址，那就说明它还没有被复制，需要复制到tospace中去
        toRef <- copy(fromRef)
    return toRef
    
copy(fromRef):
    toRef <- free 
    free <- free + size(fromRef)
    move(fromRef, toRef) //从fromRef位置复制对象到toRef位置上
    forwardingAddress(fromRef) <- toRef  //将地址toRef写到fromRef位置上的对象中去
    add(worklist, toRef)
    return toRef

remove(worklist):
    ref <- scan
    scan <- scan + size(scan)
    return ref

```

### 分析
实际上半区复制算法的实现跟标记-压缩算法的实现差不多， 都是采用的深度遍历算法，理解该算法的关键点是，
1. 怎么计算可达对象的迁移地址(forwardingAddress) - 看copy(fromRef)方法的实现， 
2. 以及怎么更新对象间的引用关系 ?

![image](http://www.processon.com/chart_image/5304731d0cf2ceacbe5737cd.png)


我们假设A,B对象是根对象。

1. 首先先交换左右半区(ToSpace, FromSpace), 同时设置free指针和top指针。

2. 遍历处理根对象A,B。先将A对象复制到free指针指向的位置，同时将A对象复制后的地址(迁移地址)写到原先A对象所在的位置，图中虚线的箭头表示。可以看到A对象已经被collector访问过了，但是还没有访问其孩子节点，所以将其标为了灰色。紧接着scan，free指针继续向前移动。

3. 由于是深度遍历算法，紧接collector会先遍历处理A对象所引用的对象C，当发现对象C没有迁移地址时，说明它还没有被复制，由于它又是可达对象，所以接着collector会将它复制到当前free指针指向的位置，即对象A后面。对象C复制完后，会用其复制后的地址来更新A原先对C的引用，同时也写到原先C对象所在的地址上。

4. 接着collector会处理对象C的孩子节点（深度遍历算法），由于对象C没有引用任何对象，于是对象C的处理结束，将其标记为黑色。然后collector接着处理A对象的另外一个孩子节点E对象，处理方式跟处理对象C一致。
5. 对象E也没有孩子节点，collector也将其标识为黑色。

6. 到目前为此，A对象也全部处理结束了，于是collector将其标识为黑色，然后接着去处理对象B。当复制B对象结束后，发现B对象所引用的对象C有迁移地址，于是就更新其对对象C的引用，使其指向FromSpace半区中对象C的迁移地址 - 即C对象复制后所在ToSpace的地址。这个情况下就不需要再次复制对象C了。

7. 当所有的可达对象都从FromSpace半区复制到ToSpace半区后，垃圾收集结束。新对象的内存分配从free指针指向的位置开始进行分配。





---

## 引用计数算法

1. [reference counting](http://www.memorymanagement.org/glossary/r.html#reference.counting)

2. 现代编程语言比如Lisp，Python，Ruby等的垃圾收集算法采用的就是引用计数算法

### 算法原理

> 通过在 **对象头** 中分配一个空间来保存该对象被引用的次数。如果该对象被其它对象引用，则它的引用计数加一，如果删除对该对象的引用，那么它的引用计数就减一，当该对象的引用计数为0时，那么该对象就会被[回收](https://www.jianshu.com/p/1d5fa7f6035c)

### 例子
比如说，当我们编写以下代码时，

```
String p = new String("abc")
```

abc这个字符串对象的引用计数值为1.（rc = reference count）

![image](http://www.processon.com/chart_image/53080dca0cf262b559f77618.png)
 
而当我们 **去除** abc字符串对象的引用时，则abc字符串对象的引用计数减1


```
p = null // = null, 去除对象引用的一种方式
```

![image](http://www.processon.com/chart_image/53080eba0cf262b559f77734.png)

### 剖析
> 注意其中的对比

#### 垃圾回收时间不一样
1. 由此可见，当**对象的引用计数为0时，垃圾回收就发生了**。
2. 这跟前面三种垃圾收集算法不同，前面三种垃圾收集都是**在为新对象分配内存空间时由于内存空间不足而触发的**

#### 回收对象不一样
1. 前三种垃圾收集是针对**整个堆**中的所有对象进行的。
2. 而引用计数垃圾收集机制不一样，它只是在引用计数变化为0时即刻发生，而且**只针对某一个对象以及它所依赖的其它对象**。所以，我们一般也称呼引用计数垃圾收集为直接的垃圾收集机制，而前面三种都属于间接的垃圾收集机制。

而采用引用计数的垃圾收集机制跟前面三种垃圾收集机制最大的不同在于，垃圾收集的开销被分摊到整个应用程序的运行当中了，而不是在进行垃圾收集时，要挂起整个应用的运行，直到对堆中所有对象的处理都结束。因此，采用引用计数的垃圾收集不属于严格意义上的"Stop-The-World"的垃圾收集机制。这个也可以从它的伪代码实现中看出：

#### 伪代码

```
New(): //分配内存
    ref <- allocate()
    if ref == null
        error "Out of memory"
    rc(ref) <- 0  //将ref的引用计数(reference counting)设置为0
    return ref

atomic Write(dest, ref) //更新对象的引用
    addReference(ref)
    deleteReference(dest)
    dest <- ref

addReference(ref):
    if ref != null
        rc(ref) <- rc(ref)+1
        
deleteReference(ref):
    if ref != null
        rc(ref) <- rc(ref) -1
        if rc(ref) == 0 //如果当前ref的引用计数为0，则表明其将要被回收
            for each fld in Pointers(ref)
                deleteReference(*fld)
            free(ref) //释放ref指向的内存空间

```

对于上面的伪代码，重点在于理解两点，
1. 第一个是当对象的**引用发生变化时，比如说将对象重新赋值给新的变量等，对象的引用计数如何变化**。
2. 假设我们有两个变量p和q，它们分别指向不同的对象，当我们将他们指向同一个对象时，下面的图展示了p和q变量指向的两个对象的引用计数的变化。


```
String p = new String("abc")
String q = new String("def")
p = q
```

当我们执行代码p=q时，实际上相当于调用了伪代码中的**Write(p,q)**， 即对p原先指向的对象要进行deleteReference()操作 - 引用计数减一，因为p变量不再指向该对象了，而对q原先指向的对象要进行addReference()操作 - 引用计数加一。

![image](http://www.processon.com/chart_image/5309c1090cf262b559f8bc25.png)

第二点需要理解的是，**当某个对象的引用计数减为0时，collector需要递归遍历它所指向的所有域，将它所有域所指向的对象的引用计数都减一，然后才能回收当前对象。在递归过程中，引用计数为0的对象也都将被回收**，比如说下图中的phone和address指向的对象。

![image](http://www.processon.com/chart_image/5309cc030cf262b559f8cbed.png)


#### 环形数据问题

但是这种引用计数算法有一个比较大的问题，那就是它不能处理环形数据 - 即如果有两个对象相互引用，那么这两个对象就不能被回收，因为它们的引用计数始终为1。这也就是我们常说的“内存泄漏”问题。比如下图展示的将p变量赋值为null值后所出现的内存泄漏。

![image](http://www.processon.com/chart_image/5309cefc0cf262b559f8d040.png)

---

##### 概念


##### 引用计数法存在什么问题？

##### 引用计数到底是如何维护所有对象引用的?
- [经典](https://www.zhihu.com/question/21539353/answer/18596488)