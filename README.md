# VL6180s-ZYNQdrive
由ZYNQ使用IIC驱动多个VL6180例程<br>
所使用模块为：<br>
![baidu](https://github.com/magua-404/VL6180s-ZYNQdrive/blob/main/picture/VL6180%E8%AE%BE%E5%A4%87%E7%BB%84ZYNQ%E9%A9%B1%E5%8A%A8%E6%B5%8B%E8%AF%95-1.HEIC)
VL6180模块本身是可以自定义设备地址的，但是由于其掉电后设备地址会自动恢复为默认的0x29，所以如何在上电后为串接在IIC总线上的多个VL6180分配独立地址是串接多个VL6180所需要解决的问题。<br>
本例程通过上电后依次激活VL6180的GPIO1引脚，来逐个对VL6180的地址进行配置。（GPIO1引脚为传感器的关闭引脚，默认情况下它被拉高，当它被拉低时芯片停止工作）。<br>
<br>
以3个VL6180串联为例，系统上电后将3个GPIO1引脚分别配置为1,0,0,这样IIC总线上只能检测到一个VL6180，然后将其设备地址由0x29改为新的地址,例如0x30；<br>
之后将GPIO引脚配置为1,1,0，激活第二个VL6180，此时他的地址为默认值0x29，然后将其改为新值，例如0x31；<br>
最后将GPIO引脚配置为1,1,1，激活第三个VL6180，将其由默认地址0x29改为0x32.<br>
然后可以通过0x30、0x31、0x32分别对三个设备进行操作。<br>
<br>
本例程中以两个VL6180为例，使用ZYNQ7020芯片对其进行驱动，函数功能的具体实现在vl6180drive.c中，主程序main.c如下:<br>
```c
/***************************** Include Files *********************************/
#include <stdio.h>
#include "xparameters.h"
#include <xgpiops.h>
#include "xscugic.h"
#include "xil_exception.h"
#include "xil_printf.h"
#include "xplatform_info.h"
#include "sleep.h"

#include "vl6180drive.h"
/************************** Variable Definitions *****************************/
#define GPIOPS_ID XPAR_XGPIOPS_0_DEVICE_ID   //PS端  GPIO器件 ID
#define EMIO_CS1 54  //片选引脚1 连接到EMIO0
#define EMIO_CS2 55  //片选引脚2 连接到EMIO1

	u8 distanceBuff[1];
	u8 distance_out[2];
/************************** Function Definitions *****************************/
//主函数调用了Iic EEPROM interrupt example函数
int main(void)
{
	/************************** 初始化io引脚配置 *****************************/
    XGpioPs gpiops_inst;            //PS端 GPIO 驱动实例
    XGpioPs_Config *gpiops_cfg_ptr; //PS端 GPIO 配置信息

    //根据器件ID查找配置信息
    gpiops_cfg_ptr = XGpioPs_LookupConfig(GPIOPS_ID);
    //初始化器件驱动
    XGpioPs_CfgInitialize(&gpiops_inst, gpiops_cfg_ptr, gpiops_cfg_ptr->BaseAddr);

    //设置两个EMIO_CS为输出引脚
    XGpioPs_SetDirectionPin(&gpiops_inst, EMIO_CS1, 1);
    XGpioPs_SetDirectionPin(&gpiops_inst, EMIO_CS2, 1);
    //使能两个EMIO_CS输出
    XGpioPs_SetOutputEnablePin(&gpiops_inst, EMIO_CS1, 1);
    XGpioPs_SetOutputEnablePin(&gpiops_inst, EMIO_CS2, 1);


    /************************** 初始化VL6180设备组 *****************************/
	int Status;
	Status = IIC_inital();								//初始化IIC
	XGpioPs_WritePin(&gpiops_inst, EMIO_CS1,1);			//配置第一个VL6180
	XGpioPs_WritePin(&gpiops_inst, EMIO_CS2,0);
	usleep(10000);          //等待10ms
	Status = vl6180_drive_inital();						//初始化第一个vl6180
	Status = vl6180_drive_ChangeDeviceAddress(0x30);	//将第一个vl6180的设备地址由默认值改为新值
	if (Status != XST_SUCCESS) {
		xil_printf("第一个设备地址更改失败\n");
	}
	else{
		xil_printf("第一个设备地址更改成功\n");
	}

	usleep(100000);          //等待100ms

	Status = IIC_inital();
	XGpioPs_WritePin(&gpiops_inst, EMIO_CS1,1);			//配置第二个VL6180
	XGpioPs_WritePin(&gpiops_inst, EMIO_CS2,1);
	usleep(10000);          //等待10ms
	Status = vl6180_drive_inital();						//初始化第二个vl6180
	Status = vl6180_drive_ChangeDeviceAddress(0x31);	//将第二个vl6180的设备地址由默认值改为新值
	if (Status != XST_SUCCESS) {
		xil_printf("第二个设备地址更改失败\n");
	}
	else{
		xil_printf("第二个设备地址更改成功\n");
	}

	/************************** 循环测量距离 *****************************/

	while(1)
	{
		Status = vl6180_drive_returnDistance(distanceBuff,0x30);
		distance_out[0] = distanceBuff[0];
		usleep(10000);          //等待10ms
		Status = vl6180_drive_returnDistance(distanceBuff,0x31);
		distance_out[1] = distanceBuff[0];
		xil_printf("The first distance is:%d; The second distance is:%d\n",distance_out[0],distance_out[1]);
		usleep(10000);          //等待10ms
	}
}
```
主程序中包括：ZYNQ芯片的EMIO初始化（用以控制VL6180的GPIO1引脚）、根据上文提到的逻辑依次对每个VL6180进行初始化以及设备地址更改、主循环中循环采集每个VL6180的数据。<br>
<br>
