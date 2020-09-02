---
title: "[난독화-LLVM]MBA Expression 난독화: BasicBlock에 새 instruction 삽입하기(또는 교체하기)"
excerpt: "난독화-LLVM"
date: 2020-09-02 12:31:00 -0400
categories: obfuscation
---

# MBA Expression 난독화

MBA Experssion은 산술연산을 난독화하는 방법 중 하나로, 논리연산과 산술연산을 섞어서 난독화 하는 방법이다.

`result=x+y`에 MBA 난독화를 적용한다면 `result = ( x | y ) + y - ( ~x & y )`로 표현할 수 있다.
논리연산과 산술연산을 함께 사용하여 수식을 작성하였기 때문에 결과를 유추하기 힘들어보이지만, 결국은 `result=x+y`과 같은 결과를 반환하는 수식이다.
앞서 든 예제 외에도 다양하게 MBA Expression을 작성할 수 있다.


이번 포스트에서는 LLVM IR에 MBA 난독화를 수행하기 위한 instruction을 삽입하는 것을 목표로 한다.


# BasicBlock에 새 instruction 삽입하기(또는 교체하기)

이전 포스트에서는 IR Builder를 이용하여 특정 instruction을 새롭게 만든 instruction으로 교체하였다.
하지만 MBA Expression을 구현하기 위해서는 여러 추가적인 연산을 하는 instruction들을 추가해야한다.
IR Builder를 이용한 방법은 Value 타입의 객체를 이용하는데, Value의 개념이 나에겐 아직 어려워서 다른 방법을 찾아보았다.


[OLLVM의 Substitution Obfuscation 코드](https://github.com/obfuscator-llvm/obfuscator/blob/llvm-4.0/lib/Transforms/Obfuscation/Substitution.cpp)
를 참고하여 해결할 수 있었다. 만약 `a = b + c`를 `a = b - (-c)`로 바꾸고자 한다면, c를 negative 하고 b에서 -c를 빼야하기 때문에
neg연산을 추가하고, add가 아닌 sub 연산을 수행해야 한다. 이걸 pass로 구현한다면 다음과 같이 작성할 수 있다.


```c++
bool runOnBasicBlock(BasicBlock &B){
	for (auto &I : B)
	{
		if(auto *bo = dyn_cast<BinaryOperator>(&I))
		{
			if(!dyn_cast<AddOperator>(&I))
			{
				continue;
			} // 지난 포스트의 내용과 같다. a = b + c instruction을 만났을 때.

			BinaryOperator *op = NULL;
			op = BinaryOperator::CreateNeg(bo->getOperand(1), "", bo); 
			// getOperand(1)는 c를 가리킴. bo instruction의 앞에 c를 neg하는 instruction을 생성한다. 그리고 op는 이 연산을 위해 사용된 새로운 레지스터를 갖는다.(나의 추측임)
			op = BinaryOperator::Create(Instruction::Sub, bo->getOperand(0), op, "", bo); 
			// getOperand(0)는 b를 가리킴. b에서 op를 빼는 instruction을 생성하여 bo 앞에 삽입한다. 

			bo->replaceAllUsesWith(op); 
			// 기존의 a = b + c instruction을 앞서 생성한 op로 교체한다. bo의 원래 instruction인 %add = add %0 %1은 여전히 존재한다. 하지만 다른 instruction에서 쓰이는 %add를 op가 반환하는 레지스터로 교체해주는 방식인 것 같다. 
		}
	}
}
```

위의 pass를 이용하여 컴파일한 결과를 원본 IR과 비교하면 :

*origin*

```
  %retval = alloca i32, align 4
  %a = alloca i32, align 4
  store i32 0, i32* %retval, align 4
  store i32 34, i32* %a, align 4
  %0 = load i32, i32* %a, align 4
  %add = add nsw i32 %0, 56
  store i32 %add, i32* %a, align 4
  %1 = load i32, i32* %a, align 4
  %call = call i32 (i8*, ...) @printf(i8* getelementptr inbounds ([4 x i8], [4 x i8]* @.str, i32 0, i32 0), i32 %1)
  ret i32 0
```


*result*

```
  %retval = alloca i32, align 4
  %a = alloca i32, align 4
  store i32 0, i32* %retval, align 4
  store i32 34, i32* %a, align 4
  %0 = load i32, i32* %a, align 4
  %1 = sub i32 0, 56 		//op = BinaryOperator::CreateNeg(bo->getOperand(1), "", bo)
  %2 = sub i32 %0, %1		//op = BinaryOperator::Create(Instruction::Sub, bo->getOperand(0), op, "", bo)
  %add = add nsw i32 %0, 56	//기존의 bo의 instruction
  store i32 %2, i32* %a, align 4 //bo의 레지스터인 add가 아니라 op의 레지스터인 %2에 있는 값을 %a로 저장한다.
  %3 = load i32, i32* %a, align 4
  %call = call i32 (i8*, ...) @printf(i8* getelementptr inbounds ([4 x i8], [4 x i8]* @.str, i32 0, i32 0), i32 %3)
  ret i32 0
```

새롭게 연산이 추가되었고 기존의 add연산은 더이상 사용되지 않는 모습을 볼 수 있다.


# MBA Experssion 난독화하는 LLVM Pass

 앞서 작성한 내용을 바탕으로하여 MBA Experssion 난독화를 적용하는 Pass를 작성해보았다.

MBA Experssion :

`result=x+y` => `result = ( x | y ) + y - ( ~x & y )`
 
 
 ```c++
 bool runOnBasicBlock(BasicBlock &B){
	for (auto &I : B)
	{
		if(auto *bo = dyn_cast<BinaryOperator>(&I))
		{
			if(!dyn_cast<AddOperator>(&I))
			{
				continue;
			}

			Value *lhs = bo->getOperand(0);
			Value *rhs = bo->getOperand(1);

			BinaryOperator *op, *op2, *op3 = NULL;
			BinaryOperator *nextInst = bo;

			op = BinaryOperator::Create(Instruction::Or, lhs, rhs, "", nextInst);

			op = BinaryOperator::Create(Instruction::Add, op, rhs, "", nextInst);

			op2 = BinaryOperator::CreateNot(lhs, "", nextInst);
			op2 = BinaryOperator::Create(Instruction::And, op2, rhs, "", nextInst);

			op = BinaryOperator::Create(Instruction::Sub, op, op2, "", nextInst);

			nextInst->replaceAllUsesWith(op);

		}
	}
}
 ```
 
 (삽질했던 이야기) 위의 코드를 구현하는 중에, pass를 적용한 IR을 확인하였는데 bo의 instruction이 남아있길래 replaceAllUsesWith가 적용이 안된줄 알았다..
 그래서 bo의 instruction을 지우려고 엄청 고생했는데, 결국은 내가 IR을 잘못 본거였다..^^ 다음부턴 정신차리자^^
 
 삽질을 통해 알게된 건 
 
 1. bo->eraseFromParent()를 하면 bo를 지울 수 있다. (위의 코드에 적용하면 segmentation falut 발생한다^^)
 
 2. bo->getNextNode()는 다음 instruction을 반환한다.
 
 (삽질한 이야기 끝)
 
 
 IR에 적용한 결과와 원본을 비교하면:
 
 *origin*

```
  %retval = alloca i32, align 4
  %a = alloca i32, align 4
  store i32 0, i32* %retval, align 4
  store i32 34, i32* %a, align 4
  %0 = load i32, i32* %a, align 4
  %add = add nsw i32 %0, 56
  store i32 %add, i32* %a, align 4
  %1 = load i32, i32* %a, align 4
  %call = call i32 (i8*, ...) @printf(i8* getelementptr inbounds ([4 x i8], [4 x i8]* @.str, i32 0, i32 0), i32 %1)
  ret i32 0
```


*result*

```
  %retval = alloca i32, align 4
  %a = alloca i32, align 4
  store i32 0, i32* %retval, align 4
  store i32 34, i32* %a, align 4
  %0 = load i32, i32* %a, align 4
  %1 = or i32 %0, 56
  %2 = add i32 %1, 56
  %3 = xor i32 %0, -1
  %4 = and i32 %3, 56
  %5 = sub i32 %2, %4
  %add = add nsw i32 %0, 56
  store i32 %5, i32* %a, align 4
  %6 = load i32, i32* %a, align 4
  %call = call i32 (i8*, ...) @printf(i8* getelementptr inbounds ([4 x i8], [4 x i8]* @.str, i32 0, i32 0), i32 %6)
  ret i32 0
```
 
난독화하려는 의도에 맞게 여러 instruction이 추가되었고 기존의 instruction대신에 새롭게 작성한 instruction이 사용되는 것을 볼 수 있다.


위의 pass를 이용해서 아래의 코드를 컴파일하여 실행해보면:

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
 
![image](https://user-images.githubusercontent.com/33623107/92021727-28e18600-ed95-11ea-8647-b872fbbed96f.png)

여러 instruction이 추가되었지만 결국은 단순히 더하기 연산을 한 결과가 출력되는 모습을 확인할 수 있다.
