﻿# test--torch
      
/*----------------------摘要描述---------------------------------
*      	   	   	P54 1ms翻转  P53输出PWM  P00唤醒 P40.P41.VDDAD采集

   	   	   	   	注意 7031不能直接读IO某一位
   	   	   	   	   	如 if(P54D==x)
   	   	   	   	   	   	P54D=!P54D;
   	   	   	   	   	改为 将整组IO读出到一个变量
   	   	   	   	   	   	buff=IOP5;
   	   	   	   	   	   	在判断buff某一位
******************************************************************************/

#include "MC32P7031.h"
#include "user.h"

/************************************************
;  *    @Function Name       : CLR_RAM
;  *    @Description         : 上电清RAM
;  *    @IN_Parameter        :
;  *    @Return parameter    :
;  ***********************************************/
void CLR_RAM(void)
{
   	__asm
   	movai 0x7F
   	movra FSR0
   	clrr FSR1
   	clrr INDF
   	DJZR FSR0
   	goto $ -2
   	clrr INDF
   	__endasm;
}
/************************************************
;  *    @Function Name       : IO_Init
;  *    @Description         : IO初始化
;  *    @IN_Parameter      	 :
;  *    @Return parameter    :
;  ***********************************************/
void IO_Init(void)
{
   	IOP0 = 0x00; //io口数据位
   	OEP0 = 0xEB; //io口方向 1:out  0:in P02,P04输入 P00 01 03 输出
   	PUP0 = 0x04; //io口上拉电阻   1:enable  0:disable P02 上拉

   	IOP4 = 0x00;  //io口数据位
   	OEP4 = 0xE6;  //io口方向 1:out  0:in      	1110 0110
   	PUP4 = 0x18;  //io口上拉电阻   1:enable  0:disable P44 43上拉
   	ANSEL = 0x01; //io类型选择  1:模拟输入  0:通用io

   	IOP5 = 0x00; //io口数据位
   	OEP5 = 0xfb; //io口方向 1:out  0:in P52 change to input
   	PUP5 = 0x00; //io口上拉电阻   1:enable  0:disable
}
/************************************************
;  *    @Function Name       : TIMER0_INT_Init
;  *    @Description         :
;  *    @IN_Parameter        :
;  *    @Return parameter    :
;  ***********************************************/
void TIMER0_INT_Init(void)
{
   	TXCR = 0x08; //T0时钟 T0 T2 Fcpu  T1 Fhosc 禁止T0唤醒

   	T0CR = 0x94; //内部 128分频  允许自动重载
   	T0C = 211; //350ms //128 = 1s
   	T0D = 211; //350ms //128 = 1s
   	T0IF = 0;
   	T0IE = 1;
}
/************************************************
;  *    @Function Name       : TIMER1_PWM_Init
;  *    @Description         :
;  *    @IN_Parameter        :
;  *    @Return parameter    :
;  ***********************************************/
void TIMER1_PWM_Init(void)
{
   	T1CR = 0xA1;
   	T1C = 0;
   	T1D = 25;
}
/************************************************
;  *    @Function Name       : ADCinit
;  *    @Description         : ADC初始化
;  *    @IN_Parameter      	 :
;  *    @Return parameter    :
;  ***********************************************/
void ADCinit(void)
{
   	ANSEL |= 0x01; //P45  模拟输入  1 模拟输入  0  IO口
   	ADCR &= 0x00;
   	ADCR |= 0x3E; //0011 1110
   	VREF = 0x00; //设置内部参考电压 为2.0V
   	ADRL &= 0x80;
   	ADRL |= 0x60; //adc时钟  Fcpu/2
}
/************************************************
;  *    @Function Name       : Sys_Init
;  *    @Description         : 系统初始化
;  *    @IN_Parameter      	 :
;  *    @Return parameter    :
;  ***********************************************/
void Sys_Init(void)
{
   	//GIE = 0;
   	CLR_RAM();
   	WDTR = 0x5A;
   	IO_Init();
   	TIMER0_INT_Init();
   	//TIMER1_PWM_Init();
   	ADCinit();
   	GIE = 1;
   	//MCLRE = 1; //开始外部复位! 只能在烧录时候配置
}
/************************************************
;  *    @Function Name       : ADC_Get_Value
;  *    @Description         : ADC单次转换
;  *    @IN_Parameter      	 : CHX  ADC通道
;  *    @Return parameter    : 该通道ADC的值
;  ***********************************************/
uint ADC_Get_Value(uchar CHX)
{
   	ADCR = (ADCR & 0x78) | CHX; // ADC 使能  AD转换通道开启  通道  CHX

   	if (changl_Num != CHX) //通道切换了  舍去
   	{
   	   	changl_Num = CHX;
   	   	if(ADON){
   	   	ADEOC = 0;
   	   	ADST = 1; // 开始转换
   	   	WDTR = 0x5A;
   	   	while (ADEOC == 0)
   	   	{ // 检查EOC的标志位   等待转换完毕
   	   	}
   	   	}
   	}
   	if(ADON){
   	   	ADEOC = 0;
   	   	ADST = 1; // 开始转换
   	   	WDTR = 0x5A;
   	   	while (ADEOC == 0)
   	   	{ // 检查EOC的标志位   等待转换完毕
   	   	}

   	   	WDTR = 0x5A;
   	   	ADC_temp_value1 = ADRH;
   	   	ADC_temp_value1 = ADC_temp_value1 << 4 | (ADRL & 0x0f);
   	}
   	return ADC_temp_value1; // 获得ADC转换的数据
}

/************************************************
;  *    @Function Name       : ADC_Get_Value_Average
;  *    @Description         : 连续转换
;  *    @IN_Parameter      	 : 通道
;  *    @Return parameter    : 返回ADC的值
;  ***********************************************/
uint ADC_Get_Value_Average(uchar CHX)
{
   	channel = 0;
   	tmpBuff = 0;
   	ADCMAX = 0;
   	ADCMIN = 0xffff;
   	ADCR = (ADCR & 0xf8) | CHX; // ADC 使能  AD转换通道开启  通道  CHX
   	for (channel = 0; channel < 8; channel++)
   	{
   	   	WDTR = 0x5A;
   	   	ADEOC = 0;
   	   	ADST = 1; // 开始转换
   	   	while (!ADEOC)
   	   	   	; //等待转换完成
   	   	ADC_temp_value = ADRH;
   	   	ADC_temp_value = ADC_temp_value << 4 | (ADRL & 0x0f);
   	   	if (channel < 2)
   	   	   	continue; //丢弃前两次采样的
   	   	if (ADC_temp_value > ADCMAX)
   	   	   	ADCMAX = ADC_temp_value; //最大
   	   	if (ADC_temp_value < ADCMIN)
   	   	   	ADCMIN = ADC_temp_value; //最小
   	   	tmpBuff += ADC_temp_value;
   	}
   	// WDTR = 0x5A;
   	tmpBuff -= ADCMAX; 	   	   	   	 //去掉一个最大
   	tmpBuff -= ADCMIN; 	   	   	   	 //去掉一个最小
   	ADC_temp_value = (tmpBuff >> 2); //除以4，取平均值
   	return ADC_temp_value;
}
void main(void)
{
   	WDTR = 0x5A;
   	Sys_Init();
   	WDTR = 0x5A;
   	IOP0 = 0;
    low_battery_flag= 0;
   	while (1)
   	{
   	   	WDTR = 0x5A;
   	   	CLKS=1;	   	//切换到低频
   	   	Nop();
   	   	HOFF=1;	   	//关闭高频
        Nop();
        Nop();
//LED button detect
//P44 = 0; button is pressed.
        SleepModeUpdate = 0; //add 230410
        WDTR = 0x5A;
   	   	IO_buff.byte = IOP4;
        if((IO_BIT4 == 0)&&(!low_battery_flag)){ //button pull down 0
            Edge_up = 1;
            //定时器计数，两次按键时间间隔<15s,切换模式，>15s则关灯，重新循环
            //timer0_count1 = 0; //220926 开启
   	   	   	//turn on 5V
   	   	   	IO_buff.byte = IOP0;
   	   	   	IO_BIT0 = 1;
   	   	   	IOP0 = IO_buff.byte; //send data to data register
   	   	   	Second=0; //start count
   	   	   	hour = 0; //开灯会清sleep时间
        }
        else{
            if (Edge_up) {
   	   	   	  WDTR = 0x5A;
              Edge_up = 0;
              if(timer0_count1 < 15){
                LEDmode ++;
              }
              else{
                LEDmode = 0;
              }
   	   	   	  timer0_count1 = 0;
            }
        }
//Flash, lantern and night light
//Control light mode
//P42=1;P41=0; flash light
//P42=0;P41=1; lartern light
//P42=1;P41=1;  night light
//P42=0;P41=0;  OFF
        WDTR = 0x5A;
        IO_buff.byte = IOP4;
        if(LEDmode==1){
          IO_BIT2 = 1;
          IO_BIT1 = 0;
        }
        else if(LEDmode==2){
          WDTR = 0x5A;
   	   	  IO_BIT2 = 0;
          IO_BIT1 = 1;
        }
        else if (LEDmode == 3) {
   	      WDTR = 0x5A;
   	   	  IO_BIT2 = 1;
          IO_BIT1 = 1;
        }
        else{
   	   	  WDTR = 0x5A;
          IO_BIT2 = 0;
          IO_BIT1 = 0;
          LEDmode = 0;
   	   	  timer0_count1 = 0;
        }
        IOP4 = IO_buff.byte; //send data to data register
        //add rev230301

        //if((!LEDmode)||(mp3_on)){
   	   	  if(battery_detect == 180){
   	   	    SleepModeCount++;
            ADON = 1; //短按触发 开启AD使能  （测试能否调整开机播放时灯迟钝的问题）
   	   	    battery_detect =0;
   	   	    ADC_num_value = ADC_Get_Value_Average(0);
            if((ADC_num_value < 2082)&&(ADC_num_value > 1755)){  //2.95 - 3.05V //update to 2.60V - 3.05 V 
              if(SleepModeCount==1){ //计算10次 后调整//改为1次
               SleepModeUpdate = 1;
               SleepModeCount = 0;
              }
            }
          }
        //}

// turn on light 2h after turn off light
        if(light_hour ==2)
        {
              LEDmode = 0;
              light_hour = 0;
        }

/***************************************************************************
MP3 BUTTON DEDECT
P02 PULL UP 100K
PRESS DOWN active
***************************************************************************/
      WDTR = 0x5A;
      IO_buff.byte = IOP0;
      if((IO_BIT2 == 0)&&(!low_battery_flag)){
        P52OE = 1; //1 is output; 0 is input
        mp3_button = 1;
   	   	power_led_on = 0;
        timer_count++;
   	   	//turn on 5V
   	   	IO_buff.byte = IOP0;
   	   	IO_BIT0 = 1;
   	   	IOP0 = IO_buff.byte; //send data to data register
   	   	Second=0; //start count
   	   	hour = 0; //mp3 按键也会清sleep时间
   	   	ADON = 1; //短按触发 开启AD使能
   	   	ADC_num_value = ADC_Get_Value_Average(0);
      }
      else{
        if(mp3_button == 1){
   	   	  timer_count =0;
   	   	  mp3_button = 0;
        }
      }
   	  WDTR = 0x5A;
      if(timer_count == 5){
        mp3_on = !mp3_on;
        ad_count = 0;
      }
      if(mp3_on){
        IO_buff.byte = IOP0;
        IO_BIT3 = 1;
        IOP0 = IO_buff.byte;
        /***************************************************************************
                    AD check batt voltage < 2.5V turn off 5V
        ***************************************************************************/
        ad_count++;
        if(ad_count ==60){// fix from 20 to 60 ,一分钟检测一次
          ad_count = 0;
          ADON = 1; //短按触发 开启AD使能
       	   	ADC_num_value = ADC_Get_Value_Average(0);
          if((ADC_num_value < 1782)&&(ADC_num_value > 1707)){  //2.5 - 2.60V
            mp3_on = 0;
            IO_buff.byte = IOP0;
            IO_BIT0 = 0;
            IOP0 = IO_buff.byte; //send data to data register
            low_battery_flag = 1;
          }

        }
      }
      else{
   	   	IO_buff.byte = IOP0;
        IO_BIT3 = 0;
        IOP0 = IO_buff.byte;
      }
   	  if(timer_count ==20) timer_count = 10;

/***************************************************************************
      AD and Power led display
***************************************************************************/
#if 1
      WDTR = 0x5A;
   	if((!DCin)&&(power_led_on<5)){ //充电时候不只执行！
        if(ADC_num_value > 2287){ //3.35V
          WDTR = 0x5A;
   	   	  //D11 D12 D8 D10 D9
   	   	  IO_buff.byte = IOP4;
          IO_BIT5 = 1; //D12
          IOP4 = IO_buff.byte;

   	   	  IO_buff.byte = IOP0;
          IO_BIT1 = 1; //D10
          IOP0 = IO_buff.byte;
   	   	  WDTR = 0x5A;
          IO_buff.byte = IOP5;
          IO_BIT2 = 1; 	//D11
          IO_BIT3 = 1;  //D8
          IO_BIT4 = 1; 	//D9
          IOP5 = IO_buff.byte;
        }
        else if(ADC_num_value > 2253){ //3.30V
          WDTR = 0x5A;
   	   	  //D11 D12 D8 D10 D9
   	   	  IO_buff.byte = IOP4;
          IO_BIT5 = 1; //D12
          IOP4 = IO_buff.byte;

   	   	  IO_buff.byte = IOP0;
          IO_BIT1 = 1; //D10
          IOP0 = IO_buff.byte;
   	   	  WDTR = 0x5A;
          IO_buff.byte = IOP5;
          IO_BIT2 = 1; 	//D11
          IO_BIT3 = 1;  //D8
          IO_BIT4 = 0; 	//D9
          IOP5 = IO_buff.byte;
        }
        else if(ADC_num_value > 2219){ //3.25V
          WDTR = 0x5A;
   	   	  //D11 D12 D8 D10 D9
   	   	  IO_buff.byte = IOP4;
          IO_BIT5 = 1; //D12
          IOP4 = IO_buff.byte;

   	   	  IO_buff.byte = IOP0;
          IO_BIT1 = 0; //D10
          IOP0 = IO_buff.byte;
   	   	  WDTR = 0x5A;
          IO_buff.byte = IOP5;
          IO_BIT2 = 1; 	//D11
          IO_BIT3 = 1;  //D8
          IO_BIT4 = 0; 	//D9
          IOP5 = IO_buff.byte;
        }
        else if(ADC_num_value > 2185){ // >3.20V
          WDTR = 0x5A;
   	   	  //D11 D12 D8 D10 D9
   	   	  IO_buff.byte = IOP4;
          IO_BIT5 = 1; //D12
          IOP4 = IO_buff.byte;

   	   	  IO_buff.byte = IOP0;
          IO_BIT1 = 0; //D10
          IOP0 = IO_buff.byte;
   	   	  WDTR = 0x5A;
          IO_buff.byte = IOP5;
          IO_BIT2 = 1; 	//D11
          IO_BIT3 = 0;  //D8
          IO_BIT4 = 0; 	//D9
          IOP5 = IO_buff.byte;
        }
        else if(ADC_num_value > 1707){ // >2.5V

          WDTR = 0x5A;
   	   	  //D11 D12 D8 D10 D9
   	   	  IO_buff.byte = IOP4;
          IO_BIT5 = 0; //D12
          IOP4 = IO_buff.byte;

   	   	  IO_buff.byte = IOP0;
          IO_BIT1 = 0; //D10
          IOP0 = IO_buff.byte;
   	   	  WDTR = 0x5A;
          IO_buff.byte = IOP5;
          IO_BIT2 = 1; 	//D11
          IO_BIT3 = 0;  //D8
          IO_BIT4 = 0; 	//D9
          IOP5 = IO_buff.byte;
        }
        else{
          WDTR = 0x5A;
   	   	  IO_buff.byte = IOP4;
          IO_BIT5 = 0; //D12
          IOP4 = IO_buff.byte;
   	   	  
//TEST////////////////////////////////////////////////////

   	   	  IO_buff.byte = IOP0;
          IO_BIT1 = 0; //D10
          IOP0 = IO_buff.byte;


   	   	  WDTR = 0x5A;
          IO_buff.byte = IOP5;
          IO_BIT2 = 0; 	//D11
          IO_BIT3 = 0;  //D8
          IO_BIT4 = 0; 	//D9
          IOP5 = IO_buff.byte;
        }
   	  }

   	  WDTR = 0x5A;


#endif
/***************************************************************************
5s later turn off power LED
***************************************************************************/
      WDTR = 0x5A;
      if(power_led_on >= 5){
        if(!DCin){
          ADON = 0; //turn off the AD
   	   	  ADC_num_value = 0;
          WDTR = 0x5A;
   	   	  IO_buff.byte = IOP4;
          IO_BIT5 = 0; //D12
          IOP4 = IO_buff.byte;

   	   	  IO_buff.byte = IOP0;
          IO_BIT1 = 0; //D10
          IOP0 = IO_buff.byte;
   	   	  WDTR = 0x5A;
          IO_buff.byte = IOP5;
          IO_BIT2 = 0; 	//D11
          IO_BIT3 = 0;  //D8
          IO_BIT4 = 0; 	//D9
          IOP5 = IO_buff.byte;
          P52OE = 0; //1 is output; 0 is input
        }
      }
/***************************************************************************
      Control and charge led display
***************************************************************************/
   	   	WDTR = 0x5A;
        IO_buff.byte = IOP4;
        if(IO_BIT3 == 0){ //P43充电检测，充电时候 低
          low_battery_flag = 0;
          P52OE = 1; //1 is output; 0 is input
   	   	  IOP4 = IO_buff.byte;
          DCin = 1;
          ADON = 1; //开启AD使能
   	   	  ADC_num_value = ADC_Get_Value_Average(0);
        }
   	   	else {
          DCin = 0;
          if(power_led_on < 5){
            P52OE = 1; //1 is output; 0 is input
          }
          else{
            P52OE = 0; //1 is output; 0 is input
          }

        }
   	   	WDTR = 0x5A;
        if(DCin){
          if (ADC_num_value > 2396) { //3.59V //update 3.55 to 3.61 //update 3.61 to 3.59//Update to 3.50V
          WDTR = 0x5A;
   	   	  IO_buff.byte = IOP4;
          IO_BIT5 = 1; //D12
          IOP4 = IO_buff.byte;

   	   	  IO_buff.byte = IOP0;
          IO_BIT1 = 1; //D10
          IOP0 = IO_buff.byte;
   	   	  WDTR = 0x5A;
          IO_buff.byte = IOP5;
          IO_BIT2 = 1; 	//D11
          IO_BIT3 = 1;  //D8
          IO_BIT4 = 1; 	//D9
          IOP5 = IO_buff.byte;
          }
          else{
   	   	   	WDTR = 0x5A;
   	   	   	//timer_1s++;
   	   	   	//if(timer_1s < 2){
              switch(timer_1s){
              case 0:
                IO_buff.byte = IOP4;
                IO_BIT5 = 0; //D12
                IOP4 = IO_buff.byte;

                IO_buff.byte = IOP0;
                IO_BIT1 = 0; //D10
                IOP0 = IO_buff.byte;
                WDTR = 0x5A;
                IO_buff.byte = IOP5;
                IO_BIT2 = 1;   	//D11
                IO_BIT3 = 0;  //D8
                IO_BIT4 = 0;   	//D9
                IOP5 = IO_buff.byte;
                break;
              case 1:
                IO_buff.byte = IOP4;
                IO_BIT5 = 1; //D12
                IOP4 = IO_buff.byte;

                IO_buff.byte = IOP0;
                IO_BIT1 = 0; //D10
                IOP0 = IO_buff.byte;
                WDTR = 0x5A;
                IO_buff.byte = IOP5;
                IO_BIT2 = 1;   	//D11
                IO_BIT3 = 0;  //D8
                IO_BIT4 = 0;   	//D9
                IOP5 = IO_buff.byte;
                break;
                case 2:
                  IO_buff.byte = IOP4;
                  IO_BIT5 = 1; //D12
                  IOP4 = IO_buff.byte;

                  IO_buff.byte = IOP0;
                  IO_BIT1 = 0; //D10
                  IOP0 = IO_buff.byte;
                  WDTR = 0x5A;
                  IO_buff.byte = IOP5;
                  IO_BIT2 = 1; 	//D11
                  IO_BIT3 = 1;  //D8
                  IO_BIT4 = 0; 	//D9
                  IOP5 = IO_buff.byte;
                  break;
                  case 3:
                    IO_buff.byte = IOP4;
                    IO_BIT5 = 1; //D12
                    IOP4 = IO_buff.byte;

                    IO_buff.byte = IOP0;
                    IO_BIT1 = 1; //D10
                    IOP0 = IO_buff.byte;
                    WDTR = 0x5A;
                    IO_buff.byte = IOP5;
                    IO_BIT2 = 1;   	//D11
                    IO_BIT3 = 1;  //D8
                    IO_BIT4 = 0;   	//D9
                    IOP5 = IO_buff.byte;
                    break;
                    case 4:
                      IO_buff.byte = IOP4;
                      IO_BIT5 = 1; //D12
                      IOP4 = IO_buff.byte;

                      IO_buff.byte = IOP0;
                      IO_BIT1 = 1; //D10
                      IOP0 = IO_buff.byte;
                      WDTR = 0x5A;
                      IO_buff.byte = IOP5;
                      IO_BIT2 = 1; 	//D11
                      IO_BIT3 = 1;  //D8
                      IO_BIT4 = 1; 	//D9
                      IOP5 = IO_buff.byte;
                      break;
              }
              /*if(timer_1s == 1){
                  WDTR = 0x5A;
           	   	  IO_buff.byte = IOP4;
                  IO_BIT5 = 1; //D12
                  IOP4 = IO_buff.byte;

           	   	  IO_buff.byte = IOP0;
                  IO_BIT1 = 1; //D10
                  IOP0 = IO_buff.byte;
           	   	  WDTR = 0x5A;
                  IO_buff.byte = IOP5;
                  IO_BIT2 = 1; 	//D11
                  IO_BIT3 = 1;  //D8
                  IO_BIT4 = 1; 	//D9
                  IOP5 = IO_buff.byte;
            }
            else{
   	   	   	  //if(timer_1s == 2) timer_1s = 0;
                  WDTR = 0x5A;
           	   	  IO_buff.byte = IOP4;
                  IO_BIT5 = 0; //D12
                  IOP4 = IO_buff.byte;

           	   	  IO_buff.byte = IOP0;
                  IO_BIT1 = 0; //D10
                  IOP0 = IO_buff.byte;
           	   	  WDTR = 0x5A;
                  IO_buff.byte = IOP5;
                  IO_BIT2 = 0; 	//D11
                  IO_BIT3 = 0;  //D8
                  IO_BIT4 = 0; 	//D9
                  IOP5 = IO_buff.byte;
            }*/
          }

        }


        /*
        OSCM &= 0xE7; //进入休眠 系统时钟切换到低频振荡器 CLKS=1 HOFF=1
        OSCM |= 0x08; //进入休眠 高速
   	   	Nop();
   	   	Nop();
   	   	Nop();*/
   	    //GIE = 1; //开启中断使能
   	   	//PWM1OE=1; //开启PWM1 P53
   	   	//TC1EN=1;  //开启T1定时器
        //ADON = 1; //开启AD使能
//turn off 5V
        IO_buff.byte = IOP5;
        if(IO_BIT2 == 1){
          mp3_det++;
          if((mp3_on)&&(mp3_det==3)){
            IO_buff.byte = IOP0;
            IO_BIT3 = 0;
            IOP0 = IO_buff.byte;
            mp3_on = 0;
            mp3_det = 0;
          }
        }
        if(SleepModeUpdate){
          if(hour == 4) { //11*0.35 = 3.85 hours//change to 1 for test
              IO_buff.byte = IOP0;
              IO_BIT0 = 0;
              IOP0 = IO_buff.byte; //send data to data register
              Second=0; //start count
              hour = 0;
              SleepModeUpdate = 0;
   	   	   	  ADON = 0;
              mp3_on = 0;
          }
        }
        else{
          if(hour == 48) { //138*0.35 = 48 hours
              IO_buff.byte = IOP0;
              IO_BIT0 = 0;
              IOP0 = IO_buff.byte; //send data to data register
              Second=0; //start count
              hour = 0;
   	   	   	  ADON = 0;
              mp3_on = 0;
          }
        }  
   	   }
}

void int_isr(void) __interrupt
{

   	__asm
   	push
   	__endasm;
   	WDTR = 0x5A;
   	//=======T0========================

   	if (T0IF && T0IE)
   	{
   	   	T0IF = 0;
   	   	//TEST
   	   	//IO_buff.byte = IOP0;
        //IO_BIT1 = !IO_BIT1; //D10
        //IOP0 = IO_buff.byte;
   	   	Second++;
   	   	battery_detect++;
   	   	if(Second == 3600) {   //test=30
            Second = 0;
            hour++;
        }

   	   	//timer_1s = !timer_1s;
        if(DCin)
        {
          timer_1s++;
          if(timer_1s>4) timer_1s=0;

        }
        if(LEDmode != 0) timer0_count1++;
        power_led_on++;
//reset light 2hours
        if(timer0_count1==3600) { //test 10s
          light_hour++;
          timer0_count1 = 0;
        }
/*     	   	if (timer0_count1 >= 30) //防止溢出
   	   	{
   	   	   	timer0_count1 = 20;
   	   	}
*/
        if(power_led_on == 10){
          power_led_on = 8;
        }
   	}
   	//=======ADC=======================
   	if (ADIF && ADIE)
   	{
   	   	ADIF = 0;

   	}
   	__asm
   	pop
   	__endasm;
}
