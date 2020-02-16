---
title: 3. 寄存器使用惯例
date: 2019-07-05
---

程序寄存器是唯一一个被所有过程共享的资源。虽然在给定时刻只能有一个过程是活动的，我们必须保证当一个过程（调用者）调用另一个（被调用者）时，被调用者不会覆盖某个调用者稍后会使用的寄存器的值。为此，`IA32` （Intel Architecture 32 bit）采用了一组统一的寄存器使用惯例，所有的过程都必须遵守，包括程序库中的过程。

根据惯例，寄存器 `eax`，`edx`，`ecx` 被划分为调用者保存（caller save）寄存器。当过程 P 调用 Q 时，Q 可以覆盖这些寄存器，而不会破坏 P 所需要的数据。另外，寄存器 `ebx`，`esi` 和 `edi` 被划分为被调用者保存（callee save）寄存器。这意味着 Q 必须在覆盖它们之前，将这些寄存器的值保存到栈中，并在返回前恢复它们，因为 P（或某个更高层次的过程）可能会在今后的计算中需要这些值。此外，根据这里描述的惯例，必须保存寄存器 `ebp`，`esp`。

如下面这段程序

```c
int P(int x){
	int y = x * x;
	int z = Q(y);
	return y + z;
}
```

过程 P 在调用 Q 之前计算 y，但是它必须保证 y 的值在 Q 返回后是可用的。有两种方式可以做到这一点：

1. 它可以在调用 Q 之前，将 y 的值存放在自己的栈帧中；当 Q 返回时，它可以从栈中取出 y 的值。
2. 它可以将 y 的值保存在被调用者保存寄存器中。如果 Q，或任何其他 Q 调用的程序，想使用这个寄存器，它必须将这个寄存器的值保存在栈帧中，并在返回前恢复该值。因此，当 Q 返回到 P 时，y 的值会在被调用者保存寄存器中，或者是因为寄存器根本就没有改变，或者是因为它被保存并恢复了。
