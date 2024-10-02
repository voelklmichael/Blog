---
title:  "Serde Trait - Part 1: Overview"
layout: default
date:   2024-10-01 20:21:59 +0100
tags: Rust, serde
---
<h1>Serde for trait objects - Part 1: Overview</h1>


Consider
{% highlight rust %}
trait Message {
    fn message(&self)->String;
}
#[derive(serde::Serialize, serde::Deserialize)]
struct MessageContainer {
    messages: Vec<Box<dyn Message>>
}
{% endhighlight %}

Here we have a trait object, `Box<dyn Message>`.
The aim of this series of blog posts is to make this code compile. 
Part of the solution will be to add a macro on top of the trait definition

Remark: All topics covered here are well-known. We follow [typetag](https://github.com/dtolnay/typetag).

Remark: If you are in a situation where you want to serialize a trait object, please take a step back.
Check if you can replace your trait object with an enum.
In my experience, the enum approach is much easier to work with.[^FootnoteRant]

In this series of blog posts I'm explaining how to use serde with trait objects:
- [Part 1: Overview]({% post_url 2024-10-01-serde-trait-part1 %})
- Part 2: Serialization
- Part 3: Deserialization
- Part 4: Registry
- Part 5: Lifetimes
- Part 6: Sync/Send
- Part 7: Macro Part A: Trait
- Part 8: Marco Part B: Implementation


<h1>How not to do it</h1>

First, we try a very naive approach - we start with the following snippet:
{% highlight rust %}
trait Trait {}

#[derive(serde::Serialize)]
struct S {}

impl Trait for S {}

impl serde::Serialize for dyn Trait {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: serde::Serializer,
    {
        todo!("We have to implement this")
    }
}

fn main() {
    let s = S {};
    let t: &dyn Trait = &s;
    let ser = serde_json::to_string(t).unwrap();
}
{% endhighlight %}

How to do implement the todo?

<h2>First idea: Recursion</h2>

We start with a stupid idea, a recursive call. 
Later, we will write recursive code by accident.

{% highlight rust %}
impl serde::Serialize for dyn Trait {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: serde::Serializer,
    {
        self.serialize(serializer)
    }
}
{% endhighlight %}

As expected, this yields a runtime overflow:

{% highlight shell %}
fatal runtime error: stack overflow
{% endhighlight %}

<h2>Second idea: Trait bound</h2>
We use serde::Serialize as a trait bound

{% highlight rust linenos %}
    trait Trait: serde::Serialize {}

    #[derive(serde::Serialize)]
    struct S {}
    impl Trait for S {}

    fn main() {
        let s = S {};
        let t: &dyn Trait = &s;
    }
{% endhighlight %}
This code does not compile, because our trait is no longer object-safe, 
{% highlight shell %}
error[E0038]: the trait `Trait` cannot be made into an object
   --> blog/src/main.rs:9:12
    |
9   |     let t: &dyn Trait = &s;
    |            ^^^^^^^^^^ `Trait` cannot be made into an object
    |
{% endhighlight %}


<h2>Third idea: Erased serde</h2>

The solution is to use [erased_serde::Serialize](https://github.com/dtolnay/erased-serde). 
This is an object safe version of the Serialize trait.
Instead of using a generic serializer argument, it uses a trait object:

{% highlight rust %}
// serde
impl serde::Serialize for dyn Trait {
    fn serialize<S: serde::Serializer>(&self, serializer: S) 
        -> Result<S::Ok, S::Error>;    
}
// erased_serde
pub trait Serialize: erased_serde::sealed::serialize::Sealed {
    fn erased_serialize(&self, serializer: &mut dyn erased_serde::Serializer) 
        -> Result<(), Error>;
}
{% endhighlight %}


Note that `erased_serde::Serialize` is sealed and cannot be implemented.
There is a blanket implementation of it for types implementing `serde::Serialize`, so we can use it.

Using the trait bound, this is our current code.

{% highlight rust linenos %}
trait Trait: erased_serde::Serialize {}

#[derive(serde::Serialize)]
struct S {}
impl Trait for S {}

impl serde::Serialize for dyn Trait {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: serde::Serializer,
    {
        todo!("We have to implement this")
    }
}

fn main() {
    let s = S {};
    let t: &dyn Trait = &s;
    let ser = serde_json::to_string(t).unwrap();
}
{% endhighlight %}

We still need to implement line 12, the serialization code.
But this is easy, `erased_serde` contains the right function, thankfully even called `serialize`:

{% highlight rust linenos %}
trait Trait: erased_serde::Serialize {}

#[derive(serde::Serialize)]
struct S {}
impl Trait for S {}

impl serde::Serialize for dyn Trait {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: serde::Serializer,
    {        
        erased_serde::serialize(self, serializer)
    }
}

fn main() {
    let s = S {};
    let t: &dyn Trait = &s;
    let ser = serde_json::to_string(t).unwrap();
}
{% endhighlight %}

This compiles and works as expected.

Note, if we change the implementation by adding a borrow, from 
{% highlight rust %}erased_serde::serialize(self, serializer)
{% endhighlight %}
to
{% highlight rust %}erased_serde::serialize(&self, serializer)
{% endhighlight %}

we end up with a runtime overflow
{% highlight shell %}thread 'main' has overflowed its stack
fatal runtime error: stack overflow
{% endhighlight %}

Method resolution is sometimes complicated!

<h2>Discussion</h2>

Finally, we can serialize our trait object.
But, unfortunately, this is not good enough.
Let's consider the following situation, having two different structs both implementing a given trait:

{% highlight rust %}
trait Trait: erased_serde::Serialize {
    fn message(&self) -> String;
}

#[derive(serde::Serialize)]
struct S1 {
    data: i32,
}
impl Trait for S1 {
    fn message(&self) -> String {
        format!("Message: {}", self.data)
    }
}

#[derive(serde::Serialize)]
struct S2 {
    data: u64,
}
impl Trait for S2 {
    fn message(&self) -> String {
        "Message independent of data".into()
    }
}

impl serde::Serialize for dyn Trait {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: serde::Serializer,
    {
        erased_serde::serialize(self, serializer)
    }
}

fn main() {
    let s1 = S1 { data: 0 };
    let t1: &dyn Trait = &s1;
    let ser1 = serde_json::to_string(t1).unwrap();
    let s2 = S2 { data: 0 };
    let t2: &dyn Trait = &s2;
    let ser2 = serde_json::to_string(t2).unwrap();
    println!("T1: {}", ser1);
    println!("T2: {}", ser2);
}
{% endhighlight %}

This outputs:
{% highlight shell %}
T1: {"data":0}
T2: {"data":0}
{% endhighlight %}

Both trait objects yield the same serialized json string! How should we deserialize this correctly?

Remark: This is the same issue that <b>serde</b> has with serialization of `enum`s, if one opts into the <b>untagged representation</b>.
See [serde enum representation](https://serde.rs/enum-representations.html) for a detailed discussion. 
As always in this space ("serialization of trait objects"), learn what `enum`s are doing, and avoid what they are avoiding!
[^FootnoteJokeEnumsAvoidTraitObjects]

<h2>Digression: Dotnet</h2>

Let's think outside of our rusty box, and check what <b>dotnet</b> is doing.
[^FootnoteDotnetNewtonsoft]
The most important part of the snippet is it's output, shown below. So please feel free to skip the C#-code.

{% highlight C# %}
public interface IMessage
{
    string Message();
}
public class S1 : IMessage
{
    public int Data { get; set; }

    string IMessage.Message() => $"Data: {Data}";
}
public class S2 : IMessage
{
    public int Data { get; set; }

    string IMessage.Message() => "Message independent of data";
}

public class MessageContainer
{
    public List<IMessage>? Messages { get; set; }
}

static class Program
{
    static void Main()
    {
        var messages = new MessageContainer
        {
            Messages = new List<IMessage>{
                new S1 { Data = 0 },
                new S2 { Data = 0 },
            }
        };
        var settings = new Newtonsoft.Json.JsonSerializerSettings()
        {
            TypeNameHandling = Newtonsoft.Json.TypeNameHandling.Auto,
            Formatting = Newtonsoft.Json.Formatting.Indented
        };
        Console.WriteLine(Newtonsoft.Json.JsonConvert.SerializeObject(messages, settings));
    }
}
{% endhighlight %}

This prints:
{% highlight shell %}
{
  "Messages": [
    {
      "$type": "S1, serdeTraitDotnet",
      "Data": 0
    },
    {
      "$type": "S2, serdeTraitDotnet",
      "Data": 0
    }
  ]
}
{% endhighlight %}

We see: Both structs are enhanced by some type information, consisting of type name and the crate (called "assembly" in the DotNet world) it was defined in.
How does this help with deserialization?
Dotnet has "reflection", which means we can query the runtime during deserialization. 
Hence we can give our type information to the runtime, and it will look up the type for us and give us some constructor, which in turn will allows us deserialize the data into the given type. 
Finally, we cast the deserialized instance into to our interface, and we are done.

So, here are our tasks.
- Task 1: enhance the serialized data with some type information.
- Task 2: build some kind of runtime type database which allows querying types given our (de-)serialized type information
- Task 3: deserialize the serialized type into our trait object

Since Task 3 depends on the API of Task 2, we will proceed in the following order
- Task 1: Mostly straight-forward, since we already can serialize trait objects
- Task 3: An exercise in the visitor pattern
- Task 2: Some trade-offs will be discussed

Note that, once we have complete those three tasks, we will still need to write derive macros and so on, so we have a reasonable amount of work in front of us.

This is it for today, 
the next post 
will be about serialization and complete task 1.


<h3>Footnotes</h3>
[^FootnoteRant]: (Rant) This series is going to be my reference answer to the question: "I'm struggling with trait objects. How do I solve problem XYZ?" So that I can say: "I suggest using an enum instead of a trait object. This is often much easier. For example, if you want to use serde for you trait object, you need to work through all of the following."
[^FootnoteDotnetNewtonsoft]: I'm aware that this snippet is using Newtonsoft.Json instead of the System.Text.Json, but this is a topic for another day …
[^FootnoteJokeEnumsAvoidTraitObjects]: (Joke) Note that `enum`s avoid trait objects, hence you should also avoid trait objects ;-)
