本次实验内容：本次实验将接触到Call调用这个概念，什么是Call调用\? Call相当于你在编程时所编写的函数，而高级语言中的函数最终也是会被编译器转换为汇编格式的Call调用，这些关键Call普遍都会存在各种参数，关于Call的作用，打个比方有些网游外挂可以实现自动寻路,自动吃药,自动打怪,甚至是全屏秒杀,这些功能是通过修改数值也无法做到的,Call就可做到。

  其实关键Call就是作者开发过程中写的一个个处理不同事件的独立的处理函数，这些函数包括了各种独立的游戏功能，而我们可以在远程进程中开辟线程，并通过汇编形式动态的调用这些关键Call，从而实现一些变态功能。

  本次实验我们将通过查找阳光的掉落的定时器，并通过定时器变量顺藤摸瓜找到生成阳光的关键Call，通过给此Call传递不同参数实现掉落阳光，钻石，零秒通关等，阳光遍历技巧如下：

  > 进入游戏等待阳光出现 \-> 当阳光出现后 \-> 马上搜索未知初始数值  
  > 返回游戏等待时钟发生变化 \-> 马上切回CE \-> 搜索减少的数值 \-> 阳光下落一点搜一点  
  > 经过上方步骤反复排查 \-> 最终能找到一个值范围\(0-700\) \-> 锁定1即可实现无限掉落

  基址与偏移的找法，在文章开头就已经分享了查找的技巧，此处控制阳光掉落公式 `[[[006A9F38 + 768] + 5538]]`

  > 00413B7C \- 83 86 38550000 FF \- add dword ptr \[esi+00005538\],-01 \<\<  
  > 004524F4 \- 8B 86 68070000 \- mov eax,\[esi+00000768\] \<\<  
  > 00599F75 \- A1 389F6A00 \- mov eax,\[PlantsVsZombies.exe+2A9F38\] \<\<

  我们要找关键Call只需要等待阳光出现，当阳光出现后，CE会检测到一条数据 `add dword ptr [esi+00005538],-01` 说明此时定时器开始工作了，我们只需要记下这个内存地址`00413B7C`，然后关闭CE，并打开X64DBG附加到游戏。

  ![](/image/1379525-20191206173832764-1897103386.png)

  打开X64DBG附加到游戏进程，然后按下`ctrl+G`定位到`00413B7C` 此时我们需要关注`add dword ptr ds:[esi+0x5538],0xFFFFFFFF`这条计时器指令下方，就是一个大跳转该跳转跳向了结束，那么我们可以猜测`jne 0x413BF1`跳过的内容很有可能就是跳过了阳光的生成过程，也就是可能跳过了阳光生成Call。

  ![](/image/1379525-20191206173847142-2095746408.png)

  玩过此游戏的一定知道，游戏屏幕中不止可以掉落出阳光，还可以掉出其他的东西，比如钻石，金钱，奖牌等，那么我们有理由相信，该游戏中调用的Call应该有很多参数传递，比如掉落属性，掉落坐标，掉落类型等，而我们已经找到了阳光计时器每次递减的汇编代码，故猜测调用Call应该就在附近，向下查找发现有很多Call调用，但有一个比较特别的`call 0x40CB10`，之所以说它特别是因为该Call在调用之前，通过push传递了多了参数，此处很可疑，我们需要具体分析。

  ![](/image/1379525-20191206173857727-425890694.png)

  此时我们在`00413BE4`处下断点，然后回到游戏发现每当阳光出现的时候，此处就会断下，那么我们尝试给它传递不同的参数，例如给`eax`传递一个2，观察效果发现原本应该是出现阳光的地方，现在居然出现了金币。

  ![](/image/1379525-20191206173909138-1576680569.png)

  经过上方的测试，发现我们猜测是合理的，那么这段代码，转换为C语言后应该是这样的:

  ```c
  > push 0 普通掉落 push 2-3 其他掉落方式  push 4 自动收集阳光 push 6 右侧滑出掉落
  > eax=2 金钱   eax=3 钻石   eax=4 普通阳光  eax=5 小的阳光 eax=6 大的阳光  eax=8 自动通关神器

  sub_00413BE4(push 0= 物品飞出状态,push eax= 掉落物品,push 0x3c,push edi= 对象状态,ecx= 游戏基地址)
  {
	return 0;
  }
  ```

  到此，我们还没有办法完成外部注入，因为据我观察程序中的edi寄存器与ecx寄存器中的数据是动态的，每次游戏重新运行都会发生变化，如果想要在外部调用这个Call函数，我们需要找到这两个寄存器的基址，或者说找到他们的来源。

  我们先来分析第一个地址如何定位，第一个动态地址是EDI寄存器，这个寄存器每次存储一个整数，此处无法直接找基址，我采取的方式是从后向前逆推代码，观察那些指令改写了该寄存器，然后将这些改写寄存器的指令拼凑起来。

  ![](/image/1379525-20191206173921267-405052345.png)

  这些调用分别是上图中的，赋值1-赋值3外加一个`call 0x005AF400`，程序中正是通过这种方式计算出动态数据的，我们直接将其提取出来，提取后代码如下：

  ```C
  mov eax,226
  call 0x005AF400
  mov edi,eax
  add edi,64
  ```

  再来分析第二个，第二个相对于第一个来说就好找许多了，因为它是一个动态地址，如下图我们记下 `13DCE000` 这个动态地址，然后直接在X64dbg中脱离游戏，这里不能结束游戏，如果结束了下次该地址又会发生变化。

  ![](/image/1379525-20191206173951202-1184473449.png)

  CE直接附加游戏，然后我们来搜索十六进制数据`13DCE000`然后，你懂的，就是找基址，前面已经找过很多次了。

  ![](/image/1379525-20191206174001606-1405109477.png)

  通过查找访问代码，就能找到一级偏移是768，继续记下`00FE4170`然后回到CE继续搜索。

  ![](/image/1379525-20191206174011343-1734726384.png)

  到这里基地址出来了，其公式为`[6A9EC0+768]`，从这里也可以看出，这应该是一个通用对象地址。

  ![](/image/1379525-20191206174020913-172087178.png)

  最后，我们整合并写出以下汇编代码，然后使用注入器注入到游戏测试。

  ```C
  pushad
  mov eax,226
  call 0x005AF400
  mov edi,eax
  add edi,64

  mov esi,dword ptr ds:[ 006a9ec0 ]
  mov esi,dword ptr ds:[ esi + 768 ]

  push 0
  push 4
  push 3c
  push edi
  mov ecx,esi
  call 0x40cb10
  popad
  ```

  发现每当我们点击注入以后，程序中就会多出一个太阳，说明我们的汇编代码没有问题。

  ![](/image/1379525-20191206174033344-1517341867.png)

  为了能够写出自己的外挂，我们需要实现一个简单的注入器，并能够通过远程的方式调用阳光生成Call，使用C语言编写如下代码即可实现注入效果，需要注意调用约定，此处我在Win10上测试没有问题，在Xp系统上会出现异常退出的情况。

  ```C
  #include <stdio.h>
  #include <Windows.h>

  void AddSun()
  {
	__asm{
		pushad
			mov eax, 226
			mov ebx, 0x005AF400
			call ebx
			mov edi, eax
			add edi, 64
			mov esi, dword ptr ds : [0x006a9ec0]
			mov esi, dword ptr ds : [esi + 0x768]
			push 0h
			push 0x4
			push 0x3c
			push edi
			mov ecx, esi
			mov edx, 0x40cb10
			call edx
		popad

	}
  }

  void InjectCode(DWORD dwProcId, LPVOID mFunc)
  {
	HANDLE hProcess,hThread;
	LPVOID mFuncAddr,ParamAddr;
	DWORD NumberOfByte;

	hProcess = OpenProcess(PROCESS_ALL_ACCESS, FALSE, dwProcId);
	mFuncAddr = VirtualAllocEx(hProcess, NULL, 128, MEM_COMMIT, PAGE_EXECUTE_READWRITE);
	WriteProcessMemory(hProcess, mFuncAddr, mFunc, 128, &NumberOfByte);
	hThread = CreateRemoteThread(hProcess, NULL, 0, (LPTHREAD_START_ROUTINE)mFuncAddr, ParamAddr, 0, &NumberOfByte);
	WaitForSingleObject(hThread, INFINITE);
	VirtualFreeEx(hProcess, mFuncAddr, 128, MEM_RELEASE);
	CloseHandle(hThread);
	CloseHandle(hProcess);
  }

  int main()
  {
	for (int i = 0; i < 10;i++)
		InjectCode(3644, AddSun);
  }
  ```

  至此我们的植物大战僵尸关于阳光的分析就到此结束了，其他的比如植物无敌，植物攻击加速等，就留一个作业，大家开动脑筋看能不能自己实现，回顾前面所讲的外挂制作技巧，你是否已经完全理解了呢？如果你能够吃透这些基础知识，那么相信你可以做出植物无敌，攻击加速等其他变态功能，其实逆向就是一个思路的问题，大家要多去思考，多去尝试，相信你会成功的！
