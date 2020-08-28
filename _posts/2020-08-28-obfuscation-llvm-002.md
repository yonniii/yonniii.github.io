---
title: "[난독화-LLVM]binary 연산자를 다른 binary 연산자로 바꾸기"
excerpt: "난독화-LLVM"
date: 2020-08-28 12:31:00 -0400
categories: obfuscation
---

LLVM IR코드에서 binary 연산자를 다른 binary 연산자로 바꾸는 예제를 수행해보았다.

# 1. binary연산을 변경

basicBlock에서 instruction을 순회하면서 binary operator가 있는 경우에 변경한 instuction으로 교체하면 된다.
연산자를 바꾸기 위한 코드는 다음과 같다. :

```c++
bool runOnBasicBlock(BasicBlock &B){
	for (auto &I : B) // basicBlock 내의 instruction을 iterate
	{
		if(auto *op = dyn_cast<BinaryOperator>(&I)) // op가 binary operator인 경우
		{
			IRBuilder<> builder(op); // 새로 instruction을 만들기 위한 builder 객체인듯?

			Value *lhs = op->getOperand(0); // instruction에서 첫번째 operand
			Value *rhs = op->getOperand(1); // instruction에서 두번째 operand
			Value *mul = builder.CreateMul(lhs,rhs); // 기존의 operator와 다르게 multiply 연산을 하는 instruction 생성
			
			for(auto &U : op->uses()) // Value의 Use를 iterate
			{
				User *user = U.getUser(); 
				user->setOperand(U.getOperandNo(), mul); // 새롭게 만든 instruction으로 변경
			}
		}
	}
}

```

## 결과

원본:

```c++
#include <stdio.h>
  
int main()
{
        int a = 34;
        a = a+56;
        printf("%d\n",a);

        return 0;
}

```

pass를 적용하여 컴파일?빌드?한 결과:

![image](https://user-images.githubusercontent.com/33623107/91538415-c7fc1d00-e952-11ea-80e3-39dc436c109a.png)


원본의 코드라면 `90`을 출력하겠지만, 위의 결과에서는 예제에서 작성한 pass를 적용하였기 때문에 add연산이 multiply연산으로 바뀌었다. 그렇기 때문에 34X56 인 1904가 출력된 것이다.

```
clang -Xclang -load -Xclang passModule.so test.c
```

위의 코드는 특정 pass를 적용하여 빌드하는 코드이다.



# 2. 특정 binary operator만 적용하기

위의 1번 예제는 모든 binary operator에 대하여 연산을 교체한다. 이번에는 특정 binary operator에만 적용하도록 코드를 몇 줄 추가하겠다.

```c++
bool runOnBasicBlock(BasicBlock &B){
	for (auto &I : B) // basicBlock 내의 instruction을 iterate
	{
		if(auto *op = dyn_cast<BinaryOperator>(&I)) // op가 binary operator인 경우
		{
		
			if(!dyn_cast<AddOperator>(&I)) // AddOperator로 형변환을 시도하여 true인 경우에만 다음 연산을 수행하도록 한다.
			{
				continue;
			}
			
			IRBuilder<> builder(op); // 새로 instruction을 만들기 위한 builder 객체인듯?

			Value *lhs = op->getOperand(0); // instruction에서 첫번째 operand
			Value *rhs = op->getOperand(1); // instruction에서 두번째 operand
			Value *mul = builder.CreateMul(lhs,rhs); // 기존의 operator와 다르게 multiply 연산을 하는 instruction 생성
			
			for(auto &U : op->uses()) // Value의 Use를 iterate
			{
				User *user = U.getUser(); 
				user->setOperand(U.getOperandNo(), mul); // 새롭게 만든 instruction으로 변경
			}
		}
	}
}

```

위와 같이 AddOperator로 형변환 되는지 확인하는 코드를 삽입하여 add연산인 경우에만 operator를 교체하도록 하였다.

dyn_cast는 llvm에서 형변환하는 메소드이다. [dyn_cast의 document](https://llvm.org/doxygen/Casting_8h_source.html#l00323) 를 보면, 파라미터를 특정 타입으로 형변환하여 리턴한다고 한다.
형변환하려는 타입이 잘못되었다면 nullpointer를 반환한다. c++ 11 표준에서 nullpointer는 false로 변환되기 때문에 if문의 조건으로 dyn_cast메소드를 작성하면 타입을 검사할 수 있다. 


## 결과

원본:

```c++
#include <stdio.h>
  
int main()
{
        int a = 34;
        //a = a+56;
        printf("%d\n",a+2);
        printf("%d\n",a-2);

        return 0;
}
```

위와 같은 원본 코드에 2번 예제에서 생성한 pass를 적용한 결과는 다음과 같다.

![image](https://user-images.githubusercontent.com/33623107/91541385-3cd15600-e957-11ea-9b4b-b8da89aeb520.png)

add연산은 multiply연산으로 바뀌었기 때문에 34의 2배를 출력하였고, minus연산은 multiply연산으로 바뀌지 않았기 때문에 34-2인 32를 출력하였다.



