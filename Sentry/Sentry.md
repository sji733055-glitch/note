<!--
 * @Author: Lancelot jishiyu20060116@163.com
 * @Date: 2025-09-13 10:27:37
 * @LastEditors: Lancelot jishiyu20060116@163.com
 * @LastEditTime: 2025-10-10 13:38:05
 * @FilePath: \哨兵代码解析\哨兵代码解析.md
 * @Description: 这是默认设置,请设置`customMade`, 打开koroFileHeader查看配置 进行设置: https://github.com/OBKoro1/koro1FileHeader/wiki/%E9%85%8D%E7%BD%AE
-->
# 哨兵代码解析
> 此文档主要是为了记录对航哥的哨兵代码的理解，因为本人发现只是干看无法完全理解代码的组织，而且我需要记录理解的进度

## BSP层(板机支持)
### CAN
> CAN设备驱动的结构体定义与相关宏定义
#### CAN发送与接收模式
```c
/* 接收模式枚举 */
typedef enum {
    CAN_MODE_BLOCKING,
    CAN_MODE_IT
} CAN_Mode;
```
* 此结构体用于表示Can通信的接收发送模式（阻塞和中断），阻塞则不断轮询（占用CPU资源），中断则使用事件组
#### CAN设备实例结构体
```c
/* CAN设备实例结构体 */
typedef struct
{
    CAN_HandleTypeDef *can_handle;  // CAN句柄
    // 发送配置
    CAN_TxHeaderTypeDef txconf;     // 发送配置
    uint32_t tx_id;                 // 发送ID
    uint32_t tx_mailbox;            // 发送邮箱号
    uint8_t tx_buff[8];             // 发送缓冲区
    CAN_Mode tx_mode;
    // 接收配置
    uint32_t rx_id;                 // 接收ID
    uint8_t rx_buff[8];          // 接收缓冲区
    uint8_t rx_len;                 // 接收长度
    CAN_Mode rx_mode;
    // 事件
    uint32_t eventflag;
} Can_Device;
```

#### 初始化配置结构体

```c
typedef struct
{
    CAN_HandleTypeDef *can_handle;
    uint32_t tx_id;
    uint32_t rx_id;
    CAN_Mode tx_mode;
    CAN_Mode rx_mode;
} Can_Device_Init_Config_s;
```

#### CAN总线管理结构
```c
/* CAN总线管理结构 */
typedef struct {
    CAN_HandleTypeDef *hcan;
    Can_Device devices[MAX_DEVICES_PER_CAN_BUS];
    osal_mutex_t bus_mutex;
    uint8_t device_count;
} CANBusManager;
```

* 此结构体用于管理一条Can总线的所有相关资源与设备实例
* CAN_HandleTypeDef *hcan用于标记这根Can总线是hcan1还是hcan2
*  Can_Device devices[MAX_DEVICES_PER_CAN_BUS]存放挂载在这条can总线上的所以实例
*  osal_mutex_t bus_mutex互斥锁用于资源保护
*  uint8_t device_count用于记录设备数量

#### CAN接收与发送报文
```c
typedef struct
{
    CAN_HandleTypeDef *can_handle;
    CAN_TxHeaderTypeDef txconf;     // 发送配置
    uint32_t tx_mailbox;            // 发送邮箱号
    uint8_t tx_buff[8];             // 发送缓冲区
} CanTxMessage_t;
typedef struct
{
    CAN_HandleTypeDef *can_handle;
    CAN_RxHeaderTypeDef rxconf;     // 接收配置
    uint8_t rx_buff[8];             // 接收缓冲区
} CanRxMessage_t;
```

* 这两个结构体变量用于自定义发送接收报文（不必初始化设备），用于动态灵活地发送报文
* 与设备注册结构的关联就是，设备其实是集成了这两部分

#### CAN库静态变量

```c
/* CAN 事件定义 */
#define CAN_EVENT_TX_MAILBOX0_DONE (0x01 << 0)
#define CAN_EVENT_TX_MAILBOX1_DONE (0x01 << 1)
#define CAN_EVENT_TX_MAILBOX2_DONE (0x01 << 2)
```
* 这里的宏定义作为CAN三个额邮箱的事件标志位
```c
// 全局CAN事件
static osal_event_t global_can_event;
```

* 此事件作为全局CAN事件组，在初次设置后再此事件组内进行
```c
// CAN总线管理器数组
static CANBusManager can_bus_managers[CAN_BUS_NUM];
```
* 总线管理器，用于储存初始化的CAN总线，并加入进数组内
```c
// 静态声明32个事件标志
static uint32_t can_device_event_flags[32] = {
    0x00000001, 0x00000002, 0x00000004, 0x00000008,
    0x00000010, 0x00000020, 0x00000040, 0x00000080,
    0x00000100, 0x00000200, 0x00000400, 0x00000800,
    0x00001000, 0x00002000, 0x00004000, 0x00008000,
    0x00010000, 0x00020000, 0x00040000, 0x00080000,
    0x00100000, 0x00200000, 0x00400000, 0x00800000,
    0x01000000, 0x02000000, 0x04000000, 0x08000000,
    0x10000000, 0x20000000, 0x40000000, 0x80000000
};
```
* 用于分配给各个CAN设备的等待标志位
```c
// CAN过滤器索引
static uint8_t can1_filter_idx = 0, can2_filter_idx = 14; // 0-13给can1用,14-27给can2用
```
* 用于分配过滤器槽位

#### 配置过滤器函数

```c
/**
 * @description: 添加CAN过滤器
 * @param {Can_Device*} device
 * @return {*}
 */
static void CANAddFilter(Can_Device *device)
{
    CAN_FilterTypeDef can_filter_conf;
     
    can_filter_conf.FilterMode = CAN_FILTERMODE_IDLIST;                                                       // 使用id list模式,即只有将rxid添加到过滤器中才会接收到,其他报文会被过滤
    can_filter_conf.FilterScale = CAN_FILTERSCALE_16BIT;                                                      // 使用16位id模式,即只有低16位有效
    can_filter_conf.FilterFIFOAssignment = (device->tx_id & 1) ? CAN_RX_FIFO0 : CAN_RX_FIFO1;              // 奇数id的模块会被分配到FIFO0,偶数id的模块会被分配到FIFO1
    can_filter_conf.SlaveStartFilterBank = 14;                                                                // 从第14个过滤器开始配置从机过滤器(在STM32的BxCAN控制器中CAN2是CAN1的从机)
    can_filter_conf.FilterIdLow = device->rx_id << 5;                                                      // 过滤器寄存器的低16位,因为使用STDID,所以只有低11位有效,高5位要填0
    can_filter_conf.FilterBank = device->can_handle == &hcan1 ? (can1_filter_idx++) : (can2_filter_idx++); // 根据can_handle判断是CAN1还是CAN2,然后自增
    can_filter_conf.FilterActivation = CAN_FILTER_ENABLE;                                                     // 启用过滤器

    HAL_CAN_ConfigFilter(device->can_handle, &can_filter_conf);
}

```
* 这里有一点疑问就是为什么FIFO的分配要看txid的奇偶
#### 检查id冲突函数

```c
/**
 * @description: 检查设备ID冲突
 * @param {CANBusManager*} bus
 * @param {uint16_t} tx_id
 * @param {uint16_t} rx_id
 * @return {bool} true表示有冲突，false表示无冲突
 */
static bool check_device_id_conflict(CANBusManager *bus, uint16_t tx_id, uint16_t rx_id) {
    for(uint8_t i = 0; i < MAX_DEVICES_PER_CAN_BUS; i++) {
        if(bus->devices[i].can_handle != NULL) {  // 使用can_handle判断设备是否存在
            if(bus->devices[i].rx_id == rx_id) {
                return true;
            }
        }
    }
    return false;
}
```

* 入参是设备的rx和tx的id和某一条总线管理器的指针（是设备所在的总线）
* 通过遍历总线管理器上的Device的id来检验是否有总线冲突
* 此函数用于CAN设备初始化


#### 全局变量设置函数

```c
static void BSP_CAN_Init_Global(void)
{
    static uint8_t initialized = 0;
    
    if (!initialized) {
        // 创建全局CAN事件
        osal_event_create(&global_can_event, "GlobalCANEvent");
        
        initialized = 1;
    }
}

```
* 通过`initialized`静态变量，保证此全局event只创建一次，用于CAN设备的初始化函数

#### CAN总线管理器初始化函数

```c
static CANBusManager* BSP_CAN_InitBusManager(CAN_HandleTypeDef *hcan)
{
    // 查找现有的总线管理器
    for (int i = 0; i < CAN_BUS_NUM; i++) {
        if (can_bus_managers[i].hcan == hcan) {
            return &can_bus_managers[i];
        }
    }
    
    // 查找空闲的总线管理器
    for (int i = 0; i < CAN_BUS_NUM; i++) {
        if (can_bus_managers[i].hcan == NULL) {
            can_bus_managers[i].hcan = hcan;
            can_bus_managers[i].device_count = 0;
            
            // 创建总线互斥锁
            char mutex_name[16];
            snprintf(mutex_name, sizeof(mutex_name), "CAN_Mutex_%d", i);
            osal_mutex_create(&can_bus_managers[i].bus_mutex, mutex_name);
            
            return &can_bus_managers[i];
        }
    }
    
    return NULL;
}

```

* 先遍历总线管理器数组，若已有要初始的CAN，则返回此总线管理器的地址指针
* 若此CAN没有初始化，则继续遍历总线管理器数组，找到空闲的总线管理器
* 之后创建互斥量，注册进设备管理器的互斥量成员，并返回地址


#### CAN设备初始化函数

```c
/**
 * @description: 初始化CAN设备
 * @param {Can_Device_Init_Config_s*} config
 * @return {*}
 */
Can_Device* BSP_CAN_Device_Init(Can_Device_Init_Config_s *config)
{
    if (config == NULL || config->can_handle == NULL) {
        return NULL;
    }
    
    // 初始化全局资源
    BSP_CAN_Init_Global();
    
    // 获取或初始化总线管理器
    CANBusManager *bus_manager = BSP_CAN_InitBusManager(config->can_handle);
    if (bus_manager == NULL || bus_manager->device_count >= MAX_DEVICES_PER_CAN_BUS) {
        return NULL;
    }
    
    // 检查ID冲突
    if (check_device_id_conflict(bus_manager, config->tx_id, config->rx_id)) {
        return NULL;
    }
    
    // 获取设备实例
    Can_Device *device = &bus_manager->devices[bus_manager->device_count];
    memset(device, 0, sizeof(Can_Device));
    
    // 配置设备参数
    device->can_handle = config->can_handle;
    device->tx_id = config->tx_id;
    device->rx_id = config->rx_id;
    device->tx_mode = config->tx_mode;
    device->rx_mode = config->rx_mode;
    
    // 配置发送参数
    device->txconf.StdId = config->tx_id;
    device->txconf.ExtId = 0;
    device->txconf.IDE = CAN_ID_STD;
    device->txconf.RTR = CAN_RTR_DATA;
    device->txconf.DLC = 8;
    device->txconf.TransmitGlobalTime = DISABLE;
    
    // 分配事件标志
    int flag_index = bus_manager->device_count + (bus_manager - can_bus_managers) * MAX_DEVICES_PER_CAN_BUS;
    if (flag_index < 32) {
        device->eventflag = can_device_event_flags[flag_index];
    } else {
        return NULL; // 超出可用事件标志范围
    }
    
    // 添加CAN过滤器
    CANAddFilter(device);
    
    // 增加设备计数
    bus_manager->device_count++;
    
    // 启动CAN接收中断
    if (config->rx_mode == CAN_MODE_IT) {
        HAL_CAN_ActivateNotification(config->can_handle, CAN_IT_RX_FIFO0_MSG_PENDING | CAN_IT_RX_FIFO1_MSG_PENDING);
    }
    
    return device;
}
```

* 此函数用于初始化CAN设备，并将其注册进总线管理器
* 第一步先初始化全局事件（若已有，则不会进行事件组的创建）
* 之后根据config中的hcan进行总线管理器的初始化，若此总线设备管理区已存在则直接返回指针，若无创建后再返回指针
* 之后分别配置发送与接收参数
* 再然后依据已有的设备数将空闲的标志位，
























































### UART
> UART设备驱动的结构体定义与相关宏定义
#### UART事件定义
```c
#define UART_RX_DONE_EVENT    (0x01 << 0)  // 接收完成事件
#define UART_TX_DONE_EVENT    (0x01 << 1)  // 发送完成事件
#define UART_ERR_EVENT        (0x01 << 2)  // uart错误事件
```
* 这里使用了事件标志组的每一位来作为上面三种任务唤醒的条件，一旦有宏定义的位被置一就唤醒
#### UART发送与接受模式
```c
typedef enum {
    UART_MODE_BLOCKING,//阻塞
    UART_MODE_IT,//中断
    UART_MODE_DMA//DMA模式
} UART_Mode;

```
#### UART设备结构体

```c
// UART设备结构体
typedef struct {
    UART_HandleTypeDef *huart;
    
    // 接收相关（需要缓冲区管理）
    uint8_t (*rx_buf)[2];             // 指向外部定义的双缓冲区
    uint16_t rx_buf_size;             // 缓冲区大小
    volatile uint8_t rx_active_buf;   // 当前活动缓冲区
    uint16_t real_rx_len;             // 实际接收数据长度
    uint16_t expected_rx_len;         // 预期长度（0为不定长）

    // uart事件（用于接收和发送完成通知）
    osal_event_t uart_event;
    
    // 配置模式
    UART_Mode rx_mode;
    UART_Mode tx_mode;
    
    // 错误状态
    uint32_t error_status;
} UART_Device;
```
#### 初始化配置结构体

```c
// 初始化配置结构体
typedef struct {
    UART_HandleTypeDef *huart;        // UART句柄
    uint8_t (*rx_buf)[2];             // 外部接收缓冲区指针
    uint16_t rx_buf_size;             // 接收缓冲区大小
    uint16_t expected_rx_len;         // 预期长度（0为不定长）
    UART_Mode rx_mode;                // 接收模式
    UART_Mode tx_mode;                // 发送模式
} UART_Device_init_config;
```
* 此模块用于在外边调用串口并注册设备时，将配置同步于串口设备（自定义）
#### 启动串口接收函数
```c
static void Start_Rx(UART_Device *device) {
    // 阻塞模式不需要启动后台接收
    if (device->rx_mode == UART_MODE_BLOCKING) {
        return;
    }
    uint8_t next_buf = !device->rx_active_buf;
    HAL_StatusTypeDef status = HAL_OK;
    // 根据接收模式启动相应的接收
    if (device->rx_mode == UART_MODE_IT) {
        status = HAL_UARTEx_ReceiveToIdle_IT(device->huart, (uint8_t*)&device->rx_buf[next_buf], 
                                        device->expected_rx_len ? device->expected_rx_len : device->rx_buf_size);
    } else if (device->rx_mode == UART_MODE_DMA) {
        status = HAL_UARTEx_ReceiveToIdle_DMA(device->huart, (uint8_t*)&device->rx_buf[next_buf], 
                                        device->expected_rx_len ? device->expected_rx_len : device->rx_buf_size);
    }
    // 只有在启动成功时才切换缓冲区
    if (status == HAL_OK && device->rx_mode != UART_MODE_BLOCKING) {
        device->rx_active_buf = next_buf;//切换缓冲区
    }
}
```
* 此函数主要用于在串口初始化时第一次启动串口接收（DMA和中断），并在DMA和中断的接收回调函数中重新启动下一次接收
* 至于为什么阻塞模式不需要启动后台（中断中接收），因为阻塞模式直接在线程中无限等待（占用CPU），直至接收到数据，想要再次接收只需要再次调用接收函数
* 而在双缓冲区接收下，在第一次启动时由于没有缓冲区正在接收数据，这里的next_buf也没有什么意义，此时就将正在接受数据的buf设为next_buf。而接收到数据并开启下一次接收时则切换为上次未接收数据的buf
* 双缓冲区在DMA和中断模式下才使用

#### 接收完成处理函数

```c
static void Process_Rx_Complete(UART_Device *device, uint16_t Size) {
    // 计算实际接收长度
    if(device->expected_rx_len == 0 && device->rx_mode == UART_MODE_DMA){
        // 获取单缓冲区大小
        Size = device->rx_buf_size - __HAL_DMA_GET_COUNTER(device->huart->hdmarx);
        // 长度校验
        if(Size > device->rx_buf_size){ 
            Size = device->rx_buf_size;
        }
    }
    device->real_rx_len = Size;
    // 事件通知
    osal_event_set(&device->uart_event, UART_RX_DONE_EVENT);
}
```

* 此函数用于DMA和中断模式的串口中断回调函数中使用
* 首先判断是否是DMA和不定长数据，若是则计算实际接收长度（因为是不定长这个只有在你接收到数据时可以得到），并将长度返回给Size，并更新device的实际长度
* 最后设置事件标志位（接受完成位）
#### 对外接口函数
```c
static UART_Device registered_uart[UART_MAX_INSTANCE_NUM] = {0}; // 结构体数组
static bool uart_used[UART_MAX_INSTANCE_NUM] = {false};         // 添加使用状态标记


/* 对外接口函数 */
UART_Device* BSP_UART_Device_Init(UART_Device_init_config *config){
    // 参数检查
    if (!config || !config->huart) {
        return NULL;
    }
    // 检查实例是否已存在
    for(int i=0; i<UART_MAX_INSTANCE_NUM; i++){
        if(uart_used[i] && registered_uart[i].huart == config->huart)
        {   
            LOG_ERROR("UART instance already exists, huart=%p", config->huart);
            return NULL;
        }
    }
    // 查找空闲槽位
    int free_index = -1;
    for(int i=0; i<UART_MAX_INSTANCE_NUM; i++){
        if(!uart_used[i]){
            free_index = i;
            break;
        }
    }
    if(free_index == -1) {
        LOG_ERROR("No free UART instance slots available");
        return NULL;
    }
    // 初始化参数
    UART_Device *device = &registered_uart[free_index];
    memset(device, 0, sizeof(UART_Device));
    device->huart = config->huart;
    // 缓冲区配置部分
    if (config->rx_buf != NULL) {
        device->rx_buf = (uint8_t (*)[2])config->rx_buf;
        device->rx_buf_size = config->rx_buf_size;
    } 
    else{
        LOG_ERROR("RX buffer is NULL");
        return NULL;
    }
    // 配置接收长度
    device->expected_rx_len = config->expected_rx_len;
    if (device->expected_rx_len > device->rx_buf_size) {device->expected_rx_len = device->rx_buf_size;} // 确保不超过缓冲区大小
    // 配置模式
    device->rx_mode = config->rx_mode;
    device->tx_mode = config->tx_mode;
    // 创建具有唯一标识的事件名称
    static char event_name[16];
    snprintf(event_name, sizeof(event_name), "uart_event_%d",free_index);
    // 创建事件组
    osal_status_t status = osal_event_create(&device->uart_event, event_name);
    if (status != OSAL_SUCCESS) {
        LOG_ERROR("Failed to create UART event, status=%d", status);
        return NULL;
    }
    // 增加使用计数
    uart_used[free_index] = true;
    Start_Rx(device);
    LOG_INFO("UART device initialized successfully, huart=%p, rx_mode=%d, tx_mode=%d", 
             device->huart, device->rx_mode, device->tx_mode);
    return device;
}
```

* 此函数用于外部模块初始化串口实例
* registered_uart用于将初始化的实例注册进数组
* uart_used用于标志uart已经使用（stm32f407只有huart1，huart2，huart3可以使用）
* 先遍历串口设备结构体数组和使用标记与config（要配置的串口进行对比），若有则返回NULL并显示串口已存在
* 若此串口没有初始化，则遍历uart_used串口使用标志，使用free_index获得空槽位置（uart_used中为false的位代表串口未使用）
* free_index初始值为-1，在遍历后检测其值，若为-1则返回NULL，显示已无空槽
* free_index在uart_used和registered_uart两个数组中是一一对应的，分别代表某一个串口的实例指针和串口的使用情况
* 之后初始化一个类型为UART_Device的实例指针device，并将入参config结构体中的对应数据（如缓冲区长度与rxtx模式）赋得device
* 下一步创建事件标志组，创建完成后将使用标志置True
* 开启第一次接收
#### 串口发送数据函数
```c
int BSP_UART_Send(UART_Device *device, uint8_t *data, uint16_t len)
{
    if (device == NULL || data == NULL || len == 0) {
        return -1;
    }
    HAL_StatusTypeDef hal_status;
    // 根据发送模式选择发送方式
    switch (device->tx_mode) {
        case UART_MODE_BLOCKING:
            hal_status = HAL_UART_Transmit(device->huart, data, len, HAL_MAX_DELAY);//阻塞死等
            break;
        case UART_MODE_IT:
            hal_status = HAL_UART_Transmit_IT(device->huart, data, len);
            break;
        case UART_MODE_DMA:
            hal_status = HAL_UART_Transmit_DMA(device->huart, data, len);
            break;
        default:
            LOG_ERROR("Invalid TX mode: %d", device->tx_mode);
            return -1;
    }
    // 对于非阻塞模式，等待发送完成事件
    if (hal_status == HAL_OK && device->tx_mode != UART_MODE_BLOCKING) {
        unsigned int actual_flags;
        osal_status_t status = osal_event_wait(&device->uart_event, UART_TX_DONE_EVENT,
                                               OSAL_EVENT_WAIT_FLAG_OR | OSAL_EVENT_WAIT_FLAG_CLEAR, 
                                               OSAL_WAIT_FOREVER, &actual_flags);
        return (status == OSAL_SUCCESS) ? len : -1;
    }
    return -1;
}
```
* 首先根据不同的发送模式（txmode）调用不同的串口发送函数
* 如果是非阻塞发送，执行等待标志位函数（发送完成回调函数执行置位）
#### 串口读取函数
```c
uint8_t* BSP_UART_Read(UART_Device *device)
{
    if (device == NULL) {
        LOG_ERROR("Invalid device for UART read");
        return NULL;
    }
    
    // 对于阻塞模式，使用第一个缓冲区进行接收
    if (device->rx_mode == UART_MODE_BLOCKING) {
        HAL_StatusTypeDef status = HAL_UARTEx_ReceiveToIdle(device->huart, (uint8_t*)device->rx_buf[0], 
                                                          device->expected_rx_len ? device->expected_rx_len : device->rx_buf_size,
                                                          &device->real_rx_len,HAL_MAX_DELAY);
        if (status == HAL_OK) {
            return device->rx_buf[0];
        }
        return NULL;
    } else {
        // 对于中断/DMA模式，等待接收完成事件
        unsigned int actual_flags;
        osal_status_t status = osal_event_wait(&device->uart_event, UART_RX_DONE_EVENT,
                                              OSAL_EVENT_WAIT_FLAG_OR | OSAL_EVENT_WAIT_FLAG_CLEAR, OSAL_WAIT_FOREVER, &actual_flags);
        if (status == OSAL_SUCCESS) {
            // 返回非活动缓冲区的数据指针
            return device->rx_buf[!device->rx_active_buf];
        }
        return NULL;
    }
}
```
* 此函数用于读取串口实例缓冲区的数据
* 若串口接收模式（rxmode）为轮询阻塞方式则只使用一个缓冲区
* 若为非阻塞则进入等待事件标志函数（若没等到则放开cpu权限），等进入接收中断会将接收完成事件（UART_RX_DONE_EVENT）置1，后等待成功返回空闲缓冲区中的数
#### 串口实例反初始化函数

```c
void BSP_UART_Deinit(UART_Device *device) {
    if (device == NULL) {
        LOG_ERROR("Invalid device for UART deinit");
        return;
    }
    
    for(int i=0; i<UART_MAX_INSTANCE_NUM; i++){
        if(&registered_uart[i] == device){
            HAL_UART_Abort(device->huart);
            osal_event_delete(&device->uart_event);
            memset(device, 0, sizeof(UART_Device));
            uart_used[i] = false;
            LOG_INFO("UART device deinitialized successfully, huart=%p", device->huart);
            break;
        }
    }
}

```
* 轮询实例所在数组位置，执行事件删除，并解除使用计数
#### 串口回调函数
```c
// 接收完成回调
void HAL_UARTEx_RxEventCallback(UART_HandleTypeDef *huart, uint16_t Size) {
    for(int i=0; i<UART_MAX_INSTANCE_NUM; i++){
        if(uart_used[i] && registered_uart[i].huart == huart){
            Process_Rx_Complete(&registered_uart[i], Size);
            Start_Rx(&registered_uart[i]); //重启接收
            break;
        }
    }
}
//发送完成回调
void HAL_UART_TxCpltCallback(UART_HandleTypeDef *huart) {
    for(int i=0; i<UART_MAX_INSTANCE_NUM; i++){
        if(uart_used[i] && registered_uart[i].huart == huart){
            osal_event_set(&registered_uart[i].uart_event, UART_TX_DONE_EVENT);//遍历对应串口事件置1
            break;
        }
    }
}
// 错误处理回调
void HAL_UART_ErrorCallback(UART_HandleTypeDef *huart) {
    for(int i=0; i<UART_MAX_INSTANCE_NUM; i++){
        if(uart_used[i] && registered_uart[i].huart == huart){     
            osal_event_set(&registered_uart[i].uart_event, UART_ERR_EVENT);   
            HAL_UART_AbortReceive(huart);
            Start_Rx(&registered_uart[i]);
            break;
        }
    }
}
```
* 

## Modules(模块层)
### Offline
> 

## Applications(应用层)