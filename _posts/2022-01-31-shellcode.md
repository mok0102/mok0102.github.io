---
title: "[pwnable]] shellcode"
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
위에서 설명한 대로 /tmp/flag 문자열을 rdi에, rsi와 rdx는 맞는 숫자를 넣어주면 된다. open의 두번째 arg(rsi)에는 RD_ONLY(read only), O_WRONLY(write only), O_RDWR(read and write) 세 개가 올 수 있고 각각이 대응하는 숫자는 0, 1, 2이다. 따라서 지금 경우에는 RD_ONLY이므로 0을 rsi에 넣어주면 된다.

/tmp/flag라는 문자열을 rdi에 넣어주기 위해서는 이걸 아스키 코드로 바꿔야 하는데, 조금 까다로워 보일 수 있다. [ascii_to_hex](https://m.blog.naver.com/pjok1122/221325791373) 를 이용하면 조금 쉽게 할 수 있다. 터미널에 python을 입력하면 python 쉘이 뜨는데, 
### 2. read(fd, buf, 0x30)
### 3. write(1, buf, 0x30)

