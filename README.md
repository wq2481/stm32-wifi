# 基于STM32F103C8T6的OLED**数字钟的设计与实现** #





  作者：              颜孙炜

------------------ ----------------------------------------------------

------

[TOC]



# 概述 

![image-20230110223524043](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20230110223524043.png)

用STM32F103C8T6自有的RTC功能实现一款数字钟的设计，包括温度输入检测和显示模块、数字钟显示模块、定时提醒模块等。完成了，万年历，计时器，温度检测，电压监测等功能，此外随着科技的进步，物联网技术获得了飞速的发展，其已渗透到当前的多个行业领域。而温度作为我们生活中非常重要的一个环境参数，其变化情况会给我们的生产生活造成关键性的影响，这就要求我们必须做好温度的实时监测，本产品在完善其他功能的前提下,通过STM32监测温度并通过物联网系统储存温度检测的数据，并加以计算和显示在网页上。

# 项目需求

此项目旨在开发一款，集万年历，温度检测，数字显示，且较为智能的产品

故罗列出以下开发需求

1.  具有12/24小时切换的万年历

2.  通过蓝牙及手机APP进行时间的设置

3.  具有温度检测的功能

4.  数据可以在云端储存和查看

# **开发环境**

硬件：STM32F103C8T6，ESP8266，BT04-E蓝牙模块

服务器：华为云

WEB:FLASK框架

数据库:SQLITE3

开发工具及版本：KEIL 5，ARDUINOIDE，VSCODE,STM32CUBEMX

调试工具：SSCOM5.13.1，APIPOST-POSTMAN

# ![image-20230110223740107](概述 {#概述-1}.assets/image-20230110223740107.png)

## 主控芯片

本项目采用的芯片为 STM32F103C8T6 STM32F103C8T6是一款基于ARM CORTEX-M 内核STM32系列的32位的微控制器，程序存储器容量是64KB，需要电压2V\~3.6V，工作温度为-40°C \~ 85°C。本项目使用 HAL 库编写,在芯片的资源利用上，采用了 IO 高低电平输出控 ISP驱动OLED，串口 3 将数据发送至WIFI 模块通信。通用定时器TIM2用于定时中断，设置高速外部时钟HSE 选择外部时钟源。定时器的时钟频率为16MHZPRTSCALER (定时器分频系数)16COUNTER MODE(计数模式)UP(向上计数模式)COUNTER PERIOD(自动重装载值)1000CKD(时钟分频因子)NO DIVISION 不分频 定时器溢出时间： TOUT = (16\*1000)/16/1000/1000=0.5S

## 传感层搭建

温度作为我们生活中非常重要的一个环境参数, 其变化情况会给我们的生产生活造成关键性的影响, 这就要求我们必须做好温度的实时监测。热敏电阻是一种传感器电阻，其电阻值随着温度的变化而改变。按照温度系数不同分为正温度系数热敏电阻（ptc thermistor，即 positive temperature coefficient thermistor）和负温度系数热敏电阻（ntc thermistor，即 negative temperature coefficient thermistor）。正温度系数热敏电阻器的电阻值随温度的升高而增大，负温度系数热敏电阻器的电阻值随温度的升高而减小，它们同属于半导体器件。本项目温度检测采用10K热敏电阻，通过ADC1通道1（PA1）进行转换变为具体数值。

## 通信部分 

WIFI通信部分采用 ESP8266 ，ESP8266 是一款适用于物联网和家庭自动化项目的 WI-FI 模块。本项目将主控芯片STM32获取的数据处理后通过串口3发送至ESP8266，ESP8266再将数据通过WIFI发送至云端。同时为了减少遗漏，防止数据丢包漏包，ESP8266接收数据的间隔需要得到控制，使得数据完整被发送至云端。蓝牙通信选择BT04-E蓝牙模块，BT04-E蓝牙模块可通过与单片机的串口相连，借助电脑或手机的蓝牙与单片机实现异步全双工通信。此项目中我们采用该蓝牙模块将APP与主控芯片进行数据的传输。从而实现手机端对设备的控制

# **软件框架**

![image-20230110223755155](概述 {#概述-1}.assets/image-20230110223755155.png)

# **功能开发**

## 主函数设计

整体框架采用循环加switch函数,每当设备运行的时候，程序进入主函数先进行所有功能以及变量的初始化，然后进入while循环，并且进入switch函数,并在中断中执行按键的检测，用于控制进入switch函数的各个分支。Switch各分支实现不同的功能，如果在进入switch函数时刷新界面，则会造成屏幕的闪烁，本项目采用在中断检测按键触发时刷新屏幕，当按键按下切换功能时，进行屏幕的刷新，使屏幕显示更加稳定。

## 万年历设计

### 显示时间功能

为了使oled上能显示时间并时刻更新，我们采用RTC时钟进行该功能的开发，实时时钟芯片是日常生活中应用最为广泛的消费类电子产品之一。它为人们提供精确的实时时间,或者为电子系统提供精确的时间基准。在设置好RTC后RTC每次中断都会产生一组新的时间数据供开发者使用，初始化系统时钟RTC时钟第一次设置RTC(查询标记)配置RTC(产生秒中断)设置当前时间(年/月/日/小时/分/秒)以此作为RTC计数器的初值计算并显示时间秒中断循环，该功能较为简单，只需简单的配置加显示即可，配置及显示方案如下：

#### 配置：

在CUBEMX中将时钟源RCC设为外部中断

在RTC时钟中打开ACTIVATE CLOCK SOURCE以及ACTIVATE CALENDAR，同时打开中断

对于初始时间可以不进行设置，后续使用蓝牙进行设置。

#### 软件开发：

在配置完环境后在主函数中添加HAL_RTC_GETTIME，HAL_RTC_GETDATE函数，该函数可以在每次中断时更新GETTIME与GETDATE的值,在后续代码中，通过\".\"访问HOURS,MONTH等储存时间的变量。

为了实现万年历的时钟功能，只需要以\"时:分\"格式将数据显示在OLED屏上即可。主要代码如下：

```c
OLED_ShowNum(0, 20, GetTime.Hours, 2, 16);
OLED_ShowString(16, 20, (unsigned char *)":", 16);
OLED_ShowNum(20, 20, GetTime.Minutes, 2, 16);
```

OLDE显示时间X:Y

### 2.12/24小时转换功能

为了使产品与用户之间产生良好的人机交互，对于时钟类产品，满足各类用户的需求，应当具有转换12/24小时制的功能，其原理是按键检测，当检测到按键按下，运行12/24转换函数，显示转换后的时间如13：02转换后为1：02,该功能需要对按键进行配置与扫描，然后对时制进行转化，方案如下：

#### 配置：

在CUBEMX中将按键设置为GPIO_INPUT，作为输入。

#### 扫描：

在该功能中，需要实现按键按下切换为12小时制，抬起切换为24小时制。仅用独立按键就可以实现.由于按键按下会产生抖动，可以使用DELAY进行消抖。

按键检测代码如下：

```c
if (HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_8) == GPIO_PIN_RESET) {
					HAL_Delay():
					if (HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_8) == GPIO_PIN_RESET) {}}
```



#### 时制转化:

先判断输入的24小时制是属于AM时间段的还是PM时间段的。

如果是AM，就不用改变小时的数据；如果是PM的，就需要换算小时的数据

换算小时时仅需将24小时制减去12即为12小时制时间。此外24点需要额外进行考虑，在英文的习惯中，中午12点被认为是下午，所以24小时制的12:00就是12小时制的12:0 PM；而0点被认为是第二天的时间，所以是0:0 AM

只需要将转化后的变量代替原来的GETTIME.HOUR即可实现12/24小时的转化。

## 电压检测：

为了利用ADC资源以及为后续的温度采集功能做出铺垫，并且能观测到电压变化的趋势，项目中使用ADC对电压进行采集，并且将电压值处理为可观测的坐标形式设计了以下功能，PA1输入电压，STM32内部AD进行转换，通过函数计算为电压值，并显示在OLED屏上，将AD转换的数值进行处理为OLED Y轴坐标，X轴为时间，即可模拟出电压变化的波形。该功能需对ADC进行配置，软件得到ADC值后转为电压在OLED上显示，并通过对ADC值的处理进行波形的绘制，方案如下：

### 配置：

在CUBEMX中将时钟源RCC设为外部中断

打开ADC设置，本项目中打开的是ADC1的通道1。

ADC的其他配置默认即可。

最后还需设置HCLK，本项目设置的HCLK为16MHZ。

### ADC转化：

首先以中断方式启动ADC。在程序中通过ADC_VALUE = HAL_ADC_GETVALUE(&HADC1)获取AD值。其次确定基准电压，本项目中使用的主控芯片ADC的基准电压为3.3V，所以可以通过ADC_VALUE \* 3.3 / 4095计算出电压值并提供OLED加以显示。

### 波形绘制：

在波形绘制中，以X为时间轴，以Y为数据轴，但发现ADC值值过大，超过了OLED屏幕的尺寸，因此成像成为问题，想到ADC转化为电压值的方案，计划对ADC的值进行缩小。尝试对ADC值统一减小，产生如下图像

![image-20230110224151834](概述 {#概述-1}.assets/image-20230110224151834.png)

发现无法行成波形，原因是ADC值的个位与十位快速变化，因此对ADC数据进行35 + (ADC_VALUE - 1280) / 10处理，使ADC始终处于屏幕中线上下摆动，在电压值不发生大改变的情况下，可以观察到屏幕上随电压值改变产生曲线。

![image-20230110224203739](概述 {#概述-1}.assets/image-20230110224203739.png)

## **温度检测：**

搭建测温电路，测量10K热敏电阻的电压，通过公式T（℃）={（VN-VSENSE）/AVG_SLOPE}+N计算出温度,为了防止温度时刻变化，每一秒采集一次AD值并进行转换。在此功能中涉及传感电路的搭建，ADC的配置与上一个功能相同，不需要重复配置，但需要ADC值转为温度。

### 测温电路:

PA1为ADC1通道1引脚，本项目R1采用10KNTC热敏电阻，电路如下：

![image-20230110224230011](概述 {#概述-1}.assets/image-20230110224230011.png)

### ADC转温度：

通过公式T（℃）={（VN-VSENSE）/VSENSE}+N进行转化

这里：V N = V SENSE 在N °C时的数值AVG_SLOPE ＝ 温度与V SENSE 曲线的平均斜率(单位为MV/ °C 或 ΜV/ °C)N,VSENSE,VSENSE的典型值分别为25，1.43， 4.3MV/℃

只需将ADC值通过公式转化后显示在OLED上即可完成温度检测的功能。

##  跳跃游戏思路：

为了利用按键资源和熟悉oled屏幕的开发，项目中开发了一个模仿谷歌恐龙的跳跃游戏，其原理是，BMP1，BMP2储存游戏人物跳跃和落下的两种状态图片，用画直线和画方块函数绘制障碍，并定时刷新屏幕避免重影。设置两个变量，模拟障碍和人物坐标，当坐标相碰切任务处于落下状态时，游戏结束，蜂鸣器响，OLED显示\"GAME OVER\"，具体方案如下：

### 任务图形的绘制与取模：

这里我们使用PCTOLCD2002进行取模，图形的平移较为复杂，为了简单模拟跳起和落下两种状态，我们对两种状态的分别作图，分别进行显示。

当绘制完第一幅图后，为了不改变像素点间的位置，仅改变高度时，可以点击PCTOLCD2002上的上移按钮，上升合适的高度，得到第二幅图形

障碍的绘制：

障碍与人物图形不同，每一刻都在移动，即每一帧的障碍都是截然不同的，显然一帧一帧的绘制不仅麻烦，而且占用大量的单片机储存资源。因此，这里使用两种方法绘制障碍，一是通过坐标绘制线条，改变坐标改变每一帧的图形，二是通过平移图像来实现障碍的移动。本项目中采用的是第二种方法，我们需要获得四个点的坐标，并以此点为左上角点绘制正方形表示障碍，通过中断改变起始点的坐标，实现正方形的移动即障碍的移动，这里需要注意的是如何使障碍可以无限的循环下去，使障碍可以无限的循环下去需要对坐标进行计算

我们将两个屏幕进行拼接，一个屏幕长度为128，两个长度为128\*2=256

如图为了使障碍可以衔接，屏幕最前方后最后方应当各存在半个空白。

![image-20230110224322880](概述 {#概述-1}.assets/image-20230110224322880.png)

设需要3个障碍列出方程

5X+X/2\*2=128

X=21.3

故得到障碍的初始坐标

### 游戏逻辑控制：

虽然在屏幕是小人是静止的，但对于障碍来说小人是相对运动的，那么设置一个变量代表小人的运动轨迹，由于障碍的坐标不是随机的而是我们事先计算好的，所以我们可以判断小人坐标是否达到我们设置的障碍的坐标，如果相等，则代表小人与障碍在X方向上相遇，同时判断在Y方向上小人是否跳起，若小人没有跳起，则判断小人与障碍在X与Y方向上同时相遇，对于二维平面，可以以此判定小人与障碍相撞。则游戏结束。

这里给出功能代码如下：

```
OLED_DrawSquare((0 + start_point_x) % 127, 50, (10 + start_point_x) % 127, 60);
				OLED_DrawSquare((50 + start_point_x) % 127, 50, (60 + start_point_x) % 127, 60);
				OLED_DrawSquare((100 + start_point_x) % 127, 50, (110 + start_point_x) % 127, 60);

				if (jump_mark == 0) {
					OLED_ShowPicture(0, 0, 128, 8, BMP4);
				} else {
					OLED_ShowPicture(0, 0, 128, 8, BMP5);
					n++;
					if (n > 30) {
						jump_mark = 0;
						n = 0;
					}
				}
				speed++;
				if (speed > 5) {
					start_point_x = start_point_x + 2;
					speed = 0;
					OLED_Clear();
				}

				if (start_point_x == 128) {
					start_point_x = 0;
				}
				if (HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_8) == GPIO_PIN_RESET) {
					jump_mark = 1;
				}
				if (jump_mark == 0 && start_point_x % 65 == 0) {

					flag = 4;

				}
```



## 蓝牙时间修改：

在日常生活中发现对于传统的电子时钟，时间的校验和设置比较复杂和烦琐，因此本项目采用蓝牙对时间进行校验和设置，用户只需要切换到时间设置模式，并在app上连接蓝牙，即可对当前时间进行设置，开启串口2，按下KEY2与KEY3进入修改时间模式，在手机端APP上设置需要修改的时间，点击确定，OLED上显示修改后的时间，并继续计时。修改后的时间支持12/24转化。该功能分为APP端的开发和主控芯片软件的开发，方案如下：

### APP端：

APP端分为界面组件的开发，以及功能逻辑的开发

#### APP组件开发:

APP做使用的组件及功能如下表：

| 组件名称   | 功能               |
| ---------- | ------------------ |
| 列表选择框 | 选择蓝牙           |
| 标签       | 提示               |
| 时间选择框 | 获取时间，选择时间 |
| 按钮       | 提交               |
| 数字滑动条 | 数据选择           |
| 蓝牙客户端 | 提供蓝牙通信服务   |
| 计时器     | 计时               |

#### APP逻辑功能如下：

1.蓝牙连接：

当列表选择框准备选择时，执行设列表选择框的文本为蓝牙客户端的地址及名称，当列表选择框完成选择时执行设蓝牙客户瑞连接设备 地址为列表选择框的选中项。

当用户点击列表框就会显示蓝牙的列表，选择BT04-E蓝牙模块的蓝牙进行后APP会自动连接蓝牙。

1.  时间设置

- 当时间选择框完成时间设定时执行

  设时间选择框显示时间选择框的\"时\"+\"：\"加\"分\"。

  当用户点击时间选择按钮后会弹出时间选择器，当用户完成选择后，时间选择框的文本会变成选中的时间，供用户加以确定

2.  数据的发送

- 当按钮被按下时，蓝牙客户端发送数据，特别注意的是，数据的格式为时加分，

  由于在后续的蓝牙与主控芯片的开发中，对串口的进行了重定向，所以要在数据的末尾加上空格，并且由于是HAL库进行开发，串口接收的数据为字符，发送的数据应当文字符串，所以完整的数据格式应为，时加分加空格，型如\"1325 \"，\"0108 \"。

#### 配置：

蓝牙模块与主控芯片之间采用串口进行通信，首先连接好串口与主控芯片，连接方式如下表

| STM32F103 | BT04-E蓝牙模块 |
| --------- | -------------- |
| RX        | TX             |
| TX        | RX             |
| 5V        | 5V             |
| GND       | GND            |

连接完成后开始串口的配置,本项目蓝牙通信采用UART2即串口2进行通信。

启动串口2，将串口2的模式改为ASYNCHRONOUS,CUBEMX串口波特率默认为115200，但在与BT04-E蓝牙模块进行时,波特率需选择9600。

### 软件开发：

通过串口接收蓝牙从APP上获取的数据，处理后作为变量添加到RTC初始化函数中，通过RTC的初始化进行时间的重新设置。

HAL库中可以使用HAL_UART_TRANSMIT与HAL_UART_RECEIVE进行数据的收发，但HAL_UART_TRANSMIT与HAL_UART_RECEIVE接收变量的参数为UINT8_T数组，UINT8_T为无符号1个字节的整型，不能接收字节和数值,UINT8_T类似CHAR但不同，因此我们在这里利用FPUTS和FGETS的重定向对将参数类型改为CHAR数组接收字符串，那么数组中的每一个元素都是常见的CHAR变量。

对于如\"1，2，3，4..\"的CHAR类型我们通过格式化输出(\"%D\",X\[N\])将其转变为ASCII码,可以发现对于如\"1，2，3，4..\"的CHAR类型ASCLL码-48就是其代表的数值，我们从APP中收到的字符串以\"XYZW \"储存在CHAR类型数组中，我们按个十位取出并重新计算为一个数值

如((\"%D\",X\[0\])-48)\*10+(\"%D\",X\[1\])-48)代表XY所代表的数值

我们通过上述方法分别取得时和分的值，但在RTC初始化函数中，STIME所接收的变量类型为16进制数，因此我们还需要将所得的10进制数转为16进制数，才能被RCT初始化函数正确的接收。

按照以下的流程，我们就实现了通过蓝牙连接从而快速设置时间的功能

![image-20230110224605913](概述 {#概述-1}.assets/image-20230110224605913.png)



##  联网显示：

在上文中通过绘制电压温度的波形，可以观察到短时间内电压及温度变化的状态，那能否使数据更长久的进行储存和利用成为该功能的出发点，为了良好利用数据及储存数据，项目中使用ESP8266模块对数据进行上上传，并提供数据库对数据进行保存，通过web前后端对数据进行处理和显示，思路如下：开启串口3，为了不给服务器造成过大压力，在定时器中每隔15S秒发送一次数据到ESP8266，再由ESP8266将数据拼接到GET请求中，后端程序获取GET请求参数，并添加到数据库中，客户端页面显示将调用数据库中的信息并显示在设计好的页面上。此功能需要进行WEB端的开发，ESP8266的开发，以及主控芯片的开发。

### WED端开发：

WEB开发分为三个部分前端开发用于显示数据，后端开发负责数据库的连接操作，和数据的处理，接口开发用于给ESP8266提供通信的通道。

后端和接口的开发我们使用FLASK框架进行开发，FLASK是一个使用 PYTHON 编写的轻量级 WEB 应用框架。其 WSGI 工具箱采用 WERKZEUG ，模板引擎则使用 JINJA2 。本项目的数据较为单一简单，在数据库的设计上，我们采用SQLITE3数据库进行数据的储存。SQLITE，是一款轻型的数据库，是遵守ACID的关系型数据库管理系统，它包含在一个相对小的C库中。它是D.RICHARDHIPP建立的公有领域项目。它的设计目标是嵌入式的。在实际项目的开发中，发现flask对各个文件的划分有较为严格的规定

本项目flask的文件结构如下

├─static

└─templates

其中项目文件夹下存放数据库，项目启动文件，static文件夹下存放图片和组件库，template文件夹下放置HTML文件。

#### 数据库的设计

，我们这次所实现的是将单片机发送的温度值存入数据库，因此数据库结构较为简单

在文件目录下打开终端

输入SQLITE3

创建数据库.OPEN DATA.DB

通过CREATE TABLE TEMS(DAT TEXT NOT NULL);创建一个含有TEXT类型的DAT值的数据表TEMS。

#### 接口的设计

,ESP8266通过发送带参数的GET请求，向客户端发送请求，再接口设计中ROUTE()装饰器来告诉 FLASK 触发函数 的 URL，我们本项目中接口的URL为IP+/自定义字符/+\<参数\>

当接口接收到设备发送来的GET请求时，我们将GET请求的参数通过INSERT语句插入数据库中从而实现了一个简单接收储存数据的接口。

然后是显示页面后端开发，后端逻辑较为简单，连接数据库，通过SELECT\*FROM TEMS语句将所有的数据调用出来，CONN.ROW_FACTORY=SQLITE3.ROW和FETCHALL()函数可以使数据像PYTHON字典一样便于操作。

为了使数据可以利用起来，项目中以计算平均值来进行数据的利用，通过设计数据库的步骤我们可以看出数据库中DAT类型为TEXT类型，在PYTHON中为字符串STR类型，为了使DAT可以进行计算，我们需要对其类型进行转化，PYTHON 是弱类型语言,相对于强类型语言,它不需要声明变量的类型。那么在PYTHON之中，我们通过INT()函数直接对其进行转换

FOR DAT IN S:

N=N+INT(DAT\[\'DAT\'\])

N=N/LEN(S)

通过以上代码，我们遍历DAT字典中的每个值，并提供INT函数将其转化为整数类型，并进行平均值的计算，在前端中使用JINJA2语法将变量引入，即可实现温度均值的显示

#### WEB端前端页面的开发

，由于FALSK采用的JINJA2的模板引擎，所以再FLASK的前端开发中也要使用JINJA2语法，对于页面设计的HTML，CSS，JS代码依旧按照HTML等语言的规范。项目中使用了4个DIV分别作为顶端显示栏，侧边栏，主栏，底部显示栏。并在CSS中设置颜色.边框等属性。在主栏中，通过JINJA2的FOR循环构造出N个DIV，N代表温度数据的个数

仪式代码通过{%FOR%}和{%ENDFOR%}结构将结构内的所有语句重复多次，其中在\<H3\>{{DAT\[\'DAT\'\]}}\</H3\>中前一个DAT是我们从后端引入的的字典类型变量，而后一个DAT则是字典的键值，也是数据表中所设置的键名。

以上部分我们开发了一个可以接收数据储存到数据库，并且可以在前端显示出来的小型网页。

### ESP8266的开发：

ESP826612E可以使用AT指令进行控制，在本项目中我们使用了ESP8266的开发板，可以在ARDUINOIDE独立进行开发,更方便进行调试。

首先是对连接的WIFI进行设置，在代码中定义好WIFI的名称，密码，需要注意的是，WIFI名称最好为英文。

然后为了方便调试可以打开ESP8266的串口，在SETUP()函数中连接WIFI，建立通信

通信建立的步骤如下：

1.  创建HTTP的客户端

2.  通过BEGIN函数配置请求地址。

3.  由于在后端我们开发的接口通过GET请求接收数据，所以这里我们通过GET函数启动连接并发送HTTP请求

4.  发送GET请求，按照\"GET /USERS/\"+RX_BUFFER+\" HTTP/1.1\\R\\N\"+\"HOST: \"+HOST+\"\\R\\N\"+\"CONNECTION: CLOSE格式像接口发送GET请求，其中RX_BUFFER代表我们通过串口从主控芯片获取的变量，在本项目中表示温度。

-   按照以上方案，形成了ESP8266向网址发送数据的功能

### 主控芯片的开发：

这里主控芯片采用的UART3即串口3进行通信,主控芯片与ESP8266连接方式如下

| STM32F103 | IO   | ESP8266 |
| --------- | ---- | ------- |
| RX        | PA10 | TX      |
| TX        | PA11 | RX      |
| 5V        | 5V5  | 5V      |
| GND       | GND  | GND     |

#### 主控芯片串口的配置:

启动串口2，将串口2的模式改为ASYNCHRONOUS,CUBEMX串口波特率默认为115200，ESP8266开发板串口波特率设置为115200，主控芯片波特率默认即可无需修改。

#### 软件开发：

我们所得到的ADC值为FLOAT类型变量，而在HAL库中串口收发的变量为UINT8_T,因此不能通过串口直接发送ADC值，项目中采用SPRINTF函数对变量进行转换

SPRINTF((CHAR \*)STRING, \"%LF\\N\", NUM1 );

以上代码将FLOAT类型转为UINT8_T类型可以通过串口进行发送。

通过串口3将主控芯片ADC转化后的温度变量发送给ESP8266，由于以下两个原因：

一：给服务器造成巨大压力

二：发送速度过快造成数据的重叠

,本项目通过定时中断来控制数据的发送速度,为了方便展示和减缓压力，本项目采用每隔10S进行一次数据的传输。

按照以下的流程，我们就实现了通过ESP8266在网页上显示数据的功能

![image-20230110224653774](概述 {#概述-1}.assets/image-20230110224653774.png)

# ****调试部分

## 蓝牙通信的调试

蓝牙部分的调试可以用到CH340下载器和串口助手进行调试

首先按照以下连接方式进行连接

| STM32F103 | IO   | ESP8266 |
| --------- | ---- | ------- |
| RX        | PA10 | TX      |
| TX        | PA11 | RX      |
| 5V        | 5V5  | 5V      |
| GND       | GND  | GND     |

连接完成后将CH340接入电脑，在串口助手上打开对应的端口，当蓝牙从APP上获取数据时，数据会同样显示在串口助手上，通过此种调试方法，可以对蓝牙数据接收做出判断和调试，也可以通过此种方法判断数据的类型和数据的格式是否正确。此种调试方法较为较为简单

## WIFI通信的调试

WIFI通信采用主控芯片向8266发送数据，8266向WEB客户端发送数据，所以调试分两步进行

### 主控与8266通信的调试：

在确定好使用串口3后，按以下方式进行连接，连接方式与主控芯片和8266之间的连线方式相同，将8266换成CH340

| STM32F103 | IO   | CH340 |
| --------- | ---- | ----- |
| RX        | PA10 | TX    |
| TX        | PA11 | RX    |
| 5V        | 5V5  | 5V    |
| GND       | GND  | GND   |

连接完成后将CH340接入电脑，在串口助手上打开对应的端口，当主控芯片通过串口3发送数据时，数据会同样显示在串口助手上，通过此种调试方法，可以对主控芯片发送的数据做出判断和调试，也可以通过此种方法判断数据的类型和数据的格式是否正确。此种调试方法较为较为简单。

### ESP8266与WEB之间的调试：

本次项目WEB采用FLASK开发，在主程序中启用DEBUG(),当我们用VSCODE运行WEB程序后，当ESP8266向WEB发送任何请求都可以被显示在虚拟终端中，我们可以通过观察虚拟终端判断GET请求的格式与参数，以及通过观察HTTP 请求常见状态码判断客户端的响应状态下面给出请求常见状态表以共参考

```
200：正确的请求返回正确的结果，如果不想细分正确的请求结果都可以直接返回200。
201：表示资源被正确的创建。比如说，我们 POST 用户名、密码正确创建了一个用户就可以返回 201。
202：请求是正确的，但是结果正在处理中，这时候客户端可以通过轮询等机制继续请求。
203：请求的代理服务器修改了源服务器返回的 200 中的内容，我们通过代理服务器向服务器 A 请求用户信息，服务器 A 正常响应，但代理服务器命中了缓存并返回了自己的缓存内容，这时候它返回 203 告诉我们这部分信息不一定是最新的，我们可以自行判断并处理。
300：请求成功，但结果有多种选择。
301：请求成功，但是资源被永久转移。比如说，我们下载的东西不在这个地址需要去到新的地址。
302：临时重定向。
303：使用 GET 来访问新的地址来获取资源。
304：请求的资源并没有被修改过。
308：使用原有的地址请求方式来通过新地址获取资源。
400：请求出现错误，比如请求头不对等。
401：没有提供认证信息。请求的时候没有带上 TOKEN 等。
402：为以后需要所保留的状态码。
403：请求的资源不允许访问。就是说没有权限。
404：请求的内容不存在。
406：请求的资源并不符合要求。
408：客户端请求超时。
413：请求体过大。
415：类型不正确。
416：请求的区间无效。
500：服务器错误。
501：请求还没有被实现。
502：网关错误。
503：服务暂时不可用。服务器正好在更新
```



 

本项目中正常出现的状态码有200，303，302

# ****项目的部署

本项目设计嵌入式的应用开发和web端和app的开发，嵌入式设备完成代码设计后只需要通过烧录工具进行烧录即可

Web端的部署：对于web项目，web服务器软件大多选择Nginx和Apache，在本项目的实际部署中Nginx在部署flask项目上存在异常。所以本项目采用Apache+python3.7环境的环境搭载华为云的\\S x86_64\\S x86_64系统进行项目的部署。

# 其余硬件设计

### 

| STN32F103C8T6 | AD转化 |      |      |
| ------------- | ------ | ---- | ---- |
| I/O           | P1.0   | I/O  |      |

******

| STN32F103C8T6 | 按键 |      |      |
| ------------- | ---- | ---- | ---- |
| I/O           | PB5  | KEY1 |      |
| I/O           | PB4  | KEY2 |      |
| I/O           | PA15 | KEY3 |      |
| I/O           | PA8  | KEY4 |      |



| STN32F103C8T6 | 蜂鸣器 |      |      |
| ------------- | ------ | ---- | ---- |
| I/O           | PC13   | 输入 |      |



| STN32F103C8T6 | 串口1 |      |      |
| ------------- | ----- | ---- | ---- |
| I/O           | PA9   | TX   |      |
| I/O           | PA10  | RX   |      |



| STN32F103C8T6 | 串口2 |      |      |
| ------------- | ----- | ---- | ---- |
| I/O           | PA2   | TX   |      |
| I/O           | PA3   | RX   |      |



| STN32F103C8T6 | 串口3 |      |      |
| ------------- | ----- | ---- | ---- |
| I/O           | PB10  | TX   |      |
| I/O           | PB11  | RX   |      |



| STN32F103C8T6 | OLED |      |      |
| ------------- | ---- | ---- | ---- |
| I/O           | PA6  | CS   |      |
| I/O           | PA4  | DS   |      |
| I/O           | PA0  | RES  |      |
| I/O           | PA7  | D1   |      |
| I/O           | PA5  | D0   |      |
| I/O           | GND  | GND  |      |
| I/O           | VCC  | VCC  |      |



------------------ ----------------- ----------------- -----------------

#  ****部分软件模块说明

```python
@app.route("/dat")
def dat():
    n=0;
    conn=connect_db()
    s=conn.execute('select * from tems').fetchall()
    conn.close()
    s.reverse()
    for dat in s:
        n=n+int(dat['dat'])
    n=n/len(s)
    return render_template('6.html',s=s,n=n)
```

```python
@app.route("/users/<user_id>", methods=["GET"])
def get_user(user_id):
    """获取某个用户信息"""
 
    conn = connect_db()
                
    print(user_id)               
    conn.execute("INSERT INTO tems (dat) VALUES (?)",(user_id,))

    conn.commit()
    conn.close()
    print((user_id)) 
    return (user_id)
```

```c
  return;
  }
  //向服务器发送请求
   String rx_buffer;
  rx_buffer=Serial.readString();
  Serial.print(rx_buffer);
  client.print(String("GET /users/")+rx_buffer+" HTTP/1.1\r\n"+
"Host: "+host+"\r\n"+"Connection: close\r\n\r\n");
  delay(50);
char s;
  //读取返回值
  while(client.available()){
    String line=client.readString();
    for (int i=0;i<180;i++){
    // Serial.print(line[i]);
    // Serial.print(i);

    }
    s=line[144];
  }
  if(s=='1'){
    digitalWrite(2, 0);
  }
 if(s=='0'){
    digitalWrite(2, 1);
  }
  // Serial.println();
  // Serial.println("closing connection");
  delay(10);
 
}
```



# 演示效果

![image-20230110231311417](概述 {#概述-1}.assets/image-20230110231311417.png)

![image-20230110231323702](概述 {#概述-1}.assets/image-20230110231323702.png)

<img src="概述 {#概述-1}.assets/image-20230110231336085.png" alt="image-20230110231336085" style="zoom:80%;" />

![image-20230110231401448](概述 {#概述-1}.assets/image-20230110231401448.png)

![image-20230110231410001](概述 {#概述-1}.assets/image-20230110231410001.png)

![image-20230110231421294](概述 {#概述-1}.assets/image-20230110231421294.png)

# **异常处理**

## Web客户端崩溃

通过服务器的日志可以看出web客户端崩溃的原因主要是在平均值计算时，个别数据类型错误引起的，在数据库中，为了避免在通信时出错，我们统一将数据类型设置为文本格式，从数据库中调出时通过python int函数对数字文本进行转化，但是当文本内容不是数字时int函数则会进行报错，因此项目中在计算平均值之前通过sqllite中的DELETE语句对不匹配的数据进行删除。但次方法具有局限性，如果数据库记录数据多，DELETE语句进行删除时，会造成一定的耗时，对网页的开启速度造成一定的影响，因此依然要从根源上解决错误数据问题

## WEB显示中出现异常值

在测试过程中发现正常温度值为0-50，但浏览数据库中数据，有时会产生4位甚至6位数，通过检查发现是由于串口通信不正确造成的数据重叠，即多组温度同时被接收，由于数据类型都为字符串，因此当数据进入数据库后并不会相加，而是拼凑在一起成为多位数。这里一依然采用sqllite中的DELETE语句对不匹配的数据进行删除。但同时会失去多组采集的数据，以及对数据库进行操作时同样会对网页的运行速度造成影响。

## OLED绘制移动图形显示时出现闪烁

是由于频繁的清屏函数导致的，清屏函数不能随主函数直接运行，应当放入中断或者设置触发条件运行。

# **不足**

4. ## 温度值偶尔会偏高或偏低

   由于本项目使用了esp8266将数据发送到云端进行存储，在观察数据库中数据时，可以发现在统一环境温度下，偶尔会有数值偏高于其他数据的平均值或低于其他数据的平均值，猜测是由于温度检测电路动造成的故障

5. ## 单片机资源利用率低

   在本项目中所暴露出的一个问题是主控芯片的资源不够，在接入oled，按键，蓝牙，wifi模块后，无法留有其他的引脚进行其余功能的开发，是否能更高效的利用单片机资源进行开发，也将是接下来探索的重中之重。

6. ## 通信时会出现乱码

   无论是蓝牙或是esp8266，本项目在真实环境中的数据传输正确率未能达到百分之百，蓝牙通信测试20余次，发生一次异常，esp8266在通信过程中有时会产生数据的重叠。应该是由于串口通信不正确引起

#  

# **视频地址**

BILIBILI：

https://www.bilibili.com/video/bv1484y1x7gr/?vd_source=0266427d0f17b68fe74ee92098244027

# **代码地址**

github：https://github.com/wq2481/stm32-wifi

# **网站地址**

HTTP://123.60.158.55:6789/

HTTP://123.60.158.55:6789/dat

# **感悟**

本次STM32项目的开发是一个极具挑战性的过程，需要从宏观和微观两个层面来考虑问题。从宏观上，需要明确项目的目标、技术要求和实施方案；从微观上，需要考虑单片机的架构、程序设计以及硬件电路设计。

在开发本项目的过程中，一定要做好计划，对项目进行细致的规划，从而确保项目的可行性。同时，在编写程序的时候，要注重代码的可读性，尽量减少重复的代码，以及尽可能的提高代码的执行效率。此外，在编写程序的过程中，要注意检查程序的正确性，及时发现程序中可能存在的BUG，以免在最后测试的时候出现意外情况。此外这次将WEB与单片机系统之间跑通，也为接下来的项目开发提供了更多的可能性，此次项目区上学期的51，新增了RTC时钟,滴答定时器，PWM生成器的知识，使我们的开发有更多的可能，在附加功能中设计的WEB的开发和后端与数据库的设计使我开拓了新视野，对物联网的有了进一步的了解。

同时解决开发中存在的问题，包括OLED游戏显示时的闪烁，网页后端通信的丢包漏包问题时，我们能将问题与所学知识结合起来，总结出一套较为成功的调试步骤，例如在调试oled的过程中，我们可以深入了解oled绘制各种图形和显示字符的原理，在调试串口与其他设备通信，我们可以看到与平常串口收发实验不同的结果，也会遇到以往不会遇见的问题。同样的在项目中尝试新的技术也是全新的体验，能将课外知识与课内知识结合起来，特别是在观察其他类型产品和项目时，找出他们的不足之处，并且想方设法的解决或避免在自己的项目之中。

