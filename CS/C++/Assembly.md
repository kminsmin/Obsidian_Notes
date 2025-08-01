### SASM
- 어셈블리어 IDE
- 어셈블리어는 CPU, 레지스터, 메모리(RAM)가 주인공

### 데이터 기초

#### 단위
- 8 bit = 1 byte
- 16 bit = 2 byte = 1 word
- 32 bit = 4 byte = 2 word = dword (double-word)
- 64 bit = 8 byte = 4 word = qword (quad-word)

#### 진수
- 10 진수 : 0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, ....
- 2 진수 : 0b00, 0b01, 0b10, 0b11, 0b100, .....(0, 1, 2, 3, 4, ...)
- 16 진수 : 0, 1, 2, 3, 4, 5, 6, 7, 8, 9, A, B, C, D, E, F
	- 0x00,0x01, ....0x0F, 0x10, 0x11, ....


### 레지스터 기초


#### 레지스터의 구조
![[Pasted image 20250701231233.png]]

#### mov
- mov 레지스터, 상수값 : 상수값을 해당 레지스터에 저장
- mov 레지스터, 레지스터 : 오른쪽 레지스터의 값을 왼쪽 레지스터로 복사
### 변수와 레지스터
	실행파일 구조(데이터, BSS), 각 영역에서의 변수 선언, 변수의 mov를 통한 조작
#### 실행파일 구조
![[Pasted image 20250702121829.png]]
  
####  변수 선언
```Assembly

	;초기화 된 데이터
	;[변수이름] [크기] [초기값]
	;[크기] db(1) dw(2) dd(4) dq(8)
section .data

	a db 0x11
	b dw 0x2222
	c dd 0x33333333
	d dq 0x4444444444444444

	;초기화 되지 않은 데이터
	;[변수이름] [크기] [개수]
	;[크기] resb(1) resw(2) resd(4) resq(8)
section .bss
	e resb 10
```

####  메모리 <-> 레지스터
```Assembly
	; 변수의 주소를 복사
	mov rax, a

	;변수의 값을 복사. 단, a의 크기 만큼!!
	mov rax, [a]

	;그래서 딱 원하는 크기 만큼 복사를 원한다면...
	mov al, [a]

	;a에 무언가를 저장하고 싶을 때는...크기를 명시해주어야 한다!!
	mov [a], byte 0x55
	mov [a], word 0x6666
	mov [a], cl ;cl은 애초에 1바이트 크기이므로 따로 크기를 지정해줄 필요가 없다

```


### 문자와 엔디안
	 문자열이 저장되고 해석되는 방식, 리틀 엔디안& 빅 엔디안

#### 문자
- 문자가 저장되는 방식은 위에서 다룬 방식과 다르지 않다.
```Assembly
section .data
	msg db 0x48
```

이를 ==아스키코드==로 인식하여 해석하면 문자가 되는 건데, 이걸 출력하는 건 운영체제에서 해야 한다고 한다. 다행히 SASM은 문자열을 출력하는 기능을 제공한다. (PRINT_STRING)

```Assembly
%include "io64.inc"

section .text
global main
main:
	mov rbp, rsp; for correct debugging
	;write your code here

	PRINT_STRING msg

	xor rax, rax
	ret

section .data
	msg db 'Hello World', 0x00
```

변수에 "Hello World"라고 문자열 형태로 저장해도 잘 저장된다. 다만, 끝에 0x00을 반드시 넣어주어야 한다. 아스키 코드에서 0x00은 Null을 의미하는데 이는 문자열의 끝을 의미하기 때문이다.

msg에 저장된 값을 확인해보면,
![[Pasted image 20250702134227.png]]
이런 식으로 Hello World의 각 문자가 16진수값으로 저장된 것을 볼 수 있다. 이 16진수 값들을 msg 변수에 넣어서 출력을 시도해도 똑같이 Hello World가  출력된다.

#### 엔디언
아래처럼 숫자를 12345678 순서대로 저장해보자. 그런데 메모리에 저장된 방식이 이상하다.
![[Pasted image 20250702135059.png]]
![[Pasted image 20250702135127.png]]
이는 내 컴퓨터에서 메모리에 값이 저장되는 방식이 '리틀 엔디안' 방식이기 때문이다.
==**엔디언**==이란, 컴퓨터의 메모리와 같은 1차원의 공간에 여러 개의 연속된 대상을 배열하는 방식이다. 
![[Pasted image 20250702161538.png]]

- **빅 엔디언** : 값을 순서대로 저장
	- 숫자 대소 비교에 유리
- 리틀 엔디언 : 1 바이트 씩 끊어서  뒤집어서 저장됨(위 사진 참고)
	- ==**캐스팅**==에 유리
	- ex : 최 하단 1바이트만을 살리고 나머지는 날리는 경우 

### 사칙연산

```Assembly
%include "io64.inc"

section .text
global main
main:

	mov rbp, rsp; for correct debugging

	;write your code here

	GET_DEC 1, al
	GET_DEC 1, num

	;더하기 연산
	;add a, b (a = a + b)	
	;a는 레지스터 or 메모리	
	;b 는 레지스터 or 메모리 or 상수	
	; 단, a,b 모두 메모리는 X
	
	;빼기 연산	
	;sub a, b (a = a - b)
	
	add al, 1 ; 레지스터 + 상수	
	PRINT_DEC 1, al ; 1+1=2	
	NEWLINE	
	
	add al, [num] ; 레지스터 + 메모리	
	PRINT_DEC 1, al	
	NEWLINE
	
	mov bl, 3	
	add al, bl ; 레지스터 + 레지스터	
	PRINT_DEC 1, al	
	NEWLINE
	
	add [num], byte 1 ; 메모리 + 상수	
	PRINT_DEC 1, [num]	
	NEWLINE
	
	add [num], al ; 메모리 + 레지스터	
	PRINT_DEC 1, [num]	
	NEWLINE
	
	; 곱하기 연산	
	; mul reg	
	; - mul bl => al * bl	
	; -- 연산 결과를 ax에 저장	
	; - mul bx => ax * bx	
	; -- 연산 결과는 dx(상위 16비트) ax(하위 16비트)에 저장	
	; - mul ebx => eax * ebx
	
	;ex) 5 * 8 은?	
	mov ax, 0	
	mov al, 5	
	mov bl, 8	
	mul bl	
	PRINT_DEC 2, ax	
	NEWLINE
	
	; 나누기 연산	
	; div reg	
	; - div bl => ax / bl	
	; -- 연산 결과는 al(몫) ah(나머)
	
	;ex) 100 / 3 은?	
	mov ax, 100	
	mov bl, 3	
	div bl	
	PRINT_DEC 2, al	
	NEWLINE	
	mov al, ah	
	PRINT_DEC 1, al	
	NEWLINE
	
	xor rax, rax
	
	ret

section .bss

num resb 1
```

### 쉬프트 연산과 논리 연산

#### 쉬프트 연산
-  `shl reg, bit_count` : 왼쪽으로 bit_count 만큼 해당 레지스터의 비트를 민다.
-  `shr reg, bit_count` : 오른쪽으로 bit_count 만큼 해당 레지스터의 비트를 민다.
##### 예제
![[Pasted image 20250703001744.png]]
![[Pasted image 20250703001803.png]]
레지스터 할당 크기를 넘어서서 비트를 밀어버리게 되면 밀려난 부분은 소실된다.

#### 논리 연산
- not
- and
- or
- xor : 둘 다 1이거나 둘 다 0이면 0, 아니면 1
	-  ==동일한 값으로 xor 연산을 두 번 하면 자기 자신으로 되돌아옴==
	- 암호학에서 유용하다! (value xor key)

### 분기문
	특정 조건에 따라서 코드 흐름을 제어하는 것(if)
- ==CMP== dst, src (dst가 기준)
- 비교를 한 결과물은 ==Flag Register==  저장
- ==JMP== {label} 시리즈
	- JMP : 무조건 jump
	- JE : JumpEquals. 같으면 jump
	- JNE : JumpNotEquals. 다르면 jump
	- JG : JumpGreater. 크면 jump
	- JGE : JumpGreaterEquals. 크거나 같으면 jump
	- JL
	- JLE

#### 연습문제
	두 숫자가 같으면 1, 아니면 0을 출력하는 프로그램
```Assembly
%include "io64.inc"

section .text
global main
main:
	mov rbp, rsp; for correct debugging  

	; 분기문(if)
	; 연습문제 : 어떤 숫자(1~100)가 짝수면 1, 홀수면 0을 출력하는 프로그램
	mov ax, 98
	mov bl, 2
	div bl
	cmp ah, 0
	je LABEL_EQUAL
	mov rcx, 0
	jmp LABEL_EQUAL_END

LABEL_EQUAL:
	mov rcx,1
LABEL_EQUAL_END:
	PRINT_HEX 1, rcx
	NEWLINE
	xor rax, rax

ret
```

### 반복문
	 특정 조건을 만족할 때까지 반복해서 실행.
	 while, for do-while
#### 연습문제
두 가지 풀이법 
- 직접 카운팅
- loop 활용
![[Pasted image 20250703144713.png]]

### 배열과 주소

#### 배열
	동일한 타입의 데이터 묶음
- 배열을 구성하는 각 값을 배열 요소(element)라고 함
- 배열의 위치를 가리키는 숫자를 인덱스(index)라고 함

지난 시간에 메모리에 저장한 변수를 그대로 mov 명령어를 통해 레지스터로 받아오게 되면, 변수에 저장된 값이 아니라 주소값이 복사된다는  것을 배웠다.
![[Pasted image 20250703150453.png]]
![[Pasted image 20250703150550.png]]
이렇게 mov rax, a를 실행하게 되면 rax 에 a의 주소가 저장된다. 이 주소를 복사해서 메모리 창에서 확인해보면...
![[Pasted image 20250703150639.png]]
이렇게 a에 접근할 수 있다. 이 주소를 활용해서, a에 저장된 값을 출력해보자. a의 맨 앞에 있는 1 하나만 출력된다. 그렇다면 나머지 값들은 어떻게 뽑아낼 수 있을까?
![[Pasted image 20250703150822.png]]
![[Pasted image 20250703150905.png]]

#### 주소
- 시작주소 + 인덱스X크기
주소값에 1씩 더해서 출력해보자. 놀랍게도 인덱스로 접근한 것처럼  a의 나머지 값들을 가져올 수 있다.
![[Pasted image 20250703151039.png]]
![[Pasted image 20250703151048.png]]
이렇게 하드코딩 하지 말고 저번 시간에 배운 반복문을 통해 우아하게 a의 모든 값을 출력해보자.
![[Pasted image 20250703155158.png]]
b의 경우 데이터 하나 당 크기가 2바이트이기 때문에, 인덱스에 2를 곱해줬다.
결과 :
![[Pasted image 20250703155241.png]]
### 함수
	프로시저(procedure), 서브루틴(subroutine)
-  저번에 배운 점프 시리즈를 활용하면 함수 비슷한 무언가를 만들 수는 있다.
-  하지만 레지스터를 활용해서 인자를 받고 결과를 저장할 경우, 레지스터 수가 부족할 수 있다.
-  그렇다고 .data나 .bss 영역을 사용하자니 함수를 위해 메모리까지 접근하는 것은 비효율적
-  **다른 메모리 구조가 필요하다** => ==스택 메모리==
![[Pasted image 20250703165427.png]]

### 스택 메모리
	스택 메모리, 스택 프레임
- 함수가 사용하는 일종의 메모장
	-  매개 변수 전달
	- 돌아갈 주소 관리

#### 레지스터의 다양한 용도
- a b c d .. 범용 레지스터
- 포인터 레지스터 (포인터 = 위치를 가리키는~)
	- ip (Instruction Pointer) : 다음 수행 명령어의 위치
	- sp (Stack Pointer) : 현재 스택 top 위치 (일종의 cursor)
	- bp (Base Pointer) : 스택 상대주소 계산용
![[Pasted image 20250703175749.png]]