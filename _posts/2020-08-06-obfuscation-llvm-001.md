---
title: "[난독화-LLVM]LLVM Pass 이용하여 IR코드 출력"
excerpt: "난독화-LLVM"
date: 2020-08-06 00:31:00 -0400
categories: obfuscation
---

dump()라는 이름의 메소드를 이용하여 IR을 확인할 수 있다. 


```

In a function called main!
Function body: 

; Function Attrs: noinline nounwind uwtable
define i32 @main() #0 {
entry:
  %retval = alloca i32, align 4
  %a = alloca i32, align 4
  %count = alloca i32, align 4
  store i32 0, i32* %retval, align 4
  store i32 3, i32* %a, align 4
  store i32 0, i32* %count, align 4
  br label %while.cond

while.cond:                                       ; preds = %if.end, %entry
  %0 = load i32, i32* %count, align 4
  %cmp = icmp slt i32 %0, 2
  br i1 %cmp, label %while.body, label %while.end

while.body:                                       ; preds = %while.cond
  %1 = load i32, i32* %a, align 4
  %cmp1 = icmp sgt i32 %1, 0
  br i1 %cmp1, label %if.then, label %if.else

if.then:                                          ; preds = %while.body
  %call = call i32 (i8*, ...) @printf(i8* getelementptr inbounds ([7 x i8], [7 x i8]* @.str, i32 0, i32 0))
  br label %if.end

if.else:                                          ; preds = %while.body
  %call2 = call i32 (i8*, ...) @printf(i8* getelementptr inbounds ([5 x i8], [5 x i8]* @.str.1, i32 0, i32 0))
  br label %if.end

if.end:                                           ; preds = %if.else, %if.then
  %2 = load i32, i32* %count, align 4
  %inc = add nsw i32 %2, 1
  store i32 %inc, i32* %count, align 4
  br label %while.cond

while.end:                                        ; preds = %while.cond
  ret i32 0
}
Basic block:

entry:
  %retval = alloca i32, align 4
  %a = alloca i32, align 4
  %count = alloca i32, align 4
  store i32 0, i32* %retval, align 4
  store i32 3, i32* %a, align 4
  store i32 0, i32* %count, align 4
  br label %while.cond
Instruction: 
  %retval = alloca i32, align 4
Instruction: 
  %a = alloca i32, align 4
Instruction: 
  %count = alloca i32, align 4
Instruction: 
  store i32 0, i32* %retval, align 4
Instruction: 
  store i32 3, i32* %a, align 4
Instruction: 
  store i32 0, i32* %count, align 4
Instruction: 
  br label %while.cond
Basic block:

while.cond:                                       ; preds = %if.end, %entry
  %0 = load i32, i32* %count, align 4
  %cmp = icmp slt i32 %0, 2
  br i1 %cmp, label %while.body, label %while.end
Instruction: 
  %0 = load i32, i32* %count, align 4
Instruction: 
  %cmp = icmp slt i32 %0, 2
Instruction: 
  br i1 %cmp, label %while.body, label %while.end
Basic block:

while.body:                                       ; preds = %while.cond
  %1 = load i32, i32* %a, align 4
  %cmp1 = icmp sgt i32 %1, 0
  br i1 %cmp1, label %if.then, label %if.else
Instruction: 
  %1 = load i32, i32* %a, align 4
Instruction: 
  %cmp1 = icmp sgt i32 %1, 0
Instruction: 
  br i1 %cmp1, label %if.then, label %if.else
Basic block:

if.then:                                          ; preds = %while.body
  %call = call i32 (i8*, ...) @printf(i8* getelementptr inbounds ([7 x i8], [7 x i8]* @.str, i32 0, i32 0))
  br label %if.end
Instruction: 
  %call = call i32 (i8*, ...) @printf(i8* getelementptr inbounds ([7 x i8], [7 x i8]* @.str, i32 0, i32 0))
Instruction: 
  br label %if.end
Basic block:

if.else:                                          ; preds = %while.body
  %call2 = call i32 (i8*, ...) @printf(i8* getelementptr inbounds ([5 x i8], [5 x i8]* @.str.1, i32 0, i32 0))
  br label %if.end
Instruction: 
  %call2 = call i32 (i8*, ...) @printf(i8* getelementptr inbounds ([5 x i8], [5 x i8]* @.str.1, i32 0, i32 0))
Instruction: 
  br label %if.end
Basic block:

if.end:                                           ; preds = %if.else, %if.then
  %2 = load i32, i32* %count, align 4
  %inc = add nsw i32 %2, 1
  store i32 %inc, i32* %count, align 4
  br label %while.cond
Instruction: 
  %2 = load i32, i32* %count, align 4
Instruction: 
  %inc = add nsw i32 %2, 1
Instruction: 
  store i32 %inc, i32* %count, align 4
Instruction: 
  br label %while.cond
Basic block:

while.end:                                        ; preds = %while.cond
  ret i32 0
Instruction: 
  ret i32 0


```