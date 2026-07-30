# 헌마 교육 중 중요내용 정리

## RIP에 대하여
### RIP : CPU가 다음에 실행할 명령어의 주소를 가리키는 레지스터 / x86-64에서 사용하는 어셈블리어

RIP = 0x401100이라면 cpu는 0x401100에 있는 기계어 명령을 실행한다

예를 들어서
main();
vuln();
puts("DONE");
일 때 main()이 vuln()dmf 호출할 때, vuln()실행이 끝난 후 돌아올 주소를 stack에 저장한다. 이게 savedRIP, 즉 반환주소이다.
  ┌────────────────────┐
  │ saved RIP          │ → vuln 종료 후 실행할 주소
  ├────────────────────┤
  │ saved RBP          │
  ├────────────────────┤
  │ char buf[64]       │
  └────────────────────┘
정상적인 경우는
savedRIP = main함수의 다음 명렁어 주소
공격 측면에서 바라보면
savedRIP = win() {bin/sh/(공격 바이너리, 함수)} 로 이동한다.
함수의 ret실행 시  cpu는 win()으로 이동

프로그램에 
void win() {
  system("/bin/sh");
}
가 있다고 가정할 때,
win()주소가 0x401169라면 payload를 
\x90이나 A X savedRIP까지의 거리0x401169인데
ret이 실행되면 
RIP = 0x401169가 되어 win()이 실행된다.
---

공격자가 일반적으로 직접 현재 RIP에 값을 쓰는 것은 아닙니다. 먼저 Stack에 저장된 saved RIP를 덮는다.

  Buffer Overflow로 saved RIP 변경
                ↓
  함수에서 ret 실행 -- ret실행이 무슨 말?
                ↓
  saved RIP가 실제 RIP로 들어감
                ↓
  실행 흐름 변경

  따라서 RIP가 중요한 이유는 다음과 같다.

--> RIP를 제어한다는 것은 CPU가 다음에 어떤 코드를 실행할지 제어한다는 의미이기 때문.

## Shellcode
shellcode는 공격자가 실행하려는 기계어 코드

흔히 아는 기능:
- /bin/sh 실행
- 파일 열기 및 읽기
- 네트워크 연결
- Reverse shell 연결
등이 있다.

예를 들어서 C코드로 기능 구현을 해보면
execve("/bin/sh", ...);

이를 CPU가 직접 실행할 수 있는 기계어 바이트로 만들면 다음과 같은 형태가 됩니다.

shellcode = b"\x48\x31\xf6\x56\x48..."

문자열처럼 보이지만 실제로는 CPU 명령어이다.

연장선으로
### Stack shellcode
Stack shellcode는 기계어 코드를 Stack의 버퍼에 넣고 그 주소로 실행 흐름을 보내는 방식
- Stack
  ┌──────────────────────────┐
  │ saved RIP = buf 주소     │
  ├──────────────────────────┤
  │ saved RBP                │
  ├──────────────────────────┤
  │ shellcode가 들어간 buf   │
  └──────────────────────────┘

payload작성 구조
-  [shellcode][padding][buf 주소]
실행 과정 :
1) 입력을 통해 buf에 shellcode저장
2) Overflow로 saved RIP를 buf 주소로 변경
3) 함수에서 ret 실행
4) RIP가 buf 주소로 변경
5) CPU가 buf의 바이트를 명령어로 실행
6) /bin/sh 실행

## NX와 관게

Stack Shellcode가 실행되려면 Stack에 실행 권한이 있어야 한다.

```text
Stack 권한: rwx -> 읽기, 쓰기, 실행 가능
```

NX가 활성화되면 Stack은 일반적으로 x(실행권한)이 불가능하게 된다
즉,
```text
Stack에 shellcode 저장 → 가능
Stack의 shellcode 실행 → NX 때문에 실패
```

이때 새로운 shellcode를 Stack에서 실행하는 대신 이미 실행 권한이 있는 코드를 재사용한다.

- `win()` 호출
- libc의 `system()` 호출
- ROP gadget 연결

---
  
## PIE와 바이너리 코드 주소 변경

PIE는 실행 파일 자체가 메모리의 어느 위치에 로드되더라도 실행될 수 있게 만드는 방식이다.
ASLR과 함께 적용되면 프로그램 내부 함수와 gadget의 실제 주소가 실행할 때마다 달라진다

- PIE가 없는 경우
-  - 고정주소
- PIE가 있는 경우
-  - 실행할 때마다 프로그램의 베이스 주소가 바뀐다.
즉, 

```text
함수와 gadget 주소가 고정
→ 분석한 주소를 payload에 바로 사용
→ 원하는 코드로 이동하기 쉬움
```

PIE가 있으면:

```text
주소가 실행마다 변경
→ 주소를 먼저 유출
→ PIE base 계산
→ 실제 목적지 주소 계산
→ payload 구성
```

다만 PIE가 없다고 자동으로 공격이 되는 것은 아니다. saved RIP, 함수 포인터 등을 덮을 수 있는 **별도의 취약점**이 필요하다.

주의할 점 
- Offset은 고정
절대 주소는 바뀌어도 바이너리 내부의 상대 offset은 고정이다.

실행 파일 내부 주소 하나를 유출했다면:

```python
pie_base = leaked_main - main_offset
win = pie_base + win_offset
```
처럼 필요한 주소를 다시 계산할 수 있다.

## libc란?
libc는 Linux에서 사용하는 대표적인 C 표준 라이브러리다. 
보통 `libc.so.6`이라는 파일로 존재한다.
다음과 같은 함수들의 실제 코드가 들어 있다.

```c
printf();
puts();
read();
write();
malloc();
free();
strcpy();
system();
exit();
```

대부분의 동적 링크 C 프로그램은 이 함수들의 기계어 코드를 실행 파일에 전부 포함하지 않는다. 실행할 때 libc를 메모리에 불러와 사용한다.
작동 구조는 
```text
프로그램
  ├─ main()
  ├─ vuln()
  └─ printf() 호출 ─────→ libc.so.6 안의 printf()
```
### exploit에서 libc가 중요한 이유
libc에는 공격자가 재사용할 수 있는 유용한 기능이 이미 존재한다.

대표적으로:

```c
system("/bin/sh");
```

- `system()` 함수가 libc에 존재한다.
- `"/bin/sh"` 문자열도 libc 안에서 찾을 수 있는 경우가 많다.

NX 때문에 Stack Shellcode를 실행할 수 없어도 실행 권한이 있는 libc 코드는 호출할 수 있다.

```text
Stack Shellcode 직접 실행 → NX가 차단
libc의 system() 재사용   → 가능
```

이러한 공격을 **ret2libc**라고 한다.

### libc와 ASLR

libc의 base address는 ASLR 때문에 실행할 때마다 바뀐다. 하지만 같은 libc 파일 내부의 함수 offset은 고정된다.

```text
puts 실제 주소   = libc base + puts offset
system 실제 주소 = libc base + system offset
```

`puts()`의 실제 주소를 유출했다면:

```python
libc_base = puts_addr - libc.sym["puts"]
system = libc_base + libc.sym["system"]
binsh = libc_base + next(libc.search(b"/bin/sh\x00"))
```
#### 추가, libc 버전이 중요한 이유

함수 offset은 libc 버전마다 다를 수 있다. 로컬과 원격 서버가 서로 다른 libc를 사용하면 주소 계산이 틀어진다.

CTF에서 `libc.so.6` 파일을 제공하는 이유가 바로 정확한 함수 offset을 사용하게 하기 위해서다.

---
