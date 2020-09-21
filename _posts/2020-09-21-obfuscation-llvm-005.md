---
title: "[난독화-LLVM]C코드에서 가상화난독화 시도"
excerpt: "난독화-LLVM"
date: 2020-09-21 12:31:00 -0400
categories: obfuscation
---

```c
#include <stdio.h>

int binary(char op, int lhs, int rhs){
	switch (op){
		case '+':
			return lhs + rhs;
		case '-':
			return lhs - rhs;
		case '/':
			return lhs / rhs;
		case '*':
			return lhs * rhs;			
	}
	return lhs;
}

int main(){
	
	int a = 5;
	int b = 6;
	int c = binary('*', a, b);
	
	printf("%d\n", binary('+',a,c));
	
	return 0;
}
```

```
; ModuleID = 'viarlkaf.c'
source_filename = "viarlkaf.c"
target datalayout = "e-m:e-i64:64-f80:128-n8:16:32:64-S128"
target triple = "x86_64-unknown-linux-gnu"

@.str = private unnamed_addr constant [4 x i8] c"%d\0A\00", align 1

; Function Attrs: noinline nounwind uwtable
define i32 @binary(i8 signext %op, i32 %lhs, i32 %rhs) #0 {
entry:
  %retval = alloca i32, align 4
  %op.addr = alloca i8, align 1
  %lhs.addr = alloca i32, align 4
  %rhs.addr = alloca i32, align 4
  store i8 %op, i8* %op.addr, align 1
  store i32 %lhs, i32* %lhs.addr, align 4
  store i32 %rhs, i32* %rhs.addr, align 4
  %0 = load i8, i8* %op.addr, align 1
  %conv = sext i8 %0 to i32
  switch i32 %conv, label %sw.epilog [
    i32 43, label %sw.bb
    i32 45, label %sw.bb1
    i32 47, label %sw.bb2
    i32 42, label %sw.bb3
  ]

sw.bb:                                            ; preds = %entry
  %1 = load i32, i32* %lhs.addr, align 4
  %2 = load i32, i32* %rhs.addr, align 4
  %add = add nsw i32 %1, %2
  store i32 %add, i32* %retval, align 4
  br label %return

sw.bb1:                                           ; preds = %entry
  %3 = load i32, i32* %lhs.addr, align 4
  %4 = load i32, i32* %rhs.addr, align 4
  %sub = sub nsw i32 %3, %4
  store i32 %sub, i32* %retval, align 4
  br label %return

sw.bb2:                                           ; preds = %entry
  %5 = load i32, i32* %lhs.addr, align 4
  %6 = load i32, i32* %rhs.addr, align 4
  %div = sdiv i32 %5, %6
  store i32 %div, i32* %retval, align 4
  br label %return

sw.bb3:                                           ; preds = %entry
  %7 = load i32, i32* %lhs.addr, align 4
  %8 = load i32, i32* %rhs.addr, align 4
  %mul = mul nsw i32 %7, %8
  store i32 %mul, i32* %retval, align 4
  br label %return

sw.epilog:                                        ; preds = %entry
  %9 = load i32, i32* %lhs.addr, align 4
  store i32 %9, i32* %retval, align 4
  br label %return

return:                                           ; preds = %sw.epilog, %sw.bb3, %sw.bb2, %sw.bb1, %sw.bb
  %10 = load i32, i32* %retval, align 4
  ret i32 %10
}

; Function Attrs: noinline nounwind uwtable
define i32 @main() #0 {
entry:
  %retval = alloca i32, align 4
  %a = alloca i32, align 4
  %b = alloca i32, align 4
  %c = alloca i32, align 4
  store i32 0, i32* %retval, align 4
  store i32 5, i32* %a, align 4
  store i32 6, i32* %b, align 4
  %0 = load i32, i32* %a, align 4
  %1 = load i32, i32* %b, align 4
  %call = call i32 @binary(i8 signext 42, i32 %0, i32 %1)
  store i32 %call, i32* %c, align 4
  %2 = load i32, i32* %a, align 4
  %3 = load i32, i32* %c, align 4
  %call1 = call i32 @binary(i8 signext 43, i32 %2, i32 %3)
  %call2 = call i32 (i8*, ...) @printf(i8* getelementptr inbounds ([4 x i8], [4 x i8]* @.str, i32 0, i32 0), i32 %call1)
  ret i32 0
}

declare i32 @printf(i8*, ...) #1

attributes #0 = { noinline nounwind uwtable "correctly-rounded-divide-sqrt-fp-math"="false" "disable-tail-calls"="false" "less-precise-fpmad"="false" "no-frame-pointer-elim"="true" "no-frame-pointer-elim-non-leaf" "no-infs-fp-math"="false" "no-jump-tables"="false" "no-nans-fp-math"="false" "no-signed-zeros-fp-math"="false" "no-trapping-math"="false" "stack-protector-buffer-size"="8" "target-cpu"="x86-64" "target-features"="+fxsr,+mmx,+sse,+sse2,+x87" "unsafe-fp-math"="false" "use-soft-float"="false" }
attributes #1 = { "correctly-rounded-divide-sqrt-fp-math"="false" "disable-tail-calls"="false" "less-precise-fpmad"="false" "no-frame-pointer-elim"="true" "no-frame-pointer-elim-non-leaf" "no-infs-fp-math"="false" "no-nans-fp-math"="false" "no-signed-zeros-fp-math"="false" "no-trapping-math"="false" "stack-protector-buffer-size"="8" "target-cpu"="x86-64" "target-features"="+fxsr,+mmx,+sse,+sse2,+x87" "unsafe-fp-math"="false" "use-soft-float"="false" }

!llvm.ident = !{!0}

!0 = !{!"clang version 4.0.0 (tags/RELEASE_400/final)"}
```