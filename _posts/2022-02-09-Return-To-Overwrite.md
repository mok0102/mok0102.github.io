---
title: "[pwnable] Return To Overwrite"
date: 2022-02-08T00:00:00+00:00
author: 이정목
layout: post
permalink: /3273-두-수의-합/
categories: PS
tags: [백준, 투포인터, 두 수의 합]
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
위 그림의 스택 구조에서, buf와 sfp를 모두 A와 같은 아무 문자열로 밀어 버리고 ret에 get_shell의 구조를 넣으면 쉘을 딸 수 있다. 따라서 가장 먼저 해야 하는 것은 buf와 sfp가 얼마나의 공간을 차지하고 있는가이다. 그걸 알아야 몇개의 A로 buf와 sfp를 깔끔하게 밀 수 있는지를 알 수 있을 테니까. 가장먼저, 64비트 체계이기 때문에 스택 한 칸의 크기는 0x8이다. 만약 32비트라면 0x4가 된다. 따라서 buf는 찾아봐야겠지만, sfp는 0x8만큼 차지하고 있다고 알 수 있다. 
이제, buf가 얼마나 공간을 차지하는지를 찾아볼 것이다. buf가 0x28만큼 공간을 차지하고 있다고 생각하고 buf 0x28, sfp 0x8해서 0x28+0x8만큼 민다고 생각할 수 있는데, 내가 느끼기로는 c코드만 보고서는 buf의 크기를 정확하게 알 수 없다. 0x28일 때도 있고, alignment라고 해서 8의 배수를 맞추기 위해? 조금 더 커질 때도 있다. 근데 내 경험상 8의 배수랑도 상관이 잘 없는 거 같고, 그냥 gdb로 buf의 크기가 얼마만큼 할당됬는지를 보는 게 제일 마음 편하다. 
![이미지](https://github.com/JungMok-Lee/JungMok-Lee.github.io/blob/master/assets/images/2022_02_09_gdb_main.png?raw=true)