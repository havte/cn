---
title: 篇三：在Linux内核驱动中常用到API接口函数
layout: post
categories: Linux
tags: linux
excerpt: 在Linux内核中，常用驱动框架API、input输入API等
---

## 常用框架API
## platform
platform设备相关：
```sh
void platform_device_unregister(struct platform_device *pdev)//卸载platform设备
int platform_device_register(struct platform_device *pdev)//注册platform设备
```
platform驱动相关：
```sh
void platform_driver_unregister(struct platform_driver *drv)//卸载platform驱动
int platform_driver_register (struct platform_driver *driver)//注册platform驱动
```

## MISC
MISC 驱动也叫做杂项驱动，当某些外设无法进行分类的时候就可以使用 MISC 驱动。 MISC 驱动其实就是最简单的字符设备驱动，通常嵌套在 platform 总线驱动中，实现复杂的驱动

所有的 MISC 设备驱动的主设备号都为 10，不同的设备使用不同的从设备号。MISC 设备会自动创建 cdev
```sh
int misc_register(struct miscdevice * misc)//注册MISC驱动
int misc_deregister(struct miscdevice *misc)//销毁MISC驱动
```

## INPUT
初始化及卸载
申请及释放：
```sh
struct input_dev *input_allocate_device(void)//申请input_dev
void input_free_device(struct input_dev *dev)//释放input_dev
```
注册及注销：
```sh
int input_register_device(struct input_dev *dev)//注册
void input_unregister_device(struct input_dev *dev)//注销
```
修改input_dev：
```sh
struct input_dev *inputdev;
inputdev = input_allocate_device(); /* 申请 input_dev */
inputdev->name = "test_inputdev"; /* 设置 input_dev 名字 */

/*********第一种设置事件和事件值的方法***********/
__set_bit(EV_KEY, inputdev->evbit); /* 设置产生按键事件 */
__set_bit(EV_REP, inputdev->evbit); /* 重复事件 */
__set_bit(KEY_0, inputdev->keybit); /*设置产生哪些按键值 */
/************************************************/

/*********第二种设置事件和事件值的方法***********/
keyinputdev.inputdev->evbit[0] = BIT_MASK(EV_KEY) | BIT_MASK(EV_REP);
keyinputdev.inputdev->keybit[BIT_WORD(KEY_0)] |= BIT_MASK(KEY_0);
/************************************************/

/*********第三种设置事件和事件值的方法***********/
keyinputdev.inputdev->evbit[0] = BIT_MASK(EV_KEY) | BIT_MASK(EV_REP);
input_set_capability(keyinputdev.inputdev, EV_KEY, KEY_0);
/************************************************/

/* 注册 input_dev */
input_register_device(inputdev);
```

上报数据
上报指定的事件以对应值：
```sh
void input_event(struct input_dev *dev, unsigned int type, unsigned int code, int value)
```
dev：需要上报的 input_dev
type：上报的事件类型，比如 EV_KEY
code：事件码，即注册的按键值，比如 KEY_0、 KEY_1 等等
value：事件值，比如 1 表示按键按下， 0 表示按键松开

专用上报函数（本质是调用input_event）：
```sh
void input_report_key(struct input_dev *dev, unsigned int code, int value)//上报按键
void input_report_rel(struct input_dev *dev, unsigned int code, int value)
void input_report_abs(struct input_dev *dev, unsigned int code, int value)
void input_report_ff_status(struct input_dev *dev, unsigned int code, int value)
void input_report_switch(struct input_dev *dev, unsigned int code, int value)
void input_mt_sync(struct input_dev *dev)
```

同步事件：
```sh
void input_sync(struct input_dev *dev)
```

## 应用层相关
应用层通过获得 input_event 结构体来获取input子系统发送的输入事件
```sh
struct input_event {
    struct timeval time;
    __u16 type;
    __u16 code;
    __s32 value;
};
```
time：时间，即此事件发生的时间，timeval 结构体类型：
```sh
typedef long __kernel_long_t;
typedef __kernel_long_t __kernel_time_t;
typedef __kernel_long_t __kernel_suseconds_t;

struct timeval {
    __kernel_time_t tv_sec; /* 秒 */
    __kernel_suseconds_t tv_usec; /* 微秒 */
};
```
type： 事件类型，比如 EV_KEY 表示此次事件为按键事件
code： 事件码，比如在 EV_KEY 事件中 code 就表示具体的按键码，如： KEY_0、KEY_1等
value：值，比如 EV_KEY 事件中 value 就是按键值，为 1 表示按键按下，为 0 的话表示按键没有被按下

应用层读取方式：
```sh
static struct input_event inputevent;
int err = 0;
err = read(fd, &inputevent, sizeof(inputevent));
if (err > 0) { /* 读取数据成功 */
    switch (inputevent.type) {
        case EV_KEY:
            if (inputevent.code < BTN_MISC) { /* 键盘键值 */
                printf("key %d %s\r\n", inputevent.code, inputevent.value ? "press" : "release");
            } else {
                printf("button %d %s\r\n", inputevent.code, inputevent.value ? "press" : "release");
            }
            break;
            /* 其他类型的事件，自行处理 */
        case EV_REL:
            break;
        case EV_ABS:
            break;
        case EV_MSC:
            break;
        case EV_SW:
            break;
    }
} else {
    printf("读取数据失败\r\n");
}
```

## 多点触摸
linux内核中讲解多点电容触摸屏协议文档路径：Documentation/input/multitouch-protocol.txt
老版本（2.x 版本）的 linux内核不支持多点电容触摸(Multi-touch，简称 MT)

MT 协议分为两种类型：
Type A：适用于触摸点不能被区分或者追踪，此类型的设备上报原始数据(此类型在实际使用中非常少)
Type B：适用于有硬件追踪并能区分触摸点的触摸设备，此类型设备通过 slot 更新某一个触摸点的信息，一般的多点电容触摸屏 IC 都有此能力

## Type A
步骤（时序）如下：
```sh
ABS_MT_POSITION_X x[0]//上报第一个点的x坐标 input_report_abs()
ABS_MT_POSITION_Y y[0]//上报第一个点的y坐标
SYN_MT_REPORT// input_mt_sync()

ABS_MT_POSITION_X x[1]//上报第二个点的x坐标 input_report_abs()
ABS_MT_POSITION_Y y[1]//上报第二个点的y坐标
SYN_MT_REPORT// input_mt_sync()

SYN_REPORT// input_sync() 该轮数据发送完毕
```
例子：drivers/input/touchscreen/st1232.c

## Type B
步骤（时序）如下：
```sh
ABS_MT_SLOT 0// input_mt_slot()
ABS_MT_TRACKING_ID 45// input_mt_report_slot_state()
ABS_MT_POSITION_X x[0]//上报第一个点的x坐标 input_report_abs()
ABS_MT_POSITION_Y y[0]//上报第一个点的y坐标

ABS_MT_SLOT 1// input_mt_slot()
ABS_MT_TRACKING_ID 46// input_mt_report_slot_state()
ABS_MT_POSITION_X x[1]//上报第二个点的x坐标 input_report_abs()
ABS_MT_POSITION_Y y[1]//上报第二个点的y坐标

SYN_REPORT// input_sync() 该轮数据发送完毕
```
例子：drivers/input/touchscreen/ili210x.c

相关函数：

初始化 MT 的输入 slots（初始化时使用）：
```sh
int input_mt_init_slots(struct input_dev *dev, unsigned int num_slots, unsigned int flags)
```
dev： MT 设备对应的 input_dev，因为 MT 设备隶属于 input_dev
num_slots：设备要使用的 SLOT 数量，也就是触摸点的数量
flags： 其他一些 flags 信息
```sh
#define INPUT_MT_POINTER 0x0001 /* pointer device, e.g. trackpad */
#define INPUT_MT_DIRECT 0x0002 /* direct device, e.g. touchscreen */
#define INPUT_MT_DROP_UNUSED0x0004 /* drop contacts not seen in frame */
#define INPUT_MT_TRACK 0x0008 /* use in-kernel tracking */
#define INPUT_MT_SEMI_MT 0x0010 /* semi-mt device, finger count handled manually */
```
返回值： 0，成功；负值，失败

产生 ABS_MT_SLOT 事件：
```sh
static inline void input_mt_slot(struct input_dev *dev, int slot)
```
dev： MT 设备对应的 input_dev
slot：当前发送的是哪个 slot 的坐标信息，也就是哪个触摸点产生ABS_MT_TRACKING_ID和ABS_MT_TOOL_TYPE事件：
```sh
void input_mt_report_slot_state(struct input_dev *dev, unsigned int tool_type, bool active)
```
dev： MT 设备对应的 input_dev
tool_type：上报触摸工具类型（即ABS_MT_TOOL_TYPE事件），目前的协议支持MT_TOOL_FINGER(手指)、 MT_TOOL_PEN(笔)和 MT_TOOL_PALM(手掌)
active：true，连续触摸，添加一个新的触摸点，linux 内核自动分配一个 ABS_MT_TRACKING_ID；false，触摸点抬起，移除一个触摸点，ABS_MT_TRACKING_ID由内核设为-1上传真实的触摸点数量：
```sh
void input_mt_report_pointer_emulation(struct input_dev *dev, bool use_count)
```
dev： MT 设备对应的 input_dev
use_count：true，有效的触摸点数量（上报数量就是当前数模点数量）； false，追踪到的触摸点数量多于当前上报的数量（使用 BTN_TOOL_TAP 事件通知用户空间当前追踪到的触摸点总数量）

举个例子：硬件能够追踪5个触摸点，无论是否有触摸，硬件都会有5个值输出，此时use_count就是false，即无论触摸的数量为多少，追踪到的（硬件输出的5个值）总比上报的（真实触摸数量）多
例子：
```sh
static irqreturn_t ft5x06_handler(int irq, void *dev_id)
{
	......
	/* 读取FT5X06触摸点坐标从0X02寄存器开始，连续读取29个寄存器 */
	ret = ft5x06_read_regs(multidata, FT5X06_TD_STATUS_REG, rdbuf, FT5X06_READLEN);
    
	/* 上报每一个触摸点坐标 */
	for (i = 0; i < MAX_SUPPORT_POINTS; i++) {
		u8 *buf = &rdbuf[i * tplen + offset];

		/* 以第一个触摸点为例，寄存器TOUCH1_XH(地址0X03),各位描述如下：
		 * bit7:6  Event flag  0:按下 1:释放 2：接触 3：没有事件
		 * bit5:4  保留
		 * bit3:0  X轴触摸点的11~8位。
		 */
		type = buf[0] >> 6;     /* 获取触摸类型 */
		if (type == TOUCH_EVENT_RESERVED)
			continue;
 
		/* 我们所使用的触摸屏和FT5X06是反过来的 */
		x = ((buf[2] << 8) | buf[3]) & 0x0fff;
		y = ((buf[0] << 8) | buf[1]) & 0x0fff;
		
		/* 以第一个触摸点为例，寄存器TOUCH1_YH(地址0X05),各位描述如下：
		 * bit7:4  Touch ID  触摸ID，表示是哪个触摸点
		 * bit3:0  Y轴触摸点的11~8位。
		 */
		id = (buf[2] >> 4) & 0x0f;
		down = type != TOUCH_EVENT_UP;//是否按下 1：按下 0：松开

		input_mt_slot(multidata->input, id);
		input_mt_report_slot_state(multidata->input, MT_TOOL_FINGER, down);

		if (!down)
			continue;

		input_report_abs(multidata->input, ABS_MT_POSITION_X, x);
		input_report_abs(multidata->input, ABS_MT_POSITION_Y, y);
	}

	input_mt_report_pointer_emulation(multidata->input, true);
	input_sync(multidata->input);
    ......
}
```

## Framebuffer
fb_info结构体：
```sh
struct fb_info *framebuffer_alloc(size_t size, struct device *dev)
```
void framebuffer_release(struct fb_info *info)
size：分配完framebuffer结构体后附加的额外空间（一般用于存放用户私有数据）
dev：最终会绑定到fb_info->device上，可以设为NULL

注册与卸载：
```sh
int register_framebuffer(struct fb_info *fb_info)
int unregister_framebuffer(struct fb_info *fb_info)
```
显存分配与释放：
```sh
static inline void *dma_alloc_writecombine(struct device *dev, size_t size, dma_addr_t *dma_addr, gfp_t gfp)
static inline void dma_free_writecombine(struct device *dev, size_t size, void *cpu_addr, dma_addr_t dma_addr)
```

## RTC
申请并注册rtc_device：
```sh
struct rtc_device *rtc_device_register(const char *name, struct device *dev, const struct rtc_class_ops *ops, struct module *owner)
```
name：设备名字
dev： 设备
ops： RTC 底层驱动函数集
owner：驱动模块拥有者

注销rtc_device：
```sh
void rtc_device_unregister(struct rtc_device *rtc)
```

## IIC
适配器注册和注销：
一般不会用到，SOC厂商会写好这部分代码

//注册
```sh
int i2c_add_adapter(struct i2c_adapter *adapter)/* 使用动态总线号 */
int i2c_add_numbered_adapter(struct i2c_adapter *adap)/* 使用静态总线号 */
```
//注销
```sh
void i2c_del_adapter(struct i2c_adapter * adap)
```

i2c驱动注册与注销：
```sh
#define i2c_add_driver(driver) i2c_register_driver(THIS_MODULE, driver)
void i2c_del_driver(struct i2c_driver *driver)
```
## iic通信
内核驱动
内核文档：Documentation\i2c\i2c-protocol、Documentation\i2c\smbus-protocol

其中smbus-protocol是i2c-protocol的一个子集，官方更加推荐使用后者的smbus-protoco中的函数

收发函数：
```sh
//S Addr Rd [A] [Data] NA S Addr Wr [A] Data [A] P
int i2c_transfer(struct i2c_adapter *adap, struct i2c_msg *msgs, int num)
```
adap： 所使用的 I2C 适配器， i2c_client 会保存其对应的 i2c_adapter
msgs： I2C 要发送的一个或多个消息
```sh
struct i2c_msg {
    __u16 addr; /* 从机地址 */
    __u16 flags; /* 标志 */
    #define I2C_M_TEN 0x0010
    #define I2C_M_RD 0x0001
    #define I2C_M_STOP 0x8000
    #define I2C_M_NOSTART 0x4000
    #define I2C_M_REV_DIR_ADDR 0x2000
    #define I2C_M_IGNORE_NAK 0x1000
    #define I2C_M_NO_RD_ACK 0x0800
    #define I2C_M_RECV_LEN 0x0400
    __u16 len; /* 消息(本 msg)长度 */
    __u8 *buf; /* 消息数据 */
};
```
num： 消息数量，即 msgs 的数量
返回值： 负值，失败，其他非负值，发送的 msgs 数量

发送函数（最终调用i2c_transfer）：
```sh
//S Addr Wr [A] Data [A] Data [A] ... [A] Data [A] P
int i2c_master_send(const struct i2c_client *client, const char *buf, int count)
```
client： I2C 设备对应的 i2c_client
buf：要发送的数据
count： 要发送的数据字节数，必须小于 64KB（i2c_msg 的 len 成员变量是一个 u16(无符号 16 位)类型的数据）
返回值： 负值，失败，其他非负值，发送的字节数

接收函数（最终调用i2c_transfer）：
```sh
//S Addr Rd [A] [Data] A [Data] A ... A [Data] NA P
int i2c_master_recv(const struct i2c_client *client, char *buf, int count)
```
client： I2C 设备对应的 i2c_client
buf：要接收的数据
count： 要接收的数据字节数，必须小于 64KB（i2c_msg 的 len 成员变量是一个 u16(无符号 16 位)类型的数据）
返回值： 负值，失败，其他非负值，发送的字节数

smbus-protoco中的函数：
```sh
i2c_smbus_read_byte()//S Addr Rd [A] [Data] NA P
i2c_smbus_write_byte()//S Addr Wr [A] Data [A] P
i2c_smbus_read_byte_data()//S Addr Wr [A] Comm [A] S Addr Rd [A] [Data] NA P
i2c_smbus_read_word_data()//S Addr Wr [A] Comm [A] S Addr Rd [A] [DataLow] A [DataHigh] NA P
i2c_smbus_write_byte_data()//S Addr Wr [A] Comm [A] Data [A] P
i2c_smbus_write_word_data()//S Addr Wr [A] Comm [A] DataLow [A] DataHigh [A] P
i2c_smbus_read_block_data()//S Addr Wr [A] Comm [A] S Addr Rd [A] [Count] A [Data] A [Data] A ... A [Data] NA P
i2c_smbus_write_block_data()//S Addr Wr [A] Comm [A] Count [A] Data [A] Data [A] ... [A] Data [A] P
i2c_smbus_read_i2c_block_data()//S Addr Wr [A] Comm [A] S Addr Rd [A] [Data] A [Data] A ... A [Data] NA P
i2c_smbus_write_i2c_block_data()//S Addr Wr [A] Comm [A] Data [A] Data [A] ... [A] Data [A] P
```

## 应用层直接访问
内核文档：Documentation\i2c\dev-interface
通常，I2C设备由设备驱动来控制，但是通过/dev接口也可以提供用户空间直接访问适配器上的设备。前提是需要加载I2C-DEV内核模块

在应用层，i2c-tools工具包帮你写好了接口，下载链接，镜像仓库，只需要包含该库的头文件include\linux\i2c-dev.h即可直接控制iic（最新版本的4.1没有发现该文件）。其本质是调用的内核自带的驱动模块I2C-DEV。该模块位于\drivers\i2c\i2c-dev.c，通过宏CONFIG_I2C_CHARDEV配置，内核menuconfig路径为Device Drivers-> I2C support，一般默认为通过模块加载

i2c-tools工具包的本质是通过调用ioctl打开实现各种功能：
//include\linux\i2c-dev.h部分源码
```sh
static inline __s32 i2c_smbus_access(int file, char read_write, __u8 command, int size, union i2c_smbus_data *data)
{
    struct i2c_smbus_ioctl_data args;

    args.read_write = read_write;
    args.command = command;
    args.size = size;
    args.data = data;
    return ioctl(file,I2C_SMBUS,&args);//本质就是调用ioctl
}


static inline __s32 i2c_smbus_write_quick(int file, __u8 value)
{
    return i2c_smbus_access(file,value,0,I2C_SMBUS_QUICK,NULL);
}
```
上述部分源码中file就是需要访问的iic控制器，如：/dev/i2c-0，与自己写的驱动程序类似

模板
```sh
/* 设备结构体 */
struct xxx_dev {
    ......
        void *private_data; /* 私有数据，一般会设置为 i2c_client */
};

/*
* @description : 读取 I2C 设备多个寄存器数据
* @param – dev : I2C 设备
* @param – reg : 要读取的寄存器首地址
* @param – val : 读取到的数据
* @param – len : 要读取的数据长度
* @return : 操作结果
*/
static int xxx_read_regs(struct xxx_dev *dev, u8 reg, void *val, int len)
{
    int ret;
    struct i2c_msg msg[2];
    struct i2c_client *client = (struct i2c_client *)dev->private_data;

    /* msg[0]，第一条写消息，发送要读取的寄存器首地址 */
    msg[0].addr = client->addr; /* I2C 器件地址 */
    msg[0].flags = 0; /* 标记为发送数据 */
    msg[0].buf = &reg; /* 读取的首地址 */
    msg[0].len = 1; /* reg 长度 */

    /* msg[1]，第二条读消息，读取寄存器数据 */
    msg[1].addr = client->addr; /* I2C 器件地址 */
    msg[1].flags = I2C_M_RD; /* 标记为读取数据 */
    msg[1].buf = val; /* 读取数据缓冲区 */
    msg[1].len = len; /* 要读取的数据长度 */
    ret = i2c_transfer(client->adapter, msg, 2);
    if(ret == 2) {
        ret = 0;
    } else {
        ret = -EREMOTEIO;
    }
    return ret;
}

/*
* @description : 向 I2C 设备多个寄存器写入数据
* @param – dev : 要写入的设备结构体
* @param – reg : 要写入的寄存器首地址
* @param – val : 要写入的数据缓冲区
* @param – len : 要写入的数据长度
* @return : 操作结果
*/
static s32 xxx_write_regs(struct xxx_dev *dev, u8 reg, u8 *buf, u8 len)
{
    u8 b[256];
    struct i2c_msg msg;
    struct i2c_client *client = (struct i2c_client *)
        dev->private_data;

    b[0] = reg; /* 寄存器首地址 */
    memcpy(&b[1],buf,len); /* 将要发送的数据拷贝到数组 b 里面 */

    msg.addr = client->addr; /* I2C 器件地址 */
    msg.flags = 0; /* 标记为写数据 */

    msg.buf = b; /* 要发送的数据缓冲区 */
    msg.len = len + 1; /* 要发送的数据长度 */

    return i2c_transfer(client->adapter, &msg, 1);
}
```

## SPI
spi_master注册和注销：
一般不会用到，SOC厂商会写好这部分代码

//注册
```sh
struct spi_master *spi_alloc_master(struct device *dev,unsigned size)//申请
int spi_register_master(struct spi_master *master)//注册 spi_bitbang_start
```
//注销
```sh
void spi_master_put(struct spi_master *master)//释放
void spi_unregister_master(struct spi_master *master)//注销 spi_bitbang_stop
```
spi驱动注册与注销：
```sh
int spi_register_driver(struct spi_driver *sdrv);
void spi_unregister_driver(struct spi_driver *sdrv)
```

相关结构体：
```sh
/*-----------------------spi_message-----------------------*/
struct spi_message {
    struct list_head    transfers;
    struct spi_device   *spi;
    unsigned        is_dma_mapped:1;
    /* REVISIT:  we might want a flag affecting the behavior of the
     * last transfer ... allowing things like "read 16 bit length L"
     * immediately followed by "read L bytes".  Basically imposing
     * a specific message scheduling algorithm.
     *
     * Some controller drivers (message-at-a-time queue processing)
     * could provide that as their default scheduling algorithm.  But
     * others (with multi-message pipelines) could need a flag to
     * tell them about such special cases.
     */
    /* completion is reported through a callback */
    void            (*complete)(void *context);/*异步传输完成后，会调用该函数*/
    void            *context;
    unsigned        frame_length;
    unsigned        actual_length;
    int         status;
    /* for optional use by whatever driver currently owns the
     * spi_message ...  between calls to spi_async and then later
     * complete(), that's the spi_master controller driver.
     */
    struct list_head    queue;
    void            *state;
};
/*-----------------------spi_transfer-----------------------*/
struct spi_transfer {
    /* it's ok if tx_buf == rx_buf (right?)
     * for MicroWire, one buffer must be null
     * buffers must work with dma_*map_single() calls, unless
     *   spi_message.is_dma_mapped reports a pre-existing mapping
     */
    const void  *tx_buf;/* 要发送的数据 */
    void        *rx_buf;/* 保存接收到的数据 */
    unsigned    len;/* 进行传输的数据长度 */
    dma_addr_t  tx_dma;
    dma_addr_t  rx_dma;
    struct sg_table tx_sg;
    struct sg_table rx_sg;
    unsigned    cs_change:1;
    unsigned    tx_nbits:3;
    unsigned    rx_nbits:3;
#define SPI_NBITS_SINGLE    0x01 /* 1bit transfer */
#define SPI_NBITS_DUAL      0x02 /* 2bits transfer */
#define SPI_NBITS_QUAD      0x04 /* 4bits transfer */
    u8      bits_per_word;
    u16     delay_usecs;
    u32     speed_hz;
    struct list_head transfer_list;
};
```

## spi通信
初始化：
```sh
int spi_setup(struct spi_device *spi)//初始化时钟和SPI模式
void spi_message_init(struct spi_message *m)
void spi_message_add_tail(struct spi_transfer *t, struct spi_message *m)//将spi_transfer添加到spi_message队列中
```
同步传输（阻塞，会等待SPI数据传输完成）
```sh
int spi_sync(struct spi_device *spi, struct spi_message *message)
```
异步传输（不会阻塞，不会等到SPI数据传输完成）

异步传输需要设置 spi_message 中的complete成员变量，当 SPI 异步传输完成以后complete函数就会被调用
```sh
int spi_async(struct spi_device *spi, struct spi_message *message)
```
模板
```sh
/* SPI 多字节发送 */
static int spi_send(struct spi_device *spi, u8 *buf, int len)
{
    int ret;
    struct spi_message m;
    struct spi_transfer t = {
        .tx_buf = buf,
        .len = len,
    };
    spi_message_init(&m); /* 初始化 spi_message */
    spi_message_add_tail(t, &m);/* 将 spi_transfer 添加到 spi_message 队列 */
    ret = spi_sync(spi, &m); /* 同步传输 */
    return ret;
}
/* SPI 多字节接收 */
static int spi_receive(struct spi_device *spi, u8 *buf, int len)
{
    int ret;
    struct spi_message m;
    struct spi_transfer t = {
        .rx_buf = buf,
        .len = len,
    };
    spi_message_init(&m); /* 初始化 spi_message */
    spi_message_add_tail(t, &m);/* 将 spi_transfer 添加到 spi_message 队列 */
    ret = spi_sync(spi, &m); /* 同步传输 */
    return ret;
}
```
