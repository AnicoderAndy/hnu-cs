---
author: AnicoderAndy
---
# 实验三 设备树
AnicoderAndy


<!-- @import "[TOC]" {cmd="toc" depthFrom=2 depthTo=3 orderedList=false} -->

<!-- code_chunk_output -->

- [实验过程](#实验过程)
  - [获取 `dtb` 文件](#获取-dtb-文件)
  - [实现设备树解析程序](#实现设备树解析程序)
- [实验心得](#实验心得)

<!-- /code_chunk_output -->


## 实验过程

设备树（Device Tree, DT）是一种硬件描述机制，用于以独立于平台的方式描述嵌入式系统中的硬件信息。通过设备树，硬件的各个外设节点（如 CPU、内存、串口等）及其属性（地址、兼容性等）以树形结构组织，并最终以二进制 DTB（Flat Device Tree Blob）格式传递给操作系统内核。在本系列实验中，我们暂不使用 Bootloader 传递设备树给内核。因此，本实验的任务可选地实现一个设备树解析程序，而不是将设备树直接用于内核启动。

### 获取 `dtb` 文件

使用 QEMU 自带功能来导出 DTB。使用 QEMU 的 `-machine virt,dumpdtb=virt.dtb` 选项，让 QEMU 在启动时将内部生成的 DTB 导出到当前目录中：

```bash
qemu-system-aarch64 -machine virt,dumpdtb=virt.dtb -cpu cortex-a53 -nographic -m 1024
```

这条命令启动 QEMU 虚拟机并立即将设备树信息保存在 `virt.dtb`。生成后，可使用 `dtc -I dtb -O dts virt.dtb > virt.dts` 将二进制转换成人可读的 DTS 格式，以便核对和分析。该步骤相当于搭建了实验环境的硬件描述输出，为下一步编写解析程序做好准备。

### 实现设备树解析程序

下面尝试使用 C 语言编写一个简单的设备树解析程序。首先用 `fopen` 读入 `virt.dts` 文件；然后调用 libfdt 提供的 `fdt_check_header(fdt_blob)` 来验证缓冲区是否为有效的设备树格式；接着使用 libfdt 的 API 定位并读取节点。例如，可调用 `fdt_path_offset(fdt, "/cpus")` 查找 `/cpus` 节点偏移量，然后使用 `fdt_first_subnode/fdt_next_subnode` 或 `fdt_path_offset` 定位子节点 `cpu@0`、`cpu@1` 等。对于每个节点，可调用 `fdt_getprop(fdt, nodeoffset, "compatible", NULL)` 读取“compatible”属性，或读取 `reg`、`device_type`、`#address-cells` 等属性。

```c
#include <libfdt.h>
#include <stdio.h>
#include <stdlib.h>
#include <stdarg.h>

/**
 * @brief 根据指定路径加载设备树二进制文件（DTB）。
 *
 * @param path 设备树二进制文件的路径。
 * @param size `OUT`，返回加载的 DTB 文件大小。
 * @return void* 指向加载的 DTB 数据的指针。
 */
static void* load_dtb(const char* path, size_t* size) {
    FILE* fp = fopen(path, "rb");
    if (!fp) {
        perror("Open DTB");
        exit(1);
    }
    fseek(fp, 0, SEEK_END);
    long fsize = ftell(fp);
    rewind(fp);
    void* buf = malloc(fsize);
    if (!buf) {
        perror("Malloc DTB");
        exit(1);
    }
    fread(buf, 1, fsize, fp);
    fclose(fp);
    *size = fsize;
    return buf;
}

/**
 * @brief 递归打印节点及其子节点的信息
 *
 * @param fdt 设备树数据指针
 * @param node_offset 当前节点的偏移量
 * @param depth 当前递归深度（用于缩进）
 */
static void print_node_recursive(const void* fdt, int node_offset, int depth) {
    // 打印缩进
    for (int i = 0; i < depth; i++) {
        printf("  ");
    }

    // 获取并打印节点名称
    const char* name = fdt_get_name(fdt, node_offset, NULL);
    printf("Node: %s\n", name);

    // 打印该节点的 compatible 属性（如果存在）
    const char* compat = fdt_getprop(fdt, node_offset, "compatible", NULL);
    if (compat) {
        for (int i = 0; i < depth + 1; i++)
            printf("  ");
        printf("compatible: %s\n", compat);
    }

    // 遍历所有子节点
    int child = fdt_first_subnode(fdt, node_offset);
    while (child >= 0) {
        print_node_recursive(fdt, child, depth + 1);
        child = fdt_next_subnode(fdt, child);
    }
}

int main(int argc, char* argv[]) {
    if (argc < 2) {
        fprintf(stderr, "Usage: %s virt.dtb\n", argv[0]);
        return 1;
    }
    size_t dtb_size;
    void* fdt = load_dtb(argv[1], &dtb_size);
    if (fdt_check_header(fdt) != 0) {
        fprintf(stderr, "Invalid FDT file\n");
        return 1;
    }
    
    int offset = -1;
    char path[256];
    while (1) {
        printf("Enter node path to print (or press ctrl-d to quit): ");
        if (scanf("%s", path) != 1) {
            break;
        }   

        offset = fdt_path_offset(fdt, path);
        if (offset < 0) {
            printf("Node not found: %s\n", path);
        } else {
            print_node_recursive(fdt, offset, 0);
        }
    }

    free(fdt);
    return 0;
}
```

编译时链接 `libfdt`：

```bash
❯ gcc parse_fdt.c -o parse_fdt -lfdt
```

执行 `./parse_fdt virt.dts`，然后输入 `/cpus` 或 `/memory` 等路径，可以看到输出类似于：

```
Enter node path to print (or press ctrl-d to quit): /cpus
Node: cpus
  Node: cpu-map
    Node: socket0
      Node: cluster0
        Node: core0
  Node: cpu@0
    compatible: arm,cortex-a53
Enter node path to print (or press ctrl-d to quit): /memory
Node: memory@40000000
```

## 实验心得

在本次实验中，我对设备树（Device Tree）这一嵌入式系统中重要的硬件描述机制有了更加深入的理解。设备树本质上是一种将硬件结构抽象化的数据结构，它用层级式的节点来描述各种硬件资源及其属性。在 ARM 平台中，尤其是像 QEMU 模拟的虚拟平台里，设备树作为硬件信息传递给内核的桥梁，扮演着不可替代的角色。

在设备树解析程序的实现过程中，我首次系统地接触并使用了 libfdt 这一库。通过对其 API 的查阅和实际动手编写代码，我逐步掌握了如何读取 `.dtb` 文件、验证其合法性、遍历节点以及提取节点属性等操作。在程序的交互逻辑设计上，我选择使用控制台输入路径的方式，以动态展示不同节点及其子节点的结构和属性，这种方式不仅增强了程序的通用性，也提升了实验的趣味性和实用价值。调试过程中也遇到一些挑战，比如路径输入后节点查找不到的情况等。这些问题的排查和解决，锻炼了我对 C 语言基本功的掌握，也让我更关注程序的健壮性设计。

总的来说，这次实验虽然篇幅不长，但内容却涵盖了嵌入式系统开发中的多个关键知识点。从设备树的概念，到 QEMU 模拟平台的使用，再到实际的代码实现与调试，每一个环节都让我感受到底层开发工作的严谨性与系统性。设备树作为软硬件之间的重要接口，其结构清晰、语义明确的设计思想也让我对系统架构设计产生了更多兴趣。我相信，在后续更复杂的内核或驱动开发中，对设备树的理解与操作能力会成为一项非常有价值的基础技能。