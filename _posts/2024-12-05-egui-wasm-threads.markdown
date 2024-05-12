---
title:  "HowTo: Egui with webworkers"
layout: default
date:   2024-05-12 19:21:59 +0100
tags: Rust, egui, wasm
---
<h1>HowTo: Using threads on the web in an egui app</h1>

Currently, I'm porting a desktop app to the web. 
The app gui is written in egui, and compute-heavy tasks use threads both for improved user experience and for increased performance.

But: On the web threads cannot be used - because the API just does not exist.
A partial replacement is called webworkers.

In this short blog post, I adjust the eframe-template app to use webworkers.
You can find the example [here](https://github.com/voelklmichael/eframe_template_wasm).

I learned this from [minno728](https://github.com/minno726/). Thank you!

<h1>The issue</h1>
First we clone the eframe-template app:
{% highlight shell %}
git clone https://github.com/emilk/eframe_template/
{% endhighlight %}
We adjust app.rs to start a thread. From
{% highlight rust %}
  fn update(&mut self, ctx: &egui::Context, _frame: &mut eframe::Frame) {
    // Here we insert our snippet
    egui::TopBottomPanel::top("top_panel").show(ctx, |ui| {    
{% endhighlight %}
to
{% highlight rust %}
  fn update(&mut self, ctx: &egui::Context, _frame: &mut eframe::Frame) {
    log::debug!("before thread");
    let _ = std::thread::spawn(|| log::debug!("message from thread"));
    log::debug!("after thread");           
    egui::TopBottomPanel::top("top_panel").show(ctx, |ui| {    
{% endhighlight %}
This does not work. 
Here is the error message on Firefox and Chrome.

![Thread error message on Firefox]({{ "/assets/egui-wasm-threads/FireFox_Thread_Fail.png" | relative_url }})  

![Thread error message on Chrome]({{ "/assets/egui-wasm-threads/Chrome_Thread_Fail.png" | relative_url }})  

Note that the Chrome error is much more helpful.
Also note, that log statement after the thread.spawn(…) is not reached.

<h1>Solution</h1>
The solution is to replace the thread on Wasm with a webworker.
For this we will use the crate <b>gloo_worker</b>:
{% highlight shell %}
cargo add gloo-worker
{% endhighlight %}
We will need to adjust the index.html file as well as some rust code.

<h1>The Gloo worker</h1>
We add a new rust file, webworker.rs.
{% highlight rust %}
pub struct WebWorker {}    
{% endhighlight %}
Then we implement gloo_worker::Worker, using rust-analyzer:
{% highlight rust %}
impl gloo_worker::Worker for WebWorker {
    type Message;

    type Input;

    type Output;

    fn create(scope: &gloo_worker::WorkerScope<Self>) -> Self {
        todo!()
    }

    fn update(&mut self, scope: &gloo_worker::WorkerScope<Self>, msg: Self::Message) {
        todo!()
    }

    fn received(&mut self, scope: &gloo_worker::WorkerScope<Self>, msg: Self::Input, id: gloo_worker::HandlerId) {
        todo!()
    }
}
{% endhighlight %}
which we fill in with typed dummy data
{% highlight rust %}
#[derive(Debug)]
pub struct Message(pub u32);
#[derive(Debug, serde::Serialize, serde::Deserialize)]
pub struct Input(pub u32);
#[derive(Debug, serde::Serialize, serde::Deserialize)]
pub struct Output(pub u32);

pub struct WebWorker {}
impl gloo_worker::Worker for WebWorker {
    type Message = Message;
    type Input = Input;
    type Output = Output;

    fn create(_scope: &gloo_worker::WorkerScope<Self>) -> Self {
        log::debug!("create");
        Self {}
    }

    fn update(&mut self, _scope: &gloo_worker::WorkerScope<Self>, msg: Self::Message) {
        log::debug!("update {msg:?}");
    }

    fn received(
        &mut self,
        scope: &gloo_worker::WorkerScope<Self>,
        msg: Self::Input,
        _id: gloo_worker::HandlerId,
    ) {
        log::debug!("received {msg:?}");
        scope.respond(_id, Output(msg.0 + 5001));
    }
}
{% endhighlight %}

<h1>Adjusting fn new(…)</h1>
Next we proceed with app.rs.
We start our webworker together with our app, so we adjust <b>fn new(cc : ..)</b>.
If our webworker sends a response, we collect the latest and ask egui to repaint.
Do you see this funny javascript file? We come back to it later.
[^Footnote1]
{% highlight rust %}
pub fn new(cc: &eframe::CreationContext<'_>) -> Self {
    let ctx = cc.egui_ctx.clone();
    let data_update = std::rc::Rc::new(std::cell::Cell::new(None));
    let sender = data_update.clone();
    let bridge = <crate::webworker::WebWorker as gloo_worker::Spawnable>::spawner()
        .callback(move |response| {
            sender.set(Some(response.0));
            ctx.request_repaint();
        })
        .spawn("./dummy_worker.js");
    // …
{% endhighlight %}

We adjust the TemplateApp accordingly, from 
{% highlight rust %}
#[derive(serde::Deserialize, serde::Serialize)]
#[serde(default)] 
pub struct TemplateApp {
    // Example stuff:
    label: String,

    #[serde(skip)] // This how you opt-out of serialization of a field
    value: f32,
}
{% endhighlight %}
to 
{% highlight rust %}
#[derive(serde::Deserialize, serde::Serialize)]
#[serde(default)]
pub struct TemplateApp {
    // Example stuff:
    label: String,

    #[serde(skip)]
    value: f32,
    #[serde(skip)]
    bridge: Option<gloo_worker::WorkerBridge<crate::webworker::WebWorker>>,
    #[serde(skip)]
    data_update: Option<std::rc::Rc<std::cell::Cell<Option<u32>>>>,
}
{% endhighlight %}
and stores the added data in the new function.

Finally, we work with the <b>update</b> method:
We check if there was a request, by adding the following code snippet:
{% highlight rust %}
let data_update = self.data_update.as_mut().unwrap();
if let Some(update) = data_update.take() {
    log::debug!("Received update: {update:?}")
}
{% endhighlight %}

Then, in the button of the egui template, we trigger our webworker
Thus, we exchange 
{% highlight rust %}
if ui.button("Increment").clicked() {
    self.value += 1.0;
}
{% endhighlight %}
with 
{% highlight rust %}
if ui.button("Increment").clicked() {
    self.value += 1.0;
    let msg = self.value.floor().max(0.) as u32;
    self.bridge
        .as_mut()
        .unwrap()
        .send(crate::webworker::Input(msg));
    log::debug!("Message send {msg}");
}
{% endhighlight %}

Now, we finally can try it (and fail ...)
{% highlight shell %}
trunk serve
{% endhighlight %}
Remark: If you reload the egui web app, you might need to unregister the service worker, check about:serviceworkers (Firefox).

The error messages, Chrome and Firefox:

![No javascript error message on Firefox]({{ "/assets/egui-wasm-threads/Firefox_NoJavascript.png" | relative_url }})  

![No javascript error message on Chrome]({{ "/assets/egui-wasm-threads/Chrome_NoJavascript.png" | relative_url }})  

Both errors point to the javascript file, <b>dummy_worker.js</b>, which we used in the spawn function.

Remark: Note that our app does no longer run on the desktop :-/
{% highlight shell %}
cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.31s
     Running `target/debug/eframe_template`
thread 'main' panicked at ~/.cargo/registry/src/index.crates.io-6f17d22bba15001f/js-sys-0.3.66/src/lib.rs:5982:9:
cannot call wasm-bindgen imported functions on non-wasm targets
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
 {% endhighlight %}

<h1>Adding a second binary</h1>
The missing javascript file will automatically be generated by <b>trunk serve</b>.
To this end we add a second binary to our crate[^Footnote2].
{% highlight toml %}
[[bin]]
name = "dummy_worker"
path = "src/dummy_worker.rs"
{% endhighlight %}
We create a new file, dummy_worker.rs:
{% highlight rust %}
use gloo_worker::Registrable;
fn main() {
    eframe_template::webworker::WebWorker::registrar().register();
}
{% endhighlight %}
Finally, we need to adjust index.html, to communicate this change to trunk:
we replace
{% highlight html %}
<link data-trunk rel="rust" data-wasm-opt="2" />
{% endhighlight %}
with 
{% highlight html %}
<link data-trunk rel="rust" data-wasm-opt="2" data-bin="eframe_template" data-type="main" />
<link data-trunk rel="rust" href="Cargo.toml" data-wasm-opt="2" data-bin="dummy_worker" data-type="worker" />
{% endhighlight %}

Now it works :-)
{% highlight shell %}
trunk serve
{% endhighlight %}
Clicking on increment, sends messages:

![Working]({{ "/assets/egui-wasm-threads/Working.png" | relative_url }})  


<h1>Further questions</h1>
1. The logging statement from the WebWorker is not working. I'm unsure why.    
1. The trait gloo_worker::Worker has an associated type "Message". What does it do?
1. Can we start multiple web workers? A dynamic number?
    I plan to test this next.
1. When is the create function of the gloo_worker actually called?
    I've cheked that it is not called during the spawn function of our bridge.
    I tried to debug it, setting a break point inside the create function of the WebWorker struct. 
    But clicking on the button in the GUI still works, even if paused in this breakpoint ...


<h3>Footnotes</h3>
[^Footnote1]: when is the spawn actually executed? -> see below
[^Footnote2]: Crate is wrong, I guess. Package seems to be the correct term.