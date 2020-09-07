---
title: "[난독화-LLVM]MBA Expression 난독화: BasicBlock에 새 instruction 삽입하기(또는 교체하기)(2)"
excerpt: "난독화-LLVM"
date: 2020-09-07 12:31:00 -0400
categories: obfuscation
---

# 지난 post & 이번 post의 목표

지난 post에서 mba expression 난독화하는 pass를 제작하였다. mba 난독화는 add연산 뿐만 아니라 and, xor 등 다른 연산에도 적용할 수 있다.
지난 post에서는 add연산만 난독화하였으므로, 이번 포스트에서는 and, xor, or연산에 대하여 mba expression 난독화하도록 구현하겠다.


# MBA 추가 구현

## 각 연산별로 난독화하는 메소드를 나누기

지난 포스트에서의 runOnBasicBlock의 코드는 add연산인 경우에만 난독화 instruction을 추가하도록 했었다.
확장성을 높이기 위해서 각 opcode별로 호출할 메소드를 다르게 구현하였다. instruction의 종류를 검사하는 코드는 
[OLLVM의 Substitution Obfuscation 코드](https://github.com/obfuscator-llvm/obfuscator/blob/llvm-4.0/lib/Transforms/Obfuscation/Substitution.cpp)
를 참고하였다. 구조를 변경한 코드는 다음과 같다.

```c++
bool runOnBasicBlock(BasicBlock &B){
	for (auto &I : B)
	{
		if(I.isBinaryOp())
		{
			switch(I.getOpcode())
			{
			case BinaryOperator::Add:
				mba_Add(dyn_cast<BinaryOperator>(&I));
				break;
			case Instruction::And:
				mba_And(dyn_cast<BinaryOperator>(&I));
				break;
			case Instruction::Xor:
				mba_Xor(dyn_cast<BinaryOperator>(&I));
				break;
			case Instruction::Or:
				mba_Or(dyn_cast<BinaryOperator>(&I));
				break;
			default:
				break;
			}

		}
	}

}
```

각 연산 별로 난독화를 위한 instruction을 추가하기 위한 메소드를 정의하였고, switch문으로 case를 분류하여 실행하도록 하였다.


## 각 연산의 난독화 method

```
Add:
( x | y ) + y - ( ~x & y )

And:
(x | y) - (~x)

Xor:
(x | y) - y + (~x & y)

Or:
(x ^ y) + y - (~x & y)
```

각 연산 별로 위와 같이 mba 난독화하기로 하였다. 이 방법은 이전에 소스코드 레벨에서 난독화 하였을 때와 같다.
llvm의 ir에 위의 난독화를 적용하기 위한 method는 다음과 같다.

```c++
void TestModule::mba_Add(BinaryOperator *bo)
{
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

void TestModule::mba_And(BinaryOperator *bo)
{
        Value *lhs = bo->getOperand(0);
        Value *rhs = bo->getOperand(1);

        BinaryOperator *op, *op2 = NULL;

        op = BinaryOperator::CreateNot(lhs, "", bo);
        op = BinaryOperator::Create(Instruction::Or, op, rhs, "", bo);
        op2 = BinaryOperator::CreateNot(lhs, "",bo);
        op = BinaryOperator::Create(Instruction::Sub, op, op2, "", bo);

        bo->replaceAllUsesWith(op);
}

void TestModule::mba_Xor(BinaryOperator *bo)
{
        Value *lhs = bo->getOperand(0);
        Value *rhs = bo->getOperand(1);

        BinaryOperator *op, *op2 = NULL;

        op = BinaryOperator::Create(Instruction::Or, lhs, rhs, "", bo);
        op2 = BinaryOperator::CreateNot(lhs, "", bo);
        op2 = BinaryOperator::Create(Instruction::And, op2, lhs, "", bo);

        op = BinaryOperator::Create(Instruction::Sub, op, rhs, "", bo);
        op = BinaryOperator::Create(Instruction::Add, op, op2, "", bo);

        bo->replaceAllUsesWith(op);
}

void TestModule::mba_Or(BinaryOperator *bo)
{
        Value *lhs = bo->getOperand(0);
        Value *rhs = bo->getOperand(1);

        BinaryOperator *op, *op2 = NULL;

        op = BinaryOperator::Create(Instruction::Xor, lhs, rhs, "", bo);
        op2 = BinaryOperator::CreateNot(lhs, "", bo);
        op2 = BinaryOperator::Create(Instruction::And, op2, rhs, "", bo);
        op = BinaryOperator::Create(Instruction::Add, op, rhs, "", bo);
        op = BinaryOperator::Create(Instruction::Sub, op, op2, "", bo);

        bo->replaceAllUsesWith(op);
}
```

구현에 대한 자세한 설명은 이전 포스트에 작성하였으니 이번 포스트에서는 생략하겠다.

# Test

```c++
#include <stdio.h>
  
int main()
{
        int a = 34;
        a = a+56;
        printf("%d\n",a+2);
        printf("%d\n",a&2);
        printf("%d\n",a^2);
        printf("%d\n",a|2);


        return 0;
}
```

위의 c 소스코드에 이번에 만든 난독화 pass를 적용해보았다.

![image](https://user-images.githubusercontent.com/33623107/92357475-dd2b3580-f122-11ea-8a38-e3df4599a518.png)
<난독화 전>

![image](https://user-images.githubusercontent.com/33623107/92357300-950c1300-f122-11ea-8203-fa7650c1483b.png)
<난독화 후>

난독화하여 생성한 실행파일과 원본이 같은 결과를 내는 것을 볼 수있다. 그러면 IR에 난독화가 잘 적용되었는지 확인해보자.