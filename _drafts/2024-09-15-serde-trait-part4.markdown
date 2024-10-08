---
title:  "Serde Trait - Part 4: Registry"
layout: default
date:   2024-09-15 08:22:59 +0100
tags: Rust, serde
---
<h1>Serde for trait objects - Part 4: Registry</h1>

In this series of blog posts I'm explaining how to use serde with trait objects:

- Part 5: Lifetimes


Remark: If you are in a situation where you want to serialize a trait object, please take a step back.
Check if you can replace your trait object with an enum.
In my experience, the enum approach is much easier to work with.

Remark: All topics covered here are well-known. We follow [typetag](https://github.com/dtolnay/typetag) [^FootNoteTypeTagNotFollowing].

So, let's start.
Today, our quest is to implement a basic form of runtime reflection.
This will be technically simple, but we will discuss several choices and tradeoffs.

<h2>The aim</h2>

Last time, we needed a function with the following signature
{% highlight rust %}
type DeserializeFn = fn(&mut dyn erased_serde::Deserializer) -> 
    erased_serde::Result<Box<dyn Trait>>;
fn runtime_reflection(type_info: &str) -> Option<DeserializeFn> {...}
{% endhighlight %}
where `dyn Trait` is our trait which we want to deserialize.

Some remarks about this function:

<h3> Remark 1: Input</h3>
The type_info is currently a borrowed string. 

This is a choice, and we follow [typetag](https://github.com/dtolnay/typetag) here. 
But for other applications, more data might be needed. In our DotNet example, we saw that the NewtonSoft json stores both the type name, module name, crate name and crate version:
{% highlight shell %}
 "SomeType": "Namespace.TypeName, LibraryName, Version=1.0.0.0, Culture=neutral, PublicKeyToken=....", 
{% endhighlight %}
NewtonSoft does store all this information in a string, but we could stores this information in some data type, and (de-)serialize this type instead. 

Note that NewtonSoft serialize [^FootnoteNewtonsoftOptions] interfaces using the [internally tagged](https://serde.rs/enum-representations.html) representation.
We use currently the [externally tagged](https://serde.rs/enum-representations.html) one, and for json, this actually forces us to use a string as key.
So, everything is a trade-off.


<h3> Remark 2: Output</h3>

The return type should be a deserializer in the happy path, no question. 

But for the error path, there is a choice. 
Indeed, <b>TypeTag</b> has a result which indicates in the error case:
1. There is no deserializer available
2. There are several deserializer available

For today, we will detect error case 2. at some other point, and hence us an `Option`. 
(Recall that the error message was already included in the code we wrote last time)

<h3> Remark 3: Ownership</h3>

Should our function operate on some additional input data, for example, should it be `&mut self`?
{% highlight rust %}
fn runtime_reflection(&mut self, type_info: &str) -> Option<DeserializeFn> {...}
{% endhighlight %}
We note that several threads might deserialize the same trait object at the same time. Hence we cannot have mutable state in our function signature[^FootnoteDeserializeMultipleThreatsNoMutableState]! We are forced to have a static function.

<h2>How to implement it</h2>

Now, we finally implement our function:
{% highlight rust %}
pub static REGISTRY_TRAIT: Registry = Registry::new();
fn runtime_reflection(type_info: &str) -> Option<DeserializeFn> {
    REGISTRY_TRAIT.get(type_info)
}
{% endhighlight %}

Ha, that was easy.

But what is this type `Registry<dyn Trait>`?

Again, there are some choices, and again, we will deviate from [typetag](https://github.com/dtolnay/typetag).

Basically, the registry should be a dictionary: `type_info` in, `DeserializeFn` out.

But, we should also be apply to add some data to it. And it has to be static. So we need a mutable static.
This leads to the following design, which is one of many possibles:
{% highlight rust %}
type DeserializeFn = fn(&mut dyn erased_serde::Deserializer) -> 
    erased_serde::Result<Box<dyn Trait>>;
pub struct Registry {
    data: std::sync::Mutex<std::collections::BTreeMap<String, DeserializeFn>>,
}
{% endhighlight %}
The API for our `Registry` is quite simple, only `new`, `insert` and `get`:
{% highlight rust %}
impl Registry {
    // Create a new registry. Note: This item is private
    const fn new() -> Self {
        Self {
            data: std::sync::Mutex::new(std::collections::BTreeMap::new()),
        }
    }
    /// Fetch a deserializer from the collection. Returns None if the requested type is unknown.
    fn get(&self, type_info: &str) -> Option<DeserializeFn> {
        self.data.lock().unwrap().get(type_info).copied()
    }
    /// Insert a deserializer into the collection.
    /// If there already is one for the requested type, exchange it with the previous one and return the previous one.
    pub fn insert(
        &self,
        type_info: String,
        deserializer: DeserializeFn,
    ) -> Result<(), DeserializeFn> {
        match self.data.lock().unwrap().insert(type_info, deserializer) {
            Some(deserializer) => Err(deserializer),
            None => Ok(()),
        }
    }
}
{% endhighlight %}
Note that we use the `new` only once to create the registry. 
This is part of the `Deserialize`-implementation for `Box<dyn Trait>`, and hence should be done in the same module in which `Trait` is defined (if not automatically be a derive macro[^FootnoteDeriveMacroNotPossible]).
So, `new` can be private.

The get function is called in our `runtime_reflection` function, which is part of the `Deserialize`-implementation, hence it can also be private.

The most interesting function is `insert`. 
First, recall that we define this helper function last time:
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
This allows us to simply the method slightly
{% highlight rust %}
pub fn insert<A>(&self, type_info: String) -> Result<(), DeserializeFn>
where
    A: Trait,
    A: serde::de::DeserializeOwned,
    A: 'static,
{
    match self.data.lock().unwrap().insert(type_info, deserialize_fn_generic::<A>) {
        Some(deserializer) => Err(deserializer),
        None => Ok(()),
    }
}
{% endhighlight %}
But the important questions are: 
Who should call it when? 

To get some code which compiles, we call it ourselves in main, like this:
{% highlight rust %}
#[derive(serde::Deserialize)]
struct S {
    data: i32,
}
impl Trait for S {}

fn main() {
    REGISTRY_TRAIT.insert::<S>("S".into()).unwrap();
}
{% endhighlight %}

Remark: There is a fully working example at the end of this blog post.

<h2>TypeTag's magic</h2>

Actually, we want to build the runtime-reflection machinery automatically in the background.
But <b>Rust</b> does not support this out-of-the box[^FootnoteWishForRust2027].

But, of course, [u/dtolnay](https://github.com/dtolnay) managed to work around this in [typetag](https://github.com/dtolnay/typetag).
In typetag, you need to mark both the trait definition and all implementation blocks by adding macros like this:
{% highlight rust %}
#[typetag::serde]
trait Trait {
    ...
}
#[typetag::serde]
impl Trait for S {
    ...
}
{% endhighlight %}
The impl block will be enhanced so that all the functions needed for (de-)serializing are in place, for example the type_info.
In addition, the `submit!` macro from [inventory](https://github.com/dtolnay/inventory) is used.

The trait itself will also be enhanced (`serde::Serialize` gets implement for `dyn Trait` and `serde::Deserialize` for `Box<dyn Trait>`).
In addition, the `collect!` macro from [inventory](https://github.com/dtolnay/inventory) is used.

Inventory in turn uses some linker magic to call some function "before-main", to build up the registration.
As a benefit, typetag's registry doesn't need synchronization as we do (we use a `Mutex`).
Unfortunately, the downside is that this is not portable[^FootnoteWishForRust2027] (for example, this doesn't with WASM).

Also, since all the type registration happens in the background - but automatically, which is nice - there is no error message.
Consider the following example:
You have a trait implemented in your app, and load plugins via [libloading](https://github.com/nagisa/rust_libloading).
Once the library is loaded, typetag will register the implementors of your trait automatically[^FootnoteRaceCondition].
But assume, you already have loaded another version of your library. 
Then somebody needs to decide which version to use, for each implementing type which is contained in both libraries. 
Typetag is implemented in the following way: Mark the type as not deserialize-able.[^FootnoteTypeTagAPIadjustments]
Our manual API has the benefit that the user needs to register each type manually, and gets a `Result` back. Hence errors are a bit more obvious.

<h2>Generics</h2>

The final part for today is concerned with generics.

<h3>Trait generics</h3>

Consider a generic trait:
{% highlight rust %}
#[typetag::serde]
trait Trait<A> {
    ...
}
{% endhighlight %}
Recall from the beginning of the post, that we define
{% highlight rust %}
type DeserializeFn = fn(&mut dyn erased_serde::Deserializer) -> 
    erased_serde::Result<Box<dyn Trait>>;
{% endhighlight %}
which now becomes
{% highlight rust %}
type DeserializeFn<A> = fn(&mut dyn erased_serde::Deserializer) -> 
    erased_serde::Result<Box<dyn Trait<A>>>;
{% endhighlight %}
Can we store this in our registry? We don't know the generic type `A`!

It turns out, this is not an issue. The registry just stores a collection of function pointers. 
Using <b>unsafe</b>, we can forget their type and cast[^FootnoteFunctionCastPortable] them back during <b>deserialization</b> (the point: at deserialization time, the type is fully specified).

This is currently not implemented in typetag.

<h3>Implementor generics</h3>

Consider a generic struct implementing a trait:
{% highlight rust %}
#[typetag::serde]
trait Trait {
    ...
}
struct S<A>{
    ...
}
#[typetag::serde]
impl<A: Constraints> Trait for S<A> {
    ...
}
{% endhighlight %}
Now, the implementation macro needs to register all types satisfying the constraints.
This is not possible for several reasons:
- Rust macros operate on text only and know nothing about traits and implementing types. 
This problem is solvable, for example by a "magic compiler macro", i.e., the compiler explicitly helps and provides a list of implementors.
- The set of possible types might be infinite. Then, our registry gets infinitely large. Dotnet can circumvent this issue by having a Just-In-Time compiler, and just generate the monomorphized code whenever it is actually needed. For Rust, note that <b>rustc</b> needs to generate the deserialization code. If the compiler would give us a list of all types `A: Constraints` for which it created deserialization code for `S<A>` - which has to be a finite set - the issue would be solved.

Again, this situation is not supported in typetag.

Remark: The same issue happens for generic traits and universal implementations:
{% highlight rust %}
#[typetag::serde]
trait Trait<A> {
    ...
}
struct S{
    ...
}
#[typetag::serde]
impl<A: Constraints> Trait<A> for S {
    ...
}
{% endhighlight %}
Again, the macro needs to generate registration code for an unknown set of types, possibly being infinite.

<h2>Summary</h2>

We discussed a basic form of runtime reflection, noting that this should be globally available, i.e., `static`.
In our basic implementation we needed synchronization using a `Mutex`, but we studied the quite sophisticated implementation in typetag, doing all the work in the background.
To wrap up, we discussed generic types and traits


That's it for today. 
Thank you for following the blog series. 

Next time we will proceed with lifetimes.


<h2>Fully working example</h2>

{% highlight rust %}
pub trait Trait: erased_serde::Serialize {
    // Will be generate by macro
    fn type_info(&self) -> &'static str;
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

type DeserializeFn =
    fn(&mut dyn erased_serde::Deserializer) -> erased_serde::Result<Box<dyn Trait>>;
pub struct Registry {
    data: std::sync::Mutex<std::collections::BTreeMap<String, DeserializeFn>>,
}
impl Registry {
    // Create a new registry. Note: This item is private
    const fn new() -> Self {
        Self {
            data: std::sync::Mutex::new(std::collections::BTreeMap::new()),
        }
    }
    /// Fetch a deserializer from the collection. Returns None if the requested type is unknown.
    fn get(&self, type_info: &str) -> Option<DeserializeFn> {
        self.data.lock().unwrap().get(type_info).copied()
    }
    /// Insert a deserializer into the collection.
    /// If there already is one for the requested type, exchange it with the previous one and return the previous one.
    pub fn insert<A>(&self, type_info: String) -> Result<(), DeserializeFn>
    where
        A: Trait,
        A: serde::de::DeserializeOwned,
        A: 'static,
    {
        let deserializer: DeserializeFn = deserialize_fn_generic::<A>;
        match self.data.lock().unwrap().insert(type_info, deserializer) {
            Some(deserializer) => Err(deserializer),
            None => Ok(()),
        }
    }
}

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

pub static REGISTRY_TRAIT: Registry = Registry::new();

impl<'de> serde::Deserialize<'de> for Box<dyn Trait> {
    fn deserialize<D>(deserializer: D) -> Result<Self, D::Error>
    where
        D: serde::Deserializer<'de>,
    {
        let visitor = HelperVisitor {};
        deserializer.deserialize_map(visitor)
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
fn runtime_reflection(type_info: &str) -> Option<DeserializeFn> {
    REGISTRY_TRAIT.get(type_info)
}

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

#[derive(serde::Serialize, serde::Deserialize)]
struct S {
    data: i32,
}
impl Trait for S {
    // Will be generate by macro
    fn type_info(&self) -> &'static str {
        "S"
    }
}

fn main() {
    REGISTRY_TRAIT.insert::<S>("S".into()).unwrap();
    let s = S { data: 42 };
    let t: &dyn Trait = &s;
    let ser = serde_json::to_string(t).unwrap();
    dbg!(&ser);
    let de: Box<dyn Trait> = serde_json::from_str(&ser).unwrap();
}
{% endhighlight %}

<h3>Footnotes</h3>
[^FootNoteTypeTagNotFollowing]: Today, this is actually not quite true
[^FootnoteNewtonsoftOptions]: Of course, this is actually configurable.
[^FootnoteDeserializeMultipleThreatsNoMutableState]: The fact that this simple statement has simple consequences is one of the features I like most about Rust.
[^FootnoteDeriveMacroNotPossible]: Recall that derive macros cannot be attached to a trait, so we will be using an attribute macro instead.
[^FootnoteWishForRust2027]: This is one of my whishes for the 2027 Rust edition: Give me a portable way to build up a registry at compile time.
[^FootnoteRaceCondition]: I never tested this. But this looks dangerous, because there is no synchronization happening. So, you might load a plugin/library while doing some deserialization on another thread. 
[^FootnoteTypeTagAPIadjustments]: Of course, it is possible to store the conflict in the registry, and add some API which allows the user to solve this conflict.
[^FootnoteFunctionCastPortable]: Does somebody know if function pointer casting is sound in Rust for all platform/targets/...?w