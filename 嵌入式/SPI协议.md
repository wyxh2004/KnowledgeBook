SPI
SPI通讯协议
简介
SPI 是英语Serial Peripheral interface的缩写, 顾名思义就是串行外围设备接口. 是Motorola(摩托罗拉)首先在其MC68HCXX系列处理器上定义的. 与UART相比SPI是同步通信协议. 与I2C相比SPI是全双工的协议, 即使用两条数据线分别用来发送和接收数据, 从而使得SPI相比I2C简单了不少.

SPI,是一种高速的, 全双工, 同步的通信总线, 并且在芯片的管脚上只占用四根线, 节约了芯片的管脚,同时为PCB的布局上节省空间, 提供方便, 主要应用在 EEPROM, FLASH, 实时时钟, AD转换器, 还有数字信号处理器和数字信号解码器之间.

SPI区分主机与从机, 其中产生时钟的一侧称为主机, 另一侧称为从机. 总是同一时刻只能有一个主机(运行过程中允许主机切换), 但是可以有多个从机.

SPI信号线
SPI接口一般使用四条信号线通信:
硬件上为4根线: SDI (数据输入), SDO (数据输出), SCK (时钟), CS/SS (片选)

MISO: 主设备输入/从设备输出引脚. 该引脚在从模式下发送数据,在主模式下接收数据.
MOSI: 主设备输出/从设备输入引脚. 该引脚在主模式下发送数据,在从模式下接收数据.
SCLK: 串行时钟信号,由主设备产生.
CS/SS: 从设备片选信号,由主设备控制. 它的功能是用来作为“片选引脚”,也就是选择指定的从设备,让主设备可以单独地与特定从设备通讯,避免数据线上的冲突.
主机和从机对应接口相连, 其中主机可以有多个CS\SS引脚, 每个CS/SS引脚对应一个从机外设, 在从机数量较多使, 相较于I2C, SPI更费主机引脚.

传输时序
SPI单元有一个移位寄存器, 传输时在每一个时钟周期中将高位(低位)数据放到输出信号线上, 然后寄存器左移, 最后将读入的数据存至空出来的最低位, 这样便完成了一位的交换. 读取8次后完成一字节的交换.

主机先将SS信号拉低, 这样保证开始接收数据, 接着在主机时钟的指引下进行数据的交换.

SPI共定义了4种传输模式:

模式0:
CPOL=0: 空闲状态时, SCK为低电平;
CPHA=0: SCK第一个边沿移入数据, 第二个边沿移出数据;模式0
模式1:
CPOL=0: 空闲状态时, SCK为低电平;
CPHA=1: SCK第一个边沿移出数据, 第二个边沿移入数据;模式1
模式2:
CPOL=1: 空闲状态时, SCK为高电平;
CPHA=0: SCK第一个边沿移入数据, 第二个边沿移出数据;模式2
模式3:
CPOL=1: 空闲状态时, SCK为高电平;
CPHA=1: SCK第一个边沿移出数据, 第二个边沿移入数据;模式3
对于模式0和模式2, 因为要在第一个边沿移入(读取)数据, 所以要在SS信号的下降沿就进行数据移出.
电气特性
SPI的输出引脚均可以采用推挽输出增大驱动能力, 从而使电平变换更加迅速(相比I2C的开漏输出 + 总线上拉电阻).

读写
SPI的传输被定义成数据交换, 将主机上的一个数据与从机上的数据对换, 如果主机只想发送不想接收, 则可以选择发送拷贝并放弃交换回来的值(抛玉引砖). 反正, 只想接收不想发送则(抛砖引玉).

SPI读写W25Q64(STM32软件模拟)
W25Q64是一款采用SPI通讯协议的FLASK的存储芯片, 容量为8MByte. SPI工作在模式0, 或模式3(下面的代码按照模式0写的).

MySPI.c为通讯协议层驱动. W25Q64.c是模块驱动. W25Q64的DI引脚接PA7, DO引脚接PA6, CLK引脚接PA5, SS引脚接PA4, VCC接3.3V, GND接GND.

[]
// MySPI.c
#include "stm32f10x.h"                  // Device header
#include "MySPI.h"

static void MySPI_W_SS(uint8_t BitValue) { //SS引脚控制
	GPIO_WriteBit(GPIOA, GPIO_Pin_4, (BitAction)BitValue);
}

static void MySPI_W_SCK(uint8_t BitValue) { //SCK引脚控制
	GPIO_WriteBit(GPIOA, GPIO_Pin_5, (BitAction)BitValue);
}

static void MySPI_W_MOSI(uint8_t BitValue) { //MOSI引脚控制
	GPIO_WriteBit(GPIOA, GPIO_Pin_7, (BitAction)BitValue);
}

static uint8_t MySPI_R_MISO(void) { //MISO引脚控制
	return GPIO_ReadInputDataBit(GPIOA, GPIO_Pin_6);
}

void MySPI_Init(void) {
	GPIO_InitTypeDef GPIO_InitStructure;
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IPU; //PA6为MISO, 采用上拉输入
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_6;
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_Init(GPIOA, &GPIO_InitStructure);
	
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_4 | GPIO_Pin_5 | GPIO_Pin_7; //三个输出脚均采用开漏输出
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;
	GPIO_Init(GPIOA, &GPIO_InitStructure);
	
	MySPI_W_SS(1); //起始拉高SS
	MySPI_W_SCK(0); //模式0, SCK默认电平为低电平
}

void MySPI_Start(void) { //起始信号, 片选
	MySPI_W_SS(0);
}

void MySPI_Stop(void) { //终止信号, 关闭片选
	MySPI_W_SS(1);
}

uint8_t MySPI_SwapByte(uint8_t Byte) { //交换一个字节
	int8_t loc = 8;	
	
	while (loc) {
		MySPI_W_MOSI(Byte & 0x80); //把最高位放在MOSI线上
		Byte <<= 1; //数据左移
		MySPI_W_SCK(1); //拉高SCK     全程不用延迟，W25Q64速度为80MHz
		Byte |= MySPI_R_MISO(); //读取数据
		MySPI_W_SCK(0); //拉低SCK
		loc--;
	}
	
	return Byte; //返回读取到的字节
}

[]
// W25Q64.c
#include "stm32f10x.h"                  // Device header
#include "W25Q64.h"
#include "MySPI.h"
#include "W25Q64_Ins.h"

void W25Q64_Init(void) {
	MySPI_Init();
}

void W25Q64_ReadID(uint8_t *MID, uint16_t *DID) { // 读入W25Q64的设备ID和厂商ID
	MySPI_Start();
	MySPI_SwapByte(W25Q64_JEDEC_ID); //发送第一位(告知W25Q64想要操作的指令, 此处为读取ID)
	(*MID) = MySPI_SwapByte(W25Q64_DUMMY_BYTE); //读取厂商ID
	(*DID) = 0;
	(*DID) |= MySPI_SwapByte(W25Q64_DUMMY_BYTE) << 8; // 读取设备ID高8位
	(*DID) |= MySPI_SwapByte(W25Q64_DUMMY_BYTE); // 读取设备ID低8位
	MySPI_Stop();
}

void W25Q64_WriteEnable(void) { //写入使能(告知W25Q64接下来要写入了)
	MySPI_Start();
	MySPI_SwapByte(W25Q64_WRITE_ENABLE);
	MySPI_Stop();
}

void W25Q64_WaitBusy(void) { //W25Q64在写入一次后要将数据从寄存器转移到FLASK中需要10ms左右的时间, 因此在进行下一次操作时要等待写入完成
	uint32_t Timeout = 50000;
	MySPI_Start();
	MySPI_SwapByte(W25Q64_READ_STATUS_REGISTER_1);
	while ((MySPI_SwapByte(W25Q64_DUMMY_BYTE) & 0x01) && Timeout) Timeout--;
	MySPI_Stop();
}

void W25Q64_PageProgram(uint32_t Address, uint8_t *DataArray, uint16_t ArraySize) { //页编辑(W25Q64每次只能整页(256KB)进行编辑)
	W25Q64_WaitBusy(); //等待退出
	W25Q64_WriteEnable(); //写使能
	MySPI_Start(); //开始通讯
	MySPI_SwapByte(W25Q64_PAGE_PROGRAM); //发送页编辑指令
	MySPI_SwapByte(Address >> 16); //发送地址
	MySPI_SwapByte(Address >> 8);
	MySPI_SwapByte(Address);
	
	while (ArraySize) { // 逐位发送
		MySPI_SwapByte(*DataArray);
		DataArray++;
		ArraySize--;
	}
	
	MySPI_Stop(); // 停止通讯
}

void W25Q64_SetErase(uint32_t Address) { //块(2KB)擦除, W25Q64只能由1写为0, 因此写入之前要擦除一次数据
	W25Q64_WaitBusy();//等待退出
	W25Q64_WriteEnable();
	MySPI_Start();
	MySPI_SwapByte(W25Q64_SECTOR_ERASE_4KB); //发送擦除指令
	MySPI_SwapByte(Address >> 16); //擦除地址
	MySPI_SwapByte(Address >> 8);
	MySPI_SwapByte(Address);
	MySPI_Stop();
}

void W25Q64_ReadData(uint32_t Address, uint8_t *DataArray, uint32_t ArraySize) { //读取数据
	W25Q64_WaitBusy();//等待退出
	MySPI_Start();
	MySPI_SwapByte(W25Q64_READ_DATA);
	MySPI_SwapByte(Address >> 16);
	MySPI_SwapByte(Address >> 8);
	MySPI_SwapByte(Address);
	
	while (ArraySize) {
		(*DataArray) = MySPI_SwapByte(W25Q64_DUMMY_BYTE);
		DataArray++;
		ArraySize--;
	}
	MySPI_Stop();
}

[]
// main.c
#include "stm32f10x.h"                  /* Device header */
#include "Delay.h"
#include "OLED.h"
#include "W25Q64.h"
#include "Delay.h"

uint8_t ArrayWrite[12] = {72, 101, 108, 108, 111, 32, 87, 111, 114, 108, 100, 33}, ArrayReceive[12] = {}; //Hello World!

static void Init(void) {
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOB, ENABLE);
	
	OLED_Init();
	W25Q64_Init();
}

int main(void) {
	Init();
	
	uint8_t MID;
	uint16_t DID;
	
	OLED_ShowString(1, 1, "MID:   DID:");
	OLED_ShowString(2, 1, "W:");
	OLED_ShowString(3, 1, "R:");
	
	W25Q64_ReadID(&MID, &DID); //读取ID
	
	W25Q64_SetErase(0x00000000);
	W25Q64_PageProgram(0x000000, ArrayWrite, 12); //写
	W25Q64_ReadData(0x000000, ArrayReceive, 12); //读
	
	uint8_t i;
	
	while (1) {
		OLED_ShowHexNum(1, 5, MID, 2);
		OLED_ShowHexNum(1, 12, DID, 4);
		
		for (i = 0; i < 12; i++) {
			OLED_ShowChar(2, 3 + i, ArrayWrite[i]);
			OLED_ShowChar(3, 3 + i, ArrayReceive[i]);
		}
		
	}
}
 
SPI读写W25Q64(STM32硬件驱动)
除了MySPI.c, 其他文件都与软件驱动相同. SS引脚采用GPIO模拟控制.

[]
#include "stm32f10x.h"                  // Device header
#include "MySPI.h"

static void MySPI_W_SS(uint8_t BitValue) { //SS
	GPIO_WriteBit(GPIOA, GPIO_Pin_4, (BitAction)BitValue);
}

void MySPI_Init(void) {
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_SPI1, ENABLE);
	
	GPIO_InitTypeDef GPIO_InitStructure; //引脚初始化
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IPU;
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_6; //MISO上拉输入
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_Init(GPIOA, &GPIO_InitStructure);
	
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_4;
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP; //SS引脚推挽输出
	GPIO_Init(GPIOA, &GPIO_InitStructure);
	
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_5 | GPIO_Pin_7; //MOSI,SCK交给SPI外设控制
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AF_PP;
	GPIO_Init(GPIOA, &GPIO_InitStructure);
	
	SPI_InitTypeDef SPI_InitSturcture;
	SPI_InitSturcture.SPI_BaudRatePrescaler = SPI_BaudRatePrescaler_16; //预分频器
	SPI_InitSturcture.SPI_CPHA = SPI_CPHA_1Edge; //SCK上升沿读取
	SPI_InitSturcture.SPI_CPOL = SPI_CPOL_Low; //SCK默认低电平
	SPI_InitSturcture.SPI_CRCPolynomial = 7; //CRC校验码, 此程序未用到
	SPI_InitSturcture.SPI_DataSize = SPI_DataSize_8b; //每字节大小
	SPI_InitSturcture.SPI_Direction = SPI_Direction_2Lines_FullDuplex; //双线全双工
	SPI_InitSturcture.SPI_FirstBit = SPI_FirstBit_MSB; //高位先行
	SPI_InitSturcture.SPI_Mode = SPI_Mode_Master; //主机模式
	SPI_InitSturcture.SPI_NSS = SPI_NSS_Soft; //SS引脚采用软件控制
	SPI_Init(SPI1, &SPI_InitSturcture);
	
	SPI_Cmd(SPI1, ENABLE); //使能
	
	MySPI_W_SS(1);
}

void MySPI_Start(void) {
	MySPI_W_SS(0);
}

void MySPI_Stop(void) {
	MySPI_W_SS(1);
}

uint8_t MySPI_SwapByte(uint8_t Byte) { //单字交换
	while (SPI_I2S_GetFlagStatus(SPI1, SPI_I2S_FLAG_TXE) != SET);
	SPI_I2S_SendData(SPI1, Byte);
	while (SPI_I2S_GetFlagStatus(SPI1, SPI_I2S_FLAG_RXNE) != SET);
	Byte = SPI_I2S_ReceiveData(SPI1);
	
	return Byte;
}

void MySPI_SwapArray(uint8_t* Array, uint8_t ArraySize) { //连续交换
	while (SPI_I2S_GetFlagStatus(SPI1, SPI_I2S_FLAG_TXE) != SET); //等待发送队列为空
	SPI_I2S_SendData(SPI1, *Array); //发送第0位
	Array++; //指针自增
	ArraySize--; //计数器自减
	while (ArraySize) {
		while (SPI_I2S_GetFlagStatus(SPI1, SPI_I2S_FLAG_TXE) != SET); //等待发送队列为空
		SPI_I2S_SendData(SPI1, *Array); //发送下一位
		while (SPI_I2S_GetFlagStatus(SPI1, SPI_I2S_FLAG_RXNE) != SET); //等待上一位传输完成
		*(Array - 1) = SPI_I2S_ReceiveData(SPI1); //接收上一位
		Array++;
		ArraySize--;
	}
	while (SPI_I2S_GetFlagStatus(SPI1, SPI_I2S_FLAG_RXNE) != SET); 
	*(Array - 1) = SPI_I2S_ReceiveData(SPI1); //接收最后一位
}

SPI从机(软件模拟)
主机通过SPI2硬件256分频(140.625kHz)连续时序发送”Hello World!”.
从机:

[]
// MySPI.c
#include "stm32f10x.h"                  // Device header
#include "MySPI.h"
#include "LED.h"

#define MySPI_SLEEP 0 //状态定义
#define MySPI_START 1

// PA4: SS	PA5: SCK	PA6: MISO	P6: MOSI

#define SS_Pin GPIO_Pin_4 // 引脚定义
#define SCK_Pin GPIO_Pin_5 
#define MISO_Pin GPIO_Pin_6
#define MOSI_Pin GPIO_Pin_7

static uint8_t Count, Status, Loc;
uint8_t Data[13] = { 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0x00 };

static uint8_t MySPI_R_SCK() { //读取时钟
	return GPIO_ReadInputDataBit(GPIOA, SCK_Pin);
}

static uint8_t MySPI_R_SS() { //读取片选
	return GPIO_ReadInputDataBit(GPIOA, SS_Pin);
}

static uint8_t MySPI_R_MOSI() { //从机读
	return GPIO_ReadInputDataBit(GPIOA, MOSI_Pin);
}

static void MySPI_W_MISO(uint8_t BitValue) { //从机写MISO
	GPIO_WriteBit(GPIOA, MISO_Pin, (BitAction)BitValue);
}

void MySPI_Init(void) {
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_AFIO,ENABLE); //开启AFIO时钟
	
	GPIO_InitTypeDef GPIO_InitStructure;
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IPU;
	GPIO_InitStructure.GPIO_Pin = SS_Pin | SCK_Pin | MOSI_Pin; // SS | CSK | MOSI 上拉输入
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_Init(GPIOA, &GPIO_InitStructure);
	
	GPIO_InitStructure.GPIO_Pin = MISO_Pin; //MISO 推挽输出
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;
	GPIO_Init(GPIOA, &GPIO_InitStructure);

	GPIO_EXTILineConfig(GPIO_PortSourceGPIOA, GPIO_PinSource4); //EXTI
	GPIO_EXTILineConfig(GPIO_PortSourceGPIOA, GPIO_PinSource5);
	
	EXTI_InitTypeDef EXTI_InitSturcture; //EXTI初始化
	EXTI_InitSturcture.EXTI_Line = EXTI_Line4 | EXTI_Line5;
	EXTI_InitSturcture.EXTI_LineCmd = ENABLE;
	EXTI_InitSturcture.EXTI_Mode = EXTI_Mode_Interrupt;
	EXTI_InitSturcture.EXTI_Trigger = EXTI_Trigger_Rising_Falling;
	EXTI_Init(&EXTI_InitSturcture);
	
	NVIC_InitTypeDef NVIC_InitStructure; //中断配置
	NVIC_InitStructure.NVIC_IRQChannel = EXTI4_IRQn;
	NVIC_InitStructure.NVIC_IRQChannelCmd = ENABLE;
	NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority = 0;
	NVIC_InitStructure.NVIC_IRQChannelSubPriority = 0;
	NVIC_Init(&NVIC_InitStructure);
	
	NVIC_InitStructure.NVIC_IRQChannel = EXTI9_5_IRQn;
	NVIC_Init(&NVIC_InitStructure);
		
	Status = MySPI_SLEEP; //初始状态
}

void EXTI4_IRQHandler(void) {
	if (EXTI_GetITStatus(EXTI_Line4)) { //SS片选
		if (!MySPI_R_SS()) { //下降沿
			Status = MySPI_START; //起始状态
			Count = Loc = 0;
			
			MySPI_W_MISO(*(Data + Loc) & 0x80); //发送数据
			*(Data + Loc) <<= 1;
		}
		else { //上升沿
			Status = MySPI_SLEEP; // 终止状态
		}
		
		EXTI_ClearITPendingBit(EXTI_Line4);
	}
}

void EXTI9_5_IRQHandler(void) {
	if (EXTI_GetITStatus(EXTI_Line5)) {
		if (Status == MySPI_START) {
			if (!MySPI_R_SCK()) { //SCK下降沿
				MySPI_W_MISO(*(Data + Loc) & 0x80); //发送数据
				*(Data + Loc) <<= 1;
			}
			else { //SCK上升沿
				*(Data + Loc) |= MySPI_R_MOSI(); //写入数据
				Count++;
				
				if (Count == 8) {
					Count = 0;
					Loc++;
				}
			}
		}
		
		EXTI_ClearITPendingBit(EXTI_Line5);
	}
}

程序现象: Hello World! 字符串在主机与从机之间来回传输.
SPI从机(硬件驱动)
根据手册中的描述折腾了好久, 最终发现只要在初始化中进行调整, 传输改为中断或者轮询即可, 代码收发数据部分与主机保持一致即可.
实测稳定传输速度可达9MHz.
[]
// 从机 MySPI.c
#include "stm32f10x.h"                  // Device header
#include "MySPI.h"
#include "LED.h"

uint8_t Cnt_Send, Cnt_Receive, Cnt_Max;
uint8_t Data[13] = { 0x41, 0x41, 0x41, 0x41, 0x41, 0x41, 0x41, 0x41, 0x41, 0x41, 0x41, 0x41, 0x00 };

void MySPI_Init(void) {
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_SPI1, ENABLE);
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_AFIO, ENABLE);
	
	GPIO_InitTypeDef GPIO_InitStructure;
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IPU;
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_4 | GPIO_Pin_5 | GPIO_Pin_7;
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_Init(GPIOA, &GPIO_InitStructure);
	
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_6;
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AF_PP;
	GPIO_Init(GPIOA, &GPIO_InitStructure);
	
	SPI_InitTypeDef SPI_InitSturcture;
	SPI_InitSturcture.SPI_BaudRatePrescaler = SPI_BaudRatePrescaler_2;
	SPI_InitSturcture.SPI_CPHA = SPI_CPHA_2Edge;
	SPI_InitSturcture.SPI_CPOL = SPI_CPOL_Low;
	SPI_InitSturcture.SPI_CRCPolynomial = 7;
	SPI_InitSturcture.SPI_DataSize = SPI_DataSize_8b;
	SPI_InitSturcture.SPI_Direction = SPI_Direction_2Lines_FullDuplex; //双线全双工
	SPI_InitSturcture.SPI_FirstBit = SPI_FirstBit_MSB; //高位先行
	SPI_InitSturcture.SPI_Mode = SPI_Mode_Slave;
	SPI_InitSturcture.SPI_NSS = SPI_NSS_Hard;
	SPI_Init(SPI1, &SPI_InitSturcture);
	
	SPI_Cmd(SPI1, ENABLE);
	Cnt_Send = Cnt_Receive = Cnt_Max = 0;
}

uint8_t MySPI_SwapByte(uint8_t Byte) {
	while (SPI_I2S_GetFlagStatus(SPI1, SPI_I2S_FLAG_TXE) != SET);
	SPI_I2S_SendData(SPI1, Byte);
	while (SPI_I2S_GetFlagStatus(SPI1, SPI_I2S_FLAG_RXNE) != SET);
	Byte = SPI_I2S_ReceiveData(SPI1);	
	
	return Byte;
}

void MySPI_SwapArray(uint8_t* Array, uint8_t ArraySize) {
	while (SPI_I2S_GetFlagStatus(SPI1, SPI_I2S_FLAG_TXE) != SET); //等待发送队列为空
	SPI_I2S_SendData(SPI1, *Array); //发送第0位
	Array++; //指针自增
	ArraySize--; //计数器自减
	while (ArraySize) {
		while (SPI_I2S_GetFlagStatus(SPI1, SPI_I2S_FLAG_TXE) != SET); //等待发送队列为空
		SPI_I2S_SendData(SPI1, *Array); //发送下一位
		while (SPI_I2S_GetFlagStatus(SPI1, SPI_I2S_FLAG_RXNE) != SET); //等待上一位传输完成
		*(Array - 1) = SPI_I2S_ReceiveData(SPI1); //接收上一位
		Array++;
		ArraySize--;
	}
	while (SPI_I2S_GetFlagStatus(SPI1, SPI_I2S_FLAG_RXNE) != SET); 
	*(Array - 1) = SPI_I2S_ReceiveData(SPI1); //接收最后一位
}

[]
// 从机 main.c
#include "stm32f10x.h"                  /* Device header */
#include "OLED.h"
#include "MySPI.h"

static void Init(void) {
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOB, ENABLE);
	NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2);
	
	OLED_Init();
	MySPI_Init();
}

int main(void) {
	Init();

	while (1) {
		OLED_ShowString(1, 1, (char *)Data);
		MySPI_SwapArray(Data, 13);
	}
}

程序现象: 从机中的 “AAAAAAAAAAAA” 和主机中的 “Hello World!” 反复交换(主机在TIM中断中进行一次通讯).
通过DMA转运应该能达到更快的速度, 等期末考完之后试一试(顺便复习一下DMA操作).