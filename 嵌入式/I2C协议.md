I2C
title: I2C communication protocol
date: 2023-12-19 20:40:15
tags:
# I2C通讯协议

## 简介
I2C总线是Philips公司在八十年代初推出的一种串行、半双工的总线, 主要用于近距离、低速的芯片之间的通信; I2C总线有两根双向的信号线, 一根数据线SDA用于收发数据, 一根时钟线SCL用于通信双方时钟的同步; I2C总线硬件结构简单, 简化了PCB布线, 降低了系统成本, 提高了系统可靠性, 因此在各个领域得到了广泛应用.

I2C总线是一种多主机总线, 连接在 I2C总线上的器件分为主机和从机. 主机有权发起和结束一次通信, 从机只能被动呼叫, 并且SCL的控制权永远在主机手中, 从机根据主机的时钟进行应答. 每个连接到I2C总线上的器件都有一个唯一的地址(7bit), 且每个器件都可以作为主机也可以作为从机(但同一时刻只能有一个主机).多主机模式在本文中不进行讨论.

相对于USART协议, i2c通过时钟和应答机制保证了传输的可靠性. 如果在传输过程中主机进入中断, 中断期间SCL电平不会发生变化, 通讯保持在原来的过程上, 退出中断后, 数据不会丢失(错过). 而应答机制避免了接收方没有接收到数据, 通过应答机制,发送方发现接收方没有接收到数据时可以选择处理方式(从新发送等), 防止重要数据丢失.

i2c总线上器件的增加和删除不影响其他器件正常工作; I2C总线在通信时总线上发送数据的器件为发送器, 接收数据的器件为接收器.

电路连接
i2c要求设备均串联在总线上, 即所有SCL均并联在一起, 所有SDA均并联在一起. 为了避免总线上出现短路, 所有端口均不允许输出高电平, 所有接口都要配置成开漏输出模式(逻辑’1’浮空, 逻辑’0’输出VDD), 然后在总线上并联1个弱上拉电阻负责输出高电平. 总线构成线与结构, 当所有端口悬空时,总线上为高电平, 否则为低电平, 因此低电平表示有设备有动作, 因此一低电平被用于应答位的有效电平.

标志信号
起始信号(重新起始信号): 初始SDA和SCL均为高电平, SDA率先下降为低电平;起始信号
终止信号: 初始SDA和SCL均为低电平, SCL率先升为高电平;终止信号
应答信号: 在发送方发送完一个字节的信息后, 接收方拉低电平知道下一个时间周期结束.应答信号
不难看出: i2c通过SCL对数据进行了隔离, 对于除了起始和终止信号, 只能在SCL为低电平时发生翻转, 高电平时必须保持不变. 而起始和终止信号在SCL为高电平时CDA会产生翻转, 并且可以通过SDA翻转时SCL的电平来判断, SCL为高, 为终止信号, 反之为起始信号.
起始信号和终止信号决定了通讯双方的状态, 并且由于其电信号的特殊性, 一般会通过SDA上升沿和当前状态进行处理. 而其他状态(接收状态到应答状态)的转变则由时钟和计数器控制.
读写时序
i2c协议要求每段指令由起始信号开始, 在产生终止信号后结束. 一个指令可以由多个数据段组成, 每个数据段为8位, 数据高位先行, 数据段之后一个时钟周期为接收方的应答信号, 之后进入新一个数据段.

写
起始信号.
第一个数据段为7位的设备地址 + 1位 写(0) 标识码组成, 标识主机要向从机写入数据.
之后的数据段为主机发送8位(1字节)的数据和从机的应答位组成.
由从机不应答告知主机或主机主动发出终止信号结束通讯.指定地址写
读
起始信号.
第一个数据段为7位的设备地址 + 1位 读(1) 标识码组成.此时主机释放SDA, 将SDA控制权转移给从机.
从机发送数据.
主机在应答后回到 第2步 继续读取下一位的数据(此时地址指针已经自增), 或不应答并发送终止信号终止本次通讯.当前地址读
对于读取和写入的数据如何规定, 由不同设备的设计决定, 参考外设的数据手册. 对于MPU6050六轴姿态传感器:
写操作指令中的第一个数据段是对寄存器指针赋值, 后续的数据段是将值写入指针指向的数据区域, 同时指针自增.
读操作指令中每个数据段均为寄存器指针指向的数据, 注意到在读操作中, 数据流是从 从机 到主机, 此时单纯的读指令下主机无法指定读取的地址. 因此产生了复合指令: 向从机写入一个字节(即修改地址指针), 然后发送一个Start信号(ReStart信号), 将从机的状态强制切换到Start, 开始下一轮的通讯, 新一轮的通讯发出读指令即可完成指定地址读.
读写时发送与接收方的处理方式
设计原则: 由于主机完全控制了SCL线, 从机不知道SCL电平会持续多久, 对主机信号的处理尽量应该提早完成.
主机发送时: 主机在SCL上升沿前将一位数据放到SDA线上, 从机在SCL上升跳变的一瞬间就应该读取SDA的数据.
从机发送时: 在前一个SCL下降沿结束后就应该将数据放到SDA线上.
电气特性
总线采用一个大电阻(一般采用4.7kΩ)进行弱上拉, 而端口输出低电平时是强下拉. 同时由于MOS管具有电容效应(等价于在电路上并联了一个很小的电容), 电平翻转无法在一瞬间完成. 电平翻转时, 构成RC电路, 其中高电平到低电平时, 端口输出强下拉的低电平, R ≈ 0Ω, 电压变化快; 而从低电平到高电平跳变时, 上拉电阻与等效电容构成了一个RC电路, R >> 0Ω.上文提到在SCL低电平时SDA进行数据翻转, 因此为了保证翻转的可靠性, 应该适当提高SCL的占空比(低电平时间比高电平时间长).
状态机定义
由于主机对通讯享有绝对控制权, 因此无需状态机处理. 从机由于主机信号的不确定性, 需要根据协议设计一套有限状态机进行状态控制.
从机状态机
休眠(SLEEP)
起止(START)
发标志位(SACK)
收标志位(RACK)
接收数据(RECEIVE)
发送数据(SEND)
终止(END)
状态转移图
代码实现(采用两片STM32F103C8T6相互通讯)
思路: 从机地址定义为0x0F. 在从机中定义了两个数组: uint8_T Rx_Data[8], Tx_Data[8], 用于收发数据, 主机尝试写入从机, 并从从机处取出数据. 如果上一轮通讯时序(状态机记录的状态)错误了, 通过对下一轮通讯起始信号的处理可以恢复同步.

第一个测试向从机中写入两个变化的数据(主机通过TIM每5s进一次中断, 每次发送num后num++);
第二个测试向从机中写入数据后切换到读取模式, 连续读取2个值, 并判断值与预期是否相等.
主机(硬件)
```c
// main.c
#include "stm32f10x.h"                  /* Device header */
#include "Delay.h"
#include "OLED.h"
#include "i2c.h"
#include "TIM.h"
#include "LED.h"

uint8_t Num;

static void Init(void) {
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOB, ENABLE);
	NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2);
	
	OLED_Init();
	i2c_Init();
	TIM_Init();
	LED_Init();
}

int main(void) {
	Init();
	
	while (1) {
		
	}
}
void TIM2_IRQHandler(void) { //TIM2设置5秒定时中断
	if (TIM_GetITStatus(TIM2, TIM_IT_Update)) {

        /*测试2*/
		Num = i2c_ReadReg(); // 主机读取
		if (Num == 0x21) LED_set(GPIO_Pin_8, Bit_RESET); //LED接在PA8，判断是否读取成功
		
        /*测试1*/
//		static uint8_t num = 0x00; //写入两个值
//		num++;
//		MPU6050_WriteReg(num, 0x41);
		
		TIM_ClearITPendingBit(TIM2, TIM_IT_Update); //清除标志位
	}
}
[]
// i2c.c
#include "stm32f10x.h"                  // Device header
#include "i2c.h"

#define ADDRESS 0xD0

void WaitEvent(I2C_TypeDef* I2Cx, uint32_t I2C_EVENT) {
	uint32_t Timeout = 10000;
	while (I2C_CheckEvent (I2Cx, I2C_EVENT) == ERROR) { //等待模式选择标志位
		Timeout--;
		if (Timeout == 0) {
			return;
		}
	}
}

void i2cWriteReg(uint8_t RegAddress, uint8_t Data) {
	I2C_GenerateSTART(I2C2, ENABLE); // 生成start, 非阻塞式函数
	WaitEvent(I2C2, I2C_EVENT_MASTER_MODE_SELECT); //等待标志位
	
	I2C_Send7bitAddress(I2C2, ADDRESS, I2C_Direction_Transmitter);
	WaitEvent(I2C2, I2C_EVENT_MASTER_TRANSMITTER_MODE_SELECTED);
	
	I2C_SendData(I2C2, RegAddress);
	WaitEvent(I2C2, I2C_EVENT_MASTER_BYTE_TRANSMITTING); //等待标志位,数据从寄存器转移到输出移位寄存器
	
	I2C_SendData(I2C2, Data);
	WaitEvent(I2C2, I2C_EVENT_MASTER_BYTE_TRANSMITTED); //等待标志位,寄存器空
	
	I2C_GenerateSTOP(I2C2, ENABLE);
}

uint8_t i2c_ReadReg() {
	uint8_t Data;
	
	I2C_GenerateSTART(I2C2, ENABLE); // 生成start, 库函数均为非阻塞式函数需要等待标志位来判断数据传输/接收完成
	WaitEvent(I2C2, I2C_EVENT_MASTER_MODE_SELECT); //等待模式选择标志位
		
	I2C_Send7bitAddress(I2C2, ADDRESS, I2C_Direction_Transmitter); //发送外设地址和写指令
	WaitEvent(I2C2, I2C_EVENT_MASTER_TRANSMITTER_MODE_SELECTED); //等待传输模式选择标志位（从机应答）
	
	I2C_SendData(I2C2, 0X41); //写入数据
	WaitEvent(I2C2, I2C_EVENT_MASTER_BYTE_TRANSMITTED); //等待标志位,数据从寄存器转移到输出移位寄存器，数据传输完成
	
	I2C_GenerateSTOP(I2C2, ENABLE);
	
	I2C_GenerateSTART(I2C2, ENABLE); //restart状态
	WaitEvent(I2C2, I2C_EVENT_MASTER_MODE_SELECT); //等待模式选择标志位
	
	I2C_Send7bitAddress(I2C2, ADDRESS, I2C_Direction_Receiver); ////发送外设地址和读指令
	WaitEvent(I2C2, I2C_EVENT_MASTER_RECEIVER_MODE_SELECTED); //等待传输模式选择标志位（从机应答）
	
	WaitEvent(I2C2, I2C_EVENT_MASTER_BYTE_RECEIVED); //输入完成
	Data = I2C_ReceiveData(I2C2); //读取数据（会自动重置RXE标志位）
	
	I2C_AcknowledgeConfig(I2C2, DISABLE); //再开始接收最后一个数据时（不能等数据传输完），设停止应答，此时接收到下一条数据字节后主机不应答（读取模式下，应答表示继续读取下一字节）
	I2C_GenerateSTOP(I2C2, ENABLE); //设stop标志位，在等待读取完成后自动输出STOP命令
	
	WaitEvent(I2C2, I2C_EVENT_MASTER_BYTE_RECEIVED); //输入完成
	Data = I2C_ReceiveData(I2C2); //读取数据（会自动重置RXE标志位）
	
	I2C_AcknowledgeConfig(I2C2, ENABLE); //重新设回应答模式
	return Data;
}

void i2c_Init(void) {	
	RCC_APB1PeriphClockCmd(RCC_APB1Periph_I2C2, ENABLE);
	
	GPIO_InitTypeDef GPIO_InitStructure;
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AF_OD;
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_10 | GPIO_Pin_11;
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_Init(GPIOB, &GPIO_InitStructure);
	
	
	I2C_InitTypeDef I2C_InitStructure; //硬件i2c初始化
	I2C_InitStructure.I2C_Ack = I2C_Ack_Enable;
	I2C_InitStructure.I2C_AcknowledgedAddress = I2C_AcknowledgedAddress_7bit;
	I2C_InitStructure.I2C_ClockSpeed = 5000;
	I2C_InitStructure.I2C_DutyCycle = I2C_DutyCycle_2;
	I2C_InitStructure.I2C_Mode = I2C_Mode_I2C;
	I2C_InitStructure.I2C_OwnAddress1 = 0x00;
	I2C_Init(I2C2, &I2C_InitStructure);
	
	I2C_Cmd(I2C2, ENABLE);
}

从机(软件模拟)
[]
// main.c
#include "stm32f10x.h"                  /* Device header */
#include "Delay.h"
#include "OLED.h"
#include "MyI2C.h"

uint16_t Key_Num;

static void Init(void) {
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOB, ENABLE);
	NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2);
	
	OLED_Init();
	MyI2C_Init();
}

int main(void) {
	Init();
	
	while (1) {
		OLED_ShowHexNum(1, 1, Rx_Data[0], 2);
		OLED_ShowHexNum(2, 1, Rx_Data[1], 2);
	}
}
[]
#include "stm32f10x.h"                  // Device header
#include "MyI2C.h"
#include "Delay.h"
#include "OLED.h"

#define SCL_PORT 			GPIOB
#define SCL_PIN 			GPIO_Pin_10
#define SDA_PORT 			GPIOB
#define SDA_PIN 			GPIO_Pin_11

#define SW_SLAVE_ADDR			0x0F

#define I2C_MODE_SLEEP         	0          	// Waiting for commands
#define I2C_MODE_START		   	1          	// 
#define I2C_MODE_SACK      		2          	// 
#define I2C_MODE_RACK		  	3          	// 
#define I2C_MODE_SEND  			4          	// 
#define I2C_MODE_RECEIVE  		5          	// 
#define I2C_MODE_END			6			// End

static void SCL_high() 			{GPIO_SetBits(SCL_PORT, SCL_PIN);Delay_us(10);}
static void SCL_low()			{GPIO_ResetBits(SCL_PORT, SCL_PIN);Delay_us(10);}
static void SDA_high()			{GPIO_SetBits(SDA_PORT, SDA_PIN);Delay_us(10);}
static void SDA_low()		  	{GPIO_ResetBits(SDA_PORT, SDA_PIN);Delay_us(10);}

static uint8_t SCL_Read()		{uint8_t t = GPIO_ReadInputDataBit(SCL_PORT, SCL_PIN); Delay_us(10); return t;}
static uint8_t SDA_Read()		{uint8_t t = GPIO_ReadInputDataBit(SDA_PORT, SDA_PIN); Delay_us(10); return t;}

struct _SwSlaveI2C_t
{
    uint8_t State;					// I2C通信状态
	uint8_t Rw;						// I2C读写标志：0-写，1-读
	uint8_t Cnt;					// SCL计数
	uint8_t* RxBuf;					// 指向接收缓冲区的指针
	uint8_t* TxBuf;					// 指向发送缓冲区的指针
	uint8_t RxIdx;					// 接收缓冲区数据写入索引，最大值255
	uint8_t TxIdx;					// 发送缓冲区数据读取索引，最大值255
} SwSlaveI2C;

uint8_t Rx_Data[8], Tx_Data[8] = {0x11, 0x21, 0x31, 0x41, 0x51, 0x61, 0x71}, Address;

void MyI2C_Init(void) {
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_AFIO,ENABLE);
	
	GPIO_InitTypeDef GPIO_InitStructure;
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_OD;				//开漏输出
	GPIO_InitStructure.GPIO_Pin = SCL_PIN;
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_Init(SCL_PORT, &GPIO_InitStructure);						//初始化SCL
	
	GPIO_InitStructure.GPIO_Pin = SDA_PIN;
	GPIO_Init(SDA_PORT, &GPIO_InitStructure);						//初始化SDA
	
	GPIO_EXTILineConfig(GPIO_PortSourceGPIOB, GPIO_PinSource11);
	GPIO_EXTILineConfig(GPIO_PortSourceGPIOB, GPIO_PinSource10);
	
	EXTI_InitTypeDef EXTI_InitStructure;							//定义结构体变量
	EXTI_InitStructure.EXTI_Line = EXTI_Line10 | EXTI_Line11;		//选择配置外部中断的0号线和1号线
	EXTI_InitStructure.EXTI_LineCmd = ENABLE;						//指定外部中断线使能
	EXTI_InitStructure.EXTI_Mode = EXTI_Mode_Interrupt;				//指定外部中断线为中断模式
	EXTI_InitStructure.EXTI_Trigger = EXTI_Trigger_Rising_Falling;	//指定外部中断线为下降沿触发
	EXTI_Init(&EXTI_InitStructure);									//将结构体变量交给EXTI_Init，配置EXTI外设
	
	NVIC_InitTypeDef NVIC_InitStructure;							//初始化中断
	NVIC_InitStructure.NVIC_IRQChannel = EXTI15_10_IRQn;
	NVIC_InitStructure.NVIC_IRQChannelCmd = ENABLE;
	NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority = 0;
	NVIC_InitStructure.NVIC_IRQChannelSubPriority = 0;
	NVIC_Init(&NVIC_InitStructure);
		
	Address = 0;													//读取地址暂存
	
	SCL_high();                                                     //释放两条总线
	SDA_high();
	
	SwSlaveI2C.State = I2C_MODE_SLEEP;								//初始化通讯相关的数据
	SwSlaveI2C.RxBuf = Rx_Data;
	SwSlaveI2C.RxIdx = 0;
	SwSlaveI2C.Cnt = 0;
	SwSlaveI2C.Rw = 0;
	SwSlaveI2C.TxBuf = Tx_Data;
	SwSlaveI2C.TxIdx = 0;
}

void EXTI15_10_IRQHandler(void) {
	if (EXTI_GetITStatus(EXTI_Line10) == SET) { 		//SCL
		EXTI_ClearITPendingBit(EXTI_Line10);
		
		if (SCL_Read() == Bit_SET) { 					//SCL上升沿
			switch (SwSlaveI2C.State) {
				case I2C_MODE_START:				//起始模式下, SCL上升沿读取设备地址
					if (SwSlaveI2C.Cnt < 8) {
						Address |= ((SDA_Read() == Bit_SET ? 1 : 0) << (7 - SwSlaveI2C.Cnt));
						SwSlaveI2C.Cnt++;
					}
					break;
					
				case I2C_MODE_RACK: 			//读取应答
					if (SDA_Read() == Bit_SET) {//未应答
						SwSlaveI2C.State = I2C_MODE_END;
					}
					else {					//应答
						SwSlaveI2C.State = I2C_MODE_SEND;
					}
					break;
				
				case I2C_MODE_RECEIVE:          //接收数据
					if (SwSlaveI2C.Cnt < 8) {
						*(SwSlaveI2C.RxBuf + SwSlaveI2C.RxIdx) &= ~(1 << (7 - SwSlaveI2C.Cnt)); //清空原本当前位的数据
						*(SwSlaveI2C.RxBuf + SwSlaveI2C.RxIdx) |= ((SDA_Read() == Bit_SET ? 1 : 0) << (7 - SwSlaveI2C.Cnt));
						SwSlaveI2C.Cnt++;
					}
					break;
				
				default:
					break;
				
			}
		}
		else {											//SCL下降沿
			switch (SwSlaveI2C.State) {
				case I2C_MODE_START:
					if (SwSlaveI2C.Cnt != 8) return;
					SwSlaveI2C.Rw = Address & 1;	//起始模式读取完毕
					if (Address >> 1 != SW_SLAVE_ADDR) {
						SwSlaveI2C.State = I2C_MODE_SLEEP;
					}
					SwSlaveI2C.Cnt = 0;
					SwSlaveI2C.State = I2C_MODE_SACK;
					SDA_low();
					break;
					
				case I2C_MODE_SACK:				//发送标志位结束
					if (SwSlaveI2C.Rw) { 	//主机读从机 从机发送
						if (*(SwSlaveI2C.TxBuf + SwSlaveI2C.TxIdx) & (1 << (7 - SwSlaveI2C.Cnt))) SDA_high();
						else SDA_low();
						SwSlaveI2C.Cnt++;
						SwSlaveI2C.State = I2C_MODE_SEND;
					}
					else { 					//主机写从机 从机接收
						SDA_high();
						SwSlaveI2C.State = I2C_MODE_RECEIVE;
					}
					break;
					
				case I2C_MODE_SEND:				//发送模式
					if (SwSlaveI2C.Cnt < 8) {//一个字节 未 发送完毕
						if (*(SwSlaveI2C.TxBuf + SwSlaveI2C.TxIdx) & (1 << (7 - SwSlaveI2C.Cnt))) SDA_high();
						else SDA_low();
						SwSlaveI2C.Cnt++;
					}
					else {					//一个字节发送完毕
						SwSlaveI2C.Cnt = 0;
						SDA_high();
						SwSlaveI2C.TxIdx++;
						SwSlaveI2C.State = I2C_MODE_RACK;
					}
					break;
					
				case I2C_MODE_RECEIVE:     	//接收8字节完成
					if (SwSlaveI2C.Cnt == 8) {			
						SDA_low();
						SwSlaveI2C.Cnt = 0;
						SwSlaveI2C.State = I2C_MODE_SACK;
						SwSlaveI2C.RxIdx++;
					}
					break;
					
				default:
					break;
				
			}					
		}
	}
	else {												//SDA
		EXTI_ClearITPendingBit(EXTI_Line11);
		
		if (SDA_Read() == Bit_SET) {                    //上升沿
			switch (SwSlaveI2C.State) {
				case I2C_MODE_END:
					if (SCL_Read() == Bit_SET) {
						SwSlaveI2C.State = I2C_MODE_SLEEP;
					}
					break;
				
				case I2C_MODE_RECEIVE:
					if (SCL_Read() == Bit_SET && SwSlaveI2C.Cnt == 1) { // SLEEP
						SwSlaveI2C.State = I2C_MODE_SLEEP;
					}
					break;
					
				default:
					break;
			}
		}                                               
		else {                                          //下降沿
			switch (SwSlaveI2C.State) {					
				case I2C_MODE_SLEEP:				//SLEEP状态下 SDA下降沿，SCL高电平： 起始信号
					if (SCL_Read() == Bit_SET) {
						Address = 0;
						SwSlaveI2C.RxIdx = 0;
						SwSlaveI2C.TxIdx = 0;
						SwSlaveI2C.Cnt = 0;
						SwSlaveI2C.State = I2C_MODE_START;
						SDA_high();
					}
					break;
				
				case I2C_MODE_RECEIVE: 
					if (SCL_Read() == Bit_SET && SwSlaveI2C.Cnt <= 1) { // 接收第一位时遇到SDA下降沿表示RESTART信号
						Address = 0; //init
						SwSlaveI2C.RxIdx = 0;
						SwSlaveI2C.TxIdx = 0;
						SwSlaveI2C.Cnt = 0;
						SwSlaveI2C.State = I2C_MODE_START; //状态转移
						SDA_high();
					}
					break;
					
				default:
					if (SCL_Read() == Bit_SET) { //如果上一轮通讯卡死了, 通过下一轮的起始信号重置状态恢复同步
						Address = 0;
						SwSlaveI2C.RxIdx = 0;
						SwSlaveI2C.TxIdx = 0;
						SwSlaveI2C.Cnt = 0;
						SwSlaveI2C.State = I2C_MODE_START;
						SDA_high();
					}
					break;
			}
			
		}                                             
	}                                                 
}
```

## 不足之处
RESTART信号经常判断不到.
通讯只能在低频稳定进行(20kHz以下).
从机超时处理方面不一定很完善.
