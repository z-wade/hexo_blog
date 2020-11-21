---
title: fishhook 原理笔记
date: 2020-11-20 11:36:55
tags: fishhook 
categories: 
---
在OC中想在hook一个函数，绝对是利用runtime swizzle 方法。但是对于c函数我们应该怎么hook呢？
没错，就是利用今天的主角fishhook

### 使用方法
我们以 hook `NSlog()` 为例

先定义我们自己的 `log` 函数 与 `NSLog` 的函数指针

```
void mLog(NSString * format, ...) {
    format = [format stringByAppendingFormat:@"勾上了！\n"];
    sys_nslog(format);
}

static void(*sys_nslog)(NSString * format, ...);
```


声明 `rebinding` 的结构体。这个结构体包含了重新绑定的信息。
```
    struct rebinding nslog;
    nslog.name = "NSLog";
    nslog.replacement = mLog;
    nslog.replaced = (void *)&sys_nslog;
    //rebinding结构体数组
    struct rebinding rebs[1] = {nslog};
    
    rebind_symbols(rebs, 1);
```

最终我们调用 `NSLog`方法，打印如下：

>  xxx 勾上了！


此处有一个小问题？如果在这个项目里面的 c 函数能hook住吗？答案是不能！

### 原理解释

新建一个项目，在`ViewDidload`里面打印文本，把生成的可执行文件拖到 Hopper Disassembler 中。
![image](http://note.youdao.com/yws/res/11074/3E4E5CE8E1784419AA9987A131D32AB1)
![image](http://note.youdao.com/yws/res/11076/FD07B897C3AB4054B8DBCE6A09D97430)

这里跳到 `imp__stubs_NSLog` 这个位置，进来继续看
![image](http://note.youdao.com/yws/res/11079/34EF700BA34D4EFBAFFAA07AD2B18E3E)
上面是把 `0x0c00` 位置的值放到x16寄存器上，我们看一下地址

![image](http://note.youdao.com/yws/res/11081/A26D9FE9259648F9AC3CC5D1AF87C34E)
找到 `0x0C000` 的地址，这里是 `Data Segment`，`__la_symbol_ptr Section`。这里有两个点很重要：
* 属于`Data`段，可读写
* `__la_symbol_ptr section` 是存放着动态链接的符号表，这里面的符号会在该符号被调用时，通过dyld的`dyld_stb_binder`过程进行加载。
还有一种动态链接相关的表是 `__nl_symbol_ptr` 表示在动态链接库绑定的时候进行加载的

到这里就很显示了，fishhook 就是通过修改 `__la_symbol_ptr` 这张符号表里面的值来达到hook c函数的目的。
下面我们来验证一下

### 原理验证

在一个新项目的`ViewController`里面写上面代码，然后我们关注 `__nl_symbol_ptr` 这个符号表里面` NSLog` 对应地址的变化。把可执行文件拖到 `MachOView`里面
![image](http://note.youdao.com/yws/res/11084/56B20444852F487B9FEC5E0DCCBD9D9A)
![image](http://note.youdao.com/yws/res/11087/FCFC1F2CA5734559B74D21105AB1C077)
位置是这个`0xc000`，运行程序，进入断点。找到`ALSR`的偏移地址
![image](http://note.youdao.com/yws/res/11089/25D8F304A6B94246B9E29435A4BD1A14)


![image](http://note.youdao.com/yws/res/11092/CEFFF74E73734F88B3293D009B413689)
lldb `x`命令指地址的值读出来，`dis -s` 以当前赋值的地址反汇编。
可以看出来，我们`NSLog` 调用的地址，暂时看不出来是什么（实际是调了`dyld_stb_binder`相关函数，进行动态绑定)。

接下来，我们继续走下一步，NSLog动态符号表对应的值 发生改变，对当前地址反汇编后发现是 `Foundation` 的`NSLog` 方法。

![image](http://note.youdao.com/yws/res/11096/19F11E662F9748CA8E3436BCFC814599)

最后，让代码继续走过`rebind_symbols`方法，又又发现`NSLog`动态符号表对应的值 发生改变，对当前地址反汇编后发现是我们自定义的方法

![image](http://note.youdao.com/yws/res/11098/CD88539E45FF489DABF380BA2AC4BD2A)

结论很明显，`__nl_symbol_ptr` 处于Data段，可读写。动态符号可通过这里对应的地址值进行跳转，`fishhook`就是通过修改这里的值达到Hook目的。

### fishhook源码解读
拿`Mach-O`文件和源码的操作一起对着来看。

#### rebind_symbols 方法

```
int rebind_symbols(struct rebinding rebindings[], size_t rebindings_nel) {
    // 把要被hook的函数信息用链表保存起来
    // 头插法的形式添加到_rebindings_head这里
  int retval = prepend_rebindings(&_rebindings_head, rebindings, rebindings_nel);
  if (retval < 0) {
    return retval;
  }
  // 第一次的话调用，注册加载镜像的方法_dyld_register_func_for_add_image，
  // 如果已经加载过的镜像，会直接调用回调方法。其他的会在加载的时候回调）
  if (!_rebindings_head->next) {
    _dyld_register_func_for_add_image(_rebind_symbols_for_image);
  } else {
    uint32_t c = _dyld_image_count();
    for (uint32_t i = 0; i < c; i++) {
      _rebind_symbols_for_image(_dyld_get_image_header(i), _dyld_get_image_vmaddr_slide(i));
    }
  }
  return retval;
}
```

第一步：preped_rebindings，这部分主要是把 rebindings_entry 以一个链表的形式保存起来。

```
static int prepend_rebindings(struct rebindings_entry **rebindings_head,
                              struct rebinding rebindings[],
                              size_t nel) {
  struct rebindings_entry *new_entry = malloc(sizeof(struct rebindings_entry));
  if (!new_entry) {
    return -1;
  }
  new_entry->rebindings = malloc(sizeof(struct rebinding) * nel);
  if (!new_entry->rebindings) {
    free(new_entry);
    return -1;
  }
  // 头插法，把数据保存起来
  memcpy(new_entry->rebindings, rebindings, sizeof(struct rebinding) * nel);
  new_entry->rebindings_nel = nel;
  new_entry->next = *rebindings_head;
  *rebindings_head = new_entry;
  return 0;
}
```

从上面看到，入口函数最终会调到_rebind_symbols_for_image，从页调到rebind_symbols_for_image这个函数里面。
```
static void _rebind_symbols_for_image(const struct mach_header *header,
                                      intptr_t slide) {
    rebind_symbols_for_image(_rebindings_head, header, slide);
}
```

#### rebind_symbols_for_image

其实重头戏也在这个(rebind_symbols_for_image)函数里面，

```
///
/// @param rebindings 需要被交换方法的数组
/// @param header Mach-O文件的header，通过_dyld_get_image_header获取
/// @param slide ASLR的偏移地址(_dyld_get_image_vmaddr_slide获取)
static void rebind_symbols_for_image(struct rebindings_entry *rebindings,
                                     const struct mach_header *header,
                                     intptr_t slide) {

```

这里可以分三部分来讲解：

**第一部分：确定这个header是否合法**

dladdr函数主要作用是获取 &info 所在动态库的信息，如果获取不到，就证明这个header地址是不合法的。

```
  Dl_info info;
  if (dladdr(header, &info) == 0) {
    return;
  }
```


**第二部分：可以看到在查找三个 load_command**

分别是
linkedit_segment、symtab_cmd、dysymtab_cmd，

```
// loadCommand的开始地址在header的后面
    uintptr_t cur = (uintptr_t)header + sizeof(mach_header_t);
    // 遍历所有的loadCommands
    for (uint i = 0; i < header->ncmds; i++, cur += cur_seg_cmd->cmdsize) {
        cur_seg_cmd = (segment_command_t *)cur;
        // Command为LC_SEGMENT_64有好几个（PAGEZERO, TEXT, DATA_COST, DATA, LINKEDIT)
        if (cur_seg_cmd->cmd == LC_SEGMENT_ARCH_DEPENDENT) {
            // 根据segmentName 找到linkedit_segment
            if (strcmp(cur_seg_cmd->segname, SEG_LINKEDIT) == 0) {
                linkedit_segment = cur_seg_cmd;
            }
            
        } else if (cur_seg_cmd->cmd == LC_SYMTAB) {
            symtab_cmd = (struct symtab_command*)cur_seg_cmd;
        } else if (cur_seg_cmd->cmd == LC_DYSYMTAB) {
            dysymtab_cmd = (struct dysymtab_command*)cur_seg_cmd;
        }
    }
    //如果上面的三个loadCommand有一个有空，就返回
    if (!symtab_cmd || !dysymtab_cmd || !linkedit_segment ||
        !dysymtab_cmd->nindirectsyms) {
        return;
    }
```

分析一下这三个LoadCommand主要会加载哪些信息

**LINKEDIT Commond**

![image](http://note.youdao.com/yws/res/11102/F5E0CE7273D9491BBF80EDBF3E9A74E1)

![image](http://note.youdao.com/yws/res/11104/4E29A4C157B74B1BBC123FF7A45B9E30)

根据地址可以看到，linkedit_segment 这个命令是用于加载 Dynamic Loader Info 相关的信息。

**LC_SYMTAB(符号表)**

![image](http://note.youdao.com/yws/res/11118/FE859280434E41B9B586AAF9D3FF42A6)
![image](http://note.youdao.com/yws/res/11120/E0F65B3221AC4A209900176EC8386012)

**LC_DYSYMTAB(动态符号表相关)**

![image](http://note.youdao.com/yws/res/11122/62D4609C42D14B088C640625E4C5F3AE)

我们能看到这个 `IndSym Table Offset` 的值是 `0x11648`，其他的符号表的offset 是0。

> `IndSym Table Offset` 用于存放在字符表的位置

![image](http://note.youdao.com/yws/res/11125/D0E6EE3AD8A24C978CE929AAAA132E2A)
通过上面的分析，我们搞明白了上面这几个load_command的作用了。那么回过头业，继续往下看第三部分


**第三部分找到各个 符号表地址、字符串表地址、动态符号表地址，然后调用重新绑定函数的方法进行替换**

```
    // Find base symbol/string table addresses
    //链接时程序的基址 = __LINKEDIT.VM_Address -__LINKEDIT.File_Offset + silde的改变值
    uintptr_t linkedit_base = (uintptr_t)slide + linkedit_segment->vmaddr - linkedit_segment->fileoff;
    //符号表的地址 = 基址 + 符号表偏移量
    nlist_t *symtab = (nlist_t *)(linkedit_base + symtab_cmd->symoff);
    //字符串表的地址 = 基址 + 字符串表偏移量
    char *strtab = (char *)(linkedit_base + symtab_cmd->stroff);
    
    // Get indirect symbol table (array of uint32_t indices into symbol table)
    //动态符号表地址 = 基址 + 动态符号表偏移量
    uint32_t *indirect_symtab = (uint32_t *)(linkedit_base + dysymtab_cmd->indirectsymoff);
    
    cur = (uintptr_t)header + sizeof(mach_header_t);
    for (uint i = 0; i < header->ncmds; i++, cur += cur_seg_cmd->cmdsize) {
        cur_seg_cmd = (segment_command_t *)cur;
        if (cur_seg_cmd->cmd == LC_SEGMENT_ARCH_DEPENDENT) {
            //非Data段的数据过滤掉，__la_symbol_ptr和__no_la_symbol_ptr(got)这两个section都是放在Data段上
            if (strcmp(cur_seg_cmd->segname, SEG_DATA) != 0 &&
                strcmp(cur_seg_cmd->segname, SEG_DATA_CONST) != 0) {
                continue;
            }
            
            for (uint j = 0; j < cur_seg_cmd->nsects; j++) {
                section_t *sect =
                (section_t *)(cur + sizeof(segment_command_t)) + j;
                //找懒加载表
                if ((sect->flags & SECTION_TYPE) == S_LAZY_SYMBOL_POINTERS) {
                    perform_rebinding_with_section(rebindings, sect, slide, symtab, strtab, indirect_symtab);
                }
                //非懒加载表
                if ((sect->flags & SECTION_TYPE) == S_NON_LAZY_SYMBOL_POINTERS) {
                    perform_rebinding_with_section(rebindings, sect, slide, symtab, strtab, indirect_symtab);
                }
            }
        }
    }
```
![image](http://note.youdao.com/yws/res/11131/F2EA10D9D2C44B5795BD2C5F9A9834CE)

至于为什么叫 `got` 段，估计是和 ELF 文件有关系

找到了这 `__la_symbol_prt` `Section`的位置后，就进入了查找符号位置，替换方法的逻辑。

```
/// 找到符号表所对应的位置进行绑定
/// @param rebindings rebindings 要交换的数组
/// @param section section_t当前loadcommand的section(__la_symbol_ptr, __no_la_symbol_ptr)
/// @param slide slide alsr程序偏移
/// @param symtab 符号表地址 10608
/// @param strtab 字符串表地址 11708
/// @param indirect_symtab 动态符号表地址 11648
static void perform_rebinding_with_section(struct rebindings_entry *rebindings,
                                           section_t *section,
                                           intptr_t slide,
                                           nlist_t *symtab,
                                           char *strtab,
                                           uint32_t *indirect_symtab) {
    //获取__la_symbol_ptr 和 __no_la_symbol的在Indirect Symbols的起始位置
    uint32_t *indirect_symbol_indices = indirect_symtab + section->reserved1;
    //slide+section->addr 就是符号对应的存放函数实现的数组也就是我相应的__nl_symbol_ptr和__la_symbol_ptr相应的函数指针都在这里面了，所以可以去寻找到函数的地址
    void **indirect_symbol_bindings = (void **)((uintptr_t)slide + section->addr); // 0xc000 (__DATA, __la_symbol_ptr的地址)
    
    //遍历__la_symbol_ptr/ __no_la_symbol 里面的所有符号
    for (uint i = 0; i < section->size / sizeof(void *); i++) {
        //找到符号在Indrect Symbol Table表中的值
        //读取indirect table中的数据
        uint32_t symtab_index = indirect_symbol_indices[i];
        if (symtab_index == INDIRECT_SYMBOL_ABS || symtab_index == INDIRECT_SYMBOL_LOCAL ||
            symtab_index == (INDIRECT_SYMBOL_LOCAL   | INDIRECT_SYMBOL_ABS)) {
            continue;
        }
        //以symtab_index作为下标，访问symbol table
        uint32_t strtab_offset = symtab[symtab_index].n_un.n_strx;
        //获取到symbol_name
        char *symbol_name = strtab + strtab_offset;
        //判断是否函数的名称是否有两个字符，为啥是两个，因为函数前面有个_，所以方法的名称最少要1个
        bool symbol_name_longer_than_1 = symbol_name[0] && symbol_name[1];
        //遍历最初的链表，来进行hook
        struct rebindings_entry *cur = rebindings;
        while (cur) {
            for (uint j = 0; j < cur->rebindings_nel; j++) {
                //这里if的条件就是判断从symbol_name[1]两个函数的名字是否都是一致的，以及判断两个
                if (symbol_name_longer_than_1 &&
                    strcmp(&symbol_name[1], cur->rebindings[j].name) == 0) {
                    //判断replaced的地址不为NULL以及我方法的实现和rebindings[j].replacement的方法不一致
                    if (cur->rebindings[j].replaced != NULL &&
                        indirect_symbol_bindings[i] != cur->rebindings[j].replacement) {
                        //让rebindings[j].replaced保存indirect_symbol_bindings[i]的函数地址
                        *(cur->rebindings[j].replaced) = indirect_symbol_bindings[i];
                    }
                    //将替换后的方法给原先的方法，也就是替换内容为自定义函数地址
                    indirect_symbol_bindings[i] = cur->rebindings[j].replacement;
                    goto symbol_loop;
                }
            }
            cur = cur->next;
        }
    symbol_loop:;
    }
}
```


### 例子

下面以`NSLog`为例，一步一步使用 `Mach-O`对应的把流程走完看看，（`NSLog` 在 `__la_symbol_ptr` 符号表里面）

```
uint32_t *indirect_symbol_indices = indirect_symtab + section->reserved1; // 获取 __la_symbol_ptr 符号在 indSymBol的开始位置。
```
![image](http://note.youdao.com/yws/res/11134/390BDA93A84D4804A870194A5C6BA337)


下面这图可以看到在` Indirect Symbols` 偏移是 0x19，也就是第25个。
![image](http://note.youdao.com/yws/res/11136/032ECA434D004DBB8D62BC59429FA9F5)

走到这步, 其实就是 `__la_symbol_ptr`的位置
```
void **indirect_symbol_bindings = (void **)((uintptr_t)slide + section->addr);
```
![image](http://note.youdao.com/yws/res/11141/7D700537F35D4A54B4D4B4DFA22DF5B3)


进入for 循环第一句，获取在 symtab_index，在 0xE3 这个位置
uint32_t symtab_index = indirect_symbol_indices[i];

![image](http://note.youdao.com/yws/res/11143/D4B2F03BAE984DD68B4C39294D299B89)


根据在 符号表里面的Index(0xE3 = 227)，取到符号对象，并获取在字符串表 strtab的位置。
```
//以symtab_index作为下标，访问symbol table
uint32_t strtab_offset = symtab[symtab_index].n_un.n_strx;
```
![image](http://note.youdao.com/yws/res/11145/59BC19B3C3764FAF860D87B0E490C739)


所以我们取到的字符的位置 0x11708 + 0xD4 = 0x117DC
![image](http://note.youdao.com/yws/res/11147/BE8A884242424E34987CE975B362F38F)

然后判断symbol是否大于一，因为前面的 "_"字符是编译期加上的。
```
bool symbol_name_longer_than_1 = symbol_name[0] && symbol_name[1];
```

进入while循环，把rebindings都过一遍，看看和上面字符相等的函数名，进行方法替换。
```
        while (cur) {
            for (uint j = 0; j < cur->rebindings_nel; j++) {
                //这里if的条件就是判断从symbol_name[1]两个函数的名字是否都是一致的，以及判断两个
                if (symbol_name_longer_than_1 &&
                    strcmp(&symbol_name[1], cur->rebindings[j].name) == 0) {
                    //判断replaced的地址不为NULL以及我方法的实现和rebindings[j].replacement的方法不一致
                    if (cur->rebindings[j].replaced != NULL &&
                        indirect_symbol_bindings[i] != cur->rebindings[j].replacement) {
                        //让rebindings[j].replaced保存indirect_symbol_bindings[i]的函数地址
                        *(cur->rebindings[j].replaced) = indirect_symbol_bindings[i];
                    }
                    //将替换后的方法给原先的方法，也就是替换内容为自定义函数地址
                    indirect_symbol_bindings[i] = cur->rebindings[j].replacement;
                    goto symbol_loop;
                }
            }
            cur = cur->next;
        }
```

我们回顾一下上面查找符号的过程：
![image](http://note.youdao.com/yws/res/11149/7B84BD213F1847099B1AA972630B3A59)

1. 从 `Loadcomamd` 的 `reserved1` 的位置，找到在 `Indirect Symbol Table` 的开始位置。
2. 进入循环，从开始位置遍历 `Indirect Symbol Table`，长度是`Section.size`
3. 遍历过程中，拿到`Symbol Table`的位置，从而知道符号的字符串
4. 遍历过程中, 字符串和被替换相匹配，如果匹配就进行替换函数。
















