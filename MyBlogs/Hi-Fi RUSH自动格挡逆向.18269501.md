# 前言

&emsp;&emsp;Hi-Fi RUSH是个玩起来非常爽快的动作游戏，这个游戏的战斗有个非常经典的机制也就是格挡。第三关的时候旁白会教你该如何格挡，虽然看起来很简单只要在闪光（发出声音）后按E即可，但是在我进行全关卡皇冠S通关流程时我却发现实际上格挡的要求其实还挺严格的。首先在施展派生招式比如震音旋风在主角腾空发波时，如果这个时候敌人攻击，你想按E格挡是完全没用的，这个时候你只能闪避。还有在下砸招式中敌人突然攻击你按E格挡也是没用的会直接受到伤害。除此之外敌人之间也会各种配合，比如近战远程机器人有可能在某个时机同时攻击你，而且近战机器人可能会进行车轮战，你要根据不同的敌人种类按不同的节拍格挡。此外远程机器人还可能在你视线外偷袭你，这个时候只能听声音来判断格挡时机。这些限制条件加上实际战斗的状况其实完全掌握格挡机制还挺难的（对于我这种不是经常玩ACT游戏的人来说），但是抛开这些限制其实我还是非常喜欢这个格挡机制的，于是我想逆向分析下。

# 分析

&emsp;&emsp;通过直觉我认为有个函数一定是和格挡判断有关的，所以我们得找到这个函数在哪。首先在这个游戏中格挡成功与否与是否减少生命值有着直接的联系，有了这个我们就可以开始了。通过查找生命值的增加与减少来找到储存生命值的地址，我发现格挡失败时生命值减少具体实现的函数在这里。

```
Hi-Fi-RUSH.exe+1933BB0 - 40 53                 - push rbx
Hi-Fi-RUSH.exe+1933BB2 - 48 83 EC 50           - sub rsp,50 { 80 }
Hi-Fi-RUSH.exe+1933BB6 - 0F29 74 24 40         - movaps [rsp+40],xmm6
Hi-Fi-RUSH.exe+1933BBB - 48 8B 05 16BA5D05     - mov rax,[Hi-Fi-RUSH.exe+6F0F5D8] { (-1.00) }
Hi-Fi-RUSH.exe+1933BC2 - 48 33 C4              - xor rax,rsp
Hi-Fi-RUSH.exe+1933BC5 - 48 89 44 24 30        - mov [rsp+30],rax
Hi-Fi-RUSH.exe+1933BCA - 80 B9 C1010000 00     - cmp byte ptr [rcx+000001C1],00 { 0 }
Hi-Fi-RUSH.exe+1933BD1 - 0F28 E1               - movaps xmm4,xmm1
Hi-Fi-RUSH.exe+1933BD4 - F3 0F10 99 98010000   - movss xmm3,[rcx+00000198]
Hi-Fi-RUSH.exe+1933BDC - 48 8B D9              - mov rbx,rcx
Hi-Fi-RUSH.exe+1933BDF - F3 0F11 99 9C010000   - movss [rcx+0000019C],xmm3
Hi-Fi-RUSH.exe+1933BE7 - 0F57 F6               - xorps xmm6,xmm6
Hi-Fi-RUSH.exe+1933BEA - 74 0A                 - je Hi-Fi-RUSH.exe+1933BF6
Hi-Fi-RUSH.exe+1933BEC - F3 0F10 05 7C570605   - movss xmm0,[Hi-Fi-RUSH.exe+6999370] { (1.00) }
Hi-Fi-RUSH.exe+1933BF4 - EB 03                 - jmp Hi-Fi-RUSH.exe+1933BF9
Hi-Fi-RUSH.exe+1933BF6 - 0F57 C0               - xorps xmm0,xmm0
Hi-Fi-RUSH.exe+1933BF9 - F3 0F10 91 90010000   - movss xmm2,[rcx+00000190]
Hi-Fi-RUSH.exe+1933C01 - 0F28 CB               - movaps xmm1,xmm3
Hi-Fi-RUSH.exe+1933C04 - F3 0F58 CC            - addss xmm1,xmm4
Hi-Fi-RUSH.exe+1933C08 - 0F2F C8               - comiss xmm1,xmm0
Hi-Fi-RUSH.exe+1933C0B - 72 07                 - jb Hi-Fi-RUSH.exe+1933C14
Hi-Fi-RUSH.exe+1933C0D - 0F28 C2               - movaps xmm0,xmm2
Hi-Fi-RUSH.exe+1933C10 - F3 0F5D C1            - minss xmm0,xmm1
Hi-Fi-RUSH.exe+1933C14 - 0F2E C3               - ucomiss xmm0,xmm3
Hi-Fi-RUSH.exe+1933C17 - F3 0F11 81 98010000   - movss [rcx+00000198],xmm0 { 减少生命值 }
Hi-Fi-RUSH.exe+1933C1F - 74 57                 - je Hi-Fi-RUSH.exe+1933C78
Hi-Fi-RUSH.exe+1933C21 - 0F2F D6               - comiss xmm2,xmm6
Hi-Fi-RUSH.exe+1933C24 - 77 05                 - ja Hi-Fi-RUSH.exe+1933C2B
Hi-Fi-RUSH.exe+1933C26 - 0F57 C9               - xorps xmm1,xmm1
Hi-Fi-RUSH.exe+1933C29 - EB 07                 - jmp Hi-Fi-RUSH.exe+1933C32
Hi-Fi-RUSH.exe+1933C2B - 0F28 C8               - movaps xmm1,xmm0
Hi-Fi-RUSH.exe+1933C2E - F3 0F5E CA            - divss xmm1,xmm2
Hi-Fi-RUSH.exe+1933C32 - 48 81 C1 10010000     - add rcx,00000110 { 272 }
Hi-Fi-RUSH.exe+1933C39 - F3 0F11 44 24 20      - movss [rsp+20],xmm0
Hi-Fi-RUSH.exe+1933C3F - 48 8D 54 24 20        - lea rdx,[rsp+20]
Hi-Fi-RUSH.exe+1933C44 - F3 0F11 5C 24 24      - movss [rsp+24],xmm3
Hi-Fi-RUSH.exe+1933C4A - F3 0F11 4C 24 28      - movss [rsp+28],xmm1
Hi-Fi-RUSH.exe+1933C50 - E8 4BB759FF           - call Hi-Fi-RUSH.exe+ECF3A0
Hi-Fi-RUSH.exe+1933C55 - F3 0F10 83 98010000   - movss xmm0,[rbx+00000198]
Hi-Fi-RUSH.exe+1933C5D - 0F2F C6               - comiss xmm0,xmm6
Hi-Fi-RUSH.exe+1933C60 - 77 16                 - ja Hi-Fi-RUSH.exe+1933C78
Hi-Fi-RUSH.exe+1933C62 - 48 8D 8B 30010000     - lea rcx,[rbx+00000130]
Hi-Fi-RUSH.exe+1933C69 - 33 D2                 - xor edx,edx
Hi-Fi-RUSH.exe+1933C6B - E8 30B759FF           - call Hi-Fi-RUSH.exe+ECF3A0
Hi-Fi-RUSH.exe+1933C70 - F3 0F10 83 98010000   - movss xmm0,[rbx+00000198]
Hi-Fi-RUSH.exe+1933C78 - 48 8B 4C 24 30        - mov rcx,[rsp+30]
Hi-Fi-RUSH.exe+1933C7D - 48 33 CC              - xor rcx,rsp
Hi-Fi-RUSH.exe+1933C80 - E8 ABB0B103           - call Hi-Fi-RUSH.exe+544ED30
Hi-Fi-RUSH.exe+1933C85 - 0F28 74 24 40         - movaps xmm6,[rsp+40]
Hi-Fi-RUSH.exe+1933C8A - 48 83 C4 50           - add rsp,50 { 80 }
Hi-Fi-RUSH.exe+1933C8E - 5B                    - pop rbx
Hi-Fi-RUSH.exe+1933C8F - C3                    - ret 
```

通过不断下断点加上Step Out测试是否和格挡判断函数有关，我们最终来到了这个冗长的函数，先省略其它的无关指令。

```
Hi-Fi-RUSH.exe+1931ED0 - 40 55                 - push rbp
......
Hi-Fi-RUSH.exe+19322CB - FF 90 F8080000        - call qword ptr [rax+000008F8] { 格挡判断函数 }
Hi-Fi-RUSH.exe+19322D1 - 84 C0                 - test al,al
Hi-Fi-RUSH.exe+19322D3 - 74 3B                 - je Hi-Fi-RUSH.exe+1932310
Hi-Fi-RUSH.exe+19322D5 - 49 8B D7              - mov rdx,r15
Hi-Fi-RUSH.exe+19322D8 - 49 8B CD              - mov rcx,r13
Hi-Fi-RUSH.exe+19322DB - E8 E0400000           - call Hi-Fi-RUSH.exe+19363C0
Hi-Fi-RUSH.exe+19322E0 - 84 C0                 - test al,al
Hi-Fi-RUSH.exe+19322E2 - 74 2C                 - je Hi-Fi-RUSH.exe+1932310
Hi-Fi-RUSH.exe+19322E4 - 40 84 FF              - test dil,dil
Hi-Fi-RUSH.exe+19322E7 - C6 44 24 31 01        - mov byte ptr [rsp+31],01 { 1 }
Hi-Fi-RUSH.exe+19322EC - 74 0B                 - je Hi-Fi-RUSH.exe+19322F9
Hi-Fi-RUSH.exe+19322EE - 49 8B D7              - mov rdx,r15
Hi-Fi-RUSH.exe+19322F1 - 49 8B CD              - mov rcx,r13
Hi-Fi-RUSH.exe+19322F4 - E8 072E0000           - call Hi-Fi-RUSH.exe+1935100
Hi-Fi-RUSH.exe+19322F9 - 49 8D 9D C8010000     - lea rbx,[r13+000001C8]
Hi-Fi-RUSH.exe+1932300 - 45 33 E4              - xor r12d,r12d
Hi-Fi-RUSH.exe+1932303 - 41 FF C6              - inc r14d
Hi-Fi-RUSH.exe+1932306 - 44 89 74 24 38        - mov [rsp+38],r14d
Hi-Fi-RUSH.exe+193230B - E9 90FCFFFF           - jmp Hi-Fi-RUSH.exe+1931FA0 { 跳出 }
Hi-Fi-RUSH.exe+1932310 - 49 8B 04 24           - mov rax,[r12]
......
Hi-Fi-RUSH.exe+1932317 - FF 90 E8080000        - call qword ptr [rax+000008E8] { 闪避判断函数 }
Hi-Fi-RUSH.exe+193231D - 84 C0                 - test al,al
Hi-Fi-RUSH.exe+193231F - 74 44                 - je Hi-Fi-RUSH.exe+1932365
Hi-Fi-RUSH.exe+1932321 - 49 8B 5F 30           - mov rbx,[r15+30]
Hi-Fi-RUSH.exe+1932325 - 48 85 DB              - test rbx,rbx
Hi-Fi-RUSH.exe+1932328 - 74 3B                 - je Hi-Fi-RUSH.exe+1932365
Hi-Fi-RUSH.exe+193232A - E8 D1FAE5FF           - call Hi-Fi-RUSH.exe+1791E00
Hi-Fi-RUSH.exe+193232F - 48 8B 53 10           - mov rdx,[rbx+10]
Hi-Fi-RUSH.exe+1932333 - 4C 8D 40 30           - lea r8,[rax+30]
Hi-Fi-RUSH.exe+1932337 - 48 63 40 38           - movsxd  rax,dword ptr [rax+38]
Hi-Fi-RUSH.exe+193233B - 3B 42 38              - cmp eax,[rdx+38]
Hi-Fi-RUSH.exe+193233E - 7F 25                 - jg Hi-Fi-RUSH.exe+1932365
Hi-Fi-RUSH.exe+1932340 - 48 8B C8              - mov rcx,rax
Hi-Fi-RUSH.exe+1932343 - 48 8B 42 30           - mov rax,[rdx+30]
Hi-Fi-RUSH.exe+1932347 - 4C 39 04 C8           - cmp [rax+rcx*8],r8
Hi-Fi-RUSH.exe+193234B - 75 18                 - jne Hi-Fi-RUSH.exe+1932365
Hi-Fi-RUSH.exe+193234D - 40 38 B3 A0000000     - cmp [rbx+000000A0],sil
Hi-Fi-RUSH.exe+1932354 - 74 0F                 - je Hi-Fi-RUSH.exe+1932365
Hi-Fi-RUSH.exe+1932356 - 41 88 37              - mov [r15],sil
Hi-Fi-RUSH.exe+1932359 - 41 C6 47 02 01        - mov byte ptr [r15+02],01 { 1 }
Hi-Fi-RUSH.exe+193235E - C6 44 24 32 01        - mov byte ptr [rsp+32],01 { 1 }
Hi-Fi-RUSH.exe+1932363 - EB 94                 - jmp Hi-Fi-RUSH.exe+19322F9
Hi-Fi-RUSH.exe+1932365 - 49 8B 7F 30           - mov rdi,[r15+30]
......
Hi-Fi-RUSH.exe+193274F - E8 7C040000           - call Hi-Fi-RUSH.exe+1932BD0 { 执行减少生命值的函数 }
......
Hi-Fi-RUSH.exe+1932AA4 - C3                    - ret 
```

&emsp;&emsp;在入口处下断点发现在机器人发起攻击的时候，游戏会直接被CE设的断点暂停。好！那么这个函数一定和格挡判断函数有关。已知在Hi-Fi-RUSH.exe+193274F call Hi-Fi-RUSH.exe+1932BD0处会执行和生命值减少有关的函数那么前面的指令一定有jmp指令和ret指令来提前退出。这个函数的指令非常多但是不慌，我们可以对比格挡成功和格挡失败的区别。在一步步调试后发现在Hi-Fi-RUSH.exe+19322CB call qword ptr [rax+000008F8]处执行了个函数指针，然后程序会通过Hi-Fi-RUSH.exe+19322D1 test al,al来判断是否格挡成功，如果格挡成功指令会一直执行到Hi-Fi-RUSH.exe+193230B jmp Hi-Fi-RUSH.exe+1931FA0处跳转从而不执行后面丢失生命值有关的指令，如果格挡失败指令就会从Hi-Fi-RUSH.exe+19322D3 je Hi-Fi-RUSH.exe+1932310处跳转进入后面的减少生命值的指令。由此可见正确答案就在Hi-Fi-RUSH.exe+19322CB call qword ptr [rax+000008F8]这里，其实这里有点感觉了，这个[rax+0x000008F8]应该是不同对象受到攻击的函数指针，当我进行测试时主角和机器人被攻击时rax寄存器的值会变化，但是当主角受到攻击时这里的rax总是个定值。通过追踪我发现最终来到了这个函数。

```
Hi-Fi-RUSH.exe+D590B62 - 00 6B 19              - add [rbx+19],ch
Hi-Fi-RUSH.exe+D590B65 - 41 AB                 - stosd 
Hi-Fi-RUSH.exe+D590B67 - 66 0F1F 84 00 00000000  - nop word ptr [rax+rax+00000000]
Hi-Fi-RUSH.exe+D590B70 - 40 53                 - push rbx
Hi-Fi-RUSH.exe+D590B72 - 48 83 EC 20           - sub rsp,20 { 32 }
Hi-Fi-RUSH.exe+D590B76 - 48 89 CB              - mov rbx,rcx
Hi-Fi-RUSH.exe+D590B79 - E8 B2B6A4F4           - call Hi-Fi-RUSH.exe+1FDC230
Hi-Fi-RUSH.exe+D590B7E - 84 C0                 - test al,al
Hi-Fi-RUSH.exe+D590B80 - 75 2B                 - jne Hi-Fi-RUSH.exe+D590BAD
Hi-Fi-RUSH.exe+D590B82 - 38 83 E6090000        - cmp [rbx+000009E6],al
Hi-Fi-RUSH.exe+D590B88 - 7D 09                 - jnl Hi-Fi-RUSH.exe+D590B93
Hi-Fi-RUSH.exe+D590B8A - F6 83 E2090000 08     - test byte ptr [rbx+000009E2],08 { 8 }
Hi-Fi-RUSH.exe+D590B91 - 75 1A                 - jne Hi-Fi-RUSH.exe+D590BAD
Hi-Fi-RUSH.exe+D590B93 - 0F57 C0               - xorps xmm0,xmm0
Hi-Fi-RUSH.exe+D590B96 - 0F2F 83 30060000      - comiss xmm0,[rbx+00000630]
Hi-Fi-RUSH.exe+D590B9D - 66 8B 1D 9A7B2E07     - mov bx,[Hi-Fi-RUSH.exe+1487873E] { (7) }
Hi-Fi-RUSH.exe+D590BA4 - 0F92 D0               - setb al { 通过比较两个浮点数来判断格挡是否成功 }
Hi-Fi-RUSH.exe+D590BA7 - 48 83 C4 20           - add rsp,20 { 32 }
Hi-Fi-RUSH.exe+D590BAB - 5B                    - pop rbx
Hi-Fi-RUSH.exe+D590BAC - C3                    - ret 
```

这应该就是和主角格挡判定机制有关的函数，在这个地方我们发现了对应的指令。

```
Hi-Fi-RUSH.exe+D590B96 - 0F2F 83 30060000      - comiss xmm0,[rbx+00000630]
Hi-Fi-RUSH.exe+D590B9D - 66 8B 1D 9A7B2E07     - mov bx,[Hi-Fi-RUSH.exe+1487873E] { (7) }
Hi-Fi-RUSH.exe+D590BA4 - 0F92 D0               - setb al { 通过比较两个浮点数来判断格挡是否成功 }
```

&emsp;&emsp;这里比较了两个浮点数然后来判断格挡是否成功，感觉应该是和格挡时机有关，在游戏教学中告诉了玩家要在拍点时按下格挡键才能成功格挡。然后我还观察了下从Hi-Fi-RUSH.exe+1932317 - FF 90 E8080000        - call qword ptr [rax+000008E8] { 闪避判断函数 }到Hi-Fi-RUSH.exe+1932365 - 49 8B 7F 30           - mov rdi,[r15+30]之间的代码块，发现和格挡判断的代码块很像，直觉告诉我[rax+000008E8]是闪避判断函数，修改这个函数后主角也确实会在处决过程中百分之百闪避。到这里应该就大功告成了！下面稍微写点代码。

# 修改代码
```   
#include<iostream>
#include<Windows.h>
#include<psapi.h>
#include<TlHelp32.h>

using namespace std;

int main(int argc, const char* argv[])
{
	const HWND hwnd = FindWindowA(nullptr, "Hi-Fi RUSH  ");

	if (hwnd == 0x0)
	{
		cout << "cannot find window named Hi-Fi RUSH  ";

		return 0;
	}

	cout << "find window succeeded\n";

	DWORD processId = 0;

	GetWindowThreadProcessId(hwnd, &processId);

	const HANDLE processHandle = OpenProcess(PROCESS_ALL_ACCESS, 0, processId);

	const HANDLE snapShotHandle = CreateToolhelp32Snapshot(TH32CS_SNAPALL, processId);

	MODULEENTRY32 moduleEntry = {};

	moduleEntry.dwSize = sizeof(MODULEENTRY32);

	Module32First(snapShotHandle, &moduleEntry);

	wcout << moduleEntry.szExePath << "\n";

	const DWORD64 baseAddr = (DWORD64)moduleEntry.modBaseAddr;

	cout << "exe base addr " << hex << baseAddr << dec << "\n";

	//parry
	{
		const DWORD64 offset = 0xD590BA4;

		const DWORD64 addr = baseAddr + offset;

		const UINT parryByteCodeLength = 3;

		BYTE originalCode[parryByteCodeLength] = {};

		ReadProcessMemory(processHandle, (LPCVOID)addr, originalCode, parryByteCodeLength, nullptr);

		if (originalCode[0] == 0x0F)
		{
			cout << "start automatic parry\n";

			const BYTE code[] = { 0xB0, 0x01 ,0x90 };

			WriteProcessMemory(processHandle, (LPVOID)addr, code, parryByteCodeLength, nullptr);
		}
		else
		{
			cout << "stop automatic parry\n";

			const BYTE code[] = { 0x0F, 0x92 ,0xD0 };

			WriteProcessMemory(processHandle, (LPVOID)addr, code, parryByteCodeLength, nullptr);
		}
	}

	//dodge
	{
		const DWORD64 offset = 0xD58EDC7;

		const DWORD64 addr = baseAddr + offset;

		const UINT dodgeByteCodeLength = 2;

		BYTE originalCode[dodgeByteCodeLength] = {};

		ReadProcessMemory(processHandle, (LPCVOID)addr, originalCode, dodgeByteCodeLength, nullptr);

		if (originalCode[0] == 0x30)
		{
			cout << "start automatic dodge\n";

			const BYTE code[] = { 0xB0,0x01 };

			WriteProcessMemory(processHandle, (LPVOID)addr, code, dodgeByteCodeLength, nullptr);
		}
		else
		{
			cout << "stop automatic dodge\n";

			const BYTE code[] = { 0x30,0xC0 };

			WriteProcessMemory(processHandle, (LPVOID)addr, code, dodgeByteCodeLength, nullptr);
		}
	}

	CloseHandle(processHandle);

	CloseHandle(snapShotHandle);

	return 0;
}
```
