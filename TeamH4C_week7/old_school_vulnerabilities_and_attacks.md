Old School Vulnerabilities And Attacks
======================================


Contents
--------
1. 버퍼 오버플로우
   1. 변수가 스택에 배치되는 순서
   2. 쉘코드로 점프하기
	  1. 스택 프레임
	  2. 쉘코드로 점프하는 방법
   3. NOP Sled
2. 형식 문자열 버그
   1. x86과 x64 시스템에서의 printf 함수
   2. 형식 문자열 취약점
3. 메모리 쓰기 프리미티브
4. References



# 버퍼 오버플로우
 버퍼 (buffer)는 [3, Ch. 8.5.]가 설명하는 전형적인 표준 입출력의 구현에서 볼 수 있듯이 데이터를 이동시킬 때 일시적으로 보관하기 위한 메모리이다. 이러한 버퍼의 필요성은 [4]가 보이고, 그것은 상대적으로 매우 큰 양의 출력 데이터를 효율적으로 표기하는 문제가 자기 기록 장치라는 버퍼를 출력 장치로 제공하였을 때 해결됨을 보인 것이다. 이는 버퍼의 초기 구현으로 볼 수 있고, 큰 데이터의 이동을 효율적으로 수행한다는 목적은 오늘날에도 그대로 적용된다.

 버퍼 오버플로우 취약점은 프로그램이 버퍼에 저장될 것이라고 기대하는 크기를 초과하는 양의 데이터를 입력할 수 있을 때 발생하는 취약점이다. 이 취약점을 공격하는 방법은 버퍼가 어떤 메모리 영역에 위치하는지에 따라 다르다. 예를 들어, [5]는 스택 버퍼 오버플로우 취약점을 공격하는 방법을 설명한다.

## 변수가 스택에 배치되는 순서
 [6]은 하위 연산 (subsidiary operation)을 시작하고자 할 때 상위 연산 (major operation)으로부터 벗어나고, 하위 연산이 종료되었을 때 상위 연산을 계속 진행하는 방법을 설명하기 위해 상위 연산 탈출 지점 등이 적힌 노트인 딜레이 라인 (delay lines)을 제시한다. 그리고 가장 최근의 노트를 가리키는 것으로 TS를 두며 이는 하위 연산이 실행되고 종료됨에 따라 수정된다고 설명한다. 이때 수정의 방법은 하위 연산이 실행될 때 노트에 탈출 지점을 적고 (burying), 하위 연산이 종료될 때 노트에서 탈출 지점을 없애는 (disinterring) 것으로, 상기의 두 동작을 수행하는 명령어를 각각 BURY, UNBURY로 정의한다. [6]이 설명하는 개념과 동작은 현대의 스택과 크게 다르지 않고, 이들을 현대 스택에 비추어보면 딜레이 라인이 스택 메모리에 대응되고, TS가 스택 포인터에 대응되며, BURY, UNBURY는 각각 PUSH, POP에 대응됨을 알 수 있다.

```
Delay Line: [ Note1 ][ Note2 ][ Note3 ] ...
            ^
            |
            +--TS
			
BURY:
    Delay Line: [ New Note ][ Note1 ][ Note2 ][ Note3 ] ...
                ^
                |
                +--TS

UNBURY:
    Delay Line: [ Note1 ][ Note2 ][ Note3 ] ...
                ^
                |
                +--TS
    Disintered: [ New Note ]
```

이렇게 마지막에 들어간 것이 첫 번째로 나오는 형태를 후입선출 (Last In First Out, LIFO)이라고 부르고, 스택은 대표적인 후입선출 자료구조이다.

 이러한 스택은 주로 함수의 호출과 리턴을 구현할 때 사용된다. 물론, [1]은 함수의 호출과 리턴을 스택으로 구현해야 함을 명시하지는 않는다. 그러나 대부분의 C 컴파일러 (또는 C 구현체, C implementation)들은 함수의 호출과 리턴을 구현할 때 스택을 사용한다는 것이다. 따라서 스택의 요소들은 프로그램이 구동되는 기계와 언어의 구현체에 의존하게 된다. 이러한 스택의 요소 중에서 대표적인 것이 바로 지역 변수이다. 지역 변수는 대부분 스택에 배치되지만 [1]은 이를 명시하고 있지 않다. [1]이 명시하는 것은 지역 변수의 선언 시 키워드, 선언 시 코드상에서의 위치 등에 따른 기억수명 (Storage duration) 뿐이다. 즉, [1]은 스택의 요소에 대해 구현체가 (표준에 순응한다면) 마땅히 해야만 하는 동작들을 기술하고, 구현체는 [1]에 명시된 동작을 나름의 방법 (스택 등)으로 구현한다는 것이다.

 상기에 서술하였듯이 스택에는 프로그램의 실행 흐름에 관여하는 데이터 (하위 연산이 실행될 때의 탈출 지점, 리턴 주소 등)가 존재하고 이 데이터를 어떻게 스택에 배치할지 결정하는 것은 컴파일러 (구현체)의 몫이다. 예를 들어, 프로그램에서 사용자 입력을 받아들이는 배열 A (12 바이트)가 있고, A의 값에 따라 다른 값을 가지는 변수 B (4 바이트)가 있으며, A와 B가 다음과 같이 스택에 배치되었다고 가정하자.

```
(LOW) ... [AAAAAAAAAAAA][BBBB] ... (HIGH)
```

그리고 다음과 같은 코드가 동작하는 중이라고 가정하자.

```C
#include <stdio.h>
#include <string.h>

/* gcc -fno-stack-protector ex1_stack_BOF.c -o ex1_stack_BOF */
int main()
{
	char A[12];
	int B;
	
	gets(A);
	B = 0;
	if (strcmp(A, "Password") == 0)
		B = 1;
	
	if (B)
		printf("Access Granted..\n");
	else
		printf("Access Denied..\n");
	return 0;
}
```

위 코드에서 사용자는 12 바이트를 초과하는 데이터를 입력할 수 있다 (오버플로우). 그리고 초과된 데이터는 기존에 존재하던 데이터를 덮어쓰게 된다. 이를 위의 스택 배치에 비추어서 생각해보면 사용자가 입력한, 12 바이트를 초과한 데이터는 변수 B의 값을 조작할 수 있다는 것을 알 수 있다. 여기서 B는 if (B) { ... } else { ... } 블록에 따른, 프로그램의 실행 흐름에 관여하는 변수이다. 즉, 사용자는 오버플로우를 발생시켜 프로그램의 실행 흐름을 조작할 수 있다는 것이다.

```bash
$ gcc -fno-stack-protector ex1_stack_BOF.c -o ex1_stack_BOF
ex1_stack_BOF.c: In function ‘main’:
ex1_stack_BOF.c:9:5: warning: implicit declaration of function ‘gets’; did you mean ‘fgets’? [-Wimplicit-function-declaration]
    9 |     gets(A);
      |     ^~~~
      |     fgets
/usr/bin/ld: /tmp/ccoWwIX1.o: in function `main':
ex1_stack_BOF.c:(.text+0x19): warning: the `gets' function is dangerous and should not be used.
$ ./ex1_stack_BOF
AAAAAAAAAAAAA
Access Granted..
$
```

 앞의 예시는 스택 배치가 가정한대로 구성되어야만 성립한다. 하지만 이전에 설명했듯이 스택 배치의 구성은 컴파일러가 관심가지는 사항이고, 컴파일러마다 다르기 때문에 위에서 가정한대로 스택 배치를 구성하는 컴파일러로 컴파일된 프로그램에서만 성립하게 된다. 이렇게 (특정 컴파일러상에서만 성립함에도) 사용자 입력을 받는 변수가 실행 흐름에 관여하는 변수의 앞에 배치되어 발생하는 문제는 [8]이 해결책을 제시하여 일부 해결되었다. 여기서 [8]이 제시한 해결책은 지역변수의 배치 순서를 조정하는, 즉 사용자 입력을 받아들이는 변수가 다른 변수의 뒤에 위치하도록 만드는 SSP (Stack Smashing Protector)이다. 그리고 현대의 컴파일러 대부분은 이 기능을 제공한다.

## 쉘코드로 점프하기

### 스택 프레임
 [9, p. 36]은 다음과 같은 표준 스택 프레임을 제시한다.

```
Position         Contents          Frame
--------------+-----------------+---------- High address
4n + 8 (%ebp) | argument word n | Previous
 ...          | ...             |
8 (%ebp)      | argument word 0 |
--------------+-----------------+----------
4 (%ebp)      | return address  | Current
              +-----------------+
0 (%ebp)      | previous %ebp   |
              | (optional)      |
              +-----------------+
-4 (%ebp)     | unspecified     |
 ...          | ...             |
0 (%esp)      | variable size   |
--------------+-----------------+---------- Low address
```

그리고 [9, pp. 39 - 40]는 함수 프롤로그와 에필로그의 예시를 제시한다.

```
prologue:
	push %ebp		/ save frame pointer
	movl %esp, %ebp	/save new frame pointer
	subl $80, %esp	/ allocate stack space
	pushl %edi		/ save local register
	pushl %esi		/ save local register
	pushl %ebx		/ save local register
	movl %edi, %eax	/ set up return value
```

```
epilogue:
	popl %ebx	/ restore local register
	popl %esi	/ restore local register
	popl %edi	/ restore local register
	leave		/ restore frame pointer
	ret 		/ pop return address
```	

위 어셈블리어 코드에서 확인할 수 있듯이 함수 프롤로그와 에필로그는 함수의 정상적인 호출과 리턴을 위한 작업들을 포함한다. 그리고 스택 프레임에는 함수의 정상적인 호출과 리턴을 위한 정보들이 저장된다. 즉, 프로그램의 실행 흐름에 보다 직접적으로 관여하는 정보가 스택 프레임에 저장된다는 것이다.

### 쉘코드로 점프하는 방법
 앞에서 스택 프레임에 저장되는 정보가 프로그램의 실행 흐름에 깊게 관여한다고 설명하였다. 이 중에서 대표적인 것이 바로 리턴 주소 (Return Address)이다. 리턴 주소는 함수 에필로그에서 ret 명령어를 통해 pop되고 이는 명령 포인터 (Instruction Pointer)에 로드된다. 그리고 이렇게 로드된 리턴 주소에 위치한 명령어가 실행된다. 이때 리턴 주소를 조작하여 공격 코드의 주소 또는 공격 코드의 주소를 최종적으로 명령 포인터에 로드하는 코드의 주소로 변경할 수 있다면, 프로그램의 실행 흐름을 바꿀 수 있다. 일반적으로 프로그램의 실행 흐름을 조작할 수 있을 때 그 목적지는 쉘 (명령어 해석기)을 호출하는 명령어 코드가 저장된 곳이 되는데 그 이유는 리눅스의 경우에 쉘을 통해 타깃 컴퓨터를 제어할 수 있기 때문이다. 여기서는 쉘을 호출하기 위한 명령어 코드 조각이 사용되는데 이는 보통 쉘코드라고 불린다. 여기서는 쉘코드보다는 쉘코드로 점프하는 것 자체에 초점을 맞출 것이다.

 예를 들어, 다음과 같이 구성된 스택을 생각해보자.

```
(LOW) ... [BBBBBBBBBBBB][FFFF][RRRR][SSSSSSSS...] ... (HIGH)

@ B: Input Buffer
@ F: Saved Frame Pointer
@ R: Return Address
@ S: Shell Code

@ 위 그림에서 대괄호 사이에 위치한 알파벳 문자 하나는 1 바이트를 의미함
```

위와 같은 스택에서 사용자가 다음과 같은 입력을 주어서

```
["AAAAAAAAAAAA"]["AAAA"][JJJJ][SSSSSSSS...]

@ J: Address of jmp *%esp
@ S: Shell Code

@ 위 그림에서 대괄호 사이에 위치한, 큰따옴표로 묶이지 않은 알파벳 문자 하나는 1 바이트를 의미함
```

스택이 다음과 같은 상태로

```
(LOW) ... ["AAAAAAAAAAAA"]["AAAA"][JJJJ][SSSSSSSS...] ... (HIGH)
```
존재하는 상황을 생각해보자. 이 상황에서 함수 에필로그가 진행되는 것을 살펴보면 다음과 같다.

```
1. mov %ebp, %esp
(LOW) ... ["AAAAAAAAAAAA"]["AAAA"][JJJJ][SSSSSSSS...] ... (HIGH)
                          ^
                          |
                          +---%esp, %ebp

2. pop %ebp
(LOW) ... ["AAAAAAAAAAAA"]["AAAA"][JJJJ][SSSSSSSS...] ... (HIGH)
                                  ^
                                  |
                                  +---%esp
                                      %ebp = 0x41414141
									  
3. ret
(LOW) ... ["AAAAAAAAAAAA"]["AAAA"][JJJJ][SSSSSSSS...] ... (HIGH)
                                        ^
                                        |
                                        +---%esp
                                            %ebp = 0x41414141
                                            %eip = JJJJ
											
@ J: Address of jmp *%esp
@ S: Shell Code

@ 위 그림에서 대괄호 사이에 위치한, 큰따옴표로 묶이지 않은 알파벳 문자 하나는 1 바이트를 의미함
```	

위 진행 결과를 보면 %eip가 %esp로 점프하는 명령어의 주소를 가지게 됨을 알 수 있다. 이때 %esp에는 쉘 코드가 저장되어 있으므로 쉘 코드가 실행될 것이다.

 그럼 %esp로 점프하는 어셈블리 명령어의 주소를 얻는 방법에 대해 살펴보자. %esp로 점프하는 명령어의 주소를 얻기 위해서는 먼저 그 명령어의 기계어 코드를 얻어야 한다. 그 이유는 CPU가 최종적으로 실행하는 것은 기계어이고, 설령 얻고자 하는 명령어가 없다고 하더라도 명령어가 갖는 기계어의 조합을 쪼개서 CPU가 인식하는 것을 공격에 필요한 명령어로 바꿀 수 있기 때문이다. 여기서 얻고자 하는 기계어 코드를 얻는 좋은 방법은 다음과 같은 코드를 작성하고 컴파일한 후에 실행파일을 확인하는 것이다.

```C
/* gcc ex2_find_machine_code.c -o ex2_find_machine_code */
int main()
{
    __asm__ __volatile__ (
		"jmp *%esp\n\t"
	);
    return 0;
}
```

```bash
$ objdump -D ./ex2_find_machine_code | grep -A10 main | tail -n 11
080483db <main>:
 80483db:    55                       push   %ebp
 80483dc:    89 e5                    mov    %esp,%ebp
 80483de:    ff e4                    jmp    *%esp
 80483e0:    b8 00 00 00 00           mov    $0x0,%eax
 80483e5:    5d                       pop    %ebp
 80483e6:    c3                       ret    
 80483e7:    66 90                    xchg   %ax,%ax
 80483e9:    66 90                    xchg   %ax,%ax
 80483eb:    66 90                    xchg   %ax,%ax
 80483ed:    66 90                    xchg   %ax,%ax
$
```

위 결과로부터 "ff e4"가 %esp로 점프하는 코드의 기계어임을 알 수 있다. 이 기계어 코드를 실행파일에서 검색한 예시는 다음과 같다.

```bash
$ objdump -D ./example_jmp_to_esp | grep "ff e4"
 8048756:       3d ff e4 00 00          cmp    $0xe4ff,%eax
$
```

위 예시에는 "jmp *%esp"라는 명령어는 없다. 다만, "ff e4"를 포함하는 명령어가 있을 뿐이다. 이때, 0x8048756부터가 아니라 0x8048757부터 본다면 CPU는 "ff e4"를 인식하게 된다. 따라서 %esp로 점프하는 명령어의 주소는 0x8048757이 된다.

 이 방법의 가장 큰 단점은 "ff e4"라는 기계어의 주소를 알아야 한다는 것이다. 즉, 해당 기계어가 실행파일 등에 없으면, 쓸 수 없게 된다.
 
## NOP Sled

# 형식 문자열 버그
 [11]은 csh를 테스트하던 도중 printf 함수가 인자 (parameter)를 기대하는 상황에서 아무것도 제공되지 않을 때 에러를 발생시킬 수 있음을 확인하였다. 그리고 이는 printf 함수를 사용하는 에러 핸들링 함수에 전달되는 문자열로 인해 발생할 수 있고 이를 해결하기 위한 방법으로 printf 함수가 아닌 puts 함수를 사용할 것을 제안하였다.

## x86과 x64 시스템에서의 printf 함수
 [10]은 printf 함수 계열이 형식 (format)에 따른 출력을 생성함을 설명하면서 %로 시작하는 문자인 형식 지정자 (conversion specifier)에 의해 문자열 뒤의 인자 (arguments)를 해석하여 배치함을 기술하고 있다.
 
 여기서 [11]의 설명과 같이 형식 지정자의 개수와 인자 (arguments)의 개수가 일치하지 않을 때 문제가 발생한다. [9, 12]는 함수호출규약 (Function Calling Convention)의 차이로 인해 x86 시스템은 스택을 사용하여 가변 인자 (variable arguments)를 구현하고, x64 시스템의 경우에는 인자의 클래스를 나누어 인자를 레지스터 또는 스택에 적절히 배치하는 방법으로 가변 인자를 구현해야 함을 보였다. 그러나 현 상황과 같이 형식 지정자의 개수가 인자 (arguments)의 개수보다 많을 때, 그 초과분에 대해서는 x86과 x64의 동작이 같아지게 된다. 그 이유는 [12]가 설명하듯이 레지스터로 인자를 전달하는 경우에는 그 인자의 타입을 해석할 수 있어야 적절한 레지스터에 저장하여 전달할 수 있는데 현 상황의 경우에는 초과분에 한해 그렇지 못하기 때문이다. 따라서 스택으로부터 데이터를 읽어들이게 되고 결국 x86에서의 동작과 같아진다.

## 형식 문자열 취약점
 형식 문자열 취약점 (Format String Vulnerability)은 다음과 같이 printf 함수를 마치 puts 함수처럼 사용할 때 발생한다. 먼저 다음과 같은 코드를 가정해보자.
 
```C
#include <stdio.h>

/* gcc ex3_fmt_string_vuln.c -o ex3_fmt_string_vuln */
int main(int argc, char *argv[])
{
	printf(argv[1]);
	return 0;
}
```

그리고 위 프로그램에 전달할 인자가 다음과 같은 명령어의 실행 결과라고 하자.

```bash
$ perl -e 'print "A" x 4 . "/%lx" x 1024;'
```

위 명령어의 실행 결과를 복사한 문자열을 인자로 전달하면

```bash
$ ./ex3_fmt_string_vuln AAAA/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx/%lx
```

다음과 같은 결과를 얻는다.

```bash
AAAA/7ffca5fda088/7ffca5fda0a0/560e27aecdc0/7f0f3aaddf10/7f0f3ab02040/7ffca5fda088/200000000/2/7f0f3a8ecd90/0/560e27aea149/200000000/7ffca5fda088/0/edbfed02789c2693/7ffca5fda088/560e27aea149/560e27aecdc0/7f0f3ab36040/1246a6f947982693/13a1981fe2102693/0/0/0/0/0/ae9dcf9e1e6dfd00/0/7f0f3a8ece40/7ffca5fda0a0/560e27aecdc0/7f0f3ab372e0/0/0/560e27aea060/7ffca5fda080/0/0/560e27aea085/7ffca5fda078/1c/2/7ffca5fdb2ee/7ffca5fdb304/0/7ffca5fdc309/7ffca5fdc319/7ffca5fdc38f/7ffca5fdc3a2/7ffca5fdc3b6/7ffca5fdc3e3/7ffca5fdc404/7ffca5fdc41b/7ffca5fdc447/7ffca5fdc457/7ffca5fdc46e/7ffca5fdc48e/7ffca5fdc4a2/7ffca5fdc4cb/7ffca5fdc4df/7ffca5fdc4f6/7ffca5fdc50e/7ffca5fdc52a/7ffca5fdc57d/7ffca5fdc58e/7ffca5fdc5a9/7ffca5fdc5c2/7ffca5fdc5d8/7ffca5fdc60e/7ffca5fdc622/7ffca5fdc634/7ffca5fdc646/7ffca5fdc65b/7ffca5fdc66c/7ffca5fdcc5b/7ffca5fdcc7c/7ffca5fdcc8d/7ffca5fdcca7/7ffca5fdccfd/7ffca5fdcd14/7ffca5fdcd36/7ffca5fdcd4d/7ffca5fdcd61/7ffca5fdcd7f/7ffca5fdcd9f/7ffca5fdcdad/7ffca5fdcdcb/7ffca5fdcdd6/7ffca5fdcdde/7ffca5fdcdf7/7ffca5fdce09/7ffca5fdce24/7ffca5fdce43/7ffca5fdce57/7ffca5fdceac/7ffca5fdcf1e/7ffca5fdcf30/7ffca5fdcf66/7ffca5fdcf7d/7ffca5fdcf95/0/21/7ffca5ffe000/33/e30/10/f8bfbff/6/1000/11/64/3/560e27ae9040/4/38/5/d/7/7f0f3aafc000/8/0/9/560e27aea060/b/3e8/c/3e8/d/3e8/e/3e8/17/0/19/7ffca5fda3b9/1a/2/1f/7ffca5fdcfe2/f/7ffca5fda3c9/0/0/0/9dcf9e1e6dfddc00/49f6dff6813c4cae/34365f36387813/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/0/2f2e000000000000/5f746d665f337865/765f676e69727473/41414141006e6c75/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f/786c252f786c252f
```

위 결과를 잘 살펴보면 "/41414141006e6c75/786c252f786c252f"라는 바이트열을 확인할 수 있다. 이 바이트열은 리틀 엔디언으로 저장된 것이 그대로 출력된 것이므로 "/75 6c 6e 00 41 41 41 41/2f 25 6c 78 2f 25 6c 78"가 된다 (구분할 수 있도록 공백을 두었다). 그리고 이를 아스키 코드표에 따라 변환하면 "/u l n (nil) A A A A// % l x / % l x"이 된다 (마찬가지로 구분할 수 있도록 공백을 두었다). 즉, 인자로 전달된 문자열이 읽어들여졌고 전달된 문자열 중에서 "%lx"라는 문자열이 형식 지정자로 해석되어 해당되는 인자가 스택으로부터 형식에 따라 출력되었다. 이 과정이 계속 진행되면서 스택에서 인자로 전달된 문자열이 위치한 부분까지 출력된 것이다. 현재 상황을 그림으로 표현하면 다음과 같다.

```
[Format String]         [Stack]
+--------------+
|     AAAA     |                         -> printf("AAAA");
+--------------+    +------------------+
|     /%lx     | -> | 7ffc36f4edf8     | -> printf("/%lx", 0x7ffc36f4edf8);
+--------------+    +------------------+
|     ...      |    |      ...         |        ...
+--------------+    +------------------+
|     /%lx     | -> | 41414141006e6c75 | -> printf("/%lx", 0x41414141006e6c75);
+--------------+    +------------------+
|     /%lx     | -> | 786c252f786c252f | -> printf("/%lx", 0x786c252f786c252f);
+--------------+    +------------------+
|     ...      |    |      ...         |        ...
+--------------+    +------------------+
```

위 결과로부터 형식 문자열 버그를 통해 printf 함수에 전달한 문자열을 특정 형식 지정자로 출력할 수 있음을 알 수 있다. 즉, %s를 통해 해당 메모리 주소의 내용을 읽는 것도, %n을 통해 해당 메모리 주소에 값을 쓰는 것도 (메모리 주소가 유효하다면) 가능하다는 것이다. 이러한 임의 주소 쓰기가 사용되는 대표적인 곳이 동적으로 링크되느 라이브러리 함수들의 절대 주소가 저장되는 전역 오프셋 테이블 (Global Offset Table, GOT)이다.

# 메모리 쓰기 프리미티브
 메모리 쓰기 프리미티브의 용법은 [13]의 다음과 같은 설명에서 찾을 수 있다.
 
> I like to call this a write-anything-anywhere primitive, and the trick described here can be used whenever you have a write-anything-anywhere primitive,  be it a format string, an overflow over the "destination pointer of a strcpy()", several free()s in a row, a ret2memcpy buffer overflow, etc.

여기서 프리미티브 (primitive)의 용법을 살펴보면 '요소'의 의미로 사용되었음을 알 수 있다. 즉, 메모리 쓰기 프리미티브란 메모리에 쓸 수 있도록 만드는 프로그램 요소를 말하는 것이다. 따라서 1에서는 오버플로우를 허용하는 취약한 함수에 전달되고, 스택에 존재하는 인자가 메모리 쓰기 프리미티브가 되고, 2에서는 형식 문자열이 메모리 쓰기 프리미티브가 되는 것이다.

# References
[1] ISO/IEC JTC 1/SC 22/WG 14, "ISO/IEC 9899:1999, Programming languages -- C", ISO/IEC, 1999

[2] James van Artsdalen, et al., "GCC, the GNU Compiler Collection", Free Software Foundation, Inc., 2022. [Online]. Available: https://gcc.gnu.org/, [Accessed Jun. 04, 2022]

[3] Brian W. Kernighan, Dennis M. Ritchie, "The C Programming Language Second Edition", AT&T Bell Laboratories, 1988

[4] Russel A. Kirsch, "SEAC MAINTENANCE MANUAL THE OUTSCRIBER", NATIONAL BUREAU OF STANDARDS, 1953

[5] Aleph One, "Smashing The Stack For Fun And Profit", Phrack Volume 7, Issue 49, File 14, 1996

[6] Turing, A. M., "Proposals for Development in Mathematics Division of an Automatic Computing Engine (ACE), Report E882", Executive Committee, NPL, 1945

[7] B. E. Carpenter and R. W. Doran, "The other Turing machine", Department of Computer Science, Massey University, 1975

[8] Etoh, Hiroaki, "Gcc extension for protecting applications from stack-smashing attacks", IBM, 2004

[9] "SYSTEM V APPLICAITON BINARY INTERFACE Intel386 Architecture Processor Supplement", The Santa Cruz Operation, Inc.

[10] Michael Kerrisk, "printf(3) -- Linux manual page", man7.org, 2021. [Online]. Available: https://man7.org/linux/man-pages/man3/printf.3.html

[11] Barton P. Miller, et al., "An empirical study of the reliability of UNIX utilities", Communications of the ACM, Volume 33, Issue 12, 1990

[12] Michael Matz, et al., "System V Application Binary Interface AMD64 Arhitecture Processor Supplement Draft"

[13] gera, "Advances in format string exploitation", Phrack Issue 0x3b, Phile 0x07, 2002
