# 程序加载与动态链接

# Program Header

## 概述

可执行文件或者共享目标文件的头部是一个结构体数组，每一个元素都描述了一个段或者其它系统在准备程序执行时所需要的信息。一个目标文件的段包含一个或者多个节。程序的头部只有对于可执行文件和共享目标文件有意义。其中，ELF文件的头中的`e_phentsize`和`e_phnum`项指定了相应的程序头的大小。程序头的数据结构如下

```c++
typedef struct {
	ELF32_Word	p_type;
	ELF32_Off	p_offset;
	ELF32_Addr	p_vaddr;
	ELF32_Addr	p_paddr;
	ELF32_Word	p_filesz;
	ELF32_Word	p_memsz;
	ELF32_Word	p_flags;
	ELF32_Word	p_align;
} Elf32_Phdr;
```

对于每个字段的说明如下

| 字段       | 说明                                       |
| -------- | ---------------------------------------- |
| p_type   | 该字段表明了对应数组元素的类型，或者表明了该如何解释该数组元素。具体信息可以参见下面的描述。 |
| p_offset | 该字段给出了从文件开始到该段开头的第一个字节的偏移。               |
| p_vaddr  | 该字段给出了该段的第一个字节在内存中的虚拟地址。                 |
| p_paddr  | 该字段仅用于物理地址寻址相关的系统中， 由于”System V”忽略了应用程序的物理寻址，可执行文件和共享目标文件的该项内容并未被限定。 |
| p_filesz | 该字段给出了文件镜像中该段的大小，可能为0。                   |
| p_memsz  | 该字段给出了内存镜像中该段的大小，可能为0。                   |
| p_flags  | 该字段给出了与段相关的标记。                           |
| p_align  | 可加载的程序的段的p_vaddr以及p_offset的大小必须合适，即必须是page 的整数倍。该成员给出了段在文件以及内存中的对齐方式。如果该值为0或1的话，表示不需要对齐。除此之外，p_align应该是2的整数指数次方，并且p_vaddr与p_offset在模p_align的意义下，应该相等。 |

## 段类型

可执行文件中的段类型如下

| 名字                  | 取值                      | 说明                                       |
| ------------------- | ----------------------- | ---------------------------------------- |
| PT_NULL             | 0                       | 表明数组元素未使用，其结构中其他成员都是未定义的。                |
| PT_LOAD             | 1                       | 表明对应的段为一个可加载的段，段的大小由 p_filesz 和 p_memsz  描述。文件中的字节被映射到相应内存段开始处。如果 p_memsz  大于  p_filesz，“剩余”的字节都要被置为0。p_filesz 不能大于 p_memsz。可加载的段在程序头部中按照 p_vaddr 的升序排列。 |
| PT_DYNAMIC          | 2                       | 该数组元素给出动态链接信息。                           |
| PT_INTERP           | 3                       | 表明对应的段给出了一个以 NULL  结尾的字符串的位置和长度，该字符串将被当作解释器调用。这种段类型仅对与可执行文件有意义（也可能出现在在共享目标文件中）。此外，这种段在一个文件中最多出现一次。而且这种类型的段存在的话，它必须在所有可加载段项的前面。 |
| PT_NOTE             | 4                       | 此数组元素给出附加信息的位置和大小。                       |
| PT_SHLIB            | 5                       | 该段类型被保留，不过语义未指定。而且，包含这种类型的段的程序不符合ABI标准。  |
| PT_PHDR             | 6                       | 该段类型的数组元素如果存在的话，则给出了程序头部表自身的大小和位置，既包括在文件中也包括在内存中的信息。此类型的段在文件中最多出现一次。此外，只有程序头部表是程序的内存映像的一部分时，它才会出现。如果此类型段存在，则必须在所有可加载段项目的前面。 |
| PT_LOPROC~PT_HIPROC | 0x70000000  ~0x7fffffff | 此范围的类型保留给处理器专用语义。                        |

## 基地址-Base Address

程序头部的虚拟地址可能并不是程序内存镜像中实际的虚拟地址。可执行程序通常来说，都会包含绝对的代码。为了使得程序可以正常执行，段必须在相应的虚拟地址处。另一方面，共享目标文件通常来说包含与位置无关的代码。这可以使得共享目标文件可以被多个进程加载，同时保持程序执行的正确性。尽管系统会为不同的进程选择不同的虚拟地址，但是它仍然保留段的相对地址，因为位置无关代码使用段之间的相对地址来进行寻址，内存中的虚拟地址之间的差必须与文件中的虚拟地址之间的差相匹配。内存中任何段的虚拟地址与文件中对应的虚拟地址之间的差值对于任何一个可执行文件或共享对象来说是一个单一常量值。这个差值就是基地址，基地址的一个用途就是在动态链接期间重新定位程序。

可执行文件或者共享目标文件的基地址是在执行过程中由以下三个数值计算的

- 虚拟内存加载地址
- 最大页面大小
- 程序可加载段的最低虚拟地址

要计算基地址，首先要确定可加载段中p_vaddr最小的内存虚拟地址，之后把该内存虚拟地址缩小为与之最近的最大页面的整数倍即是基地址。根据要加载到内存中的文件的类型，内存地址可能与 p_vaddr 相同也可能不同。

## 段权限

被系统加载到内存中的程序必须至少有一个可加载的段。当系统为其创造可加载的段的内存镜像时，它会按照p_flags将段设置为对应的权限。

可能的段权限位有

![](/executable/ELF/figure/segment_flag_bits.png)

其中，所有在PF_MASKPROC中的比特位都是被保留用于与处理器相关的语义信息。

如果一个权限位被设置为0，这种类型的段是不可访问的。实际的内存权限取决于相应的内存管理单元，不同的系统可能操作方式不一样。尽管所有的权限组合都是可以的，但是系统一般会授予比请求更多的权限。在任何情况下，除非明确说明，一个段不会有写权限。下面给出了所有的可能组合。

![](/executable/ELF/figure/segment-permission.png)

例如，一般来说.text段一般具有读和执行权限，但是不会有写权限。数据段一般具有写，读，以及执行权限。

## 段内容

一个段可能包括一到多个节区，但是这并不会影响程序的加载。尽管如下，我们也必须需要各种各样的数据来使得程序可以执行以及动态链接等等。下面会给出一般情况下的段的内容。对于不同的段来说，它的节的顺序以及所包含的节的个数有所不同。此外，与处理相关的约束可能会改变对应的段的结构。

如下所示，代码段只包含只读的指令以及数据。其它节可能在可加载的段中。当然这个例子并没有给出所有的可能的段。

![](/executable/ELF/figure/text_segment.png)

数据段包含可写的数据以及以及指令，通常来说，包含以下内容

![](/executable/ELF/figure/data_segment.png)

程序头部的PT_DYNAMIC元素指向.dynamic节。其中got表和plt表包含与位置独立代码有关的信息。尽管在这里给出的例子中，plt节出现在代码段，但是对于不同的处理器来说，可能会有所变动。

之前我们也说了，.bss节的类型为SHT_NOBITS，这表明它在ELF文件中不占用空间，但是它却占用内存镜像中的可执行文件的空间。通常情况下，这些没有被初始化的数据在段的尾部，因此p_memsz才会比p_filesz大。



​	