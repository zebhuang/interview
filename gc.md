# 垃圾回收
GC 是 Garbage Collection 的简称，中文成为 “垃圾回收”。

GC 要做的有两件事：
- 找到内存空间里的垃圾
- 回收垃圾

内存泄漏：使用完内存后忘记释放

悬垂指针：指针指向已释放内存的地址空间
## 标记清除（Mark Sweep）
标记清除算法由标记阶段和清除阶段构成。
- 标记阶段：遍历所有活动对象并标记，一般采用深度优先搜索。
- 清除阶段：把那些没有标记的对象，也就是非活动对象回收的阶段。

**伪代码**

```
mark_sweep(){
    mark_phase()
    sweep_phase()
}

mark_phase(){
    for(r : $roots)
	mark(*r)
}

// 深度优先遍历
mark(obj){
    if(obj.mark == FALSE)
	obj.mark = TRUE
	for(child : children(obj))
	    mark(*child)
}

sweep_phase(){
    sweeping = $heap_start
    while(sweeping < $heap_end)
	if(sweeping.mark == TRUE)
	    sweeping.mark = FALSE
	else
	    // 检查这次发现的分块是否和上次发现的分块是否连续
	    // 如果分块连续，则将邻接的2个分块合并整理成1个分块
	    if(sweeping == $free_list + $free_list.size)
		$free_list.size += sweeping.size
	    else
		sweeping.next = $free_list
		$free_list = sweeping
	sweeping += sweeping.size
}

// 执行内存分配
new_obj(size){
    chunk = pickup_chunk(size, $free_list)
    if(chunk != NULL)
	return chunk
    else
	allocation_fail()
}
```

**优点**

- 实现简单
- 与保守式GC算法兼容

**缺点**

- 碎片化

### 多个空闲链表
之前采用的是一个空闲链表，在这个链表中，对大的分块和小的分块进行同样的处理。但是这样依赖，每次分配的时候都要遍历一次空闲链表来寻找合适大小的分块，这样非常浪费时间。

因此我们有一种方法，就是利用分块大小不同的空闲链表，即创建只连接大分块的空闲链表和只连接小分块的空闲链表。这样一来，只要按照所申请的分块大小选择空闲链表，就能在短时间内找到符合条件的分块了。

用一个数组来维护不同分块大小的链表，数组的各个元素都位于空闲链表的前面，第一个元素是由2个字的分块连接的空闲链表的开头，第2个元素是由3个字的分块连接的空闲链表的开头。因此，例如在分配3个字的分块时，只要查询用于3个字的空闲链表就够了。比起只利用一个空闲链表来说，此方法大幅节约了分配所需要的时间。

一般情况下，很少会申请非常大的分块。因此，我们通常会给分块大小设定一个上限，分块如果大于等于这个大小，就全部采用一个空闲链表处理。例如，如果设定分块大小上限为 100 个字，那么准备用于2个字、3个字、...、100个字，以及大于等于101个字的总共100个空闲链表就可以了。

利用多个空闲链表时，我们需要修正`new_obj()`函数以及`sweep_phase()`函数。
```
new_obj(size){
    index = size / (WORD_LENGTH / BYTE_LENGTH)
    if(index <= 100)
	if($free_list[index] != NULL)
	    chunk = $free_list[index]
	    $free_list[index] = $free_list[index].next
	    return chunk
    else
	chunk = pickup_chunk(size, $free_list[101])
	if(chunk != NULL)
	    return chunk
 

    allocation_fail()
}

sweep_phase(){
    for(i : 2..101)
	$free_list[i] = NULL

    sweeping = $heap_start

    while(sweeping < $heap_end)
	if(sweeping.mark == TRUE)
	    sweeping.mark = FALSE
	else
	index = size / (WORD_LENGTH / BYRE_LENGTH)
	    if(index <= 100)
		sweeping.next = $free_list[index]
		$free_list[index] = sweeping
	    else
		sweeping.next = $free_list[101]
		$free_list[101] = sweeping
	    sweeping += sweeping.size
}
```

### BIBOP 法
BIBOP（Big Bag Of Pages），将大小相近的对象整理成固定大小的块进行管理的做法。标记清除法会发生碎片化，碎片化的原因之一就是堆上杂乱散步者大小各异的对象。对此，我们可以把堆分割成固定大小的块，让每个块只能配置同样大小的对象。这就是 BIBOP 法。

### 位图标记
在单纯的标记清除算法中，用于标记的位是被分配到各个对象的头中的。也就是说，算法是把对象和头一并处理的。然而这跟写时复制不兼容。

对此我们可以只收集各个对象的标志位并表格化，不跟对象一起管理。在标记的时候，不再对象的头里置位，而是在这个表格中的特定场所置位。像这样集合了用于标记的位的表格成称为“位图表格”（bitmap table），利用这个表格进行标记的行为称为“位图标记”。位图表格的实现方法有多种，例如散列表和树形结构等。为了简单起见，这里我们采用整数型数组。
```
// 位图标记中的mark()
mark(obj){
    obj_num = (obj - $heap_start) / WORD_LENGTH
    index = obj_num / WORD_LENGTH
    offset = obj_num % WORD_LENGTH

    if(($bitmap_tbl[index] & (1 << offset)) == 0)
	$bitmap_tbl[index] |= (1 << offset)
	for(child : children(obj))
	    mark(*child)
}
```
在这里，`obj_num`指的是从位图表格前面数起，obj的标志位在第几个。我们用 `obj_num` 除以 `WORD_LENGTH` 得到的商 `index` 以及余数 `offset` 来分别表示位图表格的行编号和列编号。

**优点**

- 与写时复制兼容
- 清除操作更高效

```
// 位图标记的sweep_phase()
sweep_phase(){
    sweeping = $heap_start
    index = 0
    offset = 0

    while(sweeping < $heap_end)
	if($bitmap_tbl[index] & (1 << offset) == 0)
	    sweeping.next = $free_list
	    $free_list = sweeping
	index += (offset + sweeping.size) / WORD_LENGTH
	offset = (offset + sweeping.size) % WORD_LENGTH
	sweeping += sweeping.size

    for(i : 0..(HEAP_SIZE / WORD_LENGTH - 1))
	$bitmap_tbl[i] = 0
}
```
与一般的清除阶段相同，我们用 sweeping 指针遍历整个堆。不过，这里使用了 index 和 offset 两个变量，在遍历堆的同时也遍历位图表格。

### 延迟清除法
清除操作所花费的时间是与堆大小成正比的。也就是说，处理的堆越大，标记清除算法所花费的时间就越长，结果就会妨碍内存分配的申请。

延迟清除法（Lazy Sweep）是缩短因清除操作而导致的最大暂停时间的方法。在标记操作结束后，不一并进行清除操作，而是如其字面意思一样让它“延迟”，通过“延迟”来防止内存分配长时间暂停。
```
// 延迟清除中的 new_obj()
new_obj(size){
    chunk = lazy_sweep(size)
    if(chunk != NULL)
	return chunk

    mark_phase()

    chunk = lazy_sweep(size)
    if(chunk != NULL)
	return chunk

    allocation_fail()
}
```
在内存分配时直接调用 `lazy_sweep()` 函数，进行清除操作。如果它能清除操作来分配分块，就会返回分块；如果不能分配分块，就会执行标记操作。当 lazy_sweep() 函数返回 NULL 时，也就是没有找到分块时，会调用 `mark_phase()` 函数进行一遍标记操作，再调用 `lazy_sweep()` 函数来分配分块。在这里没能分配分块也就意味着堆上没有分块，`new_obj()`也就不能再进行下一步处理来。

```
lazy_sweep(szie){
    while($sweeping < $heap_end)
	if($sweeping.mark == TRUE)
	    $sweeping.mark = FALSE
	else if($sweeping.size >= size)
	    chunk = $sweeping
	    $sweeping += $sweeping.size
	    return chunk
	$sweeping += $sweeping.size
    
    $sweeping = $heap_start
    return NULL
}
```
`lazy_sweep()` 函数会一直遍历堆，直到找到大于等于所申请大小的分块为止。在找到合适分块时会将其返回。但是在这里 $sweeping 变量是全局变量。也就是说，遍历的开始位置位于上一次清除操作中发现的分块的右边。

当 `lazy_sweep()` 函数遍历到堆最后都没有找到分块时，会返回 NULL。

因为延迟清除法不是以下遍历整个堆，它只在分配时执行必要的遍历，所以可以压缩因清除操作而导致的暂停时间。这就是 “延迟” 清除操作的意思。
## 引用计数
引用计数中引入了“计数器”的概念。计数器是无符号的整数，用于计数器的位数根据算法和实现而有所不同。

在引用计数法中没有明确启动 GC 的语句。引用计数法与增减计数器密切相关。在两种情况下计数器的值会发生增减，这设计了 `new_obj()` 和 `update_ptr()`
```
new_obj(size){
    obj = pickup_chunk(size, $free_list)

    if(obj == NULL)
	allcation_fail()
    else
	obj.ref_cnt = 1
	return obj
}

update_ptr(prt, obj){
    int_ref_cnt(obj) // 对指针ptr新引用的对象obj的计数器进行增量操作
    dec_ref_cnt(*ptr) // 对指针ptr之前引用的对象*ptr的计数器进行减量操作
    *ptr = obj
}

inc_ref_cnt(obj){
    obj.ref_cnt++
}

dec_ref_cnt(obj){
    obj.ref_cnt--
    if(obj.ref_cnt == 0)
	// 当引用数为0时，对子对象对计数器进行减量操作
	for(child : children(obj))
	    dec_ref_cnt(*child)
	reclaim(obj)
}
```

**优点**
- 可即刻回收垃圾
- 最大暂停时间短
- 没有必要沿指针查找

**缺点**
- 计数器值对增减处理繁重
- 计数器需要占用很多位
- 实现繁琐复杂
- 循环引用无法回收
### 延迟引用计数法
### Sticky 引用计数法
### 1位引用计数法
### 部分标记清除算法

## GC复制算法
GC复制算法（Copying GC），简单来说，就是只把某个空间里的活动对象复制到其他空间，把原空间里的所有对象都回收掉。将复制活动对象的原空间称为 From 空间，将粘贴活动对象的新空间称为 To 空间。

GC复制算法是利用From空间进行分配的。当From空间被完全占满时，GC会将活动对象全部复制到To空间。当复制完成后，该算法会把From空间和To空间互换，GC也就结束了。From空间和To空间大小必须一致。这是为了保证能把From空间中的所有活动对象都收纳到To空间里。
```
copying(){
    $free = $to_start
    for(r : $roots)
	*r = copy(*r)

    swap($from_start, $to_start)
}
```
$free 是指示分块开头的变量。首先在第2行将$free设置在 To 空间的开头，然后复制能从根引用的对象。copy()函数将作为参数传递的对象`*r`复制的同时，也将其子对象进行递归复制。复制结束后返回指针，这里返回的指针指向的是 *r 所在的新空间的对象。

在 GC 复制算法中，在GC结束时，原空间的对象会作为垃圾被回收。因此，由根指向原空间对象的指针也会被重写成指向返回值的新对象的指针。

最后把 From 空间和 To 空间互换，GC就结束了。

GC复制算法的关键当然要数 copy() 函数
```
// 将作为参数给出的对象复制，再递归复制其子对象。
copy(obj){
    if(obj.tag != COPIED)
	copy_data($free, obj, obj.size)
	obj.tag = COPIED
	obj.forwarding = $free
	$free += obj.size

	for(child : children(obj.forwarding))
	    *child = copy(*child)

    return obj.forwarding
}

new_obj(size){
    if($free + size > $from_start + HEAP_SIZE/2)
	copying()
	if($free + size > $from_start + HEAP_SIZE/2)
	    allocation_fail()
    
    obj = $free
    obj.size = size
    $free += size
    return obj
}
```
**优点**
- 优秀的吞吐量
- 可实现告诉分配
- 不会发生碎片化
- 与缓存兼容

**缺点**
- 堆使用效率低下
- 不兼容保守式GC算法
- 递归调用函数，栈溢出风险

### Cheney的GC复制算法
### 近似深度优先搜索方法
### 多空间复制算法
## GC标记压缩算法
GC标记压缩算法（Mark Compact GC）是将GC标记清除算法与GC复制算法相结合的产物。

GC标记压缩算法也是由标记阶段和压缩阶段构成。

首先，这里的标记阶段和我们在讲解GC标记清除算法时提到的标记阶段完全一样。

接下来，我们要搜索数次堆来进行压缩。压缩阶段通过数次搜索堆来重新装填活动对象。
因压缩而产生的优点与GC复制算法一样，不过不需要牺牲半个堆。

**压缩阶段**
- 设定forwarding指针
- 更新指针
- 移动对象
```
compaction_phase(){
    set_forwarding_ptr()
    adjust_ptr()
    move_obj()
}

// 设定foewarding指针
set_forwarding_ptr(){
    scan = new_address = $heap_start // scan 是用来搜索堆中的对象的指针，new_address 是指向目标地点的指针。
    while(scan < $heap_end)
	if(scan.mark == TRUE)
	    scan.forwarding = new_address
	    new_address += scan.size
	scan += scan.size
}

// 更新指针
adjust_ptr(){
    for(r : $roots)
	*r = (*r).forwarding

    scan = $heap_start
    while(sacn < $heap_end)
	if(scan.mark == TRUE)
	    for(child : children(scan)
		*child = (*child).forwarding
	scan += scan.size
}

// 移动对象
move_obj(){
    scan = $free = $heap_start
    while(scan < $heap_end)
	if(scan.mark == TRUE)
	    new_address = scan.forwarding
	    copy_data(new_address, scan, scan.size)
	    new_address.forwarding = NULL
	    new_address.mark = FALSE
	    $free += new_address.size
	    scan += scan.size
}
```

**优点**
- 可有效利用堆

**缺点**
- 必须堆整个堆进行3次搜索，时间长
- 吞吐量劣于其他算法

### Two-Finger算法
### 表格算法
### ImmixGC算法

## 保守式
保守式（Conservative GC）指的是“不能识别指针和非指针的GC”

不明确的根（ambiguous roots）
- 寄存器
- 调用栈
- 全局变量空间

例子：调用栈里的值在GC看来就是一堆位的排列，因此GC不能识别指针和非指针，所以才叫做不明确的根。

在不明确的根这一条件下，GC不能识别指针和非指针。也就是说，不明确的根里所有的值都可能是指针。然而这样一来，在GC时就会大量出现指针和被错误识别成指针的非指针。因此保守式GC会检查不明确的根，以“某种程度”的精度来识别指针。

下面是保守式GC在检查不明确的根时所进行的基本项目。
1. 是不是被正确对齐的值？（在32位CPU的情况下，为4的倍数）
2. 是不是指着堆内？
3. 是不是指着对象的开头？

第1个项目是利用CPU的对齐来检查的。如果CPU是32位的话，指针的值（地址）就是4的倍数；如果CPU是64位的话，指针的值就是8的倍数；如果是其他情况，就会被识别为非指针。在使用这个检查项目时，我们必须在语言处理程序中令要使用的指针符合对齐规则，不符合规则的指针会被GC视为非指针。

第2个项目是调查不明确的根里的值是否指向作为GC对象的堆。当分配里GC专用的堆时，对象就会被分配到堆里。也就是说，指向对象的指针按道理肯定指向堆内。这个检查项目就是利用了这一点。

第3个项目是调查不明确的根内的值是不是指着对象的开头。具体可以用“BIBOP法”等，把对象（在块中）按固定大小对齐，核对检查对象的值是不是对象固定大小的倍数。

以上举出的3个项目是“基本的检查项目”。很惧内存布局和对象结构等（实现），这些检查项目也会有所变化。

当基于不明确的根运行GC时，偶尔会出现非指针和堆里的对象的地址一样的情况，这时GC会遵守原则——“不废弃活动对象”

如果能从头的标志获得结构体的信息（对象的类型信息），GC就能识别对象域里的值是指针还是非指针。以C语言为例，所有的域里面都包含了类型信息，只要程序员没有放入与类型不同含义的值（比如把指针转换类型，放入int类型的域），就是有可能正确识别指针的。

**优点**
- 语言处理程序不依赖于GC

**缺点**
- 识别指针和非指针需要付出成本
- 错误识别指针会压迫堆
- 能够使用的GC算法有限

### 准确式GC
### 间接引用
### MostlyCopyingGC
### 黑名单
## 分代垃圾回收
分代垃圾回收（Generational GC）在对象中倒入了“年龄”的概念，通过优先回收容易成为垃圾的对象，提高垃圾回收的效率。


## 增量式
## RC Immix
