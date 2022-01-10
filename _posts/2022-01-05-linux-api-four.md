---
title: 篇四：在Linux内核驱动中常用到API接口函数
layout: post
categories: Linux
tags: linux
excerpt: 在Linux内核中，我们时常会用到API接口函数，设备树节点API。
---

## 设备树
## 通用
## 查找节点
通过名字查找
```sh
struct device_node *of_find_node_by_name(struct device_node *from, const char *name);
from：开始查找的节点，如果为 NULL 表示从根节点开始查找整个设备树。
name：要查找的节点名字（不是table和name属性）。
返回值： 找到的节点，如果为 NULL 表示查找失败。
```

通过device_type 属性查找
```sh
struct device_node *of_find_node_by_type(struct device_node *from, const char *type)
from：开始查找的节点，如果为 NULL 表示从根节点开始查找整个设备树。
type：要查找的节点对应的 type 字符串，即 device_type 属性值。
返回值： 找到的节点，如果为 NULL 表示查找失败。
```

根据 device_type 和 compatible查找
```sh
struct device_node *of_find_compatible_node(struct device_node *from, const char *type, const char *compatible)
from：开始查找的节点，如果为 NULL 表示从根节点开始查找整个设备树。
type：要查找的节点对应的 type 字符串，即 device_type 属性值（若为 NULL则表示忽略 device_type 属性）
compatible： 要查找的节点所对应的 compatible 属性列表。
返回值： 找到的节点，如果为 NULL 表示查找失败
```

通过 of_device_id 匹配表来查找
```sh
struct device_node *of_find_matching_node_and_match(struct device_node *from, const struct of_device_id *matches, const struct of_device_id **match)
from：开始查找的节点，如果为 NULL 表示从根节点开始查找整个设备树。
matches： of_device_id 匹配表，也就是在此匹配表里面查找节点。
match： 找到的匹配的 of_device_id。
返回值： 找到的节点，如果为 NULL 表示查找失败
```

通过路径查找
```sh
inline struct device_node *of_find_node_by_path(const char *path)
path：带有全路径的节点名，可以使用节点的别名。
返回值： 找到的节点，如果为 NULL 表示查找失败
```

查找指定节点的父节点
```sh
struct device_node *of_get_parent(const struct device_node *node)
node：要查找的父节点的节点。
返回值： 找到的父节点。
```

查找指定节点的子节点
```sh
struct device_node *of_get_next_child(const struct device_node *node, struct device_node *prev)
node：父节点。
prev：前一个子节点，也就是从哪一个子节点开始迭代的查找下一个子节点。可以设置为NULL，表示从第一个子节点开始。
返回值： 找到的下一个子节点。
```

## 提取属性
查找节点中的指定属性
```sh
property *of_find_property(const struct device_node *np, const char *name, int *lenp)
np：设备节点。
name： 属性名字。
lenp：属性值的字节数，一般为NULL
返回值： 找到的属性。
```

获取属性中元素的数量
```sh
int of_property_count_elems_of_size(const struct device_node *np, const char *propname, int elem_size)
np：设备节点。
proname： 需要统计元素数量的属性名字。
elem_size：每个元素的长度。（如果元素为u32类型则此处填sizeof(u32)）
返回值： 得到的属性元素数量。
```

从属性中获取指定标号的 u32 类型数据值
```sh
int of_property_read_u32_index(const struct device_node *np, const char *propname, u32 index, u32 *out_value)
np：设备节点。
proname： 要读取的属性名字。
index：要读取的值标号。
out_value：读取到的值
返回值： 0 读取成功，负值，读取失败， -EINVAL 表示属性不存在， -ENODATA 表示没有要读取的数据， -EOVERFLOW 表示属性值列表太小。
```

读取属性中 u8、 u16、 u32 和 u64 类型的数组数据
```sh
int of_property_read_u8_array(const struct device_node *np, const char *propname, u8 *out_values, size_t sz)
int of_property_read_u16_array(const struct device_node *np, const char *propname, u16 *out_values, size_t sz)
int of_property_read_u32_array(const struct device_node *np, const char *propname, u32 *out_values, size_t sz)
int of_property_read_u64_array(const struct device_node *np, const char *propname, u64 *out_values, size_t sz)
np：设备节点。
proname： 要读取的属性名字。
out_values：读取到的数组值，分别为 u8、 u16、 u32 和 u64。
sz： 要读取的数组元素数量。
返回值： 0，读取成功，负值，读取失败， -EINVAL 表示属性不存在， -ENODATA 表示没有要读取的数据， -EOVERFLOW 表示属性值列表太小。
```

读取只有一个整形值的属性
```sh
int of_property_read_u8(const struct device_node *np,const char *propname, u8 *out_value)
int of_property_read_u16(const struct device_node *np, const char *propname, u16 *out_value)
int of_property_read_u32(const struct device_node *np, const char *propname, u32 *out_value)
int of_property_read_u64(const struct device_node *np, const char *propname, u64 *out_value)
np：设备节点。
proname： 要读取的属性名字。
out_value：读取到的数组值。
返回值： 0，读取成功，负值，读取失败， -EINVAL 表示属性不存在， -ENODATA 表示没有要读取的数据， -EOVERFLOW 表示属性值列表太小。
```

读取属性中字符串值
```sh
int of_property_read_string(struct device_node *np, const char *propname, const char **out_string)
np：设备节点。
proname： 要读取的属性名字。
out_string：读取到的字符串值。
返回值： 0，读取成功，负值，读取失败。
```

获取#address-cells 属性值
```sh
int of_n_addr_cells(struct device_node *np)
np：设备节点。
返回值： 获取到的#address-cells 属性值。
```

获取#size-cells 属性值
```sh
int of_n_size_cells(struct device_node *np)
np：设备节点。
返回值： 获取到的#size-cells 属性值。
```

## 其他常用函数
查看节点的 compatible 属性是否有包含指定的字符串
```sh
int of_device_is_compatible(const struct device_node *device, const char *compat)
device：设备节点。
compat：要查看的字符串。
返回值： 0，节点的 compatible 属性中不包含 compat 指定的字符串； 正数，节点的compatible属性中包含 compat 指定的字符串。
```

## 获取地址相关属性

主要是“reg”或者“assigned-addresses”属性值
```sh
const __be32 *of_get_address(struct device_node *dev, int index, u64 *size, unsigned int *flags)
dev：设备节点。
index：要读取的地址标号。
size：地址长度。
flags：参数，比如 IORESOURCE_IO、 IORESOURCE_MEM 等
返回值： 读取到的地址数据首地址，为 NULL 的话表示读取失败。
```

将从设备树读取到的地址转换为物理地址
```sh
u64 of_translate_address(struct device_node *dev, const __be32 *in_addr)
dev：设备节点。
in_addr：要转换的地址。
返回值： 得到的物理地址，如果为 OF_BAD_ADDR 的话表示转换失败。
```

## 从设备树里面提取资源值

本质上是将 reg 属性值转换为 resource 结构体类型
```sh
int of_address_to_resource(struct device_node *dev, int index, struct resource *r)
dev：设备节点。
index：地址资源标号。
r：得到的 resource 类型的资源值。
返回值： 0，成功；负值，失败。
```

## 直接内存映射（获取内存地址所对应的虚拟地址 ）

本质上是将 reg 属性中地址信息转换为虚拟地址（将原来的先提取属性在映射结合起来），如果 reg 属性有多段的话，可以通过 index 参数指定要完成内存映射的是哪一段
```sh
void __iomem *of_iomap(struct device_node *np, int index)
np：设备节点。
index： reg 属性中要完成内存映射的段，如果 reg 属性只有一段的话 index 就设置为0。（从0开始，一次映射一对，即一个地址一个长度）
返回值： 经过内存映射后的虚拟内存首地址，如果为 NULL 的话表示内存映射失败。
```

## GPIO子系统
## of函数
获取设备树某个属性中定义 GPIO 的个数（空的 GPIO 信息（即值为0）也会被统计到）
```sh
int of_gpio_named_count(struct device_node *np, const char *propname)
np：设备节点。
propname：要统计的 GPIO 属性。
返回值： 正值，统计到的 GPIO 数量；负值，失败。
```

获取设备树gpios属性中定义 GPIO 的个数（空的 GPIO 信息（即值为0）也会被统计到）
```sh
int of_gpio_count(struct device_node *np)
```

获取 GPIO 编号
```sh
int of_get_named_gpio(struct device_node *np, const char *propname, int index)
index： GPIO 索引，因为一个属性里面可能包含多个 GPIO，此参数指定要获取哪个 GPIO 的编号，如果只有一个 GPIO 信息的话此参数为 0
```

## 驱动层函数
申请GPIO
```sh
int gpio_request(unsigned gpio, const char *label)
gpio：要申请的 gpio 标号，使用 of_get_named_gpio 函数返回值
label：给 gpio 设置个名字。
返回值： 0，申请成功；其他值，申请失败。
```

释放GPIO
```sh
void gpio_free(unsigned gpio)
```

设置方向
```sh
int gpio_direction_input(unsigned gpio)
int gpio_direction_output(unsigned gpio, int value)
返回值： 0，设置成功；负值，设置失败
```

设置值
```sh
#define gpio_get_value __gpio_get_value
int __gpio_get_value(unsigned gpio)
返回值： 非负值，得到的 GPIO 值；负值，获取失败
```

获取值
```sh
#define gpio_set_value __gpio_set_value
void __gpio_set_value(unsigned gpio, int value)
```

获取 gpio 对应的中断号
```sh
int gpio_to_irq(unsigned int gpio)
gpio： 要获取的 GPIO 编号
返回值： GPIO 对应的中断号
```

## 中断相关
提取 interrupts 属性中的中断号
```sh
unsigned int irq_of_parse_and_map(struct device_node *dev, int index)
dev： 设备节点
index：索引号， interrupts 属性可能包含多条中断信息，通过 index 指定要获取的信息
返回值：中断号
```

获取 gpio 对应的中断号（与上面的函数功能一样）
```sh
int gpio_to_irq(unsigned int gpio)
gpio： 要获取的 GPIO 编号，由gpio_request申请而来
返回值： GPIO 对应的中断号
```

## 总结
以上汇总四篇，总结常用内核API接口，其中有较为详细的注解，经常看一看，对于熟练编写驱动，
有非常重要的作用，当然这不是让你去记这些API接口。而是经常看看熟悉它们。

