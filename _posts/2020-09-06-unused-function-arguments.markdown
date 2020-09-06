---
title:  "Zero cost abstractions: Unused arguments in Rust"
layout: default
date:   2020-09-06 21:58:48 +0100
tags: Rust
---
<h1>Topic</h1>
This short entry is a longer answer to [stackoverflow][stackoverflow].
<br/>
The questions is:
<br/>
<b>
Is there a (negative) performance impact due to unused function arguments (in Rust)?</b>
<br/>
Claim: There is no runtime performance impact. The compiler removes all unused function arguments (in release mode).

<h1>Free Functions</h1>
To this end, we will consider the following three functions:
{% highlight rust %}
fn single_argument(x: u32) -> u32 {
    x + 424242
}
fn too_many_arguments(x: u32, y: u32) -> u32 {
    x + 424242
}
fn enough_arguments(x: u32, y: u32) -> u32 {
    x + y
}
{% endhighlight %}
Lets have a look at the generate assembly ([playground_1](
https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=0df6168f1becd11062f6c51cd9aabfde)):
{% highlight Sass %}
playground::single_argument:
	pushq	%rax
	movl	%edi, 4(%rsp)
	addl	$424242, %edi
	setb	%al
	testb	$1, %al
	movl	%edi, (%rsp)
	jne	.LBB0_2
	movl	(%rsp), %eax
	popq	%rcx
	retq

playground::too_many_arguments:
	subq	$24, %rsp
	movl	%edi, 16(%rsp)
	movl	%esi, 20(%rsp)
	addl	$424242, %edi
	setb	%al
	testb	$1, %al
	movl	%edi, 12(%rsp)
	jne	.LBB1_2
	movl	12(%rsp), %eax
	addq	$24, %rsp
	retq

playground::enough_arguments:
	subq	$24, %rsp
	movl	%edi, 16(%rsp)
	movl	%esi, 20(%rsp)
	addl	%esi, %edi
	setb	%al
	testb	$1, %al
	movl	%edi, 12(%rsp)
	jne	.LBB2_2
	movl	12(%rsp), %eax
	addq	$24, %rsp
	retq
{% endhighlight %}
Since this is a bit much to understand, release mode:
{% highlight Sass %}
playground::single_argument:
	leal	424242(%rdi), %eax
	retq

playground::too_many_arguments:
	leal	424242(%rdi), %eax
	retq

playground::enough_arguments:
	leal	(%rdi,%rsi), %eax
	retq
{% endhighlight %}
As promised, no difference between <b>single_argument</b> & <b>too_many_arguments</b>.
I guess (I'm not an assembly expert), that in debug mode <b>too_many_arguments</b> writes its argument <b>y</b> into a register.

Last check: We want to check there is some overhead at the call site.
Note that the following example forbids inlining and uses each function twice to ensure that the compiler does not replace function calls by their return values.
{% highlight rust %}
#[inline(never)]
#[no_mangle]
fn single_argument(x: u32) -> u32 {
    x + 424242
}

#[inline(never)]
#[no_mangle]
fn too_many_arguments(x: u32, y: u32) -> u32 {
    x + 424242
}

#[inline(never)]
#[no_mangle]
fn enough_arguments(x: u32, y: u32) -> u32 {
    x + y
}

fn main() {
    println!("Result: {:?}", too_many_arguments(112, 113));
    println!("Result: {:?}", enough_arguments(114, 115),);
    println!("Result: {:?}", single_argument(116));
    println!("Result: {:?}", too_many_arguments(117, 118));
    println!("Result: {:?}", enough_arguments(119, 120),);
    println!("Result: {:?}", single_argument(121));
}
{% endhighlight %}
And the generated assembly (in release mode):
{% highlight Sass %}
playground::main:
  /* unimportant */	 	
  movl	$112, %edi
	callq	too_many_arguments
  /* unimportant */	 	
  movl	$114, %edi
	movl	$115, %esi
	callq	enough_arguments
	/* unimportant */	   	
  movl	$116, %edi
	callq	single_argument
	/* unimportant */	 	
  movl	$117, %edi
	callq	too_many_arguments
	/* unimportant */	 
  movl	$119, %edi
	movl	$120, %esi
	callq	enough_arguments
	/* unimportant */	 
  movl	$121, %edi
	callq	single_argument
	/* unimportant */	   


{% endhighlight %}
We observe that calls to <b>enough_arguments</b> are preceed by two <b>movl</b> (with the expected numbers), whereas both <b>too_many_arguments</b> & <b>single_argument</b> are only preceeded by a single <b>movl</b>. 
<br>
Hence: In release mode, there is no runtime overhead associated with unused function arguments.

Sideremark: In debug mode we see the expected overhead:
{% highlight Sass %}
playground::main:
  /* unimportant */	 	
  movl	$112, %edi
  movl	$113, %esi
	callq	too_many_arguments
  /* skipped */   
  
{% endhighlight %}

<h1>Traits</h1>

The questions which motivated this post is slightly different.

Let's say we have some trait:
{% highlight rust %}
trait DummyTrait {
    fn add(&self, x: u32, y: u32) -> u32;  
}
{% endhighlight %} 
and we have an implementor who is not using all arguments:
{% highlight rust %}
fn new(z: u32) -> DummyStruct {
    DummyStruct { z }
}

impl DummyTrait for DummyStruct {
    fn add(&self, x: u32, _: u32) -> u32 {
        self.z + x + 424242
    }
}
{% endhighlight %} 
Question: Does this lead to run-time costs?

The expected answer is: Since LLVM does not know about traits, it basically sees only free function, hence there is no overhead.

But lets look at some more assembly.
Here is the example rust: (Again, no inlining and double function calls to avoid optimizations.)
{% highlight rust %}
trait DummyTrait {
    #[inline(never)]
    fn add(&self, x: u32, y: u32) -> u32 {
        y + 1234567
    }
}

struct DummyStruct {
    z: u32,
}

#[inline(never)]
#[no_mangle]
fn new(z: u32) -> DummyStruct {
    DummyStruct { z }
}

impl DummyTrait for DummyStruct {
    #[inline(never)]
    fn add(&self, x: u32, _: u32) -> u32 {
        self.z + x + 424242
    }
}

fn main() {
    let dummy = new(12345);
    println!("Result: {:?}", dummy.add(110, 111));
    println!("Result: {:?}", dummy.add(112, 113));
}
{% endhighlight %} 
and the relevant assembly:
{% highlight Sass %}
playground::main:
  /* unimportant */	 	
	movl	$12345, %edi
	movl	$110, %esi
	callq	<playground::DummyStruct as playground::DummyTrait>::add
  /* unimportant */	 	
	movl	$12345, %edi
	movl	$112, %esi
	callq	<playground::DummyStruct as playground::DummyTrait>::add
  /* unimportant */
{% endhighlight %}
Once again, no <b>$111</b> or <b>$113</b>, so no overhead.

Can we see some overhead? Yes, like this:
{% highlight rust %}
trait DummyTrait {
    #[inline(never)]
    fn add(&self, x: u32, y: u32) -> u32 {
        y + 1234567
    }
}

struct DummyStruct {
    z: u32,
}

#[inline(never)]
#[no_mangle]
fn new(z: u32) -> DummyStruct {
    DummyStruct { z }
}

impl DummyTrait for DummyStruct {
    #[inline(never)]
    fn add(&self, x: u32, _: u32) -> u32 {
        self.z + x + 424242
    }
}

#[inline(never)]
#[no_mangle]
fn overhead(d:&dyn DummyTrait, x:u32,y:u32) -> u32 {
    d.add(x,y)
}

fn main() {
    let dummy = new(12345);
    println!("Result: {:?}", overhead(&dummy, 114, 115));
    println!("Result: {:?}", overhead(&dummy, 116, 117));
}
{% endhighlight %} 
and the relevant assembly:
{% highlight Sass %}
playground::main:
  /* unimportant */	 	
	movl	$114, %edx
	movl	$115, %ecx
	callq	overhead
  /* unimportant */
{% endhighlight %}
But here we call into an unknown trait implementation, using dynamic dispatch.
So: What else should the compiler do? [^unnecessaryoptimization]

Last topic: Default implementation [playground_2](https://play.rust-lang.org/?version=stable&mode=release&edition=2015&gist=f27cb97a33d0e85cc46877b1d799b56e)
{% highlight rust %}
trait DummyTrait {
    #[inline(never)]
    fn add(&self, x: u32, y: u32) -> u32 {
        y + 1234567
    }
}

struct DummyStruct2 {
    z: u32,
}

#[inline(never)]
#[no_mangle]
fn new2(z: u32) -> DummyStruct2 {
    DummyStruct2 { z }
}

impl DummyTrait for DummyStruct2 {}

fn main() {
    let dummy2 = new2(12345);
    println!("Result: {:?}", dummy2.add(114, 115));
    println!("Result: {:?}", dummy2.add(116, 117));
}
{% endhighlight %} 
and the relevant assembly:
{% highlight Sass %}
playground::main:
  /* unimportant */	 	
	movl	$115, %edi
	callq	playground::DummyTrait::add
	/* unimportant */
  movl	$117, %edi
	callq	playground::DummyTrait::add
	/* unimportant */
{% endhighlight %}
Once again, we do not see any overhead. This is expected, since we still do static dispatch, so have a free function in LLVM (only the location of the function's source code changed).

<h1>Summary</h1>
Let's recall the definition of zero-cost abstraction due to Bjarne Stroustrup:
<br>
<i>What you don’t use, you don’t pay for. And further: What you do use, you couldn’t hand code any better.</i>
<br>
So we have checked:
Unused function arguments are a zero-cost abstraction in Rust.

Note that there is some overhead in debug mode.

Moreover, there is a linter warning about those parameters, which is really helpful.
Also, the tooling (i.e., rust playground or godbolt) is really nice to have. It easily allows to look at the generated assembly.

[stackoverflow]: https://stackoverflow.com/questions/63697356/will-rust-optimize-away-unused-function-arguments
[^unnecessaryoptimization]: Well, it could reason that there is only a unique implementation of DummyTrait. But this optimization seems unnecessary, since the whole point of dynamic dispatch is to support multiple implementations.