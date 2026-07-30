## GOT와 GOT 쓰기

GOT는 `Global Offset Table`의 약자다.

> 동적 라이브러리 함수의 실제 메모리 주소를 저장하는 표라고 이해하면 된다.

### GOT가 필요한 이유

프로그램을 컴파일할 때는 libc가 실행 중 어느 주소에 로드될지 알 수 없다.

```c
puts("Hello");
```

를 호출할 때 대략 다음 흐름을 거친다.

```text
puts@PLT
   ↓
puts@GOT 확인
   ↓
libc의 실제 puts 주소
   ↓
실제 puts 실행
```

GOT를 표로 표현하면:

| GOT 항목 | 저장된 값 |
|---|---|
| `puts@GOT` | libc의 실제 `puts()` 주소 |
| `printf@GOT` | libc의 실제 `printf()` 주소 |
| `read@GOT` | libc의 실제 `read()` 주소 |

예:

```text
puts@GOT 칸 자체의 주소 = 0x404018
그 칸에 저장된 값       = 0x7ffff7c80980
```

- `0x404018`: GOT 항목 자체의 주소
- `0x7ffff7c80980`: GOT 항목에 들어 있는 실제 `puts()` 주소

### GOT 읽기: GOT Leak

`puts@GOT`에 저장된 값을 출력하면 libc의 실제 함수 주소를 알아낼 수 있다.

ROP로 개념상 다음을 호출한다.

```c
puts(puts@GOT);
```

ROP chain:

```text
pop rdi; ret
puts@GOT
puts@PLT
main
```

유출된 주소로 libc base를 계산한다.

```python
libc_base = leaked_puts - libc.sym["puts"]
```

이것을 GOT leak이라고 한다.

### GOT 쓰기: GOT Overwrite

GOT 쓰기는 GOT에 저장된 함수 주소를 다른 함수 주소로 덮는 것이다.

원래:

```text
puts@GOT → libc의 puts()
```

변조 후:

```text
puts@GOT → libc의 system()
```

이후 프로그램이:

```c
puts("/bin/sh");
```

를 실행하면 GOT가 `system()`을 가리키므로 실제로는 다음처럼 동작하게 된다.

```c
system("/bin/sh");
```

이를 **GOT overwrite**라고 한다.

### RELRO와의 관계

Partial RELRO:

```text
GOT 읽기 → 가능
GOT 쓰기 → 가능한 경우가 많음
```

Full RELRO:

```text
GOT 읽기 → 가능
GOT 쓰기 → 차단
```

Full RELRO에서는 GOT를 덮는 대신 GOT에 저장된 함수 주소를 읽어 libc base를 계산하고 ret2libc를 수행할 수 있다.

---

## 6. syscall 제한이란?

### syscall

프로그램은 파일, 네트워크, 프로세스 같은 중요한 자원을 직접 제어할 수 없다. 운영체제의 Kernel에 작업을 요청해야 한다.

이 요청을 **System Call**, 줄여서 syscall이라고 한다.

대표적인 syscall:

| syscall | 역할 |
|---|---|
| `read` | 파일이나 소켓에서 데이터 읽기 |
| `write` | 파일이나 화면에 데이터 쓰기 |
| `openat` | 파일 열기 |
| `execve` | 새로운 프로그램 실행 |
| `socket` | 네트워크 소켓 생성 |
| `connect` | 네트워크 서버에 연결 |
| `mmap` | 새로운 메모리 영역 생성 |
| `mprotect` | 메모리 영역 권한 변경 |
| `exit` | 프로세스 종료 |

예:

```c
execve("/bin/sh", ...);
```

는 Kernel에 `/bin/sh` 프로그램을 실행해 달라고 요청한다.

### seccomp

프로그램은 seccomp를 사용해 허용할 syscall과 차단할 syscall을 제한할 수 있다.

예:

```text
허용:
read
write
exit

차단:
execve
socket
connect
mprotect
```

이 환경에서는 saved RIP와 ROP chain을 성공적으로 제어하더라도:

```c
system("/bin/sh");
```

이 실패할 수 있다. `system()`이 Shell을 실행하려면 결국 `execve` 계열 syscall이 필요하기 때문이다.

### Shell 대신 파일 읽기

`execve`는 차단됐지만 다음 syscall이 허용됐다고 가정한다.

```text
openat
read
write
```

그러면 Shell을 실행하는 대신 파일을 직접 읽을 수 있다.

```text
openat("flag.txt")
       ↓
read(fd, buffer, size)
       ↓
write(1, buffer, size)
```

이를 흔히 **ORW chain**이라고 한다.

```text
O: Open
R: Read
W: Write
```

### seccomp의 핵심

seccomp는 Buffer Overflow 같은 취약점 자체를 제거하지 않는다.

```text
RIP 제어 성공
ROP 성공
주소 계산 성공
```

까지 가능하더라도 최종적으로 Kernel에 요청할 수 있는 기능을 제한해 피해를 줄인다.

---

## 모든 개념의 연결

### 보호기법이 거의 없는 경우

```text
Buffer Overflow 발견
        ↓
Stack에 Shellcode 입력
        ↓
saved RIP를 Stack 주소로 변경
        ↓
Shellcode 실행
        ↓
execve("/bin/sh") syscall
        ↓
Shell 실행
```

### 보호기법이 적용된 경우

```text
Canary
→ saved RIP까지 덮기 전에 Stack 손상 탐지

NX
→ Stack Shellcode 실행 차단

PIE
→ 프로그램 함수와 gadget 주소 변경

ASLR
→ libc, Stack, Heap 등의 주소 변경

Full RELRO
→ GOT overwrite 차단

seccomp
→ execve 등의 위험한 syscall 차단
```

### 종합 문제의 일반적인 풀이 흐름

```text
Format String 등으로 Canary 유출
        ↓
프로그램 주소 유출 후 PIE base 계산
        ↓
GOT를 읽어 libc 함수 주소 유출
        ↓
libc base와 system 주소 계산
        ↓
Canary를 보존하며 ROP 실행
        ↓
seccomp가 없으면 system("/bin/sh")
seccomp가 있으면 허용된 syscall로 ORW 등 구성
```

---

## 28일차 + 30일차 내용 정리

- **RIP**: CPU가 다음에 실행할 명령어의 주소
- **saved RIP**: 함수가 끝난 뒤 돌아갈 주소를 Stack에 저장한 값
- **Stack Shellcode**: Stack에 저장한 기계어 코드를 직접 실행하는 방식
- **PIE**: 실행할 때마다 프로그램 내부 함수와 gadget 주소가 달라지게 함
- **libc**: `printf`, `read`, `system` 같은 C 함수가 들어 있는 공유 라이브러리
- **GOT**: 동적 라이브러리 함수의 실제 주소를 저장하는 표
- **GOT leak**: GOT 값을 읽어 libc 함수의 실제 주소를 알아내는 것
- **GOT overwrite**: GOT의 함수 주소를 다른 함수 주소로 변조하는 것
- **syscall**: 프로그램이 Kernel에 작업을 요청하는 인터페이스
- **seccomp**: 프로그램이 사용할 수 있는 syscall을 제한하는 보호기법

---

> 이 문서의 익스플로잇 개념은 본인이 소유하거나 명시적으로 허가받은 교육·실습 환경에서만 사용한다.
