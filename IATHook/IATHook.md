# IATHook
## 实验目的
* 了解dll注入原理并将其应用
* 了解IATHook的攻击原理并进行实验
## 实验完成度
* [x] hook writefile,自己调用writefile(首先create Writefile)，使得每次写入文件时，输入的hello,变成hello hacker
* [x] 把hook writefile 写为dll，把他注入到notepad类似进程中，进行hook
## 实验步骤
### MessageBoxA实例运行
1. git仓库[下载](https://github.com/tinysec/iathook.git)IATHook的实例             
```bash
cd D:\YearJunior1\SoftwareProjectSecurityDevelopmentLifecycle\EXP5\IAT
git clone  https://github.com/tinysec/iathook.git      
```
2. 将文件中的readme.md复制为main.c,并且将不是代码的部分删掉，使编译能够通过
3. 使用vs的开发者工具，转到目录                  
```bash
cd D:\YearJunior1\SoftwareProjectSecurityDevelopmentLifecycle\EXP5\IAT\iathook
```
4. 进行编译链接，生成IAT.exe           
```bash
cl /c IATHook.c
cl /c main.c
link main.obj IATHook.obj user32.lib /out:IAT.exe
```           
![](./img/23iat.png)              
6. 运行一下EXE,```IAT.exe```,会发现生成了两次弹窗                              
![](./img/23tan1.png)           
![](./img/23tan2.png)               
7. 新建一个工程```D:\YearJunior1\SoftwareProjectSecurityDevelopmentLifecycle\EXP5\Hook```
```wmain```,```iathook```和中间的```MessageBoxA```中间也下断点。执行到message，会发现这个位置改变,说明攻击的就是这里                                          
![](./img/23cha2.png)                                

### Hook WriteFile
>实验代码基于MessageBoxA()代码
1. 因为windows底下有两个notepad.exe，所以得确定他们是64位还是32位           
```bash
cd C:\Windows\System32
dumpbin /headers notepad.exe
cd C:\Windows\System32
dumpbin /headers notepad.exe
D:\YearJunior1\SoftwareProjectSecurityDevelopmentLifecycle\EXP6\IATHook\Debug
dumpbin /headers IATHook.exe
```
```windows```下面的notepad是64位的，```system32```下面的notepad是32位的,而我们所写的IATHook.exe是32位的，所以进行攻击的是32位的notepad                                                 
![](./img/64note.png)                               
![](./img/32notepad.png)                               
![](./img/32hook.png)                               
2. 修改记事本的关键函数是WriteFile()函数，所以我们得自己写一个满足自己设定的writefile()函数     
```cpp
int __stdcall Fake_WriteFile(__in HANDLE fFile,
	__in LPCVOID lpBuffer,
	__in DWORD nNumberOfBytesToWrite,
	__out LPDWORD lpNumberOfBytesWritten,
	__in LPOVERLAPPED lpOverlapped)
{
	
	LPFN_WriteFile fnOrigin = (LPFN_WriteFile)GetIATHookOrign(g_hHook_WriteFile);
	char buff1[256] = "hello hacker";  //代替的输出话语  
	DWORD dwWrite1;
	return fnOrigin(fFile, &buff1, strlen(buff1), &dwWrite1, lpOverlapped);
}
```
3. 根据通过windebug攻击记事本的讲解中可知，我们所要攻击的writefile函数是在kernel32.dll中，所以我们也要进行进一步修改                          
```cpp
IATHook(
			GetModuleHandleW(NULL) ,
			"kernel32.dll" , 
			"WriteFile" ,
			Fake_WriteFile,
			&g_hHook_WriteFile
		);
```
4. 最后main.c代码如下，IATHook.c的代码与实例代码保持不变                             
```cpp
#include <windows.h>

#include <stdio.h>

LONG IATHook(
	__in_opt void* pImageBase ,
	__in_opt char* pszImportDllName ,
	__in char* pszRoutineName ,
	__in void* pFakeRoutine ,
	__out HANDLE* phHook
);

LONG UnIATHook( __in HANDLE hHook );

void* GetIATHookOrign( __in HANDLE hHook );

typedef int (__stdcall *LPFN_WriteFile)( __in HANDLE fFile,
	__in LPCVOID lpBuffer, 
	__in DWORD nNumberOfBytesToWrite, 
	__out LPDWORD lpNumberOfBytesWritten, 
	__in LPOVERLAPPED lpOverlapped);

HANDLE g_hHook_WriteFile = NULL;
//////////////////////////////////////////////////////////////////////////

//int __stdcall Fake_MessageBoxA( __in_opt HWND hWnd , __in_opt char* lpText , __in_opt char* lpCaption , __in UINT uType)
//{
//	LPFN_MessageBoxA fnOrigin = (LPFN_MessageBoxA)GetIATHookOrign(g_hHook_MessageBoxA);
//
//	return fnOrigin(hWnd , "hello hack" , lpCaption , uType);
//}
int __stdcall Fake_WriteFile(__in HANDLE fFile,
	__in LPCVOID lpBuffer,
	__in DWORD nNumberOfBytesToWrite,
	__out LPDWORD lpNumberOfBytesWritten,
	__in LPOVERLAPPED lpOverlapped)
{
	
	LPFN_WriteFile fnOrigin = (LPFN_WriteFile)GetIATHookOrign(g_hHook_WriteFile);
	char buff1[256] = "hello hacker";
	DWORD dwWrite1;
	return fnOrigin(fFile, &buff1, strlen(buff1), &dwWrite1, lpOverlapped);
}

int __cdecl wmain(int nArgc, WCHAR** Argv)
{
	do 
	{
		UNREFERENCED_PARAMETER(nArgc);
		UNREFERENCED_PARAMETER(Argv);

		IATHook(
			GetModuleHandleW(NULL) ,
			"kernel32.dll" , 
			"WriteFile" ,
			Fake_WriteFile,
			&g_hHook_WriteFile
		);
		
		
		HANDLE hFILE = CreateFile("D://YearJunior1//SoftwareProjectSecurityDevelopmentLifecycle//EXP5//Hook//Debug// 1.txt", GENERIC_WRITE, 0, NULL, CREATE_ALWAYS, FILE_ATTRIBUTE_NORMAL, NULL);

		
		char buff[256] = "hello";

		DWORD dwWrite;

		WriteFile(hFILE, &buff, strlen(buff), &dwWrite, NULL); //实际记事本中显示的是hello

		UnIATHook(g_hHook_WriteFile);//攻击后显示的是hello hacker

		CloseHandle(hFILE);

		//MessageBoxA(NULL , "test" , "caption1" , 0);

		//UnIATHook( g_hHook_MessageBoxA);

		//MessageBoxA(NULL , "test" , "caption2" , 0);
	
	} while (FALSE);
	
	return 0;
}
```        
5. 代码中显示调用writefile这个函数时，输入的参数是hello，最后在记事本中显示的是hello hacker          
![](./img/hacker.png)                               
### DLL注入——攻击记事本

![](./img/)                               
![](./img/)                               
![](./img/)                               
![](./img/)                               
![](./img/)                               
![](./img/)                               
![](./img/)                               
![](./img/)                               
![](./img/)                               
![](./img/)                               
![](./img/)                               
![](./img/)                               
![](./img/)                               
![](./img/)                               
![](./img/)                               
![](./img/)                               
![](./img/)                               
![](./img/)                               
![](./img/)                               

## 实验问题

## 实验结论

## 参考资料
* [IATHook实例仓库](https://github.com/tinysec/iathook.git)              
* [C/C++ 文件操作之CreateFile、ReadFile和WriteFile](https://blog.csdn.net/jeanphorn/article/details/44982273)