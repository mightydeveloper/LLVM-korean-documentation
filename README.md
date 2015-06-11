# LLVM-korean-documentation

This repository contains LLVM Korean Documentation.

## 목적 
- LLVM 프로젝트 문서의 한글화
- LLVM 프로젝트 국내 커뮤니티 활성화
- LLVM 프로젝트 기여

를 위해 작성하였습니다. 

## 참고 자료
- [LLVM Language Reference Manual] 


### 개요
이 문서는 LLVM 어셈블리 언어에 대한 참고 메뉴얼입니다. LLVM은 정적 단일 할당 (Static Single Assignment) 에 기반한 표현 방식을 따르고, 타입 세이프티, 로우레벨 연산, 유연성, 그리고 '모든' 하이레벨 언어들을 깔끔하게 표현할 수 있는 기능을 지원합니다. 이 LLVM 어셈블리 언어는 모든 LLVM 컴파일 단계에서 공통적으로 쓰이는 코드 표현 방식입니다.

### 서론
LLVM 코드 표현방식은 크게 다음과 같이 쓰이기 위해 디자인되었습니다. 
- 인 메모리 컴파일러 IR (in-memory compiler IR)
- 온 디스크 비트코드 표현 (on-disk bitcode representation) - JIT 컴파일러를 사용하여 빠르게 로딩하기에 적합함.
- 사람이 읽기 쉬운 어셈블리 언어의 표현

LLVM IR은 효율적이게 컴파일러 변형과 분석을 돕는 동시에, 편하게 디버깅하고 컴파일러 변형을 시각적으로 볼 수 있게 해줍니다.
위의 세가지 표현 방식은 모두 동등한(equivalent) 표현 방식입니다. 이 문서에서는 사람이 읽기 쉬운 표현방식과 표기법에 대해 설명합니다.

LLVM 언어는 가볍고, 로우레벨이면서 동시에 표현력이 좋고, 타입이 있으며, 확장성이 있는것을 목표로 설계되었습니다. 로우레벨 이면서 동시에 하이레벨 아이디어가 깔끔하게 맵핑되기 쉽도록 일종의 "보편적인 IR"이 되는 것을 목표로 하였습니다. (많은 소스 언어들이 매핑되니까 마이크로프로세서들이 일종의 "보편적인 IR"인것처럼). 타입 정보를 제공하면서, LLVM은 최적화에 이용될 수 있습니다. 예를 들어, 포인터 분석을 통해 C의 자동 변수가 접근된적이 없다는것을 증명하여, 메모리에 올리지 않고 SSA로 처리를 할 수 있도록 해줄 수 있습니다.

## Well-Formedness
(중요) 이 문서는 LLVM 어셈블리 언어의 "Well formed"에 대해 기술합니다.
파서가 받아들이는것과 "Well formed"인것과는 확실한 차이가 있습니다. 예를들어 다음 instruction은 신텍스 문제는 없지만, well formed 가 아닙니다. 
```LLVM
%x = add i32 1, %x
```
왜나하면 %x 의 정의가 %x의 모든 사용에 대해 지배하지 않기 때문입니다. LLVM 은 LLVM 모듈이 well formed되어있는지 검증하는 검증 pass를 제공합니다. 이 pass는 파서가 input 어셈블리를 파싱하고 비트코드를 optimizer가 만들기 전에 파서에서 자동으로 돌아갑니다. 이 verifer pass가 발견한 위반사항들은 transformation pass에 버그가 있었거나 파서의 input이 잘못되었음을 뜻합니다.


## Identifiers
LLVM 식별자(identifier)는 두가지 종류가 있습니다 : 글로벌, 로컬 (전역, 지역 으로 번역할 수 있지만 이 문서에서는 글로벌 로컬로 쓰겠습니다-엮은이)

글로벌 식별자 (함수, 전역 변수) 는 '@' 문자로 시작합니다. <br>
로컬 식별자는 (레지스터 이름, 타입) 은 '%' 문자로 시작합니다. <br>
추가적으로, 다음과 같은 3가지 다른 포멧의 식별자가 존재합니다. <br>
1. named values들은 prefix를 사용하여 표현합니다. 예를들면,  %foo, @DivisionByZero, %a.really.long.identifier 와 같이 표현할 수 있습니다. 여기에 사용되는 실제 정규표현식은 ‘[%@][-a-zA-Z$._][-a-zA-Z$._0-9]*‘입니다. 이름을 다른 문자열을 쓰는 경우 quote를 양쪽에 씌워주어야 합니다. Special character를 쓰는 경우, xx에 ASCII code의 16진수 값을 넣어 "\xx" 와 같은 방식으로 씁니다. 이렇게 쓰면 어떠한 문자도 name value로 쓸 수 있습니다. quote 자체도 쓸 수 있습니다. "\01" prefix를 쓰면 글로벌 variable의 mangling을 방지할 수도 있습니다.
2. Unnamed values들은 prefix가 있는 unsigned numeric value로 표현할 수 있습니다. 예를들면, %12, @2, %44 가 있습니다.
3. 상수(Constants)는 constants 섹션에서 따로 설명하겠습니다.

LLVM에서는 value들이 prefix로 시작해야되는 두가지 이유가 있습니다 : 컴파일러가 reserved keyword랑 이름이 곂치는걸 걱정하지 않기 위해 그리고, reserved keyword를 추후 확장하기 편하기 위하여 입니다. 추가적으로, unnamed identifer들은 컴파일러가 임시적인 variable을 symbol table과 곂치지 않게 하기 위하여 입니다.

LLVM에 있는 Reserved word들은 다른 랭귀지와 비숫합니다. 각각 다른 opcodes (‘add‘, ‘bitcast‘, ‘ret‘, 등등...) 이 있고, primitive type name (‘void‘, ‘i32‘, 등등...) 이 있습니다. 이와같은 reserved word는 prefix 문자('%' 나 '@')로 시작할 수 없기 때문에 reserved word는 variable name과 conflict될 수 없습니다.

다음은 integer variable '%X' 에 8을 곱하는 LLVM 코드입니다. <br>
쉬운 방법은 :
```LLVM
%result = mul i32 %X, 8
```
Strength reduction을 하면 : 
```LLVM
%result = shl i32 %X, 3
```
어려운 방법으로는 :
```LLVM
%0 = add i32 %X, %X           ; yields i32:%0
%1 = add i32 %0, %0           ; yields i32:%1
%result = add i32 %1, %1
```
이 마지막 방법에서는 몇가지 중요한 LLVM 의 Lexical feature들이 있습니다.
1. Comment 는 ; 를 통해 라인 끝에 쓰여집니다.
2. named value에 쓰여지지 않는 경우 Unnamed temporary들이 생깁니다.
3. Unnamed temporary들은 순서대로 넘버링 됩니다.(per-function 카운터를 이용하여 0부터 시작합니다.) 여기서 주의할점은 basic block들이랑 unnamed function parameter들도 이 넘버링에 포함이 됩니다. 예를들어, entry basic block이 lobel name이 주어지지 않고 모든 function parameter에는 name이 주어졌으면, 숫자 0이 할당됩니다.

그리고 추가적으로 이 문서에서 쓰이는 컨밴션을 보여줍니다. instruction을 보여줄 때, 코멘트에 타입이랑 생성되는 값의 이름을 보여줍니다.

## High level 구조
### 모듈 구조
LLVM 프로그램은 여러개의 모듈로 구성되어있습니다. 각각은 input program에 대한 translation unit입니다. 각 모듈은 함수들, 글로벌 변수들, 그리고 symbol table entries로 구성되어있습니다. 모듈은 LLVM Linker와 합쳐질 수 있고, 함수 (그리고 글로벌 변수들)의 정의를 합치며, forward declarations에 대해 resolve하고, symbol table entry를 합칩니다. 다음은 "Hello world"에 대한 모듈 예제입니다 :
```LLVM
; Declare the string constant as a global constant.
@.str = private unnamed_addr constant [13 x i8] c"hello world\0A\00"

; External declaration of the puts function
declare i32 @puts(i8* nocapture) nounwind

; Definition of main function
define i32 @main() {   ; i32()*
  ; Convert [13 x i8]* to i8  *...
  %cast210 = getelementptr [13 x i8], [13 x i8]* @.str, i64 0, i64 0

  ; Call puts function to write out the string to stdout.
  call i32 @puts(i8* %cast210)
  ret i32 0
}

; Named metadata
!0 = !{i32 42, null, !"string"}
!foo = !{!0}
```
이 예제는 ".str"라는 global variable, external declaration으로 정의된 "puts" 함수, main 에 대한 함수 정의와 "foo"라는 named metadata로 이루어져있습니다.

일반적으로 모듈은 global value의 리스트로 구성되어있습니다 (여기서 function과 global variable을 global value로 간주합니다).
글로벌 value는 특정 memory location에 대한 포인터로 표현되고 (여기서는 char 배열에 대한 포인터, 함수에 대한 포인터), 다음과 같은 linkage types를 가집니다.




[LLVM Language Reference Manual]:http://llvm.org/docs/LangRef.html
