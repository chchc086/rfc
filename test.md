# Hypercall微库设计

## 背景

mortise当前版本不支持hypercall调用功能，一些场景下比如请求创建虚机/销毁虚机是需要通过hypercall来支持，为了解决这些问题，本文档提出了hypercall微库相关的方案设计。

### 目标

实现hypercall微库模块，该微库有如下主要特性：

- 提供注册接口，支持注册ARM [SMC Calling Convention](https://developer.arm.com/documentation/den0028/f/?lang=en)规范中定义的各种服务调用, hypercall微库不止支持注册SMCCC标准定义的一些标准服务, 还需要支持mortise内部模块和外部模块自定义函数的注册.
- 可扩展的注册方式, 各个外部模块可以在无需修改mortise源码的情况下注册自己实现的函数
- 注册功能的灵活性, 为了提高注册的灵活性，hypercall微库需要实现2种不同的注册方式:
  - 一次注册单个fid，用户使用这种注册方式只会注册一个单独的fid, handler处理函数只能通过该fid被触发调用
  - 一次注册批量fid，使用这种方式可以一次性注册用户指定范围内批量连续的fid, 这样就可以让多个不同的fid通过smc/hvc指令都路由到同一个handler处理函数。
- 根据smc/hvc指令传递的fid路由到相应的handelr，需要考虑路由时的时间开销与空间开销。

### 非目标

- mortise的各个微库模块包括外部第三方模块在使用hypercall微库的接口注册handler时，分配或预留给它们使用的fid范围并不由hypercall微库管理

## 方案提议

1. hypercall微库提供注册hypercall的宏接口，mortise内部模块或者外部的模块想要guest可以通过smc/hvc指令请求调用到自己实现的hypercall函数，需要通过宏接口注册自己实现的hypercall handler，注册分2种：
   1. 为handler分配一个单独的fid
   2. 为handler分配一组连续的fid

2. hypercall微库提供smc/hvc指令路由的接口，guest调用smc/hvc指令并传递一个fid时，会陷入mortise的异常处理流程，mortise处理smc/hvc指令产生的异常时通过hypercall微库提供的接口路由到传入fid对应的handler进行调用。

3. mortise在启动阶段进行hypercall微库相关数据结构的初始化，为后续smc/hvc的路由做准备。

### 使用场景和案例

#### 场景案例1: mortise实现psci相关的接口并注册为组hypercall，供guest启动时调用

hypercall微库提供的注册接口可以一次性注册一组fid，只要smc/hvc指令传递的fid处于注册范围内，均会路由到同一个处理函数。尽管psci相关调用有多个，注册时可以将它们注册为一组hypercall处理函数，省去了每添加一个相关调用就注册一次的麻烦。

#### 场景案例2: mortise实现虚机管理的微库，注册相关hypercall调用

实现虚机管理相关功能，并注册为hypercall，分配相应的fid。虚机启动之后，管理虚拟机(MVM)可提供shell命令行来管理虚机(创建/销毁)：执行相关命令时MVM通过smc/hvc指令路由到管理虚机的hypercall处理函数。

#### 场景案例3：外部第三方模块使用接口注册一组fid，调用自己模块内部实现的hypercall函数

外部第三方模块可以使用Hypercall微库提供的接口注册一组fid，并实现自己的Hypercall函数。通过这种方式，模块可以独立实现并管理其Hypercall功能，而无需修改Mortise的核心代码。

#### 约束限制说明

1. 注册hypercall handler时，如果为其分配的是单个fid，该fid需要满足:
   - 根据SMCCC规范描述，fid是一个32位的整数，其中比特位[31]表示该调用的类型:
     1. Fast Call
     2. Yield Call
目前hypercall微库只支持fast call类型，注册的fid[31]比特位必须为1。
   - 根据SMCCC规范描述, 对于Fast Call类型，fid的[23:16]比特位必须是0
   - 注册单个的fid值不能与已经注册过的单个fid或者组fid中的任何值重复
2. 注册hypercall handler时，如果为其分配的是一组连续的fid，注册时需要设置这一组fid的最小值base_fid，以及一个掩码fid_mask, base_fid与fid_maks需要满足：
   - base_fid与注册单个fid时的要约束相同，最高比特[31]必须为1，[23:16]比特位必须是0。
   - fid_mask表示注册的一组连续fid的掩码，不能和base_fid在相同比特位上同为1。
   - 注册的组fid中的任何值不能与已经注册过的单个fid或者组fid中的任何值重复

3. 注册的hypercall函数原型，不能超过所配置的最多参数个数以及返回的结果数量。
4. 可配置项：参数数量，返回结果数量，目前没有针对arm32位架构做区分

有关fid各个字段的描述：[SMCCC](https://developer.arm.com/documentation/den0028/f/?lang=en):Table2-1

#### 风险点和缓解措施

- 在各个模块中注册的fid，如果出现彼此冲突的情况，mortise在启动时会判断出这种冲突的存在，导致mortise启动失败。hypercall微库在初始化数据结构过程中将所有注册fid及其对应掩码(对于注册的单个fid，掩码为0)按照fid值有序存储到数组中，hypercall微库在判断是否有重复注册时, 将有序排列的单个fid值与组fid均视为组fid类型(单个fid可视为左右边界相同的组fid范围)。一旦存在一组的fid左边界值与上一个组fid范围发生重叠, 视为注册冲突发生，对于冲突的两组fid，mortise会分别将两组范围左边界fid值输出到报错信息中，通过该信息可以判断出冲突产生的fid范围。
- 如果注册的hypercall函数原型与规定类型不一致，会导致smc/hvc指令路由时运行崩溃。通过在编译期静态校验函数原型的方式，确保注册的hypercall函数原型符合要求。
note: 在支持GNU C11标准及以上的编译器环境下，静态校验功能才生效。

## 详细设计

### 功能设计

#### 1. 注册fid的格式要求

注册fid的字段:

| Bits  | 含义                                                                                  |
| ----- | ------------------------------------------------------------------------------------- |
| 31    | 必须设置为1,表示该hypercall调用时Fast Call类型                                        |
| 30    | 如果是0, 表示使用的是SMC32/HVC32调用规范 <br> 如果是1, 表示使用的是SMC64/HVC64调用规范 |
| 29:24 | 这6个比特用于区分不同的<a href="#table1" target="_self">服务的类型</a>                                               |
| 23:16 | 该字段注册时必须设置为0                                                               |
| 15:0  | 标识[29:24]字段决定的服务类型内部的具体函数                                           |

注册hypercall服务以及为其分配单个fid值或者组fid时, 需要遵从[SMCCC规范已规划的服务种类和相应的fid范围](#table2)

#### 2. API设计

##### 注册单个fid

```c
REGISTER_HYPERCALLX(<handler function pointer>, <handler fid>)
```

注册单个`<handler fid`，并将`<handler fid>`与`<handler function pointer>`进行关联。`REGISTER_HYPERCALLX`宏用于注册接收`X`个参数的handler函数

- `<handler function pointer>`: 注册的hypercall handler函数指针, 函数原型要求返回类型void, 参数类型为unsigned long, 最后一个参数为hypercall_res结构体指针,该结构体存储handler处理的返回结果.

| 注册宏                        | handler函数原型                                                                                 |
| ----------------------------- | ----------------------------------------------------------------------------------------------- |
| `REGISTER_HYPERCALL0`         | `void handler(struct hypercall_res* res)`                                                       |
| `REGISTER_HYPERCALL1`         | `void handler(unsigned long x1, struct hypercall_res* res)`                                     |
| `REGISTER_HYPERCALL2`         | `void handler(unsigned long x1, unsigned long x2, struct hypercall_res* res)`                   |
| `REGISTER_HYPERCALL3`         | `void handler(unsigned long x1, unsigned long x2, unsigned long x3, struct hypercall_res* res)` |
| <div align="center">...</div> | <div align="center">...</div>                                                                   |

- `<handler fid>`: 与`<handler funciton type>`关联的fid值

##### 注册组fid

```c
REGISTER_HYPERCALLX_GROUP(<handler function pointer>, <group base fid>, <group fid mask>)
```

`REGISTER_HYPERCALLX_GROUP`用于批量注册fid，注册的fid范围由`<group base fid>`和`<group fid mask>`决定, 并将该范围的所有fid值与`<handler function pointer>`进行关联。`REGISTER_HYPERCALLX_GROUP`表示该宏用于注册接收X个参数的handler函数.

- `<handler function pointer>`: 注册的hypercall handler函数指针, 函数原型要求返回类型void, 参数类型为unsigned long, 特殊的是第一个参数,类型为unsigned long, 其值为接收到的组fid范围内的具体某个fid值, 最后一个参数为hypercall_res结构体指针, 该结构体存储handler处理的返回结果.
- `<group base fid>`: 注册的组fid范围中最小的fid值(fid范围区间的左边界)
- `<group fid mask>`: 注册的组fid对应的掩码

| 注册宏                        | handler函数原型                                                                                                  |
| ----------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| REGISTER_HYPERCALL0_GROUP     | void handler(unsigned long fid, struct hypercall_res* res)                                                       |
| REGISTER_HYPERCALL1_GROUP     | void handler(unsigned long fid, unsigned long x1, struct hypercall_res* res)                                     |
| REGISTER_HYPERCALL2_GROUP     | void handler(unsigned long fid, unsigned long x1, unsigned long x2, struct hypercall_res* res)                   |
| REGISTER_HYPERCALL3_GROUP     | void handler(unsigned long fid, unsigned long x1, unsigned long x2, unsigned long x3, struct hypercall_res* res) |
| <div align="center">...</div> | <div align="center">...</div>                                                                                    |

##### hypercall初始化函数接口

hypercall初始化:`void hypercall_init(void)`
初始化hypercall微库相关的数据结构，确保smc/hvc指令路由时的性能及空间开销, 该函数通过遍历存储handler相关信息的段，将所有handler相关信息组织到有序数组中。

##### smc/hvc指令路由函数接口

smc/hvc指令路由处理：`long int hypercall_route(unsigned long args[MAX_HYPERCALL_FUNC_ARGC + 1], struct hypercall_res *res)`
将smc/hvc指令传递的fid以及参数存储到一个数组中, 作为参数传给该函数接口, 该函数根据fid执行路由逻辑.

`args`: 存储smc/hvc指令传递的参数，除了传递进来的fid，参数数量不能超过所配置的最大个数`MAX_HYPERCALL_FUNC_ARGC`, 传递进来的fid存放在`args[0]`中.

`res`: 指向返回结果结构体的指针, 存储返回结果的结构体中存储smc/hvc调用的结果以及返回结果个数:

```c
typedef struct hypercall_res {
 /* store result retured by hypercall handler */
 unsigned long x[MAX_RETURN_NUM];

 /* actual result count retured by hypercall handler */
 uint8_t res_cnt;

} hypercall_res;
```

`返回值`: 返回值为0表示传递进来的`fid(args[0])`值成功路由到了对应的handler函数, 非0表示传递进来的fid没有注册过, 结果为`UNKNOWN_FUNC_ID`.

**notes: hypercall_route函数接口如果路由失败,并不会填充res中的内容, 因此在调用该函数路由之前, 定义的hypercall_res结构体被传入`hypercall_route`之前需要将res_cnt字段初始化为1**

##### 注册宏接口设计

- 注册单个fid

```c
//X的可能取值0~5
#define REGISTER_HYPERCALLX(handler, func_id)                                  \
 REGISTER_HYPERCALL(handler, func_id, X)


#define REGISTER_HYPERCALL(handler, func_id, argc)                             \
 CHECK_FUNC_PROT(handler, argc);                                        \
 CHECK_FUNC_ARGC(argc);                                                 \
 CHECK_FUNC_ID(func_id);                                                \
 __attribute__((section(".mt_hypercall_handlers"),                      \
         used)) static struct hypercall_desc                     \
     hypercall_handler_##handler = {                                    \
      .func = handler,                                           \
      .base_fid = func_id,                                       \
      .fid_mask = 0,                                             \
      .arg_count = argc,                                         \
     }
```

`CHECK_FUNC_PROT(handler, argc)`: 编译期校验注册的`handler`函数原型.

`CHECK_FUNC_ARGC(argc)`: 编译期校验注册的handler函数参数个数`argc`是否超过配置的最大值限制.

`CHECK_FUNC_ID(func_id)`: 编译期校验注册的`func_id`是否符合设计要求.

使用宏注册时会创建hypercall_desc结构体实例，通过attribute关键字指定将其放置在名为.mt_hypercall_handlers特定的段中, 结构体字段描述如下:

`func`: 注册的handler函数指针

`base_fid`: 注册的单个fid值

`fid_mask`: 对于注册单个fid的情况, 掩码统一为0

`arg_count`: handler接受的参数个数

- 注册组fid

```c
//X的可能取值0~5
#define REGISTER_HYPERCALLX_GROUP(handler, func_id)                                  \
 REGISTER_HYPERCALL_GROUP(handler, func_id, X+1)


#define REGISTER_HYPERCALL_GROUP(handler, grp_base_fid, grp_fid_mask, argc)    \
 CHECK_FUNC_PROT(handler, argc);                                        \
 CHECK_FUNC_ARGC(argc - 1);                                             \
 CHECK_GRP_ID(grp_base_fid, grp_fid_mask);                              \
 __attribute__((section(".mt_hypercall_handlers"),                      \
         used)) static struct hypercall_desc                     \
     hypercall_handler_##handler = {                                    \
      .func = handler,                                           \
      .base_fid = grp_base_fid,                                  \
      .fid_mask = grp_fid_mask,                                  \
      .arg_count = argc,                                         \
     }
```

使用宏注册组fid时与注册单个fid一样也会创建hypercall_desc结构体实例，
`base_fid`: 注册的单个fid值
唯一不同的是`fid_mask`字段, 将其设置为注册时传入的组fid掩码.

#### 3. hypercall的初始化与路由

不论是注册的单个fid的hypercall_desc还是组fid的hypercall_desc, 都会根据掩码将其处理为一个fid范围, 特殊之处在于针对注册单个fid的hypercall_desc, 其fid_mask设置为0, 后续处理时便可以将单个fid视为只包含fid的长度为1的范围区间.

`示例代码`:

```c
REGISTER_HYPERCALLX_GROUP(group_handler1, 0xC8000000, 0x000000FF) 

REGISTER_HYPERCALLX(handler1, 0x86000001) 

REGISTER_HYPERCALLX_GROUP(group_handler2, 0x89000000, 0x4000001F) 
```

处理流程:
![alt text](images/image.png)

### 测试设计

#### 单元测试

notes: mortise目前版本还不支持单元测试功能

#### 集成测试

- 验证编译时校验功能, 支持编译期校验以下3点:
  - hanlder函数原型
  - handler参数个数
  - 注册的fid

针对上述校验类型, 使用注册宏分别注册函数原型, 参数个数, fid字段不符合规定的handler处理函数,
预期在编译阶段校验出错误.

- 验证smc/hvc指令路由
    1. TenonOS端使用smc/hvc指令,传递预先注册好的fid请求相应服务
        - 预先注册组fid, smc/hvc指令使用组范围内的任意fid值, 均路由到注册的同一handler
        - 预先注册单个fid, smc/hvc指令使用该fid, 路由到注册的handler
    2. TenonOS端使用smc/hvc指令,传递没有注册过的fid, 判断返回的结果是否是UNKNOWN_FUNC_ID错误码(-1).

## 缺点

- mortise初始化时可能会因为注册的fid出现冲突而崩溃，不支持静态验证。
- mortise的hypercall微库通过有序数组组织hanlder相关数据，并通过二分定位fid关联的handler处理函数。
在smc/hvc指令路由时，二分相较于switch case的路由方式会存在一些性能损耗。

## 备选方案

- 数据结构优化: 考虑使用哈希表或基数树来存储fid和handler的映射关系，该方案较原方案空间开销更大，但是有着更优秀的路由转发时间复杂度.

## 附

<a id="table1">SMCC</a>

| fid [29:24]字段值 |                 服务类型                 |
| :---------------: | :--------------------------------------: |
|         0         |          Arm Architecture Calls          |
|         1         |            CPU Service Calls             |
|         2         |            SiP Service Calls             |
|         3         |            OEM Service Calls             |
|         4         |      Standard Secure Service Calls       |
|         5         |    Standard Hypervisor Service Calls     |
|         6         | Vendor Specific Hypervisor Service Calls |
|         7         |    Vendor Specific EL3 Monitor Calls     |
|       8-47        |         Reserved for future use          |
|       48-49       |        Trusted Application Calls         |
|       50-63       |             Trusted OS Calls             |

<a id="table2">SMCC</a>

| 注册的funciton id范围 | handler的服务类型                                |
| --------------------- | ------------------------------------------------ |
| 0x80000000-0x8000FFFF | SMC32: Arm Architecture Calls                    |
| 0x81000000-0x8100FFFF | SMC32: CPU Service Calls                         |
| 0x82000000-0x8200FFFF | SMC32: SiP Service Calls                         |
| 0x83000000-0x8300FFFF | SMC32: OEM Service Calls                         |
| 0x84000000-0x8400FFFF | SMC32: Standard Service Calls                    |
| 0x85000000-0x8500FFFF | SMC32: Standard Hypervisor Service Calls         |
| 0x86000000-0x8600FFFF | SMC32: Vendor Specific Hypervisor Service Calls  |
| 0x87000000-0x8700FFFF | SMC32: Vendor Specific EL3 Monitor Service Calls |
| 0x88000000-0xAFFFFFFF | Reserved for future expansion                    |
| 0xB0000000-0xB100FFFF | SMC32: Trusted Application Calls                 |
| 0xB2000000-0xBFFFFFFF | SMC32: Trusted OS Calls                          |
| 0xC0000000-0xC0FFFFFF | SMC64: Arm Architecture Calls                    |
| 0xC1000000-0xC10FFFFF | SMC64: CPU Service Calls                         |
| 0xC2000000-0xC20FFFFF | SMC64: SiP Service Calls                         |
| 0xC3000000-0xC30FFFFF | SMC64: OEM Service Calls                         |
| 0xC4000000-0xC40FFFFF | SMC64: Standard Service Calls                    |
| 0xC5000000-0xC50FFFFF | SMC64: Standard Hypervisor Service Calls         |
| 0xC6000000-0xC60FFFFF | SMC64: Vendor Specific Hypervisor Service Calls  |
| 0xC7000000-0xC70FFFFF | SMC64: Vendor Specific EL3 Monitor Service Calls |
| 0xC8000000-0xEFFFFFFF | Reserved for future expansion                    |
| 0xF0000000-0xF10FFFFF | SMC64: Trusted Application Calls                 |
| 0xF2000000-0xFFFFFFF  | SMC64: Trusted OS Calls                          |

