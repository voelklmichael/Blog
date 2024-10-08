---
title:  "Serde Trait - Part 2: Serialization"
layout: default
date:   2024-10-08 20:08:59 +0100
tags: Rust, serde
---
<h1>Serde for trait objects - Part 2: Serialization</h1>

In this series of blog posts I'm explaining how to use serde with trait objects:
- [Part 1: Overview]({% post_url 2024-10-01-serde-trait-part1 %})
- [Part 2: Serialization]({% post_url 2024-10-08-serde-trait-part2 %})
- Part 3: Deserialization
- Part 4: Registry
- Part 5: Lifetimes
- Part 6: Sync/Send
- Part 7: Macro Part A: Trait
- Part 8: Marco Part B: Implementation

Remark: If you are in a situation where you want to serialize a trait object, please take a step back.
Check if you can replace your trait object with an enum.
In my experience, the enum approach is much easier to work with.

Remark: All topics covered here are well-known. We follow [typetag](https://github.com/dtolnay/typetag).

So, let's start.
Today, our quest is to serialize a trait object.
This will be a short post.

We start with the following code, which defines a trait and a struct implementing it. 
We try to serialize a trait object instance:
{% highlight rust %}
trait Trait: erased_serde::Serialize {}

#[derive(serde::Serialize)]
struct S {
    data: i32,
}
impl Trait for S {  }

impl serde::Serialize for dyn Trait {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: serde::Serializer,
    {
        erased_serde::serialize(self, serializer)
    }
}

fn main() {
    let s = S { data: 0 };
    let t: &dyn Trait = &s;
    let ser = serde_json::to_string(t).unwrap();    
    println!("Serialized json: {}", ser);
}
{% endhighlight %}

This compiles and prints:
{% highlight shell %}
Serialized json: {"data":0}
{% endhighlight %}

Last time, we have seen, that we need to enhance the serialized trait object with some information about the underlying type.
Note, there are several possibilities to serialize an enum ([enum representation](https://serde.rs/enum-representations.html)).
Today, we will use "externally tagged", which is the default for enums in Serde.
[^FootnoteLastTimeDotnetInternallyTagged]

Hence, our aim is to get the following output:
{% highlight shell %}
Serialized json: {"S":{"data":0}}
{% endhighlight %}

First, we need to get some type information. In the final version, the addition of the function to the trait will be part of the macro. [^FootnoteProcMacro]
{% highlight rust %}
trait Trait: erased_serde::Serialize {
    // New requirement: We need some information about our type.
    // This is a first draft (but good enough for typetag)
    fn type_info(&self) -> &'static str;
}

#[derive(serde::Serialize)]
struct S {
    data: i32,
}
impl Trait for S {    
    fn type_info(&self) -> &'static str {
        "S"
    }
}
{% endhighlight %}

Now we want to use this type information during serialization.
Note that our json template <b>{type-info: type-serialization}</b> corresponds to a "key-value map" <b>{key: value}</b> in the serde world. (In our example, <b>{"S": {"data":0}}</b>, we have the key/type-info <b>"S"</b> and the value/type-serialization <b>{"data": 0}</b>).

Here is our first try, which compiles, but is recursive - wrong method resolution used.
{% highlight rust %}
impl serde::Serialize for dyn Trait {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: serde::Serializer,
    {
        use serde::ser::SerializeMap;
        let mut ser = serializer.serialize_map(Some(1))?;
        let type_info = self.type_info();
        ser.serialize_entry(type_info, self)?;
        ser.end()
    }
}
{% endhighlight %}

The problem is that `serialize_entry` takes it arguments by reference. 
As we saw last time, this leads to a recursive function call. 
We can help the compiler by adding a new layer: We introduce a newtype wrapper `Wrap`, which in turn calls our working implementation of `serialize`.
{% highlight rust %}
struct Wrap<'a, T: ?Sized>(pub &'a T);
impl<'a, T> serde::Serialize for Wrap<'a, T>
where
    T: ?Sized + erased_serde::Serialize + 'a,
{
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: serde::Serializer,
    {
        erased_serde::serialize(self.0, serializer)
    }
}
{% endhighlight %}

This gives our final code:
{% highlight rust %}
trait Trait: erased_serde::Serialize {
    fn message(&self) -> String;

    // Will be generate by macro    
    fn type_info(&self) -> &'static str;
}

#[derive(serde::Serialize)]
struct S {
    data: i32,
}
impl Trait for S {
    fn message(&self) -> String {
        format!("Message: {}", self.data)
    }

    // Will be generate by macro    
    fn type_info(&self) -> &'static str {
        "S"
    }
}

struct Wrap<'a, T: ?Sized>(pub &'a T);
impl<'a, T> serde::Serialize for Wrap<'a, T>
where
    T: ?Sized + erased_serde::Serialize + 'a,
{
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: serde::Serializer,
    {
        erased_serde::serialize(self.0, serializer)
    }
}

impl serde::Serialize for dyn Trait {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: serde::Serializer,
    {
        use serde::ser::SerializeMap;
        let mut ser = serializer.serialize_map(Some(1))?;
        let type_info = self.type_info();        
        ser.serialize_entry(type_info, &Wrap(self))?;
        ser.end()
    }
}

fn main() {
    let s = S { data: 0 };
    let t: &dyn Trait = &s;
    let ser = serde_json::to_string(t).unwrap();
    println!("Serialized json: {}", ser);
}
{% endhighlight %}
This gives our target output:
{% highlight shell %}
Serialized json: {"S":{"data":0}}
{% endhighlight %}
So we're done for today. 
Thank you for following the blog series. 

Next time we will proceed with deserialization. Lifetimes for serialization (and deserialization) will be the content of Part 5.



<h3>Footnotes</h3>
[^FootnoteProcMacro]: Since derive macros cannot change the code, only add code, we will need a attribute macro.
[^FootnoteLastTimeDotnetInternallyTagged]: Last time, we used dotnet as motivation, which used "Internally tagged". This "Internally tagged" variant is my preferred variant for self-describing formats, but a lot more involved to implement in Rust. We will implement this is a later blog post.
