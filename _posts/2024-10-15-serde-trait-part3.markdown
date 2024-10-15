---
title:  "Serde Trait - Part 3: Deserialization"
layout: default
date:   2024-10-15 08:22:59 +0100
tags: Rust, serde
---
<h1>Serde for trait objects - Part 3: Deserialization</h1>

In this series of blog posts I'm explaining how to use serde with trait objects:
- [Part 1: Overview]({% post_url 2024-10-01-serde-trait-part1 %})
- [Part 2: Serialization]({% post_url 2024-10-08-serde-trait-part2 %})
- [Part 3: Deserialization]({% post_url 2024-10-15-serde-trait-part3 %})
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
Today, our quest is to deserialize a trait object.
This will be an exercise in the visitor pattern.

We start with the following code, which defines a trait and a struct implementing it. 
{% highlight rust %}
trait Trait {}

#[derive(serde::Deserialize)]
struct S {
    data: i32,
}
impl Trait for S {  }
{% endhighlight %}

We try to deserialize a trait object instance.
Our json input will be:
{% highlight shell %}
{"S":{"data":0}}
{% endhighlight %}
which is a "key-value map" <b>{key: value}</b> in the serde world, and corresponds to the [externally tagged](https://serde.rs/enum-representations.html) serialization of an enum[^FootnotePreferenceInternallyTagged]. 
In our example, <b>{"S": {"data":0}}</b>, we have the key/type-info <b>"S"</b> and the value/type-serialization <b>{"data": 0}</b>.

To implement `Deserialize`, we start with the following snippet. 
Note that the "output" will be a boxed trait object, which is "just a type" (as opposed to a trait, and it is "owned"[^FootnoteOwnership]).
{% highlight rust %}
impl<'de> serde::Deserialize<'de> for Box<dyn Trait> {
    fn deserialize<D>(deserializer: D) -> Result<Self, D::Error>
    where
        D: serde::Deserializer<'de>,
    {
        // deserialization code
    }
}
{% endhighlight %}

Since we want to deserialize a "key-value map", we want to call `deserializer.deserialize_map`, so we need a visitor:

{% highlight rust %}
impl<'de> serde::Deserialize<'de> for Box<dyn Trait> {
    fn deserialize<D>(deserializer: D) -> Result<Self, D::Error>
    where
        D: serde::Deserializer<'de>,
    {
        let visitor = HelperVisitor {};
        deserializer.deserialize_map(visitor)
    }
}
struct HelperVisitor {}
impl<'de> serde::de::Visitor<'de> for HelperVisitor {
    type Value = Box<dyn Trait>;

    fn expecting(&self, formatter: &mut std::fmt::Formatter) -> std::fmt::Result {
        write!(formatter, "Trait object 'dyn Trait'")
    }

    fn visit_map<A>(self, map: A) -> Result<Self::Value, A::Error>
    where
        A: serde::de::MapAccess<'de>,
    {
        // deserialization code
    }
}
{% endhighlight %}


In our visitor helper, we first need to deserialize the key. 
Recall that we serialized the key as a String.
{% highlight rust %}
fn visit_map<A>(self, map: A) -> Result<Self::Value, A::Error>
where
    A: serde::de::MapAccess<'de>,
{
    let type_info = map.next_key::<String>()?.ok_or(serde::de::Error::custom(
        "Expected externally tagged 'dyn Trait'",
    ))?; 
    // deserialize underlying type, using type_info
}
{% endhighlight %}

At this point, I always prefer to see the code in action, so here is a debuggable snippet, which compiles, outputs the deserialized type information, and panics.
{% highlight rust %}
trait Trait {}

#[derive(serde::Deserialize)]
struct S {
    data: i32,
}
impl Trait for S {}

impl<'de> serde::Deserialize<'de> for Box<dyn Trait> {
    fn deserialize<D>(deserializer: D) -> Result<Self, D::Error>
    where
        D: serde::Deserializer<'de>,
    {
        let visitor = HelperVisitor {};
        deserializer.deserialize_map(visitor)
    }
}
struct HelperVisitor {}
impl<'de> serde::de::Visitor<'de> for HelperVisitor {
    type Value = Box<dyn Trait>;

    fn expecting(&self, formatter: &mut std::fmt::Formatter) -> std::fmt::Result {
        write!(formatter, "Trait object 'dyn Trait'")
    }

    fn visit_map<A>(self, mut map: A) -> Result<Self::Value, A::Error>
    where
        A: serde::de::MapAccess<'de>,
    {
        let type_info = map.next_key::<String>()?.ok_or(serde::de::Error::custom(
            "Expected externally tagged 'dyn Trait'",
        ))?;    
        dbg!(&type_info);
        todo!();
    }
}

fn main() {
    let json = r#"{"S":{"data":0}}"#;
    let t: Box<dyn Trait> = serde_json::from_str(json).unwrap();
}
{% endhighlight %}

Let's take a step back. What do we know now? 
- We want to deserialize some json into a boxed trait object, `Box<dyn Trait>`.
- After serialization of the type information, we know that our underlying type is `S`.
- The type `S` implements `Deserialize`
- So we deserialize the remaining part of our json into an instance of our type `S`.
- Finally, we cast our instance of `S` into a boxed trait object

Let's implement this:

{% highlight rust %}
fn visit_map<A>(self, mut map: A) -> Result<Self::Value, A::Error>
where
    A: serde::de::MapAccess<'de>,
{
    let type_info = map.next_key::<String>()?.ok_or(serde::de::Error::custom(
        "Expected externally tagged 'dyn Trait'",
    ))?;    
    dbg!(&type_info);
    let s = map.next_value::<S>()?;
    let boxed_trait_object: Box<dyn Trait> = Box::new(s);
    Ok(boxed_trait_object)
}
{% endhighlight %}

This works and compiles, so we are done.

We are done, right?

Not?

Why not?

Well, we knew from our type_info that the underlying type is `S`, but given some other json, we might find some other type.
As of now, we hard-coded that the serialized type has to be `S`. In the enum correspondence, we assume that our enum has a single variant.

Let's try to be a bit for flexible. To this end we use `map.next_value_seed` instead of `map.next_value`. Visitors for the rescue!
{% highlight rust %}
fn visit_map<A>(self, mut map: A) -> Result<Self::Value, A::Error>
where
    A: serde::de::MapAccess<'de>,
{
    let type_info = map.next_key::<String>()?.ok_or(serde::de::Error::custom(
        "Expected externally tagged 'dyn Trait'",
    ))?;
    struct TypeVisitor {
    }
    impl<'de> serde::de::DeserializeSeed<'de> for TypeVisitor {
        type Value = Box<dyn Trait>;

        fn deserialize<D>(self, deserializer: D) -> Result<Self::Value, D::Error>
        where
            D: serde::Deserializer<'de>,
        {
            // what to do here?               
        }
    }
    
    let boxed_trait_object = map.next_value_seed(TypeVisitor {  })?;
    Ok(boxed_trait_object)
}
{% endhighlight %}

Recall from our dotnet-digression in Part 1, that we need a runtime-reflection mechanism at some point.
This mechanism will be covered in the next part. 
Today, I want to finish the deserialization machine.

First, we introduce an abstract <b>deserialization function</b>.
We need this to be non-generic, so we employ <b>erased_serde</b> as follows:
{% highlight rust %}
type DeserializeFn = 
    fn(&mut dyn erased_serde::Deserializer) -> erased_serde::Result<Box<dyn Trait>>;
{% endhighlight %}

Next, we enhance our visitor:
{% highlight rust %}
struct TypeVisitor {
    deserialize_fn: DeserializeFn,
}
impl<'de> serde::de::DeserializeSeed<'de> for TypeVisitor {
    type Value = Box<dyn Trait>;

    fn deserialize<D>(self, deserializer: D) -> Result<Self::Value, D::Error>
    where
        D: serde::Deserializer<'de>,
    {
        let mut erased = <dyn erased_serde::Deserializer>::erase(deserializer);
        let deserialize_fn = self.deserialize_fn;
        deserialize_fn(&mut erased).map_err(|e| serde::de::Error::custom(e))    
    }
}
{% endhighlight %}

Finally, we generate a dummy runtime-reflection:

{% highlight rust %}
fn runtime_reflection(type_info: &str) -> Option<DeserializeFn> {
    if type_info == "S" {
        let deserialize_fn = |deserializer: &mut dyn erased_serde::Deserializer| {
            let s: S = erased_serde::deserialize(deserializer)?;
            let boxed_trait_object: Box<dyn Trait> = Box::new(s);
            Ok(boxed_trait_object)
        };
        Some(deserialize_fn)
    } else {
        None
    }
}
{% endhighlight %}

This works!
Now, given our `runtime_reflection` function, we can deserialize any instance of our trait object. Hurray!

Here is a complete code snippet:
{% highlight rust %}
trait Trait {}

#[derive(serde::Deserialize)]
struct S {
    data: i32,
}
impl Trait for S {}

impl<'de> serde::Deserialize<'de> for Box<dyn Trait> {
    fn deserialize<D>(deserializer: D) -> Result<Self, D::Error>
    where
        D: serde::Deserializer<'de>,
    {
        let visitor = HelperVisitor {};
        deserializer.deserialize_map(visitor)
    }
}

type DeserializeFn =
    fn(&mut dyn erased_serde::Deserializer) -> erased_serde::Result<Box<dyn Trait>>;
fn runtime_reflection(type_info: &str) -> Option<DeserializeFn> {
    if type_info == "S" {
        let deserialize_fn = |deserializer: &mut dyn erased_serde::Deserializer| {
            let s: S = erased_serde::deserialize(deserializer)?;
            let boxed_trait_object: Box<dyn Trait> = Box::new(s);
            Ok(boxed_trait_object)
        };
        Some(deserialize_fn)
    } else {
        None
    }
}

struct TypeVisitor {
    deserialize_fn: DeserializeFn,
}
impl<'de> serde::de::DeserializeSeed<'de> for TypeVisitor {
    type Value = Box<dyn Trait>;

    fn deserialize<D>(self, deserializer: D) -> Result<Self::Value, D::Error>
    where
        D: serde::Deserializer<'de>,
    {
        let mut erased = <dyn erased_serde::Deserializer>::erase(deserializer);
        let deserialize_fn = self.deserialize_fn;
        deserialize_fn(&mut erased).map_err(|e| serde::de::Error::custom(e))
    }
}

struct HelperVisitor {}
impl<'de> serde::de::Visitor<'de> for HelperVisitor {
    type Value = Box<dyn Trait>;

    fn expecting(&self, formatter: &mut std::fmt::Formatter) -> std::fmt::Result {
        write!(formatter, "Trait object 'dyn Trait'")
    }

    fn visit_map<A>(self, mut map: A) -> Result<Self::Value, A::Error>
    where
        A: serde::de::MapAccess<'de>,
    {
        let type_info = map.next_key::<String>()?.ok_or(serde::de::Error::custom(
            "Expected externally tagged 'dyn Trait'",
        ))?;
        let deserialize_fn = runtime_reflection(&type_info).ok_or(serde::de::Error::custom(
            format!("Unknown type for 'dyn Trait': {type_info}"),
        ))?;
        let boxed_trait_object = map.next_value_seed(TypeVisitor { deserialize_fn })?;
        Ok(boxed_trait_object)
    }
}

fn main() {
    let json = r#"{"S":{"data":0}}"#;
    let t: Box<dyn Trait> = serde_json::from_str(json).unwrap();
}
{% endhighlight %}

For fun, let's rewrite the closure in our runtime-reflection.
Currently, we have:
{% highlight rust %}
let deserialize_fn = |deserializer: &mut dyn erased_serde::Deserializer| {
    let s: S = erased_serde::deserialize(deserializer)?;
    let boxed_trait_object: Box<dyn Trait> = Box::new(s);
    Ok(boxed_trait_object)
};
{% endhighlight %}
And here is some nice, generic code.
The bounds are actually requested by the compiler, so we just add them.
[^FootnoteTraitBounds]
{% highlight rust %}
fn deserialize_fn_generic<A>(
    deserializer: &mut dyn erased_serde::Deserializer,
) -> erased_serde::Result<Box<dyn Trait>>
where
    A: Trait,
    A: serde::de::DeserializeOwned,
    A: 'static,
{
    let a: A = erased_serde::deserialize(deserializer)?;
    let boxed_trait_object: Box<dyn Trait> = Box::new(a);
    Ok(boxed_trait_object)
}
{% endhighlight %}
Using this, we update our runtime-reflection function, which looks already quite good.
{% highlight rust %}
fn runtime_reflection(type_info: &str) -> Option<DeserializeFn> {
    fn deserialize_fn_generic<A>(
        deserializer: &mut dyn erased_serde::Deserializer,
    ) -> erased_serde::Result<Box<dyn Trait>>
    where
        A: Trait,
        A: serde::de::DeserializeOwned,
        A: 'static,
    {
        let a: A = erased_serde::deserialize(deserializer)?;
        let boxed_trait_object: Box<dyn Trait> = Box::new(a);
        Ok(boxed_trait_object)
    }
    if type_info == "S" {
        Some(deserialize_fn_generic::<S>)
    } else {
        None
    }
}
{% endhighlight %}

That's it for today. 
Thank you for following the blog series. 

Next time we will proceed with runtime reflection and replace the hard-coded 
{% highlight rust %}
if type_info == "S"
{% endhighlight %}
by a lookup-table.


<h3>Footnotes</h3>
[^FootnotePreferenceInternallyTagged]: Personally, I prefer the internally tagged variant. But this is quite more involved to code. There might be a later post about it. If you are interested in reading about it, please give me some feedback.
[^FootnoteOwnership]: I'm referring to the fact that `Box<dyn Trait>` actually means `Box<dyn Trait + 'static>`. Part 5 is about lifetimes.
[^FootnoteTraitBounds]: Recall from the other footnote that `Box<dyn Trait>` actually means `Box<dyn Trait + 'static>`. Hence, the bound `A:'static` is not surprising. More on lifetimes is discussed in part 5.