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

分代垃圾回收中的把对象分类成几代，针对不同的代使用不同的GC算法，我们把刚生成的对象称为新生代对象，称为新生代对象，到达一定年龄的对象则称为老年代对象。

总所周知，新生代对象大部分会变成垃圾。如果我们只对这些新生代对象执行GC会怎么样呢？除了引用计数法以外的基本算法，都会进行只寻找活动对象的操作（如GC标记清除算法的标记阶段和GC复制算法等）。因此，如果很多对象都会死去，花费在GC上的时间就能减少。

我们将堆新对象执行的GC称为新生代GC（minor GC）。minor在这里的意思是“小规模的”。新生代GC的前提是大部分新生代对象都没存活下来，GC在短时间内就结束了。

另一方面，新生代GC将存活了一定次数的新生代对象当作老年代对象来处理。我们把类似于这样的新生代对象上升为老年代对象的情况称为晋升（promotion）。

因为老年代对象很难成为垃圾，所以我们对老年代对象减少执行GC的频率。相对于新生代GC，我们将面向老年代对象的GC称为老年代GC（major GC）。

在这里有一点需要注意，那就是分代垃圾回收不能单独用来执行GC。我们需要把它和之前介绍的基本算法结合在一起使用，来提高那些基本算法的效率。

也就是说，分代垃圾回收不是根GC标记清除算法和GC复制算法并列在一起供我们选择的算法，而是需要跟这些基本算法一起使用。

### GC复制和分代垃圾回收
在 Ungar 的分代垃圾回收中，总共需要利用4个空间，分别是生成空间、2个大小相等的幸存空间以及老年代空间，并分别用 `$new_start`、`$survivor1_start`、`$survivor2_start`、`$old_start` 这4个变量引用它们的开头。我们将生成空间和幸存空间合称为新生代空间。新生代对象会被分配到新生代空间，老年代对象则会被分配到老年代空间。Ungar 在论文里把生成空间、幸存空间以及老年代空间的大小分别设成了140K字节、28K字节和940K字节。

此外我们准备出一个和堆不同堆数组，称为记录集（remembered set），设为$rs。

生成空间就如它堆字面意思一样，是生成对象堆空间，也就是进行分配的空间。当生成空间满了的时候，新生代GC就会启动，将生成空间中的所有活动对象复制，这跟GC复制算法是一个道理。目标空间是幸存空间。

2个幸存空间和GC复制算法里的From空间、To空间很像，我们经常只利用其中的一个。在每次执行新生代GC的时候，活动对象就会被复制到另一个幸存空间里。在此我们将正在使用的幸存空间作为From幸存空间，将没有使用的幸存空间作为To幸存空间。

不过新生代GC也必须复制生成空间里的对象。也就是说，生成空间和From幸存空间这两个空间里的活动对象都会被复制到To幸存空间里去。这就是新生代GC。

只有从一定次数的新生代GC中存活下来的对象才会得到晋升，也即是会被复制到老年代空间去。

在执行新生代GC时有一点需要注意，那就是我们必须考虑到从老年代空间到新生代空间的引用。新生代对象不会只被根和新生代空间引用，也可能被老年代对象引用。因此，除了一般GC里的根，我们还需要将从老年代空间的引用当作根（像根一样的东西）来处理。

分代垃圾回收的优点是只将垃圾回收的重点放在新生代对象身上，以此来缩减GC所需要的时间。不过考虑到从老年代对象的引用，结果还是要搜索堆中的所有对象，这样一来就大大削减了分代垃圾回收的优势。

因此我们才需要记录集数组。记录集用来记录从老年代对象到新生代对象的引用。这样在新生代GC时就可以不搜索老年代空间的所有对象，只通过搜索记录集来发现从老年代对象到新生代对象的引用。

那么，通过新生代GC得到晋升的对象把老年代空间占满后，就要执行老年代GC了。老年代GC没什么难的地方，它只用了GC标记清除算法。

**记录集**

记录集被用于高效地寻找从老年代对象到新生代对象的引用。具体来说，在新生代GC时将记录集看成根（像根一样的东西），并进行搜索，以发现指向新生代空间的指针。

不过如果我们为此记录了引用的目标对象（即新生代对象），那么在对这个对象进行晋升（老年化）操作时，就没法改写所引用对象（即老年代对象）的指针了。

因此，在记录集里不会记录引用的目标对象，而是记录发出引用的对象。这样一来，我们就能通过记录集搜索发出引用的对象，进而晋升引用的目标对象，再将发出引用的对象的指针更新到目标空间了。

记录集基本上是用固定大小的数组来实现的。各个元素是指向对象的指针。

**写入屏障**

在分代垃圾回收中，为了将老年代对象记录到记录集里，我们利用写入屏障（write barrier）。在更新对象间的指针的操作中，写入屏障是不可或缺的。write_barrier() 函数的伪代码如下，这跟引用计数算法中的 update_ptr() 函数是在完全相同的情况下被调用的。

```
write_barrier(obj, field, new_obj){
    if(obj >= $old_start && new_obj < $old_start && obj.remembered == FALSE)
	$rd[$rs_index] = obj
	$rs_index++
	obj.remembered = TRUE

    *field = new_obj
}
```
参数 obj 是发出引用的对象，obj内存在要更新的指针，而field指的就是obj内的域，new_obj 是在指针更新后称为引用目标的对象。

在 `obj >= $old_start && new_obj < $old_start && obj.remembered == FALSE` 语句中检查以下3点：
- 发出引用的对象是不是老年代对象
- 指针更新后的引用的目标对象是不是新生代对象
- 发出引用的对象是否还没有被记录到记录集中

当这些检查结果都为真时，obj就记录到记录集中了。

`$rs_index` 是用于新记录对象的索引。

**对象的结构**

在 Ungar 的分代垃圾回收中，对象的头部除了包含对象的种类和大小之外，还有以下这3条信息。
- 对象的年龄（age）
- 已经复制完毕的标志（forwarded）
- 已经向记录集记录完毕的标志（remembered）

age 表示的是对象从新生代GC中存活下来的次数，这个指如果超过一定次数（AGE_MAX），对象就会被当成老年代对象处理。我们在GC复制算法和GC标记压缩算法中也用到过forwarded，这里它的作用是一样的，都是用来防止重复复制相同对象的标志。这里的remembered 也一样，是用来防止向记录集中重复记录的标志。不过remembered指用于老年代对象，age 和 forwarded 只用于新生代对象。

此外，根GC复制算法一样，在这里我们也使用forwarding 指针。

**分配**

分配是在生成空间进行的。执行分配的 new_obj() 函数如下
```
new_obj(size){
    if($new_free + size >= $survivor1_start)
	minor_gc()
	if($new_free + size >= $survivor1_start)
	    allocation_fail()
    
    obj = $new_free
    $new_free += size
    obj.age = 0
    obj.forwarded = FALSE
    obj.remembered = FALSE
    obj.size = size
    return obj
}
```
这里的分配和GC复制算法中的分配基本一样。不过这里的 $new_free 是指向生成空间的分块开头的指针。

首先检查在生成空间中是否存在 size 大小的分块。如果没有足够大小的分块，就执行新生代GC。因为在执行新生代GC后，就可以利用全部的生成空间，所以只要对象的大小不大于生成空间的大小，就肯定能被分配到生成空间。

另一方面，那些大于生成空间的对象在此则会分配失败。不过即使不能被分配到生成空间，它们也有希望被分配到更大的老年代空间。因此，将这些对象分配到比生成空间更大的老年代空间也不失为一个可行的方法。

**新生代GC**

生成空间被对象占满后，新生代GC就会启动，执行这项操作的是 minor_gc() 函数。minor_gc() 函数负责把新生代空间中的活动对象复制到 To 幸存空间和老年代空间。我们先来讲解一下在 minor_gc() 函数中执行复制操作的 copy() 函数。

```
copy(obj){
    if(obj.forwarded == FALSE)
	if(obj.age < AGE_MAX)
	    copy_data($to_survivor_free, obj, obj.size)
	    obj.forwarded = TRUE
	    obj.forwarding = $to_survivor_free
	    $to_survivor_free.age++
	    $to_survivor_free += obj.size
	    for(child : children(obj))
		*child = copy(*child)
	else
	    promote(obj)
    
    return obj.forwarding
}

promote(obj){
    new_obj = allocate_in_old(obj)
    if(new_obj == NULL)
	major_gc()
	new_obj = allocate_in_old(obj)
	if(new_obj == NULL)
	    allocation_fail()
    
    obj.forwarding = new_obj
    obj.forwarded = TRUE

    for(child : children(new_obj))
	if(*child < $old_start)
	    $rs[$rs_index] = new_obj
	    $rs_index+=
	    new_obj.remembered = TRUE
	    return
}
```
晋升可以看成是一项把对象分配到老年代空间的操作，不过在这里被分配的对象是“新生代空间中年龄达到了 AGE_MAX 的对象”。在 promote() 函数中，参数obj是需要晋升的对象。

```
minor_gc(){
    $to_survivor_free = $to_survivor_start
    for(r : $roots)
	if($r < $old_start)
	    *r = copy(*r)
    
    i = 0
    while(i < $rs_index)
	has_new_obj = FALSE
	for(child : children($rs[i]))
	    if(*child < $old_start)
		*child = copy(*child)
		if(*child < $old_start)
		    has_new_obj = TRUE
	if(has_new_obj == FALSE)
	    $rs[i].remembered = FALSE
	    $rs_index--
	    swap($rs[i], $rs[$rs_index])
	else
	    i++
    
    swap($from_survivor_start, $to_survivor_start)
}
```

**老年代GC**

Ungar的论文里在老年代GC中用到了GC标记清除算法。

**优点**
- 吞吐量得到改善

**缺点**
- 在部分程序中会起反作用

### 记录隔代之间的引用方法
### 多代垃圾回收
### 列车垃圾回收
## 增量式垃圾回收
增量式垃圾回收（Incremental GC）是一种通过逐渐推进垃圾回收来控制最大暂停时间的方法。

通常的GC处理很繁重，一旦GC开始执行，不过多久内存分配就没法执行来，这是常有的事。因此人们想出了增量式垃圾回收这种方法。增量（incremental）这个词有“慢慢发生变化”的意思。就如它的名字一样，增量式垃圾回收是将GC和内存分配一点点交替运行的手法。

**三色标记算法**

描述增量式垃圾回收的算法时我们有个方便的概念，那就是三色标记算法（Tri-color marking）。顾名思义，这个算法就是将GC中的对象按照各自的情况分成三种，这三种颜色和所包含的意思分别如下所示。
- 白色：还未搜索过的对象
- 灰色：正在搜索的对象
- 黑色：搜索完成的对象

我们以GC标记清除算法为例向大家再详细地说明一下。

GC开始运行前所有的对象都是白色。GC一开始运行，所有从根能到达的对象都会被标记，然后被堆到栈里。GC只是发生了这样的对象，但还没有搜索完它们，所以这些对象就成了灰色对象。

灰色对象会被一次从栈中取出，其子对象也会被涂成灰色。当其所有但子对象都被涂成灰色时，对象就会被涂成黑色。

当GC结束时已经不存在灰色对象来，活动对象全部为黑色，垃圾则为白色。

这就是三色标记算法但概念。有一点需要我们注意，那就是为了表现黑色对象和灰色对象，不一定要在对象头里设置标志（事实上也有通过标志来表现黑色对象和灰色对象的情况）。在这里我们根据对象的情况，更抽象地把对象用三个颜色表现出来。每个对象是什么样的状况，意味着什么颜色，这些都根据算法的不同而不同。

此外，虽然本书中没有为大家详细说明，不过三色标记算法这个概念不仅能应用于GC标记清除算法，还能应用于其他所有搜索型GC算法。

**GC标记清除算法的分割**

那么，如果GC标记清除算法增量式运行会如何呢？

增量式的GC标记清除算法可分为以下三个阶段。
- 根查找阶段
- 标记阶段
- 清除阶段

我们在根查找阶段把能直接从根引用的对象涂成灰色。在标记阶段查找灰色对象，将其子对象也涂成灰色，查找结束后将灰色对象涂成黑色。在清除阶段则查找堆，将白色对象连接到空闲链表，将黑色对象变回白色。

```
incremental_gc(){
    case $gc_phase
    when GC_ROOT_SCAN
	root_scan_phase() // 根查找阶段只在GC开始时运行一次
    when GC_MARK
	incremental_mark_phase() // 增量式标记
    else
	incremental_sweep_phase() // 增量式清除
}

root_scan_phase(){
    for(r : $roots)
	mark(*r)
    $gc_phase = GC_MARK
}

mark(obj){
    if(obj.mark == FALSE)
	obj.mark = TRUE
	push(obj. $mark_stack) // 把obj由白色涂成灰色
}

incremental_mark_phase(){
    for(i : 1..MARK_MAX)
	if(is_empty($mark_stack) == FALSE)
	    obj = pop($mark_stack)
	    for(child : children(obj))
		mark(*child)
	else
	    for(r : $roots)
		mark(*r)
	    while(is_empty($mark_stack) == FALSE)
		obj = pop($mark_stack)
		for(child : children(obj))
		    mark(*child)

	    $gc_phase = GC_SWEEP
	    $sweeping = $heap_start
	    return
}
```

**写入屏障**

为了防止黑色对象指向白色对象的引用的“标记遗漏”，在增量式垃圾回收中引入写入屏障。
```
write_barrier(obj, field, newobj){
    // 如果newobj是白色对象，就把它涂成灰色
    if(newobj.mark == FALSE)
	newobj.mark = TRUE
	push(newobj, $mark_stack)

    *field = newobj
}
```

**清除阶段**

当标记栈为空时，GC就会进入清除阶段。清除阶段比标记阶段要简单
```
incremental_sweep_phase(){
    swept_count = 0
    whle(swept_count < SWEEP_MAX)
	if($sweeping < $heap_end)
	    if($sweeping.mark == TRUE)
		$sweeping.mark = FALSE
	    else
		$sweeping.next = $free_list
		$free_list = $sweeping
		$free_size += $sweeping.size

	    $sweeping += $sweeping.size
	    swept_count++
	else
	    $gc_phase = GC_ROOT_SCAN
	    return
}
```

**分配**
这里的分配也和停止型GC标记清除算法中的分配没什么两样（增量式是非暂停型）。
```
newobj(size){
    if($free_size < HEAP_SIZE * GC_THRESHOLD)
	incremental_gc()

    chunk = pickup_chunk(size, $free_list)
    if(chunk != NULL)
	chunk.size = size
	$free_size -= size
	if($gc_phase == GC_SWEEP && $sweeping <= chunk)
	    chunk.mark = TRUE
	return chunk
    else
	allocation_fail()
```

**优点**
- 缩短最大暂停时间

**缺点**
- 降低了吞吐量

### Steele 的算法
### 汤浅的算法
### 比较各个写入屏障
## RC Immix
Rc Immix 算法将引用计数的一大缺点————吞吐量低改善到了使用级别。本算法将改善了引用计数法的“合并型引用计数法”（Coalesced Reference Counting）和 Immix 组合了起来。
