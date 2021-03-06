# 设备树展开

**参考博客：**

https://blog.csdn.net/shenlong1356/article/details/105937866?utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-1.no_search_link&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-1.no_search_link

https://blog.csdn.net/qq_35065875/article/details/82852902
https://blog.csdn.net/qq_35065875/article/details/82868976?utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-1.no_search_link&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-1.no_search_link

https://www.cnblogs.com/sky-heaven/p/13855719.html

https://blog.csdn.net/qq_35552527/article/details/105499408

https://www.cnblogs.com/schips/p/linux_driver_dtb_to_device_node.html
https://www.cnblogs.com/schips/p/linux_driver_device_node_to_platform_device.html

https://blog.csdn.net/zqixiao_09/article/details/50889458



关于设备树的基本语法可以参考：

https://zhuanlan.zhihu.com/p/135280350

https://www.eet-china.com/mp/a86114.html

或 《Linux设备驱动开发详解》-- 宋宝华



## 1. 内核入口

linux最底层的初始化部分在HEAD.s中，这是汇编代码，我们暂且不作过多讨论。
在head.s完成部分初始化之后，就开始调用C语言函数，而被调用的第一个C语言函数就是**start_kernel**
而对于设备树的处理，基本上就在setup_arch()这个函数中。可以看到，在start_kernel()中调用了**setup_arch**(&command_line) 注意这里的 setup_arch 跟架构有关，每个架构有自己的 setup_arch

```c
void __init setup_arch(char **cmdline_p)
{
    const struct machine_desc *mdesc;
    
    // 根据传入的设备树dtb的首地址完成一些初始化操作
    mdesc = setup_machine_fdt(__atags_pointer);
    
    // ...
    
    // 保证设备树dtb本身存在于内存中而不被覆盖
    arm_memblock_init(mdesc);
    
    // ...
    // 对设备树具体的解析
    unflatten_device_tree();
    // ...
}

```

这三个被调用的函数就是主要的设备树处理函数：

1. setup_machine_fdt()：根据传入的设备树dtb的首地址完成一些初始化操作。**(cpu架构相关)**
2. arm_memblock_init()：主要是内存相关函数，为设备树保留相应的内存空间，保证设备树dtb本身存在于内存中而不被覆盖。用户可以在设备树中设置保留内存，这一部分同时作了保留指定内存的工作。**(cpu架构相关)**
3. unflatten_device_tree()：对设备树具体的解析，事实上在这个函数中所做的工作就是**将设备树各节点转换成相应的struct device_node结构体**。**(cpu架构无关)**



## 2. 设备树dtb转换成device_node

在dts文件里，每个大括号{ }代表一个节点，比如根节点里有个大括号，对应一个device_node结构体；
memory也有一个大括号，也对应一个device_node结构体。
节点里面有各种属性，也可能里面还有子节点，所以它们还有一些父子关系。
根节点下的memory、chosen、led等节点是并列关系，兄弟关系。对于父子关系、兄弟关系，在device_node结构体里面肯定有成员来描述这些关系。

```c
struct device_node {
    const char *name;         // 来自节点中的name属性, 如果没有该属性, 则设为"NULL"
    const char *type;         // 来自节点中的device_type属性, 如果没有该属性, 则设为"NULL"
    phandle phandle;
    const char *full_name;    // 节点的名字, node-name[@unit-address]
    struct fwnode_handle fwnode;

    struct  property *properties;   // 节点的属性
    struct  property *deadprops;    /* removed properties */
    struct  device_node *parent;    // 节点的父亲
    struct  device_node *child;     // 节点的孩子(子节点)
    struct  device_node *sibling;   // 节点的兄弟(同级节点)
#if defined(CONFIG_OF_KOBJ)
    struct  kobject kobj;
#endif
    unsigned long _flags;
    void    *data;
#if defined(CONFIG_SPARC)
    const char *path_component_name;
    unsigned int unique_id;
    struct of_irq_controller *irq_trans;
#endif
};
```

device_node结构体表示一个节点，property结构体表示节点的具体属性。properties结构体的定义如下：

```c
struct property {
    char    *name;        // 属性名字, 指向dtb文件中的字符串 例如reg  interrupts
    int     length;       // 属性值的长度
    void    *value;       // 属性值, 指向dtb文件中value所在位置, 数据仍以big endian存储
    struct property *next;
#if defined(CONFIG_OF_DYNAMIC) || defined(CONFIG_SPARC)
    unsigned long _flags;
#endif
#if defined(CONFIG_OF_PROMTREE)
    unsigned int unique_id;
#endif
#if defined(CONFIG_OF_KOBJ)
    struct bin_attribute attr;
#endif
};
```



### 2.1 setup_machine_fdt

```c
    const struct machine_desc *mdesc;
    
    // 根据传入的设备树dtb的首地址完成一些初始化操作
    mdesc = setup_machine_fdt(__atags_pointer);
```

__atags_pointer这个全局变量存储的就是r2的寄存器值，是设备树在内存中的起始地址,将设备树起始地址传递给setup_machine_fdt，对设备树进行解析。

```c
const struct machine_desc * __init setup_machine_fdt(unsigned int dt_phys)
{
    const struct machine_desc *mdesc, *mdesc_best = NULL;
    // 内存地址检查
    if (!dt_phys || !early_init_dt_verify(phys_to_virt(dt_phys)))
        return NULL;

    // 读取 compatible 属性
    mdesc = of_flat_dt_match_machine(mdesc_best, arch_get_next_mach);

    // 扫描各个子节点
    early_init_dt_scan_nodes();
    // ...
}
```

setup_machine_fdt 主要是获取了一些设备树提供的总览信息。

#### 2.1.1 内存地址检查

先将设备树在内存中的物理地址转换为虚拟地址
然后再检查该地址上是否有设备树的魔数(magic)，魔数就是一串用于识别的字节码：
		如果没有或者魔数不匹配，表明该地址没有设备树文件，函数返回失败
		否则验证成功，将设备树地址赋值给全局变量initial_boot_params。

#### 2.1.2 读取compatible属性

逐一读取设备树**根目录下的** compatible 属性，与内核中的描述比对。整个过程可以看作内核用一个ID，去内存中查找该ID对应的设备树，因为可能有不止一棵设备树。

```c
/**
 * of_flat_dt_match_machine - Iterate match tables to find matching machine.
 *
 * @default_match: A machine specific ptr to return in case of no match.
 * @get_next_compat: callback function to return next compatible match table.
 *
 * Iterate through machine match tables to find the best match for the machine
 * compatible string in the FDT.
 */
const void * __init of_flat_dt_match_machine(const void *default_match, const void * (*get_next_compat)(const char * const**))
{
    const void *data = NULL;
    const void *best_data = default_match;
    const char *const *compat;
    unsigned long dt_root;
    unsigned int best_score = ~1, score = 0;

    // 获取首地址
    dt_root = of_get_flat_dt_root();
    // 遍历
    while ((data = get_next_compat(&compat))) {
        // 将compatible中的属性一一与内核中支持的硬件单板相对比，
        // 匹配成功后返回相应的 machine_desc 结构体指针。
        score = of_flat_dt_match(dt_root, compat);
        if (score > 0 && score < best_score) {
            best_data = data;
            best_score = score;
        }
    }

    // ...

    pr_info("Machine model: %s\n", of_flat_dt_get_machine_name());

    return best_data;
}
```

machine_desc 结构体中描述了单板相关的一些硬件信息，这里不过多描述。
主要的的行为就是根据这个 compatible 属性选取相应的硬件单板描述信息；一般 compatible 属性名就是"厂商，芯片型号"。

#### 2.1.3 扫描各子节点

第三部分就是扫描设备树中的各节点，主要分析这部分代码。

```c
void __init early_init_dt_scan_nodes(void)      // 该接口与架构无关
{
    of_scan_flat_dt(early_init_dt_scan_chosen, boot_command_line);
    of_scan_flat_dt(early_init_dt_scan_root, NULL);
    of_scan_flat_dt(early_init_dt_scan_memory, NULL);
}
```

出人意料的是，这个函数中只有一个函数的三个调用，每次调用时，参数不一样。



##### of_scan_flat_dt

首先of_scan_flat_dt()这个函数接收两个参数，一个是函数指针it，一个为相当于函数it执行时的参数。

```c
/**
 * of_scan_flat_dt - scan flattened tree blob and call callback on each.
 * @it: callback function
 * @data: context data pointer
 *
 * This function is used to scan the flattened device-tree, it is
 * used to extract the memory information at boot before we can
 * unflatten the tree
 */
int __init of_scan_flat_dt(int (*it)(unsigned long node, const char *uname, int depth, void *data), void *data)
{
    unsigned long p = ((unsigned long)initial_boot_params) + be32_to_cpu(initial_boot_params->off_dt_struct);
    int rc = 0;
    int depth = -1;

    do {
        u32 tag = be32_to_cpup((__be32 *)p);
        const char *pathp;

        p += 4;
        if (tag == OF_DT_END_NODE) {
            depth--;
            continue;
        }
        if (tag == OF_DT_NOP)
            continue;
        if (tag == OF_DT_END)
            break;
        if (tag == OF_DT_PROP) {
            u32 sz = be32_to_cpup((__be32 *)p);
            p += 8;
            if (be32_to_cpu(initial_boot_params->version) < 0x10)
                p = ALIGN(p, sz >= 8 ? 8 : 4);
            p += sz;
            p = ALIGN(p, 4);
            continue;
        }
        if (tag != OF_DT_BEGIN_NODE) {
            pr_err("Invalid tag %x in flat device tree!\n", tag);
            return -EINVAL;
        }
        depth++;
        pathp = (char *)p;
        p = ALIGN(p + strlen(pathp) + 1, 4);
        if (*pathp == '/')
            pathp = kbasename(pathp);
        rc = it(p, pathp, depth, data);
        if (rc != 0)
            break;
    } while (1);

    return rc;
}
```

结论：of_scan_flat_dt() 函数的作用就是扫描设备树中的节点，然后对各节点分别调用传入的回调函数。
那么重点关注函数指针，在上述代码中，传入的参数分别为
		early_init_dt_scan_chosen
		early_init_dt_scan_root
		early_init_dt_scan_memory
从名称可以猜测，这三个函数分别是处理chosen节点、root节点中除子节点外的属性信息、memory节点。



##### early_init_dt_scan_chosen

```c
of_scan_flat_dt(early_init_dt_scan_chosen, boot_command_line);
```

boot_command_line 是一个静态数组，存放着启动参数，

```c
int __init early_init_dt_scan_chosen(unsigned long node, const char *uname, int depth, void *data)
{
    unsigned long l;
    char *p;

    pr_debug("search \"chosen\", depth: %d, uname: %s\n", depth, uname);

    if (depth != 1 || !data ||
        (strcmp(uname, "chosen") != 0 && strcmp(uname, "chosen@0") != 0))
        return 0;

    early_init_dt_check_for_initrd(node);

    /* Retrieve command line */
    // 找到设备树中的的chosen节点中的bootargs，并作为cmd_line
    p = of_get_flat_dt_prop(node, "bootargs", &l);
    if (p != NULL && l > 0)
        strlcpy(data, p, min((int)l, COMMAND_LINE_SIZE));

   // ...

    pr_debug("Command line is: %s\n", (char*)data);

    /* break now */
    return 1;
}
```

经过代码分析，early_init_dt_scan_chosen的作用是获取从chosen节点中获取bootargs，然后将bootargs放入boot_command_line中，作为启动参数。而非字面意思的处理整个chosen。

在支持设备树的嵌入式系统中，实际上：
		uboot基本上可以不通过显式的bootargs=xxx来传递给内核，而是在env拿出，并存放进设备树中的chosen节点中
		Linux也开始在设备树中的chosen节点中获取出来
这样子就可以做到针对uboot与Linux在bootargs传递上的统一。



##### early_init_dt_scan_root

```c
int __init early_init_dt_scan_root(unsigned long node, const char *uname,int depth, void *data)
{
    dt_root_size_cells = OF_ROOT_NODE_SIZE_CELLS_DEFAULT;
    dt_root_addr_cells = OF_ROOT_NODE_ADDR_CELLS_DEFAULT;

    prop = of_get_flat_dt_prop(node, "#size-cells", NULL);
    if (prop)
        dt_root_size_cells = be32_to_cpup(prop);
    prop = of_get_flat_dt_prop(node, "#address-cells", NULL);
    if (prop)
        dt_root_addr_cells = be32_to_cpup(prop);
    // ...
}
```

通过代码分析，early_init_dt_scan_root 将 root 节点中的#size-cells和#address-cells属性提取出来，放到全局变量dt_root_size_cells和dt_root_addr_cells中，并非获取root节点中所有的属性。

父节点的 #address-cells 和 #size-cells 分别决定了子节点 reg 属性的 address 和 length 字段的长度，例如：#address-cells = <2>  #size-cells = <1> 则对应的reg形式为 

```c
reg = < address1_1 address1_2 length1 [address2_1 address2_2 length2 ...] >
```

不同的字段长度有不同的含义：
#address-cells = <1>;就是说你的地址是32位，#address-cells = <2>;就是说你的地址是64位，现在的设备树中表示地址范围的#address-cells还没有超过2的 #address-cells只能是1和2
**例如：**
对于32位的地址#address-cells = <1>：
reg =  <0x02000000 0x10000>这种情况我们很常见了就是 0x02000000~0x02000000+0x10000地址空间
对于64位的地址#address-cells = <2>：
reg =  <0x10000000 0x02000000 0x10000>这种情况我们64位的地址就是高32位是0x10000000，低32位是0x02000000的64位地址所以他代表的地址空间就是：0x1000000002000000 + 0x10000的地址空间

reg 的组织形式为:

```c
// address 和 length 对表示一个地址范围， address 是起点，终点是 address+length-1，如果存在两个 address 表示是64位的地址
// address 和 length 都是四字节长度
reg = <address1 length1 [address2 length2] [address3 length3]>
```

```c
// 表示数据大小为一个4字节描述，32位
#size-cells = 1

// 表示数据起始地址由一个四字节描述
#address-cells = 1

// 而reg属性由四个四字节组成，所以存在两组地址描述，
// 第一组是起始地址为0x12345678，长度为0x100，
// 第二组起始地址为0x22，长度为0x4, 
// 因为在<>中，所有数据都是默认为32位。
reg = <0x12345678 0x100 0x22 0x4>  
```



##### early_init_dt_scan_memory

```c
int __init early_init_dt_scan_memory(unsigned long node, const char *uname,int depth, void *data){
    // ...
    if (!IS_ENABLED(CONFIG_PPC32) || depth != 1 || strcmp(uname, "memory") != 0)
      return 0;
    reg = of_get_flat_dt_prop(node, "reg", &l);
    endp = reg + (l / sizeof(__be32));

    while ((endp - reg) >= (dt_root_addr_cells + dt_root_size_cells)) {
        base = dt_mem_next_cell(dt_root_addr_cells, &reg);
	    size = dt_mem_next_cell(dt_root_size_cells, &reg);
        early_init_dt_add_memory_arch(base, size);
    }
}
```

函数先判断节点的unit name是memory,如果不是，则返回。如果是则将所有memory相关的reg属性取出来，根据address-cell和size-cell的值进行解析，然后调用early_init_dt_add_memory_arch()来申请相应的内存空间。

```c
    memory@0 {
        device_type = "memory";
        reg = <0x0 0x0 0x0 0x80000000>, <0x8 0x00000000 0x0 0x80000000>;
    };
```

到这里，setup_machine_fdt()函数对于设备树的第一次扫描解析就完成了，主要是获取了一些设备树提供的总览信息。



### 2.2 arm_memblock_init

```c
// arch/arm/mm/init.c
void __init arm_memblock_init(const struct machine_desc *mdesc)
{
    // ...
    early_init_fdt_reserve_self();
    early_init_fdt_scan_reserved_mem();
    // ...
}
```

对于设备树的初始化而言，主要做了两件事：

1. 调用early_init_fdt_reserve_self，根据设备树的大小为设备树分配空间，设备树的totalsize在dtb头部中有指明，因此当系统启动之后，设备树就一直存在在系统中。
2. 扫描设备树节点中的"reserved-memory"节点，为其分配保留空间。

arm_memblock_init 对于设备树的部分解析就完成了，主要是为设备树指定保留内存空间。



### 2.3 unflatten_device_tree

这一部分就进入了设备树的解析部分：
注意of_root这个对象，我们后续文章中会提到它。实际上，解析以后的数据都是放在了这个对象里面。

```c
void __init unflatten_device_tree(void)
{
    // 展开设备树
    __unflatten_device_tree(initial_boot_params, NULL, &of_root,early_init_dt_alloc_memory_arch, false);

    // 扫描设备树
    of_alias_scan(early_init_dt_alloc_memory_arch);
    // ...
}
```

#### 2.3.1 展开设备树

**property原型**

```c
struct property {
    char	*name;
    int	length;
    void	*value;
    struct property *next;
    // ...
};
```

在设备树中，对于属性的描述是 key = value，这个结构体中的name和value分别对应key和value，而length表示value的长度；
next指针指向下一个struct property结构体（用于构成单链表）。

**__unflatten_device_tree**

```c
__unflatten_device_tree(initial_boot_params, NULL, &of_root,early_init_dt_alloc_memory_arch, false);
```

我们再来看最主要的设备树解析函数：

```c
void *__unflatten_device_tree(const void *blob,struct device_node *dad,
                              struct device_node **mynodes,void *(*dt_alloc)(u64 size, u64 align),bool detached)
{
    int size;
    // ...
    size = unflatten_dt_nodes(blob, NULL, dad, NULL);
    // ...
    mem = dt_alloc(size + 4, __alignof__(struct device_node));
    // ...
    unflatten_dt_nodes(blob, mem, dad, mynodes);
}
```

主要的解析函数为unflatten_dt_nodes()，在__unflatten_device_tree()函数中，unflatten_dt_nodes()被调用两次：
		第一次是扫描得出设备树转换成device node需要的空间，然后系统申请内存空间；
		第二次就进行真正的解析工作，我们继续看unflatten_dt_nodes()函数：
值得注意的是，在第二次调用unflatten_dt_nodes()时传入的参数为unflatten_dt_nodes(blob, mem, dad, mynodes);



**unflatten_dt_nodes**

第一个参数是设备树存放首地址，第二个参数是申请的内存空间，第三个参数为父节点，初始值为NULL，第四个参数为mynodes，初始值为of_node.

```c
static int unflatten_dt_nodes(const void *blob,void *mem,struct device_node *dad,struct device_node **nodepp)
{
    // ...
    for (offset = 0;offset >= 0 && depth >= initial_depth;offset = fdt_next_node(blob, offset, &depth)) {
        populate_node(blob, offset, &mem,nps[depth],fpsizes[depth],&nps[depth+1], dryrun);
        // ...
    }
}
```

这个函数中主要的作用就是从根节点开始，对子节点依次调用 populate_node() ，从函数命名上来看，这个函数就是填充节点，为节点分配内存。



**device_node原型**

```c
// include/linux/of.h
struct device_node {
    const char *name;
    const char *type;
    phandle phandle;
    const char *full_name;
    // ...
    struct	property *properties;
    struct	property *deadprops;	/* removed properties */
    struct	device_node *parent;
    struct	device_node *child;
    struct	device_node *sibling;
    struct	kobject kobj;
    unsigned long _flags;
    void	*data;
    // ...
};
```

**name**：设备节点中的name属性转换而来。
**type**：由设备节点中的device_type转换而来。
**phandle**：由设备节点中的"phandle"和"linux,phandle"属性转换而来，特殊的还可能由"ibm,phandle"属性转换而来。
**full_name**：这个指针指向整个结构体的结尾位置，在结尾位置存储着这个结构体对应设备树节点的unit_name，意味着一个struct device_node结构体占内存空间为sizeof(struct device_node)+strlen(unit_name)+字节对齐。
**properties**：这是一个设备树节点的属性链表，属性可能有很多种，比如："interrupts","timer"，"hwmods"等等。
**parent,child,sibling**：与当前属性链表节点相关节点，所以相关链表节点构成整个device_node的属性节点。
**kobj**：用于在/sys目录下生成相应用户文件。



```c
static unsigned int populate_node(const void *blob,int offset,void **mem,
			  struct device_node *dad,unsigned int fpsize,struct device_node **pnp,bool dryrun){
    struct device_node *np;
    // 申请内存
    // 注，allocl是节点的unit_name长度(类似于chosen、memory这类子节点描述开头时的名字，并非.name成员)
    np = unflatten_dt_alloc(mem, sizeof(struct device_node) + allocl,__alignof__(struct device_node));
    
    // 初始化node（设置kobj，接着设置node的fwnode.ops。）
    of_node_init(np);
    
    // 将device_node的full_name指向结构体结尾处，
    // 即，将一个节点的unit name放置在一个struct device_node的结尾处。
    np->full_name = fn = ((char *)np) + sizeof(*np);
    
    // 设置其 父节点 和 兄弟节点（如果有父节点）
    if (dad != NULL) {
		np->parent = dad;
		np->sibling = dad->child;
		dad->child = np;
	}
    
    // 为节点的各个属性分配空间
    populate_properties(blob, offset, mem, np, pathp, dryrun);
    
    // 获取，并设置device_node节点的name和type属性
    np->name = of_get_property(np, "name", NULL);
	np->type = of_get_property(np, "device_type", NULL);
    if (!np->name)
		np->name = "<NULL>";
	if (!np->type)
		np->type = "<NULL>";
    // ...
}  
```

一个设备树中节点转换成一个struct device_node结构的过程渐渐就清晰起来，现在我们接着看看populate_properties()这个函数，看看属性是怎么解析的，

```c
static void populate_properties(const void *blob,int offset,void **mem,struct device_node *np,const char *nodename,bool dryrun){
    // ...
    for (cur = fdt_first_property_offset(blob, offset);
         cur >= 0;
         cur = fdt_next_property_offset(blob, cur)) 
    {
        fdt_getprop_by_offset(blob, cur, &pname, &sz);
        unflatten_dt_alloc(mem, sizeof(struct property),__alignof__(struct property));
        if (!strcmp(pname, "phandle") ||  !strcmp(pname, "linux,phandle")) {
            if (!np->phandle)
                np->phandle = be32_to_cpup(val);

            pp->name   = (char *)pname;
            pp->length = sz;
            pp->value  = (__be32 *)val;
            *pprev     = pp;
            pprev      = &pp->next;
            // ...
        }
    }
}
```

从属性转换部分的程序可以看出，对于大部分的属性，都是直接填充一个 struct property 属性；
而对于 "phandle" 属性和 "linux,phandle" 属性，直接填充 struct device_node 的 phandle 字段，不放在属性链表中。

#### 2.3.2 扫描节点：of_alias_scan

从名字来看，这个函数的作用是解析根目录下的alias

```c
struct device_node *of_chosen;
struct device_node *of_aliases;

void of_alias_scan(void * (*dt_alloc)(u64 size, u64 align)){
    of_aliases = of_find_node_by_path("/aliases");
    of_chosen = of_find_node_by_path("/chosen");
    if (of_chosen) {
        if (of_property_read_string(of_chosen, "stdout-path", &name))
            of_property_read_string(of_chosen, "linux,stdout-path",
                                    &name);
        if (IS_ENABLED(CONFIG_PPC) && !name)
            of_property_read_string(of_aliases, "stdout", &name);
        if (name)
            of_stdout = of_find_node_opts_by_path(name, &of_stdout_options);
    }
    for_each_property_of_node(of_aliases, pp) {
        // ...
        ap = dt_alloc(sizeof(*ap) + len + 1, __alignof__(*ap));
        if (!ap)
            continue;
        memset(ap, 0, sizeof(*ap) + len + 1);
        ap->alias = start;
        of_alias_add(ap, np, id, start, len);
        // ...
    }
}
```

of_alias_scan() 函数先是处理设备树chosen节点中的"stdout-path"或者"stdout"属性(两者最多存在其一)，然后将stdout指定的path赋值给全局变量of_stdout_options，并将返回的全局 struct device_node 类型数据赋值给of_stdout，指定系统启动时的log输出。

接下来为aliases节点申请内存空间，如果一个节点中同时没有name/phandle/linux,phandle，则被定义为特殊节点，对于这些特殊节点将不会申请内存空间。

然后，使用of_alias_add()函数将所有的aliases内容放置在aliases_lookup链表中。



### 2.4 转换过程总结

![img](device_tree.assets/2120938-20210626073039172-236790166-20210928150139641.png)

此后，内核就可以根据 device_node 来创建设备。



## 3. device_node转platfrom_device

由 **设备树dtb转换成device_node** 部分的内容可知：每个设备树子节点都将转换成一个对应的device_node节点。

### 3.1 设备树对于驱动

设备树的产生就是为了替代driver中过多的platform_device部分的静态定义，将硬件资源抽象出来，由系统统一解析，这样就可以避免各驱动中对硬件资源大量的重复定义。这样一来，几乎可以肯定的是，设备树中的节点最终目标是转换成platform device结构，在驱动开发时就只需要添加相应的platform driver部分进行匹配即可。



### 3.2 device_node转换为platform_device的条件

首先，对于所有的device_node，如果要转换成platform_device，除了节点中必须有compatible属性以外，必须满足以下条件：

1. 一般情况下，只对设备树中根的第1级节点（/xx）注册成platform device，也就是对它们的子节点（/xx/*）并不处理。
2. 如果一个结点的compatile属性含有这些特殊的值("simple-bus","simple-mfd","isa","arm,amba-bus")之一，并且自己成功注册成了platform_device，, 那么它的子结点(需含compatile属性)也可以转换为platform_device（当成总线看待）。
3. 根节点（/）是例外的，生成platfrom_device时，即使有compatible属性也不会处理

注意：i2c, spi等总线节点会转换为platform_device，但是，spi、i2c下的子节点无论compatilbe是否为: “simple-bus”,“simple-mfd”,“isa”,"arm,amba-bus "都应该交给对应的总线驱动程序来处理而不会被转换为platform_device
举例：

```c
/ {
          mytest {  //根节点下的子节点并且包含compatile，因此可以转换为platform_device
              compatile = "mytest", "simple-bus";
              mytest@0 {  //该节点不属于根节点的子节点，但父节点包含 "simple-bus"  因此会转换为platform_device
                    compatile = "mytest_0";  
              };
          };

          i2c { //根节点下的子节点并且包含compatile，因此可以转换为platform_device
              compatile = "samsung,i2c";
              at24c02 {   //不属于根节点的子节点，并且还是i2c的子节点，一般交给i2c驱动转换为i2c_client
                    compatile = "at24c02";                      
              };
          };
 
          spi {  //根节点下的子节点并且包含compatile，因此可以转换为platform_device
              compatile = "samsung,spi"; 
              flash@0 {   //不属于根节点的子节点，并且还是spi的子节点，一般交给spi驱动转换为spi_device
                    compatible = "winbond,w25q32dw";
                    spi-max-frequency = <25000000>;
                    reg = <0>;
                  };
          };
 };
```









### 3.3 设备树结点会被转换的属性

如果是device_node转换成platform device，这个转换过程又是怎么样的呢？
在老版本的内核中，platform_device部分是静态定义的，其实最主要的部分就是resources部分，这一部分描述了当前驱动需要的硬件资源，一般是IO，中断等资源。在设备树中，这一类资源通常通过reg属性来描述，中断则通过interrupts来描述；所以，**设备树中的reg和interrupts资源将会被转换成platform_device内的struct resources资源**。可以通过platform_get_resource等平台API获取资源，对于IRQ而言，platform_getresource()还有一个进行了封装的变体platform_get_irq()。关于reg在设备树中的表达在 2.1.3 小节的 early_init_dt_scan_root 接口介绍中给出，关于中断的表达形式如下介绍：

```
/{
		compatible = "acme,coyotes-revenge";
		#address-cells = <1>;
		#size-cells = <1>;
		interrupt-parent = <&intc>;
		
		gpio@1-101f3000{
				compatible = "arm,p1061";
				reg = <0x101f3000 0x1000
							 0x101f4000 0x0010>;
				interrupts = < 3 0 >;
		};
		
		intc: interrupt-controller@10140000{
				compatible = "arm,pll90";
				reg = <0x10140000 0x1000>;
				interrupt-controller;
				#interrupt-cells = <2>;
		};

};

对于中断控制器：
通过 interrupt-controller 属性表明当前节点为中断控制器，该属性为空
通过 #interrupt-cells 表明连接此中断控制器的设备的中断属性的cell大小，含义可以参考 reg 中的 #address-cells 和 #size-cells

其他中断控制器相关的属性：
interrupt-parent  设备节点通过该属性指定它所依附的中断控制器的 phandle，当节点没有指定 interrupt-parent 时，则从父节点继承，
									例如上例中的 interrupt-parent = <&intc>;
interrupts  该属性具体含有多少个cell取决于它所依附的中断控制器中的 #interrupt-cells 属性。而每个cell又是什么含义，一般由驱动的实现决定。

```



那么，设备树中其他属性是怎么转换的呢？
答案是：不需要转换，在platform_device中有一个成员struct device dev，这个dev中又有一个指针成员struct device_node *of_node。linux的做法就是将这个of_node指针直接指向由设备树转换而来的device_node结构；留给驱动开发者自行处理。
例如，有这么一个struct platform_device of_test。我们可以直接通过 of_test.dev.of_node 来访问设备树中的信息。
又例如存在一个自定义属性：rockchip,taskqueue-node = <2>;
可以使用接口 of_property_read_u32(dev->of_node, "rockchip,taskqueue-node", &taskqueue_node);读取属性值



### 3.4 platform_device转换的开始

**of_platform_default_populate_init**

函数的执行入口是，在系统启动的早期进行的 of_platform_default_populate_init：

```c
// drivers/of/platform.c
static int __init of_platform_default_populate_init(void)
{
    struct device_node *node;

    if (!of_have_populated_dt())
        return -ENODEV;

    /*
     * Handle ramoops explicitly, since it is inside /reserved-memory,
     * which lacks a "compatible" property.
     */
    node = of_find_node_by_path("/reserved-memory");
    if (node) {
        node = of_find_compatible_node(node, NULL, "ramoops");
        if (node)
            of_platform_device_create(node, NULL, NULL);
    }

    /* Populate everything else. */
    of_platform_default_populate(NULL, NULL, NULL);

    return 0;
}
arch_initcall_sync(of_platform_default_populate_init);
```

在函数 of_platform_default_populate_init() 中，调用了 of_platform_default_populate(NULL, NULL, NULL); ，传入三个空指针：

```c
int of_platform_default_populate(struct device_node *root,const struct of_dev_auxdata *lookup,struct device *parent)
{
    return of_platform_populate(root, of_default_bus_match_table, lookup,
                    parent);
}
```

of_platform_default_populate()调用了of_platform_populate()，我们注意下of_default_bus_match_table

```c
const struct of_device_id of_default_bus_match_table[] = {
    { .compatible = "simple-bus", },
    { .compatible = "simple-mfd", },
    { .compatible = "isa", },
    #ifdef CONFIG_ARM_AMBA
        { .compatible = "arm,amba-bus", },
    #endif /* CONFIG_ARM_AMBA */
        {} /* Empty terminated list */
};
```

如果节点的属性值为 "simple-bus","simple-mfd","isa","arm,amba-bus "之一的话，那么它子节点就可以转化成platform_device。

接下来看 **of_platform_populate**

```c
int of_platform_populate(struct device_node *root,
            const struct of_device_id *matches,
            const struct of_dev_auxdata *lookup,
            struct device *parent)
{
    struct device_node *child;
    int rc = 0;

    // 从设备树中获取根节点的device_node结构体
    root = root ? of_node_get(root) : of_find_node_by_path("/");
    if (!root)
        return -EINVAL;

    pr_debug("%s()\n", __func__);
    pr_debug(" starting at: %pOF\n", root);

    //遍历所有的子节点
    for_each_child_of_node(root, child) {
        // 然后对每个根目录下的一级子节点 创建 bus
        // 例如， /r1 , /r2，而不是 /r1/s1
        rc = of_platform_bus_create(child, matches, lookup, parent, true);
        if (rc) {
            of_node_put(child);
            break;
        }
    }
    of_node_set_flag(root, OF_POPULATED_BUS);

    of_node_put(root);
    return rc;
}
EXPORT_SYMBOL_GPL(of_platform_populate);
```

在调用of_platform_populate()时传入了的matches参数是of_default_bus_match_table[]；
这个table是一个静态数组，这个静态数组中定义了一系列的compatible属性："simple-bus"、"simple-mfd"、"isa"、"arm,amba-bus"。
按照我们上文中的描述，当某个根节点下的一级子节点的compatible属性为这些属性其中之一时，它的一级子节点也将由device_node转换成platform_device.

**of_platform_bus_create**

```c
/**
 * of_platform_bus_create() - Create a device for a node and its children.
 * @bus: device node of the bus to instantiate
 * @matches: match table for bus nodes
 * @lookup: auxdata table for matching id and platform_data with device nodes
 * @parent: parent for new device, or NULL for top level.
 * @strict: require compatible property
 *
 * Creates a platform_device for the provided device_node, and optionally
 * recursively create devices for all the child nodes.
 */
static int of_platform_bus_create(struct device_node *bus,
                  const struct of_device_id *matches,
                  const struct of_dev_auxdata *lookup,
                  struct device *parent, bool strict)
{
    const struct of_dev_auxdata *auxdata;
    struct device_node *child;
    struct platform_device *dev;
    const char *bus_id = NULL;
    void *platform_data = NULL;
    int rc = 0;

    // ...

    // 创建of_platform_device、赋予私有数据
    dev = of_platform_device_create_pdata(bus, bus_id, platform_data, parent);
    
    // 判断当前节点的compatible属性是否包含上文中compatible静态数组中的元素
    // 如果不包含，函数返回0，即，不处理子节点。
    if (!dev || !of_match_node(matches, bus))
        return 0;

    for_each_child_of_node(bus, child) {
        pr_debug("   create child: %pOF\n", child);
        // 创建 of_platform_bus
        /* 
           如果当前compatible属性中包含静态数组中的元素，
           即当前节点的compatible属性有"simple-bus"、"simple-mfd"、"isa"、"arm,amba-bus"其中一项，
           把子节点当作对应的总线来对待，递归地对当前节点调用`of_platform_bus_create()`
           即，将符合条件的子节点转换为platform_device结构。
        */
        rc = of_platform_bus_create(child, matches, lookup, &dev->dev, strict);
        if (rc) {
            of_node_put(child);
            break;
        }
    }
    of_node_set_flag(bus, OF_POPULATED_BUS);
    return rc;
}
```

**of_platform_device_create_pdata**
这个函数实现了：创建of_platform_device、赋予私有数据
此时的参数platform_data为NULL。

```c
/**
 * of_platform_device_create_pdata - Alloc, initialize and register an of_device
 * @np: pointer to node to create device for
 * @bus_id: name to assign device
 * @platform_data: pointer to populate platform_data pointer with
 * @parent: Linux device model parent device.
 *
 * Returns pointer to created platform device, or NULL if a device was not
 * registered.  Unavailable devices will not get registered.
 */
static struct platform_device *of_platform_device_create_pdata(
                    struct device_node *np,
                    const char *bus_id,
                    void *platform_data,
                    struct device *parent)
{
    // 终于看到了平台设备
    struct platform_device *dev;

    // ...

    // 创建实例，将传入的device_node链接到成员：dev.of_node中
    dev = of_device_alloc(np, bus_id, parent);
    if (!dev)
        goto err_clear_flag;

    // 登记到 platform 中
    dev->dev.bus = &platform_bus_type;
    dev->dev.platform_data = platform_data;
    of_msi_configure(&dev->dev, dev->dev.of_node);

    // 添加当前生成的platform_device。
    if (of_device_add(dev) != 0) {
        platform_device_put(dev);
        goto err_clear_flag;
    }

    return dev;

err_clear_flag:
    of_node_clear_flag(np, OF_POPULATED);
    return NULL;
}
```



**of_device_alloc**

```c
struct platform_device *of_device_alloc(struct device_node *np,const char *bus_id,struct device *parent)
{
    //统计reg属性的数量
    while (of_address_to_resource(np, num_reg, &temp_res) == 0)
	    num_reg++;
    //统计中断irq属性的数量
    num_irq = of_irq_count(np);
    //根据num_irq和num_reg的数量申请相应的struct resource内存空间。
    if (num_irq || num_reg) {
        res = kzalloc(sizeof(*res) * (num_irq + num_reg), GFP_KERNEL);
        if (!res) {
            platform_device_put(dev);
            return NULL;
        }
        //设置platform_device中的num_resources成员
        dev->num_resources = num_reg + num_irq;
        //设置platform_device中的resource成员
        dev->resource = res;

        //将device_node中的reg属性转换成platform_device中的struct resource成员。  
        for (i = 0; i < num_reg; i++, res++) {
            rc = of_address_to_resource(np, i, res);
            WARN_ON(rc);
        }
        //将device_node中的irq属性转换成platform_device中的struct resource成员。 
        if (of_irq_to_resource_table(np, res, num_irq) != num_irq)
            pr_debug("not all legacy IRQ resources mapped for %s\n",
                np->name);
    }
    //将platform_device的dev.of_node成员指针指向device_node。  
    dev->dev.of_node = of_node_get(np);
    //将platform_device的dev.fwnode成员指针指向device_node的fwnode成员。
    dev->dev.fwnode = &np->fwnode;
    //设备parent为platform_bus
    dev->dev.parent = parent ? : &platform_bus;
}
```

首先，函数先统计设备树中reg属性和中断irq属性的个数，然后分别为它们申请内存空间，链入到platform_device中的struct resources成员中。
除了设备树中"reg"和"interrupt"属性之外，还有可选的"reg-names"和"interrupt-names"这些io中断资源相关的设备树节点属性也在这里被转换。
将相应的设备树节点生成的device_node节点链入到platform_device的dev.of_node中。
最终，我们能够通过在自己的驱动中，使用struct device_node *node = pdev->dev.of_node;获取到设备树节点中的数据。



**of_device_add**

```c
int of_device_add(struct platform_device *ofdev){
    // ...
    return device_add(&ofdev->dev);
}
```

将当前platform_device中的struct device成员注册到系统device中，并为其在用户空间创建相应的访问节点。
这一步会调用platform_match，因此最终也会执行设备树的match，以及probe。



## 4. 设备树语法

### 4.1 节点格式

```c
label: node-name@unit-address

其中：
label：标号
node-name：节点名字
unit-address：单元地址
label 是标号，可以省略。 label 的作用是为了方便地引用 node。比如：
//dts-v1
/{
  uart0: uart@fe001000{
    compatible="ns16550";
    reg=<0xfe001000 0x100>;
  };
};

可以使用如下两种方法修改uart@fe001000这个node:
// 在根结点之外使用 label 引用 node：
&uart0{
  status="disabled";
};

// 或在根节点之外使用全路径：
&{/uart@fe001000}{
  status="disabled";
}
// 这里的含义为修改 uart0 的成员: status
```



### 4.2 属性格式

简单地说， properties 就是“name=value”， value 有多种取值方式。 示例：
一个32位的数据，用尖括号包围起来，如

```c
interrupts = <17 0xc>; 
```

一个64位数据（使用2个32位数据表示），用尖括号包围起来，如：

```c
clock-frequency = <0x00000001 0x00000000>; 
```

有结束符的字符串，用双引号包围起来，如：

```c
compatible = "simple-bus"; 
```

字节序列，用中括号包围起来，如：

```c
local-mac-address = [00 00 12 34 56 78]; // 每个byte使用2个16进制数来表示 
local-mac-address = [000012345678];      // 每个byte使用2个16进制数来表示 
```

可以是各种值的组合，用逗号隔开，如：

```c
compatible = "ns16550", "ns8250"; 
example = <0xf00f0000 19>, "a strange property format";   
```

### 4.3 一些标准属性

（1） compatible 属性
“compatible”表示“兼容”，对于某个LED，内核中可能有A、B、C三个驱动都支持它，那可以这样写：

```c
led { 
    compatible = “A”, “B”, “C”; 
};
```

内核启动时，就会为这个LED按这样的优先顺序为它找到驱动程序：A、B、C。

（2）model 属性
model属性与compatible属性有些类似，但是有差别。compatible属性是一个字符串列表，表示可以你的硬件兼容A、B、C等驱动；model用来准确地定义这个硬件是什么。比如根节点中可以这样写：

```c
/ { 
    compatible = "samsung,smdk2440", "samsung,mini2440"; 
    model = "jz2440_v3"; 
};
```

它表示这个单板，可以兼容内核中的“smdk2440”，也兼容“mini2440”。从compatible属性中可以知道它兼容哪些板，但是它到底是什么板？用model属性来明确。

（3）status 属性
status 属性看名字就知道是和设备状态有关的， status 属性值也是字符串，字符串是设备的状态信息，可选的状态如下所示：

| 值         | 描述                                                         |
| ---------- | ------------------------------------------------------------ |
| "okay"     | 表明设备是可操作的                                           |
| "disabled" | 表明当前设备是不可操作的，但是在未来可以变为可操作的，比如热插拔设备插入之后。至于disabled的具体含义还是要看设备的绑定文档。 |
| "fail"     | 表明设备不可操作，设备检测到了一系列的错误，而且设备也不大可能变得可操作。 |
| "fail-sss" | 含义和"fail"相同，后面的sss部分是检测到的错误内容。          |

4）#address-cells 和#size-cells 属性
格式：

```c
address-cells：address要用多少个32位数来表示； 
size-cells：size要用多少个32位数来表示。 
```

比如一段内存，怎么描述它的起始地址和大小？下例中，address-cells为1，所以reg中用1个数来表示地址，即用0x80000000来表示地址；size-cells为1，所以reg中用1个数来表示大小，即用0x20000000表示大小：

```c
/ { 
address-cells = <1>;   
size-cells = <1>; 
    memory { 
        reg = <0x80000000 0x20000000>; 
    }; 
}; 
```

（5）reg 属性
reg的本意是register，用来表示寄存器地址。但是在设备树里，它可以用来描述一段空间。反正对于ARM系统，寄存器和内存是统一编址的，即访问寄存器时用某块地址，访问内存时用某块地址，在访问方法上没有区别。

reg属性的值，是一系列的“address size”，用多少个32位的数来表示address和size，由其父节点的# address-cells、#size-cells决定。示例：

```c
/dts-v1/; 
/ { 
    # address-cells = <1>;   
    # size-cells = <1>; 
    memory { 
        reg = <0x80000000 0x20000000>; 
    }; 
}; 
```

（7）name 属性
过时了，建议不用。它的值是字符串，用来表示节点的名字。在跟platform_driver匹配时，优先级最低。compatible属性在匹配过程中，优先级最高。

（8）device_type 属性
过时了，建议不用。它的值是字符串，用来表示节点的类型。在跟platform_driver匹配时，优先级为中。compatible属性在匹配过程中，优先级最高。

### 4.4 常用的节点

（1）根节点
用 / 标识根节点，如：

```c
/dts-v1/;   
/ {   
    model = "SMDK24440";   
    compatible = "samsung,smdk2440"; 
    
    # address-cells = <1>;   
    # size-cells = <1>;   
};   
```

（2）CPU节点
一般不需要我们设置，在 dtsi 文件中都定义好了，如：

```c
cpus {   
    # address-cells = <1>;   
    # size-cells = <0>;
    cpu0: cpu@0 {   
    	.......
    }   
};   
```

（3）memory 节点
芯片厂家不可能事先确定你的板子使用多大的内存，所以 memory 节点需要板厂设置，比如：

```c
memory { 
    reg = <0x80000000 0x20000000>; 
}; 
```

（4）chosen 节点
我们可以通过设备树文件给内核传入一些参数，这要在chosen节点中设置bootargs属性：

```c
chosen { 
    bootargs = "noinitrd root=/dev/mtdblock4 rw init=/linuxrc console=ttySAC0,115200"; 
};   
```

### 4.5 操作设备树的函数

Linux 内核给我们提供了一系列的函数来获取设备树中的节点或者属性信息，这一系列的函数都有一个统一的前缀“of_”（“open firmware”即开放固件。 ），所以在很多资料里面也被叫做 OF 函数。

#### 4.5.1 节点相关操作函数

Linux 内核使用 device_node 结构体来描述一个节点，此结构体定义在文件 include/linux/of.h 中，定义如下：

```c
struct device_node {
    const char *name;                       /* 节点名字 */
    phandle phandle;
    const char *full_name;                  /* 节点全名 */
    struct fwnode_handle fwnode;

    struct  property *properties;           /* 属性 */
    struct  property *deadprops;            /* removed 属性 */
    struct  device_node *parent;            /* 父节点 */
    struct  device_node *child;             /* 子节点 */
    struct  device_node *sibling;
#if defined(CONFIG_OF_KOBJ)
    struct  kobject kobj;
#endif
    unsigned long _flags;
    void    *data;
#if defined(CONFIG_SPARC)
    unsigned int unique_id;
    struct of_irq_controller *irq_trans;
#endif
};
```

与查找节点有关的 OF 函数有 5 个：

（1） of_find_node_by_name 函数
of_find_node_by_name 函数通过节点名字查找指定的节点，函数原型如下：

```c
struct device_node *of_find_node_by_name(struct device_node *from, const char *name);
```

（2） of_find_node_by_type 函数
of_find_node_by_type 函数通过 device_type 属性查找指定的节点，函数原型如下：

```c
struct device_node *of_find_node_by_type(struct device_node *from, const char *type);
```

（3） of_find_compatible_node 函数
of_find_compatible_node 函数根据 device_type 和 compatible 这两个属性查找指定的节点，函数原型如下：

```c
struct device_node *of_find_compatible_node(struct device_node *from,const char *type,const char *compatible);
```

（4）of_find_matching_node_and_match 函数
of_find_matching_node_and_match 函数通过 of_device_id 匹配表来查找指定的节点，函数原型如下：

```c
struct device_node *of_find_matching_node_and_match(struct device_node *from,const struct of_device_id *matches,const struct of_device_id **match);
```

（5）of_find_node_by_path 函数
of_find_node_by_path 函数通过路径来查找指定的节点，函数原型如下：

```c
inline struct device_node *of_find_node_by_path(const char *path);
```



#### 4.5.2 提取属性值的 OF 函数

Linux 内核中使用结构体 property 表示属性，此结构体同样定义在文件 include/linux/of.h 中，内容如下：

```c
struct property {
    char    *name;
    int length;
    void    *value;
    struct property *next;
#if defined(CONFIG_OF_DYNAMIC) || defined(CONFIG_SPARC)
    unsigned long _flags;
#endif
#if defined(CONFIG_OF_PROMTREE)
    unsigned int unique_id;
#endif
#if defined(CONFIG_OF_KOBJ)
    struct bin_attribute attr;
#endif
};
```

Linux 内核也提供了提取属性值的 OF 函数 ：

（1） of_find_property 函数
of_find_property 函数用于查找指定的属性，函数原型如下：

```c
property *of_find_property(const struct device_node *np,const char *name,int *lenp);
```

（2）of_property_count_elems_of_size 函数
of_property_count_elems_of_size 函数用于获取属性中元素的数量，比如 reg 属性值是一个数组，那么使用此函数可以获取到这个数组的大小，此函数原型如下：

```c
int of_property_count_elems_of_size(const struct device_node *np,const char *propname,int elem_size);
```

（3）读取 u8、 u16、 u32 和 u64 类型的数组数据

```c
static inline int of_property_read_u8_array(const struct device_node *np, const char *propname, u8 *out_values, size_t sz)
static inline int of_property_read_u16_array(const struct device_node *np, const char *propname, u16 *out_values, size_t sz)
static inline int of_property_read_u32_array(const struct device_node *np, const char *propname, u32 *out_values, size_t sz)
static inline int of_property_read_u64_array(const struct device_node *np, const char *propname, u64 *out_values, size_t sz)
```

（4）读取 u8、 u16、 u32 和 u64 类型属性值

```c
static inline int of_property_read_u8(const struct device_node *np, const char *propname, u8 *out_value)
static inline int of_property_read_u16(const struct device_node *np, const char *propname, u16 *out_value)
static inline int of_property_read_u32(const struct device_node *np, const char *propname, u32 *out_value)
static inline int of_property_read_s32(const struct device_node *np, const char *propname, s32 *out_value)
int of_property_read_u64(const struct device_node *np, const char *propname, u64 *out_value);
```

（5）of_property_read_string 函数
of_property_read_string 函数用于读取属性中字符串值，函数原型如下：

```c
int of_property_read_string(struct device_node *np,const char *propname,const char **out_string)
```
