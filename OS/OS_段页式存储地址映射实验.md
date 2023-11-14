# 安装`Bochs`

### 下载相关文件

配置好的文件和`bochs`以及指导PPT的下载地址：https://github.com/shwhwhwh/UESTC_course/tree/main/OS

### 打开OS-BAK中的`Bochs-2.5.1.exe`安装`bochs`

一路默认就行，不要改安装路径，记住默认路径为`"C:\Program Files (x86)\Bochs-2.5.1"`就行，如果安装在其他路径下，打开写好的配置文件时会出现无法读取ROM文件的错误，因为它配置时是认为你装在默认路径下的

# 如何进入debug状态

在`Bochs-2.5.1`文件夹中打开`bochsdbg.exe`

<img src="C:\Users\Reirui\AppData\Roaming\Typora\typora-user-images\image-20231114172728174.png" alt="image-20231114172728174" style="zoom: 50%;" />

点击`load`，在`OS-BAK`文件夹下的`linux-0.11-devel-050518`中找到`mybochsrc-hd.bxrc`选中并打开

<img src="C:\Users\Reirui\AppData\Roaming\Typora\typora-user-images\image-20231114172938115.png" alt="image-20231114172938115" style="zoom:50%;" />

点击`start`，此时Bochs用户界面是这样的

<img src="C:\Users\Reirui\AppData\Roaming\Typora\typora-user-images\image-20231114173043157.png" alt="image-20231114173043157" style="zoom:50%;" />

切换到控制台窗口

<img src="C:\Users\Reirui\AppData\Roaming\Typora\typora-user-images\image-20231114173121311.png" alt="image-20231114173121311" style="zoom:50%;" />

在`<bochs:1>`光标处输入`c`，回车，等待控制台窗口不再变化后，用户界面变为如下情形

<img src="C:\Users\Reirui\AppData\Roaming\Typora\typora-user-images\image-20231114173315368.png" alt="image-20231114173315368" style="zoom:50%;" />

输入cat test.c查看代码

<img src="C:\Users\Reirui\AppData\Roaming\Typora\typora-user-images\image-20231114173635741.png" alt="image-20231114173635741" style="zoom:50%;" />

在这个界面中输入`./test`，系统就会执行test这个可执行文件，然后在while(j!=0)处进入死循环

<img src="C:\Users\Reirui\AppData\Roaming\Typora\typora-user-images\image-20231114173733184.png" alt="image-20231114173733184" style="zoom:50%;" />

切换到控制台窗口，按住`ctrl`+`c`，进入debug状态

<img src="C:\Users\Reirui\AppData\Roaming\Typora\typora-user-images\image-20231114173830159.png" alt="image-20231114173830159" style="zoom:50%;" />

# 如何一步一步找到j的地址

每次运行得到的j地址可能不同，实际进行实验时应以理解本经验的思路为主，不要原封不动地copy本经验中得到的相关数据

## 第一步——找到j所在段基址

因为j是全局变量，存储在Data Segment中，我们需要知道Data Segment的段基地址：

1. 输入sreg，查看所有段选择符的值

   ![image-20231114174336396](C:\Users\Reirui\AppData\Roaming\Typora\typora-user-images\image-20231114174336396.png)

2. 发现ds的段选择符寄存器值为0x0017 = 0000 0000 0001 0(选择子) + 1(TI) + 11(RPL)，因为TI为1，即 使用局部描述符表，所以我们需要LDT的段基地址，而这个地址存储在LDT的描述符中，所以我们需要先找到LDT的描述符的值

3. 发现LDT的选择符寄存器ldtr为0x0068 = 0000 0000 0110 1 + 0 + 00，TI = 0，说明该LDT的段描述符存储在GDT中，同时选择子 = 0000 0000 0110 1 = 13，所以该LDT的描述符为GDT中的第14个描述符(从0开始计数)，有因为一个描述符的大小为64bit，即8字节，2字，所以输入`xp /28w 0x5cb8`(0x5cb8为GDT的基址，可以从gdtr处得到)

   ![image-20231114175830748](C:\Users\Reirui\AppData\Roaming\Typora\typora-user-images\image-20231114175830748.png)

4. 倒数两项为  0x92d00068  0x000082fd, 采用小端存储，所以LDT描述符的高32位为0x**00**0082**fd**，低32位为0x**92d0**0068，比较sreg中ldtr的结果(dh=0x000082fd, dl=0x92d00068)可以发现该值就是LDT描述符的内容，因为描述符中的63-56为段基址的31-24位，39-32位为段基址的23-16位，31-16为基址的15-0位，所以LDT的段基址为0x00 + fd + 92d0 = 0x00fd92d0

5. 找到LDT的基址后就可以去找数据段的基址了，在2. 中，我们得到ds的段描述符为LDT中的第0000 0000 0001 0 + 1 = 3个描述符，所以用`xp /6w 0x00fd92d0`去得到数据段基地址

   ![image-20231114181317327](C:\Users\Reirui\AppData\Roaming\Typora\typora-user-images\image-20231114181317327.png)

6. 同4. ，我们得到ds的段描述符值为dh = 0x**10**c0f3**00**, dl = 0x**0000**3fff，从而得到数据段段基址为0x10 + 00 + 0000 = 0x10000000

7. 至此，我们已经得到了j的线性地址即段基址 + 偏移地址 = 0x10000000 + 0x004(因为页的大小为4K(通过描述符中的G = 1确定)，所以偏移地址为虚拟地址的低12位，然后程序在进入死循环前打印出了j的虚拟地址：0x3004)

## 第二步由线性地址到物理地址

![image-20231114182248801](C:\Users\Reirui\AppData\Roaming\Typora\typora-user-images\image-20231114182248801.png)

1. 线性地址的结构为 目录 + 页 + 偏移量，偏移量我们已经知道为低12位，目录和页为哪几位需要通过页目录表的大小和页目录项的大小，页表的大小和页表项的大小来确定，在本实验中，页目录和页表大小都为4K，数据项大小都为4字节，所以`目录`和`页`各占10位

2. 输入`cr`，查看CR3寄存器的值

   ![image-20231114182551460](C:\Users\Reirui\AppData\Roaming\Typora\typora-user-images\image-20231114182551460.png)

3. 由CR3和线性地址中的高10位找到页目录项的地址 = 0x0 + 0001 0000 00 = 64，用`xp /w 0+64*4`查看页目录项的值

   ![image-20231114183106370](C:\Users\Reirui\AppData\Roaming\Typora\typora-user-images\image-20231114183106370.png)

4. 页基址：页表的起始地址是4K的整数倍，因此32位地址的低12位总为0，用高20位表示即可，即页目录表项中给出的是页基址的高20位。而这20位被存储在页目录项的高20位，所以页表的起始地址位0x00fa7，由虚拟地址为0x3004，可以知道`页`的值为00 0000 0011 = 3，即页框号存在页表中的正数第4个位置，所以页表项地址为0x00fa7000 + 3*4，页框号用`xp /w  0x00fa7000+3*4`得出

   ![image-20231114184610719](C:\Users\Reirui\AppData\Roaming\Typora\typora-user-images\image-20231114184610719.png)

5. 所以j的物理地址为页框号: 0x00fa6000 + 偏移量 0x004 = 0xfa6004，用`xp /w 0xfa6004`可以验证这个地址上存的是否为j

6. 用`setpmem 0xfa6004 4 0`即可将j的值置为0

7. 输入`c`，控制台窗口不再出现`<bochs:x>`，让程序执行完退出循环的指令，可以看到用户界面程序继续向下执行

   ![image-20231114185012287](C:\Users\Reirui\AppData\Roaming\Typora\typora-user-images\image-20231114185012287.png)

