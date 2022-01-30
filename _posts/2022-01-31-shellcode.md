---
title: "[pwnable] shellcode"
date: 2022-01-23T00:00:00+00:00
author: 이정목
layout: post
permalink: /shellcode-설명/
categories: pwnable
tags: [dreamhack, pwnable, shellcode]
---

드림핵의 강의를 차근차근 수강해 보려고 시도하고 있다. 나는 요 로드맵 [링크](https://dreamhack.io/lecture/roadmaps/2)를 따라서 수강하고 있는데 여간 빡센 게 아니다... ㅎㅎ 여기서는 [링크](https://dreamhack.io/lecture/courses/50)의 예제를 따라하고 설명해볼 것이다!

## 정의
쉘코드는 어셈블리 코드 조각을 말한다. 보통은 쉘을 딸 때 많이 사용하기 때문에 쉘코드라고 부른다. 

## 어셈블리
쉘코드는 어셈블리 코드 조각이기 때문에 결국 어셈블리로 어떻게 파일을 읽고, 쓰고, 열고 실행하는지를 알아야 한다. 보통은 해당하는 레지스터에 알맞은 값을 넣어 준 후에, 마지막에 syscall을 치면 어셈블리어로 syscall의 활용이 가능해진다. 예를 들어 파일을 읽는 c 코드 open(filename, 0, 0)를 어셈블리어의 관점에서 살펴보면, 가장 먼저 rdi에 파일 이름을 넣어주고, rsi와 rdx에는 0을 넣어준 후에 rax에는 2 값을 넣어주면 된다. 이후 syscall을 해주면 파일을 열 수 있다. 밑은 read, write, open의 syscall을 정리한 것이다. (드림핵 내의 표)

|syscall|rax (arg0)|rdi (arg1)|rsi (arg2)|rdx (arg3)|
|-------|---|---|---|---|
|read|0|unsigned int fd|char *buf|size_t count|
|write|1|unsigned int fd|const char *buf|size_t count|
|open|2|const char *filename|int flags|umode_t mode|

## orw 쉘코드 작성 ~~
우리가 예제로 작성할 c 코드는 다음과 같다. open, read, write syscall을 순서대로 실행한다. /tmp/flag 파일을 읽고 출력한다.

```c++
char buf[0x30];
int fd = open("/tmp/flag", RD_ONLY, NULL);
read(fd, buf, 0x30); 
write(1, buf, 0x30);
```

### 1. open("/tmp/flag", RD_ONLY, NULL)
위에서 설명한 대로 /tmp/flag 문자열을 rdi에, rsi와 rdx는 맞는 숫자를 넣어주면 된다. open의 두번째 arg(rsi)에는 RD_ONLY(read only), O_WRONLY(write only), O_RDWR(read and write) 세 개가 올 수 있고 각각이 대응하는 숫자는 0, 1, 2이다. 따라서 지금 경우에는 RD_ONLY이므로 0을 rsi에 넣어주면 된다. rdx는 NULL이므로 0을 넣어준다.

/tmp/flag라는 문자열을 rdi에 넣어주기 위해서는 이걸 아스키 코드로 바꿔야 하는데, 조금 까다로워 보일 수 있다. [ascii_to_hex](https://m.blog.naver.com/pjok1122/221325791373) 를 이용하면 조금 쉽게 할 수 있다. 터미널에 python을 입력하면 python 쉘이 뜨는데, 여기서 나는 python2에서 사용하는 방법을 이용하였다. python3을 입력하면 python3의 쉘이 뜨고, python3에서 사용하는 방법을 이용하면 되는데 나는 첫번째가 더 편해서 첫번째 방법을 썼다. 파이썬 쉘에 "/tmp/flag".encode("hex")를 입력하면 해당 문자열을 16진수로 바꿨을 때 어떻게 되는지를 알 수 있다. 그러나 little endian이기 때문에, 두 자리씩 끊어서 거꾸로 넣어주어야 한다. 결국 2f746d702f666c6167 의 문자열을 '2f'74'6d'70'2f'66'6c'61'67' 이렇게 끊은 후, 뒤집어서 67616c662f706d742f 이 된다. 
![이미지](https://github.com/JungMok-Lee/JungMok-Lee.github.io/blob/master/assets/images/2022_01_31_python.png?raw=true)



이렇게 얻는 /tmp/flag의 문자열을 가장 먼저 스택에 올려 주어야 한다. 스택에 올리는 방법은 push를 사용한다. 내가 만약 push rbp를 하면 rsp(현재 스택 프레임)가 0x4만큼 적어지면서 위를 가리키고, 그렇게 생긴 공간에는 rbp의 값을 채워 넣는다. 따라서 우리는 push 0x616c662f706d742f를 해서 이 값을 스택에 올릴 것이다. 이것을 어떻게 해야 하냐면, 가장 먼저 67616c662f706d742f는 총 18자리이므로, 9바이트이다. 스택은 8바이트씩 값을 저장하므로, 우리는 push를 두번 해서 1바이트, 8바이트 이렇게 올릴 것이다. 따라서 어셈블리 코드는 다음과 같아진다. 먼저 67을 push 하고, rax에 나머지 문자열 (/tmp/fla 가 된다)을 담는다. push rax를 통해 push하여 스택에 올리면, 거의 다 끝났다.
이제 스택에 문자열이 담겨 있으므로 rsp가 문자열을 가리키고 있는 상태가 되고, 이 문자열을 rdi에 넣어주고, rsi, rdx, rax에 도 알맞게 값을 넣어주면 된다. 이후 syscall을 실행한다.
```c
push 0x67
mov rax, 0x616c662f706d742f 
push rax
mov rdi, rsp    ; rdi = "/tmp/flag"
xor rsi, rsi    ; rsi = 0 ; RD_ONLY
xor rdx, rdx    ; rdx = 0
mov rax, 2      ; rax = 2 ; syscall_open
syscall         ; open("/tmp/flag", RD_ONLY, NULL)
```
### 2. read(fd, buf, 0x30)
read와 write은 open과 매우 비슷하다. open의 syscall이 실행 된 후, 실행의 반환값은 rax에 담기고 open syscall의 반환값은 fd이다. 이 fd를 read의 첫 번째 인자로 넘겨주면 된다. 따라서 open 실행 직후에는 rax에 fd가 담겨 있고, 이 rax값을 rdi에 넘겨 주면 된다. rsi에는 읽어야 할 문자열이 담겨야 하므로 현재 스택 프레임(rsp)가 가리키고 있는 문자열 (/tmp/flag)를 담아주면 된다. 이때 0x30만큼 읽어야 하므로 rsi는 rsp에서 0x30만큼 빼준 값을 담는다. rdx는 길이이므로 마찬가지로 0x30을 담으면 된다. 
```c
mov rdi, rax      ; rdi = fd
mov rsi, rsp
sub rsi, 0x30     ; rsi = rsp-0x30 ; buf
mov rdx, 0x30     ; rdx = 0x30     ; len
mov rax, 0x0      ; rax = 0        ; syscall_read
syscall  
```
### 3. write(1, buf, 0x30)
write 또한 동일하다.
```c
mov rdi, 1        ; rdi = 1 ; fd = stdout
mov rax, 0x1      ; rax = 1 ; syscall_write
syscall           ; write(fd, buf, 0x30)
```


이렇게 어셈블리어 작성이 끝났다면, 이 어셈블리 코드를 컴파일해야 한다. 컴파일하는 방법에는 여러 가지가 있겠지만 우리는 .c 파일에 어셈블리 코드를 직접 넣어서 컴파일하는 방법을 사용하겠다! 방법은 다음과 같다. 저 중간 위치에 아까 위에서 쓴 1,2,3번의 코드를 넣어주면 된다. 항상 뒤에 \n을 붙인다.

```c
// File name: sh-skeleton.c
// Compile Option: gcc -o sh-skeleton sh-skeleton.c -masm=intel
__asm__(
    ".global run_sh\n"
    "run_sh:\n"
    "Input your shellcode here.\n"
    "Each line of your shellcode should be\n"
    "seperated by '\n'\n"
    "xor rdi, rdi   # rdi = 0\n"
    "mov rax, 0x3c	# rax = sys_exit\n"
    "syscall        # exit(0)");
void run_sh();
int main() { run_sh(); }
```

/tmp/flag에 직접 파일을 생성하는 방법에는 vim을 쓰거나 여러가지 방법이 있겠지만 너무 편리한 방법이 있다! echo를 사용하면 된다. 이후에는 이 명령어를 쳐서 .c파일을 컴파일하면 된다.
```c
echo "flag{this_is_open_read_write_shellcode!}" > /tmp/flag
gcc -o orw orw.c -masm=intel
```

이렇게 하면 실행파일이 만들어지는데, 이 실행파일을 쉘코드로 바꾸기 위해서는 또 다른 작업이 필요하다. 이 실행파일에 대해 터미널에
```c
objdump -d orw
```
를 입력하면 역어셈블을 할 수 있는데, 이 역어셈블 한 코드에서 기계어를 추출해내야 한다. 