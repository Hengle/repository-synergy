### Learning Node.js
The sole purpose of this project is to aid in learning Node.js internals.
Please note that viewing this README.md as the main page of the project might
truncate it as it has become quite large. By opening the file (clicking on it) 
you can view the whole content.

## Prerequisites
You'll need to have checked out the node.js source.

### Compiling Node.js with debug enbled:

    $ ./configure --debug
    $ make -C out BUILDTYPE=Debug

After compiling (with debugging enabled) start node using lldb:

    $ cd node/out/Debug
    $ lldb ./node

Node uses Generate Your Projects (gyp) for which I was not familiar with so there is a 
example project in [gyp](./gyp) to look into it.

### Running the Node.js tests

    $ make -j8 test

## Notes
The rest of this page contains notes gathred while setting through the code base:
(These are more sections than listed here but they might be hard to follow)

1. [Background](#background)
2. [Start up](#starting-node)
3. [Loading of builtins](#loading-of-builtins)
4. [Environment](#environment)
5. [TCPWrap](#tcpwrapinitialize)
6. [Running a script](#running-a-script)
7. [Event loop](#event-loop)
8. [setTimeout](#settimeout)
9. [setImmediate](#setimmediate)
10. [nextTick](#process._nexttick)
11. [AsyncWrap](#asyncwrap)
12. [lldb](#lldb)
13. [Promises](#promises)
14. [Libuv Thread pool](#libuv-thread-pool)
15. [bootstrap_node.js compilation and execution walkthrough](#bootstrap_nodejs-compilation-and-execution-walkthrough)

### Background
Node.js is roughly [Google V8](https://github.com/v8/v8), [libuv](https://github.com/libuv/libuv) and Node.js core which glues
everything together.

V8 bascially consists of the memory management of the heap and the execution stack (very simplified but helps
make my point). If you are used to web client side development you'll know about the WebAPIs that are also
available like DOM, AJAX, setTimeout etc. This functionality is not provided by V8 but in instead by chrome.
There is also nothing about a event loop in V8, this is also something that is provided by chrome.

    +------------------------------------------------------------------------------------------+
    | Google Chrome                                                                            |
    |                                                                                          |
    | +----------------------------------------+          +------------------------------+     |
    | | Google V8                              |          |            WebAPIs           |     |
    | | +-------------+ +---------------+      |          |                              |     |
    | | |    Heap     | |     Stack     |      |          |                              |     |
    | | |             | |               |      |          |                              |     |
    | | |             | |               |      |          |                              |     |
    | | |             | |               |      |          |                              |     |
    | | |             | |               |      |          |                              |     |
    | | |             | |               |      |          |                              |     |
    | | +-------------+ +---------------+      |          |                              |     |
    | |                                 |      |          |                              |     |
    | +----------------------------------------+          +------------------------------+     |
    |                                                                                          |
    |                                                                                          |
    | +---------------------+     +---------------------------------------+                    |
    | |     Event loop      |     |          Render task queue            |                    |
    | |                     |     |                                       |                    |
    | +---------------------+     +---------------------------------------+                    |
    |                             +---------------------------------------+                    |
    |                             |          Callback/task queue          |                    |
    |                             |                                       |                    |
    |                             +---------------------------------------+                    |
    |                                                                                          |
    |                                                                                          |
    +------------------------------------------------------------------------------------------+

The execution stack is a stack of frame pointers. For each function called, that function will be pushed onto
the stack. When a function returns it will be removed. If that function calls other functions
they will be pushed onto the stack.
When all functions have returned execution can proceed from the returned to point. If one of the functions performs
an operation that takes time, progress will not be made until it completes as the only way to complete is that the
function returns and is popped off the stack. This is what happens when you have a single threaded programming language.

Aychnronous work can be done by calling into the WebAPIs, for example calling setTimeout which will call out to the
WebAPI and then return. The functionality for setTimeout is provided by the WebAPI and when the timer is due the
WebAPI will push the callback onto the callback queue. Items from the callback queue will be picked up by the event
loop and pushed onto the stack for execution.

TODO: Is the microtask queue considered part of V8 or part of chrome?

Now lets compare this with Node.js:

    +------------------------------------------------------------------------------------------+
    | Node.js                                                                                  |
    |                                                                                          |
    | +----------------------------------------+          +------------------------------+     |
    | | Google V8                              |          |        Node Core APIs        |     |
    | | +-------------+ +---------------+      |          |                              |     |
    | | |    Heap     | |     Stack     |      |          |                              |     |
    | | |             | |               |      |          |                              |     |
    | | |             | |               |      |          |                              |     |
    | | |             | |               |      |          |                              |     |
    | | |             | |               |      |          |                              |     |
    | | |             | |               |      |          |                              |     |
    | | +-------------+ +---------------+      |          |                              |     |
    | |                                 |      |          |                              |     |
    | +----------------------------------------+          +------------------------------+     |
    |                                                                                          |
    |                                                                                          |
    | +---------------------+     +---------------------------------------+                    |
    | | libuv               |     |          Microtask queue              |                    |
    | |     Event Loop      |     |                                       |                    |
    | +---------------------+     +---------------------------------------+                    |
    |                             +---------------------------------------+                    |
    |                             |          Callback queue               |                    |
    |                             |                                       |                    |
    |                             +---------------------------------------+                    |
    |                                                                                          |
    |                                                                                          |
    +------------------------------------------------------------------------------------------+

Taking the same example from above, `setTimeout`, this would be a call to Node Core API and then
the function will return. When the timer expires Node Core API will push the callback onto the
callback queue.
The event loop in Node is provided by libuv, whereas in chrome this is provided by the browser
(chromium I believe)
TODO: Is the microtask queue considered part of V8 or part of node?

### Starting Node
To start and stop at first line in a js program use:

    $ lldb -- node --debug-brk test.js

Set a break point in node_main.cc:

    (lldb) breakpoint set --file node_main.cc --line 52
    (lldb) run

### Walkthrough
`node_main.cc` will bascially call node::Start which we can find in src/node.cc.

#### Start(int argc, char** argv)
Starts by calling PlatformInit

    default_platform = v8::platform::CreateDefaultPlatform(v8_thread_pool_size);
    V8::InitializePlatform(default_platform);
    V8::Initialize();
    ...
    Init(&argc, const_cast<const char**>(argv), &exec_argc, &exec_argv);

#### PlatformInit()
From what I understand this mainly sets up things like signals and file descriptor limits.

#### Init
Init has some libuv code that looks familiar to what I played around with in [learning-libuv](https://github.com/danbev/learning-libuv).

    uv_async_init(uv_default_loop(),
                 &dispatch_debug_messages_async,
                 DispatchDebugMessagesAsyncCallback);

Now I've not used `uv_async_init` but looking at the docs this is done to allow a different thread to wake up the event loop and have the
callback invoked. uv_async_init looks like this:

    int uv_async_init(uv_loop_t* loop, uv_async_t* async, uv_async_cb async_cb)

To understand this better this standalone [example](https://github.com/danbev/learning-libuv/blob/master/thread.c) helped my clarify things a bit.

    uv_unref(reinterpret_cast<uv_handle_t*>(&dispatch_debug_messages_async));

I believe this is done so that the ref count of the dispatch_debug_message_async handle is decremented. If this handle is the only thing 
referened that would cause the event loop to be considered alive and it will continue to iterate.

So a different thread can use uv_async_sent(&dispatch_debug_messages_async) to to wake up the eventloop and have the DispatchDebugMessagesAsyncCallback
function called.


### EnableDebugSignalHandler
This is where we signal the semaphore which will increment the counter, and any threads in the wait queue will now run. So our thread that is
blocked waiting for this debug_semaphore will be able to proceed and TryStartDebugger will be called.

    uv_sem_post(&debug_semaphore);

But what will actually send the signal for all this to happen? 
I think this is done DebugProcess(const FunctionCallbackInfo<Value>& args). Setting a break point confirmed this and the back trace:

	(lldb) bt
	* thread #1: tid = 0x11f57b1, 0x0000000100cafddc node`node::DebugProcess(args=0x00007fff5fbf4700) + 12 at node.cc:3754, queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
	  * frame #0: 0x0000000100cafddc node`node::DebugProcess(args=0x00007fff5fbf4700) + 12 at node.cc:3754
	    frame #1: 0x000000010028618b node`v8::internal::FunctionCallbackArguments::Call(this=0x00007fff5fbf4878, f=(node`node::DebugProcess(v8::FunctionCallbackInfo<v8::Value> const&) at node.cc:3753))(v8::FunctionCallbackInfo<v8::Value> const&)) + 139 at arguments.cc:33
	    frame #2: 0x00000001002f58f3 node`v8::internal::MaybeHandle<v8::internal::Object> v8::internal::(anonymous namespace)::HandleApiCallHelper<false>(isolate=0x0000000104004000, args=BuiltinArguments<v8::internal::BuiltinExtraArguments::kTarget> @ 0x00007fff5fbf49d0)::BuiltinArguments<(v8::internal::BuiltinExtraArguments)1>) + 1619 at builtins.cc:3915
	    frame #3: 0x000000010031ce36 node`v8::internal::Builtin_Impl_HandleApiCall(args=v8::internal::(anonymous namespace)::HandleApiCallArgumentsType @ 0x00007fff5fbf4a38, isolate=0x0000000104004000)::BuiltinArguments<(v8::internal::BuiltinExtraArguments)1>, v8::internal::Isolate*) + 86 at builtins.cc:3939
	    frame #4: 0x00000001002f9c8f node`v8::internal::Builtin_HandleApiCall(args_length=3, args_object=0x00007fff5fbf4b28, isolate=0x0000000104004000) + 143 at builtins.cc:3936
	    frame #5: 0x00003ca66bf0961b
	    frame #6: 0x00003ca66c081e0d
	    frame #7: 0x00003ca66bf0d17a
	    frame #8: 0x00003ca66c01edb3
	    frame #9: 0x00003ca66c01e802
	    frame #10: 0x00003ca66bf0d17a
	    frame #11: 0x00003ca66c081b25
	    frame #12: 0x00003ca66c074fd2
	    frame #13: 0x00003ca66c074c6b
	    frame #14: 0x00003ca66bf38024
	    frame #15: 0x00003ca66bf22962
	    frame #16: 0x00000001006f11df node`v8::internal::(anonymous namespace)::Invoke(isolate=0x0000000104004000, is_construct=false, target=Handle<v8::internal::Object> @ 0x00007fff5fbf4f50, receiver=Handle<v8::internal::Object> @ 0x00007fff5fbf4f48, argc=0, args=0x0000000000000000, new_target=Handle<v8::internal::Object> @ 0x00007fff5fbf4f40) + 607 at execution.cc:97
	    frame #17: 0x00000001006f0f61 node`v8::internal::Execution::Call(isolate=0x0000000104004000, callable=Handle<v8::internal::Object> @ 0x00007fff5fbf50a8, receiver=Handle<v8::internal::Object> @ 0x00007fff5fbf50a0, argc=0, argv=0x0000000000000000) + 1313 at execution.cc:163
	    frame #18: 0x000000010023f4af node`v8::Function::Call(this=0x0000000104062c20, context=(val_ = 0x00000001040404c8), recv=(val_ = 0x00000001040628a0), argc=0, argv=0x0000000000000000) + 671 at api.cc:4404
	    frame #19: 0x000000010023f611 node`v8::Function::Call(this=0x0000000104062c20, recv=(val_ = 0x00000001040628a0), argc=0, argv=0x0000000000000000) + 113 at api.cc:4413
	    frame #20: 0x0000000100c8f3b8 node`node::AsyncWrap::MakeCallback(this=0x0000000104800d50, cb=(val_ = 0x00000001040404a0), argc=3, argv=0x00007fff5fbf5690) + 2600 at async-wrap.cc:284
	    frame #21: 0x0000000100c937e6 node`node::AsyncWrap::MakeCallback(this=0x0000000104800d50, symbol=(val_ = 0x000000010403e5b0), argc=3, argv=0x00007fff5fbf5690) + 198 at async-wrap-inl.h:110
	    frame #22: 0x0000000100d06c67 node`node::StreamBase::EmitData(this=0x0000000104800d50, nread=43, buf=(val_ = 0x0000000104040488), handle=(val_ = 0x0000000000000000)) + 551 at stream_base.cc:427
	    frame #23: 0x0000000100d0adc3 node`node::StreamWrap::OnReadImpl(nread=43, buf=0x00007fff5fbf58f8, pending=UV_UNKNOWN_HANDLE, ctx=0x0000000104800d50) + 675 at stream_wrap.cc:222
	    frame #24: 0x0000000100ca25a7 node`node::StreamResource::OnRead(this=0x0000000104800d50, nread=43, buf=0x00007fff5fbf58f8, pending=UV_UNKNOWN_HANDLE) + 119 at stream_base.h:171
	    frame #25: 0x0000000100d0b93f node`node::StreamWrap::OnReadCommon(handle=0x0000000104800df0, nread=43, buf=0x00007fff5fbf58f8, pending=UV_UNKNOWN_HANDLE) + 351 at stream_wrap.cc:246
	    frame #26: 0x0000000100d0b3d4 node`node::StreamWrap::OnRead(handle=0x0000000104800df0, nread=43, buf=0x00007fff5fbf58f8) + 116 at stream_wrap.cc:261
	    frame #27: 0x0000000100f70e93 node`uv__read(stream=0x0000000104800df0) + 1555 at stream.c:1192
	    frame #28: 0x0000000100f6cb8c node`uv__stream_io(loop=0x00000001019ee200, w=0x0000000104800e78, events=1) + 348 at stream.c:1259
	    frame #29: 0x0000000100f7b784 node`uv__io_poll(loop=0x00000001019ee200, timeout=7073) + 3492 at kqueue.c:276
	    frame #30: 0x0000000100f5e62f node`uv_run(loop=0x00000001019ee200, mode=UV_RUN_ONCE) + 207 at core.c:354
	    frame #31: 0x0000000100cb33a0 node`node::StartNodeInstance(arg=0x00007fff5fbfea60) + 912 at node.cc:4303
	    frame #32: 0x0000000100cb2f8d node`node::Start(argc=2, argv=0x0000000103404a60) + 253 at node.cc:4380
	    frame #33: 0x0000000100cede9b node`main(argc=2, argv=0x00007fff5fbfeb18) + 75 at node_main.cc:54
	    frame #34: 0x0000000100001634 node`start + 52

So to recap, SetupProcessObject sets up the process object for node and one of the methods it sets is '_debugProcess':

     env->SetMethod(process, "_debugProcess", DebugProcess);

SetupProcessObject is called from Environment::Start (src/env.cc):

    * thread #1: tid = 0x1207377, 0x0000000100cad2fc node`node::SetupProcessObject(env=0x00007fff5fbfe108, argc=2, argv=0x0000000103604a20, exec_argc=0, exec_argv=0x0000000103604410) + 11020 at node.cc:3205, queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
     * frame #0: 0x0000000100cad2fc node`node::SetupProcessObject(env=0x00007fff5fbfe108, argc=2, argv=0x0000000103604a20, exec_argc=0, exec_argv=0x0000000103604410) + 11020 at node.cc:3205
       frame #1: 0x0000000100c91bc7 node`node::Environment::Start(this=0x00007fff5fbfe108, argc=2, argv=0x0000000103604a20, exec_argc=0, exec_argv=0x0000000103604410, start_profiler_idle_notifier=false) + 919 at env.cc:91
       frame #2: 0x0000000100cb32b1 node`node::StartNodeInstance(arg=0x00007fff5fbfeaa0) + 673 at node.cc:4274
       frame #3: 0x0000000100cb2f8d node`node::Start(argc=2, argv=0x0000000103604a20) + 253 at node.cc:4380
       frame #4: 0x0000000100cede9b node`main(argc=2, argv=0x00007fff5fbfeb58) + 75 at node_main.cc:54
       frame #5: 0x0000000100001634 node`start + 52


After that detour we are back in the Start method, and the next line is:

    default_platform = v8::platform::CreateDefaultPlatform(v8_thread_pool_size);
    (lldb) p v8_thread_pool_size
    (int) $30 = 4

We can find the implementation of this in `deps/v8/src/libplatform/default-platform.cc`.
The call is the same as was used in the hello_world example except here the size of the thread pool is being passed in 
and in the hello_world the no arguments method was called. I only skimmed this part when I was working through that 
example so it might be good to figure out what is going on here.

An instance of DefaultPlatform is created and then its SetThreadPoolSize method is called with v8_thread_pool_size. When
the size is not given it will default to `p SysInfo::NumberOfProcessors()`. 
Next, EnsureInitialized is called which does a check to see if the instance has already been initilized and if not:

     for (int i = 0; i < thread_pool_size_; ++i)
       thread_pool_.push_back(new WorkerThread(&queue_));

This will create new workers and threads for them. This call finds its way down into deps/v8/src/base/platform/platform-posix.c
and its Thread::Start method:

    LockGuard<Mutex> lock_guard(&data_->thread_creation_mutex_);
    result = pthread_create(&data_->thread_, &attr, ThreadEntry, this);    

We can see this is where the creation and starting of a new thread is done. The first argument is the pthread_t to associate with
the function ThreadEntry which is the top level entry point for the new thread. 
The second argument are additional attributes. The third argument is the function as already mentioned and the fourth parameter is
the argument to the function. So we can see that ThreadEntry takes the current instance as an argument (well it takes a void pointer):

    static void* ThreadEntry(void* arg) {
     Thread* thread = reinterpret_cast<Thread*>(arg);
     // We take the lock here to make sure that pthread_create finished first since
     // we don't know which thread will run first (the original thread or the new
     // one).
     { LockGuard<Mutex> lock_guard(&thread->data()->thread_creation_mutex_); }
     SetThreadName(thread->name());
     DCHECK(thread->data()->thread_ != kNoThread);
     thread->NotifyStartedAndRun();
     return NULL;
   }

ThreadEntry is using a LockGuard and creates a scope to use the Resource Acquisition Is Initialization (RAII) idiom for a mutex. The 
scope is very limited but like the comment says is really just trying to aquire the lock, which was the same that was used when 
creating the thread above.
So, lets take a look at thread->NotifyStartedAndRun()

### NotifyStartedAndRun

    void NotifyStartedAndRun() {
      if (start_semaphore_) start_semaphore_->Signal();
      Run();
   }

### LockGuard
So a lock guard is an implementation of Resource Acquisition Is Initialization (RAII) and takes a mutex in its constructor which it then
calls lock on. When this instance goes out of scope its descructor will be called and it will call unlock guarenteeing that the mutex 
will be unlocked even if an exception is thrown.
The Mutex class can be found in deps/v8/src/base/platform/mutex.h. On a Unix system the mutex will be of type pthread_mutex_t

We can verify this by inspecting the threads before and after 
the calls.
Before:

    (lldb) thread list
    Process 4614 stopped
    * thread #1: tid = 0xe0d19a, 0x0000000100f80cc1 node`v8::base::Thread::Start(this=0x0000000103206110) + 321 at platform-posix.cc:618, queue = 'com.apple.main-thread', stop reason = step over
      thread #2: tid = 0xe0d2f2, 0x00007fff858affae libsystem_kernel.dylib`semaphore_wait_trap + 10

After: 

    (lldb) thread list
    Process 4669 stopped
    * thread #1: tid = 0xe0e3a7, 0x0000000100f80cdb node`v8::base::Thread::Start(this=0x0000000103206530) + 347 at platform-posix.cc:619, queue = 'com.apple.main-thread', stop reason = step over
      thread #2: tid = 0xe0e46d, 0x00007fff858affae libsystem_kernel.dylib`semaphore_wait_trap + 10
      thread #3: tid = 0xe0f230, 0x0000000100f81570 node`v8::base::Thread::data(this=0x0000000103206530) at platform.h:463

What does one of these thread do?
Lets set a breakpoint in the ThreadEntry method:

    (lldb) breakpoint set --file platform-posix.cc --line 582


#### NodeInstanceData
In the Start method we can see a block with the creation of a new NodeInstanceData instance:

    int exit_code = 1;
    {
      NodeInstanceData instance_data(NodeInstanceType::MAIN,
                                     uv_default_loop(),
                                     argc,
                                     const_cast<const char**>(argv),
                                     exec_argc,
                                     exec_argv,
                                     use_debug_agent);
      StartNodeInstance(&instance_data);
      exit_code = instance_data.exit_code();
    }	

There are two NodeInstanceTypes, MAIN and WORKER.
The second argument is the libuv event loop to be used.

#### StartNodeInstance
We are passing the NodeInstanceData instance we created above.
The code in this method is very similar to the code that we used in the [hello-world.cc](https://github.com/danbev/learning-v8/blob/76ec09b60019e893cc23036aa2bc3bdc07c85f77/hello-world.cc#L63-L70).
A new Isolate is created. Remember that an Isolate is an independant copy of the V8 runtime, with its own heap.

    Environment* env = CreateEnvironment(isolate, context, instance_data);


#### CreateEnvironment(isolate, context, instance_data)

    Local<FunctionTemplate> process_template = FunctionTemplate::New(isolate);
    process_template->SetClassName(FIXED_ONE_BYTE_STRING(isolate, "process"));
    ...
    SetupProcessObject(env, argc, argv, exec_argc, exec_argv);

This looks like the node `process` object is being created here. 
All the JavaScript built-in objects are provided by the V8 runtime but the process object is not one of them. So here we are doing 
the same as in the hello-world example above but naming the object 'process'

    auto maybe = process->SetAccessor(env->context(),
                                 env->title_string(),
                                 ProcessTitleGetter,
                                 ProcessTitleSetter,
                                 env->as_external());
    CHECK(maybe.FromJust());

Notice that SetAccessor returns an "optional" MayBe type.

    READONLY_PROPERTY(process,
       "version",
       FIXED_ONE_BYTE_STRING(env->isolate(), NODE_VERSION));

The above is adding properties to the 'process' object. The first being version and then:

    process.moduleLoadList
    process.versions[
      http_parser,
      node,
      v8,
      vu,
      zlib,
      ares,
      icu,
      modules
    ]
    process.icu_data_dir
    process.arch
    process.platform
    process.release
    process.release.name
    process.release.lts
    process.release.sourceUrl
    process.release.headersUrl

    process.env
    process.pid
    process.features



    READONLY_PROPERTY(process,
                     "moduleLoadList",
                     env->module_load_list_array());
I was not aware of this one but process.moduleLoadList will return an array of modules loaded.

     READONLY_PROPERTY(process, "versions", versions);

Next up is process.versions which on my local machine returns:

    > process.versions
    { http_parser: '2.5.2',
      node: '4.4.3',
      v8: '4.5.103.35',
      uv: '1.8.0',
      zlib: '1.2.8',
      ares: '1.10.1-DEV',
      icu: '56.1',
      modules: '46',
      openssl: '1.0.2g' }

After setting up all the object (SetupProcessObject) this methods returns. There is still no sign of the loading of the 'bootstrap_node.js' script. 
This is done in LoadEnvironment.

#### LoadEnvironment

```c++
  Local<String> loaders_name = FIXED_ONE_BYTE_STRING(env->isolate(), "internal/bootstrap/loaders.js");
  Local<Function> loaders_bootstrapper = GetBootstrapper(env, LoadersBootstrapperSource(env), loaders_name);
  Local<String> node_name = FIXED_ONE_BYTE_STRING(env->isolate(), "internal/bootstrap/node.js");
  Local<Function> node_bootstrapper = GetBootstrapper(env, NodeBootstrapperSource(env), node_name);

  ...
  // Create binding loaders
  v8::Local<v8::Function> get_binding_fn = env->NewFunctionTemplate(GetBinding)->GetFunction(env->context()).ToLocalChecked();
  v8::Local<v8::Function> get_linked_binding_fn = env->NewFunctionTemplate(GetLinkedBinding)->GetFunction(env->context()).ToLocalChecked();
  v8::Local<v8::Function> get_internal_binding_fn = env->NewFunctionTemplate(GetInternalBinding)->GetFunction(env->context()).ToLocalChecked();

  Local<Value> loaders_bootstrapper_args[] = {
    env->process_object(),
    get_binding_fn,
    get_linked_binding_fn,
    get_internal_binding_fn
  }; 

  Local<Value> bootstrapped_loaders;
  if (!ExecuteBootstrapper(env, loaders_bootstrapper,
                           arraysize(loaders_bootstrapper_args),
                           loaders_bootstrapper_args,
                           &bootstrapped_loaders)) {
    return;
  } 
```
So ExecuteBootstrapper will call the function in `internal/bootstrap/loaders.js`:
```console
(lldb) jlh bootstrapper
```
I'm not showing the output as it is a little long but you can see the contents of loaders.js.
We can see the arguments that the function takes:
```javascript
- source code: (process, getBinding, getLinkedBinding, getInternalBinding) {
```
These match the arguments from `loaders_bootstrapper_args` above.
Next we have:
```c++
  // Bootstrap Node.js
  Local<Value> bootstrapped_node;
  Local<Value> node_bootstrapper_args[] = {
    env->process_object(),
    bootstrapped_loaders
  };
  if (!ExecuteBootstrapper(env, node_bootstrapper,
                           arraysize(node_bootstrapper_args),
                           node_bootstrapper_args,
                           &bootstrapped_node)) {
    return;
```
Notice that `bootstrapped_loaders` is passed as an argument:
```console
(lldb) jlh bootstrapped_loaders
0x9b2977bc179: [JS_OBJECT_TYPE]
 - map: 0x9b269e9e361 <Map(HOLEY_ELEMENTS)> [FastProperties]
 - prototype: 0x9b27a204479 <Object map = 0x9b269e822a1>
 - elements: 0x9b2e0782251 <FixedArray[0]> [HOLEY_ELEMENTS]
 - properties: 0x9b2e0782251 <FixedArray[0]> {
    #internalBinding: 0x9b2977b3e31 <JSFunction internalBinding (sfi = 0x9b27a2794d9)> (data field 0)
    #NativeModule: 0x9b2977b3369 <JSFunction NativeModule (sfi = 0x9b27a2792f9)> (data field 1)
 }
```
`internalBinding` and `NativeModule` properties are destructed in the function:
```console
- source code: (process, { internalBinding, NativeModule }) {
```

So we have `GetBinding`, `GetLinkedBinding`, and `GetInternalBinding` which are all passed to the bootstrap function.
A native module can be created using using one of the following types ('src/node_internals.h'):
```c++
enum {
  NM_F_BUILTIN  = 1 << 0,
  NM_F_LINKED   = 1 << 1,
  NM_F_INTERNAL = 1 << 2,
};
```
For example, the crypto module (`src/node_crypto.cc`) is registered using the macro:
```c++
NODE_BUILTIN_MODULE_CONTEXT_AWARE(crypto, node::crypto::Initialize)

#define NODE_BUILTIN_MODULE_CONTEXT_AWARE(modname, regfunc)                   \
  NODE_MODULE_CONTEXT_AWARE_CPP(modname, regfunc, nullptr, NM_F_BUILTIN)
```
Modules that use `NF_F_BUILTIN`:
```c++
NODE_BUILTIN_MODULE_CONTEXT_AWARE(inspector, node::inspector::Initialize);
NODE_BUILTIN_MODULE_CONTEXT_AWARE(util, node::util::Initialize)
NODE_BUILTIN_MODULE_CONTEXT_AWARE(tcp_wrap, node::TCPWrap::Initialize)
NODE_BUILTIN_MODULE_CONTEXT_AWARE(url, node::url::Initialize)
NODE_BUILTIN_MODULE_CONTEXT_AWARE(udp_wrap, node::UDPWrap::Initialize)
NODE_BUILTIN_MODULE_CONTEXT_AWARE(inspector, Initialize)
NODE_BUILTIN_MODULE_CONTEXT_AWARE(process_wrap, node::ProcessWrap::Initialize)
NODE_BUILTIN_MODULE_CONTEXT_AWARE(buffer, node::Buffer::Initialize)
NODE_BUILTIN_MODULE_CONTEXT_AWARE(contextify, node::contextify::Initialize)
NODE_BUILTIN_MODULE_CONTEXT_AWARE(os, node::os::Initialize)
NODE_BUILTIN_MODULE_CONTEXT_AWARE(async_wrap, node::AsyncWrap::Initialize)
NODE_BUILTIN_MODULE_CONTEXT_AWARE(fs_event_wrap, node::FSEventWrap::Initialize)
NODE_BUILTIN_MODULE_CONTEXT_AWARE(spawn_sync, node::SyncProcessRunner::Initialize)
NODE_BUILTIN_MODULE_CONTEXT_AWARE(js_stream, node::JSStream::Initialize)
NODE_BUILTIN_MODULE_CONTEXT_AWARE(pipe_wrap, node::PipeWrap::Initialize)
NODE_BUILTIN_MODULE_CONTEXT_AWARE(tty_wrap, node::TTYWrap::Initialize)
NODE_BUILTIN_MODULE_CONTEXT_AWARE(crypto, node::crypto::Initialize)
NODE_BUILTIN_MODULE_CONTEXT_AWARE(tls_wrap, node::TLSWrap::Initialize)
NODE_BUILTIN_MODULE_CONTEXT_AWARE(config, node::Initialize)
NODE_BUILTIN_MODULE_CONTEXT_AWARE(zlib, node::Initialize)
NODE_BUILTIN_MODULE_CONTEXT_AWARE(fs, node::fs::Initialize)
NODE_BUILTIN_MODULE_CONTEXT_AWARE(icu, node::i18n::Initialize)
NODE_BUILTIN_MODULE_CONTEXT_AWARE(cares_wrap, node::cares_wrap::Initialize)
```
There is [work](https://github.com/nodejs/node/issues/22160) in progress to make the above internal.

If a module uses `NODE_MODULE_CONTEXT_AWARE_INTERNAL` it will use `NM_F_INTERNAL which is used by:
```c++
NODE_MODULE_CONTEXT_AWARE_INTERNAL(heap_utils, node::heap::Initialize)
NODE_MODULE_CONTEXT_AWARE_INTERNAL(types, node::InitializeTypes)
NODE_MODULE_CONTEXT_AWARE_INTERNAL(timers, node::Initialize)
NODE_MODULE_CONTEXT_AWARE_INTERNAL(http2, node::http2::Initialize)
NODE_MODULE_CONTEXT_AWARE_INTERNAL(string_decoder, node::InitializeStringDecoder)
NODE_MODULE_CONTEXT_AWARE_INTERNAL(http_parser, node::Initialize)
NODE_MODULE_CONTEXT_AWARE_INTERNAL(performance, node::performance::Initialize)
NODE_MODULE_CONTEXT_AWARE_INTERNAL(uv, node::Initialize)
NODE_MODULE_CONTEXT_AWARE_INTERNAL(messaging, node::worker::InitMessaging)
NODE_MODULE_CONTEXT_AWARE_INTERNAL(trace_events, node::Initialize)
NODE_MODULE_CONTEXT_AWARE_INTERNAL(serdes, node::Initialize)
NODE_MODULE_CONTEXT_AWARE_INTERNAL(v8, node::Initialize)
NODE_MODULE_CONTEXT_AWARE_INTERNAL(stream_pipe, node::InitializeStreamPipe)
NODE_MODULE_CONTEXT_AWARE_INTERNAL(domain, node::domain::Initialize)
NODE_MODULE_CONTEXT_AWARE_INTERNAL(module_wrap, node::loader::ModuleWrap::Initialize)
NODE_MODULE_CONTEXT_AWARE_INTERNAL(worker, node::worker::InitWorker)
NODE_MODULE_CONTEXT_AWARE_INTERNAL(symbols, node::symbols::Initialize)
NODE_MODULE_CONTEXT_AWARE_INTERNAL(signal_wrap, node::SignalWrap::Initialize)
NODE_MODULE_CONTEXT_AWARE_INTERNAL(stream_wrap, node::LibuvStreamWrap::Initialize)
```


So how about `NM_F_LINKED`? 
```
// - process._linkedBinding(): intended to be used by embedders to add
//   additional C++ bindings in their applications. These C++ bindings
//   can be created using NODE_MODULE_CONTEXT_AWARE_CPP() with the flag
```


#### GetBinding
Will be bound to `process.binding`. This function will try to find a builtin module with the passed in module name.

#### GetLinkedBinding
Will be bound to process._linkedBinding.

#### GetInternalBinding
Will be bound to process.internalBinding.




#### lib/internal/bootstrap_node.js
This is the file that is loaded by `LoadEnvironment` as "bootstrap_node.js". 
I read that this file is actually precompiled, where/how?

This file is referenced in node.gyp and is used with the target `node_js2c`. This target calls `tools/js2c.py` which is a tool for converting 
JavaScript source code into C-Style char arrays. This target will process all the library_files specified in the variables section which 
`lib/internal/bootstrap_node.js` is one of. The output of this `out/Debug/obj/gen/node_javascript.h`, depending on the type of build being performed. 
So `lib/internal/bootstrap_node.js` will become `internal_bootstrap_node_value` in `node_javascript.h`. 
This is then later included in `src/node_javascript.cc`. 

We can see the contents of this in lldb using:

    (lldb) p internal_bootstrap_node_value

### Loading of Node Native JavaScript
The JavaScript source files that are located in the lib directory are not loaded in the normal way the JavaScript sources you provide yourself.
Instead these are converted first into c arrays for faster execution. This is done by a build and more specifically a Python tool called `js2c.py`
(JavaScript to C).
If we take a look at node_javascript.h we find that it declares two functions:
```c++
void DefineJavaScript(Environment* env, v8::Local<v8::Object> target);
v8::Local<v8::String> MainSource(Environment* env);
```
Main source is what is used to load `bootstrap_node.js` and `DefineJavaScript` is used in the `Binding` function in src/node.js for loading
the `natives` modules using the `binding` call. For example, in `lib/internal/bootstrap_node.js`:
```javascript
NativeModule._source = process.binding('natives');
```
Lets set a breakpoint in `DefineJavaScript`:
```console
(lldb) br s -n node::DefineJavaScript
```
```c++
CHECK(target->Set(env->context(),
                  internal_bootstrap_node_key.ToStringChecked(env->isolate()),
                  internal_bootstrap_node_value.ToStringChecked(env->isolate())).FromJust());
```
```console
(lldb) jlh internal_bootstrap_node_key.ToStringChecked(env->isolate())
"internal/bootstrap_node"
```
And the value of this will the contents of `boostrap_node.js`. So this will be set as the property on the export object
(the object named target above).

### Loading of builtins
I wanted to know how builtins, like tcp\_wrap and others are loaded. The loading is done explicitely upon initialization by this call in node.cc:
```c++
void Init(int* argc,
          const char** argv,
          int* exec_argc,
          const char*** exec_argv) {
  // Initialize prog_start_time to get relative uptime.
  prog_start_time = static_cast<double>(uv_now(uv_default_loop()));

  // Register built-in modules
  RegisterBuiltinModules();
```
`RegisterBuiltinModules` 
```c++
void RegisterBuiltinModules() {
#define V(modname) _register_##modname();
  NODE_BUILTIN_MODULES(V)
#undef V
}
```
The macro `NODE_BUILTIN_MODULES` can be found in node_internals.h and list all the modules:
```c++
#define NODE_BUILTIN_OPENSSL_MODULES(V) V(crypto) V(tls_wrap)

#define NODE_BUILTIN_ICU_MODULES(V) V(icu)

#define NODE_BUILTIN_STANDARD_MODULES(V)                                      \
    V(async_wrap)                                                             \
    V(buffer)                                                                 \
    V(cares_wrap)                                                             \
    V(config)                                                                 \
    V(contextify)                                                             \
    V(domain)                                                                 \
    V(fs)                                                                     \
    V(fs_event_wrap)                                                          \
    V(http2)                                                                  \
    V(http_parser)                                                            \
    V(inspector)                                                              \
    V(js_stream)                                                              \
    V(module_wrap)                                                            \
    V(os)                                                                     \
    V(performance)                                                            \
    V(pipe_wrap)                                                              \
    V(process_wrap)                                                           \
    V(serdes)                                                                 \
    V(signal_wrap)                                                            \
    V(spawn_sync)                                                             \
    V(stream_wrap)                                                            \
    V(string_decoder)                                                         \
    V(tcp_wrap)                                                               \
    V(timer_wrap)                                                             \
    V(trace_events)                                                           \
    V(tty_wrap)                                                               \
    V(udp_wrap)                                                               \
    V(url)                                                                    \
    V(util)                                                                   \
    V(uv)                                                                     \
    V(v8)                                                                     \
    V(zlib)

#define NODE_BUILTIN_MODULES(V)                                               \
  NODE_BUILTIN_STANDARD_MODULES(V)                                            \
  NODE_BUILTIN_OPENSSL_MODULES(V)                                             \
  NODE_BUILTIN_ICU_MODULES(V)

```
So for example, `tcp_wrap` would have the following function call generated by the preprocesor:
```c++
void RegisterBuiltinModules() {
  _register_tcp_wrap();

```
This will call the `_register_tcp_wrap()` function that is generated by the `NODE_BUILTIN_MODULE_CONTEXT_AWARE` in tcp_wrap.cc. 
Lets take a look at the following line from src/tcp_wrap.cc:
```c++
NODE_BUILTIN_MODULE_CONTEXT_AWARE(tcp_wrap, node::TCPWrap::Initialize)
```

Now, setting a breakpoint on this and printing the thread backtrace gives:

    -> 436 	NODE_BUILTIN_MODULE_CONTEXT_AWARE(tcp_wrap, node::TCPWrap::Initialize)
    (lldb) bt
    * thread #1: tid = 0x18d8053, 0x0000000100d1056b node`_register_tcp_wrap() + 11 at tcp_wrap.cc:436, queue = 'com.apple.main-thread', stop reason = breakpoint 5.1
      * frame #0: 0x0000000100d1056b node`_register_tcp_wrap() + 11 at tcp_wrap.cc:436
        frame #1: 0x00007fff5fc1310b dyld`ImageLoaderMachO::doModInitFunctions(ImageLoader::LinkContext const&) + 265
        frame #2: 0x00007fff5fc13284 dyld`ImageLoaderMachO::doInitialization(ImageLoader::LinkContext const&) + 40
        frame #3: 0x00007fff5fc0f8bd dyld`ImageLoader::recursiveInitialization(ImageLoader::LinkContext const&, unsigned int, ImageLoader::InitializerTimingList&, ImageLoader::UninitedUpwards&) + 305
        frame #4: 0x00007fff5fc0f743 dyld`ImageLoader::processInitializers(ImageLoader::LinkContext const&, unsigned int, ImageLoader::InitializerTimingList&, ImageLoader::UninitedUpwards&) + 127
        frame #5: 0x00007fff5fc0f9b3 dyld`ImageLoader::runInitializers(ImageLoader::LinkContext const&, ImageLoader::InitializerTimingList&) + 75
        frame #6: 0x00007fff5fc020f1 dyld`dyld::initializeMainExecutable() + 208
        frame #7: 0x00007fff5fc05d98 dyld`dyld::_main(macho_header const*, unsigned long, int, char const**, char const**, char const**, unsigned long*) + 3596
        frame #8: 0x00007fff5fc01276 dyld`dyldbootstrap::start(macho_header const*, int, char const**, long, macho_header const*, unsigned long*) + 512
        frame #9: 0x00007fff5fc01036 dyld`_dyld_start + 54

First things to note is that `NODE_BUILTIN_MODULE_CONTEXT_AWARE` is a macro defined in node.h which takes a `modname` and `regfunc` argument. This in turn calls another macro function named `NODE_MODULE_CONTEXT_AWARE_X`.
This macro is invoked with the following arguments:

    NODE_MODULE_CONTEXT_AWARE_X(modname, regfunc, NULL, NM_F_BUILTIN)

We already know that in our case modname is `tcp_wrap` and that `regfunc` is node::TCPWrap::Initialize. 

#### NODE\_MODULE\_CONTEXT\_AWARE\_X

    #define NODE_MODULE_CONTEXT_AWARE_CPP(modname, regfunc, priv, flags)  \
      extern "C" {                                                        \
        static node::node_module _module =                                \
        {                                                                 \
          NODE_MODULE_VERSION,                                            \
          flags,                                                          \
          nullptr,                                                        \
          __FILE__,                                                       \
          nullptr,                                                        \
          (node::addon_context_register_func) (regfunc),                  \
          NODE_STRINGIFY(modname),                                        \
          priv,                                                           \
          nullptr                                                         \
        };                                                                \
        void _register_ ## modname() {                                    \
          node_module_register(&_module)                                  \
        }
    }
    
First, extern "C" means that C linkage should be used and no C++ name mangling should occur. This is saying that everything in the block should have this kind of linkage.
With that out of the way we can focus on the contents of the block.

        static node::node_module _module =                                \
        {                                                                 \
          NODE_MODULE_VERSION,                                            \
          flags,                                                          \
          NULL,                                                           \
          __FILE__,                                                       \
          NULL,                                                           \
          (node::addon_context_register_func) (regfunc),                  \
          NODE_STRINGIFY(modname),                                        \
          priv,                                                           \
          NULL                                                            \
        };                                                                \

We are creating a static variable (it exists for the lifetime of the program, but the name is not visible outside of the block. Remember that we are in tcp_wrap.cc in this walk through so the preprocessor will add a definition of the `_module` to that tcp_wrap.
`node_module` is a struct in node.h and looks like this:

    struct node_module {
      int nm_version;
      unsigned int nm_flags;
      void* nm_dso_handle;
      const char* nm_filename;
      node::addon_register_func nm_register_func;
      node::addon_context_register_func nm_context_register_func;
      const char* nm_modname;
      void* nm_priv;
      struct node_module* nm_link;
    };

### Environment
To create an Environment we need to have an v8::Isolate instance and also an IsolateData instance:

     inline Environment(IsolateData* isolate_data, v8::Local<v8::Context> context);

Such a call can be found during startup: 

    (lldb) bt
      * thread #1: tid = 0x946796, 0x00000001008fa39f node`node::Start(int, char**) + 307 at node.cc:4390, queue = 'com.apple.main-thread', stop reason = step over
        * frame #0: 0x00000001008fa39f node`node::Start(int, char**) + 307 at node.cc:4390
          frame #1: 0x00000001008fa26c node`node::Start(argc=<unavailable>, argv=0x0000000102a00000) + 205 at node.cc:4503
          frame #2: 0x0000000100000b34 node`start + 52

See IsolateData for details about that class and the members that are proxied through via an Environment instance.

An Environment has a number of nested classes:

    AsyncHooks
    AsyncHooksCallbackScope
    DomainFlag
    TickInfo

The above nested classes call the `DISALLOW_COPY_AND_ASSIGN` macro, for example:

    DISALLOW_COPY_AND_ASSIGN(TickInfo);

This macro uses `= delete` for the copy and assignment operator functions:

    #define DISALLOW_COPY_AND_ASSIGN(TypeName) \
    TypeName(const TypeName&) = delete;      \
    void operator=(const TypeName&) = delete

The last nested class is: 

    HandleCleanup

Environment also has a number of static methods:

    static inline Environment* GetCurrent(v8::Isolate* isolate);

This got me wondering, how can we get an Environment from an Isolate, an Isolate is a V8 thing
and an Environment a Node thing?

    inline Environment* Environment::GetCurrent(v8::Isolate* isolate) {
      return GetCurrent(isolate->GetCurrentContext());
    }

So we are going to use the current context to get the Environment pointer, but the context is also a V8 concept, not a node.js concept.

    inline Environment* Environment::GetCurrent(v8::Local<v8::Context> context) {
     return static_cast<Environment*>(context->GetAlignedPointerFromEmbedderData(kContextEmbedderDataIndex));
    }

Alright, now we are getting somewhere. Lets take a closer look at `context->GetAlignedPointerFromEmbedderData(kContextEmbedderDataIndex)`.
We have to look at the Environment constructor to see where this is set (env-inl.h):

    inline Environment::Environment(IsolateData* isolate_data, v8::Local<v8::Context> context) 
    ...
    AssignToContext(context);

So, we can see that `AssignToContext` is setting the environment on the passed-in context:

    static const int kContextEmbedderDataIndex = 5;

    inline void Environment::AssignToContext(v8::Local<v8::Context> context) {
      context->SetAlignedPointerInEmbedderData(kContextEmbedderDataIndex, this);
    }

So this how the Environment is associated with the context, and this enables us to get the environment for a context above. 
The argument to `SetAlignedPointerInEmbedderData` is a void pointer so it can be anything you want. 
The data is stored in a V8 FixedArray, the `kContextEmbedderDataIndex` is the index into this array (I think, still learning here).
TODO: read up on how this FixedArray and alignment works.

There are also static methods to get the Environment using a context.

So an Isolate is like single instance of V8 runtime.
A Context is a separate execution context that does not know about other context.

An Environment is a Node.js concept and multiple environments can exist within a single isolate.
What I'm trying to figure out is how a AtExit callback can be registered with an environment, 
and also how to force that callback to be called when that particular environment is about to
exit. Currently, this is done with a thread-local, but if there are multiple environments per
thread these will overwrite each other.

#### ENVIRONMENT_STRONG_PERSISTENT_PROPERTIES
These are declared in env.h:

    #define ENVIRONMENT_STRONG_PERSISTENT_PROPERTIES(V)                           \
      V(as_external, v8::External)                                                \
      V(async_hooks_destroy_function, v8::Function)                               \
      V(async_hooks_init_function, v8::Function)                                  \
      V(async_hooks_post_function, v8::Function)                                  \
      V(async_hooks_pre_function, v8::Function)                                   \
      V(binding_cache_object, v8::Object)                                         \
      V(buffer_constructor_function, v8::Function)                                \
      V(buffer_prototype_object, v8::Object)                                      \
      V(context, v8::Context)                                                     \
      V(domain_array, v8::Array)                                                  \
      V(domains_stack_array, v8::Array)                                           \
      V(fs_stats_constructor_function, v8::Function)                              \
      V(generic_internal_field_template, v8::ObjectTemplate)                      \
      V(jsstream_constructor_template, v8::FunctionTemplate)                      \
      V(module_load_list_array, v8::Array)                                        \
      V(pipe_constructor_template, v8::FunctionTemplate)                          \
      V(process_object, v8::Object)                                               \
      V(promise_reject_function, v8::Function)                                    \
      V(push_values_to_array_function, v8::Function)                              \
      V(script_context_constructor_template, v8::FunctionTemplate)                \
      V(script_data_constructor_function, v8::Function)                           \
      V(secure_context_constructor_template, v8::FunctionTemplate)                \
      V(tcp_constructor_template, v8::FunctionTemplate)                           \
      V(tick_callback_function, v8::Function)                                     \
      V(tls_wrap_constructor_function, v8::Function)                              \
      V(tls_wrap_constructor_template, v8::FunctionTemplate)                      \
      V(tty_constructor_template, v8::FunctionTemplate)                           \
      V(udp_constructor_function, v8::Function)                                   \
      V(write_wrap_constructor_function, v8::Function)                            \

Notice that `V` is passed in enabling different macros to be passed in. This is used
to create setters/getters like this:

    #define V(PropertyName, TypeName)                                             \
      inline v8::Local<TypeName> PropertyName() const;                            \
      inline void set_ ## PropertyName(v8::Local<TypeName> value);
      ENVIRONMENT_STRONG_PERSISTENT_PROPERTIES(V)
    #undef V

The field itself is private and defined in env.h:

    #define V(PropertyName, TypeName)                                             \
      v8::Persistent<TypeName> PropertyName ## _;
      ENVIRONMENT_STRONG_PERSISTENT_PROPERTIES(V)
    #undef V

The above is defining getters and setter for all the properties in `ENVIRONMENT_STRONG_PERSISTENT_PROPERTIES`. Notice the usage of V and that is is passed into the macro.
Lets take a look at one:

    V(tcp_constructor_template, v8::FunctionTemplate)

Like before these are only the declarations, the definitions can be found in `src/env-inl.h`:

    #define V(PropertyName, TypeName)                                             \
      inline v8::Local<TypeName> Environment::PropertyName() const {              \
        return StrongPersistentToLocal(PropertyName ## _);                        \
      }                                                                           \
      inline void Environment::set_ ## PropertyName(v8::Local<TypeName> value) {  \
        PropertyName ## _.Reset(isolate(), value);                                \
      }
      ENVIRONMENT_STRONG_PERSISTENT_PROPERTIES(V)
    #undef V 

So, in the case of `tcp_constructor_template` this would expand to:

    inline v8::Local<v8::FunctionTemplate> Environment::tcp_constructor_template() const {              
      return StrongPersistentToLocal(tcp_constructor_template_);                        
    }                                                                           
    inline void Environment::set_tcp_constructor_template(v8::Local<v8::FunctionTempalate> value) {  
      tcp_constructor_template_.Reset(isolate(), value);                                
    }

So where is this setter called?   
It is called from `TCPWrap::Initialize`:

    env->set_tcp_constructor_template(t); 

And when is `TCPWrap::Initialize` called?  
From Binding in `node.cc`:

    ...
    mod->nm_context_register_func(exports, unused, env->context(), mod->nm_priv);

Recall (from `Loading of builtins`) how a module is registred:

    NODE_MODULE_CONTEXT_AWARE_BUILTIN(tcp_wrap, node::TCPWrap::Initialize)

The `nm_context_register_func` is `node::TCPWrap::Initialize`, which is a static method declared in src/tcp_wrap.h:

    static void Initialize(v8::Local<v8::Object> target,
                           v8::Local<v8::Value> unused,
                           v8::Local<v8::Context> context);


    wrap_data->MakeCallback(env->onconnection_string(), arraysize(argv), argv);

`env->onconnection_string()` is a simple getter generated by the preprocessor by a macro in `env-inl.h`.


### TCPWrap::Initialize
First thing that happens is that the Environment is retreived using the current context.

Next, a function template is created:

    Local<FunctionTemplate> t = env->NewFunctionTemplate(New);

Just to be clear `New` is the address of the function and we are just passing that to the `NewFunctionTemplate` method. It will use that address when creating a NewFunctionTemplate.

### TcpWrap::New
This class is called TcpWrap because is wraps a libuv [uv_tcp_t](http://docs.libuv.org/en/v1.x/tcp.html) handle. 
    
    static void SetNoDelay(const v8::FunctionCallbackInfo<v8::Value>& args);
    static void SetKeepAlive(const v8::FunctionCallbackInfo<v8::Value>& args);
    static void Bind(const v8::FunctionCallbackInfo<v8::Value>& args);
    static void Bind6(const v8::FunctionCallbackInfo<v8::Value>& args);
    static void Listen(const v8::FunctionCallbackInfo<v8::Value>& args);
    static void Connect(const v8::FunctionCallbackInfo<v8::Value>& args);
    static void Connect6(const v8::FunctionCallbackInfo<v8::Value>& args);
    static void Open(const v8::FunctionCallbackInfo<v8::Value>& args);

Each of the functions above, for example `SetNoDelay` will all wrap a function call in libuv:

    void TCPWrap::SetNoDelay(const FunctionCallbackInfo<Value>& args) {
      TCPWrap* wrap;
      ASSIGN_OR_RETURN_UNWRAP(&wrap,
                              args.Holder(),
                              args.GetReturnValue().Set(UV_EBADF));
      int enable = static_cast<int>(args[0]->BooleanValue());
      int err = uv_tcp_nodelay(&wrap->handle_, enable);
      args.GetReturnValue().Set(err);
    }

When a new instance of this class is created it will initialize the handle which must be of type uv_tcp_t:

    int r = uv_tcp_init(env->event_loop(), &handle_);

Now, a uv_tcp_t could be used for [accepting](https://github.com/danbev/learning-libuv/blob/master/server.c) connection but also for 
[connecting](https://github.com/danbev/learning-libuv/blob/master/client.c) to sockets. 

When this is used in JavaScript is would look like this:

    var TCP = process.binding('tcp_wrap').TCP;
    var handle = new TCP();

When the second line is executed the callback `New` will be invoked. This is set up by this line later in TCPWrap::Initialize:

    target->Set(FIXED_ONE_BYTE_STRING(env->isolate(), "TCP"), t->GetFunction());

New takes a single argument of type v8::FunctionCallbackInfo which holds information about the function call make. 
These are things like the number of arguments used, the arguments can be retreived using with the operator[]. `New` looks like this:

    void TCPWrap::New(const FunctionCallbackInfo<Value>& a) {
      CHECK(a.IsConstructCall());
      Environment* env = Environment::GetCurrent(a);
      TCPWrap* wrap;
      if (a.Length() == 0) {
        wrap = new TCPWrap(env, a.This(), nullptr);
      } else if (a[0]->IsExternal()) {
        void* ptr = a[0].As<External>()->Value();
        wrap = new TCPWrap(env, a.This(), static_cast<AsyncWrap*>(ptr));
      } else {
        UNREACHABLE();
      }
      CHECK(wrap);
    }
Like mentioned above when the constructor of TCPWrap is called it will initialize the uv_tcp_t handle.
Using the example above we can see that `Length` should be 0 as we did not pass any arguments to the TCP function. Just wondering, what could be passed as a parameter?  
What ever it might look like it should be a pointer to an AsyncWrap.

So this is where the instance of TCPWrap is created. Notice `a.This()` which is passed all the way up to BaseObject's constructor and made into a persistent handle.

    const req = new TCPConnectWrap();
    const err = client.connect(req, '127.0.0.1', this.address().port);

Now, new TcpConnectWrap() is setup in TCPWrap::Initalize and the only thing that happens here is that it configured with a constructor that checks that this function is called with the `new` keyword. So there is really nothing else happening at this stage. But, when we call `client.connect` something interesting does happen:
TCPWrap::Connect

    if (err == 0) {
    ConnectWrap* req_wrap = new ConnectWrap(env, req_wrap_obj, AsyncWrap::PROVIDER_TCPCONNECTWRAP);
    err = uv_tcp_connect(req_wrap->req(),
                         &wrap->handle_,
                         reinterpret_cast<const sockaddr*>(&addr),
                         AfterConnect);
    req_wrap->Dispatched();
    if (err)
      delete req_wrap;
  }

So we can see that we are creating a new ConnectWrap instance which extends AsyncWrap and also ReqWrap. Thinking about this makes sense I think. If we recall that the classes with Wrap in them wrap libuv concepts, and in this case we are going to make a tcp connection. If we look at our [client](https://github.com/danbev/learning-libuv/blob/master/client.c) example we can see that we are using uv_connect_t make the connection (named `connection_req`):

    r = uv_tcp_connect(&connect_req,
                       &tcp_client,
                       (const struct sockaddr*) &addr,
                       connect_cb);
    
`tcp_client` in the above example is of type `uv_tcp_t`.
But ConnectWrap also extend AsyncWrap. See the AsyncWrap section for details.
What might be of interest and something to look into a little deeper is that ReqWrap will add the request wrap (wrapping a uv_req_t remember) to the current env req_wrap_queue. Keep in mind that a reqest is shortlived.
The last thing that the ConnectWrap constructor does is call Wrap:

    Wrap(req_wrap_obj, this);

Now, you might not remember what this `req_wrap_obj` is, but it was the first argument to `client.connect` and was the new `TCPConnectWrap` instance. But this was nothing more than a constructor and nothing else:

    (lldb) p req_wrap_obj
    (v8::Local<v8::Object>) $34 = (val_ = 0x00007fff5fbfd018)
    (lldb) p *(*req_wrap_obj)
    (v8::Object) $35 = {}

We can see that this is a v8::Local<v8::Object> and we are going to store the ConnectWrap instance in this object:

    req_wrap_obj->SetAlignedPointerInInternalField(0, this);

So why is this being done?   
Well if you take a look in AfterConnect you can see that this will be accessed and passed as a parameter to the oncomplete function:

    ConnectWrap* req_wrap = static_cast<ConnectWrap*>(req->data);
    ...
    Local<Value> argv[5] = {
      Integer::New(env->isolate(), status),
      wrap->object(),
      req_wrap->object(),
      Boolean::New(env->isolate(), readable),
      Boolean::New(env->isolate(), writable)
    };
    req_wrap->MakeCallback(env->oncomplete_string(), arraysize(argv), argv);
  
This will then invoke the `oncomplete` callback set up on the `req` object:
    
      req.oncomplete = function(status, client_, req_, readable, writable) {
      }
       
### NewFunctionTemplate
NewFunctionTemplate in env.h specifies a default value for the second parameter `v8::Local<v8::Signature>() so it does not have to be specified. 

     v8::Local<v8::External> external = as_external();
     return v8::FunctionTemplate::New(isolate(), callback, external, signature);

    (lldb) p callback
    (v8::FunctionCallback) $0 = 0x0000000100db8540 (node`node::TCPWrap::New(v8::FunctionCallbackInfo<v8::Value> const&) at tcp_wrap.cc:107)

So `t` is a function template, a blueprint for a single function. You create an instance of the template by calling `GetFunction`. Recall that in JavaScript to create a new type of object you use a function. When this function is used as a constructor, using new, the returned object will be an instance of the InstanceTemplate (ObjectTemplate) that will be discussed shortly.

    t->SetClassName(FIXED_ONE_BYTE_STRING(env->isolate(), "TCP"));

The class name is is used for printing objects created with the function created from the
FunctionTemplate as its constructor.

     t->InstanceTemplate()->SetInternalFieldCount(1);

`InstanceTemplate` returns the ObjectTemplate associated with the FunctionTemplate. Every FunctionTemplate has one. Like mentioned before this is the object that is returned after having used the FunctionTemplate as a constructor.
`SetInternalFieldCount(1)` instructs V8 to allocate internal storage for every instance created using this template. Anything can be stored that space allocated, and
for node this is often done using `SetAlignedPointerInInternalField`. This could then be retrieved using `GetAlignedPointerFromInternalField`.

Next, the ObjectTemplate is set up. First a number of properties are configured:

    t->InstanceTemplate()->Set(String::NewFromUtf8(env->isolate(), "reading"),
                               Boolean::New(env->isolate(), false));

Then, a number of prototype methods are set:
 
    env->SetProtoMethod(t, "close", HandleWrap::Close);

Alright, lets take a look at this `SetProtoMethod` method in Environment: 

    inline void Environment::SetProtoMethod(v8::Local<v8::FunctionTemplate> that,
                                         const char* name,
                                         v8::FunctionCallback callback) {
    v8::Local<v8::Signature> signature = v8::Signature::New(isolate(), that);
    v8::Local<v8::FunctionTemplate> t = NewFunctionTemplate(callback, signature);
    // kInternalized strings are created in the old space.
    const v8::NewStringType type = v8::NewStringType::kInternalized;
    v8::Local<v8::String> name_string =
       v8::String::NewFromUtf8(isolate(), name, type).ToLocalChecked();
    that->PrototypeTemplate()->Set(name_string, t);
    t->SetClassName(name_string);  // NODE_SET_PROTOTYPE_METHOD() compatibility.
   }

A `Signature` has the following class documentation: "A Signature specifies which receiver is valid for a function.". So the receiver is set to be `that` which is `t`, our newly created FunctionTemplate.

Next, we are creating a FunctionTemplate for the call back `HandleWrap::Close` with the signature just created.
Then, we will set the function template as a PrototypeTemplate. Again we see `t->SetClassName` which I believe is for when this is printed.
There are few more prototype methods that use HandleWrap callbacks:
    
    env->SetProtoMethod(t, "ref", HandleWrap::Ref);
    env->SetProtoMethod(t, "unref", HandleWrap::Unref);
    env->SetProtoMethod(t, "hasRef", HandleWrap::HasRef);

So have have a class called `HandleWrap`, which I think requires a section of its own.

After this we find the following line:

    StreamWrap::AddMethods(env, t, StreamBase::kFlagHasWritev);

This method is defined in stream_wrap.cc:

    env->SetProtoMethod(target, "setBlocking", SetBlocking);
    StreamBase::AddMethods<StreamWrap>(env, target, flags);

I've been wondering about the class names that end with Wrap and what they are wrapping. My thinking now is that they are wrapping libuv things. 
For instance, take StreamWrap, in libuv src/unix/stream.c which is what SetBlocking calls:

     void StreamWrap::SetBlocking(const FunctionCallbackInfo<Value>& args) {
       StreamWrap* wrap;
       ASSIGN_OR_RETURN_UNWRAP(&wrap, args.Holder());

       CHECK_GT(args.Length(), 0);
       if (!wrap->IsAlive())
         return args.GetReturnValue().Set(UV_EINVAL);

       bool enable = args[0]->IsTrue();
       args.GetReturnValue().Set(uv_stream_set_blocking(wrap->stream(), enable));
    }

Lets take a look at `ASSIGN_OR_RETURN_UNWRAP`: 

    #define ASSIGN_OR_RETURN_UNWRAP(ptr, obj, ...)                                \
      do {                                                                        \
        *ptr =                                                                    \
            Unwrap<typename node::remove_reference<decltype(**ptr)>::type>(obj);  \
        if (*ptr == nullptr)                                                      \
          return __VA_ARGS__;                                                     \
    } while (0)

So what would this look like after the preprocessor has processed it (need to double check this):

    do {
      *wrap = Unwrap<uv_stream_t>(obj);
      if (*wrap == nullptr)
         return;
    } while (0);


What does `__VA_ARGS__` do?   
I've seen this before with variadic methods in c, but not sure what it means to return it. Turns out that if you don't pass anything apart from the required arguments then the `return __VA_ARGS_ statement will just be `return;`. There are other places when the usage of this macro does pass additional arguments, for example: 

    ASSIGN_OR_RETURN_UNWRAP(&wrap,
                           args.Holder(),
                           args.GetReturnValue().Set(UV_EBADF));

    do {
      *wrap = Unwrap<uv_stream_t>(obj);
      if (*wrap == nullptr)
         return args.GetReturnValue.Set(UV_EBADF);
    } while (0);

So we will be returning early with a BADF (bad file descriptor) error.

### BaseObject

    inline BaseObject(Environment* env, v8::Local<v8::Object> handle);

I'm thinking that the handle is the Node representation of a libuv handle. 

    inline v8::Persistent<v8::Object>& persistent();

A persistent handle lives on the heap just like a local handle but it does not correspond
to C++ scopes. You have to explicitly call Persistent::Reset. 


### ReqWrap

    class ReqWrap : public AsyncWrap

cares_wrap.cc has a subclass named `GetAddrInfoReqWrap`
node_file.cc has a subclass named `FSReqWrap`
stream_base.cc has a subclass name `ShutDownWrap`
stream_base.cc has a subclass name `WriteWrap`
udp_wrap.cc has a subclass named `SendWrap` 
connect_wrap.cc has a subclass named 'ConnectWrap` which is subclassed by PipeWrap and TCPWrap.


### AsyncWrap
Some background about AsyncWrap can be found [here](https://github.com/nodejs/diagnostics/blob/master/tracing/AsyncWrap/README.md)
So using AsyncWrap we can have callbacks invoked during the life of handle objects. A handle object would for example be a TCPWrap
which extends ConnectionWrap -> StreamWrap -> HandleWrap.

Being a builtin module it follows the same initialization as others. So lets take a look at the initialization function and see
what kind of functions are made available from JavaScript:

  env->SetMethod(target, "setupHooks", SetupHooks);
  env->SetMethod(target, "pushAsyncIds", PushAsyncIds);
  env->SetMethod(target, "popAsyncIds", PopAsyncIds);
  env->SetMethod(target, "queueDestroyAsyncId", QueueDestroyAsyncId);
  env->SetMethod(target, "enablePromiseHook", EnablePromiseHook);
  env->SetMethod(target, "disablePromiseHook", DisablePromiseHook);
  env->SetMethod(target, "registerDestroyHook", RegisterDestroyHook);

You can confirm this by using:

    $ ./node --expose-internals  -p "require('internal/test/binding').internalBinding('async_wrap')"


    env->set_async_hooks_init_function(init_v.As<Function>());

So, if you are like me you might have gone searching for this `set_async_hooks_init_function` and not finding it. You
might recall this coming up [before](#environment\_strong\_persistent\_properties). So every environment will have such setters and getters for 
  
     V(async_hooks_destroy_function, v8::Function)                               
     V(async_hooks_init_function, v8::Function)                                  
     V(async_hooks_post_function, v8::Function)                                  
     V(async_hooks_pre_function, v8::Function)

So, we are setting a field named async_hooks_init_function_ in the current env.
An example of this usage might be:

    const asyncWrap = process.binding('async_wrap');
    let asyncObject = {
      init: function(uid, provider, parentUid, parentHandle) {
        process._rawDebug('init uid:', uid, ', provider:', provider);
      },
      pre: function(uid) {
        process._rawDebug('pre uid:', uid);
      },
      post: function(uid, didThrow) {
        process._rawDebug('post. uid:', uid, 'didThrow:', didThrow);
      },
      destroy: function(uid) {
        process._rawDebug('destroy: uid:', uid);
      }
    };

There is also a module named `async_hoooks` in lib/async_hook.js that can be used:
```javascript
var ah = require('async_hooks');
let asyncObject = {
  init: function(uid, provider, parentUid, parentHandle) {
    process._rawDebug('init uid:', uid, ', provider:', provider);
  },
  before: function(uid) {
    process._rawDebug('pre uid:', uid);
  },
  after: function(uid, didThrow) {
    process._rawDebug('post. uid:', uid, 'didThrow:', didThrow);
  },
  destroy: function(uid) {
    process._rawDebug('destroy: uid:', uid);
  }
};
let asynchook = ah.createHook(asyncObject);
```
Now, we can create a break point and see when `SetupHooks` is called. 
```console
(lldb) br s -n node::SetupHooks
```
When the startup function in bootstrap_node.js is executed it will call the function `setupProcessFatal`.
```javascript
function setupProcessFatal() {
  const {
    executionAsyncId,
    clearDefaultTriggerAsyncId,
    clearAsyncIdStack,
    hasAsyncIdStack,
    afterHooksExist,
    emitAfter
  } = NativeModule.require('internal/async_hooks');
  ...
```
This will load `internal/async_hooks` module which will call:
```javascript
const async_wrap = process.binding('async_wrap');
...
async_wrap.setupHooks({ init: emitInitNative,
                        before: emitBeforeNative,
                        after: emitAfterNative,
                        destroy: emitDestroyNative,
                        promise_resolve: emitPromiseResolveNative });
```
So this is the first time that setupHooks is called. 
In `SetupHooks` we have the following macro:
```c++
#define SET_HOOK_FN(name)                                                     \
  Local<Value> name##_v = fn_obj->Get(                                        \
      env->context(),                                                         \
      FIXED_ONE_BYTE_STRING(env->isolate(), #name)).ToLocalChecked();         \
  CHECK(name##_v->IsFunction());                                              \
  env->set_async_hooks_##name##_function(name##_v.As<Function>());

  SET_HOOK_FN(init);
  SET_HOOK_FN(before);
  SET_HOOK_FN(after);
  SET_HOOK_FN(destroy);
  SET_HOOK_FN(promise_resolve);
#undef SET_HOOK_FN
```
Lets expand it for the `init` function:
```c++
  Local<Value> init_v = fn_obj->Get(env->context(), FIXED_ONE_BYTE_STRING(env->isolate(), "init")).ToLocalChecked();
  CHECK(init_v->IsFunction());
  env->set_async_hooks_init_function(init_v.As<Function>());
```
After these function have been set we also have the following code in `SetupHooks`:
```c++
  Local<FunctionTemplate> ctor = FunctionTemplate::New(env->isolate()); 
  ctor->SetClassName(FIXED_ONE_BYTE_STRING(env->isolate(), "PromiseWrap"));
  Local<ObjectTemplate> promise_wrap_template = ctor->InstanceTemplate();
  promise_wrap_template->SetInternalFieldCount(PromiseWrap::kInternalFieldCount);  // kInternalFieldCount = 3
  promise_wrap_template->SetAccessor(FIXED_ONE_BYTE_STRING(env->isolate(), "promise"), PromiseWrap::GetPromise);
  promise_wrap_template->SetAccessor(FIXED_ONE_BYTE_STRING(env->isolate(), "isChainedPromise"), PromiseWrap::getIsChainedPromise);
  env->set_promise_wrap_template(promise_wrap_template);
```
`PromiseWrap` is a class defined in async_wrap.cc. So we are setting up a constructor template for a PromiseWrap on the environment. 

```console
(lldb) expr env->async_hooks_init_function()
(v8::Local<v8::Function>) $66 = (val_ = 0x00000001060013c0)
(lldb) jlh env->async_hooks_init_function()
0x329732782401: [Function]
 - map = 0x329757882521 [FastProperties]
 - prototype = 0x3297917043d1
 - elements = 0x3297cd382251 <FixedArray[0]> [HOLEY_ELEMENTS]
 - function prototype =
 - initial_map =
 - shared_info = 0x32979b184ce1 <SharedFunctionInfo emitInitNative>
 - name = 0x32979b1840f9 <String[14]: emitInitNative>
 - formal_parameter_count = 4
 - kind = [ NormalFunction ]
 - context = 0x329732782241 <FixedArray[38]>
 - code = 0x19146b71b241 <Code BUILTIN>
 - source code = (asyncId, type, triggerAsyncId, resource) {
  active_hooks.call_depth += 1;
  // Use a single try/catch for all hook to avoid setting up one per iteration.
  try {
    for (var i = 0; i < active_hooks.array.length; i++) {
      if (typeof active_hooks.array[i][init_symbol] === 'function') {
        active_hooks.array[i][init_symbol](
          asyncId, type, triggerAsyncId,
          resource
        );
      }
    }
  } catch (e) {
    fatalError(e);
  } finally {
    active_hooks.call_depth -= 1;
  }

  // Hooks can only be restored if there have been no recursive hook calls.
  // Also the active hooks do not need to be restored if enable()/disable()
  // weren't called during hook execution, in which case active_hooks.tmp_array
  // will be null.
  if (active_hooks.call_depth === 0 && active_hooks.tmp_array !== null) {
    restoreActiveHooks();
  }
}
 - properties = 0x3297cd382251 <FixedArray[0]> {
    #length: 0x3297cd3b8c81 <AccessorInfo> (const accessor descriptor)
    #name: 0x3297cd3b8c11 <AccessorInfo> (const accessor descriptor)
    #prototype: 0x3297cd3b8cf1 <AccessorInfo> (const accessor descriptor)
 }

 - feedback vector: not available
```
So lets take a closer look at `emitInitNative` in 'lib/internal/async_hooks.js':
```javascript
active_hooks.call_depth += 1;
try {
  for (var i = 0; i < active_hooks.array.length; i++) {
    if (typeof active_hooks.array[i][init_symbol] === 'function') {
      active_hooks.array[i][init_symbol]( asyncId, type, triggerAsyncId, resource);
    }
  }
} catch (e) {
  fatalError(e);
} finally {
  active_hooks.call_depth -= 1;
}
```

When will `env->async_hooks_init_function()` be called?

Each Environment has an AsyncHook as a member (src/env.h):
```c++
AsyncHooks async_hooks_;
```
AsyncHook is a nested class of Environment.

Lets take a look at tcp_wrap.h. TCPWrap extends ConnectionWrap, which extends LibuvStreamWrap, which extends HandleWrap, which
extends AsyncWrap. 

`listen` will eventually call:
```javascript
handle = new TCP(TCPConstants.SERVER);
```
This will call `void TCPWrap::New`:
```c++
  ...
  ProviderType provider;
  switch (type) {
    case SOCKET:
      provider = PROVIDER_TCPWRAP;
      break;
    case SERVER:
      provider = PROVIDER_TCPSERVERWRAP;
      break;
    default:
      UNREACHABLE();
  }

  new TCPWrap(env, args.This(), provider);
```
This constructor call will delegate up to the BaseObject class's constructor and then continue with AsyncWrap's constructor:
```c++
#define NODE_ASYNC_ID_OFFSET 0xA1C
  ...
  // Shift provider value over to prevent id collision.
  persistent().SetWrapperClassId(NODE_ASYNC_ID_OFFSET + provider);
```
Lets take a look SetWrapperClassId more closely. e following will use
`NODE_ASYNC_ID_OFFSET` is 2588 in decimal.
```console
(lldb) expr provider_type_
(const node::AsyncWrap::ProviderType) $3 = PROVIDER_PROMISE
(lldb) expr 2588 + provider_type_
(int) $4 = 2607
```
`persistent()` is a member function of BaseObject which returns the handle for this AsyncWrap instance.
`SetWrapperClassId` is a member function in `PersistentBase<T>`:
```c++
internal::Object** obj = reinterpret_cast<internal::Object**>(this->val_);
uint8_t* addr = reinterpret_cast<uint8_t*>(obj) + I::kNodeClassIdOffset
```
`obj` is a pointer to a pointer which I find hard to visualize but here is an attempt:
```console
(lldb) expr this->val_
(v8::Object *) $76 = 0x000000010604b680
(lldb) expr obj
(v8::internal::Object **) $75 = 0x000000010604b680

lldb) expr &this->val_
(v8::Object **) $84 = 0x0000000104507a28

(lldb) expr &obj
(v8::internal::Object ***) $85 = 0x00007fff5fbfc898

```
So `this->val_` is a pointer to v8::Object and we are using reinterpret_cast to instruct the compiler to treat it 
like it was of type v8::internal::Object**.
```
&this->val
0x0000000104507a28 -----------------------> 0x000000010604b680 -------------> v8::Object
                                                  | 
                                                  | 
&obj                                              |
0x00007fff5fbfc898 -------------------------------+
```
Next, we are going to interpret the obj as an uint8_t* type so that we can perform the addition:
```c++
uint8_t* addr = reinterpret_cast<uint8_t*>(obj) + I::kNodeClassIdOffset
*reinterpret_cast<uint16_t*>(addr) = class_id;
```
And then we set the value that addr is pointing to, to the passed in class it. But that does not really explain why
this is being done. Lets stick a break point in the PromiseWrap constructor:
```console
(lldb) br s -f async_wrap.cc -l 611
```
Now, lets try casting `obj` to
```console
(lldb) expr *reinterpret_cast<v8::internal::GlobalHandles::Node*>(obj)
(v8::internal::GlobalHandles::Node) $96 = {
  object_ = 0x000033d8d5325b49
  class_id_ = 0
  index_ = 'D'
  flags_ = 'a'
  weak_callback_ = 0x0000000000000000
  parameter_or_next_free_ = {
    parameter = 0x0000000000000000
    next_free = 0x0000000000000000
  }
}
```
`v8::internal::GlobalHandles::Node` can be found in `deps/v8/src/global-handles.cc` and we can see that it has
the following private members:
```c++
  Object* object_;
  // Wrapper class ID.
  uint16_t class_id_;
  // Index in the containing handle block.
  uint8_t index_;
```
Notice the order. This is what addr is pointing to in :
```c++
uint8_t* addr = reinterpret_cast<uint8_t*>(obj) + I::kNodeClassIdOffset
*reinterpret_cast<uint16_t*>(addr) = class_id;
```
After having set the value we can again inspect the value:
```console
(lldb) expr *reinterpret_cast<v8::internal::GlobalHandles::Node*>(obj)
(v8::internal::GlobalHandles::Node) $107 = {
  object_ = 0x000033d8d5325b49
  class_id_ = 2607
  index_ = 'D'
  flags_ = 'a'
  weak_callback_ = 0x0000000000000000
  parameter_or_next_free_ = {
    parameter = 0x0000000000000000
    next_free = 0x0000000000000000
  }
}
```
TODO: Figure out how the connection between the Node and the stored object actually works.

So lets take a step back and see how things all fits together. 
```console
(lldb) br s -f async_wrap.cc -l 264
(lldb) br s -f v8.h -l 9082
```
When we create a new PromiseWrap it will delegate calling the above constructors, the last on
is the BaseObject constructor it will invoke .

```c++
template <class T>
T* PersistentBase<T>::New(Isolate* isolate, T* that) {
  if (that == NULL) return NULL;
  internal::Object** p = reinterpret_cast<internal::Object**>(that);
  return reinterpret_cast<T*>(V8::GlobalizeReference(reinterpret_cast<internal::Isolate*>(isolate), p));
}
```
`GlobalizeReference` will then do the following:
```c++
i::Object** V8::GlobalizeReference(i::Isolate* isolate, i::Object** obj) {
  LOG_API(isolate, Persistent, New);
  i::Handle<i::Object> result = isolate->global_handles()->Create(*obj);
  return result.location();
}
```
`GlobalHandles` can be found in `deps/v8/src/global-handles.h`. A global handle is alive until it's `Destroy`
function is called (so it is not cleared when it does out of scope like a local handle with a HandleScope).
```console
(lldb) expr *this
(v8::internal::GlobalHandles) $126 = {
  isolate_ = 0x0000000105807800
  number_of_global_handles_ = 1
  first_block_ = 0x0000000104801800
  first_used_block_ = 0x0000000104801800
  first_free_ = 0x0000000104801820
  new_space_nodes_ = size=1 {
    [0] = 0x0000000104801800
  }
  post_gc_processing_count_ = 0
  number_of_phantom_handle_resets_ = 0
  pending_phantom_callbacks_ = size=0 {}
}
```
GlobalHandles class has a number of private members as we can see above: 
```c++
Isolate* isolate_;
int number_of_global_handles_;
NodeBlock* first_block_;
NodeBlock* first_used_block_;
Node* first_free_;
std::vector<Node*> new_space_nodes_;
int post_gc_processing_count_;
size_t number_of_phantom_handle_resets_;
std::vector<PendingPhantomCallback> pending_phantom_callbacks_;
```
So lets take a closer look at `Create` and see what it does. 
```c++
Node* result = first_free_;
result->Acquire(value);
```
`Acquire` performs the following on the passed on Object:
```c++
  DCHECK(state() == FREE);
  object_ = object;
  class_id_ = v8::HeapProfiler::kPersistentHandleNoClassId;
  set_active(false);
  set_state(NORMAL);
  parameter_or_next_free_.parameter = nullptr;
  weak_callback_ = nullptr;
  IncreaseBlockUses();
```
Now, Aquire is a member function on GlobalHandles::Node and we can see that the object pointer is set, 
and the class_id_ (and others but I'm focusing on these two).

Create then returns:
```c++
return result->handle();
```
Which does:
```c++
Handle<Object> handle() { return Handle<Object>(location()); }
```

v8::Object does not have any members and an Object can be either a Smi or a HeapObject.

`AsyncReset()` is called from AsyncWrap's constructor:
```c++
  // Use AsyncReset() call to execute the init() callbacks.
  AsyncReset(execution_async_id);
```
Note that AsyncWrap has a default value for `execution_async_id` in async_wrap.h:
```c++
double execution_async_id = -1
```
Which is the value of execution_async_id in this call:
```console
(lldb) expr execution_async_id
(double) $25 = -1
```
Also not the `AsyncReset` has default values defined for its parameters:
```c++
void AsyncReset(double execution_async_id = -1, bool silent = false);
```
So lets take a look at `AsyncReset`:
```c++
async_id_ = execution_async_id == -1 ? env()->new_async_id() : execution_async_id;
```
In our case we will get a new async_id (double) from the environment instance:
```console
inline double Environment::new_async_id() {
  async_hooks()->async_id_fields()[AsyncHooks::kAsyncIdCounter] += 1;
  return async_hooks()->async_id_fields()[AsyncHooks::kAsyncIdCounter];
}
```

```console
lldb) expr *async_hooks()
(node::Environment::AsyncHooks) $28 = {
...
async_id_fields_ = {
  isolate_ = 0x0000000105007400
  count_ = 4
  byte_offset_ = 0
  buffer_ = 0x0000000104a1a2e0
  js_array_ = {
    v8::PersistentBase<v8::Float64Array> = (val_ = 0x000000010505c840)
  }
  free_buffer_ = true
}
```

`async_id_fields_` is defined in src/env.h:
```c++
AliasedBuffer<double, v8::Float64Array> async_id_fields_;
```
The types of fields are:
```c++
enum UidFields {
  kExecutionAsyncId,
  kTriggerAsyncId,
  kAsyncIdCounter,
  kDefaultTriggerAsyncId,
  kUidFieldsCount,
};
```
So if we look at the above call again:
```c++
  async_hooks()->async_id_fields()[AsyncHooks::kAsyncIdCounter] += 1;
```
So we obtaining an id for this AsyncWrap resource which remember is of type TCPWrap.
Next we have:
```c++
trigger_async_id_ = env()->get_default_trigger_async_id();
```
The trigger id is an id for what triggered.
Which we can find in src/env-inl.h:
```c++
inline double Environment::get_default_trigger_async_id() {
  double default_trigger_async_id = async_hooks()->async_id_fields()[AsyncHooks::kDefaultTriggerAsyncId];
  // If defaultTriggerAsyncId isn't set, use the executionAsyncId
  if (default_trigger_async_id < 0)
    default_trigger_async_id = execution_async_id();
  return default_trigger_async_id;
}
```
This time we are using kDefaultTriggerAsyncId as the index.
In our case:
```console
(lldb) expr default_trigger_async_id
(double) $36 = -1
```
So `execution_async_id()` will be called which does:
```c++
return async_hooks()->async_id_fields()[AsyncHooks::kExecutionAsyncId];
```
Next in `AsyncReset` we have:
```c++
 switch (provider_type()) {
#define V(PROVIDER)                                                           \
    case PROVIDER_ ## PROVIDER:                                               \
      TRACE_EVENT_NESTABLE_ASYNC_BEGIN2("node.async_hooks",                   \
        #PROVIDER, static_cast<int64_t>(get_async_id()),                      \
        "executionAsyncId",                                                   \
        static_cast<int64_t>(env()->execution_async_id()),                    \
        "triggerAsyncId",                                                     \
        static_cast<int64_t>(get_trigger_async_id()));                        \
      break;
    NODE_ASYNC_PROVIDER_TYPES(V)
#undef V
    default:
      UNREACHABLE();
  }
```
```console
(lldb) expr provider_type()
(node::AsyncWrap::ProviderType) $39 = PROVIDER_TCPSERVERWRAP
```
Lets expand the macro for the provider:
```c++
  case PROVIDER_TCPSERVERWRAP:
    TRACE_EVENT_NESTABLE_ASYNC_BEGIN2("node.async_hooks",                   
      TCPSERVERWRAP, static_cast<int64_t>(get_async_id()),                 
      "executionAsyncId",                                                   
      static_cast<int64_t>(env()->execution_async_id()),                    
      "triggerAsyncId",                                                     
      static_cast<int64_t>(get_trigger_async_id()));                        
    break;
```
This add a v8 tracing event. TODO: take a closer look at this.

Next, we have the following:
```c++
  if (silent) return;

  EmitAsyncInit(env(), object(),
                env()->async_hooks()->provider_string(provider_type()),
                async_id_, trigger_async_id_);
```
`EmitAsyncInit` has the following code:
```c++
Local<Function> init_fn = env->async_hooks_init_function();
```
This is the function that we looked at earlier. We can see the arguments being created:
```c++
Local<Value> argv[] = {
    Number::New(env->isolate(), async_id),
    type,
    Number::New(env->isolate(), trigger_async_id),
    object,
  };
```
And these match the parameters that `init` takes:
```console
init uid: 6 , provider: TCPSERVERWRAP , parentUid: 1 , parentHandle: TCP { reading: false, owner: null, onread: null, onconnection: null }
```
And the function will execute the code generated from `emitInitNative` which will call all the `init` functions that have been registered.
So where/when are the init functions added to the active_hooks.array ?
Well, if we take a look at lib/internal/async_hooks.js we find:
```javascript
const active_hooks = {
  // Array of all AsyncHooks that will be iterated whenever an async event fires.
  array: [],
  call_depth: 0,
  tmp_array: null,
  tmp_fields: null
};
```
`active_hooks` is accessed by calling `setHookArrays()` in lib/async_hooks.js, called in `enable` and `disable`.
Let's take a look at `enable` first:
```javascript
  const [hooks_array, hook_fields] = getHookArrays();

  // Each hook is only allowed to be added once.
  if (hooks_array.includes(this))
    return this;

  const prev_kTotals = hook_fields[kTotals];
  hook_fields[kTotals] = 0;

  // createHook() has already enforced that the callbacks are all functions,
  // so here simply increment the count of whether each callbacks exists or
  // not.
  hook_fields[kTotals] += hook_fields[kInit] += +!!this[init_symbol];
  hook_fields[kTotals] += hook_fields[kBefore] += +!!this[before_symbol];
  hook_fields[kTotals] += hook_fields[kAfter] += +!!this[after_symbol];
  hook_fields[kTotals] += hook_fields[kDestroy] += +!!this[destroy_symbol];
  hook_fields[kTotals] += hook_fields[kPromiseResolve] += +!!this[promise_resolve_symbol];
  hooks_array.push(this);
 
  if (prev_kTotals === 0 && hook_fields[kTotals] > 0) {
      enableHooks();
  }

  return this;
```
hook_fields[kTotals] is first set to 0. Not sure why the first line is doing `+=` as it would
only be adding zero. Could that not just be:
```javascript
  hook_fields[kTotals] = hook_fields[kInit] += +!!this[init_symbol];
```
But lets take a look of the rest of this expression. 
```javascript
  hook_fields[kInit] += +!!this[init_symbol];
```
hook_fields[kInit] is a counter of all the init hooks. This is going to add either 0 or 1 depending
on if there is a function for the `init_symbol`. Notice the `+` unary operator sign before double ! operators.
It will try to convert that boolean to a number. So if an init function was defined this will add 1 to 
hooks_fields[kInit] otherwise 0.
Next, we see that this AsyncHook instance is added to the hooks_array. This answers the above question
regarding here the init fuctions are added to the hooks array.

Following this, lets take a look at enableHooks() which can be found in lib/internal/async_wrap.js:
```javascript
function enableHooks() {
  enablePromiseHook();
  async_hook_fields[kCheck] += 1;
}
```
`enablePromiseHook` can be found in src/async_wrap.cc:
```c++
static void EnablePromiseHook(const FunctionCallbackInfo<Value>& args) {
  Environment* env = Environment::GetCurrent(args);
  env->AddPromiseHook(PromiseHook, static_cast<void*>(env));
}
```
This will land us in `env.cc` `AddPromiseHook`:
```c++
void Environment::AddPromiseHook(promise_hook_func fn, void* arg) {
  auto it = std::find_if(
      promise_hooks_.begin(), promise_hooks_.end(),
      [&](const PromiseHookCallback& hook) {
        return hook.cb_ == fn && hook.arg_ == arg;
      });
  if (it != promise_hooks_.end()) {
    it->enable_count_++;
    return;
  }
  promise_hooks_.push_back(PromiseHookCallback{fn, arg, 1});

  if (promise_hooks_.size() == 1) {
    isolate_->SetPromiseHook(EnvPromiseHook);
  }
}
```
Every environment has a vector of PromiseHookCallbacks (env.h):
```c++
struct PromiseHookCallback {
    promise_hook_func cb_;
    void* arg_;
    size_t enable_count_;
  };
std::vector<PromiseHookCallback> promise_hooks_;
```
Next, remember that `std::find_if` will return an iterator to the first element for which the 
predicate returns true. If nothing is found the function returns last/end.
So if the passed in `promise_hook_func` which is a typedef for a function pointer:
```c++
typedef void (*promise_hook_func) (v8::PromiseHookType type,
                                   v8::Local<v8::Promise> promise,
                                   v8::Local<v8::Value> parent,
                                   void* arg);
```
if the fn and the arg already match an existing PromiseHookCallback, the iterator 
will not be equal to end() and in that case that PromiseHookCallback  will have
it's enable_count incremented and return. If the passed in promise_hook_func and args
have not been added then a new PromiseHookCallback will be created and added to 
`promise_hooks_`.

Last thing that happens in `AddPromiseHook` is `SetPromiseHook` is called on the 
isolate. The function EnvPromiseHook is a promise hook that will run all the 
promise_hook_'s added.

So when will `EnvPromiseHook` be called?
Lets take a closer look at the V8 side of this. If we look in `deps/v8/include/v8.h` we 
can find:
```c++
enum class PromiseHookType { kInit, kResolve, kBefore, kAfter };

typedef void (*PromiseHook)(PromiseHookType type, Local<Promise> promise,
                            Local<Value> parent);
```
So we can see that PromiseHook is a function pointer to a function that takes a type, 
promise, and a parent value.
This callback will be invoked by `Isolate::RunPromiseHook`:
```c++
void Isolate::RunPromiseHook(PromiseHookType type, Handle<JSPromise> promise,
                             Handle<Object> parent) {
  if (debug()->is_active()) debug()->RunPromiseHook(type, promise, parent);
  if (promise_hook_ == nullptr) return;
  promise_hook_(type, v8::Utils::PromiseToLocal(promise), v8::Utils::ToLocal(parent));
}
```
The first time this is called is from `deps/v8/src/runtime/runtime-promise.cc` and:
```c++
RUNTIME_FUNCTION(Runtime_PromiseHookInit) {
  HandleScope scope(isolate);
  DCHECK_EQ(2, args.length());
  CONVERT_ARG_HANDLE_CHECKED(JSPromise, promise, 0);
  CONVERT_ARG_HANDLE_CHECKED(Object, parent, 1);
  isolate->RunPromiseHook(PromiseHookType::kInit, promise, parent);
  return isolate->heap()->undefined_value();
}
```
V8 provides four hooks, `init`, `resolve`, `before`, and `after`.
Init is run when a new Promise is created in V8. `resolve` when a promise is resolved.
`before` is run before a PromiseReactionJob, and `after` after that job. 

Lets say we create the following JavaScript:
```javascript
const p = new Promise((resolve, reject) => {
  resolve('ok');
});
```
If I'm reading this correctly, this would invoke code generated by `v8/src/builtins/builtins-promise-gen.cc`:
```c++
TF_BUILTIN(PromiseConstructor, PromiseBuiltinsAssembler) {
  ...
  GotoIfNot(IsPromiseHookEnabledOrDebugIsActive(), &debug_push);
  CallRuntime(Runtime::kPromiseHookInit, context, instance, UndefinedConstant());
}
```
So, `Runtime_PromiseHookInit` would only be called if a promise hook was enabled. Lets see if we can force this:
```console
(lldb) br s -n ExecuteScript
(lldb) r
(lldb) expr ((v8::internal::Isolate*)env->isolate())->promise_hook_or_debug_is_active_
(bool) $8 = false
(lldb) expr ((v8::internal::Isolate*)env->isolate())->promise_hook_or_debug_is_active_ = true
(bool) $9 = true
(lldb) br s -f runtime-promise.cc -l 113
(lldb) continue
```
Indeed, we can now see that `Runtime_PromiseHookInit` is getting called and it will in turn call
`Isolate::RunPromiseHook`, but in this case promise_hook_ will be a nullptr as it was not set
it will simply return.

When a PromiseHook as been set on the isolate it will call `EnvPromiseHook`
`EnvPromiseHook` will iterate of all the added promise_hook_s in the environment
and call there:
```c++
Environment* env = Environment::GetCurrent(promise->CreationContext());
for (const PromiseHookCallback& hook : env->promise_hooks_) {
  hook.cb_(type, promise, parent, hook.arg_);
}
```
Recall, that the `cb_` in this case it the callback added from `async_wrap.cc` by
`EnablePromiseHook` which was called from `enableHooks()` in `lib/internal/async_wrap.js`:
```c++
env->AddPromiseHook(PromiseHook, static_cast<void*>(env));
```
So, lets take a look a `PromiseHook`. For this we need to update our example to use both async_hooks and
a promise:
```javascript
const http = require('http')
const ah = require('async_hooks');
let asyncObject = {
  init: function(uid, provider, parentUid, parentHandle) {
    process._rawDebug('init uid:', uid, ', provider:', provider, ', parentUid:', parentUid);
  },
  before: function(uid) {
    process._rawDebug('before uid:', uid);
  },
  after: function(uid, didThrow) {
    process._rawDebug('after. uid:', uid, 'didThrow:', didThrow);
  },
  destroy: function(uid) {
    process._rawDebug('destroy: uid:', uid);
  },
  promiseResolve: function(uid) {
    process._rawDebug('promiseResolve: uid:', uid);
  }
};
let asynchook = ah.createHook(asyncObject);
asynchook.enable();

const p = new Promise((resolve, reject) => {
  resolve('ok');
});

p.then(msg => {
  console.log(msg);
});
```
We should now be able to set a breakpoint in `PromiseHook`:
```console
(lldb) br s -f async_wrap.cc -l 288
(lldb) r
```
Just to recap, `Runtime_PromiseHookInit` in `deps/v8/src/runtime/runtime-promise.cc` will
call our `PromiseHook`:
```c++
isolate->RunPromiseHook(PromiseHookType::kInit, promise, parent);
```
And `Isolate::RunPromiseHook` will call the registered hook, which is `EnvPromiseHook`, which
iterates over the registered `PromiseHookCallback` (which is a struct containing the callback,
 the arg and a counter) calling the struct's callback (which is PromiseHook):
```c++
static void PromiseHook(PromiseHookType type, Local<Promise> promise,
                        Local<Value> parent, void* arg) {
  Environment* env = static_cast<Environment*>(arg);
  Local<Value> resource_object_value = promise->GetInternalField(0);
```
```console
(lldb) expr promise
(v8::Local<v8::Promise>) $67 = (val_ = 0x00007fff5fbfce10)
(lldb) expr promise->State()
(v8::Promise::PromiseState) $12 = kPending
(lldb) expr promise->HasHandler()
(bool) $11 = false
```
`v8::Promise` can be found in deps/v8/include/v8.h. A v8::Promise also have a Result() function
which can be called if the state is not pending. And a Catch and Then function to register function
handlers. So this is the promise that was passed up from V8. 
`GetInternalField` is a function of v8::Object. For kInit there will not be any thing in the internal field (at least
not during this debugging session) but later when `PromiseWrap::New` is called:

```c++
wrap = PromiseWrap::New(env, promise, nullptr, silent);
```
So, lets take a closer look at `PromiseWrap::New`:
```c++
  Local<Object> object = env->promise_wrap_template()->NewInstance(env->context()).ToLocalChecked();
  object->SetInternalField(PromiseWrap::kPromiseField, promise);
  object->SetInternalField(PromiseWrap::kIsChainedPromiseField,
                           parent_wrap != nullptr ?
                              v8::True(env->isolate()) :
                              v8::False(env->isolate()));
  CHECK_EQ(promise->GetAlignedPointerFromInternalField(0), nullptr);
  promise->SetInternalField(0, object);
  return new PromiseWrap(env, object, silent);
```
As the name of this class indicates this will wrap a v8::Promise. So we first create
a new instance of the template that was created previously in `SetupHooks`. The wrapping
is done by setting the promise on this object as an internal field (kPromiseField).
Also note that the object is set as an internal field on the promise, as index 0. This is
later accessed in `PromiseHook`
```c++
Local<Value> resource_object_value = promise->GetInternalField(0);
PromiseWrap* wrap = nullptr;
  if (resource_object_value->IsObject()) {
    Local<Object> resource_object = resource_object_value.As<Object>();
    wrap = Unwrap<PromiseWrap>(resource_object);
  }
```
So `resource_object_value` can contain `PromiseWrap` and if so it is unwrapped.

When a parent promise exists a DefaultTriggerAsyncIdScope is used before calling `PromiseWrap::New`:
```c++
  AsyncHooks::DefaultTriggerAsyncIdScope trigger_scope(parent_wrap);
```
The constructor that takes a `AsyncWrap` can be found in `src/async_wrap-inl.h`:
```c++
inline Environment::AsyncHooks::DefaultTriggerAsyncIdScope ::DefaultTriggerAsyncIdScope(AsyncWrap* async_wrap)
    : DefaultTriggerAsyncIdScope(async_wrap->env(),
                                 async_wrap->get_async_id()) {}
```
So this constructor just delegates to the one that takes an Environment pointer and a double. This constructor 
can be found in `src/env-inl.h`.

Recall that this following `SetAccessor` was set on the template:
```c++
  promise_wrap_template->SetAccessor(FIXED_ONE_BYTE_STRING(env->isolate(), "promise"), PromiseWrap::GetPromise);
```
So accessing `promise` on `object` above will invoke `PromiseWrap::GetPromise`:
```c++
  info.GetReturnValue().Set(info.Holder()->GetInternalField(kPromiseField));
```

Next, a new PromiseWrap is created using this object. And since `PromiseWrap` extends AsyncWrap it's constructor will
call `AsyncReset`:
```c++
  AsyncReset(-1, silent);
```

which will call `AsyncWrap::EmitAsyncInit`:
```c++
  Local<Value> argv[] = {
    Number::New(env->isolate(), async_id),
    type,
    Number::New(env->isolate(), trigger_async_id),
    object,
  };
USE(init_fn->Call(env->context(), object, arraysize(argv), argv));
```
These are the arguments that will be passed to `init`:
```console
init uid: 6 , provider: PROMISE , parentUid: 1 , parentHandle: PromiseWrap { isChainedPromise: false, promise: Promise { <pending> } }
```




In lib/internal/async_hooks.js we have these two fields:
```javascript
const async_wrap = process.binding('async_wrap');
const { async_hook_fields, async_id_fields } = async_wrap;
```
When `process.binding('async_wrap')` is called this will invoke `AsyncWrap::Initialize`:
```c++
#define FORCE_SET_TARGET_FIELD(obj, str, field)                               \
  (obj)->DefineOwnProperty(context,                                           \
                           FIXED_ONE_BYTE_STRING(isolate, str),               \
                           field,                                             \
                           ReadOnlyDontDelete).FromJust()

  // Attach the uint32_t[] where each slot contains the count of the number of
  // callbacks waiting to be called on a particular event. It can then be
  // incremented/decremented from JS quickly to communicate to C++ if there are
  // any callbacks waiting to be called.
  FORCE_SET_TARGET_FIELD(target, "async_hook_fields", env->async_hooks()->fields().GetJSArray());
```

This will expand to:
```c++
  target->DefineOwnProperty(context,
                           FIXED_ONE_BYTE_STRING(isolate, "async_hook_fields"),
                           env->async_hooks()->fields().GetJSArray(),
                           ReadOnlyDontDelete).FromJust()
```
Notice that the field here is `env->async_hooks()->fields().GetJSArray()`. If we look at the fields function of 
`AsyncHooks` we see:
```c++
inline AliasedBuffer<uint32_t, v8::Uint32Array>& fields();
```
So the native type
And inspecting it we can see:
```console
(lldb) expr *this
(node::AliasedBuffer<unsigned int, v8::Uint32Array>) $6 = {
  isolate_ = 0x0000000105007400
  count_ = 8
  byte_offset_ = 0
  buffer_ = 0x0000000104a1c590
  js_array_ = {
    v8::PersistentBase<v8::Uint32Array> = (val_ = 0x000000010505ba20)
  }
  free_buffer_ = true
}
```
When an Environment is created the AsyncHooks constructor will be called as it is a member of Environment (src/env.h):
```c++
AsyncHooks async_hooks_;
```
AsyncHooks constructor (src/env-inl.h) will set construct a AliasedBuffer for the the async_id_fields_ member:
```c++
fields_(env()->isolate(), kFieldsCount),
```
In the constructor for AliasedBuffer we can see that a buffer is created:
```c++
buffer_ = Calloc<NativeT>(count)
```
So we are going to allocate zeroed out memory for an unsigned int (buffer_ pointing to it)
```console
(lldb) expr count
(size_t) $11 = 8
(lldb) expr buffer_
(unsigned int *) $13 = 0x0000000106000000
```
Next, we are going to create a V8 ArrayBuffer using the newly allocated block of memory:
```c++
  v8::Local<v8::ArrayBuffer> ab = v8::ArrayBuffer::New(isolate_, buffer_, sizeInBytes);
```
```console
(lldb) jlh ab
0x24f52a506c11: [JSArrayBuffer]
 - map = 0x24f5fbb02c01 [FastProperties]
 - prototype = 0x24f522e89819
 - elements = 0x24f5eb102251 <FixedArray[0]> [HOLEY_ELEMENTS]
 - embedder fields: 2
 - backing_store = 0x106000000
 - byte_length = 32
 - external
 - neuterable
 - properties = 0x24f5eb102251 <FixedArray[0]> {}
 - embedder fields = {
    0x0
    0x0
 }
```
We can see that the `backing_store` is pointing to the memory that was allocated.
An ArrayBuffer is an object that represents a block of data. A view is used to 
access the data, which are called TypedArray views. This is what we create next:
```c++
v8::Local<V8T> js_array = V8T::New(ab, byte_offset_, count);
```
```console
(lldb) expr byte_offset_
(size_t) $31 = 0
(lldb) expr count
(size_t) $32 = 8
(lldb) jlh js_array
0x24f52a506c61: [JSTypedArray]
 - map = 0x24f5fbb03011 [FastProperties]
 - prototype = 0x24f522e8ac51
 - elements = 0x24f52a506ca9 <FixedUint32Array[8]> [UINT32_ELEMENTS]
 - embedder fields: 2
 - buffer = 0x24f52a506c11 <ArrayBuffer map = 0x24f5fbb02c01>
 - byte_offset = 0
 - byte_length = 32
 - length = 8
 - properties = 0x24f5eb102251 <FixedArray[0]> {}
 - elements = 0x24f52a506ca9 <FixedUint32Array[8]> {
         0-7: 0
 }
 - embedder fields = {
    0x0
    0x0
 }
```
Next in AsyncHooks contructor we have the following line:
```c++
fields_[kCheck] = 1;
```
`AliasedBuffer` has overloaded the [] operator so this call will invoke AliasedBuffer::operator[]:
```c++
Reference operator[](size_t index) {
  return Reference(this, index);
}
```
And `Reference` in turn overloads the = operator so it will be invoked:
```c++
template <typename T>
inline Reference& operator=(const T& val) {
  aliased_buffer_->SetValue(index_, val);
  return *this;
}
```
`Reference` is used store the index for the elmenet being used (plus the AliasedBuffer pointer).

So we can see that we are setting index_ which is kCheck, to value which is 1:
```console
(lldb) expr index_
(size_t) $57 = 6
(lldb) expr ::Fields::kCheck
(int) $60 = 6
(lldb) expr val
(const int) $61 = 1
```

So we can get and set values using:
```console
(lldb) expr fields_.SetValue(::Fields::kCheck, 2)
(lldb) expr fields_.GetValue(::Fields::kCheck)
(unsigned int) $69 = 2
(lldb) expr fields_
(node::AliasedBuffer<unsigned int, v8::Uint32Array>) $72 = {
  isolate_ = 0x0000000105007400
  count_ = 8
  byte_offset_ = 0
  buffer_ = 0x0000000106000000
  js_array_ = {
    v8::PersistentBase<v8::Uint32Array> = (val_ = 0x0000000105057a20)
  }
  free_buffer_ = true
}
(lldb) memory read -f x -s 4 -c 7 0x0000000106000000
0x106000000: 0x00000000 0x00000000 0x00000000 0x00000000
0x106000010: 0x00000000 0x00000000 0x00000002
(lldb) expr fields_.SetValue(::Fields::kCheck, 1)
(lldb) memory read -f x -s 4 -c 7 0x0000000106000000
0x106000000: 0x00000000 0x00000000 0x00000000 0x00000000
0x106000010: 0x00000000 0x00000000 0x00000001
```
So we bacially have an 32 bit array which have 8 values in indexed using AsyncHooks::Fields enum.
These are basically counters as far as I can tell. The entries are used to keep track of the number
of init functions that should be called.
Looking at the code we can see that


Getting back on track we were in `AsyncWrap::Initialize` :
```c++
  target->DefineOwnProperty(context,
                           FIXED_ONE_BYTE_STRING(isolate, "async_hook_fields"),
                           env->async_hooks()->fields().GetJSArray(),
                           ReadOnlyDontDelete).FromJust()
```
So looking again at this like we can see that we are retreiving the JSTypedArray and setting that 
on the target as `async_hook_fields`
```console
(lldb) expr env->async_hooks()->fields().GetJSArray()->Buffer()->GetContents()
(v8::ArrayBuffer::Contents) $302 = {
  data_ = 0x0000000106000000
  byte_length_ = 32
  allocation_base_ = 0x0000000106000000
  allocation_length_ = 32
  allocation_mode_ = kNormal
}
```
Notice that the `data_` field points to the same memory location as `buffer_` above.
Next `constants` property is populated.


### AliasedBuffer

```c++
v8::Isolate* isolate_;
  size_t count_;
  size_t byte_offset_;
  NativeT* buffer_;
  v8::Global<V8T> js_array_;
  bool free_buffer_;
```



### HandleWrap
HandleWrap represents a libuv handle which represents . Take the following functions:

    static void Close(const v8::FunctionCallbackInfo<v8::Value>& args);
    static void Ref(const v8::FunctionCallbackInfo<v8::Value>& args);
    static void Unref(const v8::FunctionCallbackInfo<v8::Value>& args);
    static void HasRef(const v8::FunctionCallbackInfo<v8::Value>& args);

There are libuv counter parts for these in uv_handle_t:

    void uv_close(uv_handle_t* handle, uv_close_cb close_cb)
    void uv_ref(uv_handle_t* handle)
    void uv_unref(uv_handle_t* handle)
    int uv_has_ref(const uv_handle_t* handle)

Just like in libuv where uv_handle_t is a base type for all libuv handles, HandleWrap is a base class for all Node.js Wrap classes.

Every uv_handle_t can have a [data member](http://docs.libuv.org/en/v1.x/handle.html#c.uv_handle_t.data), and this is being set in the constructor to this instance of HandleWrap.
    
    handle__->data = this;
    HandleScope scope(env->isolate());
    Wrap(object, this);

In HandleWrap's constructor the HandleWrap is added to the queue of HandleWraps in the Environment:

    env->handle_wrap_queue()->PushBack(this);

libuv has the following types of handle types:

    #define UV_HANDLE_TYPE_MAP(XX)                                               \
     XX(ASYNC, async)                                                            \
     XX(CHECK, check)                                                            \
     XX(FS_EVENT, fs_event)                                                      \
     XX(FS_POLL, fs_poll)                                                        \
     XX(HANDLE, handle)                                                          \
     XX(IDLE, idle)                                                              \
     XX(NAMED_PIPE, pipe)                                                        \
     XX(POLL, poll)                                                              \
     XX(PREPARE, prepare)                                                        \
     XX(PROCESS, process)                                                        \
     XX(STREAM, stream)                                                          \
     XX(TCP, tcp)                                                                \
     XX(TIMER, timer)                                                            \
     XX(TTY, tty)                                                                \
     XX(UDP, udp)                                                                \
     XX(SIGNAL, signal)                                                          \ 


    struct uv_tcp_s {
      UV_HANDLE_FIELDS
      UV_STREAM_FIELDS
      UV_TCP_PRIVATE_FIELDS
    };

We know that TCPWrap is a built-in module and that it's Initialize method is called, which sets up all the prototype functions available, 
among them `listen`:

    env->SetProtoMethod(t, "listen", Listen);

And in `Listen` we find:

    int backlog = args[0]->Int32Value();
    int err = uv_listen(reinterpret_cast<uv_stream_t*>(&wrap->handle_),
                        backlog,
                        OnConnection);
    args.GetReturnValue().Set(err);


We can find a similarity in Node where TCPWrap indirectly also extends StreamWrap (which extends HandleWrap).

### Wrap

    template <typename TypeName>
    void Wrap(v8::Local<v8::Object> object, TypeName* pointer) {
     CHECK_EQ(false, object.IsEmpty());
     CHECK_GT(object->InternalFieldCount(), 0);
     object->SetAlignedPointerInInternalField(0, pointer);
    }

Here we can see that we are setting a pointer in field 0. The `object` in question, and `pointer` the pointer to this HandleWrap.

persistent().Reset will destroy the underlying storage cell if it is non-empty, and create a new one the handle.

MakeWeak:

     inline void MakeWeak(void) {
       persistent().SetWeak(this, WeakCallback, v8::WeakCallbackType::kParameter);
       persistent().MarkIndependent();
    }

The above is installing a finalization callback on the persistent object. Marking the persistent object as independant means that the GC is free to ignore object 
groups containing this persistent object. Why is this done? I don't know enough about the V8 GC yet to answer this.

The callback may be called (best effort) and it looks like this:

    static void WeakCallback(const v8::WeakCallbackInfo<ObjectWrap>& data) {
      ObjectWrap* wrap = data.GetParameter();
      assert(wrap->refs_ == 0);
      wrap->handle_.Reset();
      delete wrap;
    }


ContextifyScript will call `MakeWeak` in it's constructor:
```c++
ContextifyScript(Environment* env, Local<Object> object) : BaseObject(env, object) {
  MakeWeak();
}
```
So we are calling `MakeWeak` so that a callback (a finalizer) will be called when the GC has determined that there are not more 
refs to the object.

If we take a look at `MakeWeak`:
```c++
void BaseObject::MakeWeak() {
  persistent_handle_.SetWeak(
      this,
      [](const v8::WeakCallbackInfo<BaseObject>& data) {
        std::unique_ptr<BaseObject> obj(data.GetParameter());
        // Clear the persistent handle so that ~BaseObject() doesn't attempt
        // to mess with internal fields, since the JS object may have
        // transitioned into an invalid state.
        // Refs: https://github.com/nodejs/node/issues/18897
        obj->persistent_handle_.Reset();
      }, v8::WeakCallbackType::kParameter);
}
```
From looking at the code the callback is called after the 


And the destructor looks like this now:
```c++
BaseObject::~BaseObject() {
  env_->RemoveCleanupHook(DeleteMe, static_cast<void*>(this));

  if (persistent_handle_.IsEmpty()) {
    // This most likely happened because the weak callback below cleared it.
    return;
  }

  {
    v8::HandleScope handle_scope(env_->isolate());
    object()->SetAlignedPointerInInternalField(0, nullptr);
  }
}
```
Notice the call to persitent_handle_.IsEmpty(), so if it is emtpy we will not do anything. So could this callback could be avoided completely?
Making something a weak pointer will allow it to be GC'd but you might not require any callback to be invoked. In that case you can just call:
```c++
  persistent_handle_.SetWeak();
```

Remember, `obj` is of type BaseObject and is not managed by V8, but the persistent_handle_ is managed by V8's GC. When we get the callback that
the persistent_handle_ is about to be freed. If we don't call `Reset` then `~BaseObject` will try to set the internal field on the now
freed `persistent_handle_`. `object()` calls and `SetAlignedPointerInInternalField` will segfault as the handle has already been freed.
```c++
v8::Local<v8::Object> BaseObject::object() const {
  return PersistentToLocal(env_->isolate(), persistent_handle_);
}
```

My understanding of this is that when the callback/finalizer lambda is called the underlying Persistent object will have been freed, but the BaseObject
instance still has a reference to it. It uses this reference in `~ObjectBaset()` create a new Local, and then calls `SetAlignedPointerInInternalField`
which will segfault when trying to OpenHandle (which has been feed). 

So, what does Reset do?

V8::DisposeGlobal(reinterpret_cast<internal::Object**>(this->val_));
i::GlobalHandles::Destroy(location);
global-handles.cc:94


(lldb) expr persistent_handle_
(node::Persistent<v8::Object>) $2 = {
  v8::PersistentBase<v8::Object> = (val_ = 0x0000000108005fa0)
}



### TcpWrap
TcpWrap extends ConnectionWrap
Lets take a look at the creation of a TcpWrap:

    wrap = new TCPWrap(env, args.This(), nullptr);

What is args.This(). That will be the (v8::Local<v8::Object>) object that will be wrapped. 

This be passed to ConnectionWrap's constructor, which in turn will pass it to StreamWrap's constructor, which will pass it to HandleWrap's constructor, which will pass it to AsyncWrap's constructor, which will pass it to BaseObject's constructor which will set this/create a persistent object to store the handle:

    : persistent_handle_(env->isolate(), handle)

I've not seen this before, initializing a member with two parameters, and I cannot find a function that matches this signature. What is going on there?   
Well, the type of `persistent_handle_` is :

    v8::Persistent<v8::Object> persistent_handle_;

And the constructor for Persistent looks like this:

     template <class S>
     V8_INLINE Persistent(Isolate* isolate, Local<S> that)
        : PersistentBase<T>(PersistentBase<T>::New(isolate, *that)) {
      TYPE_CHECK(T, S);
    }


### IsolateData
Has a public constructor that takes a pointer to Isolate, a pointer to uv_loop_t, and a pointer to uint32 zero_fill_field.
An IsolateData instance also has a number of public methods:

    #define VP(PropertyName, StringValue) V(v8::Private, PropertyName, StringValue)
    #define VS(PropertyName, StringValue) V(v8::String, PropertyName, StringValue)
    #define V(TypeName, PropertyName, StringValue)                                \
      inline v8::Local<TypeName> PropertyName(v8::Isolate* isolate) const;
      PER_ISOLATE_PRIVATE_SYMBOL_PROPERTIES(VP)
      PER_ISOLATE_STRING_PROPERTIES(VS)
    #undef V
    #undef VS
    #undef VP

What is happening here is that we are declaring methods for each for the PER_ISOLATE_PRIVATE_SYMBOL_PROPERTIES. Since VP is being 
passed and the type for those methods is v8::Private there will be the following methods:

    v8::Local<Private> alpn_buffer_private_symbol(v8::Isolate* isolate) const;
    v8::Local<Private> arrow_message_private_symbol(v8::Isolate* isolate) const;
    ...
But what is the StringValue used for?  
The StringValue is actually not used here, see [#7905](https://github.com/nodejs/node/pull/7905) for details.

The StringValue is used in the definition though which can be found in src/env-inl.h:

    inline IsolateData::IsolateData(v8::Isolate* isolate, uv_loop_t* event_loop,
                                  uint32_t* zero_fill_field)
      :
    #define V(PropertyName, StringValue)                                          \
      PropertyName ## _(                                                        \
          isolate,                                                              \
          v8::Private::New(                                                     \
              isolate,                                                          \
              v8::String::NewFromOneByte(                                       \
                  isolate,                                                      \
                  reinterpret_cast<const uint8_t*>(StringValue),                \
                  v8::NewStringType::kInternalized,                             \
                  sizeof(StringValue) - 1).ToLocalChecked())),
    PER_ISOLATE_PRIVATE_SYMBOL_PROPERTIES(V)
    #undef V
    #define V(PropertyName, StringValue)                                          \
      PropertyName ## _(                                                        \
          isolate,                                                              \
          v8::String::NewFromOneByte(                                           \
              isolate,                                                          \
              reinterpret_cast<const uint8_t*>(StringValue),                    \
              v8::NewStringType::kInternalized,                                 \
              sizeof(StringValue) - 1).ToLocalChecked()),
      PER_ISOLATE_STRING_PROPERTIES(V)
    #undef V

This is the definition of the IsolateData constructor, and it is setting each of the private member fields
to the StringValue. I created an [example](https://github.com/danbev/learning-cpp11/blob/master/src/fundamentals/macros/macros.cc) to try this out. While it might not be easy on the eyes this does have a major advantage of not having to maintain all of these accessor methods. Adding a new one is simply a matter of adding an entry to the macro.

So now that we understand the macro, lets take a look at the actual information that this class stores/provides.  
All the property accessors defined above are available using he IsolateData instance but also they can be called using an Environment instance which just passes the calls through to the IsolateData instance.
The per isolate private members are the following:

    V(alpn_buffer_private_symbol, "node:alpnBuffer")
    V(npn_buffer_private_symbol, "node:npnBuffer")
    V(selected_npn_buffer_private_symbol, "node:selectedNpnBuffer")

The above are used by node_crypto.cc which makes sense as Application Level Protocol Negotiation (ALPN) is an TLS protocol, as it Next Prototol Negotiation (NPN).

    V(arrow_message_private_symbol, "node:arrowMessage")
Not sure exactly what this does but from a quick search it looks like it has to do with exception handling and printing of error messages. TODO: revisit this later.

An IsolateData (and also an Environement as it proxies these members) actually has a lot of members, too many to list here it is easy to do a search for them.


### Running lint

    $ make lint
    $ make jslint

Run lint os one file:
```console
$ ./tools/node_modules/eslint/bin/eslint.js --rulesdir=tools/eslint-rules --ext=.js,.mjs,.md test/sequential/test-benchmark-tls.js
```


### Running tests
To run the test use the following command:

    $ make -j4 test

The -j is the number of processes to use.

#### Mac firewall exceptions
On mac you might find it popping up dialogs about the firwall blocking access to the `node` and `cctest` applications when running
the tests. There is a script in `node/tools` that can run to add rules to the firewall:

    $ sudo tools/macosx-firewall.sh

### Running a script
This section attempts to explain the process of running a javascript file. We will create a break point in the javascript source and see how it is executed.

    $ ./node -inspect-brk

Next, start `lldb` and 

    $ lldb -- out/Debug/node --inspect-brk test/parallel/test-tcp-wrap-connect.js

Now, when a script is executed it will be read and loaded. Where is this done?
To recap the loading is done by `LoadEnvironment` which loads and executes `lib/internal/bootstrap/node.js`. This is a function which is
then executed:
```c++
    Local<Value> arg = env->process_object();
    f->Call(Null(env->isolate()), 1, &arg);
```

As we can see the process_object which was configured earlier is passed into the function:
```javascript
    (function(process) {
      function startup() {
        ...
      }
      //other functions
      
      startup();
    });
```

We can see that the `startup` function will be called when the the `f` is called. Since we are specifying a script to run we will
be looking at setting up the various object in the environment, mosty using the passed in process object (TODO: need to write out the
details for this later) and eventually running:
```javascript
     preloadModules();
     run(Module.runMain);
```

`Module.runMain` is a function in `lib/module.js`:
```javascript

    // bootstrap main module.
    Module.runMain = function() {
      // Load the main module--the command line argument.
      Module._load(process.argv[1], null, true);
      // Handle any nextTicks added in the first tick of the program
      process._tickCallback();
    };
```

#### _load
Will check the module cache for the filename and if it already exists just returns the exports object for this module. But otherwise
the filename will be loaded using the file extension. Possible extensions are `.js`, `.json`, and `.node` (defaulting to .js if no extension is given).
```javascript
    Module._extensions[extension](this, filename);
```

We know our extension is `.js` so lets look closer at it:
```javascript
     // Native extension for .js
     Module._extensions['.js'] = function(module, filename) {
       var content = fs.readFileSync(filename, 'utf8');
       module._compile(internalModule.stripBOM(content), filename);
     };
```

So lets take a look at `_compile_`

#### module._compile
After removing the shebang from the `content` which is passed in as the first parameter the content is wrapped:
```javascript
    var wrapper = Module.wrap(content);

    var compiledWrapper = vm.runInThisContext(wrapper, {
      filename: filename,
      lineOffset: 0,
      displayErrors: true
    });
```

`vm.runInThisContext` :
```javascript
    var dirname = path.dirname(filename);
    var require = internalModule.makeRequireFunction.call(this);
    var args = [this.exports, require, this, filename, dirname];
    var depth = internalModule.requireDepth;
    if (depth === 0) stat.cache = new Map();
    var result = compiledWrapper.apply(this.exports, args);
```


#### Module.wrap
This is declared as:
```javascript
    const NativeModule = require('native_module');
    ....
    Module.wrap = NativeModule.wrap;
```

NativeModule can be found lib/internal/bootstrap/node.js:
```javascript
     NativeModule.wrap = function(script) {
       return NativeModule.wrapper[0] + script + NativeModule.wrapper[1];
     };

     NativeModule.wrapper = [
       '(function (exports, require, module, __filename, __dirname) { ',
       '\n});'
     ];
```

We can see here that the content of our JavaScript file will be included/wrapped in
```javascript
    (function (exports, require, module, __filename, __dirname) { 
	// script content
    });'
```

So this is also how `exports`, `require`, `module`, `__filename`, and `__dirname` are made available
to all scripts.

So, to recap we have a `wrapper` instance that is a function. The next thing that happens in `lib/modules.js` is:
```javascript
    var compiledWrapper = vm.runInThisContext(wrapper, {
      filename: filename,
      lineOffset: 0,
      displayErrors: true
    });
```

So what does `vm.runInThisContext` do?  
This is defined in `lib/vm.js`:
```javascript
    exports.runInThisContext = function(code, options) {
      var script = new Script(code, options);
      return script.runInThisContext(options);
    };
```

As described in the [vm](https://nodejs.org/api/vm.html) the vm module provides APIs for compiling and running code within V8 Virtual Machine contexts.
Creating a new Script will compile but not run the code. It can later be run multiple times.

So what is a Script? 
It is declared as:
```javascript
    const binding = process.binding('contextify');
    const Script = binding.ContextifyScript;
```

What is Contextify about?  
This is related to V8 contexts and all JavaScript code is run in a context.

`src/node_contextify.cc` is a builtin module and contains an `Init` function that does the following (among other things): 
```c++
    env->SetProtoMethod(script_tmpl, "runInContext", RunInContext);
    env->SetProtoMethod(script_tmpl, "runInThisContext", RunInThisContext);
```

`script.runInThisContext` in vm.js overrides `runInThisContext` and then delegates to src/node_contextify.cc `RunInThisContext`.
```c++
     // Do the eval within this context
     Environment* env = Environment::GetCurrent(args);
     EvalMachine(env, timeout, display_errors, break_on_sigint, args, &try_catch);
```

After all this processing is done we will be back in node.cc and continue processing there. As everything is event driven the event loop start running
and trigger callbacks for anything that has been set up by the script.
Just think about a V8 example you create yourself, you set up the c++ code that is to be called from JavaScript and then V8 takes care of the rest. 
In node, the script is first wrapped in node specific JavaScript and then executed.  Node code uses libuv there are callbacks setup that are called 
by libuv and more actions taken, like invoking a JavaScript callback function.

#### Remove need to specify a no-operation immediate_idle_handle
When calling setImmediate, this will schedule the callback passed in to be scheduled for execution after I/O events:

    setImmediate(function() {
	console.log("In immediate...");
    });

Currently this is done by using a libuv uv_check_handle. Since checks are performed after polling for I/O, if there 
are no idle handle or prepare handle (need to check this) then the I/O polling would block as there would be nothing
for the event loop to process until there is an I/O event. But if we have an idle handler there is something for the 
event loop to do which will cause the poll timeout to be zero and the event loop will not block on I/O.

In src/node.cc there is currently an empty uv_idle_handle callback (IdleImmediateDummy) for this which could be removed 
if it was possible to pass in a NULL callback. Currently there is a check in libuv checcing if the callback is null and this might not be able to change. 

My first idea was to overload the function but C does not suppport overloading, so perhaps having a new function named something like:

     uv_idle_start_nop(&handle)

Another option might be to make uv_idle_start an varargs function and if the only one argument is passed (not null but actually missing) then assume that a nop-callback. But looking into a variadic function there is no way to know when there if an argument was provided or not (of the optional arguments that is). 
Currently I'm only adding a function for uv_idle_start_nop to uv-common.c to try this out and see if I can get some feedback on a better place for this.

This task did not come to anything yet. Perhaps with libuv 2.0 libuv might accepts a null callback.


### tcp\_wrap and pipe\_wrap
Lets take a look at the following statement:
```javascript
    var TCPConnectWrap = process.binding('tcp_wrap').TCPConnectWrap;
    var req = new TCPConnectWrap();
```
    
We know from before that `binding` is set as function on the process object. This was done in SetupProcessObject in node.cc:
```c++
    env->SetMethod(process, "binding", Binding);
```

So we are invoking the Binding function in node.cc with the argument 'tcp_wrap':
```c++
    static void Binding(const FunctionCallbackInfo<Value>& args) {
```

Binding will extract the first (and only) argument which is the name of the module. 
Every environment seems to have a cache, and if the module is in this cache it is returned:
```c++
    Local<Object> cache = env->binding_cache_object();
```

It will also create a instance of Local<Object> exports which is the object that will be returned.
```c++
    Local<Object> exports;
```

So, when the tcp_wrap.cc was Initialized (see section about Builtins):
```c++
    // Create FunctionTemplate for TCPConnectWrap.
    auto constructor = [](const FunctionCallbackInfo<Value>& args) {
      CHECK(args.IsConstructCall());
    };
    auto cwt = FunctionTemplate::New(env->isolate(), constructor);
    cwt->InstanceTemplate()->SetInternalFieldCount(1);
    cwt->SetClassName(FIXED_ONE_BYTE_STRING(env->isolate(), "TCPConnectWrap"));
    target->Set(FIXED_ONE_BYTE_STRING(env->isolate(), "TCPConnectWrap"), cwt->GetFunction());
```

What is going on here. We create a new FunctionTemplate using the `constructor` lamba, this is then added to the target (the object that we are initializing).
The constructor is only checking that the passed in args can be used as a constructor (using new in JavaScript)  
The object returned from the constructor call does not have any methods as far as I can tell. 
The constructor would late be used like this:
```javascript
    var client = new TCP();
    var req = new TCPConnectWrap();
    var err = client.connect(req, '127.0.0.1', this.address().port);
```

Now, we saw that TCP has a bunch of methods set up in Initialize, one of the being connect:
```c++
    void TCPWrap::Connect(const FunctionCallbackInfo<Value>& args) {
      ...
      Local<Object> req_wrap_obj = args[0].As<Object>();
```

This is the instance of TCPConnectWrap `req` created above and we can see that it is of type `v8::Local<v8::Local>`.
```c++
    ConnectWrap* req_wrap = new ConnectWrap(env, req_wrap_obj, AsyncWrap::PROVIDER_TCPCONNECTWRAP);
```

Remember that `ConnectWrap` extends `ReqWrap` which extends `AsyncWrap`.

We know that ConnectWrap takes `Local<Object>` as the `req_wrap_obj`
```c++
    err = uv_tcp_connect(req_wrap->req(), &wrap->handle_, reinterpret_cast<const sockaddr*>(&addr), AfterConnect);
```

`uv_tcp_connect` takes a pointer `uv_connect_t` and a pointer to `uv_tcp_t` handle. This will connect to the specified `sockaddr_in` and the
callback will be called when the connection has been established or if an error occurs. So it makes sense that ConnectWrap extends ReqWrap
as uv_connect_t is a request type in libuv:
```c
    /* Request types. */
    typedef struct uv_req_s uv_req_t;
    typedef struct uv_getaddrinfo_s uv_getaddrinfo_t;
    typedef struct uv_getnameinfo_s uv_getnameinfo_t;
    typedef struct uv_shutdown_s uv_shutdown_t;
    typedef struct uv_write_s uv_write_t;
    typedef struct uv_connect_s uv_connect_t;         <--------------------
    typedef struct uv_udp_send_s uv_udp_send_t;
    typedef struct uv_fs_s uv_fs_t;
    typedef struct uv_work_s uv_work_t;
```

AsyncWrap extends BaseObject which 
```console
    (lldb) p *this
    (node::BaseObject) $28 = {
      persistent_handle_ = {
        v8::PersistentBase<v8::Object> = (val_ = 0x0000000105010c60)
      }
      env_ = 0x00007fff5fbfe108
    }
```

So each BaseObject instance has a v8::Persistent<v8:Object>. This is a persistent object as it needs to be preserved accross C++ function boundries.
Also, we can see that each BaseObject instance also has a node::Environment associated with it.
The only thing that BaseObject's constructor does (baseobject-inl-h) is :
```c++
    // The zero field holds a pointer to the handle. Immediately set it to
    // nullptr in case it's accessed by the user before construction is complete.
    if (handle->InternalFieldCount() > 0)
      handle->SetAlignedPointerInInternalField(0, nullptr);
```

So after we have returned to AsyncWraps constructor, and then ReqWrap's we are back in ConnectWrap's constructor:
```c++
    Wrap(req_wrap_obj, this);
```

`Wrap` in util-inl.h:
```c++
    template <typename TypeName>
    void Wrap(v8::Local<v8::Object> object, TypeName* pointer) {
      CHECK_EQ(false, object.IsEmpty());
      CHECK_GT(object->InternalFieldCount(), 0);
      object->SetAlignedPointerInInternalField(0, pointer);
    }
```

We are now setting index 0 to the pointer which is the current 
Object is the `v8::Local<v8::Object>`, the one we created in our JavaScript file and passed to the connect method named `req`:
```javascript
    var req = new TCPConnectWrap();
    var err = client.connect(req, '127.0.0.1', this.address().port);
```

So we are setting/storing a pointer to the ConnectWrap instance at index 0 of the `req_wrap_obj`.

After all that we are ready to make the uv_tcp_connect call:
```c++
    err = uv_tcp_connect(req_wrap->req(), &wrap->handle_, reinterpret_cast<const sockaddr*>(&addr), AfterConnect);
```

We can see the callback is node::ConnectionWrap<node::TCPWrap, uv_tcp_s>::AfterConnect(uv_connect_s*, int)


Notice that target is of type Local<Object>. 
```c++
    Local<Object> exports;
    ....
    exports = Object::New(env->isolate());
    ...
    mod->nm_context_register_func(exports, unused, env->context(), mod->nm_priv);
```

`exports` is what is returned to the caller. 
```c++
    args.GetReturnValue().Set(exports);
```

And we access the TCPConnectWrap member, which is a function which can be used as 
a constructor by using new. Lets start with where is ConnectWrap called?
It is called from tcp_wrap.cc and its Connect method.
ConnectWrap extends ReqWrap which extends AsyncWrap which extens BaseObject
```javascript
    req.oncomplete = function(status, client_, req_) {
```

So, we know from earlier that our `req` object is basically empty. Here we are setting a property name `oncomplete` to be
a function. This will be called in connection_wrap.cc 111:
```c++
    req_wrap->MakeCallback(env->oncomplete_string(), arraysize(argv), argv);
```

oncomplete_string() is a generated method from a macro in env.h
```c++
    v8::Local<v8::Value> cb_v = object()->Get(symbol);
    CHECK(cb_v->IsFunction());
    return MakeCallback(cb_v.As<v8::Function>(), argc, argv);
```

`object()` will return the persistent object to out handle (from base-object-inl.h) :
```c++
    return PersistentToLocal(env_->isolate(), persistent_handle_);
```

We can see that the `persistent_handle_` is the handle that was created using which makes sense as this 
is the object that oncomplete was created for:
```c++
    var req = new TCPConnectWrap();
```

We are then calling Get(symbol) which will be a Symbol representing 'oncomplete'. And the calling it with number of arguments, and the
arguments themselves.


### tcp\_wrap.cc
In `OnConnect` I found the following:
```c++
    TCPWrap* tcp_wrap = static_cast<TCPWrap*>(handle->data);
    ....
    Local<Object> client_obj = Instantiate(env, static_cast<AsyncWrap*>(tcp_wrap));
```

```
class TCPWrap : public StreamWrap
class StreamWrap : public HandleWrap, public StreamBase
class HandleWrap : public AsyncWrap {
```
As far as I can tell `TCPWrap` is of type `AsyncWrap`. Looking at `src/pipe_wrap.cc` which has a very similar OnConnect method
(which I'm going to take a stab at refactoring) but does not have this cast.


### Refactoring tcpwrap and pipewrap
This comment exist on pipewrap OnConnect:
```c++
// TODO(bnoordhuis) maybe share with TCPWrap?
```

```c++
    void PipeWrap::OnConnection(uv_stream_t* handle, int status) {
    PipeWrap* pipe_wrap = static_cast<PipeWrap*>(handle->data);
    CHECK_EQ(&pipe_wrap->handle_, reinterpret_cast<uv_pipe_t*>(handle));
```

The reinterpret_cast operator changes one data type into another. Recall how the types of libuv have a type of c inheritance allowing 
casting.
```c
    /*
     * uv_pipe_t is a subclass of uv_stream_t.
     *
     * Representing a pipe stream or pipe server. On Windows this is a Named
     * Pipe. On Unix this is a Unix domain socket.
     */
    struct uv_pipe_s {
      UV_HANDLE_FIELDS
      UV_STREAM_FIELDS
      int ipc; /* non-zero if this pipe is used for passing handles */
      UV_PIPE_PRIVATE_FIELDS
   };
```

The main difference that I've been able to find is in pipewrap `status` is checked:
```c++
    if (status != 0) {
      pipe_wrap->MakeCallback(env->onconnection_string(), arraysize(argv), argv);
      return;
   } 
```

### src/stream_wrap.cc
Looking into a task where the public member field req_ in src/req_wrap.cc is to be made private, I came accross the 
following method:
```console
    286 void StreamWrap::AfterShutdown(uv_shutdown_t* req, int status) {
    287   ShutdownWrap* req_wrap = ContainerOf(&ShutdownWrap::req_, req);
    288   HandleScope scope(req_wrap->env()->isolate());
    289   Context::Scope context_scope(req_wrap->env()->context());
    290   req_wrap->Done(status);
    291 }
```

What I did for the public req_ member is made it private and then added a public accessor method for it. This was easy
to update in most places but in src/stream_wrap.cc we have the following line:
```c++
   ShutdownWrap* req_wrap = ContainerOf(&ShutdownWrap::req_, req);
```

## Extracting AfterConnect into connection_wrap.cc
Just like `OnConnect` was extracted into connection_wrap and shared by both tcp_wrap and pipe_wrap the same should be done for `AfterConnect`.

The main difference I found was in `PipeWrap::AfterConnect`:
```c++
    bool readable, writable;

    if (status) {
      readable = writable = 0;
    } else {
      readable = uv_is_readable(req->handle) != 0;
      writable = uv_is_writable(req->handle) != 0;
    } 
    Local<Object> req_wrap_obj = req_wrap->object();
    Local<Value> argv[5] = {
      Integer::New(env->isolate(), status),
      wrap->object(),
      req_wrap_obj,
      Boolean::New(env->isolate(), readable),
      Boolean::New(env->isolate(), writable)
    };
```

AfterConnect is a callback that is passed to uv_pipe_connect. The status will be 0 if uv_connect() was successful and < 0 otherwise. 

The thing to notice is the difference compared to tcp_wrap:
```c++
    Local<Object> req_wrap_obj = req_wrap->object();
    Local<Value> argv[5] = {
      Integer::New(env->isolate(), status),
      wrap->object(),
      req_wrap_obj,
      v8::True(env->isolate()),
      v8::True(env->isolate())
    };
```

TCPWrap always sets the readable and writable values to true where as PipeWrap checks if the handle is readble/writeble. Seems like a the TCPWrap will always
be both readable and writable. 


## Making ReqWrap req_ member private
Currently the member req_ is public in src/req

One issue when doing this was that after renaming req_ to req() I had to rename a macro in src/node_file.cc to avoid a collision with the macro 
parameter with the same name.

The second issue I ran into was with src/stream_wrap.cc:
```c++
    void StreamWrap::AfterShutdown(uv_shutdown_t* req, int status) {
      ShutdownWrap* req_wrap = ContainerOf(&ShutdownWrap::req_, req);
```

We can find `ContainerOf` in src/util-inl.h :
```c++
    template <typename Inner, typename Outer>
    inline ContainerOfHelper<Inner, Outer> ContainerOf(Inner Outer::*field, Inner* pointer) {
      return ContainerOfHelper<Inner, Outer>(field, pointer);
    }
```

The call in question is auto-deducing the paremeter types from the arguments, it could also have been explicit:
```c++
    ShutdownWrap* req_wrap = ContainerOf<uv_shutdown_t*, ShutdownWrap>(&ShutdownWrap::req_, req);
```


## ContainerOfHelper
`src/util.h` declares a class named ContainerOfHelper:
```c++
    // The helper is for doing safe downcasts from base types to derived types.
    template <typename Inner, typename Outer>
    class ContainerOfHelper {
     public:
       inline ContainerOfHelper(Inner Outer::*field, Inner* pointer);
       template <typename TypeName>
       inline operator TypeName*() const;
     private:
       Outer* const pointer_;
 };
```

So back to our call using ContainerOf which will invoke:
```c++
    template <typename Inner, typename Outer>
    ContainerOfHelper<Inner, Outer>::ContainerOfHelper(Inner Outer::*field, Inner* pointer)
        : pointer_(reinterpret_cast<Outer*>(reinterpret_cast<uintptr_t>(pointer) - reinterpret_cast<uintptr_t>(&(static_cast<Outer*>(0)->*field)))) {
    }
```

First, note that the parameter `field` is a pointer-to-member, which gives the offset of the member within the class object as opposed to using the
address-of operator on a data member bound to an actual class object which yields the member's actual address in memory.
`uintptr_t` is an unsigned int that is capable of storing a pointer. Such a type can be used when you need to perform integer operations on a pointer.
[reinterpret_cast](http://en.cppreference.com/w/cpp/language/reinterpret_cast) is a compiler directive which instructs the compiler to treat the sequence
of bits as if it had the new type:
```c++
    reinterpret_cast<uintptr_t>(pointer) 
```
reinterpret_cast is used to convert any pointer type to any other pointer type and the result is a binary copy of the value. 
```c++
        reinterpret_cast<uintptr_t>(&(static_cast<ShutdownWrap*>(0)->*field))
```

I've not seen this usage before using 0 as the argument to static_cast:
```c++
    static_cast<ShutdownWrap*>(0)->*field)
```

The static_cast part of this expression will give a nullptr, but we are not accessing a member, but a pointer-to-member which remember is the offset.
A pointer is only a memory address but the type of the object determines how a pointer can be used, like using a member it needs to know the offsets 
of those members.
So we creating a pointer to Outer which by using the offset of the field and substracting that from `pointer`. So when using a pointer and dereferencing 
`field` this will point to same value of `pointer`


Why does the protected field req_ have to be last:
```console
    Command: out/Release/node /Users/danielbevenius/work/nodejs/node/test/parallel/test-child-process-stdio-big-write-end.js
    --- CRASHED (Signal: 10) ---
    === release test-cluster-disconnect ===
    Path: parallel/test-cluster-disconnect
    /Users/danielbevenius/work/nodejs/node/out/Release/node[84341]: ../src/connection_wrap.cc:83:static void node::ConnectionWrap<node::TCPWrap, uv_tcp_s>::AfterConnect(uv_connect_t *, int) [WrapType = node::TCPWrap, UVType = uv_tcp_s]: Assertion `(req_wrap->env()) == (wrap->env())' failed.
     1: node::Abort() [/Users/danielbevenius/work/nodejs/node/out/Release/node]
     2: node::RunMicrotasks(v8::FunctionCallbackInfo<v8::Value> const&) [/Users/danielbevenius/work/nodejs/node/out/Release/node]
     3: node::ConnectionWrap<node::TCPWrap, uv_tcp_s>::AfterConnect(uv_connect_s*, int) [/Users/danielbevenius/work/nodejs/node/out/Release/node]
     4: uv__stream_io [/Users/danielbevenius/work/nodejs/node/out/Release/node]
     5: uv__io_poll [/Users/danielbevenius/work/nodejs/node/out/Release/node]
     6: uv_run [/Users/danielbevenius/work/nodejs/node/out/Release/node]
     7: node::Start(int, char**) [/Users/danielbevenius/work/nodejs/node/out/Release/node]
     8: start [/Users/danielbevenius/work/nodejs/node/out/Release/node]
     9: 0x2
```

"req_wrap_queue_ needs to be at a fixed offset from the start of the struct because it is used by ContainerOf to calculate the address of the embedding ReqWrap.
ContainerOf compiles down to simple, fixed pointer arithmetic. sizeof(req_) depends on the type of T, so req_wrap_queue_ would no longer be at a fixed offset if it came after req_."

This is what ReqWrap currently looks like:
```c++
     private:
      friend class Environment;
      ListNode<ReqWrap> req_wrap_queue_;
```

Notice that this is not a pointer and when a ReqWrap instance is created the ListNode::ListNode() constructor will be called:
```c++
    template <typename T>
    ListNode<T>::ListNode() : prev_(this), next_(this) {}
```

So every instance will have it's own doubly link linked list and each entry contains a ReqWrap instance which has a type T member. Depending on the type of T
the size of the ReqWrap object in memory will be different. So it would not be possible to have req_wrap_queue after req_, or req_ before req_wrap_queue as this
would make the offset different during runtime (compile time would still work fine).

Every Environment instance has the following queues:
```c++
    HandleWrapQueue handle_wrap_queue_;
    ReqWrapQueue req_wrap_queue_; 
```

And a typedef for this is created using a pointer-to-member: 
```c++
    typedef ListHead<ReqWrap<uv_req_t>, &ReqWrap<uv_req_t>::req_wrap_queue_> ReqWrapQueue;
```

Each time a instance of ReqWrap is created that instance will be added to the queue:
```c++
    env->req_wrap_queue()->PushBack(reinterpret_cast<ReqWrap<uv_req_t>*>(this));
```


### Share AfterWrite with with udp_wrap and stream_wrap 
So, the task is basically to follow this comment in udb_wrap.cc:
```c++
    // TODO(bnoordhuis) share with StreamWrap::AfterWrite() in stream_wrap.cc
    void UDPWrap::OnSend(uv_udp_send_t* req, int status) {
```

At first glance this don't look that similar that they could be shared:
```c++
     void UDPWrap::OnSend(uv_udp_send_t* req, int status) {
       SendWrap* req_wrap = static_cast<SendWrap*>(req->data);
       if (req_wrap->have_callback()) {
         Environment* env = req_wrap->env();
         HandleScope handle_scope(env->isolate());
         Context::Scope context_scope(env->context());
         Local<Value> arg[] = {
           Integer::New(env->isolate(), status),
           Integer::New(env->isolate(), req_wrap->msg_size),
         };
         req_wrap->MakeCallback(env->oncomplete_string(), 2, arg);
      }
      delete req_wrap;
    }
```

`have_callback()` is a method on the SendWrap class and does not exist for WriteWrap.
```c++
   void StreamWrap::AfterWrite(uv_write_t* req, int status) {
    WriteWrap* req_wrap = WriteWrap::from_req(req);
    CHECK_NE(req_wrap, nullptr);
    HandleScope scope(req_wrap->env()->isolate());
    Context::Scope context_scope(req_wrap->env()->context());
    req_wrap->Done(status);
  }
```

First thing to notice is the checking for a callback, StreamWrap::AfterWrite seems to assume that there will
always be a callback by looking at `req_wrap->Done`:
```c++
      inline void Done(int status, const char* error_str = nullptr) {
         Req* req = static_cast<Req*>(this);
         Environment* env = req->env();
         if (error_str != nullptr) {
           req->object()->Set(env->error_string(), OneByteString(env->isolate(), error_str));
         }
        cb_(req, status);
      }
```

When `DoShutdown` is called the last thing that is done is:
```c++
    req_wrap->Dispatched();
```

which will set req_.data = this; this being the Shutdown wrap instance. Later when the `AfterShutdown` method is called that instance will be available 
by using the req->data.


### Stream class hierarchy
```
    class TTYWrap : public StreamWrap

    class PipeWrap : public ConnectionWrap<PipeWrap, uv_pipe_t>
    class TCPWrap : public ConnectionWrap<TCPWrap, uv_tcp_t>
    
    class ConnectionWrap : public StreamWrap
    class StreamWrap : public HandleWrap, public StreamBase
    class HandleWrap : public AsyncWrap
    class AsyncWrap : public BaseObject
    class BaseObject

    class StreamBase : public StreamResource
    class StreamResource
```


### Wrapped 
```javascript
    var TCP = process.binding('tcp_wrap').TCP;
    var TCPConnectWrap = process.binding('tcp_wrap').TCPConnectWrap;
    var ShutdownWrap = process.binding('stream_wrap').ShutdownWrap;

    var client = new TCP();
    var shutdownReq = new ShutdownWrap();
```

This above will invoke the constructor set up by `TCPWrap::Initialize`:
```c++
    auto constructor = [](const FunctionCallbackInfo<Value>& args) {
      CHECK(args.IsConstructCall());
    };
    auto cwt = FunctionTemplate::New(env->isolate(), constructor);
    cwt->InstanceTemplate()->SetInternalFieldCount(1);
    SetClassName(FIXED_ONE_BYTE_STRING(env->isolate(), "TCPConnectWrap"));
    Set(FIXED_ONE_BYTE_STRING(env->isolate(), "TCPConnectWrap"), GetFunction());
```

The only thing the constructor does is check that `new` is used with the function (as in new ShutdownWrap).
```javascript
    var err = client.shutdown(shutdownReq);
```

The methods available to a TCP instance are also configured in `TCPWrap::Initialize`. The `shutdown` method is set up using the 
following call:
```c++
    StreamWrap::AddMethods(env, t, StreamBase::kFlagHasWritev);
```

`src/stream_base-inl.h` contains the shutdown method:
```c++
    env->SetProtoMethod(t, "shutdown", JSMethod<Base, &StreamBase::Shutdown>); 
````

So we are using a referece to StreamBase::Shutdown which can be found in src/stream_base.cc:
```c++
   int StreamBase::Shutdown(const FunctionCallbackInfo<Value>& args) {
     Environment* env = Environment::GetCurrent(args);
 
     CHECK(args[0]->IsObject());
     Local<Object> req_wrap_obj = args[0].As<Object>();

     ShutdownWrap* req_wrap = new ShutdownWrap(env,
                                               req_wrap_obj,
                                               this,
                                               AfterShutdown);
```

The Shutdown constructor delegates to ReqWrap:
```c++
    ReqWrap(env, req_wrap_obj, AsyncWrap::PROVIDER_SHUTDOWNWRAP),
```

Which delegates to AsyncWrap:
```c++
    AsyncWrap(env, req_wrap_obj, AsyncWrap::PROVIDER_SHUTDOWNWRAP),
```

Which delegates to BaseObject:
```c++
    BaseObject(env, req_wrap_obj)
````

req_wrap_obj is refered to handle in BaseObject and is made into a persistent V8 handle

`AfterShutdown` is of type typedef void (*DoneCb)(Req* req, int status). This callback is passed to the constructor of 
StreamReq:
```c++
    StreamReq<ShutdownWrap>(cb)
```

This will simply store the callback in a private field.

The `StreamBase` instance (`this` in the call above) will be set as a private member of Shutdown wrap.
There is a single function call in the constructor which is:

    Wrap(req_wrap_obj, this);

    void Wrap(v8::Local<v8::Object> object, TypeName* pointer) {
      CHECK_EQ(false, object.IsEmpty());
      CHECK_GT(object->InternalFieldCount(), 0);
      object->SetAlignedPointerInInternalField(0, pointer);
    }

So we are setting the ShutdownWrap instance pointer on the V8 local object. So wrap means that we are wrapping the ShutdownWrap instance
in the req_warp_obj.

## Compiling the test in this project
First step is that Google Test needs to be added. Follow the steps in "Adding Google test to the project" before proceeding.

### Building and running the tests

    make check 

### Clean

    make clean

## Adding Google test to the project

### Build the gtest lib:

    $ mkdir lib
    $ mkdir deps ; cd deps
    $ git clone git@github.com:google/googletest.git
    $ cd googletest/googletest
    $ mkdir build ; cd build
    $ c++ -std=gnu++0x -stdlib=libstdc++ -I`pwd`/../include -I`pwd`/../ -pthread -c `pwd`/../src/gtest-all.cc
    $ ar -rv libgtest.a gtest-all.o
    $ cp libgtest.a ../../../../lib

We will be linking against Node.js which is build (on mac) using c++ and using the GNU Standard library. 
Before OS X 10.9.x the default was libstdc++, but after OS X 10.9.x the default is libc++. I'm ususing 10.11.5 so the default would be libc++ in my case. I ran into an issue when compiling and not explicitely specifying `-stdlib=libstdc++` as this would mix two different standard library implementations. 
Instead of our program crashing at runtime we get a link time error. libc++ uses a C++11 language feature called inline namespace to change the ABI of std::string without impacting the API of std::string. That is, to you std::string looks the same. But to the linker, std::string is being mangled as if it is in namespace std::__1. Thus the linker knows that std::basic_string and std::__1::basic_string are two different data structures (the former coming from gcc's libstdc++ and the latter coming from libc++).

### Writing a test file

    $ mkdir test
    $ vi main.cc
    #include "gtest/gtest.h"
    #include "base-object_test.cc"

    int main(int argc, char* argv[]) {
      ::testing::InitGoogleTest(&argc, argv);
      return RUN_ALL_TESTS();
    }

    $ vi base-object_test.cc
    #include "gtest/gtest.h"

    TEST(BaseObject, base) {
    }

then compile using:

    $ clang++ -I`pwd`/../deps/googletest/googletest/include -pthread main.cc ../lib/libgtest.a -o base-object_test

Run the test:

    ./base-object_test


#### use of undeclared identifier 'node'
After making sure that I can include the 'base-object.h' header I get the following error when compiling:

    In file included from test/main.cc:2:
    test/base-object_test.cc:9:3: error: use of undeclared identifier 'node'
    node::BaseObject bo;
     ^
    1 error generated.
    make: *** [test/base-object_test] Error 1

After taking a closer look at `src/base-object.j` I noticed this line:

    #if defined(NODE_WANT_INTERNALS) && NODE_WANT_INTERNALS

I've not set this in the test, so there is not much being included by the preprocessor.
Adding `#define NODE_WANT_INTERNALS 1` should fix this.


#### Default implicit destructor
When working on a task involving extracting commmon code to a superclass I caused an issue
with the CI builds. 

What I had originally done was added an empty destructor:

    ~ConnectionWrap() {
    }

I later changed this to be an explicitly defaulted destructor generated by the compiler

    ~ConnectionWrap() = default;

While I did not see any failures on my local machine during development the CI server did. I currently don't have more information than this but will try to gather some.

My understanding/assumption was that these two would be equivalent. So what is doing on?
Let's start by taking a look a the inheritance tree and the various destructors:


    class ConnectionWrap : public StreamWrap
      protected:
        ~ConnectionWrap() {}

    class StreamWrap : public HandleWrap, public StreamBase
      protected:
        ~StreamWrap() { }

    class HandleWrap : public AsyncWrap
      protected:
        ~HandleWrap() override;

    class AsyncWrap : public BaseObject
      public:
        inline virtual ~AsyncWrap();

    class BaseObject
      public:
        inline virtual ~BaseObject();

    class StreamBase : public StreamResource
      public:
        virtual ~StreamBase() = default;

    class StreamResource
      public:
        virtual ~StreamResource() = default;

From looking at the error:

    In file included from ../src/pipe_wrap.h:7:0,
                 from ../src/pipe_wrap.cc:1:
    ../src/connection_wrap.h:26:3: internal compiler error: in use_thunk, at cp/method.c:338
    ~ConnectionWrap() = default;
    ^
    Please submit a full bug report,
   with preprocessed source if appropriate.
   See <file:///usr/share/doc/gcc-4.8/README.Bugs> for instructions.
   Preprocessed source stored into /tmp/ccvbqQz3.out file, please attach this to your bugreport.
   ERROR: Cannot create report: [Errno 17] File exists: '/var/crash/_usr_lib_gcc_x86_64-linux-gnu_4.8_cc1plus.1000.crash'
    make[2]: *** [/home/iojs/build/workspace/node-test-commit-linux/nodes/ubuntu1204-64/out/Release/obj.target/node/src/pipe_wrap.o] Error 1

it looks like GCC (G++) 4.8 is being used. The reason for asking is I did a search and found a few indications that this might be a bug in the compiler. This is reported as sovled in 4.8.3 which is also why I'm curious about the compiler version. The centos machines use devtoolset-2, which comes with g++ 4.8.2.

If a class has no user-declared destructor, one is declared implicitly by the compiler and is called an implicitly-declared destructor. An implicitly-declared destructor is inline.
Another aspect about destructors that is important to understand is that even if the body of a destructor is empty, it doesn’t mean that this destructor won’t execute any code. The C++ compiler augments the destructor with calls to destructors for bases and non-static data members


### Chrome debugger
Open developer tools from Chrome `CMD+OPT+I`

#### Debugging
`CMD+;`         step into  
`CMD+'`         step over  
`CMD+SHIFT+;`   step out  
`CMD+\`         continue  
`CTRL+.`        next call frame  
`CTRL+,`        previous call frame  
`CMD+B`         toggle breakpoint  
`CTRL+SHIFT+E`  run highlighted snipped and show output in console.  

#### Searching
`CMD+F`         search current file  
`CMD+ALT+F`     search all sources  
`CMD+P`         go to source file. Opens a dialog where you can type in a file name  
`CTRL+G`        go to line  

#### Editor
`SHIFT+CMD+P`   go to member  
  
`ESC`           toggle drawer  
`CTRL+~`        jump to console  
`CMD+[`         next panel  
`CMD+]`         previous panel  
`CMD+ALT+[`     next panel in history  
`CMD+ALT+]`     previous panel history  
`CMD+SHIFT+D`   toggle location of panels (separate screen/docked)  
`?`             show settings dialog  
                You can see all the shortcuts from here    
`ESC`           close settings/dialog  


### Node Package Manager (NPM) 
I was curious about what type of program it is. Looking at the shell script on my machine it is a simple wrapper that calls node with the javascript file being the shell script itself. Kinda like doing:

    #!/bin/sh
    // 2>/dev/null; exec "`dirname "~/.nvm/versions/node/v4.4.3/bin/npm"`/node" "$0" "$@"

    console.log("bajja");

### Make
GNU make has two phases. During the first phase it reads all the makefiles, and internalizes all variables. Make will expand any variables or functions in that section as the makefile is parsed. This is called
immediate expansion since this happens during the first phase. The expansion is called deferred if it is not performed immediately.

Take a look at this rule:

    config.gypi: configure
        if [ -f $@ ]; then
                $(error Stale $@, please re-run ./configure)
        else
                $(error No $@, please run ./configure first)
        fi

The recipe in this case is a shell if statement, which is a deferred construct. But the control function `$(error)` is an immediate construct which will cause the makefile processing to stop processing.
If I understand this correctly the only possible outcome of this rule is the Stale config.gypi message which will be done in the first phase and then exit. The shell condition will not be considered.

For example, if we delete `config.gypi` we would expect the result to be an error saying that `No config.gypi, please run ./configure first`. But the result is:

    Makefile:81: *** Stale config.gypi, please re-run ./configure.  Stop.

Keep in mind that `config.gypi` is not a .PHONY target, so it is a file on the file system and if it is missing the recipe will be run.
So we could use a simple echo statement and and exit to work around this:

    config.gypi: configure
        @if [ -f $@ ]; then \
          echo Stale $@, please re-run ./$<; \
        else \
          echo No $@, please run ./$< first; \
        fi
        @exit 1;

But that will produce the kind of ugly result:

    $ make config.gypi
    Stale config.gypi, please re-run ./configure
    make: *** [config.gypi] Error 1


### AtExit
    // TODO(bnoordhuis) Turn into per-context event.
    4278 void RunAtExit(Environment* env) {


What exactly is a AtExit function. An "AtExit" hook is a function that is invoked after the Node.js event loop has ended but before the JavaScript VM is terminated and Node.js shuts down.

So in node.cc you can find:

    void AtExit(void (*cb)(void* arg), void* arg) {

This would be called like this:

    static void callback(void* arg) {
    }

    AtExit(callback);

    static AtExitCallback* at_exit_functions_;

I notices that AtExist is declared in node.h:

    NODE_EXTERN void RunAtExit(Environment* env);

`NODE_EXTERN` is declared as:

So the idea is that at_exit_functions_ should be a per-environment property rather than a global.
Like bnoordhuis pointed out, AtExit does not take a pointer to an Environment but we have to add the callbacks to the Environment
associated with the addon.
Is the environemnt available when the addons init function is called?  

To answer that question, what is the type contained in the init function of an addon?  

    void init(Local<Object> target) {
      AtExit(at_exit_cb1, target->CreationContext()->GetIsolate());
    }
 
    NODE_MODULE(binding, init);

So a user will still have to call AtExit but instead of node.cc holding a static linked list of callbacks to call these should be
added to the current environment.

    void AtExit(void (*cb)(void* arg), void* arg) {

So AtExit takes a function pointer as its first argument, and a void pointer as its second.
The function pointer is to a function that returns void and takes a void pointer as an argument.

mp->nm_register_func(exports, module, mp->nm_priv);

The above call can be found in `DLOpen` in src/node.cc`. The first thing that happens in DLOpen is:
```c++
    Environment* env = Environment::GetCurrent(args);
```

I've covered the setting of the Environment in `AssignToContext` previously. This is done by the Environment contructor and by 
node_contextify.cc. 


The only `Start` function exposed in node.h is the one that takes `argc` and `argv`. Calling node::Start multiple times does
not work and result in the following error:
```console
# Fatal error in ../deps/v8/src/isolate.cc, line 2021
# Check failed: thread_data_table_.
#
==== C stack trace ===============================

    0   cctest                              0x0000000100324fce v8::base::debug::StackTrace::StackTrace() + 30
    1   cctest                              0x0000000100325005 v8::base::debug::StackTrace::StackTrace() + 21
    2   cctest                              0x000000010031dd94 V8_Fatal + 452
    3   cctest                              0x0000000100cc053c v8::internal::Isolate::Isolate(bool) + 2092
    4   cctest                              0x0000000100cc0ad5 v8::internal::Isolate::Isolate(bool) + 37
    5   cctest                              0x0000000100370a59 v8::Isolate::New(v8::Isolate::CreateParams const&) + 41
    6   cctest                              0x000000010003323f node::Start(uv_loop_s*, int, char const* const*, int, char const* const*) + 79
    7   cctest                              0x0000000100032e38 node::Start(int, char**) + 200
    8   cctest                              0x00000001000cdb33 EnvironmentTest_StartMultipleTimes_Test::TestBody() + 51
    9   cctest                              0x000000010014089a void testing::internal::HandleSehExceptionsInMethodIfSupported<testing::Test, void>(testing::Test*, void (testing::Test::*)(), char const*) + 122
    10  cctest                              0x00000001001190be void testing::internal::HandleExceptionsInMethodIfSupported<testing::Test, void>(testing::Test*, void (testing::Test::*)(), char const*) + 110
    11  cctest                              0x0000000100118fa5 testing::Test::Run() + 197
    12  cctest                              0x0000000100119f98 testing::TestInfo::Run() + 216
    13  cctest                              0x000000010011b227 testing::TestCase::Run() + 231
    14  cctest                              0x0000000100129ccc testing::internal::UnitTestImpl::RunAllTests() + 908
    15  cctest                              0x00000001001444aa bool testing::internal::HandleSehExceptionsInMethodIfSupported<testing::internal::UnitTestImpl, bool>(testing::internal::UnitTestImpl*, bool (testing::internal::UnitTestImpl::*)(), char const*) + 122
    16  cctest                              0x00000001001298be bool testing::internal::HandleExceptionsInMethodIfSupported<testing::internal::UnitTestImpl, bool>(testing::internal::UnitTestImpl*, bool (testing::internal::UnitTestImpl::*)(), char const*) + 110
    17  cctest                              0x00000001001297b5 testing::UnitTest::Run() + 373
    18  cctest                              0x0000000100147a81 RUN_ALL_TESTS() + 17
    19  cctest                              0x0000000100147a5b main + 43
    20  cctest                              0x00000001000010f4 start + 52
make: *** [cctest] Illegal instruction: 4
```

The Environment created when using the above start function is done in
```c++
    inline int Start(Isolate* isolate, IsolateData* isolate_data,
                     int argc, const char* const* argv,
                     int exec_argc, const char* const* exec_argv) {
```
Would it be safe to use GetCurrent using the isolate in 



### Thread-local
```c++
    static thread_local Environment* thread_local_env;
```

The object is allocated when the thread begins and deallocated when the thread ends. Each thread has its own instance of the object. 
Only objects declared thread_local have this storage duration.  thread_local can appear together with static or extern to adjust linkage.

So we are specifying static only to specify that it should only have internal linkage, meaning that it can be referred to from all scopes in the current translation unit. It does not 
mean that it is static as in "static storage" meaning that it would be allocated when the program begins and deallocated when the program ends.
But without the static linkage it would be external by default which is not what we want.

When used in a declaration of an object, it specifies static storage duration (except if accompanied by thread_local). When used in a declaration at 
namespace scope, it specifies internal linkage.

```console
Using a `while(more == true)' :
    0x1011d81a9 <+1177>: jmp    0x1011d81ae               ; <+1182> at node.cc:4453
    0x1011d81ae <+1182>: movb   -0xd31(%rbp), %al         ; move byte value of -0xd31(%rpb) (move variable) into al register
    0x1011d81b4 <+1188>: andb   $0x1, %al                 ; AND 1 and the content of move variable
    0x1011d81b6 <+1190>: movzbl %al, %ecx                 ; conditional move into eax if zero
    0x1011d81b9 <+1193>: cmpl   $0x1, %ecx                ; compare 1 and the contents of eax
    0x1011d81bc <+1196>: je     0x1011d80e9               ; <+985> at node.cc:4437

    0x1011d81c2 <+1202>: leaq   -0xd30(%rbp), %rdi
    0x1011d81c9 <+1209>: callq  0x1002214e0               ; v8::SealHandleScope::~SealHandleScope at api.cc:926


Compared to using `while(more)`:

    0x1011d81a9 <+1177>: jmp    0x1011d81ae               ; <+1182> at node.cc:4453
    0x1011d81ae <+1182>: testb  $0x1, -0xd31(%rbp)        ; AND 1 and more
    0x1011d81b5 <+1189>: jne    0x1011d80e9               ; <+985> at node.cc:4437

    0x1011d81bb <+1195>: leaq   -0xd30(%rbp), %rdi
    0x1011d81c2 <+1202>: callq  0x1002214e0               ; v8::SealHandleScope::~SealHandleScope at api.cc:926
```


#### Calling conventions
Are the rules when making functions calls regarding how parameters are passed, who is responsible for cleaning up the stack, 
how the return value is to be retrieved, and also how the function calls are decorated.

### cdecl
A calling convention that is used for standard C where the the stack must be cleaned up by the callee as there is support for varargs
and there is now way for the called function to know the actual number of values pushed onto the stack before the function was called.
Function name is decorated by prefixing it with an underscore character '_' .

### stdcall
Here arguments are fixed and the called function can to the stack clean up. The advantage here is that the stack clean up code is only
done once in one place.
Function name is decorated by prepending an underscore character and appending a '@' character and the number of bytes of stack space required.

### Issue
When running the Node.js build on windows (trying to get cctest to work for a test I added), I got the following link error:
```console
    env.obj : error LNK2001: unresolved external symbol 
    "public: __cdecl node::Utf8Value::Utf8Value(class v8::Isolate *,class v8::Local<class v8::Value>)" (??0Utf8Value@node@@QEAA@PEAVIsolate@v8@@V?$Local@VValue@v8@@@3@@Z) [c:\workspace\node-compile-windows\label\win-vs2015\cctest.vcxproj]
```

Now, we can see that the calling convention used is `__cdecl` but the name mangling does not look correct as it is using @@
```c++
    process_title.len = argv[argc - 1] + strlen(argv[argc - 1]) - argv[0];
```

This would be the same as :
```console
    (lldb) p (size_t) argv[argc-1] + (size_t) strlen(argv[argc-1]) - (size_t)argv[0]
    (unsigned long) $9 = 56
```

When in my unit test the same gives me:
```console
    (lldb) p (size_t) argv[argc-1] + (size_t) strlen(argv[argc-1]) - (size_t)argv[0]
    (unsigned long) $10 = 34693
```

What is happening is that we are taking the memory address of argv[argc-1] + 

#### Debugging a Node addon
The task a hand was to debug Realm's addon to see why test were just hanging even though I made sure to call Tape test's end function.
So, realm is a normal dependency and exist in node_modules.

Setup:
```console
    $ npm install --save realm
    $ cd node_modules/realm
    $ env REALMJS_USE_DEBUG_CORE=true node-pre-gyp install --build-from-source --debug
    $ lldb -- node test/datastores/realm-store-test.js
    (lldb) breakpoint set --file node_init.cpp --line 26
```

It turns out that when breaking in the debugger (CTRL+C) and then stepping through it was in a kevent and this migth be some kind of
listerner for events, and there is a realm.removeAllListeners() that can be called and this solved my issue.


### Generate Your Project
For node the various targets in node.gyp will generate make files in the `out` directory.
For example the target named `cctest` will generate out/cctest.target.mk file.

### Profiling
You can use Google V8's built in profiler using the `--prof` command line option:
```console
    $ out/Debug/node --prof test.js
```


This will generate a file in the current directory named something like `isolate-0x104005e00-v8.log`.
Now we can process this file:
```console
    $ export D8_PATH=~/work/google/javascript/v8/out/x64.debug
    $ deps/v8/tools/mac-tick-processor isolate-0x104005e00-v8.log
```

or you can use node's `---prof-process` option:
```console
    $ ./out/Debug/node --prof-process isolate-0x104005e00-v8.log
```

    Statistical profiling result from isolate-0x104005e00-v8.log, (332 ticks, 86 unaccounted, 0 excluded).


The profiler is sample based so with wakes up and takes a sample. The intervals that is wakes up is called a tick. It will look
at where the instruction pointer is RIP and reports the function if that function can be resolved. If cannot resolve the function
this will be reported as an unaccounted tick.
```console

    [Summary]:
     ticks  total  nonlib   name
        0    0.0%    0.0%  JavaScript
      236   71.1%   73.3%  C++
        4    1.2%    1.2%  GC
       10    3.0%          Shared libraries
       86   25.9%          Unaccounted
```

We can see that `71.1%` of the time was spent in C++ code. Inspecting the C++ section you should be able to see were the most
time is being spent and the sources.

```console
    [C++]:
     ticks  total  nonlib   name
       66   19.9%   20.5%  node::ContextifyScript::New(v8::FunctionCallbackInfo<v8::Value> const&)
       20    6.0%    6.2%  node::Binding(v8::FunctionCallbackInfo<v8::Value> const&)
        5    1.5%    1.6%  v8::internal::HandleScope::ZapRange(v8::internal::Object**, v8::internal::Object**)
```

The `[Bottom up]` section shows us which the primary callers of the above are:
```console

    [Bottom up (heavy) profile]:
    Note: percentage shows a share of a particular caller in the total amount of its parent calls.
    Callers occupying less than 2.0% are not shown.

     ticks parent  name
       86   25.9%  UNKNOWN

       66   19.9%  node::ContextifyScript::New(v8::FunctionCallbackInfo<v8::Value> const&)
       66  100.0%    v8::internal::Builtin_HandleApiCall(int, v8::internal::Object**, v8::internal::Isolate*)
       66  100.0%      LazyCompile: ~runInThisContext bootstrap_node.js:427:28
       66  100.0%        LazyCompile: ~NativeModule.compile bootstrap_node.js:509:44
       66  100.0%          LazyCompile: ~NativeModule.require bootstrap_node.js:443:34
       15   22.7%            LazyCompile: ~startup bootstrap_node.js:12:19
       11   16.7%            Function: ~<anonymous> module.js:1:11
        8   12.1%            Function: ~<anonymous> stream.js:1:11
        7   10.6%            LazyCompile: ~setupGlobalVariables bootstrap_node.js:192:32
        6    9.1%            Function: ~<anonymous> util.js:1:11
        6    9.1%            Function: ~<anonymous> tty.js:1:11
        3    4.5%            LazyCompile: ~setupGlobalTimeouts bootstrap_node.js:226:31
        2    3.0%            LazyCompile: ~createWritableStdioStream internal/process/stdio.js:134:35
        2    3.0%            Function: ~<anonymous> fs.js:1:11
        2    3.0%            Function: ~<anonymous> buffer.js:1:11
```

LazyCompile: Simply means that the function was complied lazily and not that this was the time spent compiling.
* before function name means that time is being spent in optimized function.
~ before a function means that is was not optimized.

The % in the parent column shows the percentage of samples for which the function in the row above was called by the
function in the current row.
So, 
```console

       66   19.9%  node::ContextifyScript::New(v8::FunctionCallbackInfo<v8::Value> const&)
       66  100.0%    v8::internal::Builtin_HandleApiCall(int, v8::internal::Object**, v8::internal::Isolate*)
```

would be read as when `v8::internal::Builting_HandleApiCall` was sampled it called node:ContextifyScript every time.
And
```console

       66  100.0%          LazyCompile: ~NativeModule.require bootstrap_node.js:443:34
       15   22.7%            LazyCompile: ~startup bootstrap_node.js:12:19
```
that when startup in bootstrap_node.js was called, in 22% of the samples it called NativeModule.require.
```console

    [Shared libraries]:
     ticks  total  nonlib   name
        6    1.8%          /usr/lib/system/libsystem_kernel.dylib
        2    0.6%          /usr/lib/system/libsystem_platform.dylib
        1    0.3%          /usr/lib/system/libsystem_malloc.dylib
        1    0.3%          /usr/lib/system/libsystem_c.dylib
```



### setTimeout

Let's take the following example:
```javascript
    setTimeout(function () {
      console.log('bajja');
    }, 5000);
```
```console
$ ./out/Debug/node --inspect --inspect-brk settimeout.js
```

In Node you can call setTimeout with out having a require. This is done by `lib/boostrap/node.js`:
```javascript
    function setupGlobalTimeouts() {
      const timers = NativeModule.require('timers');
      global.clearImmediate = timers.clearImmediate;
      global.clearInterval = timers.clearInterval;
      global.clearTimeout = timers.clearTimeout;
      global.setImmediate = timers.setImmediate;
      global.setInterval = timers.setInterval;
      global.setTimeout = timers.setTimeout;
   }
```

So we can see that we are able to call setTimout without having to require any module and that it is part of a
native modules named timers. This is located in lib/timers.js.

The first thing that will happen is a new Timeout will be created in `createSingleTimeout`. A timeout looks like:
```javascript
    function Timeout(after, callback, args) {
      this._called = false;
      this._idleTimeout = after;  // this will be 5000 in our use-case
      this._idlePrev = this;
      this._idleNext = this;
      this._idleStart = null;
      this._onTimeout = callback; // this is our callback that just logs to the console
      this._timerArgs = args;
      this._repeat = null;
    }
```

This `timer` instance is then passed to `active(timer)` which will insert the timer by calling `insert`:
```javascript
     insert(item, false);
```

`item` is the timer, and false is the value of the unrefed argument)
```javascript
    item._idleStart = TimerWrap.now();

So we can see that we are using timer_wrap which is located in src/timer_wrap.cc and the now function which is 
initialized to:
```c++
    env->SetTemplateMethod(constructor, "now", Now);
```

Back in the insert function we then have the following:
```javascript
    const lists = unrefed === true ? unrefedLists : refedLists;
```

We know that unrefed is false so lists will be the refedLists which is an object keyed with the millisecond that a timeout is due
to expire. The value of each key is a linkedlist of timers that expire at the same time. 
```javascript
    var list = lists[msecs];
```

If there are other timers that also expire after 5000ms then there might already be a list for them. But in this case there is not
and a new list will be created:
```javascript
    lists[msecs] = list = createTimersList(msecs, unrefed);

    const list = new TimersList(msecs, unrefed); // 5000 and false

    function TimersList(msecs, unrefed) {
      this._idleNext = null; // Create the list with the linkedlist properties to
      this._idlePrev = null; // prevent any unnecessary hidden class changes.
      this._timer = new TimerWrap();
      this._unrefed = unrefed; // will be false in our case
      this.msecs = msecs; // will be 5000 in our case
   }
```

The `new TimerWrap` call will invoke `New` in timer_wrap.cc as setup in the initialize function:
```c++
    Local<FunctionTemplate> constructor = env->NewFunctionTemplate(New);
```

`New` will invoke TimerWrap's constructor which does:

```c++
    int r = uv_timer_init(env->event_loop(), &handle_);
```

So we can see that it is setting up a libuv [timer](https://github.com/danbev/learning-libuv/blob/master/timer.c).
Shortly after we have the following code (back in JavaScript land and lib/timers.js):

Next the list (TimerList) is initialized setting _idleNext and _idlePrev to list. After this we are adding
a field to the list:

```javascript
    list._timer._list = list;

    list._timer.start(msecs);
```

Start is initialized using :
```c++
    env->SetProtoMethod(constructor, "start", Start);

    static void Start(const FunctionCallbackInfo<Value>& args) {
      TimerWrap* wrap = Unwrap<TimerWrap>(args.Holder());

      CHECK(HandleWrap::IsAlive(wrap));

      int64_t timeout = args[0]->IntegerValue();
      int err = uv_timer_start(&wrap->handle_, OnTimeout, timeout, 0);
      args.GetReturnValue().Set(err);
   }
```

Compare this with [timer.c](https://github.com/danbev/learning-libuv/blob/master/timer.c). and you can see that these is not
that much of a difference. Let's look at the callback OnTimeout:

```c++
    static void OnTimeout(uv_timer_t* handle) {
      TimerWrap* wrap = static_cast<TimerWrap*>(handle->data);
      Environment* env = wrap->env();
      HandleScope handle_scope(env->isolate());
      Context::Scope context_scope(env->context());
      wrap->MakeCallback(kOnTimeout, 0, nullptr);
    }
```

The callback in question looks like:
```console
    0xa90abea6961: [Function]
     - map = 0x2fecee786da1 [FastProperties]
     - prototype = 0x1b3829484539
     - elements = 0x2203fd302241 <FixedArray[0]> [FAST_HOLEY_ELEMENTS]
     - initial_map =
     - shared_info = 0x28b2bad27aa1 <SharedFunctionInfo listOnTimeout>
     - name = 0x28b2bad26b31 <String[13]: listOnTimeout>
     - formal_parameter_count = 0
     - context = 0x19bfe9663951 <FixedArray[48]>
     - feedback vector cell = 0x28b2bad2a549 <Cell value= 0x2203fd302311 <undefined>>
     - code = 0x26d0e2004941 <Code BUILTIN>
     - properties = 0x2203fd302241 <FixedArray[0]> {
        #length: 0x35bd747eed51 <AccessorInfo> (const accessor descriptor)
        #name: 0x35bd747eedc1 <AccessorInfo> (const accessor descriptor)
        #prototype: 0x35bd747eee31 <AccessorInfo> (const accessor descriptor)
     }
```

Notice that the callback is `listOnTimeout` and this can be found in `lib/timer.js`.

### setImmediate
The very simple JavaScript looks like this:

```javascript
    setImmediate(function () {
      console.log('bajja');
    });
```

Like setTimeout the implementation is found in lib/timers.js. 
A new Immediate will be created in `createImmediate` which looks like this:

```javascript
    function Immediate() {
      // assigning the callback here can cause optimize/deoptimize thrashing
      // so have caller annotate the object (node v6.0.0, v8 5.0.71.35)
      this._idleNext = null;
      this._idlePrev = null;
      this._callback = null;
      this._argv = null;
      this._onImmediate = null;
      this.domain = process.domain;
    }
```

The following check will then be done:

```javascript
    if (!process._needImmediateCallback) {
      process._needImmediateCallback = true;
      process._immediateCallback = processImmediate;
    }
```

In this case `process._needImmediateCallback` is false so we'll enter the above block and set process._needImmediateCallback
to `true`. 

Also, notice that we are setting the processImmediate instance as a member of the process object. 
`processImmediate` is a function defined in timer.js. There is a V8 accessor for the field `_immediateCallback` on the process object which is set up in node.cc (SetupProcessObject function):

```javascript
    auto need_immediate_callback_string =
        FIXED_ONE_BYTE_STRING(env->isolate(), "_needImmediateCallback");
    CHECK(process->SetAccessor(env->context(), need_immediate_callback_string,
                               NeedImmediateCallbackGetter,
                               NeedImmediateCallbackSetter,
                               env->as_external()).FromJust());
```
So when we do `process_.immediateCallback` `NeedImmediateCallbackSetter` will be invoked.
Looking closer at this function and comparing it with a [libuv check example](https://github.com/danbev/learning-libuv/blob/master/check.c) we should see some similarties.

```c++
    uv_check_t* immediate_check_handle = env->immediate_check_handle();

    uv_idle_t* immediate_idle_handle = env->immediate_idle_handle();

    uv_check_start(immediate_check_handle, CheckImmediate);
    // Idle handle is needed only to stop the event loop from blocking in poll.
    uv_idle_start(immediate_idle_handle, IdleImmediateDummy);
```

So we can see that when this setter is called it will set up check handle (if the value
was true as in `process._needImmediateCallback = true`).
When the check phase is reached the `CheckImmediate` callback will be invoked. Lets set a breakpoint in that function and verify this:
```console
    (lldb) breakpoint set --file node.cc --line 286 

    static void CheckImmediate(uv_check_t* handle) {
      Environment* env = Environment::from_immediate_check_handle(handle);
      HandleScope scope(env->isolate());
      Context::Scope context_scope(env->context());
      MakeCallback(env, env->process_object(), env->immediate_callback_string());
    }
```

Following `MakeCallback` will will find ourselves in timers.js and its `processImmediate` function which you might recall that we set:

```javascript
     process._immediateCallback = processImmediate;

     immediate._callback = immediate._onImmediate;
```

`immediate._onImmediate` will be our callback function (anonymous in setimmediate.js)

```javascript
    tryOnImmediate(immediate, tail);
```

will call:

```javascript
    runCallback(immediate);
```

will call:

```javascript
    return timer._callback();
```

And the callback is:

```javascript
    function () {
       console.log('bajja');
    }
```

And there we have how setImmediate works in Node.js.

### process._nextTick
The very simple JavaScript looks like this:

```javascript
    process.nextTick(function () {
      console.log('bajja');
    });
```

`nextTick` is defined in `lib/internal/process/next_tick.js`.
After a few checks what happens is that the callback is added to the nextTickQueue:

```javascript
    nextTickQueue.push({
      callback,
      domain: process.domain || null,
      args
    });
```

`nextTickQueue` is an array:

```javascript
    var nextTickQueue = [];
```

And we are pushing an object with the callback as a function named callback, domain
and args. So for every nextTick called an entry will be added to the queue. 

```javascript
    tickInfo[kLength]++;
```

Recall that TickInfo is an inner class of Environment. Lets back up a little. `bootstrap/node.js` will call next_tick's setup() function from its start function:

```javascript
    NativeModule.require('internal/process/next_tick').setup();

    exports.setup = setupNextTick;

    var microtasksScheduled = false;

    // Used to run V8's micro task queue.
    var _runMicrotasks = {};

    // *Must* match Environment::TickInfo::Fields in src/env.h.
    var kIndex = 0;
    var kLength = 1;

    process.nextTick = nextTick;
    // Needs to be accessible from beyond this scope.
    process._tickCallback = _tickCallback;
    process._tickDomainCallback = _tickDomainCallback;

    // This tickInfo thing is used so that the C++ code in src/node.cc
    // can have easy access to our nextTick state, and avoid unnecessary
    // calls into JS land.
    const tickInfo = process._setupNextTick(_tickCallback, _runMicrotasks);
```

`process._setupNextTick` is initialized in `SetupProcessObject` in src/node.cc:

```c++
    env->SetMethod(process, "_setupNextTick", SetupNextTick);
```

Lets take a look at what SetupNextTick does...

```c++
    env->set_tick_callback_function(args[0].As<Function>());

    env->SetMethod(args[1].As<Object>(), "runMicrotasks", RunMicrotasks);
```

So, here we are setting a method named `runMicrotasks` on the `_runMicrotasks` object
passed to `_setupNextTick`.

```c++
    // Do a little housekeeping.
    env->process_object()->Delete(
        env->context(),
        FIXED_ONE_BYTE_STRING(args.GetIsolate(), "_setupNextTick")).FromJust();
```

Looks like this removes the _setupNextTick function from the process object afterwards.

```c++
    uint32_t* const fields = env->tick_info()->fields();
    uint32_t const fields_count = env->tick_info()->fields_count();
```

What are 'fields'? What are 'fields_count'?  
```console
    (lldb) p fields_count
    (uint32_t) $23 = 2

    Local<ArrayBuffer> array_buffer =
        ArrayBuffer::New(env->isolate(), fields, sizeof(*fields) * fields_count);

    args.GetReturnValue().Set(Uint32Array::New(array_buffer, 0, fields_count));
```

So `tickInfo` returned will be an ArrayBuffer:

```javascript
    const tickInfo = process._setupNextTick(_tickCallback, _runMicrotasks);
```

Next we assign the `RunMicroTasks` callback to the `_runMicrotasks` variable:

```javascript
    _runMicrotasks = _runMicrotasks.runMicrotasks;
```

After this we are done in bootstrap/node.js and the setup of next_tick.
So, lets continue and break in our script and follow process.setNextTick.

```javascript
    nextTickQueue.push({
      callback,
      domain: process.domain || null,
      args
    });
```

So we are again showing that we add callback info to the nextTickQueue (after a few checks)
Then we do the following:

```javascript
    tickInfo[kLength]++;
```

For each object added to the nextTickQueue we will increment the second element of the tickInfo
array.

And that is it, the stack frames will start returning and be poped off the call stack. What we
are interested in is in module.js and `Module.runMain`:
  
```javascript
    process._tickCallback();


    do {
      while (tickInfo[kIndex] < tickInfo[kLength]) {
        tock = nextTickQueue[tickInfo[kIndex]++];
        ...
        _combinedTickCallback(args, callback);
        if (kMaxCallbacksUntilQueueIsShortened < tickInfo[kIndex])
           tickDone();
      }
    } while (tickInfo[kLength] !== 0);
```

The check is to see if tickInfo[kIndex] (is this the index of being processed?) is less than
the number of tick callbacks in the `nextTickQueue`.
Next tickInfo[kIndex] is retrieved from the nextTickQueue and then tickInfo[kIndex] is incremented.

`tickDone()`:

```javascript
    function tickDone() {
      if (tickInfo[kLength] !== 0) {
        if (tickInfo[kLength] <= tickInfo[kIndex]) {
          nextTickQueue = [];
          tickInfo[kLength] = 0;
        } else {
          nextTickQueue.splice(0, tickInfo[kIndex]);
          tickInfo[kLength] = nextTickQueue.length;
        }
      }
      tickInfo[kIndex] = 0;
     }
```

Lets take a look at:

```javascript
    if (tickInfo[kLength] <= tickInfo[kIndex]) {
```

If the number of callbacks added is less than or equal to the just processed callbacks index this would mean
that all of the callbacks in the queue have been processed and the following clause will make nextTickQueue 
point to an empty array and reset tickInfo[kLength] to zero.
But if there are more callback in the queue than the just processed callbacks index the else clause will be 
taken:

```javascript
    nextTickQueue.splice(0, tickInfo[kIndex]);
    tickInfo[kLength] = nextTickQueue.length;
```

splice will remove all elements from 0 to tickInfo[kIndex], which is removing all the processed callbacks.
The new length is set as tickInfo[kLength]. This is done so that the nextTickQueue array does not become
too large and run the process out of memory. By shortning the array this reduces the likelyhood of this 
happening.

#### Compiling with a different version of libuv
What I'd like to do is use my local fork of libuv instead of the one in the deps
directory. I think the way to do this is to `make install` and then run configure with the following options:
```console
    $ ./configure --debug --shared-libuv --shared-libuv-includes=/usr/local/include
```

The location of the library is `/usr/local/lib`, and `/usr/local/include` for the headers on my machine.

### Updating addons test
Some of the addons tests are not version controlled but instead generate using:
```console
   $ ./node tools/doc/addon-verify.js doc/api/addons.md
```

The source for these tests can be found in `doc/api/addons.md` and these might need to be updated if a 
change to all tests is required, for a concrete example we wanted to update the build/Release/addon directory
to be different depending on the build type (Debug/Release) and I forgot to update these tests.


### Using nvm with Node.js source
Install to the nvm versions directory:
```console
    $ make install DESTDIR=~/.nvm/versions/node/ PREFIX=v8.0.0
```

You can then use nvm to list that version and versions:
```console
   $ nvm ls 
         v6.5.0
         v7.0.0
         v7.4.0
         v8.0.0

   $ nvm use 8
```

### lldb
There is a [.lldbinit](./.lldbinit) which contains a number of useful alias to 
print out various V8 objects. This are most of the aliases defined in [gdbinit](https://github.com/v8/v8/blob/master/tools/gdbinit).

For example, you can print a v8::Local<v8::Function> using the builtin print command:
```console

    (lldb) p init_fn
    (v8::Local<v8::Function>) $3 = (val_ = 0x000000010484f900)
```

This does not give much, but if we instead use jlh:
```console
    (lldb) jlh init_fn
    0x19417e265ba9: [Function]
     - map = 0x382d7ba86ea9 [FastProperties]
     - prototype = 0xd21f3203f39
     - elements = 0x18ede4802241 <FixedArray[0]> [FAST_HOLEY_ELEMENTS]
     - initial_map =
     - shared_info = 0x23c6dbac1ce1 <SharedFunctionInfo init>
     - name = 0x21a813bbd419 <String[4]: init>
     - formal_parameter_count = 4
     - context = 0x19417e203b41 <FixedArray[8]>
     - literals = 0x18ede4804a49 <FixedArray[1]>
     - code = 0x1aa594184481 <Code: BUILTIN>
     - properties = {
       #length: 0x18ede4850bd9 <AccessorInfo> (accessor constant)
       #name: 0x18ede4850c49 <AccessorInfo> (accessor constant)
       #prototype: 0x18ede4850cb9 <AccessorInfo> (accessor constant)
     }
```

So that gives us more information, but lets say you'd like to see the name of the function:
```console
    (lldb) jlh init_fn->GetName()
    #init
```

### Promise builtin
For debugging the builtin promise we are going to disable V8 snapshots:
```console
$ ./configure --without-snapshot --debug
$ lldb -- out/Debug/node ../scripts/promise.js
(lldb) br s -f bootstrapper.cc -l 2321
```
It takes a while before the breakpoint is hit but it will be.
And we are going to look at the following js:
```javascript
const p = new Promise((resolve, reject) => {
  resolve('ok');
});
```

```c++
Handle<JSFunction> promise_fun = InstallFunction(global,
    "Promise", JS_PROMISE_TYPE, JSPromise::kSizeWithEmbedderFields,
    0, factory->the_hole_value(), Builtins::kPromiseConstructor);
```
```console
(lldb) expr isolate()->builtins()->builtin_handle(Builtins::Name::kPromiseConstructor)->Print()
0x3c5be1744d61: [Code]
 - map: 0x3657d7704051 <Map(HOLEY_ELEMENTS)>
kind = BUILTIN
name = PromiseConstructor
compiler = turbofan
address = 0x3c5be1744d61
Body (size = 3644)
Instructions (size = 3320)
0x3c5be1744dc0     0  55             push rbp
0x3c5be1744dc1     1  4889e5         REX.W movq rbp,rsp
0x3c5be1744dc4     4  56             push rsi
0x3c5be1744dc5     5  57             push rdi
0x3c5be1744dc6     6  50             push rax
0x3c5be1744dc7     7  4883ec40       REX.W subq rsp,0x40
0x3c5be1744dcb     b  4989e2         REX.W movq r10,rsp
0x3c5be1744dce     e  4883ec08       REX.W subq rsp,0x8
0x3c5be1744dd2    12  4883e4f0       REX.W andq rsp,0xf0
0x3c5be1744dd6    16  4c891424       REX.W movq [rsp],r10
0x3c5be1744dda    1a  488bc2         REX.W movq rax,rdx
0x3c5be1744ddd    1d  488955e0       REX.W movq [rbp-0x20],rdx
0x3c5be1744de1    21  488bde         REX.W movq rbx,rsi
0x3c5be1744de4    24  488bfe         REX.W movq rdi,rsi
0x3c5be1744de7    27  48be000000001c000000 REX.W movq rsi,0x1c00000000
0x3c5be1744df1    31  48ba69bce15e57360000 REX.W movq rdx,0x36575ee1bc69    ;; object: 0x36575ee1bc69 <String[157]: CAST(Parameter(Linkage::GetJSCallContextParamIndex( static_cast<int>(call_descriptor->JSParameterCount())))) at ../deps/v8/src/compiler/code-assembler.cc:385>
0x3c5be1744dfb    3b  498d85185620fa REX.W leaq rax,[r13-0x5dfa9e8]
....
```
```c++
  Handle<SharedFunctionInfo> shared(promise_fun->shared(), isolate);
  shared->SetConstructStub(*BUILTIN_CODE(isolate, JSBuiltinsConstructStub));
  shared->set_internal_formal_parameter_count(1);
  shared->set_length(1);

  InstallSpeciesGetter(promise_fun);
  SimpleInstallFunction(promise_fun, "all", Builtins::kPromiseAll, 1, true);
  SimpleInstallFunction(promise_fun, "race", Builtins::kPromiseRace, 1, true);
  SimpleInstallFunction(promise_fun, "resolve", Builtins::kPromiseResolveTrampoline, 1, true);
  SimpleInstallFunction(promise_fun, "reject", Builtins::kPromiseReject, 1, true);
```
So we can see that the Promise function is set up as a global. The `then` function is later setup using:
```c++
  Handle<JSFunction> promise_then = SimpleInstallFunction(prototype, isolate->factory()->then_string(), Builtins::kPromisePrototypeThen, 2, true);
  native_context()->set_promise_then(*promise_then);
```

`deps/v8/src/builtins/builtins-definitions.h`:
```c++
TFJ(PromiseConstructor, 1, kExecutor)
```
TFJ means TurboFan JavaScript linkage and means it is callable as a JavaScript function.

### Debug JavaScript tests
Just example commands that I use in different projects to run the debugger with 
different test suites.

#### Mocha
```console
    $ mocha --inspect --debug-brk  -u exports --recursive -t 10000 ./test/setup.js  test/sync/test_index.js
```

### crypto
The current version of openssl is 1.0.2k (run process.versions.openssl). So this is major version 1, minor 0
and patch 2k I guess.
To investigate lets take a look what happens when one requires crypto:
```javascript
    const crypto = require('crypto');
```
```console
    $ lldb -- ./out/Debug/node crypto.js
```
    
src/node_crypto.cc is a builtin:

```c++
    NODE_MODULE_CONTEXT_AWARE_BUILTIN(crypto, node::crypto::InitCrypto)
```

So, lets set a breakpoint in `InitCrypto`:
```console
    (lldb) breakpoint set -f node_crypto.cc -l 6007
    (lldb) breakpoint set -f node_crypto.cc -l 5880
```

First things that happens is that libcrypto must be initializes. My understanding is that OpenSSL has two libraries which are 
libssl which used libcrypto. Node is using libcrypto in this case.

From InitCryptOnce:
```c++
    SSL_load_error_strings();
    OPENSSL_no_config();
```

OPENSSL_no_config() marks OpenSSL as configured. It seems that if this is not done to avoid some of OpenSSLs standard init functions
that automatically call the configuration to just return hence do nothing.

```c++
   if (!openssl_config.empty())
```
This path would be taken if a openssl config had been passed using `--openssl-config`.

Next, we have:

```c++
    SSL_library_init();
```

This function can be found in `deps/openssl/openssl/ssl/ssl_algs.c`

```c++
    EVP_add_cipher(EVP_des_cbc());
```

EVP I think stands for envelope and has a number of high level cryptographic functions.

#### CNNIC
China Internet Network Information Center (CNNIC) is referenced in some code. It is a Certificate Authority (may be other things as well).


#### Signed Public Key and Challenge (SPKAC)
Also known as Netscape SPKI (spooky). There was originally an element named keygen in the html5 spec which was later
removed. The intention was to create client side certificates through a web service for protocols like WebID.

#### Building with shared openssl
Building an [locally built version](https://github.com/danbev/learning-libcrypto#building-openssl) of OpenSSL.
```console
$ ./configure --debug --shared-openssl --shared-openssl-libpath=/Users/danielbevenius/work/security/build_1_1_0g/lib --shared-openssl-includes=/Users/danielbevenius/work/security/build_1_1_0g/include
$ make -j8
```

#### Building OpenSSL without elliptic curve support
```console
$ ./Configure no-ec --debug --prefix=/Users/danielbevenius/work/security/openssl/build  --libdir="openssl" darwin64-x86_64-cc
```

Then building Node against that version:
```console
$ ./configure --shared-openssl --shared-openssl-libpath=/Users/danielbevenius/work/security/openssl/build/openssl --shared-openssl-includes=/Users/danielbevenius/work/security/openssl/build/include
```

This will not compile are the headers for ec will not exist in the `--shared-openssl-includes` directory. You'll have to use the source
include directory instead so that all the headers can be found. 
```console
    $ ./configure --debug --shared-openssl --shared-openssl-libpath=/Users/danielbevenius/work/security/openssl/build/openssl --prefix=/Users/danielbevenius/work/nodejs/build --shared-openssl-includes=/Users/danielbevenius/work/security/openssl/include
```

There will instead be a runtime error if you try to call functions that require EC but you'll be able to build.

Notice here that we have configured OpenSSL without Elliptic curve support

I was wondering what the values will if you just specify `--shared-openssl` which I've seen. In this case `pkg-config` will be called to retrieve information about
install libraries in the system. For my system this will be:

    'libraries': ['-lcrypto', '-lssl']},

#### RHEL8
OpenSSL on RHEL8 will be OpenSSL 1.1.1 and TLS 1.3. For node this will have 
implications if node is built and dynamically linking OpenSSL.

Is the OpenSSL version that RHEL ships the same as the upstream one?  
If there is no difference than perhaps using different certificates or something,
I think we should be using the statically linked version of OpenSSL. I know that 
previously dynamically linking was argued as better as that would mean that only 
the OpenSSL dynamic library would have to be updated in the case of a CVE. But 
in node's case it is very tightly coupled with OpenSSL and changes are that there 
are code changes in node core required. So just updating the dynamic OpenSSL 
library might break a node application. So in reality if a CVE coming out for OpenSSL the node
package/executable will also have to be updated. Also, at least for us with
in rhoar our target is a container we would have to update the base image in
addition to an update to node.. 

### Building on Solaris
I used VirtualBox to build and run the test suite on Solaris
```console
    $ uname -a
    SunOS solaris 5.11 11.3 i86pc i386 i86pc
```

#### Setup
```console
    $ sudo pkgadd -d http://get.opencsw.org/now
```

Install git:
```console
    $ sudo /opt/csw/bin/pkgutil -y -i git
    $ export PATH=/opt/csw/bin:$PATH      // added to ~/.bashrc
```

Install binutils:
```console
    $ sudo pkgutil -y -i binutils
    $ export PATH=/opt/csw/bin:/opt/csw/gnu:$PATH
```

Set GNU Make as the default:
```console
    $ sudo ln -s /usr/bin/gmake /usr/bin/make
```

Clone node:
```console
    $ git clone https://github.com/nodejs/node.git
```

Install gcc 49:
```console
    $ sudo pkg install --accept --license gcc-49
    $ gmake CXXFLAGS+="--function-sections -fdata-sections"
```

Install stdc++6:
```console
    $ sudo pkgchk -L CSWlibstdc++6
    $ export LD_LIBRARY_PATH=/opt/csw/lib/:$LD_LIBRARY_PATH
```

Patches:
```console
danbev@solaris:~/work/node$ git show 8ff7afd
commit 8ff7afd2aba1cc13348e5d639f292b2fbb3b86d0
Author: Daniel Bevenius <daniel.bevenius@gmail.com>
Date:   Thu Mar 16 06:59:05 2017 +0100

    add include ldflag for solaris
    
    It looks like when cctest is to be compiled the includes are
    missing. Trying to add the SHARED_INTERMEDIATE_DIR and see if
    that fixes one of the includes. If that works I'll add the rest
    if required.

diff --git a/node.gyp b/node.gyp
index 2407844..4bd3769 100644
--- a/node.gyp
+++ b/node.gyp
@@ -667,6 +667,9 @@
             ]},
           ],
         }],
+        ['OS=="solaris"', {
+          'ldflags': [ '-I<(SHARED_INTERMEDIATE_DIR)' ]
+        }],
       ]
     }
   ], # end targets
```
```console
danbev@solaris:~/work/node$ git show 68c396b
commit 68c396be00896801bb92b5705da240d9aef30890
Author: Daniel Bevenius <daniel.bevenius@gmail.com>
Date:   Thu Mar 16 12:21:24 2017 +0100

    fix for building on solaris

diff --git a/common.gypi b/common.gypi
index 3aad8e7..e001a1d 100644
--- a/common.gypi
+++ b/common.gypi
@@ -282,7 +282,8 @@
       }],
       [ 'OS in "linux freebsd openbsd solaris android aix"', {
         'cflags': [ '-Wall', '-Wextra', '-Wno-unused-parameter', ],
-        'cflags_cc': [ '-fno-rtti', '-fno-exceptions', '-std=gnu++0x' ],
+        #'cflags_cc': [ '-fno-rtti', '-fno-exceptions', '-std=gnu++0x' ],
+        'cflags_cc': [ '-fno-rtti', '-fno-exceptions' ],
         'ldflags': [ '-rdynamic' ],
         'target_conditions': [
           # The 1990s toolchain on SmartOS can't handle thin archives.
@@ -323,6 +324,8 @@
             'ldflags': [ '-m64', '-march=z196' ],
           }],
           [ 'OS=="solaris"', {
+            'defines': ['_GLIBCXX_USE_C99_MATH'],
+            'cflags_cc': [ '-std=c++11' ],
             'cflags': [ '-pthreads' ],
             'ldflags': [ '-pthreads' ],
             'cflags!': [ '-pthread' ],
```



Build and run node tests:
```console
    $ gmake cctest
```


### Building on Windows
I used VirtualBox to build and run the test suite on Windows
```console
    $ .\vcbuild test
```


### Debugging 
```console
    $ lldb -- out/Debug/cctest
    (lldb) r
    [ RUN      ] EnvironmentTest.MultipleEnvironmentsPerIsolate
    cctest(86419,0x7fff799ca000) malloc: *** error for object 0x104401058: incorrect checksum for freed object - object was probably modified after being freed.
*** set a breakpoint in malloc_error_break to debug
Process 86419 stopped
* thread #1: tid = 0x2bd84d0, 0x00007fff97229f06 libsystem_kernel.dylib`__pthread_kill + 10, queue = 'com.apple.main-thread', stop reason = signal SIGABRT
    frame #0: 0x00007fff97229f06 libsystem_kernel.dylib`__pthread_kill + 10
libsystem_kernel.dylib`__pthread_kill:
->  0x7fff97229f06 <+10>: jae    0x7fff97229f10            ; <+20>
    0x7fff97229f08 <+12>: movq   %rax, %rdi
    0x7fff97229f0b <+15>: jmp    0x7fff972247cd            ; cerror_nocancel
    0x7fff97229f10 <+20>: retq

    (lldb) memory read 0x104401058


    (lldb) jlh handle
    0x1872340551b1: [Symbol]
     - hash: 60730982
     - name: 0x187234028391 <String[14]: node:npnBuffer>
     - private: 1


(lldb) jlh result
0x1e9a8f728239: [Symbol]
 - hash: 171677908
 - name: 0x1e9a8f728211 <String[15]: node:alpnBuffer>
 - private: 1
```


### Switching between clang and gcc
```console
    CXX=g++ CXX.host=g++ && ./configure -- -Dclang=0.
```

### [Ninja](https://ninja-build.org/)

Macosx:
```console
 
    $ ./configure --ninja
    $ ninja -C out/Release

    $ ./configure && tools/gyp_node.py -f ninja && ninja -C out/Release && ln -fs out/Release/node node
```

This will generate object files in `out/Release/src/` but the names will be `obj/src/node.node.o` instead of `obj/src/node/node.o`. This matters
as we generete object files for different operating systems. 

If you take a look in out/Release/ninja.build you'll find a bunch of subninja commands which are used to include other ninja build files:
  
    ....
    subninja obj/node.ninja

#### Building with Ninja linux
```console
    $ dnf install ninja-build
    $ ./configure --debug --ninja
    $ ninja-build -C out/Release
    $ out/Release/cctest
```

#### Building with Ninja on windows
You'll need to install Visual Studion 2015 and make sure you select Common Tools for C++.

Open cmd with administrator priveliges (Start -> Search for cmd ->CTRL+SHIFT+ENTER):
```console
    > python configure --debug --ninja --dest-cpu=x86 --without-intl
    > tools\gyp_node.py -f ninja
    > ninja -C out\Release
    > out\Release\cctest.exe
```

### Linux getauxval
A good description of this can be found here:
https://lwn.net/Articles/519085/

The getauxval is a function that was added in glibc 2.16
```c
    #if defined(__linux__) && defined(__GLIBC__) && defined(__GLIBC_PREREQ)
    # if __GLIBC_PREREQ(2, 16)
    #   define HAS_GETAUXVAL 1
    #   include <sys/auxv.h>
    # endif //  HAS_GETAUXVAL
    #endif

    # LD_SHOW_AUXV=1 ./node
    AT_SYSINFO_EHDR: 0x7fff503ee000
    AT_HWCAP:        9f8bfbff
    AT_PAGESZ:       4096
    AT_CLKTCK:       100
    AT_PHDR:         0x400040
    AT_PHENT:        56
    AT_PHNUM:        9
    AT_BASE:         0x7f223f1d6000
    AT_FLAGS:        0x0
    AT_ENTRY:        0x859c90
    AT_UID:          0
    AT_EUID:         0
    AT_GID:          0
    AT_EGID:         0
    AT_SECURE:       0
    AT_RANDOM:       0x7fff503dddc9
    AT_EXECFN:       ./node
    AT_PLATFORM:     x86_64
```

To force the setting of AT_SECURE:
```console

    $ setcap cap_net_raw+ep out/Release/node
    $ getcap out/Release/node
    $ useradd beve
    $ su beve
    $ ./out/Release/node
```

#### Show all v8 exceptions
```console
    $ out/Release/node --print_all_exceptions /work/node/test/parallel/test-process-setuid-setgid.js
```

#### Creating patches
I've found that I need to create patches from old patches that worked with a previous version. At work we 
apply patches to node sources and when new versions are released these patches might not apply cleanly making
it a manual task to make updates to that tag and then generating the patches to be applied.
```console

   $ patch -p1 < 000x-something.patch
   $ git diff --patch-with-stat > patch.out
```
Then I just copy this and replace the original patch section, but keeping the rest of the original patch.
Then try to apply the patch again to make sure it applied cleanly


#### Cherry pick a V8 commit
```console
    $ git format-patch -1 --stdout f5fad6d > out.patch
    $ git am --directory deps/v8  ~/work/google/javascript/v8/verbose.patch
```

Don't forget to bump the patch version in deps/v8/include/v8-version.h
Also common common.gypi needs to be updated as well:
```console
'v8_embedder_string': '-node.5',
```
#### HTTP/2
```javascript
    const server = h2.createServer();
```

`createServer` is defined in `lib/internal/http2/core.js` and it returns a 
new Http2Server.
Http2Server extends Server in net.js. After calling the super classes constructor
the following call is performed:

```javascript
    this.on('newListener', setupCompat);
```

setupCompat is a event listener callback which only handles 'request' events, 
in which case it 


### Messages/Exceptions with V8
Each V8 Isolate allows listeners to be added for various error messages/exceptions.

```c++
    isolate->AddMessageListener(OnMessage);
    isolate->SetFatalErrorHandler(OnFatalError);
    isolate->SetAbortOnUncaughtExceptionCallback(ShouldAbortOnUncaughtException);
```

`AddMessageListener` is a callback that will be called when an error occurs. How does this work then?

When a script is run, for example:
```javascript
    const char *js = "age = ajj40";  // intentionally trigger an exception
    Local<String> source = String::NewFromUtf8(isolate, js, NewStringType::kNormal).ToLocalChecked();
    Local<Script> script = Script::Compile(context, source).ToLocalChecked();
    Local<Value> result = script->Run(context).ToLocalChecked();
```

Script::Run can be found in api.cc 
```console
    (lldb) br s -f api.cc -l 2017
```

```c++
    has_pending_exception = !ToLocal<Value>(i::Execution::Call(isolate, fun, receiver, 0, nullptr), &result);
```

See the returned value is a bool indicating if there was an error. 
`i::Execution::Call` will call:

```c++
    return CallInternal(isolate, callable, receiver, argc, argv, MessageHandling::kReport)
```

Notice that in the MessageHandling::kReport being passed in. CallInternal looks like this (in our case):

```c++
    return Invoke(isolate, false, callable, receiver, argc, argv, isolate->factory()->undefined_value(), message_handling);

    MUST_USE_RESULT MaybeHandle<Object> Invoke(
        Isolate* isolate, bool is_construct, Handle<Object> target,
        Handle<Object> receiver, int argc, Handle<Object> args[],
        Handle<Object> new_target, Execution::MessageHandling message_handling)
```

In our case the target is a JavaScript Function which can be checked by:
```console
    (lldb) p target->IsJSFunction()
    (bool) $58 = true
```

This will cause us to enter the following block in the Invoke function:

```c++
    if (target->IsJSFunction()) {
      Handle<JSFunction> function = Handle<JSFunction>::cast(target);
```
     
But there is a check:

```c++
   if ((!is_construct || function->IsConstructor()) &&
        function->shared()->IsApiFunction()) {
```
 
which causes us to exit the block.

```c++
    // Placeholder for return value.
    Object* value = NULL;

    typedef Object* (*JSEntryFunction)(Object* new_target, Object* target,
                                      Object* receiver, int argc,
                                      Object*** args);
```

So we are declaring a JSEntryFunction which is of type pointer to function that returns a pointer to Object
and takes the arguments listed.
    
```c++
    Handle<Code> code = is_construct
      ? isolate->factory()->js_construct_entry_code()
      : isolate->factory()->js_entry_code();
```

How are js_entry_code() set? 
See `Heap objects` for details.

```c++
    JSEntryFunction stub_entry = FUNCTION_CAST<JSEntryFunction>(code->entry());

    if (FLAG_clear_exceptions_on_js_entry) isolate->clear_pending_exception();

    // Call the function through the right JS entry stub.
    Object* orig_func = *new_target;
    Object* func = *target;
    Object* recv = *receiver;
    Object*** argv = reinterpret_cast<Object***>(args);
    if (FLAG_profile_deserialization && target->IsJSFunction()) {
      PrintDeserializedCodeInfo(Handle<JSFunction>::cast(target));
    }
    RuntimeCallTimerScope timer(isolate, &RuntimeCallStats::JS_Execution);
    value = CALL_GENERATED_CODE(isolate, stub_entry, orig_func, func, recv, argc, argv);
```
```console
    (lldb) br s -f execution.cc -l 145
```

CALL_GENERETED_CODE is a macro which is used to call code generated by one of the compilers. 
I'm on a x64 some guessing that src/x64/simulator-x64.h is getting called which does:

```c++
    // TODO(X64): Don't pass p0, since it isn't used?
   #define CALL_GENERATED_CODE(isolate, entry, p0, p1, p2, p3, p4) \
   (entry(p0, p1, p2, p3, p4))
```

Setting a breakpoint after this call we can inspect the value returned:
```console
    (lldb) job value
    #exception
```

```c++
    // Update the pending exception flag and return the value.
    bool has_exception = value->IsException(isolate);
    DCHECK(has_exception == isolate->has_pending_exception());
    if (has_exception) {
      if (message_handling == Execution::MessageHandling::kReport) {
        isolate->ReportPendingMessages();
      }
      return MaybeHandle<Object>();
    } else {
      isolate->clear_pending_message();
    }
```

Remember that we passed Execution::MessageHandling::kReport earlier so we will call isolate-ReportPendingMessages()

```c++
    void Isolate::ReportPendingMessages() {
      DCHECK(AllowExceptions::IsAllowed(this));

      Object* exception = pending_exception();
```

pending_exception() performs some checks and then returns:

```c++
    return thread_local_top_.pending_exception_;
```
```console
    (lldb) job exception
    0x94c7c108af1: [JS_ERROR_TYPE]
    - map = 0xcc576b8e179 [FastProperties]
    - prototype = 0x1993f4b15221
    - elements = 0x37d706602241 <FixedArray[0]> [FAST_HOLEY_SMI_ELEMENTS]
    - properties = 0x94c7c108ec9 <FixedArray[3]> {
     #stack: 0x3aeec9e69bf1 <AccessorInfo> (const accessor descriptor)
     #message: 0x94c7c108ac9 <String[20]: ajj40 is not defined> (data field 0)
     0x37d706604d71 <Symbol: stack_trace_symbol>: 0x94c7c109099 <JSArray[6]> (data field 1)
```

```c++

    // Try to propagate the exception to an external v8::TryCatch handler. If
    // propagation was unsuccessful, then we will get another chance at reporting
    // the pending message if the exception is re-thrown.
    bool has_been_propagated = PropagatePendingExceptionToExternalTryCatch();
    if (!has_been_propagated) return;
```

Looking into PropagatePendingExceptionToExternalTryCatch:

```c++
    bool Isolate::PropagatePendingExceptionToExternalTryCatch() {
      Object* exception = pending_exception();
```

The first thing it does is again call pending_exceptions() invoking the same checks are before
which should really be the same. I wonder if there might be use for an overloaded function taking
a pointer to exception?

```c++
    HandleScope scope(this);
    Handle<JSMessageObject> message(JSMessageObject::cast(message_obj), this);
    Handle<JSValue> script_wrapper(JSValue::cast(message->script()), this);
    Handle<Script> script(Script::cast(script_wrapper->value()), this);
    int start_pos = message->start_position();
    int end_pos = message->end_position();
    MessageLocation location(script, start_pos, end_pos);
    MessageHandler::ReportMessage(this, &location, message);
```

`MessageHandler::ReportMessage` prepares the error message (very simplified) and ends up calling:

```c++
    ReportMessageNoExceptions(isolate, loc, message, api_exception_obj);


    v8::MessageCallback callback =FUNCTION_CAST<v8::MessageCallback>(callback_obj->foreign_address());
    Handle<Object> callback_data(listener->get(1), isolate);
    {
      // Do not allow exceptions to propagate.
      v8::TryCatch try_catch(reinterpret_cast<v8::Isolate*>(isolate));
      callback(api_message_obj, callback_data->IsUndefined(isolate) ? api_exception_obj : v8::Utils::ToLocal(callback_data));
    }
```

This is the call that invokes are message callback. After this we will back out of the frame stacks. On thing I notice that there
was an ExceptionScope which the deconstructor sets the exception object. Need to look into why this is done.
So we will eventually end up back where in v8::Script::Run:

```c++
    has_pending_exception = !ToLocal<Value>(i::Execution::Call(isolate, fun, receiver, 0, nullptr), &result);

    (lldb) p has_pending_exception
    (bool) $208 = true

     RETURN_ON_FAILED_EXECUTION(Value);
     RETURN_ESCAPED(result);
```

Lets take a closer look at RETURN_ON_FAILED_EXECUTION(Value):

```c++
#define RETURN_ON_FAILED_EXECUTION(T) \
  EXCEPTION_BAILOUT_CHECK_SCOPED(isolate, MaybeLocal<T>())

So that will be EXCEPTION_BAILOUT_CHECK_SCOPED(isolate, MaybeLocal<Value>()) which becomes:

    #define EXCEPTION_BAILOUT_CHECK_SCOPED(isolate, value) \
      do {                                                 \
        if (has_pending_exception) {                       \
          call_depth_scope.Escape();                       \
          return value;                                    \
        }                                                  \
     } while (false)
```

So in our case the value returned will be a MaybeLocal<Value> object.
This is what gets returned here:

```c++
    Local<Value> result = script->Run(context).ToLocalChecked();
```

Now, notice that we are calling ToLocalChecked(), which 

```c++
    if (V8_UNLIKELY(val_ == nullptr)) V8::ToLocalEmpty();

    Utils::ApiCheck(false, "v8::ToLocalChecked", "Empty MaybeLocal.");

    if (!condition) Utils::ReportApiFailure(location, message);


    FatalErrorCallback callback = isolate->exception_behavior();

    callback(location, message);
```

And this is where our OnFatalError function is called.
But without the ToLocalChecked our OnFatalError would not be called.




Well, during invokation of a Script, recall from above it was mentioned that we'll end up 
in `CALL_GENERATED_CODE`. Now, if we set a breakpoint in file = 'isolate.cc', line = 1062 
which is the Throw function we can backtrace and see how we ended up there.
ic.cc:2395:

```c++
    v8::internal::Runtime_LoadGlobalIC_Miss
    Handle<Object> result;
    ASSIGN_RETURN_FAILURE_ON_EXCEPTION(isolate, result, ic.Load(name));
    return *result;

    MaybeHandle<Object> LoadIC::Load(Handle<Object> object, Handle<Name> name)
```
This will throw an ReferenceError in our case:

```c++
    return ReferenceError(name);

    THROW_NEW_ERROR(isolate(), NewReferenceError(MessageTemplate::kNotDefined, name), Object);
```

That call will bring us back to isolate.cc (1064) which is the Throw function we breaked in.
This time we have a try_catch_handler:
```console
    (lldb) p *try_catch_handler()
    (v8::TryCatch) $2 = {
      isolate_ = 0x0000000105000000
      next_ = 0x0000000000000000
      exception_ = 0x00003107fe782351
      message_obj_ = 0x00003107fe782351
      js_stack_comparable_address_ = 0x00007fff5fbfea50
      is_verbose_ = true
      can_continue_ = true
      capture_message_ = true
      rethrow_ = false
      has_terminated_ = false
    }
```

ReportPendingMessages:

```c++
    // Determine whether the message needs to be reported to all message handlers
    // depending on whether and external v8::TryCatch or an internal JavaScript
    // handler is on top.
    bool should_report_exception;
    if (IsExternalHandlerOnTop(exception)) {
      // Only report the exception if the external handler is verbose.
      should_report_exception = try_catch_handler()->is_verbose_;
    } else {
      // Report the exception if it isn't caught by JavaScript code.
      should_report_exception = !IsJavaScriptHandlerOnTop(exception);
    }
```

Is this case we have a TryCatch RAII and have set verbose to true. There is no javascript
try/catch so the MessageHandler we set by calling AddMessageHandler on the isolate should
be the target handler.


`SetFatalErrorHandler`
If this callback handler is not set then the default behaviour in V8 will be to  
print a stactrace and exit, for example:
```console
    $ ./exceptions
    OnMessage...

    #
    # Fatal error in v8::ToLocalChecked
    # Empty MaybeLocal.
    #

    Received signal 4 <unknown> 000109100351

    ==== C stack trace ===============================

     [0x00010910322e]
     [0x000109103265]
     [0x000109103186]
     [0x7fff8a2c252a]
     [0x000107ac8423]
     [0x00010624ffbc]
     [0x000106254bef]
     [0x000106254c1d]
     [0x00010621d4dc]
     [0x7fff86cc45ad]
     [0x000000000001]
    [end of stack trace]
    Illegal instruction: 4
```

You can set a handler for this that does something before exiting of choose to not exit.
    
When an exceptions is raised in V8 it will cause the function Throw in e

```c++
    Object* Isolate::Throw(Object* exception, MessageLocation* location) {
```

#### TryCatch
Is used to create a try/catch block in V8 which used RAII:
```c++
    {
      TryCatch try_catch(isolate);
      try_catch.SetVerbose(true);

      ... perform operations

      if (try_catch.HasCaught()) {
        printf("Caught: %s\n", *String::Utf8Value(try_catch.Exception()));
      }
```

How does setting `verbose` affect things?
In Isolate::Throw:
Which can be called if there is a JavaScript error and if an external TryCatch exist and 
verbose is true. If that is the case the Message will be created:

```c++
    Handle<Object> message_obj = CreateMessage(exception_handle, location);
    thread_local_top()->pending_message_obj_ = *message_obj;
    ...
    set_pending_exception(*exception_handle);
    return heap()->exception();
```

So that is what throw does, sets the message_obj and the exception_handle and then returns.

What if we set verbose to false:
```console
    (lldb) expr try_catch.SetVerbose(false);
```

In this case the message will still be created and the exception set on the thread_local, but 
when we get to ReportPendingMessages is_verbose_ will be false and the callback will be skipped.

In `src/api.cc` (line 2710) we can find the TryCatch constructor:
```c++
v8::TryCatch::TryCatch(v8::Isolate* isolate)
    : isolate_(reinterpret_cast<i::Isolate*>(isolate)),
      next_(isolate_->try_catch_handler()),
      is_verbose_(false),
      can_continue_(true),
      capture_message_(true),
      rethrow_(false),
      has_terminated_(false) {
  ResetInternal();
  ...
  isolate_->RegisterTryCatchHandler(this);
}
`ResetInternal` will do the following:
```c++
  i::Object* the_hole = isolate_->heap()->the_hole_value();
  exception_ = the_hole;
  message_obj_ = the_hole;
```
```console
(lldb) p the_hole
(v8::internal::Object *) $4 = 0x0000278095682321
```
```
`RegisterTryCatchHandler` will do:
```c++
thread_local_top()->set_try_catch_handler(that);
```

After this call we can inspect thread_local_top_:
```c++
(lldb) p *thread_local_top()
(v8::internal::ThreadLocalTop) $22 = {
  isolate_ = 0x0000000104806e00
  context_ = 0x000025b6fbf03af1
  thread_id_ = (id_ = 1)
  pending_exception_ = 0x000025b693f02321
  wasm_caught_exception_ = 0x0000000000000000
  pending_handler_context_ = 0x0000000000000000
  pending_handler_code_ = 0x0000000000000000
  pending_handler_offset_ = 0
  pending_handler_fp_ = 0x0000000000000000 <no value available>
  pending_handler_sp_ = 0x0000000000000000 <no value available>
  rethrowing_message_ = false
  pending_message_obj_ = 0x000025b693f02321
  scheduled_exception_ = 0x000025b693f02321
  external_caught_exception_ = false
  save_context_ = 0x0000000000000000
  c_entry_fp_ = 0x0000000000000000 <no value available>
  handler_ = 0x0000000000000000 <no value available>
  c_function_ = 0x0000000000000000 <no value available>
  promise_on_stack_ = 0x0000000000000000
  js_entry_sp_ = 0x0000000000000000 <no value available>
  external_callback_scope_ = 0x0000000000000000
  current_vm_state_ = EXTERNAL
  failed_access_check_callback_ = 0x0000000000000000
  try_catch_handler_ = 0x00007fff5fbfde60
}
```
Notice that `pending_exception`, `pending_message_obj_`, and `scheduled_exception_` are all set to `the_hole`. The `try_catch_handler_` is also
set.
So these are memory locations, and from what I understand so far is that these are made available to generated code, enabling them
to set values, like pending_message_obj_, but also have access to the try_catch_handler_.
This means that a generated piece of code would directly call/jump to try_catch_handler_ and have it execute. But what sets this up?  
When `CALL_GENERATED_CODE(isolate, stub_entry, orig_func, func, recv, argc, argv);` is called the stub_entry will point to the code generated
for `JSEntryStub`.  This will place the correct values in the expected registers.

For example, take the following from the generated assemble (created using (lldb) job *code):
```console
movq r10,0x104808790    ;; external reference (Isolate::context_address)
movq r10,0x104808790    ;; external reference (Isolate::context_address)
```
This instruction is generated by `src/x64/code-stubs-x64.cc` `JSEntryStub::Generate`:
```c++
ExternalReference context_address(IsolateAddressId::kContextAddress, isolate());
__ Load(kScratchRegister, context_address);o
__ Push(kScratchRegister);  // context
```
The above will set `ExternalReference` address_:
```c++
ExternalReference::ExternalReference(IsolateAddressId id, Isolate* isolate)
    : address_(isolate->get_address_from_id(id)) {}
```
And `get_address_from_id` look like this:
```c++
Address Isolate::get_address_from_id(IsolateAddressId id) {
  return isolate_addresses_[id];
}
```
```c++
Move(kScratchRegister, source);
```
```console
(lldb) expr kScratchRegister
(const v8::internal::Register) $7 = {
  v8::internal::RegisterBase<v8::internal::Register, 16> = (reg_code_ = 10)
}
```
We can find the following function in `src/x64/macro-assembler-x64.h`:
```c++
void Move(Register dst, ExternalReference ext) {
  movp(dst, reinterpret_cast<void*>(ext.address()), RelocInfo::EXTERNAL_REFERENCE);
}
```
Notice the `RelocInfo::EXTERNAL_REFERENCE` which indicates that this is the address of an
external C++ function

```console
(lldb) p ext.address()
(v8::internal::Address) $8 = 0x0000000105813b50 <no value available>
```

To understand how this works we can set a break point and run mkshnapshot:
```console
$ $ lldb out.gn/learning/mksnapshot
(lldb) br s -f code-stubs-x64.cc -n JSEntryStub::Generate
(lldb) expr StackFrame::TypeToMarker(type())
(int32_t) $1 = 2                                            // pushq  $0x2
```
```console
0x3db66db84066  REX.W movq r10,0x104808790    ;; external reference (Isolate::context_address)
0x3db66db84070: movq   (%r10), %r10
```
Notice that `0x104808790` is a pointer. This is then dereferenced and that value placed into r10:
```console
(lldb) register read r10
     r10 = 0x0000000104808790
(lldb) memory read -f x -c 1 -s 8 `($r10)`
0x104808790: 0x000025b6fbf03af1
```
So it contains the value `0x000025b6fbf03af1` and if we look at the `context_` entry from above we can see
these match:
```console
context_ = 0x000025b6fbf03af1
```

One thing I noticed in x64/code-stubs-x64.cc JSEntry function was:
```c++
__ bind(&handler_entry);
handler_offset_ = handler_entry.pos();
```
Now, `handler_offset_` is a private member of `JSEntryStub` in `src/code-stubs.h`. This field is set
by `FinishCode`:
```c++
Handle<Code> new_object = GenerateCode();
new_object->set_stub_key(GetKey());
FinishCode(new_object);
```
Now, `handler_entry` is a label and this gets the position of the label. I'm guessing (at the moment) that the `handler_offet_` allows other 
generated functions to addess the exception handler.


deps/v8/src/x64/macro-assembler-x64.cc `EnterFrame`. Just note that when Builtins::Generate_JSEntryTrampoline generates 
code it calls `EnterFrame` which does more than store/set rbp. Remember this or the code will be hard to follow.

### Compiling on windows
I looking into an issue on windows:
https://github.com/nodejs/node/issues/12952

The issue is a link error with the cctest target. In this particular case on windows it was built using:
```console

    .\vcbuild.bat dll debug x64 vc2015

This will result in the following options:
```console
    configure --debug --shared --dest-cpu=x64 --tag=
```

So this is saying that node should be created as a shared library but
not any of its dependencies. In this case the focus is on OpenSSL which
will be a static library.
The link error is:
```console
    openssl.lib(err.obj) : error LINK2005: ERR_put_error already defined in node.lib(node.dll
```

We can find the exported symbols of Debug/node.dll using:
```console
    dumpbin /EXPORTS Debug/node.dll > node-exports
```

We can find the ERR_put_error is indeed exported from node.dll
This is done because in node.gyp we have:
```

      [ 'OS=="win" and '
        'node_use_openssl=="true" and '
        'node_shared_openssl=="false" and node_shared=="false"', {
        'use_openssl_def': 1,
      }, {
        'use_openssl_def': 0,
      }],
```

In our case use_openssl_def will be 1 which is the used in node.gypi:
```

      # openssl.def is based on zlib.def, zlib symbols
      # are always exported.
      ['use_openssl_def==1', {
        'sources': ['<(SHARED_INTERMEDIATE_DIR)/openssl.def'],
      }],
      ['OS=="win" and use_openssl_def==0', {
        'sources': ['deps/zlib/win32/zlib.def'],
```

A .def file is a module definition file which describes attributes of a DLL. In our case openssl.def can be found in Debug\obj\global_intermediate:
```
    EXPORTS
    ...
    ERR_put_error
```

So we can see this function is indeed exported, so if we link against openssl.lib 
this symbol (and the others) will be duplicates.


### V8 Platform
This is used by an embedder and is an interface that must be implemented. This interface is defined in deps/v8/include/v8-platform.h.
This interface has a number of functions related to threading and running task on threads:
* CallOnBackgroundThread
* CallOnForegroundThread
* CallDelayedOnForegroundThread
...
* AddTraceEvent

Node can be configured --without-v8-platform which will set environment variables that will be picked up by the pre-processor. For example, if you
take a look in src/node.cc you will find:

```c++
    #if NODE_USE_V8_PLATFORM
    #include "libplatform/libplatform.h"
    #endif  // NODE_USE_V8_PLATFORM
```

libplatform/libplatform.h is the interface for a default v8::Platform implementation in V8.
Next, we have a v8_platform struct that will use the above default v8::Platform implementation is NODE_USE_V8_PLATFORM is set.
But what does it mean to not use the default v8::Platform implementation?
Basically this will just create noop functions or functions that throw an error:

```c++
    #else  // !NODE_USE_V8_PLATFORM
    void Initialize(int thread_pool_size) {}
    void PumpMessageLoop(Isolate* isolate) {}
    void Dispose() {}
    bool StartInspector(Environment *env, const char* script_path,
                      const node::DebugOptions& options) {
      env->ThrowError("Node compiled with NODE_USE_V8_PLATFORM=0");
      return true;
    }

    void StartTracingAgent() {
      fprintf(stderr, "Node compiled with NODE_USE_V8_PLATFORM=0, "
                      "so event tracing is not available.\n");
    }
    void StopTracingAgent() {}
```


When trying to run a test --without-v8-platform I run into the following error:
```console

    #
    # Fatal error in ../deps/v8/src/v8.cc, line 110
    # Check failed: platform_.
    #

    ==== C stack trace ===============================

      0   node                                0x00000001019d2dae v8::base::debug::StackTrace::StackTrace() + 30
      1   node                                0x00000001019d2de5 v8::base::debug::StackTrace::StackTrace() + 21
      2   node                                0x00000001019ccf84 V8_Fatal + 452
      3   node                                0x00000001012bcea8 v8::internal::V8::GetCurrentPlatform() + 72
      4   node                                0x0000000100ed0c7c v8::internal::Isolate::Init(v8::internal::Deserializer*) + 1692
      5   node                                0x00000001012965ba v8::internal::Snapshot::Initialize(v8::internal::Isolate*) + 170
      6   node                                0x0000000100260265 v8::Isolate::New(v8::Isolate::CreateParams const&) + 485
      7   node                                0x000000010170901f node::Start(uv_loop_s*, int, char const* const*, int, char const* const*) + 79
      8   node                                0x0000000101708bce node::Start(int, char**) + 478
      9   node                                0x000000010175f21e main + 94
      10  node                                0x0000000100000a34 start + 52
      11  ???                                 0x0000000000000002 0x0 + 2
    Process 9882 stopped


    2644   compiler_dispatcher_ =
    2645       new CompilerDispatcher(this, V8::GetCurrentPlatform(), FLAG_stack_size);
```

v8::GetCurrentPlatform will perform the following:

```c++
    v8::Platform* V8::GetCurrentPlatform() {
      DCHECK(platform_);
      return platform_;
   }
```

But since we have not set a platform, remember that would have been done if NODE_USE_V8_PLATFORM:

```c++
    #if NODE_USE_V8_PLATFORM
    void Initialize(int thread_pool_size) {
      platform_ = v8::platform::CreateDefaultPlatform(thread_pool_size);
      V8::InitializePlatform(platform_);
      tracing::TraceEventHelper::SetCurrentPlatform(platform_);
    }
```

My understanding is that the in this code path the snapshot blob is being deserialized and when 
this is done.


### Upgrading OpenSSL

Download and verify the download:
```console

    $ gpg openssl-1.0.2l.tar.gz.asc
    gpg: Signature made Thu May 25 14:55:41 2017 CEST using RSA key ID 0E604491
    gpg: Can't check signature: public key not found
    $ gpg --keyserver pgpkeys.mit.edu --recv-key 0E604491
    gpg: requesting key 0E604491 from hkp server pgpkeys.mit.edu
    gpg: key 0E604491: public key "Matt Caswell <matt@openssl.org>" imported
    gpg: 3 marginal(s) needed, 1 complete(s) needed, PGP trust model
    gpg: depth: 0  valid:   1  signed:   0  trust: 0-, 0q, 0n, 0m, 0f, 1u
    gpg: Total number processed: 1
    gpg:               imported: 1  (RSA: 1)
    $ gpg openssl-1.0.2l.tar.gz.asc
    gpg: Signature made Thu May 25 14:55:41 2017 CEST using RSA key ID 0E604491
    gpg: Good signature from "Matt Caswell <matt@openssl.org>"
    gpg:                 aka "Matt Caswell <frodo@baggins.org>"
    gpg: WARNING: This key is not certified with a trusted signature!
    gpg:          There is no indication that the signature belongs to the owner.
    Primary key fingerprint: 8657 ABB2 60F0 56B1 E519  0839 D9C4 D26D 0E60 4491
```


Compare the changes to make see if you have to make changes to the

### OpenSSL with FIPS
When building the OpenSSL and specifying `--openssl-fips`, which is described as 
"Build OpenSSL using FIPS canister .o file in supplied folder"
[Fipsld](https://wiki.openssl.org/index.php/Fipsld_and_C%2B%2B). 

The canister file is just the collection of the all the source code into an single 
monolihic object module to guarentee the relative order of the original code
components:
```console
$ ld -r ­-o fipscanister.o fips_start.o ... fips_end.o
```
Doing this will preserve the relative order of the original code components.

The follwing can be seen on that page:
"When performing a static link against the OpenSSL library, you have to embed 
the expected FIPS signature in your executable after final linking. Embedding 
the FIPS signature in your executable is most often accomplished with fisld."
"fisld will take the place of the linker (or compiler if invoking via a compiler driver). 
If you use fisld to compile a source file, fisld will do nothing and simply 
invoke the compiler you specify through FIPSLD_CC. When it comes time to link, 
fisld will compile fips_premain.c, add fipscanister.o, and then perform the 
final link of your program. Once your program is linked, fisld will then invoke 
incore to embed the FIPS signature in your program."

But how about when we dynamically link to an OpenSSL version that has FIPS support
enabled. Since we would not be statically linking we should not have to specify
the fipscanister.o. But without specifying the `--openssl-fips` flag the options
to enable/disable FIPS are not avaiable in node (nore are functions enabled/disabled
that would otherwise be when when FIPS mode is enabled).

Perhaps we can set the macro `NODE_FIPS_MODE` if `OPENSSL_FIPS` is set which should
be done when linking against an OpenSSL version that support FIPS?


For instructions building an OpenSSL version with FIPS support see:
https://github.com/danbev/learning-libcrypto#fips

Detecting FIPS support:
Currently, FIPS support is not available in the version that is shipped with
Node.js. But we still have the option to dynamically link to a FIPS compatible
OpenSSL library. This is for example what we do at Red Hat.
There is a problem here though...

If we run the following command on default build (statically linking with OpenSSL
in deps, so no FIPS support):
```console
$ ./node -p "require('crypto').getFips()"
0
```
This expected. 

Node.js can also be dynamically linked with OpenSSL, for example:
```console
$ ./configure --shared-openssl --openssl-is-fips
```
The `--openssl-is-fips` option specifies that the OpenSSL library is FIPS compatible
and this enables FIPS related functionality in Node to be available. 
Building and running the same command on this version gives:
```console
$./node -p "require('crypto').getFips()"
0
```
Now this is not expected. 

None-FIPS configuration:
```console
$ node -p 'process.config.variables' | grep openssl
  node_shared_openssl: false,
  node_use_openssl: true,
  openssl_fips: '',
  openssl_is_fips: false,
```

FIPS Compatible configuration:
```console
./node -p 'process.config.variables' | grep openssl
  node_shared_openssl: true,
  node_use_openssl: true,
  openssl_fips: '',
  openssl_is_fips: true,
  openssl_system_ca_path: '/etc/pki/tls/certs/ca-bundle.crt',
```

```console
 ./node --expose-internals -p "require('internal/test/binding').internalBinding('config')"
{
  isDebugBuild: false,
  hasOpenSSL: true,
  fipsMode: true,
  hasIntl: true,
  hasSmallICU: true,
  hasTracing: true,
  hasNodeOptions: true,
  hasInspector: true,
  noBrowserGlobals: false,
  bits: 64,
  hasDtrace: true,
  hasCachedBuiltins: true
}
We can see that fipsMode has be set which is also correct. But why does getFips()
return 0? Because you have to also enable fips:
```console
$ ./node --enable-fips -p "require('crypto').getFips()"
1
```

Enable fips in a container (UBI8/RHEL8):
```console
$ update-crypto-policies --set FIPS
$ fips-mode-setup --enable --no-bootcfg
$ export OPENSSL_FORCE_FIPS_MODE=true
```

#### OpenSSL FIPS Object Module
OpenSSL itself is not validated, and never will be. Instead a carefully defined software component
called the OpenSSL FIPS Object Module has been created. The Module was designed for
compatibility with the OpenSSL library so products using the OpenSSL library and API can be
converted to use FIPS 140-2 validated cryptography with minimal effort.

FIPS Mode in which the FIPS approved algorithms are implemented by
the FIPS Object Module and non-FIPS approved algorithms are disabled by default. 
These non-validated algorithms include, but are not limited to, Blowfish, CAST, 
IDEA, RC-family, and non-SHA message digest and other algorithms.

The v1.2.x Module is only compatible with OpenSSL 0.9.8 releases, while the v2.0 Module is
compatible with OpenSSL 1.0.1 and 1.0.2 releases. The v2.0 Module is the best choice for all new
software and product development.

After following the [instructions](https://github.com/danbev/learning-libcrypto#fips) to configure OpenSSL with FIPS support, building Node can be done using the following commands:

```console
    $ ./configure --debug --shared-openssl --shared-openssl-libpath=/Users/danielbevenius/work/security/build_1_0_2k/lib --shared-openssl-includes=/Users/danielbevenius/work/security/build_1_0_2k/include --openssl-fips=/Users/danielbevenius/work/security/build_1_0_2k/
    $ make -j8 test
```

#### Crypto init

```c++
    void InitCrypto(Local<Object> target,
                Local<Value> unused,
                Local<Context> context,
                void* priv) {
    static uv_once_t init_once = UV_ONCE_INIT;
    uv_once(&init_once, InitCryptoOnce);

    Environment* env = Environment::GetCurrent(context);
    SecureContext::Initialize(env, target);
```

`SecureContext::Initialize`:

```c++
    void CipherBase::Initialize(Environment* env, Local<Object> target) {
        Local<FunctionTemplate> t = env->NewFunctionTemplate(New);
```

Lets take a look what `New` does:

```c++
    void SecureContext::New(const FunctionCallbackInfo<Value>& args) {
      Environment* env = Environment::GetCurrent(args);
      new SecureContext(env, args.This());
    }
```

Usage of tls would look like a normal require (though since this is an native module the loading will be done by `NativeModule.require(filename)`:

```javascript
    const tls = require('tls');
```

This module exports (among other functions): 

```javascript
    exports.createSecureContext = require('_tls_common').createSecureContext;
    exports.SecureContext = require('_tls_common').SecureContext;
    exports.TLSSocket = require('_tls_wrap').TLSSocket;
    exports.Server = require('_tls_wrap').Server;
    exports.createServer = require('_tls_wrap').createServer;
    exports.connect = require('_tls_wrap').connect;
```

So calling `tls.createSecureContext` will end up in `lib/_tls_common.js:

```javascript
    exports.createSecureContext = function createSecureContext(options, context) {
    ...
      var c = new SecureContext(options.secureProtocol, secureOptions, context);
```

And here we find the call to `SecureContext` using `new`. At this point this
function will have been bound by `SecureContext::Initialize` which is called
by `node::crypto::Initialize` in `src/node_crypto.cc`. When the `tls` module
is required it will load the internal `crypto` module` which is what will 
cause `node::crypto::Initialize` to be invoked.

`SecureContext::Initialize(env, target)` has the following functions that is
adds:
```c++
    Local<FunctionTemplate> t = env->NewFunctionTemplate(SecureContext::New);
    t->InstanceTemplate()->SetInternalFieldCount(1);
    t->SetClassName(FIXED_ONE_BYTE_STRING(env->isolate(), "SecureContext"));

    env->SetProtoMethod(t, "init", SecureContext::Init);
    env->SetProtoMethod(t, "setKey", SecureContext::SetKey);
    env->SetProtoMethod(t, "setCert", SecureContext::SetCert);
    env->SetProtoMethod(t, "addCACert", SecureContext::AddCACert);
    ...
    target->Set(FIXED_ONE_BYTE_STRING(env->isolate(), "SecureContext"), t->GetFunction());
```

So, when `tls.createSecureContext` returns `c` will have been bound to a new
function that was set up above.

In most/all of the functions in the SecureContext class you'll find the following:
```c++
  SecureContext* sc;
  ASSIGN_OR_RETURN_UNWRAP(&sc, args.Holder());
```
If call to such a function could look like this:
```javascript
   c.context.addCACert('something');
```
The JavaScript layer will add wrap the native secure context in a object with
a context member for the native secure context, which is the SecureContext on
the C++ side. So when we call `addCACert` what is unwrapped by 
`ASSIGN_OR_RETURN_UNWRAP` is `c` which is the native context. 
    
Remember that these are prototype functions that are being setup on instance created when using
`new SecureContext` which will be an instance of `CipherBase` since this is the type that the `New` function returned.

When is the underlying OpenSSL Context created? 
This is one in `SecureContext::Init` which is called in `_tls_common.js`, in the
SecureContext constructor:
```javascript
  this.context = new NativeSecureContext();
  this.context.init(secureProtocol);
```
`SecureContext::Init` will setup/initialize the OpenSSL context. This includes
parsing the options passed, `secureProtocol` above and calling the correct 
TLS_xxx_method(), and then using that `SSL_METHOD` pointer to create a new
SSL_CTX.


```javascript
    const binding = internalBinding('crypto');
```

Recall that internalBinding is a function that is set on the process object when it is set up on node.cc. This 
will be a call to Binding:
```console
    (lldb) br s -f node.cc -l 2723
```
```c++
    static void Binding(const FunctionCallbackInfo<Value>& args) {
      Local<String> module = args[0]->ToString(env->isolate());
      node::Utf8Value module_v(env->isolate(), module);
      ...
    }
```
```console
    (lldb) jlh module
    #crypto
```
```c++
    Local<Array> modules = env->module_load_list_array();
```
```console
    (lldb) jlh modules
    0x16da0049c641: [JSArray]
     - map = 0x34afb7a04099 [FastProperties]
     - prototype = 0xe7de5189b01
     - elements = 0x39e4730d5221 <FixedArray[82]> [FAST_HOLEY_ELEMENTS]
     - length = 69
     - properties = 0x25494902241 <FixedArray[0]> {
        #length: 0x4ddc16a6369 <AccessorInfo> (const accessor descriptor)
     }
     - elements = 0x39e4730d5221 <FixedArray[82]> {
           0: 0x246944578d59 <String[18]: Binding contextify>
           1: 0x246944578d89 <String[15]: Binding natives>
           2: 0x246944578db1 <String[14]: Binding config>
           3: 0x246944578dd9 <String[19]: NativeModule events>
           4: 0x246944578e01 <String[18]: Binding async_wrap>
           5: 0x246944578e31 <String[11]: Binding icu>
           6: 0x246944578e59 <String[17]: NativeModule util>
           7: 0x246944578e81 <String[10]: Binding uv>
           8: 0x246944578ea9 <String[19]: NativeModule buffer>
           9: 0x246944578ed1 <String[14]: Binding buffer>
          10: 0x246944578ef9 <String[12]: Binding util>
          11: 0x246944578f21 <String[26]: NativeModule internal/util>
          12: 0x246944578f49 <String[28]: NativeModule internal/errors>
          13: 0x246944578f71 <String[17]: Binding constants>
          14: 0x246944578fa1 <String[28]: NativeModule internal/buffer>
          15: 0x246944578fc9 <String[19]: NativeModule timers>
          16: 0x246944578ff1 <String[18]: Binding timer_wrap>
          17: 0x246944579021 <String[32]: NativeModule internal/linkedlist>
          18: 0x246944579049 <String[24]: NativeModule async_hooks>
          19: 0x246944579071 <String[19]: NativeModule assert>
          20: 0x246944579099 <String[29]: NativeModule internal/process>
          21: 0x2469445790c1 <String[37]: NativeModule internal/process/warning>
          22: 0x2469445790e9 <String[39]: NativeModule internal/process/next_tick>
          23: 0x246944579111 <String[38]: NativeModule internal/process/promises>
          24: 0x246944579139 <String[35]: NativeModule internal/process/stdio>
          25: 0x246944579161 <String[25]: NativeModule internal/url>
          26: 0x246944579189 <String[33]: NativeModule internal/querystring>
          27: 0x2469445791b1 <String[24]: NativeModule querystring>
          28: 0x2469445791d9 <String[11]: Binding url>
          29: 0x246944579201 <String[17]: NativeModule path>
          30: 0x246944579229 <String[19]: NativeModule module>
          31: 0x246944579251 <String[28]: NativeModule internal/module>
          32: 0x246944579279 <String[15]: NativeModule vm>
          33: 0x2469445792a1 <String[15]: NativeModule fs>
          34: 0x39e4730e0651 <String[10]: Binding fs>
          35: 0x39e4730e0679 <String[19]: NativeModule stream>
          36: 0x39e4730e06a1 <String[36]: NativeModule internal/streams/legacy>
          37: 0x39e4730e06c9 <String[29]: NativeModule _stream_readable>
          38: 0x39e4730e06f1 <String[40]: NativeModule internal/streams/BufferList>
          39: 0x39e4730e0719 <String[37]: NativeModule internal/streams/destroy>
          40: 0x39e4730e0741 <String[29]: NativeModule _stream_writable>
          41: 0x39e4730e0769 <String[27]: NativeModule _stream_duplex>
          42: 0x39e4730e0791 <String[30]: NativeModule _stream_transform>
          43: 0x39e4730e07b9 <String[32]: NativeModule _stream_passthrough>
          44: 0x39e4730e07e1 <String[21]: Binding fs_event_wrap>
          45: 0x39e4730e0811 <String[24]: NativeModule internal/fs>
          46: 0x39e4730e0839 <String[17]: Binding inspector>
          47: 0x21515bc747c9 <String[15]: NativeModule os>
          48: 0x21515bc747f1 <String[10]: Binding os>
          49: 0x21515bc74819 <String[26]: NativeModule child_process>
          50: 0x21515bc74841 <String[18]: Binding spawn_sync>
          51: 0x21515bc74871 <String[17]: Binding pipe_wrap>
          52: 0x21515bc748a1 <String[35]: NativeModule internal/child_process>
          53: 0x21515bc748c9 <String[27]: NativeModule string_decoder>
          54: 0x21515bc748f1 <String[16]: NativeModule net>
          55: 0x21515bc74919 <String[25]: NativeModule internal/net>
          56: 0x21515bc74941 <String[18]: Binding cares_wrap>
          57: 0x21515bc74971 <String[16]: Binding tty_wrap>
          58: 0x21515bc74999 <String[16]: Binding tcp_wrap>
          59: 0x21515bc749c1 <String[19]: Binding stream_wrap>
          60: 0x21515bc749f1 <String[18]: NativeModule dgram>
          61: 0x21515bc74a19 <String[16]: Binding udp_wrap>
          62: 0x21515bc74a41 <String[20]: Binding process_wrap>
          63: 0x21515bc74a71 <String[33]: NativeModule internal/socket_list>
          64: 0x21515bc74a99 <String[20]: NativeModule console>
          65: 0x21515bc74ac1 <String[16]: NativeModule tty>
          66: 0x21515bc74ae9 <String[19]: Binding signal_wrap>
          67: 0x35756b2729e1 <String[16]: NativeModule tls>
          68: 0xd6549a0f479 <String[16]: NativeModule url>
       69-81: 0x25494902351 <the hole>
    }
```

A `NativeModule` is a module which has access to the module property and implemented in JavaScript.
A `Binding` is something that does not have a module and only set exports. 

```c++
    modules->Set(l, OneByteString(env->isolate(), buf));
```
```console
    (lldb) p buf
    (char [1024]) $16 = "Binding crypto"
```

```c++
    node_module* mod = get_builtin_module(*module_v);
```
```console
    (lldb) p *mod
    (node::node_module) $24 = {
      nm_version = 55
      nm_flags = 1
      nm_dso_handle = 0x0000000000000000
      nm_filename = 0x0000000101bc4053 "../src/crypto_impl/node_crypto.cc"
      nm_register_func = 0x0000000000000000
      nm_context_register_func = 0x000000010180f630 (node`node::crypto::InitCrypto(v8::Local<v8::Object>, v8::Local<v8::Value>, v8::Local<v8::Context>, void*) at node_crypto.cc:6230)
      nm_modname = 0x0000000101baffde "crypto"
      nm_priv = 0x0000000000000000
      nm_link = 0x00000001027c54c0
   }
```

In this case I'm actually documenting and troubleshooting which is the reason for the strange path names to the source files.

```c++
     if (mod != nullptr) {
       exports = Object::New(env->isolate());
       // Internal bindings don't have a "module" object, only exports.
       CHECK_EQ(mod->nm_register_func, nullptr);
       CHECK_NE(mod->nm_context_register_func, nullptr);
       Local<Value> unused = Undefined(env->isolate());
       mod->nm_context_register_func(exports, unused, env->context(), mod->nm_priv);
       cache->Set(module, exports);
```

So we check that the `nm_register_func` is not set but we should have a context aware register node module function but set the `exports` instance to Undefined.
Next `mod->nm_context_register_func` is called which was configured in node_crypto.cc:

```c++
    NODE_MODULE_CONTEXT_AWARE_BUILTIN(crypto, node::crypto::InitCrypto)
```

So InitCrypto will be the function we end up in which has the call to `SecureContext::Initialize(env, target);` which I wanted to know how it ended up there.

#### InitCrypto
This will intialize the following:

```c++
    SecureContext::Initialize(env, target);
    Connection::Initialize(env, target); // SSLConnection
    CipherBase::Initialize(env, target);
    DiffieHellman::Initialize(env, target);
    ECDH::Initialize(env, target);
    Hmac::Initialize(env, target);
    Hash::Initialize(env, target);
    Sign::Initialize(env, target);
    Verify::Initialize(env, target); 
```

Should the constructors for these be checking that they are called with new?
Even if these are internal it might be possible to call them using:

```c++
    const binding = process.binding('crypto');
    //const h = new binding.Hmac();
    const h = binding.Hmac();
```
   
### Builtins/Constants/Natives
Using process.binding you can load builtin modules, the constant module, or native modules.
The builtin modules are those that are initialized by the loader, for example: 
```c++
    NODE_MODULE_CONTEXT_AWARE_BUILTIN(config, node::InitConfig)
```

The above module would then be loaded using process.binding('config').
If you look at the `Binding` function is src/node.cc you can see that there are two special cases if the module is not 
found as a builtin. One if the name passed in is `constants` and one if the name passed in is `natives`.

#### constants
Just a note here about how src/node_constant.cc is loaded as there is no `NODE_MOUDULE_CONTEXT_AWARE_BUILTIN` macro or anything like that. Instead this will be loaded when:

```c++
    process.binding('constants').
```

Which like mentioned earlier in this document this will invoke `Binding` (in node.cc) and there is a special clause for `constants`:

```c++
    } else if (!strcmp(*module_v, "constants")) {
      exports = Object::New(env->isolate());
      CHECK(exports->SetPrototype(env->context(), Null(env->isolate())).FromJust());
      DefineConstants(env->isolate(), exports);
      cache->Set(module, exports);
```

`DefineConstants`: 

```c++
    DefineErrnoConstants(err_constants);
    DefineWindowsErrorConstants(err_constants);
    DefineSignalConstants(sig_constants);
    DefineUVConstants(os_constants);
    DefineSystemConstants(fs_constants);
    DefineOpenSSLConstants(crypto_constants);
    DefineCryptoConstants(crypto_constants);
    DefineZlibConstants(zlib_constants);

    os_constants->Set(OneByteString(isolate, "errno"), err_constants);
    os_constants->Set(OneByteString(isolate, "signals"), sig_constants);
    target->Set(OneByteString(isolate, "os"), os_constants);
    target->Set(OneByteString(isolate, "fs"), fs_constants);
    target->Set(OneByteString(isolate, "crypto"), crypto_constants);
    target->Set(OneByteString(isolate, "zlib"), zlib_constants);
```
    

#### Natives
When you see:
```javascript
    NativeModule._source = process.binding('natives');
```

Similar to when `process.binding('constants')` is used there is a special clause in Binding for this:
```c++
    } else if (!strcmp(*module_v, "natives")) {
      exports = Object::New(env->isolate());
      DefineJavaScript(env, exports);
      cache->Set(module, exports);
```

`DefineJavaScript` is declared in `src/node_javascript.h` but as you might recall there is no implementation in the source code tree for this header the source file is generated
using the JavaScript source files and config.gypi that is generated by configure:
```
    'action_name': 'node_js2c',
    'process_outputs_as_sources': 1,
    'inputs': [
      '<@(library_files)',
      './config.gypi',
    ],
    'outputs': [
      '<(SHARED_INTERMEDIATE_DIR)/node_javascript.cc',
    ...
```
So we can see that the JavaScript library files are included and config.gypi.

If you call `process.binding('config')` what will be returned will be a builtin module as there is one registered:

```c++
    NODE_MODULE_CONTEXT_AWARE_BUILTIN(config, node::InitConfig)
```

But this does not populate the process objects config variables which you might think. Instead that is done in node_bootstrap.js:

```javascript
    const _process = NativeModule.require('internal/process');
    _process.setupConfig(NativeModule._source);
```


When I was working on decoupling OpenSSL from node.cc I missed out the macros that are used to conditionally set various settings. For example in `GetFeatures` we have:

```c++
    #ifdef SSL_CTRL_SET_TLSEXT_SERVERNAME_CB
      Local<Boolean> tls_sni = True(env->isolate());
    #else
      Local<Boolean> tls_sni = False(env->isolate());
    #endif
      obj->Set(FIXED_ONE_BYTE_STRING(env->isolate(), "tls_sni"), tls_sni);
```


Fix formatting in src/node_crypto.cc X509ToObject


### TLS OpenSSL 1.1.0 Issue
I'm tryin to make test/parallel/test-tls-ecdh-disable.js to be used with both OpenSSL 1.0.x and 1.1.x.
The issue is that the test successfully listens and the mustNotCall function is called which is the callback passed to the listener event handler registration.

The cipher in use is `ECDHE-RSA-AES128-GCM-SHA256`. We can check what ciphers are supported by using the following command:
```console
    $ ~/work/security/build_1_1_0f/bin/openssl ciphers -v 'ECDH+AESGCM:DH+AESGCM:ECDH+AES256:DH+AES256:ECDH+AES128:DH+AES:RSA+AESGCM:RSA+AES:!aNULL:!MD5:!DSS'
    ECDHE-ECDSA-AES256-GCM-SHA384 TLSv1.2 Kx=ECDH     Au=ECDSA Enc=AESGCM(256) Mac=AEAD
    ECDHE-RSA-AES256-GCM-SHA384 TLSv1.2 Kx=ECDH     Au=RSA  Enc=AESGCM(256) Mac=AEAD
    ECDHE-ECDSA-AES128-GCM-SHA256 TLSv1.2 Kx=ECDH     Au=ECDSA Enc=AESGCM(128) Mac=AEAD

    ECDHE-RSA-AES128-GCM-SHA256 TLSv1.2 Kx=ECDH     Au=RSA  Enc=AESGCM(128) Mac=AEAD <----

    DHE-RSA-AES256-GCM-SHA384 TLSv1.2 Kx=DH       Au=RSA  Enc=AESGCM(256) Mac=AEAD
    DHE-RSA-AES128-GCM-SHA256 TLSv1.2 Kx=DH       Au=RSA  Enc=AESGCM(128) Mac=AEAD
    ECDHE-ECDSA-AES256-CCM8 TLSv1.2 Kx=ECDH     Au=ECDSA Enc=AESCCM8(256) Mac=AEAD
    ECDHE-ECDSA-AES256-CCM  TLSv1.2 Kx=ECDH     Au=ECDSA Enc=AESCCM(256) Mac=AEAD
    ECDHE-ECDSA-AES256-SHA384 TLSv1.2 Kx=ECDH     Au=ECDSA Enc=AES(256)  Mac=SHA384
    ECDHE-RSA-AES256-SHA384 TLSv1.2 Kx=ECDH     Au=RSA  Enc=AES(256)  Mac=SHA384
    ECDHE-ECDSA-AES256-SHA  TLSv1 Kx=ECDH     Au=ECDSA Enc=AES(256)  Mac=SHA1
    ECDHE-RSA-AES256-SHA    TLSv1 Kx=ECDH     Au=RSA  Enc=AES(256)  Mac=SHA1
    DHE-RSA-AES256-CCM8     TLSv1.2 Kx=DH       Au=RSA  Enc=AESCCM8(256) Mac=AEAD
    DHE-RSA-AES256-CCM      TLSv1.2 Kx=DH       Au=RSA  Enc=AESCCM(256) Mac=AEAD
    DHE-RSA-AES256-SHA256   TLSv1.2 Kx=DH       Au=RSA  Enc=AES(256)  Mac=SHA256
    DHE-RSA-AES256-SHA      SSLv3 Kx=DH       Au=RSA  Enc=AES(256)  Mac=SHA1
    ECDHE-ECDSA-AES128-CCM8 TLSv1.2 Kx=ECDH     Au=ECDSA Enc=AESCCM8(128) Mac=AEAD
    ECDHE-ECDSA-AES128-CCM  TLSv1.2 Kx=ECDH     Au=ECDSA Enc=AESCCM(128) Mac=AEAD
    ECDHE-ECDSA-AES128-SHA256 TLSv1.2 Kx=ECDH     Au=ECDSA Enc=AES(128)  Mac=SHA256
    ECDHE-RSA-AES128-SHA256 TLSv1.2 Kx=ECDH     Au=RSA  Enc=AES(128)  Mac=SHA256
    ECDHE-ECDSA-AES128-SHA  TLSv1 Kx=ECDH     Au=ECDSA Enc=AES(128)  Mac=SHA1
    ECDHE-RSA-AES128-SHA    TLSv1 Kx=ECDH     Au=RSA  Enc=AES(128)  Mac=SHA1
    DHE-RSA-AES128-CCM8     TLSv1.2 Kx=DH       Au=RSA  Enc=AESCCM8(128) Mac=AEAD
    DHE-RSA-AES128-CCM      TLSv1.2 Kx=DH       Au=RSA  Enc=AESCCM(128) Mac=AEAD
    DHE-RSA-AES128-SHA256   TLSv1.2 Kx=DH       Au=RSA  Enc=AES(128)  Mac=SHA256
    DHE-RSA-AES128-SHA      SSLv3 Kx=DH       Au=RSA  Enc=AES(128)  Mac=SHA1
    AES256-GCM-SHA384       TLSv1.2 Kx=RSA      Au=RSA  Enc=AESGCM(256) Mac=AEAD
    AES128-GCM-SHA256       TLSv1.2 Kx=RSA      Au=RSA  Enc=AESGCM(128) Mac=AEAD
    AES256-CCM8             TLSv1.2 Kx=RSA      Au=RSA  Enc=AESCCM8(256) Mac=AEAD
    AES256-CCM              TLSv1.2 Kx=RSA      Au=RSA  Enc=AESCCM(256) Mac=AEAD
    AES128-CCM8             TLSv1.2 Kx=RSA      Au=RSA  Enc=AESCCM8(128) Mac=AEAD
    AES128-CCM              TLSv1.2 Kx=RSA      Au=RSA  Enc=AESCCM(128) Mac=AEAD
    AES256-SHA256           TLSv1.2 Kx=RSA      Au=RSA  Enc=AES(256)  Mac=SHA256
    AES128-SHA256           TLSv1.2 Kx=RSA      Au=RSA  Enc=AES(128)  Mac=SHA256
    AES256-SHA              SSLv3 Kx=RSA      Au=RSA  Enc=AES(256)  Mac=SHA1
    AES128-SHA              SSLv3 Kx=RSA      Au=RSA  Enc=AES(128)  Mac=SHA1
```


### Zlib
The version we are using on Fedora is:
```console
     zlib: '1.2.8'
```

The version building locally is:
```console
     zlib: '1.2.11',
```

The issue I'm seeing is that `test/parallel/test-zlib-failed-init.js` passes as expected when using version 1.2.11 but fails
when using 1.2.8. 
The change log for zlib can be found here: http://zlib.net/ChangeLog.txt
```console
    Changes in 1.2.9 (31 Dec 2016)
    ...
    - Reject a window size of 256 bytes if not using the zlib wrapper
    ...
````

### ICU
We are currently building using --with-intl=system-icu which gives the following icu version:
```console
    icu: '57.1'
```

When I build locally and without any icu flags the version is:

The issue I'm seeing is that test/parallel/test-icu-data-dir.js is failing:
```console
    assert.js:60
    throw new errors.AssertionError({
    ^

    AssertionError [ERR_ASSERTION]: false == true
      at Object.<anonymous> (/root/rpmbuild/BUILD/node-v8.1.0/test/parallel/test-icu-data-dir.js:13:3)


    const child = spawnSync(process.execPath, ['--icu-data-dir=/', '-e', '0']);
    assert(child.stderr.toString().includes(expected));
```


### Fips
When using the bundled/deps version of OpenSSL and the just building without any configuration options, OpenSSL FIPS support is not enabled. 
The test `test/parallel/test-crypto-fips.js` performs a number of checks and assumes the above default configuration. I'm running into an issue when configuring Node using --shared-openssl and the system version of OpenSSL supports FIPS (but I'm not setting anything FIPS related when configuring only the shared includes and shared lib directory. When I do this the mentioned test fails when trying to toggle fips_mode using a OpenSSL configuration file.
```console
    [root@1b05bc2e415c node-v8.1.0]# openssl version
    OpenSSL 1.0.2k-fips  26 Jan 2017    
```

    out/Release/node test/parallel/test-icu-data-dir.js:
```console
    Spawned child [pid:9757] with cmd 'require("crypto").fips' expect 0 with args '--openssl-config=/root/rpmbuild/BUILD/node-v8.1.0/test/fixtures/openssl_fips_enabled.cnf' OPENSSL_CONF=undefined
    assert.js:60
      throw new errors.AssertionError({
      ^

    AssertionError [ERR_ASSERTION]: 0 === 1
        at responseHandler (/root/rpmbuild/BUILD/node-v8.1.0/test/parallel/test-crypto-fips.js:56:14)
````


You might have to manually remove `config_fips.gypi` when you want to reconfigure fips.

#### Building the bundled openssl with fips
```console
    $ ./configure --openssl-fips=/Users/danielbevenius/work/security/build_1_0_2k/
    $ out/Release/node
    > process.versions.openssl
    '1.0.2l-fips'
```

### Node N-API
Lets start from the beginning and look at the initialization of a napi addons:
```c
NAPI_MODULE(NODE_GYP_MODULE_NAME, Init)
```
This macro is defined in `src/node_api.h`:
```c
#define NAPI_MODULE_X(modname, regfunc, priv, flags)                  \
  EXTERN_C_START                                                      \
    static napi_module _module =                                      \
    {                                                                 \
      NAPI_MODULE_VERSION,                                            \
      flags,                                                          \
      __FILE__,                                                       \
      regfunc,                                                        \
      #modname,                                                       \
      priv,                                                           \
      {0},                                                            \
    };                                                                \
    NAPI_C_CTOR(_register_ ## modname) {                              \
      napi_module_register(&_module);                                 \
    }                                                                 \
  EXTERN_C_END

#define NAPI_MODULE(modname, regfunc)                                 \
  NAPI_MODULE_X(modname, regfunc, NULL, 0)  // NOLINT (readability/null_usage)
```
This is basically the same as what we have seen for a normal addons and would expend to:
```c
extern "C" {
  static napi_module _module = {
      NAPI_MODULE_VERSION,
      flags,
      __FILE__,
      Init,
      NODE_GYP_MODULE_NAME,
      priv,
      {0},
  };
  static void _register_NODE_GYP_MODULE_NAME(void) __attribute__((constructor));
  static void _register_NODE_GYP_MODULE_NAME(void) {
      napi_module_register(&_module);
  }
}
```
It's run when a shared library is loaded. 

```c++
void napi_module_register(napi_module* mod) {
  node::node_module* nm = new node::node_module {
    -1,
    mod->nm_flags,
    nullptr,
    mod->nm_filename,
    nullptr,
    napi_module_register_cb,
    mod->nm_modname,
    mod,  // priv
    nullptr,
  };
  node::node_module_register(nm);
}
```
Let's set a break point on `NAPI_MODULE_INIT` and take a closer look at it:
```console
$ ./configure --debug && make -j8
$ make build-addons-napi
$ lldb -- ./node test/addons-napi/1_hello_world/test.js
(lldb) br s -f binding.c -l 13
Breakpoint 1: no locations (pending).
WARNING:  Unable to resolve breakpoint to any actual locations.
(lldb) r
````
Doing a back trace we will see 
```console
(lldb) bt 10
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
  * frame #0: 0x00000001027adc5b binding.node`_register_binding at binding.c:13
    frame #1: 0x00000001026eeac6 dyld`ImageLoaderMachO::doModInitFunctions(ImageLoader::LinkContext const&) + 420
    frame #2: 0x00000001026eecf6 dyld`ImageLoaderMachO::doInitialization(ImageLoader::LinkContext const&) + 40
    frame #3: 0x00000001026ea218 dyld`ImageLoader::recursiveInitialization(ImageLoader::LinkContext const&, unsigned int, char const*, ImageLoader::InitializerTimingList&, ImageLoader::UninitedUpwards&) + 330
    frame #4: 0x00000001026e934e dyld`ImageLoader::processInitializers(ImageLoader::LinkContext const&, unsigned int, ImageLoader::InitializerTimingList&, ImageLoader::UninitedUpwards&) + 134
    frame #5: 0x00000001026e93e2 dyld`ImageLoader::runInitializers(ImageLoader::LinkContext const&, ImageLoader::InitializerTimingList&) + 74
    frame #6: 0x00000001026dd3e5 dyld`dyld::runInitializers(ImageLoader*) + 82
    frame #7: 0x00000001026e60a8 dyld`dlopen + 527
    frame #8: 0x00007fff6704fd86 libdyld.dylib`dlopen + 86
    frame #9: 0x000000010003b238 node`node::DLOpen(v8::FunctionCallbackInfo<v8::Value> const&) [inlined] node::DLib::Open() at node.cc:1158 [opt]
    frame #10: 0x000000010003b224 node`node::DLOpen(args=<unavailable>) at node.cc:1253 [opt]
```
The example we are looking at is `test/addons-napi/1_hello_world` and 
```javascript
const binding = require(`./build/${common.buildType}/binding`);
```
Notice that this does not have a file extension, and one of the file extensions that will be tried is `.node`.
which will be `test/addons-napi/1_hello_world/build/Debug/binding.node`.

From `lib/internal/modules/cjs/loader.js`:
```javascript
// Native extension for .node
Module._extensions['.node'] = function(module, filename) {
  return process.dlopen(module, path.toNamespacedPath(filename));
};
```
Each addon is a dynamical library which is loaded by LDOpen in `src/node.cc`

```c++
NAPI_MODULE_INIT() {
  napi_property_descriptor desc = DECLARE_NAPI_PROPERTY("hello", Method);
  NAPI_CALL(env, napi_define_properties(env, exports, 1, &desc));
  return exports;
```
Note that `DECLARE_NAPI_PROPERTY` is a macro defined in `test/common.h` and just initializes the struct with:
```c++
  napi_property_descriptor desc = { "hello", 0, Method, 0, 0, 0, napi_default, 0);
```
The `napi_property_descriptor` struct look like this:
```c++
typedef struct {
  // One of utf8name or name should be NULL.
  const char* utf8name;
  napi_value name;

  napi_callback method;
  napi_callback getter;
  napi_callback setter;
  napi_value value;

  napi_property_attributes attributes;
  void* data;
} napi_property_descriptor;
```

`napi_callback` is a function pointer, which is getting set to 0. Why is this not set to NULL?
```c+++
typedef napi_value (*napi_callback)(napi_env env, napi_callback_info info);
```

`napi_default` is an enum in `src/node_api_types.h`:
```c++
typedef enum {
  napi_default = 0,
  napi_writable = 1 << 0,
  napi_enumerable = 1 << 1,
  napi_configurable = 1 << 2,

  // Used with napi_define_class to distinguish static properties
  // from instance properties. Ignored by napi_define_properties.
  napi_static = 1 << 10,
} napi_property_attributes;
```

All JavaScript values are abstracted behind an opaque type named napi_value. This is a way hide details from the using code
and they don't have access to the actual implementation of the struct. In `napi_value`'s case it is just a pointer and
no additional information exist, but for other types like napi_env:
```c++
typedef struct napi_env__* napi_env;
```
`napi_env__` contains information like the Isolate, the node::Environment for example and is defined in src/node_api.cc which
is the implementation. So the while you can see the contents of the struct in the debugger trying to access any member 
information will get an error like the following:
```console
../binding.c:7:6: error: incomplete definition of type 'struct napi_env__'
  env->isolate;
  ~~~^
/Users/danielbevenius/work/nodejs/node-poc/src/node_api_types.h:13:16: note: forward declaration of 'struct napi_env__'
typedef struct napi_env__* napi_env;
               ^
1 error generated.
````
The way these types are used is by passing them to functions that operate on them, and those functions have access to the
internal type definitions. This is also what OpenSSL does for it's type system.

### FIXED_ONE_BYTE_STRING
This is a macro which can be found in `src/util.h`:
```c++
  // Used to be a macro, hence the uppercase name.
  template <int N>
  inline v8::Local<v8::String> FIXED_ONE_BYTE_STRING(
    v8::Isolate* isolate,
    const char(&data)[N]) {
    return OneByteString(isolate, data, N - 1);
  }
```
`const char(&data)` is the same as `const char&[]`
const char (&)[]
So we are passing in the string which recieved as a const char* in the OneByteString function. This is then reinterpreted to const uint8_t*.

OneByteString: the string's bytes are not UTF-8 encoded, can only contain characters in the first 256 unicode code points
TwoByteString: Same, but uses two bytes for each character (using surrogate pairs to represent unicode characters that can't be represented in two bytes).


## Binary Compatability
Is when a program is linked dynmically to a former version and does not have to be recompiled.
If a program needs to be recompiled to run with a new version of library but doesn't require 
any further modifications, the library is source compatible.

### Application Binary Interface (ABI)
An ABI defines the structures and methods used to access external, already compiled libraries/code at the level of machine code.
The headers decsribing classes, functions etc are compiled to a set of addresses
and expected parameters, memory structure sizes and layout. The application using
the ABI must be compiled so that these addresses, the expected paramters, memory
layout etc match those that that the ABI provider provided.
Changes to the header files can be made but must be done carfully to ensure that
the compability is not changed, so that the addresses produced when compiling
are not changed which would mean that exising users of the compiled code would 
become incompatible.


Keeping an ABI stable means not changing function interfaces (return type and number, 
types, and order of arguments), definitions of data types or data structures, defined constants, etc. 
New functions and data types can be added, but existing ones must stay the same.

Making a namespace change in C++ will generate a different mangled name so the symbol in the object file will be different.

In Node there is `NODE_MODULE_VERSION` which is defined in `node_version.h`


#### C++ Lint task
Verify that functions that return pointers have the pointer operator in the correct place (to the left).

The rules can be found in 'tools/cpplint.py' and can be run using:
```console
    tools/cpplint.py files
```

The script will run through the files and for each one call ProcessFile which does some checking and then calls
ProcessFileData and later ProcessLine. Not that you can pass in extra_check_functions which might become handy.


### NDEBUG
I've seen this in couple of places in the node source code and did not know about it.
NDBUG is used to toggle/control whether the assert macro will expand into something that will perform a check or not.

From `<assert.h>`:
```c++
    #ifdef NDEBUG
    #define assert(condition) ((void)0)
    #else
    #define assert(condition) 
    #endif
```

### assert
This macro is disabled if, at the moment of including <assert.h>, a macro with the name NDEBUG has already been defined. 
This allows for a coder to include as many assert calls as needed in a source code while debugging the program and then disable all of them for the production version by simply including a line like:
 
```c++
    #define NDEBUG 
```

If the NDEBUG macro is defined when <assert.h> is included the assets are disabled. While looking into adding a addons test I noticed that at-exit undefined
NDEBUG but not the other tests that use assert. As far as I can tell there is no need to undefine this and non of the other tests do. This commit removes
the undef for consistency.

### Building a addons
Change to the directory of the addons and then you can rebuild using:
```console
    $ out/Debug/node deps/npm/node_modules/node-gyp/bin/node-gyp.js rebuild --directory=test/addons/at-exit --nodedir=../../../
```

On windows you can just copy the command from the output from, for example:
```console
    "C:\\Users\\danbev\\working\\node\\Release\\node.exe" "C:\\Users\\danbev\\working\\node\\deps\\npm\\node_modules\\node-gyp\\bin\\node-gyp" "rebuild" "--directory=test\\addons\\openssl-client-cert-engine" "--nodedir=C:\\Users\\danbev\\working\\node"
```

### Run an a set of JavaScript tests
You can specify the directory to run test, for example this would only run the async-hooks tests:
```console
    $ python tools/test.py --mode=release -J async-hooks
```

### Event Loop
It all starts with the javascript file to be executed, which is you main program.
```

    ------------> javascript.js ------+-----------------------------+ 
                                     /|\                            | setTimeout/SetInterval
                                      |                             | JavaScript callbacks --------------------------------------+
                                      |                             |                                                            |
                                      |                             | network/disk/child_processes                               |
                                      |                             | JavaScript callbacks --------------------------------------+
                                      |                             |                                                            |----> callback ------> nextTick callback ------------------------------+
                                      |                             | setImmedate                                                |                             /|\                                       |
                                      +                             | JavaScript callbacks --------------------------------------|                              |                                        |
                                       \                            |                                                            |                              +---- process resolved promises ---------+ 
                                        \                           | close events                                               |                    
                                         \                          | JavaScript callbacks --------------------------------------|
                                          \                         |
                                           \                        |
    <----------- process.exit (event) <-----------------------------+
```

Where is the first interaction with libuv in node?
There very first call is (in node.cc):

```c++
    argv = uv_setup_args(argc, argv);
```

But this does not do anything with the event loop. For that we have to look at `Init`:

```c++
    prog_start_time = static_cast<double>(uv_now(uv_default_loop()));
```

This will land us in uv_common.c:

```c++
    uv_loop_t* uv_default_loop(void) {
     if (default_loop_ptr != NULL)
       return default_loop_ptr;
   
     if (uv_loop_init(&default_loop_struct))
```
```console
    (lldb) p default_loop_ptr
    (uv_loop_t *) $4 = 0x0000000000000000
```

So we can see that this is the first call to `uv_default_loop` so lets take a closer look at `uv_loop_init` in src/unix/loop.c (in libuv that is):

```c++
    uv__signal_global_once_init();
```

Which will call:

```c++
    uv_once(&uv__signal_global_init_guard, uv__signal_global_init);
```

I just skimmed the code but this looks like setting up fork handlers to reset signals. uv_once uses [pthread_once](https://github.com/danbev/learning-c/blob/master/pthreads-once.c).
After this we will be back in loop.c:

```c++
    heap_init((struct heap*) &loop->timer_heap);
```

Where are initializing a min heap data structur for timers.

```c++
    QUEUE_INIT(&loop->wq)
```

`wq` is a work queue (include/uv-threadpool.h):

```c++
    void* wq[2];
```

What does the macro `QUEUE_INIT` do?
It is defined in `src/queue.h` as:

```c++
    QUEUE_NEXT(q) = (q);                                                      \
    QUEUE_PREV(q) = (q);

    typedef void *QUEUE[2];
    ...
    #define QUEUE_NEXT(q)       (*(QUEUE **) &((*(q))[0]))
```

This will set &loop->wp[0]

And the same goes for handle_queue and active_reqs which will be initialized using QUEUE_INIT as well:

```c++
    void* handle_queue[2];
    void* active_reqs[2];

    QUEUE_INIT(&loop->active_reqs);
    QUEUE_INIT(&loop->idle_handles);

    ...
    err = uv__platform_loop_init(loop);
```


`uv__platform_loop_init` can be found in src/unix/darwin.c:

```c++
    if (uv__kqueue_init(loop)) 
```

`uv__kqueue_init` can be found in src/unix/kqueue.c:

```c++
    loop->backend_fd = kqueue()
```

#### Resolve promises
After calling the callback provided by the user, a smaller loop will check if there are any resolved promises.

Where is this done?
If we take a look at lib/internal/bootstrap.js and the start function we find the following line:
```javascript
    NativeModule.require('internal/process/next_tick').setup();
```

And lib/internal/process/next_tick.js: 

```javascript
    exports.setup = setupNextTick;
```

`setupNextTick` then does:

```javascript
    const promises = require('internal/process/promises');
    ...
    const emitPendingUnhandledRejections = promises.setup(scheduleMicrotasks);
```

promises.setup will call process._setupPromises:

```javascript
      process._setupPromises(function(event, promise, reason) {
```

Notice that this call takes an anonymous function as its callback. `_setupPromises` is configured in node.cc in the SetupProcessObject function:

```javascript
    env->SetMethod(process, "_setupPromises", SetupPromises);
```

Lets set a break point in SetupPromis and see what is going on:
```console
    (lldb) br s -f node.cc -l 1278
    (lldb) r
```

```c++
    isolate->SetPromiseRejectCallback(PromiseRejectCallback);
```

So an isolate has a `promise_reject_callback_` which is being set here to `PromiseRejectCallback`. This is what V8 will call if a promise is rejected.

```c++
    env->set_promise_reject_function(args[0].As<Function>());
```

Remember `_setupPromises` was passed an anonymous function as its callback argument which is what is being set as the
set_promise_reject_function on the evnvironment instance. This will then be used by `PromiseRejectCallback`:

```c++
    Local<Function> callback = env->promise_reject_function();
    ...
    callback->Call(process, arraysize(args), args);
```

Next things that happens in `SetupPromises` is that `_setupPromises` is deleted from the process object:

```c++
    env->process_object()->Delete(
      env->context(),
      FIXED_ONE_BYTE_STRING(args.GetIsolate(), "_setupPromises")).FromJust();
```

This way it cannot be called again.

Back in JavaScript (setupNextTick) now we have:

```javascript
    var _runMicrotasks = {};
    ...
    const tickInfo = process._setupNextTick(_tickCallback, _runMicrotasks);
```

`process._setupNextTick` is also configured in node.cc:

```c++
    env->SetMethod(process, "_setupNextTick", SetupNextTick);
```

The arguments to SetupNextTick will be:

```c++
    CHECK(args[0]->IsFunction());  // _tickCallback
    CHECK(args[1]->IsObject());    // _runMicrotasks which is just an empty object at this point.

    env->set_tick_callback_function(args[0].As<Function>());
    env->SetMethod(args[1].As<Object>(), "runMicrotasks", RunMicrotasks);
```

RunMicroTasks does:
 
```c++
    void RunMicrotasks(const FunctionCallbackInfo<Value>& args) {
      args.GetIsolate()->RunMicrotasks();
    }
```
    
So we are setting a method on the _runMicrotasks object named runMicrotasks.
Next `_setupNextTick` is removed from the process object. Then we have:

```c++
    uint32_t* const fields = env->tick_info()->fields();
    uint32_t const fields_count = env->tick_info()->fields_count();
    Local<ArrayBuffer> array_buffer = ArrayBuffer::New(env->isolate(), fields, sizeof(*fields) * fields_count);
    args.GetReturnValue().Set(Uint32Array::New(array_buffer, 0, fields_count));
```

```javascript
    const p = new Promise((resolve, reject) => {
      resolve('ok');
    });
```

So how are promises created in V8?
```console
    (lldb) br s -f isolate.cc -l 1862
```

This will break in isolate.cc `PushPromise`:

```c++
    void Isolate::PushPromise(Handle<JSObject> promise) {
      ThreadLocalTop* tltop = thread_local_top();
      PromiseOnStack* prev = tltop->promise_on_stack_;
      Handle<JSObject> global_promise = global_handles()->Create(*promise);
      tltop->promise_on_stack_ = new PromiseOnStack(global_promise, prev);
    }    
```


Lets take a closer look at `new PromiseOnStack`:

```c++
    class PromiseOnStack {
     public:
      PromiseOnStack(Handle<JSObject> promise, PromiseOnStack* prev)
        : promise_(promise), prev_(prev) {}
      Handle<JSObject> promise() { return promise_; }
      PromiseOnStack* prev() { return prev_; }

     private:
      Handle<JSObject> promise_;
      PromiseOnStack* prev_;
    }; 
```

So that looks simple enough, it has a promise and a pointer to the previous promise. The callback passed to Promise (which
takes a resolve, reject), when is it called?
This is part of a constructor call so it would be executed straight way I think, like any constructor.


##### PromiseRejectCallback

```c++
    void PromiseRejectCallback(PromiseRejectMessage message) {
      Local<Promise> promise = message.GetPromise();
      Isolate* isolate = promise->GetIsolate();
      Local<Value> value = message.GetValue();
      Local<Integer> event = Integer::New(isolate, message.GetEvent());

      Environment* env = Environment::GetCurrent(isolate);
      Local<Function> callback = env->promise_reject_function();

      if (value.IsEmpty())
        value = Undefined(isolate);

      Local<Value> args[] = { event, promise, value };
      Local<Object> process = env->process_object();

      callback->Call(process, arraysize(args), args);
  }
```


After a module has been loaded in Module.runMain, the process._tickCallback function will first process all the 
callbacks in the nextTickQueue and then call _runMicrotasks(). 

#### nextTick
After resolving all the promises and callbacks added using nextTick will be called. 

```javascript
    // bootstrap main module.
    Module.runMain = function() {
      // Load the main module--the command line argument.
      Module._load(process.argv[1], null, true);
      // Handle any nextTicks added in the first tick of the program
      process._tickCallback();
    };
```

### Libuv Thread pool 
The following modules use the thread pool:
* fs.*
* dns.lookup
  This is because when using a hostname to lookup there are operating system specific tasks that have to be 
  performed, like on unix /etc/hosts should be used etc.
* pipes
* crypto (crypto.randomBytes(), and crypto.pbkdf2())
* http.get/request() is called with a name in which case dns.loopup() is used
* any c++ addons that uses it

The follwing modules use the Kernel:
* tcp/udp sockets, servers
* unix domain sockets, servers
* pipes
* tty input
* dns.resolveXXX

The default size of the thread pool is 4 (uv_threadpool_size).
Just to be clear, the libuv Event Loop is single threaded. The thread pool is used for file I/O operations.



### SEGV_MAPERR
Is a segmentation fault, which is an invalid memory access. This can happen when trying to access a page
that is not mapped into the address space of the running process. This can happen when dereferencing a
null pointer or a pointer that was corrupted with a small integer value.

#### core dump
Make sure that you specified enough room for a core dump:
```console
    $ ulimit -c unlimited

If possible compile the executable with debugging options. The example I'm using for this is a case where
cctest on arm7 produces a core dump:
```console
    
    [----------] 2 tests from EnvironmentTest
    [ RUN      ] EnvironmentTest.AtExitWithEnvironment
    [       OK ] EnvironmentTest.AtExitWithEnvironment (114 ms)
    [ RUN      ] EnvironmentTest.AtExitWithArgument
    Received signal 11 SEGV_MAPERR 000007a0547c

    ==== C stack trace ===============================

    [end of stack trace]
    Segmentation fault (core dumped)
```
    
```console
    $ gdb out/Debug/cctest qemu_cctest_20170809-174708_18638.core

    Reading symbols from out/Release/cctest...done.
    [New LWP 18638]
    [New LWP 18644]
    [New LWP 18645]
    [New LWP 18646]
    [Thread debugging using libthread_db enabled]
    Using host libthread_db library "/lib/arm-linux-gnueabihf/libthread_db.so.1".
    Core was generated by `out/Release/cctest'.
    #0  0x005e4f8c in v8::Context::Exit() ()

    (gdb) bt
    #0  0x005e4f8c in v8::Context::Exit() ()
    #1  0x01033d94 in node::Environment::Start(int, char const* const*, int, char const* const*, bool) ()
    #2  0x010395f4 in node::CreateEnvironment(node::IsolateData*, v8::Local<v8::Context>, int, char const* const*, int, char const* const*) ()
    #3  0x00e6087c in EnvironmentTest_AtExitWithArgument_Test::TestBody() ()
    #4  0x00eafb12 in testing::Test::Run() ()
    #5  0x00eafcfc in testing::TestInfo::Run() [clone .part.402] ()
    #6  0x00eafde0 in testing::TestCase::Run() [clone .part.403] ()
    #7  0x00eb15b0 in testing::internal::UnitTestImpl::RunAllTests() [clone .part.407] ()
    #8  0x00eb17e6 in testing::UnitTest::Run() ()
    #9  0x004302dc in main ()
```

I was not able to reproduce this issue when compiling with debugging symbols (./configure --debug). Possible reasons for this?
The debugger puts more on the stack and if you overwrite an arrays capacity it migth cause a segment fault in release mode but
not in debug mode.

Trying to find out where v8::Context::Exit() is call when in node::Environment::Start. Upon entry there is the following:
```c++
    Context::Scope context_scope(context());
```

This is the context scope used and this is using RAII, and the descructor that will be called when this instance goes out
of scope looks like this (in v8.h):

```c++
    V8_INLINE ~Scope() { context_->Exit(); }
```


### Workaround issue with test/async-hooks/init-hooks.js
```console
    $ launchctl unload -w /System/Library/LaunchAgents/com.apple.ReportCrash.plist
    $ sudo launchctl unload -w /System/Library/LaunchDaemons/com.apple.ReportCrash.Root.plist
```


### LIKELY/UNLIKELY
If you take a look at the CHECK macro you will se that it is defined like this:
```c++
   
    #define CHECK(expr)                                                         \
    do {                                                                        \
      if (UNLIKELY(!(expr))) {                                                  \
        static const char* const args[] = { __FILE__, STRINGIFY(__LINE__),      \
                                            #expr, PRETTY_FUNCTION_NAME };      \
        node::Assert(&args);                                                    \
      }                                                                         \
    } while (0)
```

And UNLIKELY is defined as:

```c++
    #define UNLIKELY(expr) __builtin_expect(!!(expr), 0)
```

This is done to give the compiler a hint that we expect the value to be true so it 
can optimise for that case and not the opposite case. 

### Context
Allows separate unrelated JavaScript to run in an isolate without interfering with each other.
What does that mean for a C++ function that is called by javascript. The context should be the
one associated with the call. 


### Tracing
Tracing refers to tracing events that can be emitted by V8. V8 takes a `--enable-tracing` flag which enables
tracing and will create a v8_trace.json file. This file can be opened in chrome and inspected using
`chrome://tracing`

This can also be enabled in Node using `--trace-events-enabled` and you can specify categories to be traced 
using `--trace-event-categories`. 


### Microsoft Build Engine (MSBuild)
Is a build platform that allows for an xml configuration file to configure and run complex build systems. 
This is what is produced by gyp for windows (the project file I mean). So you can take a look at the project
file for the specific gyp target and see what shows up where in it to troubleshoot configuration issues.

A target is like a procedure. A target can contain properties in a PropertiesGroup and tasks.
A task is like a builtin function that performs an action, compiles, links, prints a message etc.
A project is what hooks tasks together. A task can have a `DefaultTargets` attribute which are the targets
that will be run if you specify the project file on the command line for msbuild.
```console

    msbuild sample.vcxproj 

The target can also be specified on the command line:
```console
    msbuild sample.vcxproj /t:something
    msbuild sample.vcxproj /t:Clean;Build
```
    
In the xml you might come across something like: %(AdditionalOptions). This an iterate directive.

You can increase the level of information provided by msbuild by specifying the /verbosity setting:
quit, minimal, normal, detailed, and diagnostic.

Options:
/MP  Compile multiple sources by using multiple processes
/link passes the specified option to LINK


### vcbuild.bat
This is the Windows batch script that is used on Windows systems.
```console
    vcbuild.bat /help
```

```
%var%   replaced with the environment variable named 'var'.
%n      where n is a number from 0-9 and gives access to the n argument passed on the command line.
%*      all the arguments specified on the command line
```

You can place quotation marks around a string to avoid escaping.
You can place caret (^), an escape character, immediately before the special characters
The special characters that need quoting or escaping are usually <, >, |, &, and ^
```console
echo "You & me"
echo You ^& me
```
```
cd %~dp0        this will to %~dp (the directory of) on %0 (the first command line argument). 
                So if this is run from a diferent directory this will switch to the directory where
                this script is.

if /i           case-insensitive, for example:

     if /i "%1"=="help" goto help
```


Configuring on windows:
```console
    python configure --openssl-system-ca-path=PATH
    vcbuild noprojgen
```

#### Compiler
`cl.exe` is the the compiler.
```console
    cl -help 
```

#### Scripting
```
`%~1`          removes quotes from the first command line argument
```

Use `NUL` to discard out put, as in comman > NUL
```console
    IF "%var%"=="" (SET var=something)
    IF NOT DEFINED var (SET var=something)
    IF /I "%ERRORLEVEL%" NEQ "0" (
      echo "failed to execute something"
    )
```


### Performace counters
Are used used to provide information as to how well the operating system or an application is performing.

### DTrace
is supported on linux, solaris, mac, and bsd and is a tracing framework originally developed for Solaris
by Sun.
This is enabled by configuring with the `--with-dtrace` flag. There are three object files for dtrace in 
Node. These are: 
node_dtrace.o which is used by all systems that support dtrace  
node_dtrace_ustack.o is not supported on linux or mac  
node_dtrace_provider.o is supported on all but macosx  
Note that the directory where node_dtrace_provider.o is generated differs so if you are working on a task
that effects linking take that into account.

### Event Tracing for Windows (ETW)
Is an efficient kernel level tracing mechanism allowing logging of kernel or application defined events to
a log file. It also allows dynamic enable/disable so this can be performed on a running application.

An event provider writes events to an ETW session. Additional data is added by ETW like the time the event
happened, the process that wrote it and the thread id, the processor number, the CPU usage data. This info
is then available to event consumers which are applications that read the log file or the consumer can listen
to realtime events and process them.

There is a condition in node.gypi that depends on `node_etw` which looks like this:
```

    [ 'node_use_etw=="true"', {
      'defines': [ 'HAVE_ETW=1' ],
      'dependencies': [ 'node_etw' ],
      'sources': [
        'src/node_win32_etw_provider.h',
        'src/node_win32_etw_provider-inl.h',
        'src/node_win32_etw_provider.cc',
        'src/node_dtrace.cc',
        'tools/msvs/genfiles/node_etw_provider.h',
        'tools/msvs/genfiles/node_etw_provider.rc',
      ]
    } ],
```

So remember that `node_etw` will be run first so lets take a look at it first.
```
    # generate ETW header and resource files
    {
      'target_name': 'node_etw',
      'type': 'none',
      'conditions': [
        [ 'node_use_etw=="true"', {
          'actions': [
            {
              'action_name': 'node_etw',
              'inputs': [ 'src/res/node_etw_provider.man' ],
              'outputs': [
                'tools/msvs/genfiles/node_etw_provider.rc',
                'tools/msvs/genfiles/node_etw_provider.h',
                'tools/msvs/genfiles/node_etw_providerTEMP.BIN',
              ],
              'action': [ 'mc <@(_inputs) -h tools/msvs/genfiles -r tools/msvs/genfiles' ]
            }
          ]
        } ]
      ]
    },
```

`mc` is the [Message Compiler](https://msdn.microsoft.com/en-us/library/windows/desktop/aa385638(v=vs.85).aspx).  
`-h` is where you want the generated header files to be placed.  
`-r` is where you want the generated resource compiler script (.rc file) and the generated binary resource file (.bin)  
The node_etw_providerTEMP.bin is a binary resource file that contains the provider and event metadata. This is the
template resource, which is signified by the TEMP suffix of the base name of the file.

And the input is the manifest file (.man). The manifest registers an event producer named `NodeJS-ETW-provider`:
```
    <provider name="NodeJS-ETW-provider"
        guid="{77754E9B-264B-4D8D-B981-E4135C1ECB0C}"
        symbol="NODE_ETW_PROVIDER"
        message="$(string.NodeJS-ETW-provider.name)"
        resourceFileName="node.exe"
        messageFileName="node.exe">
```

Notice the `message` attribute which is used for localization which will be matched with :
```
    <localization>
        <resources culture="en-US">
            <stringTable>
                <string id="NodeJS-ETW-provider.name" value="Node.js ETW Provider"/>
```


Tasks are typically used to identify major components of the provider, some form of grouping.
In node there is one task:
```
    <task name="MethodRuntime" value="1" symbol="JSCRIPT_METHOD_RUNTIME_TASK">
        <opcodes>
            <opcode name="MethodLoad" value="10" symbol="JSCRIPT_METHOD_METHODLOAD_OPCODE"/>
        </opcodes>
    </task> 
```

This grouping enables consumers to query only for specific tasks and opcode combinations. The opcodes
are specific to the task `MethodRuntime`. But there are also global opcodes:
```
    <opcodes>
        <opcode name="NODE_HTTP_SERVER_REQUEST" value="10"/>
        <opcode name="NODE_HTTP_SERVER_RESPONSE" value="11"/>
        <opcode name="NODE_HTTP_CLIENT_REQUEST" value="12"/>
        <opcode name="NODE_HTTP_CLIENT_RESPONSE" value="13"/>
        <opcode name="NODE_NET_SERVER_CONNECTION" value="14"/>
        <opcode name="NODE_NET_STREAM_END" value="15"/>
        <opcode name="NODE_GC_START" value="16"/>
        <opcode name="NODE_GC_DONE" value="17"/>
        <opcode name="NODE_V8SYMBOL_REMOVE" value="21"/>
        <opcode name="NODE_V8SYMBOL_MOVE" value="22"/>
        <opcode name="NODE_V8SYMBOL_RESET" value="23"/>
    </opcodes>
```

Templates are used to define event specific data that the provider includes with an event. For example in
node one template looks like this:
```
    <template tid="node_connection">
      <data name="fd" inType="win:UInt32" />
      <data name="port" inType="win:UInt32" />
      <data name="remote" inType="win:AnsiString" />
      <data name="buffered" inType="win:UInt32" />
   </template>
```

Events have to be defined and can refer to template like the following:
```
    <event value="2" 
      opcode="NODE_HTTP_SERVER_RESPONSE"
      template="node_connection"
      symbol="NODE_HTTP_SERVER_RESPONSE_EVENT"
      message="$(string.NodeJS-ETW-provider.event.2.message)"
      level="win:Informational"/>
```

Ok, so we can see how the provider and events are configured. This information is used by the `mc` tool
to generate headers and a binary file.

The provider headers is in src/node_win32_etw_provider.h. For each of the opcodes there are function declarations
and init_etw() and shutdown_etw(). There is an internal header `src/node_win32_etw_provider-inl.h` which includes
the generated `node_etw_provider.h` which is located in 'tools/msvs/genfiles/node_etw_provider.h'.

### Performace counters 
Are used to provide information how node is performing.
There is condition in node.gypi that depends on `node_perfctr` which looks like this:
```
    [ 'node_use_perfctr=="true"', {
      'defines': [ 'HAVE_PERFCTR=1' ],
      'dependencies': [ 'node_perfctr' ],
      'sources': [
        'src/node_win32_perfctr_provider.h',
        'src/node_win32_perfctr_provider.cc',
        'src/node_counters.cc',
        'src/node_counters.h',
        'tools/msvs/genfiles/node_perfctr_provider.rc',
      ]
    } ],
```

This file is used in the `node_perfctr` (node performance counter) target and it is an action target that invokes `ctrpp`
The input to ctrpp is src/res/node_perfctr_provider.man
```
    {
      'action_name': 'node_perfctr_man',
      'inputs': [ 'src/res/node_perfctr_provider.man' ],
      'outputs': [
         'tools/msvs/genfiles/node_perfctr_provider.h',
         'tools/msvs/genfiles/node_perfctr_provider.rc',
         'tools/msvs/genfiles/MSG00001.BIN',
      ],
      'action': [ 'ctrpp <@(_inputs) '
        '-o tools/msvs/genfiles/node_perfctr_provider.h '
        '-rc tools/msvs/genfiles/node_perfctr_provider.rc'
      ]
    },
```

`-o`  specifies the header that will be generated.
`-rc` specifies the file that will be generated.

'src/node_win32_perfctr_provider.cc' includes `node_perfctr_provider.h`
Notice this this is much like the ETW and manifest based. Lets look at the manifest. Instead of containing events this
file contains counters.
The MSG00001.BIN file is a binary resource file for each language you specify (only one in our case).

But when is the actual .res file created?  
I can see these are part of the statically linked library:
```console
    dumpbin /archivemembers Release/lib/node.lib
```

I can see that path is Release\obj\node\node_etw_provider.res and Release\obj\node\node_perfctr_provider.res. 
Do these files somehow refer to node.lib causing /WHOLEARCHIVE to fail?
Is there some way to exclude object from the lib specified by /WHOLEARCHIVE?
A .res file is a compiled resource file generated by rc.exe. By default this is created in the same directory as the .rc file.

### CTRPP
The CTRPP tool is a pre-processor that parses and validates your counters manifest. The tool also generates code that you use to provide your counter data. 

### Tracker.exe
Scans .tlog files to figure out what a projects output files are. This is used to know if something should be updated/recompiled I guess. Adding this as
you might come across an issue on windows where the last things that is logged is output from the tracker but infact if there is a link error this would 
not be something that the tracker would report, so you'll need to look further back in the console output. 

### Inspecting a .lib on windows
```console
    DUMPBIN /ARCHIVEMEMBERS Release\lib\node.lib
```

### Linux Trace Toolkit: next generation (lttng)
LTTng is an open source tracing framework for Linux.


Does it make sense to have the node.res file when node is linked as a static library? Would'nt the case be that the icon and other resource info
be that of the embedders.

But out about the ETW and performance counters resources?
Resources (as far as I can tell) are intended to be linked to the exe or a dynamic library. It does not seem like it is possible to add 


### Node executable linking
By default, just building node using ./configure && make -j8 the node executable will be statically linked:
```console
    $ otool -L out/Debug/node
    out/Debug/node:
      /System/Library/Frameworks/CoreFoundation.framework/Versions/A/CoreFoundation (compatibility version 150.0.0, current version 1259.11.0)
      /usr/lib/libSystem.B.dylib (compatibility version 1.0.0, current version 1226.10.1)
      /usr/lib/libc++.1.dylib (compatibility version 1.0.0, current version 120.1.0)
```

### GYP
For addon tests in node there can be the need to link against a library in the root directory. The problem is that you cannot use <(PRODUCT_DIR) as
that is the product dir for the addon itself. There is also the issue that the output directory is different on windows, it is not `out` but instead
the `Release` and `Debug` directories are in the root of the project. But there are variables available by the respective node-gyp evironments, 
for example on Window you can use:
```

    ['OS=="win"', {
      'libraries': ['../../../../$(Configuration)/lib/zlib.lib'],
    }, {
      'libraries': ['../../../../out/$(BUILDTYPE)/libzlib.a'],
    }],
```


### serdes
This is a built in module for serialization/deserialization which is currently marked as experimental.
It allows for serializing JavaScript values in a way compatiable with https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Structured_clone_algorithm, 
which an algorithm for copying complex JavaScript objects and used internally for transferring data to and from Web Workers view postMessage().


### Failing test without network connection
```console
make[1]: Leaving directory `/root/rpmbuild/BUILD/node-v8.7.0'
/usr/bin/python2.7 tools/test.py --mode=release -J \
async-hooks \
abort doctool es-module inspector known_issues message parallel pseudo-tty sequential \
addons addons-napi
=== release test-dgram-membership ===
Path: parallel/test-dgram-membership
assert.js:41
  throw new errors.AssertionError({
  ^

AssertionError [ERR_ASSERTION]: Got unwanted exception.
    at innerThrows (assert.js:653:7)
    at Function.doesNotThrow (assert.js:665:3)
    at Object.<anonymous> (/root/rpmbuild/BUILD/node-v8.7.0/test/parallel/test-dgram-membership.js:83:10)
    at Module._compile (module.js:624:30)
    at Object.Module._extensions..js (module.js:635:10)
    at Module.load (module.js:545:32)
    at tryModuleLoad (module.js:508:12)
    at Function.Module._load (module.js:500:3)
    at Function.Module.runMain (module.js:665:10)
    at startup (bootstrap_node.js:187:16)
Command: out/Release/node /root/rpmbuild/BUILD/node-v8.7.0/test/parallel/test-dgram-membership.js
=== release test-dgram-multicast-set-interface-lo ===
Path: parallel/test-dgram-multicast-set-interface-lo
assert.js:41
  throw new errors.AssertionError({
  ^

AssertionError [ERR_ASSERTION]: undefined == true
    at Object.<anonymous> (/root/rpmbuild/BUILD/node-v8.7.0/test/parallel/test-dgram-multicast-set-interface-lo.js:64:8)
    at Module._compile (module.js:624:30)
    at Object.Module._extensions..js (module.js:635:10)
    at Module.load (module.js:545:32)
    at tryModuleLoad (module.js:508:12)
    at Function.Module._load (module.js:500:3)
    at Function.Module.runMain (module.js:665:10)
    at startup (bootstrap_node.js:187:16)
    at bootstrap_node.js:608:3
Command: out/Release/node /root/rpmbuild/BUILD/node-v8.7.0/test/parallel/test-dgram-multicast-set-interface-lo.js
[03:09|% 100|+ 1893|-   2]: Done
```

### Trouble shooting DNS issue on RHEL
BIND (the NameServer) consists of a set of dns related programs. The nameserver itself is called named. There is
an admin tool named rdnc and dig or debugging.
named reads it's configuration from /etc/named.conf.

#### /etc/nsswitch.conf
Before this configuration file existed name service lookups were hardcoded into the c library, as well as the search order. 
Name Service Switch was introduced by Sun Microsystems in there C library implementation in Solaris 2. This uses modules
which allowes for adding new services without adding them to the GNU C library.


‘unavail’
The service is permanently unavailable. This can either mean the needed file is not available, or, for DNS, the server 
is not available or does not allow queries. The default action is continue.

For the hosts and networks databases the default value is dns [!UNAVAIL=return] files. I.e., the system is prepared for 
the DNS service not to be available but if it is available the answer it returns is definitive.


Regarding test/parallel/test-net-connect-immediate-finish.js and test/parallel/test-net-better-error-messages-port-hostname.js the and the error:
```console
Path: parallel/test-net-connect-immediate-finish
assert.js:41
  throw new errors.AssertionError({
  ^
AssertionError [ERR_ASSERTION]: 'EAI_AGAIN' === 'ENOTFOUND'
    at Socket.client.once.common.mustCall (/builddir/build/BUILD/node-v8.6.0/test/parallel/test-net-connect-immediate-finish.js:35:10)
    at Socket.<anonymous> (/builddir/build/BUILD/node-v8.6.0/test/common/index.js:514:15)
    at Object.onceWrapper (events.js:316:30)
    at emitOne (events.js:115:13)
    at Socket.emit (events.js:210:7)
    at emitErrorNT (internal/streams/destroy.js:64:8)
    at _combinedTickCallback (internal/process/next_tick.js:138:11)
    at process._tickCallback (internal/process/next_tick.js:180:9)
Command: out/Release/node /builddir/build/BUILD/node-v8.6.0/test/parallel/test-net-connect-immediate-finish.js
```

This only occurs when there is no available dns (you might need to comment it out in /etc/resolve.conf to reproduce this), I manually update /etc/nsswitch.conf and the update the hosts entry:

    hosts:      files dns

Notice that the last entry is dns which so it will return the result which will be 
Using the default (at least this is the default for the RHEL version that we are using in our image: registry.access.redhat.com/rhel7) this error does not occur as the default is:

hosts:      files dns myhostname

Since this is an external configuration that could be different for different installations I think the best option is to update the test to be able to handle this situation. 
I'll create a pull request and see what people think.


### Inspector console
In the start function of lib/internal/bootstrap_node.js we have a function that sets up the inspector console.
After the global console object is set up, it is passed to setupInspector(originalConsole, wrappedConsole, Module);
It does so with the help of a builtin module 'inspector' which can be found in src/inspector_js_api.cc.

    const { addCommandLineAPI, consoleCall } = process.binding('inspector');


    wrappedConsole[key] = consoleCall.bind(wrappedConsole,
                                           originalConsole[key],
                                           wrappedConsole[key],
                                           config);

The above will create a new function which has its `this` set to wrappedConsole
```console
    $ out/Debug/node
    [Function: log] [Function: bound log] {}
    [Function: info] [Function: bound log] {}
    [Function: warn] [Function: bound warn] {}
    [Function: error] [Function: bound warn] {}
    [Function: dir] [Function: bound dir] {}
    [Function: time] [Function: bound time] {}
    [Function: timeEnd] [Function: bound timeEnd] {}
    [Function: trace] [Function: bound trace] {}
    [Function: assert] [Function: bound assert] {}
    [Function: clear] [Function: bound clear] {}
    [Function: count] [Function: bound count] {}
    [Function: group] [Function: bound group] {}
    [Function: groupCollapsed] [Function: bound group] {}
    [Function: groupEnd] [Function: bound groupEnd] {}
```

So for all of the above functions they will now got through the ConsoleCall function in inspector_agent.cc which will check if the inspector is enabled, which is done by passing --inspect,  for this process:
```console

    if (env->inspector_agent()->enabled()) {
      ...
    }
```

In this case the writing to stdout will be performed twice, once to the inspector using:

```c++
    inspector_method.As<Function>()->Call(context,
                                          info.Holder(),
                                          call_args.size(),
                                          call_args.data()).IsEmpty());
```

For example you'll seen the output in both stdout and in chrome's console.


### ccache
```console
    $ git clone https://github.com/ccache/ccache.git
    $ cd ccache
    $ ./autogen.sh
    $ ./configure && make 
```

Add ccache to your PATH. 
```console
    export CC="ccache clang -Qunused-arguments"
    export CXX="ccache clang++ -Qunused-arguments"
```

Now you can configure and build as normal.

To see what is in the cache:
```console
   $ ccache -s
```

To clear the cache:
```console
   $ ccache -C 
```


### GetPeerCertificate

```c++
    STACK_OF(X509)* ssl_certs = SSL_get_peer_cert_chain(w->ssl_);
```

STACK_OF is a macro defined in `deps/openssl/openssl/crypto/stack/safestack.h` and will expand to:
```c++
    struct stack_st_X509* ssl_certs = SSL_get_peer_cert_chain(w->ssl_);
```

A Local<Object> instance holds a pointer to an Object. If you pass this instance to a function a copy of the 
object will be created. But both instances will point to the same object (the pointer is copied). 

For example:
```c++

    static void AddIssuer(X509** cert,
                          const STACK_OF(X509)* const peer_certs,
                          Local<Object> info,
                          Environment* const env) {

    ...
    Local<Object> ca_info = X509ToObject(env, ca);
    // Now we want to update what info points to so that is points to the value of ca_info instead.
```
```console
    (lldb) expr info
    (v8::Local<v8::Object>) $4 = (val_ = 0x0000000106014b08)
    (lldb) expr *info
    (v8::Object *) $6 = 0x0000000106014b08
```


The copy constructor for Local<> will copy the value, which includes the pointer so these are separate
objects:
```console
    (lldb) p &result
    (v8::Local<v8::Object> *) $86 = 0x00007fff5fbf04b0
    (lldb) p &info
    (v8::Local<v8::Object> *) $87 = 0x00007fff5fbf04a8
```

But what is info supposed to represent?
Well they both initially point to the same thing as we can see but as mentioned they are separate objects. So the following will update what they both (result and info) point to:
```c++
    Local<Object> ca_info = X509ToObject(env, ca);
    info->Set(env->issuercert_string(), ca_info);
```

But the following will cause info to copy constructed (I think) to ca_info so the connection with result is lost here. That is incorrect!
What is happening is that the first time info == result so the issuer is set on it, and it contains the ca_info instance. So result can get to it. Next when info is set to ca_info:

```c++
    info = ca_info;
```

This is for the next iteration and if there are more they will be chained to ca_info using the info->Set(env->issuercert_string(), ca_info), which remember can be accessed from result via its issuercert_string property. 
```console
result -> #issuercertificate -> ca_info -> #issuercerficate -> ca_info
```

If this is not done result will not be linked to them all and the chain broken.


### Crypto SetKey

```c++
    void DiffieHellman::SetPublicKey(const FunctionCallbackInfo<Value>& args) {
      SetKey(args, [](DH* dh, BIGNUM* num) { DH_set0_key(dh, num, nullptr); },
         "Public key");
    }

    void DiffieHellman::SetPrivateKey(const FunctionCallbackInfo<Value>& args) {
    #if OPENSSL_VERSION_NUMBER >= 0x10100000L && OPENSSL_VERSION_NUMBER < 0x10100070L
    // Older versions of OpenSSL 1.1.0 have a DH_set0_key which does not work for
    // Node. See https://github.com/openssl/openssl/pull/4384.
    #error "OpenSSL 1.1.0 revisions before 1.1.0g are not supported"
    #endif
    SetKey(args, [](DH* dh, BIGNUM* num) { DH_set0_key(dh, nullptr, num); },
       "Private key");
    }

    void DiffieHellman::SetKey(const v8::FunctionCallbackInfo<v8::Value>& args,
                           void (*set_field)(DH*, BIGNUM*), const char* what) {
      ...
```

Notice that the second paramenter to SetKey is a function pointer:

```c++
     void function_name(DH* dh, BIGHUM* n);
```

I noticed that DH_set0_key returns one and wondering why as the function pointer is declared as returning
void:

```c++
    static int DH_set0_key(DH* dh, BIGNUM* pub_key, BIGNUM* priv_key) {
      if (pub_key != nullptr) {
        BN_free(dh->pub_key);
        dh->pub_key = pub_key;
      }
      if (priv_key != nullptr) {
        BN_free(dh->priv_key);
        dh->priv_key = priv_key;
      }
      return 1;
    }
```

And the call looks like this in `SetKey`:

```c++
    set_field(dh->dh, num);
```


### Object Set
The `Set` functions in include/v8.h for Object are deprecated. The ones that should be used are the ones that
return a MayBe<bool> result instead. This means that you also have to call either FromJust() or ToChecked()
before using/retrieving the value.
```console
     3109 class V8_EXPORT Object : public Value {
     3110  public:
     3111   V8_DEPRECATE_SOON("Use maybe version",
     3112                     bool Set(Local<Value> key, Local<Value> value));
     3113   V8_WARN_UNUSED_RESULT Maybe<bool> Set(Local<Context> context,
     3114                                         Local<Value> key, Local<Value> value);
     3115
     3116   V8_DEPRECATE_SOON("Use maybe version",
     3117                     bool Set(uint32_t index, Local<Value> value));
     3118   V8_WARN_UNUSED_RESULT Maybe<bool> Set(Local<Context> context, uint32_t index,
     3119                                         Local<Value> value);
```


### Unqualified name lookup
```c++
    void SecureContext::Initialize(Environment* env, Local<Object> target) {
      ...
      env->SetProtoMethod(t, "init", SecureContext::Init);
```

Even without the `SecureContext` namespace `Init` will be resolved correctly as resolution will 
look in class of the member funtion which is SecureContext.

```c++
    const binding = process.binding('crypto');
```

This will invoke SecureContext::Initialize.

```javascript
const server = tls.Server(options, function(socket) {

exports.Server = require('_tls_wrap').Server;
```

_tls_common.js:
```javascript
const binding = process.binding('crypto');
const NativeSecureContext = binding.SecureContext;

function SecureContext(secureProtocol, secureOptions, context) {
  ... 
  this.context = new NativeSecureContext();
}
```
So we can see that SecureContext::New is bound to NativeSecureContext.


### node_crypto_bio

```c++
    static const BIO_METHOD method = {
      BIO_TYPE_MEM,
      "node.js SSL buffer",
      Write,
      Read,
      Puts,
      Gets,
      Ctrl,
      New,
      Free,
      nullptr
    };


BIO* NodeBIO::NewFixed(const char* data, size_t len) {
  BIO* bio = New();

  if (bio == nullptr ||
      len > INT_MAX ||
      BIO_write(bio, data, len) != static_cast<int>(len) ||
      BIO_set_mem_eof_return(bio, 0) != 1) {
    BIO_free(bio);
    return nullptr;
  }

  return bio;
}
```

`BIO_set_mem_eof_return` is a macro defined as:

```c++
    # define BIO_set_mem_eof_return(b,v) BIO_ctrl(b,BIO_C_SET_BUF_MEM_EOF_RETURN,v,NULL)
```

which in our case would give:

```c++
    BIO_strl(bio, BIO_C_SET_BUF_MEM_EOF_RETURN, 0, NULL)
```

This call will end up in bio_lib.c and its BIO_ctrl function:

```c++
    ret = b->method->ctrl(b, cmd, larg, parg);
```

Now, b is the BIO we passed in, cmd is BIO_C_SET_BUF_MEM_EOF_RETURN (130), larg is 0, and parg is NULL.
The interesting thing here is that b->method will be the method defined in node_crypto_bio.cc:
```console
    (lldb) p *b->method
    (BIO_METHOD) $12 = {
      type = 1025
      name = 0x0000000101d44053 "node.js SSL buffer"
      bwrite = 0x000000010192ed10 (node`node::crypto::NodeBIO::Write(bio_st*, char const*, int) at node_crypto_bio.cc:142)
      bread = 0x000000010192e940 (node`node::crypto::NodeBIO::Read(bio_st*, char*, int) at node_crypto_bio.cc:92)
      bputs = 0x000000010192ef90 (node`node::crypto::NodeBIO::Puts(bio_st*, char const*) at node_crypto_bio.cc:151)
      bgets = 0x000000010192efe0 (node`node::crypto::NodeBIO::Gets(bio_st*, char*, int) at node_crypto_bio.cc:156)
      ctrl = 0x000000010192f2f0 (node`node::crypto::NodeBIO::Ctrl(bio_st*, int, long, void*) at node_crypto_bio.cc:182)
      create = 0x000000010192e7c0 (node`node::crypto::NodeBIO::New(bio_st*) at node_crypto_bio.cc:68)
      destroy = 0x000000010192e830 (node`node::crypto::NodeBIO::Free(bio_st*) at node_crypto_bio.cc:77)
      callback_ctrl = 0x0000000000000000
    }
```

So b->method->ctrl will call NodeBIO::Ctrl:
The first things that is done is that the NodeBIO instance is retrieved from the bio:

```c++
    NodeBIO* nbio = FromBIO(bio);
```


'FromBIO': 

```c++
    CHECK_NE(BIO_get_data(bio), nullptr);
    return static_cast<NodeBIO*>(BIO_get_data(bio));
```

BIO_get_data is a macro for OPENSSL versions greater than 1.0.1:

```c++
    #define BIO_get_data(bio) bio->ptr
```

So we can see that the BIO's ponter is a pointer to the NodeBIO instance.

```c++
    case BIO_C_SET_BUF_MEM_EOF_RETURN
      nbio->set_eof_return(num);
      break;
```

So we are calling 'set_eof_return' with 
```console
    (lldb) p num
    (long) $14 = 0
```

### Node Crypto BIO Buffer
node_crypto_bio.h has a private Buffer class.

To understand what this buffer does and how it works lets take a look at a usage of it...
SecureContext::AddCACert:

```c++
    ...
    BIO* bio = LoadBIO(env, args[0]);
```
    
In this case args[0] is a certificate (a string) so the following path in LoadBIO will be taken:

```c++
    return NodeBIO::NewFixed(*s, s.length());
```

NodeBIO::NewFixed will call:

```c++
    if (bio == nullptr ||
      len > INT_MAX ||
      BIO_write(bio, data, len) != static_cast<int>(len) ||
      BIO_set_mem_eof_return(bio, 0) != 1) {
      BIO_free(bio);
      return nullptr;
    }
```

We are interested in the BIO_write call which will call Write on the bio's method which is NodeBIO::Write:

```c++
    void NodeBIO::Write(const char* data, size_t size) {
    ...
    // Allocate initial buffer if the ring is empty
    TryAllocateForWrite(left);

    void NodeBIO::TryAllocateForWrite(size_t hint) {
      Buffer* w = write_head_;
      Buffer* r = read_head_;
      // If write head is full, next buffer is either read head or not empty.
      if (w == nullptr ||
        (w->write_pos_ == w->len_ &&
         (w->next_ == r || w->next_->write_pos_ != 0))) {
        size_t len = w == nullptr ? initial_ : kThroughputBufferLength;
        if (len < hint)
          len = hint;
        Buffer* next = new Buffer(env_, len);

        if (w == nullptr) {
          next->next_ = next;
          write_head_ = next;
          read_head_ = next;
        } else {
          next->next_ = w->next_;
          w->next_ = next;
        }
      }
    }
```

First time entering this function w and r will be null as write_head_ and read_head_ will be null:
```console
    (lldb) expr w
    (node::crypto::NodeBIO::Buffer *) $24 = 0x0000000000000000
    (lldb) expr r
    (node::crypto::NodeBIO::Buffer *) $25 = 0x0000000000000000

    size_t len = initial_; // since w == nullptr;
    (lldb) p len
    (size_t) $28 = 1024
    (lldb) p hint
    (size_t) $29 = 1224
    if (len < hint)  // 1024 < 1224
      len = 1224
    
    Buffer* next = new Buffer(env_, 1224);


    Buffer constructor: 
    data_ = new char[len];

    if (w == nullptr) {
      next->next_ = next;
      write_head_ = next;
      read_head_ = next;
    }
```
So initially there is only a single entry all pointing to this first one.
Next, we are back in NodeBIO::Write:

```c++
    while (left > 0) { // first time left is == 1224
    ....
    memcpy(write_head_->data_ + write_head_->write_pos_, data + offset, to_write);
```

void* memcpy( void* dest, const void* src, std::size_t count ) so the destination in our case is
write_head_data_ + write_head_write_pos_, the data to be copied is data + 0, and to_write is 1224.
 
### ClientHelloParser
node_crypto.h has a class member that is declared like:

```c++
  ClientHelloParser hello_parser_;
```

So when a new SSLWrap is created constructor of ClientHelloParser will be called. Looking at the constructor for ClientHelloParser
it first initilizes its fields and then calls Reset():

```c++
      ClientHelloParser() : state_(kEnded),
                        onhello_cb_(nullptr),
                        onend_cb_(nullptr),
                        cb_arg_(nullptr),
                        session_size_(0),
                        session_id_(nullptr),
                        servername_size_(0),
                        servername_(nullptr),
                        ocsp_request_(0),
                        tls_ticket_size_(0),
                        tls_ticket_(nullptr) {
      Reset();
     }

    inline void ClientHelloParser::Reset() {
      frame_len_ = 0;
      body_offset_ = 0;
      extension_offset_ = 0;
      session_size_ = 0;
      session_id_ = nullptr;
      tls_ticket_size_ = -1;
      tls_ticket_ = nullptr;
      servername_size_ = 0;
      servername_ = nullptr;
      ocsp_request_ = 0;  // added by me
    }
```

I can't find a reason for not resetting ocsp_request_ here. I'll create a PR to get some feedback.

### v8::Object::Set and exceptions
This section goes through a call to Set and the possible exceptions that might be throws and 
how they can be handled.
```console
    $ lldb -- out/Debug/node --inspect-brk test/parallel/test-tls-legacy-onselect.js
```

```c++
    env->SetProtoMethod(t, "setSNICallback",  Connection::SetSNICallback);
```
```console
    (lldb) br s -f node_crypto.cc -l 3594
    (lldb) r
```

Open chrome://inspect and set a break point on this line:

```c++
    const pair = tls.createSecurePair(null, true, false, false);
```

A `SecurePair` is a pair of streams to do encrypted communication with.

```c++
    Local<Object> obj = Object::New(env->isolate());
    obj->Set(env->context(), env->onselect_string(), args[0]).FromJust();
```

We are creating a new Local<v8::Object> and setting a property on that object. The 
property name is taken from the Environment function onselect_string(). This function
is generated by a macro and by:

```c++
    V(onselect_string, "onselect")
```

Now, `obj->Set` is only setting a property to be a function and here is a check in this 
specific code path that arg[0] exists and is a function. 
```console
    (lldb) jlh obj
    0x10583e1675b9: [JS_OBJECT_TYPE]
    - map = 0x105859b5fe79 [FastProperties]
    - prototype = 0x105819c04679
    - elements = 0x1058b5982241 <FixedArray[0]> [HOLEY_ELEMENTS]
    - properties = 0x1058b5982241 <FixedArray[0]> {
      #onselect: 0x10583e167571 <JSFunction (sfi = 0x1058fc5f3969)> (const data descriptor)
    }
```


But, there is a chance where an exception might be thrown and that is if there is a setter for `onselect'. This
migth look like:

```c++
    Object.defineProperty(Object.prototype, 'onselect', {
      set: function(f) {
        console.log('throw error from setter...');
        throw Error('dummy setter error');
      }
    });
```

Calling `obj->Set(env->context(), env->onselect_string(), args[0]).FromJust();` would cause an exception to be
thrown:

```console
$ out/Debug/node  test/parallel/test-tls-legacy-onselect.js
throw error from setter...
FATAL ERROR: v8::FromJust Maybe value is Nothing.
 1: node::Abort() [/Users/danielbevenius/work/nodejs/node/out/Debug/node]
 2: node::OnFatalError(char const*, char const*) [/Users/danielbevenius/work/nodejs/node/out/Debug/node]
 3: v8::Utils::ReportApiFailure(char const*, char const*) [/Users/danielbevenius/work/nodejs/node/out/Debug/node]
 4: v8::Utils::ApiCheck(bool, char const*, char const*) [/Users/danielbevenius/work/nodejs/node/out/Debug/node]
 5: v8::V8::FromJustIsNothing() [/Users/danielbevenius/work/nodejs/node/out/Debug/node]
 6: v8::Maybe<bool>::FromJust() const [/Users/danielbevenius/work/nodejs/node/out/Debug/node]
 7: node::crypto::Connection::SetSNICallback(v8::FunctionCallbackInfo<v8::Value> const&) [/Users/danielbevenius/work/nodejs/node/out/Debug/node]
 8: v8::internal::FunctionCallbackArguments::Call(void (*)(v8::FunctionCallbackInfo<v8::Value> const&)) [/Users/danielbevenius/work/nodejs/node/out/Debug/node]
 9: v8::internal::MaybeHandle<v8::internal::Object> v8::internal::(anonymous namespace)::HandleApiCallHelper<false>(v8::internal::Isolate*, v8::internal::Handle<v8::internal::HeapObject>, v8::internal::Handle<v8::internal::HeapObject>, v8::internal::Handle<v8::internal::FunctionTemplateInfo>, v8::internal::Handle<v8::internal::Object>, v8::internal::BuiltinArguments) [/Users/danielbevenius/work/nodejs/node/out/Debug/node]
10: v8::internal::Builtin_Impl_HandleApiCall(v8::internal::BuiltinArguments, v8::internal::Isolate*) [/Users/danielbevenius/work/nodejs/node/out/Debug/node]
11: v8::internal::Builtin_HandleApiCall(int, v8::internal::Object**, v8::internal::Isolate*) [/Users/danielbevenius/work/nodejs/node/out/Debug/node]
12: 0x10e0737043c4
13: 0x10e0737f1600
Abort trap: 6
```
Notice the `FromJust()` which will crash if there is an error in the JavaScript setter.

But instead what should happen is (in SetSNICallback):
```c++

if (obj->Set(env->context(), env->onselect_string(), args[0]).IsNothing()) {
    return;
}
```
`return` where will return to `api-arguments.cc` line 26:
```c++
return GetReturnValue<Object>(isolate);
```
`GetReturnValue` can be found in `api-arguments.h'. It looks like this will see if there is a return value and 
return a Handle to it or an empty Handle.
After returning from `v8::internal::FunctionCallbackArguments::Call` we are the builtins-api.cc HandleApiCallHelper:

```c++
    Handle<Object> result = custom.Call(callback); // this was our call to SetSNICallback
 
    RETURN_EXCEPTION_IF_SCHEDULED_EXCEPTION(isolate, Object);
```

`RETURN_EXCEPTION_IF_SCHEDULED_EXCEPTION` is a macro:

    #define RETURN_EXCEPTION_IF_SCHEDULED_EXCEPTION(isolate, T) \
      RETURN_VALUE_IF_SCHEDULED_EXCEPTION(isolate, MaybeHandle<T>())

Which will expend to
```c++
      RETURN_VALUE_IF_SCHEDULED_EXCEPTION(isolate, MaybeHandle<Object>())


    #define RETURN_VALUE_IF_SCHEDULED_EXCEPTION(isolate, value) \
      do {                                                      \
        Isolate* __isolate__ = (isolate);                       \
        DCHECK(!__isolate__->has_pending_exception());          \
        if (__isolate__->has_scheduled_exception()) {           \
          __isolate__->PromoteScheduledException();             \
          return value;                                         \
        }                                                       \
      } while (false)
```

So this will expand to (which will be places in HandleApiCallHelper replacing RETURN_EXCEPTION_IF_SCHEDULED_EXCEPTION) as:

```c++
        Isolate* __isolate__ = (isolate);            
        DCHECK(!__isolate__->has_pending_exception());
        if (__isolate__->has_scheduled_exception()) {
          __isolate__->PromoteScheduledException(); 
          return MaybeHandle<Object(); 
        }                             
```

In this case there will be a pending exception so __isolate__->PromoteScheduledException will be called.

PromoteScheduledException:
```c++
Object* thrown = scheduled_exception();
clear_scheduled_exception();
return ReThrow(thrown);
```

```console
(lldb) job thrown
0x11728e9a5401: [JS_ERROR_TYPE]
 - map = 0x1172bc86cf79 [FastProperties]
 - prototype = 0x117204d0d679
 - elements = 0x1172afa82241 <FixedArray[0]> [HOLEY_SMI_ELEMENTS]
 - properties = 0x11728e9a5469 <PropertyArray[3]> {
    #stack: 0x1172b59ca9b9 <AccessorInfo> (const accessor descriptor)
    #message: 0x1172c10b6711 <String[18]: dummy setter error> (data field 0) properties[0]
    0x1172afa86899 <Symbol: detailed_stack_trace_symbol>: 0x11728e9a5491 <FixedArray[5]> (data field 1) properties[1]
    0x1172afa87069 <Symbol: stack_trace_symbol>: 0x11728e9a5cf1 <JSArray[26]> (data field 2) properties[2]
 }
```

ReThrow:
```c++
set_pending_exception(exception);
return heap()->exception();
```

```c++
void Isolate::set_pending_exception(Object* exception_obj) {
  DCHECK(!exception_obj->IsException(this));
  thread_local_top_.pending_exception_ = exception_obj;
}
```

Now when returning we will land in `bootstrap_node.js` and `process_fatalException` callback:
```javascript
process._fatalException = function(er) {
  ...
     if (!caught)
        caught = process.emit('uncaughtException', er);

      // If someone handled it, then great.  otherwise, die in C++ land
      // since that means that we'll exit the process, emit the 'exit' event
      if (!caught) {
        try {
          if (!process._exiting) {
            process._exiting = true;
            process.emit('exit', 1);
          }
        } catch (er) {
          // nothing to be done about it at this point.
        }

      } else {
        ...
      }

      return caught;
}
```
Where er will be:
```console
er = Error: dummy setter error at Object.set
```

The error would be: 

```console
throw error from setter...
/Users/danielbevenius/work/nodejs/node/test/parallel/test-tls-legacy-onselect.js:13
    throw Error('dummy setter error');
    ^

Error: dummy setter error
    at Object.set (/Users/danielbevenius/work/nodejs/node/test/parallel/test-tls-legacy-onselect.js:13:11)
    at Server.<anonymous> (/Users/danielbevenius/work/nodejs/node/test/parallel/test-tls-legacy-onselect.js:20:12)
    at Server.<anonymous> (/Users/danielbevenius/work/nodejs/node/test/common/index.js:520:15)
    at Server.emit (events.js:126:13)
    at TCP.onconnection (net.js:1595:8)
```
This is more informative about the error and easier to understand where it happend.


### v8::Object::Set walkthrough
This function can be found in `api.cc`. In the walkthrough what we are setting is a callback on an object.

    Maybe<bool> v8::Object::Set(v8::Local<v8::Context> context, v8::Local<Value> key, v8::Local<Value> value) {
      auto isolate = reinterpret_cast<i::Isolate*>(context->GetIsolate());
      ENTER_V8(isolate, context, Object, Set, Nothing<bool>(), i::HandleScope);
      ...
    }

In this walk through the arguments are:
```console
(lldb) jlh key
#onselect

(lldb) jlh value
0x2cfe064e7d11: [Function]
 - map = 0x2cfea4f02411 [FastProperties]
 - prototype = 0x2cfec3204631
 - elements = 0x2cfea2482241 <FixedArray[0]> [HOLEY_ELEMENTS]
 - initial_map =
 - shared_info = 0x2cfec4a73ae1 <SharedFunctionInfo>
 - name = 0x2cfea2482431 <String[0]: >
 - formal_parameter_count = 0
 - kind = [ NormalFunction ]
 - context = 0x2cfe064e1be1 <FixedArray[5]>
 - code = 0x6b17cf07d01 <Code BUILTIN>
 - source code = () {
    context.actual++;
    return fn.apply(this, arguments);
  }
 - properties = 0x2cfea2482241 <FixedArray[0]> {
    #length: 0x2cfed24ca4f1 <AccessorInfo> (const accessor descriptor)
    #name: 0x2cfed24ca561 <AccessorInfo> (const accessor descriptor)
    #prototype: 0x2cfed24ca5d1 <AccessorInfo> (const accessor descriptor)
 }

 - feedback vector: 0x2cfe7b857fa9: [FeedbackVector] in OldSpace
 - length: 9
 SharedFunctionInfo: 0x2cfec4a73ae1 <SharedFunctionInfo>
 Optimized Code: 0
 Invocation Count: 1
 Profiler Ticks: 0
 Slot #0 LoadProperty PREMONOMORPHIC
  [0]: 0x2cfea2486d59 <Symbol: premonomorphic_symbol>
  [1]: 0x2cfea24871c1 <Symbol: uninitialized_symbol>
 Slot #2 StoreNamedStrict PREMONOMORPHIC
  [2]: 0x2cfea2486d59 <Symbol: premonomorphic_symbol>
  [3]: 0x2cfea24871c1 <Symbol: uninitialized_symbol>
 Slot #4 BinaryOp MONOMORPHIC
  [4]: 1
 Slot #5 Call MONOMORPHIC
  [5]: 0x2cfe7b858019 <WeakCell value= 0x2cfec32088c1 <JSFunction apply (sfi = 0x2cfea24b2181)>>
  [6]: 1
 Slot #7 LoadProperty PREMONOMORPHIC
  [7]: 0x2cfea2486d59 <Symbol: premonomorphic_symbol>
  [8]: 0x2cfea24871c1 <Symbol: uninitialized_symbol>
```
This is the callback passed in :
```js
pair.ssl.setSNICallback(common.mustCall(function() {
  raw.destroy();
  server.close();
}));
```

`ENTER_V8` is a macro which is defined as:

```c++
    #define ENTER_V8(isolate, context, class_name, function_name, bailout_value, \
                     HandleScopeClass)                                           \
      ENTER_V8_HELPER_DO_NOT_USE(isolate, context, class_name, function_name,    \
                             bailout_value, HandleScopeClass, true)
```

So `ENTER_V8(isolate, context, Object, Set, Nothing<bool>(), i::HandleScope);` would expend to:

```c++
    ENTER_V8_HELPER_DO_NOT_USE(isolate, context, Object, Set, Nothing<bool>(), i::HandleScope, true)

    #define ENTER_V8_HELPER_DO_NOT_USE(isolate, context, Object,      \
                                       function_name, bailout_value,  \
                                       HandleScopeClass, do_callback) \
      if (IsExecutionTerminatingCheck(isolate)) {                     \
        return bailout_value;                                         \
      }                                                               \
      HandleScopeClass handle_scope(isolate);                         \
      CallDepthScope<do_callback> call_depth_scope(isolate, context); \
      LOG_API(isolate, class_name, function_name);                    \
      i::VMState<v8::OTHER> __state__((isolate));                     \
      bool has_pending_exception = false
```

So back in our v8::Object::Set function these macros would expend to:

```c++
    Maybe<bool> v8::Object::Set(v8::Local<v8::Context> context, v8::Local<Value> key, v8::Local<Value> value) {
      auto isolate = reinterpret_cast<i::Isolate*>(context->GetIsolate());
      if (IsExecutionTerminatingCheck(isolate)) {                     
        return Nothing<bool>();                                         
      }                                                               
      HandleScopeClass handle_scope(isolate);                         
      CallDepthScope<do_callback> call_depth_scope(isolate, context); 
      LOG_API(isolate, Object, Set);                    
      i::VMState<v8::OTHER> __state__((isolate));                     
      bool has_pending_exception = false;

      auto self = Utils::OpenHandle(this);
      auto key_obj = Utils::OpenHandle(reinterpret_cast<Name*>(*key));
      auto value_obj = Utils::OpenHandle(*value);
      has_pending_exception = i::Runtime::SetObjectProperty(isolate, 
                                                            self,
                                                            key_obj,
                                                            value_obj,
                                                            i::LanguageMode::kSloppy).is_null();
     RETURN_ON_FAILED_EXECUTION_PRIMITIVE(bool);
     return Just(true);
    }
```

So, lets take a closer look at i::Runtime::SetObjectProperty (`i` is V8's internal namespace) which we can find in
src/runtime/runtime-object.cc.

```c++
    // Check if the given key is an array index.
    bool success = false;
    LookupIterator it = LookupIterator::PropertyOrElement(isolate, object, key, &success);
    if (!success) return MaybeHandle<Object>();

    MAYBE_RETURN_NULL(Object::SetProperty(&it, value, language_mode,
                                          Object::MAY_BE_STORE_FROM_KEYED));
    return value;
```

`MAYBE_RETURN_NULL` macro :

```c++
    #define MAYBE_RETURN_NULL(call) MAYBE_RETURN(call, MaybeHandle<Object>())

    #define MAYBE_RETURN(call, value)         \
      do {                                    \
        if ((call).IsNothing()) return value; \
      } while (false)
```
    
So, in this case our call will expend to:

```c++
    if (Object::SetProperty(&it, value, language_mode, Object::MAY_BE_STORE_FROM_KEYED).IsNothing())
       return MaybeHandle<Object>();
```

Lets follow `SetProperty` (in src/objects.cc line 4753) and see what it does:

```c++
    Maybe<bool> Object::SetProperty(LookupIterator* it, Handle<Object> value,
                                LanguageMode language_mode,
                                StoreFromKeyed store_mode) {
    if (it->IsFound()) {
      bool found = true;
      Maybe<bool> result =
        SetPropertyInternal(it, value, language_mode, store_mode, &found);
    if (found) return result;
  }

  // If the receiver is the JSGlobalObject, the store was contextual. In case
  // the property did not exist yet on the global object itself, we have to
  // throw a reference error in strict mode.  In sloppy mode, we continue.
  if (is_strict(language_mode) && it->GetReceiver()->IsJSGlobalObject()) {
    it->isolate()->Throw(*it->isolate()->factory()->NewReferenceError(
        MessageTemplate::kNotDefined, it->name()));
    return Nothing<bool>();
  }

  ShouldThrow should_throw =
      is_sloppy(language_mode) ? kDontThrow : kThrowOnError;
  return AddDataProperty(it, value, NONE, should_throw, store_mode);
}
```

`SetPropertyInternal` (in src/objects.cc):

```c++
    ShouldThrow should_throw = is_sloppy(language_mode) ? DONT_THROW : THROW_ON_ERROR;

    switch (it->state()) {
      ...
    }
 
    lldb) p it->state()
    (v8::internal::LookupIterator::State) $22 = ACCESSOR
    (lldb) expr should_throw
    (ShouldThrow) $28 = DONT_THROW

    ....
    return SetPropertyWithAccessor(it, value, should_throw);
```

`SetPropertyWithAccessors`:
```c++

    Handle<Object> structure = it->GetAccessors();
    Handle<Object> receiver = it->GetReceiver();
    ...
    Handle<Object> setter(AccessorPair::cast(*structure)->setter(), isolate);
```

```console
(lldb) job *setter
0x2cfe064bd1b1: [Function]
 - map = 0x2cfea4f02411 [FastProperties]
 - prototype = 0x2cfec3204631
 - elements = 0x2cfea2482241 <FixedArray[0]> [HOLEY_ELEMENTS]
 - initial_map =
 - shared_info = 0x2cfe232eed51 <SharedFunctionInfo set>
 - name = 0x2cfea2485f41 <String[3]: set>
 - formal_parameter_count = 1
 - kind = [ NormalFunction ]
 - context = 0x2cfedc582a29 <FixedArray[8]>
 - code = 0x6b17cf07d01 <Code BUILTIN>
 - source code = (f) {
    console.log('throw error from setter...');
    throw Error('dummy setter error');
  }
 - properties = 0x2cfea2482241 <FixedArray[0]> {
    #length: 0x2cfed24ca4f1 <AccessorInfo> (const accessor descriptor)
    #name: 0x2cfed24ca561 <AccessorInfo> (const accessor descriptor)
    #prototype: 0x2cfed24ca5d1 <AccessorInfo> (const accessor descriptor)
 }

 - feedback vector: not available
```
We can see that this is our setter.

So the next line of interest is:
```c++
  return SetPropertyWithDefinedSetter(receiver, Handle<JSReceiver>::cast(setter), value, should_throw);
```

`SetPropertyWithDefinedSetter`
```c++
RETURN_ON_EXCEPTION_VALUE(isolate, Execution::Call(isolate, setter, receiver,
                                                   arraysize(argv), argv),
                          Nothing<bool>());
return Just(true);
```

`RETURN_ON_EXCEPTION_VALUE` checks if `Call` returned null and if so returns `Nothing<bool>()`. 

So lets follow `Call` (src/execution.cc line 191):
```c++
return CallInternal(isolate, callable, receiver, argc, argv, MessageHandling::kReport);
```
`MessageHandling` is an enum define as:
```c++
enum class MessageHandling { kReport, kKeepPending };
```

`CallInternal`: 
```c++
return Invoke(isolate, false, callable, receiver, argc, argv,
              isolate->factory()->undefined_value(), message_handling);
```
The second argument name is `is_construct` and undefined is passed for `new_target`.
`Invoke` (src/execution.cc line 58):
```c++
MUST_USE_RESULT MaybeHandle<Object> Invoke(
    Isolate* isolate, bool is_construct, Handle<Object> target,
    Handle<Object> receiver, int argc, Handle<Object> args[],
    Handle<Object> new_target, Execution::MessageHandling message_handling) {
    ...
    typedef Object* (*JSEntryFunction)(Object* new_target, Object* target,
                                       Object* receiver, int argc,
                                       Object*** args);
    ...
    Object* value = NULL;

    JSEntryFunction stub_entry = FUNCTION_CAST<JSEntryFunction>(code->entry());

    Handle<Code> code = is_construct
       ? isolate->factory()->js_construct_entry_code()
       : isolate->factory()->js_entry_code();
```
```console
    (lldb) p is_construct
    (bool) $74 = false
```

So `isolate->factory()->js_entry_code()` will be called. Lets take a look at this code. To do this we have to
take the address from code->entry()


```console
(lldb) job *code
0x6b17cf04001: [Code]
kind = STUB
major_key = JSEntryStub
compiler = unknown
Instructions (size = 234)
0x6b17cf04060     0  55             push rbp
0x6b17cf04061     1  4889e5         REX.W movq rbp,rsp
0x6b17cf04064     4  6a02           push 0x2
0x6b17cf04066     6  49ba7819000601000000 REX.W movq r10,0x106001978    ;; external reference (Isolate::context_address)
0x6b17cf04070    10  4d8b12         REX.W movq r10,[r10]
0x6b17cf04073    13  4152           push r10
0x6b17cf04075    15  4154           push r12
0x6b17cf04077    17  4155           push r13
0x6b17cf04079    19  4156           push r14
0x6b17cf0407b    1b  4157           push r15
0x6b17cf0407d    1d  53             push rbx
0x6b17cf0407e    1e  49bd4800000601000000 REX.W movq r13,0x106000048    ;; external reference (Heap::roots_array_start())
0x6b17cf04088    28  4981c580000000 REX.W addq r13,0x80
0x6b17cf0408f    2f  49bae819000601000000 REX.W movq r10,0x1060019e8    ;; external reference (Isolate::c_entry_fp_address)
0x6b17cf04099    39  41ff32         push [r10]
0x6b17cf0409c    3c  48a1081a000601000000 REX.W movq rax,(0x106001a08)    ;; external reference (Isolate::js_entry_sp_address)
0x6b17cf040a6    46  4885c0         REX.W testq rax,rax
0x6b17cf040a9    49  0f8514000000   jnz 0x6b17cf040c3  <+0x63>
0x6b17cf040af    4f  6a02           push 0x2
0x6b17cf040b1    51  488bc5         REX.W movq rax,rbp
0x6b17cf040b4    54  48a3081a000601000000 REX.W movq (0x106001a08),rax    ;; external reference (Isolate::js_entry_sp_address)
0x6b17cf040be    5e  e902000000     jmp 0x6b17cf040c5  <+0x65>
0x6b17cf040c3    63  6a00           push 0x0
0x6b17cf040c5    65  e916000000     jmp 0x6b17cf040e0  <+0x80>
0x6b17cf040ca    6a  48a38819000601000000 REX.W movq (0x106001988),rax    ;; external reference (Isolate::pending_exception_address)
0x6b17cf040d4    74  498b8588000000 REX.W movq rax,[r13+0x88]
0x6b17cf040db    7b  e932000000     jmp 0x6b17cf04112  <+0xb2>
0x6b17cf040e0    80  49baf019000601000000 REX.W movq r10,0x1060019f0    ;; external reference (Isolate::handler_address)
0x6b17cf040ea    8a  41ff32         push [r10]
0x6b17cf040ed    8d  49baf019000601000000 REX.W movq r10,0x1060019f0    ;; external reference (Isolate::handler_address)
0x6b17cf040f7    97  498922         REX.W movq [r10],rsp
0x6b17cf040fa    9a  6a00           push 0x0
0x6b17cf040fc    9c  e8bf000000     call 0x6b17cf041c0  (JSEntryTrampoline)    ;; code: BUILTIN
0x6b17cf04101    a1  49baf019000601000000 REX.W movq r10,0x1060019f0    ;; external reference (Isolate::handler_address)
0x6b17cf0410b    ab  418f02         pop [r10]
0x6b17cf0410e    ae  4883c400       REX.W addq rsp,0x0
0x6b17cf04112    b2  5b             pop rbx
0x6b17cf04113    b3  4883fb02       REX.W cmpq rbx,0x2
0x6b17cf04117    b7  0f8511000000   jnz 0x6b17cf0412e  <+0xce>
0x6b17cf0411d    bd  49ba081a000601000000 REX.W movq r10,0x106001a08    ;; external reference (Isolate::js_entry_sp_address)
0x6b17cf04127    c7  49c70200000000 REX.W movq [r10],0x0
0x6b17cf0412e    ce  49bae819000601000000 REX.W movq r10,0x1060019e8    ;; external reference (Isolate::c_entry_fp_address)
0x6b17cf04138    d8  418f02         pop [r10]
0x6b17cf0413b    db  5b             pop rbx
0x6b17cf0413c    dc  415f           pop r15
0x6b17cf0413e    de  415e           pop r14
0x6b17cf04140    e0  415d           pop r13
0x6b17cf04142    e2  415c           pop r12
0x6b17cf04144    e4  4883c410       REX.W addq rsp,0x10
0x6b17cf04148    e8  5d             pop rbp
0x6b17cf04149    e9  c3             retl


Handler Table (size = 24)

RelocInfo (size = 23)
0x6b17cf04068  external reference (Isolate::context_address)  (0x106001978)
0x6b17cf04080  external reference (Heap::roots_array_start())  (0x106000048)
0x6b17cf04091  external reference (Isolate::c_entry_fp_address)  (0x1060019e8)
0x6b17cf0409e  external reference (Isolate::js_entry_sp_address)  (0x106001a08)
0x6b17cf040b6  external reference (Isolate::js_entry_sp_address)  (0x106001a08)
0x6b17cf040cc  external reference (Isolate::pending_exception_address)  (0x106001988)
0x6b17cf040e2  external reference (Isolate::handler_address)  (0x1060019f0)
0x6b17cf040ef  external reference (Isolate::handler_address)  (0x1060019f0)
0x6b17cf040fd  code target (BUILTIN)  (0x6b17cf041c0)
0x6b17cf04103  external reference (Isolate::handler_address)  (0x1060019f0)
0x6b17cf0411f  external reference (Isolate::js_entry_sp_address)  (0x106001a08)
0x6b17cf04130  external reference (Isolate::c_entry_fp_address)  (0x1060019e8)
```

```c++
    Object* orig_func = *new_target;
    Object* func = *target;
    Object* recv = *receiver;
    Object*** argv = reinterpret_cast<Object***>(args);
    if (FLAG_profile_deserialization && target->IsJSFunction()) {
      PrintDeserializedCodeInfo(Handle<JSFunction>::cast(target));
    }
    RuntimeCallTimerScope timer(isolate, &RuntimeCallStats::JS_Execution);
    value = CALL_GENERATED_CODE(isolate, stub_entry, orig_func, func, recv, argc, argv);
```
`CALL_GENEREATED_CODE` is defined in `src/x64/simulator-x64.h`:
```c++
#define CALL_GENERATED_CODE(isolate, entry, p0, p1, p2, p3, p4) \
  (entry(p0, p1, p2, p3, p4))
```


Now, `stub_entry is of type `JSEntryFunction` which is a typedef 
```c++
    typedef Object* (*JSEntryFunction)(Object* new_target, Object* target,
                                      Object* receiver, int argc,
                                      Object*** args);
    stub_entry(orig_func, func, recv, argc, argv)

    JSEntryFunction stub_entry = FUNCTION_CAST<JSEntryFunction>(code->entry());
```

From the comments about FUNCTION_CAST it is used to invoke generated code from within C:
```c++
// FUNCTION_CAST<F>(addr) casts an address into a function
// of type F. Used to invoke generated code from within C.
template <typename F>
F FUNCTION_CAST(Address addr) {
  return reinterpret_cast<F>(reinterpret_cast<intptr_t>(addr));
}
```

Lets take a look at the arguments to entry which are:

```c++
    stub_entry(orig_func, func, recv, argc, argv)
```


To follow the code in `stub_entry` we need to step-inst (instruction level step) but there is not debug information
as this is generated code. What we can do is after every step call dissasemble --pc:
```console
(lldb) target stop-hook add
Enter your stop hook command(s).  Type 'DONE' to end.
> disassemble --pc
> DONE
Stop hook #1 added.
(lldb) undisplay
```

Now we should be able to step into using:
```console
(lldb) si
Process 72301 stopped
* thread #1: tid = 0xc2e41e, 0x000006b17cf04060, queue = 'com.apple.main-thread', stop reason = instruction step into
    frame #0: 0x000006b17cf04060
->  0x6b17cf04060: pushq  %rbp
    0x6b17cf04061: movq   %rsp, %rbp
    0x6b17cf04064: pushq  $0x2
    0x6b17cf04066: movabsq $0x106001978, %r10        ; imm = 0x106001978
```
Notice that the address matches that of the output from `(lldb) job *code` above:
```console
0x6b17cf04060     0  55             push rbp
0x6b17cf04061     1  4889e5         REX.W movq rbp,rsp
0x6b17cf04064     4  6a02           push 0x2
0x6b17cf04066     6  49ba7819000601000000 REX.W movq r10,0x106001978    ;; external reference (Isolate::context_address)
```
V8 uses Intel's assembly syntax and lldb uses AT&T syntax so the src/dest arguments are switched around, and registers 
prefixes are not used in Intel syntax.

Start of `(lldb) job *code`:
```console
0x6b17cf04060     0  55             push rbp
0x6b17cf04061     1  4889e5         REX.W movq rbp,rsp
0x6b17cf04064     4  6a02           push 0x2
0x6b17cf04066     6  49ba7819000601000000 REX.W movq r10,0x106001978    ;; external reference (Isolate::context_address)
0x6b17cf04070    10  4d8b12         REX.W movq r10,[r10]
```
I'm trying to line up the assembly code output with src/x64/code-stubs-x64.cc and `JSEntryStub::Generate` and see if I'm 
looking at the correct code:

`push rbp`       __ pushq(rbp);  
`movq rpb, rsp`  __ movp(rbp, rsp);  
`push 0x2`       -- Push(Immediate(StackFrame::TypeToMarker(type()))) 
This pushed the type of the stack frame which is an emmediate value 2. I think this value is taken from src/frames.h 
and StackFrame class which has an enum:
```c++
enum Type {
  NONE = 0,
  STACK_FRAME_TYPE_LIST(DECLARE_TYPE)
  NUMBER_OF_TYPES,
  // Used by FrameScope to indicate that the stack frame is constructed
  // manually and the FrameScope does not need to emit code.
  MANUAL
};
```
And the first in the STACK_FRAME_LIST is:
```c++
#define STACK_FRAME_TYPE_LIST(V)                                          \
  V(ENTRY, EntryFrame)                                                    \
  ...
```

`movq r10,0x106001978` matches:
```c++
ExternalReference context_address(IsolateAddressId::kContextAddress, isolate());
__ Load(kScratchRegister, context_address);
```
`kScratchRegister` is r10 and we are moving `0x106001978` (the context_address) into register r10.  
```
`movq r10,[r10]`   __ Load(kScratchRegister, context_address);  // get the pointer
`push r10`         __ Push(kScratchRegister);  // push the pointer onto the stack
`push r12`         __ pushq(r12);  
`push r13`         __ pushq(r13);  
`push r14`         __ pushq(r14);  
`push r15`         __ pushq(r15); 
`push rbx`         __ pushq(rbx);  

`movq r13,0x106000048`   __ InitializeRootRegister();  //  external reference (Heap::roots_array_start())
`addq r13,0x80`          __ Push(c_entry_fp_operand); 

`movq r10,0x1060019e8`   __ Load(rax, js_entry_sp);   ;; external reference (Isolate::c_entry_fp_address)
```
```console
0x6b17cf04099    39  41ff32         push [r10]
0x6b17cf0409c    3c  48a1081a000601000000 REX.W movq rax,(0x106001a08)    ;; external reference (Isolate::js_entry_sp_address)
0x6b17cf040a6    46  4885c0         REX.W testq rax,rax
0x6b17cf040a9    49  0f8514000000   jnz 0x6b17cf040c3  <+0x63>
```

You might be wondering what that `__` is, well it is a macro:
```c++
#define __ ACCESS_MASM(masm)

#define ACCESS_MASM(masm) masm->
```
So `__` will get expanded to masm->pushq(rbp) for example. I think this is done so that it look more like assembly and 
not so much as C++/C.
```console
0x6b17cf04060     0  55             push rbp                                 // prologue
0x6b17cf04061     1  4889e5         REX.W movq rbp,rsp                       // prologue
0x6b17cf04064     4  6a02           push 0x2                                 // Stack frame type = Context Address type
0x6b17cf04066     6  49ba7819000601000000 REX.W movq r10,0x106001978         // external reference (Isolate::context_address)
0x6b17cf04070    10  4d8b12         REX.W movq r10,[r10]                     // dereference the context_address pointer
0x6b17cf04073    13  4152           push r10                                 // push contents of r10 (ScratchRegister)
0x6b17cf04075    15  4154           push r12                                 // push contents of r12 argument passed
0x6b17cf04077    17  4155           push r13             
0x6b17cf04079    19  4156           push r14
0x6b17cf0407b    1b  4157           push r15
0x6b17cf0407d    1d  53             push rbx                                 // push
0x6b17cf0407e    1e  49bd4800000601000000 REX.W movq r13,0x106000048         // external reference (Heap::roots_array_start())
0x6b17cf04088    28  4981c580000000 REX.W addq r13,0x80                      // push e_entry_fp_operand
0x6b17cf0408f    2f  49bae819000601000000 REX.W movq r10,0x1060019e8         // external reference (Isolate::c_entry_fp_address)
0x6b17cf04099    39  41ff32         push [r10]                               // push the scratch targed used above
0x6b17cf0409c    3c  48a1081a000601000000 REX.W movq rax,(0x106001a08)       // external reference (Isolate::js_entry_sp_address)
0x6b17cf040a6    46  4885c0         REX.W testq rax,rax                      // check if rax is zero
0x6b17cf040a9    49  0f8514000000   jnz 0x6b17cf040c3  <+0x63>               // jump if not zero (ZF = 0)
0x6b17cf040af    4f  6a02           push 0x2                                 //StackFrame::OUTERMOST_JSENTRY_FRAME
0x6b17cf040b1    51  488bc5         REX.W movq rax,rbp                       // __ movp(rax, rbp)
0x6b17cf040b4    54  48a3081a000601000000 REX.W movq (0x106001a08),rax       // __ Store(js_entry_sp, rax)
0x6b17cf040be    5e  e902000000     jmp 0x6b17cf040c5  <+0x65> ---------+
0x6b17cf040c3    63  6a00           push 0x0                            |    // __ Push(Immediate(StackFrame::INNER_JSENTRY_FRAME));
    +-------------------------------------------------------------------+
    |
0x6b17cf040c5    65  e916000000     jmp 0x6b17cf040e0  <+0x80> ---------+    // __ jmp(&invoke);
0x6b17cf040ca    6a  48a38819000601000000 REX.W movq (0x106001988),rax  |    // __ Store(pending_exception, rax)
0x6b17cf040d4    74  498b8588000000 REX.W movq rax,[r13+0x88]           |    // __ LoadRoot(rax, Heap::kExceptionRootIndex);
0x6b17cf040db    7b  e932000000     jmp 0x6b17cf04112  <+0xb2>          |    // __ jump(&exit)
    +-------------------------------------------------------------------+
    |
0x6b17cf040e0    80  49baf019000601000000 REX.W movq r10,0x1060019f0         // external reference (Isolate::handler_address)
0x6b17cf040ea    8a  41ff32         push [r10]                               // push value of the pointer onto the stack
0x6b17cf040ed    8d  49baf019000601000000 REX.W movq r10,0x1060019f0         // PushStackHandler (Isolate::handler_address)
0x6b17cf040f7    97  498922         REX.W movq [r10],rsp                     // 
0x6b17cf040fa    9a  6a00           push 0x0                                 // RelocInfo::CODE_TARGET  ?
0x6b17cf040fc    9c  e8bf000000     call 0x6b17cf041c0  (JSEntryTrampoline)  //  code: BUILTIN
0x6b17cf04101    a1  49baf019000601000000 REX.W movq r10,0x1060019f0         // PopStackHandnler (Isolate::handler_address)
0x6b17cf0410b    ab  418f02         pop [r10]
0x6b17cf0410e    ae  4883c400       REX.W addq rsp,0x0
0x6b17cf04112    b2  5b             pop rbx
0x6b17cf04113    b3  4883fb02       REX.W cmpq rbx,0x2
0x6b17cf04117    b7  0f8511000000   jnz 0x6b17cf0412e  <+0xce>
0x6b17cf0411d    bd  49ba081a000601000000 REX.W movq r10,0x106001a08    ;; external reference (Isolate::js_entry_sp_address)
0x6b17cf04127    c7  49c70200000000 REX.W movq [r10],0x0
0x6b17cf0412e    ce  49bae819000601000000 REX.W movq r10,0x1060019e8    ;; external reference (Isolate::c_entry_fp_address)
0x6b17cf04138    d8  418f02         pop [r10]
0x6b17cf0413b    db  5b             pop rbx
0x6b17cf0413c    dc  415f           pop r15
0x6b17cf0413e    de  415e           pop r14
0x6b17cf04140    e0  415d           pop r13
0x6b17cf04142    e2  415c           pop r12
0x6b17cf04144    e4  4883c410       REX.W addq rsp,0x10
0x6b17cf04148    e8  5d             pop rbp
0x6b17cf04149    e9  c3             retl

0x6b17cf040fd  code target (BUILTIN)  (0x6b17cf041c0)
```

If we set a break point after CALL_GENERATED_CODE we will see that this code does return and a value is
provided:
```c++
bool has_exception = value->IsException(isolate);
In this case there was no exceptions so:
isolate->clear_pending_message();

return Handle<Object>(value, isolate);
```  

```c++
Handle<Object> result = custom.Call(callback);

RETURN_EXCEPTION_IF_SCHEDULED_EXCEPTION(isolate, Object);
```
which will expend to :
```c++
Isolate* __isolate__ = (isolate);                       
DCHECK(!__isolate__->has_pending_exception());        
if (__isolate__->has_scheduled_exception()) {        
  __isolate__->PromoteScheduledException();         
  return MaybeHandle<Object>();                                    
}
```
In this case there will be a scheduled_exception.

```c++
Object* Isolate::PromoteScheduledException() {
  Object* thrown = scheduled_exception();
  clear_scheduled_exception();
  // Re-throw the exception to avoid getting repeated error reporting.
  return ReThrow(thrown);
}
```
```console
(lldb) job thrown
0x136bbb969d61: [JS_ERROR_TYPE]
 - map = 0x136bd865de81 [FastProperties]
 - prototype = 0x136bddd8d679
 - elements = 0x136ba4c82241 <FixedArray[0]> [HOLEY_SMI_ELEMENTS]
 - properties = 0x136bbb969d79 <PropertyArray[3]> {
    #stack: 0x136baa1ca9b9 <AccessorInfo> (const accessor descriptor)
    #message: 0x136b27c6e981 <String[18]: dummy setter error> (data field 0) properties[0]
    0x136ba4c87069 <Symbol: stack_trace_symbol>: 0x136bbb969f49 <JSArray[26]> (data field 1) properties[1]
 }
```

`ReThrow` will:
```c++
set_pending_exception(exception);
return heap()->exception();
``

Notes:
```c++
  RETURN_ON_FAILED_EXECUTION_PRIMITIVE(bool);
```
This macro is defined as:
```c++
#define RETURN_ON_FAILED_EXECUTION_PRIMITIVE(T) \
  EXCEPTION_BAILOUT_CHECK_SCOPED_DO_NOT_USE(isolate, Nothing<T>())
```
Which will expend to
```c++
  EXCEPTION_BAILOUT_CHECK_SCOPED_DO_NOT_USE(isolate, Nothing<bool>())

  #define EXCEPTION_BAILOUT_CHECK_SCOPED_DO_NOT_USE(isolate, Nothing<bool>) \
  do {                                                            \
    if (has_pending_exception) {                                  \
      call_depth_scope.Escape();                                  \
      return Nothing<bool>;                                       \
    }                                                             \
  } while (false)
```
So the last lines in v8::Object::Set function will be:
```c++
    if (has_pending_exception) {
      call_depth_scope.Escape();
      return Nothing<bool>;
    }                                                    
    return Just(true);
```



### Setters
I'm guessing getters/setters are added using V8 Accessor (TODO: look at the v8 examples I have)

### factory.h
When debugging you might come across a call that invokes a function in V8's factory, for example:
```c++
isolate->factory()->undefined_value();
```
Now, in the debugger you see something like:
```console
ROOT_LIST(ROOT_ACCESSOR)
```
Lets take a closer look at the `ROOT_LIST` macro. It is defined in factory.h:
```c++
#define ROOT_ACCESSOR(type, name, camel_name)                         \
  inline Handle<type> name() {                                        \
    return Handle<type>(bit_cast<type**>(                             \
        &isolate()->heap()->roots_[Heap::k##camel_name##RootIndex])); \
  }
  ROOT_LIST(ROOT_ACCESSOR)
#undef ROOT_ACCESSOR
```

src/heap/heap.h:
```c++
#define STRONG_ROOT_LIST(V)
...
V(Oddball, undefined_value, UndefinedValue)
```

Which would expand to:
```c++
  inline Handle<Oddball> undefined_value() {  
    return Handle<Oddball>(bit_cast<Oddball**>(
        &isolate()->heap()->roots_[Heap::kUndefinedValueRootIndex]));
```
Recall that Oddball describes objects null, undefined, true, and false.

`bit_cast` can be found in src/base/macros.h.

### bootstrap_node.js compilation and execution walkthrough
The goal of this section is to understand what happens when bootstrap_node.js is compiled and run mainly focusing on
the V8 side of things.

```console
    $ lldb -- out/Debug/node --print-ast
    (lldb) br s -f node::LoadEnvironment
    (lldb) r
```

Lets step through to the following line in `node::ExecuteString`:
```c++
Local<Value> f_value = ExecuteString(env, MainSource(env), script_name);
```
MainSource is a function in `node_javascript.h` which is used by `node_js2c`. This was documented earlier so I won't go
into details about it now. You can see the content using:
```console
(lldb) jlh MainSource(env)
Which is the same thing as calling:
(lldb) expr (*(v8::internal::Object**)*MainSource(env))->Print()
```
`jlh` can be found in the V8 source tree in `tools/lldbinit`.

`ExecuteString` is a function in node.cc which calls `Compile`:
```c++
MaybeLocal<v8::Script> script = v8::Script::Compile(env->context(), source, &origin);
```

Will delegate to `ScriptCompiler::Compile`, and then to `ScriptCompiler::CompileUnboundInternal` which can be found in
`deps/v8/src/api.cc`:
```c++
MaybeLocal<UnboundScript> ScriptCompiler::CompileUnboundInternal(
   Isolate* v8_isolate, Source* source, CompileOptions options) {
     ...
     i::MaybeHandle<i::SharedFunctionInfo> maybe_function_info = i::Compiler::GetSharedFunctionInfoForScript(
        str, name_obj, line_offset, column_offset, source->resource_options,
        source_map_url, isolate->native_context(), NULL, &script_data,
        options, i::NOT_NATIVES_CODE, host_defined_options);
     
```

`Compiler::GetSharedFunctionForScript` in `deps/v8/src/compiler.cc`:
```c++
  ParseInfo parse_info(script);
  Zone compile_zone(isolate->allocator(), ZONE_NAME);
  ...
  maybe_result = CompileToplevel(&parse_info, isolate);
```
Lets take a look at parse_info:
```console
(lldb) job *parse_info.script()
0x169ddd9abfb1: [Script] in OldSpace
 - source: 0x169df8f0d3d1 <Very long string[21923]>
 - name: 0x169df8f0d3a1 <String[17]: bootstrap_node.js>
 - line_offset: 0
 - column_offset: 0
 - type: 2
 - id: 14
 - context data: 0x169dd88022e1 <undefined>
 - wrapper: 0x169dd88022e1 <undefined>
 - compilation type: 0
 - line ends: 0x169dd88022e1 <undefined>
 - eval from shared: 0x169dd88022e1 <undefined>
 - eval from position: 0
 - shared function infos: 0x169dd8802251 <FixedArray[0]>
```
This will be passed to `CompileToplevel`:
```c++
  if (parse_info->literal() == nullptr && 
      !parsing::ParseProgram(parse_info, isolate)) {
  ...
  std::forward_list<std::unique_ptr<CompilationJob>> inner_function_jobs;
  std::unique_ptr<CompilationJob> outer_function_job(
      GenerateUnoptimizedCode(parse_info, isolate, &inner_function_jobs));
  ...
```
In this case 
```console
(lldb) expr parse_info->literal() == nullptr
(bool) $80 = true
```

First, `parsing::ParseProgram` will parse the JavaScript and produce the abstract syntax tree.
`ParseProgram`:
```c++
result = parser.ParseProgram(isolate, info);
info->set_literal(result);
```
We can inspect parse_info after this function returns:
```console
(lldb) expr parse_info->literal()->Print()
FUNC LITERAL at 0
. NAME <nil>
. INFERRED NAME 1
```
Now, this does not look like much but this is only the function literal:
```console
(function(process) {
});
```

We can print the AST generated using:
```console
(lldb) expr AstPrinter(isolate()).PrintProgram(parse_info()->literal())
(const char *) $56 = 0x0000000105016e20 "FUNC at 0\n. KIND 0\n. SUSPEND COUNT 0\n. NAME ""\n. INFERRED NAME ""\n. EXPRESSION STATEMENT at 284\n. . LITERAL "use strict"\n. EXPRESSION STATEMENT at 299\n. . ASSIGN at -1\n. . . VAR PROXY local[0] (0x10682f138) (mode = TEMPORARY) ".result"\n. . . FUNC LITERAL at 300\n. . . . NAME \n. . . . INFERRED NAME \n. . . . PARAMS\n. . . . . VAR (0x10681cd48) (mode = VAR) "process"\n. RETURN at -1\n. . VAR PROXY local[0] (0x10682f138) (mode = TEMPORARY) ".result"\n"
```
I've not figured out a good way to make this print nicely in lldb so using this is the more readable output:
```console
[generating bytecode for function: ]
--- AST ---
FUNC at 0
. KIND 0
. SUSPEND COUNT 0
. NAME ""
. INFERRED NAME ""
. EXPRESSION STATEMENT at 284
. . LITERAL "use strict"
. EXPRESSION STATEMENT at 299
. . ASSIGN at -1
. . . VAR PROXY local[0] (0x10683f938) (mode = TEMPORARY) ".result"
. . . FUNC LITERAL at 300
. . . . NAME
. . . . INFERRED NAME
. . . . PARAMS
. . . . . VAR (0x10682d548) (mode = VAR) "process"
. RETURN at -1
. . VAR PROXY local[0] (0x10683f938) (mode = TEMPORARY) ".result"
```
Notice that we have `use strict` starting a character `284`. This due to the comment that preceeds it.
We can see that the function literal starts at `300` and that if we would have given it a name it would
have shown up as `NAME somename`. We can also see that it takes a parameter named `process`

Next `GenerateUnoptimizedCode` will be called (`deps/v8/src/compiler.cc`):
```c++
  Compiler::EagerInnerFunctionLiterals inner_literals;
  if (!Compiler::Analyze(parse_info, &inner_literals)) {
    return std::unique_ptr<CompilationJob>();
  }
  std::unique_ptr<CompilationJob> outer_function_job(
      PrepareAndExecuteUnoptimizedCompileJob(parse_info, parse_info->literal(), isolate));
```
`PrepareAndExecuteUnoptimizedCompileJob`:
```c++
  if (job->PrepareJob() == CompilationJob::SUCCEEDED &&
      job->ExecuteJob() == CompilationJob::SUCCEEDED) {
```
`PrepareJob()` will print the ast as shown above. Lets take a closer look at `ExecuteJob`:
```c++
return UpdateState(ExecuteJobImpl(), State::kReadyToFinalize);
```
`InterpreterCompilationJob::ExecuteJobImpl`:
```c++
generator()->GenerateBytecode(stack_limit());
```
This will land in `BytecodeGenerator::GenerateBytecode` `bytecode-generator.cc:906`
```c++
GenerateBytecodeBody();
```
This function will visit all of the nodes in the AST and generate the bytecodes for them.

Later in `FinalizeUnoptimizedCode`:
```c++
outer_function_job->compilation_info()->set_shared_info(shared_info);
```
```console
(lldb) expr shared_info->code()->Print()
0x1a00905c50e1: [Code]
kind = BUILTIN
name = CompileLazy
compiler = unknown
Instructions (size = 983)
0x1a00905c5140     0  488b5f2f       REX.W movq rbx,[rdi+0x2f]
0x1a00905c5144     4  488b5b07       REX.W movq rbx,[rbx+0x7]
0x1a00905c5148     8  493b5da0       REX.W cmpq rbx,[r13-0x60]
0x1a00905c514c     c  0f844c030000   jz 0x1a00905c549e  (CompileLazy)
...
```

Later in a call to `InterpreterCompilationJob::FinalizeJobImpl` will delegate to `BytecodeGenerator::FinalizeBytecode`
where the the BytecodeArray is generated:
```c++
Handle<BytecodeArray> bytecode_array = builder()->ToBytecodeArray(isolate);
```
The bytecodes can be inspected using:
```console
```console
(lldb) expr bytecodes->Print()
0x39d9bd7ade49: [BytecodeArray] in OldSpaceParameter count 1
Frame size 8
    0 E> 0x39d9bd7ade82 @    0 : 93                StackCheck
   15 S> 0x39d9bd7ade83 @    1 : 6f 00 00 00       CreateClosure [0], [0], #0
         0x39d9bd7ade87 @    5 : 1e fb             Star r0
21644 S> 0x39d9bd7ade89 @    7 : 97                Return
Constant pool (size = 1)
0x39d9bd7ade31: [FixedArray] in OldSpace
 - map = 0x39d9d04022f1 <Map(HOLEY_ELEMENTS)>
 - length: 1
           0: 0x39d9bd7add81 <SharedFunctionInfo bajja>
Handler Table (size = 16)
```
This bytecode array will be set on the compilation_info instance:
```c++
compilation_info()->SetBytecodeArray(bytecodes);
```
And the code will be set to `InterpreterEntryTrampoline`:
```c++
compilation_info()->SetCode(
    BUILTIN_CODE(compilation_info()->isolate(), InterpreterEntryTrampoline));
return SUCCEEDED;
```
This will return us into `FinalizeUnoptimizedCompilationJob`:
```c++
if (status == CompilationJob::SUCCEEDED) {
  InstallUnoptimizedCode(compilation_info);
  CodeEventListener::LogEventsAndTags log_tag;
```
`InstallUnoptimizedCode` will set up the `FeedbackMetadata`:
```c++
   Handle<FeedbackMetadata> feedback_metadata = FeedbackMetadata::New(
        compilation_info->isolate(),
        compilation_info->literal()->feedback_vector_spec());
    compilation_info->shared_info()->set_feedback_metadata(*feedback_metadata);
```
We can inspect `feedback_metadata` using:
```console
(lldb) expr feedback_metadata->Print()
0x39d9bd7adea9: [FeedbackMetadata] in OldSpace
 - length: 2
 - slot_count: 1
 Slot #0 kCreateClosure
```
Back in `InstallUnoptimizedCode`:
```c++
shared->set_code(*compilation_info->code())
shared->set_bytecode_array(*compilation_info->bytecode_array());
```
This is setting the SharedFunctionInfo instance code to `InterpreterEntryTrampoline` and the bytecode array
is also set that we saw before.
After this control will be returned to `FinalizeUnoptimizedCompilationJob` and we go through all the inner_function_jobs
and set the SharedFunctionInfo for them.
```c++
for (auto&& inner_job : *inner_function_jobs) {
    Handle<SharedFunctionInfo> inner_shared_info =
        Compiler::GetSharedFunctionInfo(
            inner_job->compilation_info()->literal(), parse_info->script(),
            isolate);
    if (inner_shared_info->is_compiled()) continue;
    inner_job->compilation_info()->set_shared_info(inner_shared_info);
    if (FinalizeUnoptimizedCompilationJob(inner_job.get()) !=
        CompilationJob::SUCCEEDED) {
      return false;
    }
  }
```
We can inspect `inner_shared_info` using:
```console
(lldb) expr inner_shared_info->Print()
...
(lldb) expr inner_shared_info->code()->Print()
0x1a00905c50e1: [Code]
kind = BUILTIN
name = CompileLazy
...
```

After this the shared_info will be returned from `CompileToplevel`. Which will the return to `Compiler::GetSharedFunctionInfoForScript`:
```c++
maybe_result = CompileToplevel(&parse_info, isolate);
...
Handle<FeedbackVector> feedback_vector = FeedbackVector::New(isolate, result);
vector = isolate->factory()->NewCell(feedback_vector);
compilation_cache->PutScript(source, context, language_mode, result, vector);
```

```console
(lldb) expr feedback_vector->Print()
0x265e1d0b03b1: [FeedbackVector] in OldSpace
 - length: 1
 SharedFunctionInfo: 0x265e1d0adae9 <SharedFunctionInfo>
 Optimized Code: 0
 Invocation Count: 0
 Profiler Ticks: 0
 Slot #0 kCreateClosure
  [0]: 0x265e1d0b03e1 <Cell value= 0x265eb5c022e1 <undefined>>
```

So we are backing out of the calls now and the next return will land us in `CompileUnboundInternal`:
```c++
has_pending_exception = !maybe_function_info.ToHandle(&result);
```
This will later return to `ScriptCompiler::Compile`:
```c++
auto maybe = CompileUnboundInternal(isolate, source, options);
...
v8::Context::Scope scope(context);
return result->BindToCurrentContext();
```
```console
(lldb) expr maybe
(lldb) expr maybe.ToLocalChecked()->GetId()
(int) $415 = 14
(lldb) jlh maybe.ToLocalChecked()->GetScriptName()
"bootstrap_node.js"
```
An `UnboundScript` is a compiled JavaScript but it is not yet tied to a Context.

After this the compilation is finished and control will be returned to node.cc which will now run the script:
```c++
Local<Value> result = script.ToLocalChecked()->Run();
```
So we have compiled the script and now are are going to run it. Just to clarify something here, the script contains an expression
(the surrounding ()) that defines a function. So we are not executing the `startup` function here but instead only only the
expression that contains it.

`Run` will call `Execution::Call`, which will call `CallInternal`, which will call `Invoke`:
```c++
    typedef Object* (*JSEntryFunction)(Object* new_target, Object* target,
                                     Object* receiver, int argc,
                                     Object*** args);
    Handle<Code> code = is_construct
       ? isolate->factory()->js_construct_entry_code()
       : isolate->factory()->js_entry_code();
    ...
  
    // start of the function identified by code->entry() address.
    JSEntryFunction stub_entry = FUNCTION_CAST<JSEntryFunction>(code->entry());

    // Call the function through the right JS entry stub.
    Object* orig_func = *new_target;
    Object* func = *target;
    Object* recv = *receiver;
    Object*** argv = reinterpret_cast<Object***>(args);
    if (FLAG_profile_deserialization && target->IsJSFunction()) {
      PrintDeserializedCodeInfo(Handle<JSFunction>::cast(target));
    }
    RuntimeCallTimerScope timer(isolate, &RuntimeCallStats::JS_Execution);
    value = CALL_GENERATED_CODE(isolate, stub_entry, orig_func, func, recv, argc, argv);
  }
```

Notice that `code` will be what `isolate->factory()->js_entry_code()` returns:
```console
(lldb) expr isolate->factory()->js_entry_code()->Print()
0xe6d84b84001: [Code]
kind = STUB
major_key = JSEntryStub
compiler = unknown
Instructions (size = 232)
0xe6d84b84060     0  55             push rbp
0xe6d84b84061     1  4889e5         REX.W movq rbp,rsp
0xe6d84b84064     4  6a02           push 0x2
...
```
I'm not showing the complete output above as this will be shown later in detail.

Next, we have `CALL_GENERATED_CODE` which can be found in `src/x64/simulator-x64.h`:
```c++
// Since there is no simulator for the x64 architecture the only thing we can
// do is to call the entry directly.
// TODO(X64): Don't pass p0, since it isn't used?
#efine CALL_GENERATED_CODE(isolate, entry, p0, p1, p2, p3, p4) \
  (entry(p0, p1, p2, p3, p4))
```
So this will be an call that looks like this:
```c++
entry(orig_func, func, recv, argc, argv);
```
If we use `step instruction` (si) we can follow the setup and calling of this function:
```console
    0x100e381b4 <+1348>: movq   -0x118(%rbp), %rdx                  // move the value of address of local variable code->entry() into rdx
    0x100e381bb <+1355>: movq   -0x120(%rbp), %rdi                  // first argument which is local variable orig_func
->  0x100e381c2 <+1362>: movq   -0x128(%rbp), %rsi                  // second argument which is local variable func
    0x100e381c9 <+1369>: movq   -0x130(%rbp), %rcx                  // third argument which is local variable recv
    0x100e381d0 <+1376>: movl   -0x30(%rbp), %eax                   // fourth arg which is local variable argc 
    0x100e381d3 <+1379>: movq   -0x138(%rbp), %r8                   // fifth argument which is local variable argv
    0x100e381da <+1386>: movq   %rdx, -0x1c0(%rbp)                  // move code->entry from rdx into local variable 
    0x100e381e1 <+1393>: movq   %rcx, %rdx                          // move recv into rdx
    0x100e381e4 <+1396>: movl   %eax, %ecx                          // move argc into ecx
    0x100e381e6 <+1398>: movq   -0x1c0(%rbp), %r9                   // move code-entry into register r9
    0x100e381ed <+1405>: callq  *%r9                                // call address in register r9
```
Below we verify the contents of the above instructions:
```console
(lldb) memory read -f x -c 1 -s 8 `$rbp - 0x118`
0x7fff5fbfd828: 0x0000302f34084060
(lldb) expr code->entry()
(byte *) $186 = 0x0000302f34084060

(lldb) memory read -f x -c 1 -s 8 `$rbp - 0x120`
0x7fff5fbfd820: 0x00001d64e5d822e1
(lldb) expr orig_func
(v8::internal::Object *) $209 = 0x00001d64e5d822e1

(lldb) memory read -f x -c 1 -s 8 `$rbp - 0x128`
0x7fff5fbfd818: 0x00001d6417d30669
(lldb) expr func
(v8::internal::Object *) $211 = 0x00001d6417d30669

(lldb) memory read -f x -c 1 -s 8 `$rbp - 0x130`
0x7fff5fbfd810: 0x00001d64f0a82239
(lldb) expr recv
(v8::internal::Object *) $213 = 0x00001d64f0a82239

(lldb) memory read -f x -c 1 -s 4 `$rbp - 0x30`
0x7fff5fbfd910: 0x00000000
(lldb) expr argc
(int) $216 = 0

(lldb) memory read -f x -c 1 -s 8 `$rbp - 0x138`
0x7fff5fbfd808: 0x0000000000000000
(lldb) expr argv
(v8::internal::Object ***) $219 = 0x0000000000000000

(lldb) memory read -f x -c 1 -s 8 `$rbp - 0x1c0`
0x7fff5fbfd780: 0x0000302f34084060
```

The last instruction `callq *%r9` will call into JSEntryStub:
```console
->  0x302f34084060: pushq  %rbp                                  // push callers base frame pointer saving it so we can restore it
(lldb) memory read -f x -c 1 -s 8 `$rsp`                         // inspect the stack
    0x302f34084061: movq   %rsp, %rbp                            // mov the current value of rsp to into rbp which will be the frame pointer for this function
    0x302f34084064: pushq  $0x2                                  // this is pushing an immediate value 2 onto the stack. Where does this come from, the following:
(lldb) expr v8::internal::StackFrame::MarkerToType(2)
(v8::internal::StackFrame::Type) $494 = ENTRY
v8::internal::StackFrame::Type::ENTRY))
(int32_t) $262 = 2
(lldb) memory read -f x -c 2 -s 8 `$rsp`                         // inspect the stack 
    0x302f34084066: movabsq $0x106001990, %r10                   // move the context address into r10
(lldb) up 2
(lldb) expr isolate->isolate_addresses_[IsolateAddressId::kContextAddress]
(v8::internal::Address) $230 = 0x0000000106001990 
    0x302f34084070: movq   (%r10), %r10                          // the context address is a pointer, this will dereference it
(lldb) memory read -f x -c 1 -s 8 `isolate->isolate_addresses_[IsolateAddressId::kContextAddress]`
0x106001990: 0x00001d6417d03a59
(lldb) register read r10
     r10 = 0x00001d6417d03a59
     0x302f34084073: pushq  %r10                                // push the dereferences context onto the stack
     0x302f34084075: pushq  %r12                                // r12 must be preserved accross function calls so save it and pop it later before returning
     0x302f34084077: pushq  %r13                                // r13 must be preserved accross function calls so save it and pop it later before returning
     0x302f34084079: pushq  %r14                                // r14 must be preserved accross function calls so save it and pop it later before returning
     0x302f3408407b: pushq  %r15                                // r14 must be preserved accross function calls so save it and pop it later before returning
     0x302f3408407d: pushq  %rbx                                // rbx must be preserved accross function calls so save it and pop it later before returning
     0x302f3408407e: movabsq $0x106000048, %r13                 // move the value of the roots_array_start into r13
(lldb) expr isolate->heap()->roots_array_start()
(v8::internal::Object **) $233 = 0x0000000106000048
     0x302f34084088: addq   $0x80, %r13                         // addp(kRootRegister, Immediate(kRootRegisterBias)); kRootRegisterBias is 128
     0x302f3408408f: movabsq $0x106001a00, %r10                 // move he CEntryFPAddress into r10
(lldb) memory read -f x -c 1 -s 8 isolate->isolate_addresses_[IsolateAddressId::kCEntryFPAddress]
0x106001a00: 0x0000000000000000
    0x302f34084099: pushq  (%r10)                               // dereference CEntryAddress and push onto the stack
(lldb) memory read -f x -c 1 -s 8 0x0000000106001a00
0x106001a00: 0x0000000000000000
    0x302f3408409c: movabsq 0x106001a20, %rax                   // move JSEntrySPAddress into rax
(lldb) memory read -f x -c 1 -s 8 isolate->isolate_addresses_[IsolateAddressId::kJSEntrySPAddress]
0x106001a20: 0x0000000000000000 
    0x302f340840a6: testq  %rax, %rax                           // is rax zero? if so there is an outer js call
(lldb) register read rax
     rax = 0x0000000000000000
    0x302f340840af: pushq  $0x2                                 // v8::internal::StackFrame::OUTERMOST_JSENTRY_FRAME))
    0x302f340840b1: movq   %rbp, %rax                           // move this functions base pointer into rax 
    0x302f340840b4: movabsq %rax, 0x106001a20                   // store the base pointer in isolate_addresses_[IsolateAddressId::kJSEntrySPAddress]
(lldb) memory read -f x -c 1 -s 8 `isolate->isolate_addresses_[IsolateAddressId::kJSEntrySPAddress]`
0x106001a20: 0x0000000000000000
after
(lldb) memory read -f x -c 1 -s 8 `isolate->isolate_addresses_[IsolateAddressId::kJSEntrySPAddress]`
0x106001a20: 0x00007fff5fbfd760
(lldb) register read rbp
     rbp = 0x00007fff5fbfd940
    0x302f340840be: jmp    0x302f340840c5
    0x302f340840c5: jmp    0x302f340840e0
    0x302f340840e0: movabsq $0x106001a08, %r10                  // move isolate_addresses_[IsolateAddressId::kHandlerAddress]` into r10
(lldb) memory read -f x -c 1 -s 8 `isolate->isolate_addresses_[IsolateAddressId::kHandlerAddress]`
0x106001a08: 0x0000000000000000
    0x302f340840ea: pushq  (%r10)                               // push the dereferenced handler address
    0x302f340840ed: movabsq $0x106001a08, %r10                  // move isolate_addresses_[IsolateAddressId::kHandlerAddress]` into r10
    0x302f340840f7: movq   %rsp, (%r10)                         // move the value of the current stack pointer into the object pointed to be r10
(lldb) memory read -f x -c 1 -s 8 0x106001a08
0x106001a08: 0x00007fff5fbfd710
(lldb) register read rsp
     rsp = 0x00007fff5fbfd710
    0x302f340840fa: callq  0x302f341418c0                       // call JSEntryTrampoline builtin
(lldb) expr isolate->builtins()->builtin_handle(Builtins::Name::kJSEntryTrampoline)->entry()
(byte *) $255 = 0x0000302f341418c0 
      
```
In `deps/v8/src/builtins/x64/builtins-x64.cc` we can find `Generate_JSEntryTrampolineHelper` which is what generates the builtin. As this is done
a compile time we can put a break point in it (you can debug mksnapshot though which is done elsewhere in this document).
```console
    0x302f341418c0: movq   %rdi, %r11                           // move the orig_fun into r11
    0x302f341418c3: movq   %rsi, %rdi                           // move func into rdi
    0x302f341418c6: xorl   %esi, %esi                           // zero out esi
    0x302f341418c8: pushq  %rbp                                 // EnterFrame (deps/v8/src/x64/macro-assembler-x64.cc)
    0x302f341418c9: movq   %rsp, %rbp                           // 
    0x302f341418cc: pushq  $0x1c                                // push 0x1c (decimal 28)
(lldb) p v8::internal::StackFrame::TypeToMarker(static_cast<v8::internal::StackFrame::Type>(v8::internal::StackFrame::Type::INTERNAL))
(int32_t) $256 = 28
    0x302f341418ce: movabsq $0x302f34141861, %r10               // CodeObject() what is this, seems like it is Handle<HeapObject> which will be patched later
                                                                // I think this might be the register file?
(lldb) memory read -f x -c 1 -s 8 0x302f34141861
0x302f34141861: 0x6900001d64ceb827
    0x302f341418d8: pushq  %r10                                 // push the HandleHeapObject into the stack
    0x302f341418da: movabsq $0x1d64e5d822e1, %r10               // move undefined value into r10 
(lldb) expr *isolate->factory()->undefined_value()
(v8::internal::Oddball *) $264 = 0x00001d64e5d822e1
    0x302f341418e4: cmpq   %r10, (%rsp)
    0x302f341418e8: jne    0x302f341418fa                       // last of EnterFrame if we don't abort that is
    0x302f341418fa: movabsq $0x106001990, %r10                  // move the ContextAdress into r10 (the scratch register for x86)
(lldb) memory read -f x -c 1 -s 8 `isolate->isolate_addresses_[IsolateAddressId::kContextAddress]`
0x106001990: 0x00001d6417d03a59
    0x302f34141904: movq   (%r10), %rsi                         // deref and move into context into rsi
    0x302f34141907: pushq  %rdi                                 // push function onto the stack
    0x302f34141908: pushq  %rdx                                 // push recv onto the stack (this was moved above with 0x302f34141908: pushq  %rdx)
    0x302f34141909: movq   %rcx, %rax                           // move argc into rax (this was moved above with 0x100e381e4 <+1396>: movl   %eax, %ecx)
    0x302f3414190c: movq   %r8, %rbx                            // move argv into rbx
    0x302f3414190f: movq   %r11, %rdx                           // move orig_func into rdx
// Generate_CheckStackOverflow  TODO: got through this as I struggled to understand/map the generated instructions to the source code :( 
    0x302f34141912: movq   0xd08(%r13), %r10                    // 
    0x302f34141919: movq   %rsp, %rcx
    0x302f3414191c: subq   %r10, %rcx
    0x302f3414191f: movq   %rax, %r11
    0x302f34141922: shlq   $0x3, %r11
    0x302f34141926: cmpq   %r11, %rcx
    0x302f34141929: jg     0x302f34141940
// Generate_JSEntryTrampolineHelper
    0x302f34141940: xorl   %ecx, %ecx                           // zero out ecx, for the following loop
    0x302f34141942: jmp    0x302f3414194f                       // jump to entry label
    0x302f3414194f: cmpq   %rax, %rcx                           // compare argc with rcx (this was moved above). 
    0x302f34141952: jne    0x302f34141944                       // entry the loop if. This is pushing the argv values onto the stack in a loop, but we don't have any so we fall through
    0x302f34141954: callq  0x302f3413c400                       //
(lldb) expr isolate->builtins()->Call(static_cast<ConvertReceiverMode>(ConvertReceiverMode::kAny))->entry()
(byte *) $279 = 0x0000302f3413c400 "@\xfffffff6\xffffffc7\x01\x0f\xffffff84F"
// for the full object with assembly language instructions:
(lldb) job *isolate->builtins()->Call(static_cast<ConvertReceiverMode>(ConvertReceiverMode::kAny))
// Builtins::Generate_Call_ReceiverIsAny 'deps/v8/src/builtins/builtins-call-gen.cc':
// void Builtins::Generate_Call_ReceiverIsAny(MacroAssembler* masm) {
//   Generate_Call(masm, ConvertReceiverMode::kAny);
// }
// Builtins::Generate_Call `deps/v8/src/builtins/x64/builtins-x64.cc`
    0x302f3413c400: testb  $0x1, %dil                           // move the immediate 1 into the low 8 bits or rdi  // __ JumpIfSmi(rdi, &non_callable); which consists of CheckSmi
    0x302f3413c404: je     0x302f3413c450                       // if ZF = 1 then jump. This would happen if rdi was a SMI// __ JumpIfSmi(rdi, &non_callable: which after CheckSmi will jump. rdi is the target and not a smi in our case
// recall that rdi is the function:
(lldb) register read rdi
     rdi = 0x00001d6417d30669
(lldb) expr func
(v8::internal::Object *) $280 = 0x00001d6417d30669
```
Now I think that func is/was of type JSFunction (deps/v8/src/objects.h) as it was cast to Object* by:
```c++
auto fun = i::Handle<i::JSFunction>::cast(Utils::OpenHandle(this));
```

```c++
class JSFunction: public JSObject {
 public:
  // [prototype_or_initial_map]:
  DECL_ACCESSORS(prototype_or_initial_map, Object)
```
```console
    0x302f3413c40a: movq   -0x1(%rdi), %rcx                      // move the map into rcx. CmpObjectType(rdi, JS_FUNCTION_TYPE, rcx). HeapObject::kMapOffset which is the first field in JSFunction:
(lldb) memory read -f x -c 1 -s 8 `$rdi - 1`
0x1d6417d30668: 0x00001d6433c82521
(lldb) expr JSFunction::cast(func)->prototype_or_initial_map()
(v8::internal::Object *) $286 = 0x00001d64e5d82321
    0x302f3413c40e: cmpb   $-0x1, 0xb(%rcx)                      // CmpObjectType(rdi, JS_FUNCTION_TYPE, rcx) calls CmpInstanceType. Check if the HeapObject is a JSFunction
    0x302f3413c412: je     0x302f3413be20                        // __ j(equal, masm->isolate()->builtins()->CallFunction(mode), RelocInfo::CODE_TARGET);
                                                                 // I was not sure where to find this `j` function but it is in src/x64/assembler-x64.cc (Assembler::j but note
                                                                 // that there are multiple overloaded j functions so make sure  you are looking at the correct one. There is 
                                                                 // as section that discusses Assembler::j in detail later in this document.
                                                                 // so what are we jumping to? We can back up in the debugger and find out:
                                                                 // (lldb) up 2
                                                                 // (lldb job *isolate->builtins()->CallFunction(static_cast<ConvertReceiverMode>(ConvertReceiverMode::kAny))
                                                                 // This will be a builtin named CallFunction_ReceiverIsAny, and being a builtin you'll find it in 
                                                                 // `src/builtins/builtins-definitions.h`. The implementation will be in `src/builtins/builtins-call.cc` and
                                                                 // will result in `return builtin_handle(kCallFunction_ReceiverIsAny)`. This code repsonsible for generating
                                                                 // is Builtins::Generate_CallFunction and in our case that means `src/builtins/x64/builtins-x64.cc`
(lldb) expr isolate->builtins()->CallFunction(static_cast<ConvertReceiverMode>(ConvertReceiverMode::kAny))->entry()
(byte *) $317 = 0x0000302f3413be20
// Builtins::Generate_CallFunction (deps/v8/src/builtins/x64/builtins-x64.cc)
    0x302f3413be20: testb  $0x1, %dil                            // AssertFunction (deps/v8/src/x64/macro-assembler-x64.cc) check if rdi is of type smi (testb(object, Immediate(kSmiTagMask));)
                                                                 // rdi is the function to call:
(lldb) register read rdi
     rdi = 0x00001d6417d30669
(lldb) expr JSFunction::cast(func)
(v8::internal::JSFunction *) $320 = 0x00001d6417d30669
    0x302f3413be24: jne    0x302f3413be36                        // this is also generated by AssertFunciton and the call to Assembler::j If not equal this code will fall through
    0x302f3413be36: pushq  %rdi                                  // AssertFunction still, push rdi (the function) onto the stack
    0x302f3413be37: movq   -0x1(%rdi), %rdi                      // move the map into rdi. CmpObjectType(object, JS_FUNCTION_TYPE, object). HeapObject::kMapOffset which is the first field in JSFunction
    0x302f3413be3b: cmpb   $-0x1, 0xb(%rdi)                      // CmpObjectType(rdi, JS_FUNCTION_TYPE, rcx) calls CmpInstanceType. Check if the HeapObject is a JSFunction
    0x302f3413be3f: popq   %rdi                                  // pop function from stack into rdi again
    0x302f3413be40: je     0x302f3413be52                        // jump if equal will jump to a L label and return from the Check call and then return from AssertFunction
// Builtins::Generate_CallFunction (deps/v8/src/builtins/x64/builtins-x64.cc) 
    0x302f3413be52: movq   0x1f(%rdi), %rdx                      // move the SharedFunctionInfo into rdx:
(lldb) memory read -f x -c 1 -s 8 `$rdi + 0x1f`
0x1d6417d30688: 0x00001d6417d2dc21
(lldb) expr JSFunction::cast(func)->shared()
(v8::internal::SharedFunctionInfo *) $325 = 0x00001d6417d2dc21
    0x302f3413be56: testb  $-0x20, 0x87(%rdx)                    //  testl(FieldOperand(rdx, SharedFunctionInfo::kCompilerHintsOffset)
(lldb) expr JSFunction::cast(func)->shared()->compiler_hints()
(int) $332 = 1056770
(lldb) memory read -f dec -c 1 -s 4 `($rdx + 0x87)`
0x1d6417d2dca8: 1056770
    0x302f3413be5d: jne    0x302f3413bfbc                        // __ j(not_zero, &class_constructor);
    0x302f3413be63: movq   0x27(%rdi), %rsi                      // move the functions context info rsi:
(lldb) memory read -f x -c 1 -s 8 `$rdi + 0x27`
0x1d6417d30690: 0x00001d6417d03a59
(lldb) expr JSFunction::cast(func)->context()
(v8::internal::Context *) $345 = 0x00001d6417d03a59
    0x302f3413be67: testb  $0x3, 0x87(%rdx)                      // check SharedFunctionInfo::IsNativeBit::kMask | SharedFunctionInfo::IsStrictBit::kMask 
    0x302f3413be6e: jne    0x302f3413bf14                        // 
    0x302f3413bf14: movslq 0x73(%rdx), %rbx                      // __ movsxlq(rbx, FieldOperand(rdx, SharedFunctionInfo::kFormalParameterCountOffset))
    0x302f3413bf18: movabsq $0x105300b92, %r10                   // move hook_on_function_call_address into scratch register; part of CheckDebugHook
(lldb) expr isolate->debug()->hook_on_function_call_address()
(v8::internal::Address) $353 = 0x0000000105300b92
    0x302f3413bf22: cmpb   $0x0, (%r10)                          // how is this generate? In CheckDebugHook I can only find cmpb(debug_hook_active_operand, Immediate(0)); but not he previous moveabsq
    0x302f3413bf26: je     0x302f3413bfa4                        // will jump to the label at the end of CheckDebugHook
// MacroAssembler::InvokeFunction
    0x302f3413bfa4: movq   -0x60(%r13), %rdx                     // move the UndefinedValueRootIndex into rdx generated by LoadRoot(rdx, Heap::kUndefinedValueRootIndex);
(lldb) memory read -f x -c 1 -s 8 `$r13 - 0x60`
0x106000068: 0x00001d64e5d822e1
lldb) expr isolate->heap()->roots_[Heap::RootListIndex::kUndefinedValueRootIndex]
(v8::internal::Object *) $354 = 0x00001d64e5d822e1
// InvokePrologue(expected, actual, &done, &definitely_mismatches, flag, Label::kNear)
    0x302f3413bfa8: cmpq   %rax, %rbx                            // Set(rax, actual.immediate()); 
    0x302f3413bfab: je     0x302f3413bfb2                        // will return from InvokePrologue
    0x302f3413bfb2: movq   0x37(%rdi), %rcx                      // move the function code into rcx (movp(rcx, FieldOperand(function, JSFunction::kCodeOffset)))
(lldb) memory read -f x -c 1 -s 8 `$rdi + 0x37`
0x1d6417d306a0: 0x0000302f34144281
(lldb) expr JSFunction::cast(func)->code()
(v8::internal::Code *) $358 = 0x0000302f34144281
    0x302f3413bfb6: addq   $0x5f, %rcx                           // addp(rcx, Immediate(Code::kHeaderSize - kHeapObjectTag)); the instruction start follows the Code object header
(lldb) job JSFunction::cast(func)->code()
0x302f34144281: [Code]
yind = BUILTIN
name = InterpreterEntryTrampoline
compiler = unknown
Instructions (size = 1004)
0x302f341442e0     0  488b5f2f       REX.W movq rbx,[rdi+0x2f]
(lldb) memory read -f x -c 1 -s8 `$rcx + 0x5f`
0x302f341442e0: 0x075b8b482f5f8b48
// notice that rcx now point to the first instruction.
    0x302f3413bfba: jmpq   *%rcx                                // now jump to the first instruction of function :) 

    0x302f341442e0: movq   0x2f(%rdi), %rbx                     // move the feedback vector into rbx
(lldb) memory read -f x -c 1 -s 8 `$rdi + 0x2f`
0x1d6417d30698: 0x00001d6417d306e9
(lldb) expr JSFunction::cast(func)->feedback_vector()
(v8::internal::FeedbackVector *) $363 = 0x00001d6417d306a9
(lldb) job JSFunction::cast(func)->feedback_vector()
0x1d6417d306a9: [FeedbackVector] in OldSpace
 - length: 1
 SharedFunctionInfo: 0x1d6417d2dc21 <SharedFunctionInfo>
 Optimized Code: 0
 Invocation Count: 0
 Profiler Ticks: 0
 Slot #0 kCreateClosure
  [0]: 0x1d6417d306d9 <Cell value= 0x1d64e5d822e1 <undefined>>

    302f341442e4: movq   0x7(%rbx), %rbx                     // move Slot[0] from the feedback vector into rbx
(lldb) memory read -f x -c 1 -s 8 `$rbx + 0x7`
0x1d6417d306f0: 0x00001d6417d306a9
// MaybeTailCallOptimizedCodeSlot(masm, feedback_vector, rcx, r14, r15);
// MaybeTailCallOptimizedCodeSlot(MacroAssembler* masm, Register feedback_vector, Register scratch1, Register scratch2, Register scratch3)
// Register closure = rdi;
// Register optimized_code_entry = rcx;
    0x302f341442e8: movq   0xf(%rbx), %rcx                     // __ movp(optimized_code_entry, FieldOperand(feedback_vector, FeedbackVector::kOptimizedCodeOffset));
(lldb) memory read -f x -c 1 -s 8 `$rbx + 0xf`
0x1d6417d306b8: 0x0000000000000000
(lldb) expr JSFunction::cast(func)->feedback_vector()->optimized_code()
(v8::internal::Code *) $367 = 0x0000000000000000
    0x302f341442ec: testb  $0x1, %cl                           // smi test against low 8 bit of rcx called from JumpIfNotSmi -> CheckSmi
    0x302f341442ef: jne    0x302f34144486                      // JumpIfNotSmi -> Assembler::j 
    0x302f341442f5: testb  $0x1, %cl                           // test again but this time from MacroAssembler::SmiCompare and its call to AssertSmi which calls CheckSmi
    0x302f341442f8: je     0x302f3414430a                      // SmiCompare -> AssertSmi -> Check
```


Look into static inline Code* GetCodeFromTargetAddress(Address address); which could possible be
used to get the code of an address which could be useful when debugging and you only have the
address that is being jumped to.

```console
(lldb) expr JSFunction::cast(func)->code()
(v8::internal::Code *) $477 = 0x00000e6d84c44281

(lldb) expr JSFunction::cast(func)->shared()->abstract_code()->Print()
0x265e1d0ade49: [BytecodeArray] in OldSpaceParameter count 1
Frame size 8
    0 E> 0x265e1d0ade82 @    0 : 93                StackCheck
   15 S> 0x265e1d0ade83 @    1 : 6f 00 00 00       CreateClosure [0], [0], #0
         0x265e1d0ade87 @    5 : 1e fb             Star r0
21644 S> 0x265e1d0ade89 @    7 : 97                Return
Constant pool (size = 1)
0x265e1d0ade31: [FixedArray] in OldSpace
 - map = 0x265ed93822f1 <Map(HOLEY_ELEMENTS)>
 - length: 1
           0: 0x265e1d0add81 <SharedFunctionInfo bajja>
Handler Table (size = 16)

Side note on the interpreters dispatch_table_ ...
Every isolate has a interpreter as a member. An interpreter has a dispatch_table_ array of Address's (byte*):
```console
(lldb) expr isolate->interpreter()->dispatch_table_
(Address [768]) $292 = {
```
Notice that these are addresses which are index by Bytecode's. For example lets take a look the address for
`CreateClosure`:
```console
(lldb) expr isolate->interpreter()->GetBytecodeHandler(static_cast<Bytecode>(Bytecode::kCreateClosure),static_cast<OperandScale>(1))->Print()

How is `dispatch_table_` populated?  
This is done in the `Heap::IterateStrongRoots` function:
```c++
isolate_->interpreter()->IterateDispatchTable(v);
```
Which is called by `StartupDeserializer::DeserializeInto`:
```c++
isolate->heap()->IterateStrongRoots(this, VISIT_ONLY_STRONG_ROOT_LIST);
isolate->heap()->IterateSmiRoots(this);
isolate->heap()->IterateStrongRoots(this, VISIT_ONLY_STRONG);
```
 which is called by `Isolate::Init`:
```c++
if (!create_heap_objects) des->DeserializeInto(this);
```

`PrepareAndExecuteUnoptimizedCompileJob` `deps/v8/src/compiler.cc` 
```console
0x169cbdb41e04  external reference (Runtime::StackGuard)  (0x101440100)
0x169cbdb41e0d  code target (STUB)  (0x169cbda84740)
```
If you look closely the first instruction is just setting rax to zero using xor.
Next, we are pushing the pointer to the function, in this case `Runtime::StackGuard` into rbx.
We then jump to `CEntryStub`. 

What is `Runtime::StackGuard`?  
Well, it is defined in a macro in `src/runtime/runtime.h`:
```c++
...
F(StackGuard, 0, 1)
...
#define FOR_EACH_INTRINSIC(F)         \
  FOR_EACH_INTRINSIC_RETURN_PAIR(F)   \
  FOR_EACH_INTRINSIC_RETURN_OBJECT(F)


#define F(name, nargs, ressize)                                 \
  Object* Runtime_##name(int args_length, Object** args_object, \
                         Isolate* isolate);
FOR_EACH_INTRINSIC_RETURN_OBJECT(F)
#undef F
```
`StackGuard` is included in `FOR_EACH_INTRINSIC_INTERNAL` which is included by `FOR_EACH_INTRINSIC_RETURN_OBJECT`.
So that should expand to:
```c++
Object* Runtime_StackGuard(int args_lentgh, Object** args_object, Isolate* isolate);
```
If we take a look at `src/runtime/runtime.h` we can find:
```c++
static const Runtime::Function kIntrinsicFunctions[] = {
  FOR_EACH_INTRINSIC(F)
  FOR_EACH_INTRINSIC(I)
};
```

And the indexes into this array are in the enum `Runtime::FunctionId`:
```console
(lldb) expr kIntrinsicFunctions[v8::internal::Runtime::FunctionId::kStackGuard]
(const Function) $371 = (function_id = kStackGuard, intrinsic_type = RUNTIME, name = "StackGuard", entry = "UH\xffffff89\xffffffe5H\xffffff83\xffffffecP\xffffff89}\xfffffff4H\xffffff89u\xffffffe8H\xffffff89U\xffffffe0H\xffffff8b}\xffffffe0\xffffffe8\xffffffb4d\x04\xffffffff\xffffffb1\x01H\xffffff83\xfffffff8", nargs = '\0', result_size = '\x01')
```

There is also a function named `Runtime::FunctionForId` which can be used:
```console
(lldb) expr v8::internal::Runtime::FunctionForId(static_cast<v8::internal::Runtime::FunctionId>(v8::internal::Runtime::FunctionId::kStackGuard))
(const Function *) $1058 = 0x00000001029f0340
(lldb) expr v8::internal::Runtime::FunctionForId(static_cast<v8::internal::Runtime::FunctionId>(v8::internal::Runtime::FunctionId::kStackGuard))->name
(const char *const) $1059 = 0x0000000101c8be27 "StackGuard"
(lldb) expr v8::internal::Runtime::FunctionForId(static_cast<v8::internal::Runtime::FunctionId>(v8::internal::Runtime::FunctionId::kStackGuard))->nargs
(int8_t) $1060 = '\0'
(lldb) expr v8::internal::Runtime::FunctionForId(static_cast<v8::internal::Runtime::FunctionId>(v8::internal::Runtime::FunctionId::kStackGuard))->intrinsic_type
(const IntrinsicType) $1061 = RUNTIME
(lldb) expr v8::internal::Runtime::FunctionForId(static_cast<v8::internal::Runtime::FunctionId>(v8::internal::Runtime::FunctionId::kStackGuard))->entry
(v8::internal::Address) $1063 = 0x000000010142de70 "UH\x89�H��P�}�H�u�H�U�H�}���\x14\x04��\x01H\x83�
```
If `nargs` is `-1` then the funnction takes a variable number of arguments.
We can also disassemble the `entry` address using:
```console
(lldb) dis -s 0x000000010142de70
node`v8::internal::Runtime_StackGuard:
    0x10142de70 <+0>:  pushq  %rbp
    0x10142de71 <+1>:  movq   %rsp, %rbp
    0x10142de74 <+4>:  subq   $0x50, %rsp
    0x10142de78 <+8>:  movl   %edi, -0xc(%rbp)
    0x10142de7b <+11>: movq   %rsi, -0x18(%rbp)
    0x10142de7f <+15>: movq   %rdx, -0x20(%rbp)
    0x10142de83 <+19>: movq   -0x20(%rbp), %rdi
    0x10142de87 <+23>: callq  0x10046f360               ; v8::internal::Isolate::context at isolate.h:596
    0x10142de8c <+28>: movb   $0x1, %cl
```
Now that we know this we can disassemble the complete function using:
```console
(lldb) dis -n v8::internal::Runtime_StackGuard
```
As well as any other runtime function we might be interested in later. How are these stored?  
They are stored in a global named `kIntrinsicFunctions`:
```console
(lldb) target variable kIntrinsicFunctions
(lldb) expr kIntrinsicFunctions[v8::internal::Runtime::FunctionId::kStackGuard]
(const v8::internal::Runtime::Function) $227 = (function_id = kStackGuard, intrinsic_type = RUNTIME, name = "StackGuard", entry = "UH\xffffff89\xffffffe5H\xffffff83\xffffffecP\xffffff89}\xfffffff4H\xffffff89u\xffffffe8H\xffffff89U\xffffffe0H\xffffff8b}\xffffffe0\xffffffe8td\x04\xffffffff\xffffffb1\x01H\xffffff83\xfffffff8", nargs = '\0', result_size = '\x01')
(lldb) expr kIntrinsicFunctions[Builtins::Name::kStackCheck].entry
(v8::internal::Address) $229 = 0x0000000101440100 "UH\x89�H��P�}�H�u�H�U�H�}��td\x04��\x01H\x83�
```
When is this array populated?  
Being a global it is part of the data section in the object file and will be loaded into memory upon start up.

Just to recap, first the address of `Runtime_StackGuard` will be moved into register `rbx`, and then we will jump `CEntryStub`.
```console
(lldb) job *isolate->builtins()->builtin_handle(Builtins::Name::kStackCheck)
0x302f34141da1: [Code]
kind = BUILTIN
name = StackCheck
compiler = unknown
Instructions (size = 17)
0x302f34141e00     0  33c0           xorl rax,rax
0x302f34141e02     2  48bbc000440101000000 REX.W movq rbx,0x1014400c0    ;; external reference (Runtime::StackGuard)
0x302f34141e0c     c  e92f29f4ff     jmp 0x302f34084740      ;; code: STUB, CEntryStub, minor: 8


RelocInfo (size = 3)
0x302f34141e04  external reference (Runtime::StackGuard)  (0x1014400c0)
0x302f34141e0d  code target (STUB)  (0x302f34084740)

(lldb) expr kIntrinsicFunctions[v8::internal::Runtime::FunctionId::kStackGuard].entry
(v8::internal::Address) $380 = 0x00000001014400c0 "UH\x89�H��P�}�H�u�H�U�H�}��d\x04��\x01H\x83�
```
We can see that the `0x1014400c0` match. And we can disassemble this address using:
```console
(lldb) dis -s 0x1014400c0
node`v8::internal::Runtime_StackGuard:
    0x1014400c0 <+0>:  pushq  %rbp
    0x1014400c1 <+1>:  movq   %rsp, %rbp
    0x1014400c4 <+4>:  subq   $0x50, %rsp
    0x1014400c8 <+8>:  movl   %edi, -0xc(%rbp)
    0x1014400cb <+11>: movq   %rsi, -0x18(%rbp)
    0x1014400cf <+15>: movq   %rdx, -0x20(%rbp)
    0x1014400d3 <+19>: movq   -0x20(%rbp), %rdi
    0x1014400d7 <+23>: callq  0x100486590               ; v8::internal::Isolate::context at isolate.h:596
    0x1014400dc <+28>: movb   $0x1, %cl
```
What about `CEntryStub` that is being jumped to?  
```console
0x302f34141e0c     c  e92f29f4ff     jmp 0x302f34084740      ;; code: STUB, CEntryStub, minor: 8
```
`CEntryStub` is declared in `deps/v8/src/code-stubs.h`:
```c++
class CodeStub : public ZoneObject {
  ...
};
class PlatformCodeStub : public CodeStub {
  Handle<Code> GenerateCode() override;
  ...
};
class CEntryStub : public PlatformCodeStub {
  ...
};
```

Where is the CEntryStub stored?

```console
jmp 0x302f34084740      ;; code: STUB, CEntryStub, minor: 8

(lldb) expr CodeStub::Major::CEntry
(int) $404 = 4
(lldb) expr CodeStub::MajorKeyFromKey(4)
(v8::internal::CodeStub::Major) $405 = CEntry
(lldb) expr isolate->heap()->code_stubs()->FindEntry(isolate, 4)
(int) $422 = 451
(lldb) job Code::cast(isolate->heap()->code_stubs()->ValueAt(451))
0x302f34284001: [Code]
kind = STUB
major_key = CEntryStub
compiler = unknown
Instructions (size = 327)
0x302f34284060     0  55             push rbp
0x302f34284061     1  4889e5         REX.W movq rbp,rsp
0x302f34284064     4  6a06           push 0x6
0x302f34284066     6  6a00           push 0x0
0x302f34284068     8  49ba014028342f300000 REX.W movq r10,0x302f34284001
0x302f34284072    12  4152           push r10
0x302f34284074    14  4c8bf0         REX.W movq r14,rax
0x302f34284077    17  49ba001a000601000000 REX.W movq r10,0x106001a00
0x302f34284081    21  49892a         REX.W movq [r10],rbp
0x302f34284084    24  49ba9019000601000000 REX.W movq r10,0x106001990
0x302f3428408e    2e  498932         REX.W movq [r10],rsi
0x302f34284091    31  49ba101a000601000000 REX.W movq r10,0x106001a10
0x302f3428409b    3b  49891a         REX.W movq [r10],rbx
0x302f3428409e    3e  4e8d7cf508     REX.W leaq r15,[rbp+r14*8+0x8]
0x302f342840a3    43  4883e4f0       REX.W andq rsp,0xf0
0x302f342840a7    47  488965f0       REX.W movq [rbp-0x10],rsp
0x302f342840ab    4b  40f6c40f       testb rsp,0xf
0x302f342840af    4f  7401           jz 0x302f342840b2  <+0x52>
0x302f342840b1    51  cc             int3l
0x302f342840b2    52  498bfe         REX.W movq rdi,r14
0x302f342840b5    55  498bf7         REX.W movq rsi,r15
0x302f342840b8    58  48ba0000000601000000 REX.W movq rdx,0x106000000
0x302f342840c2    62  ffd3           call rbx
0x302f342840c4    64  493b8588000000 REX.W cmpq rax,[r13+0x88]
0x302f342840cb    6b  0f8447000000   jz 0x302f34284118  <+0xb8>
0x302f342840d1    71  4d8b75a8       REX.W movq r14,[r13-0x58]
0x302f342840d5    75  49baa019000601000000 REX.W movq r10,0x1060019a0
0x302f342840df    7f  4d3b32         REX.W cmpq r14,[r10]
0x302f342840e2    82  7401           jz 0x302f342840e5  <+0x85>
0x302f342840e4    84  cc             int3l
0x302f342840e5    85  488b4d08       REX.W movq rcx,[rbp+0x8]
0x302f342840e9    89  488b6d00       REX.W movq rbp,[rbp+0x0]
0x302f342840ed    8d  498d6708       REX.W leaq rsp,[r15+0x8]
0x302f342840f1    91  51             push rcx
0x302f342840f2    92  49ba9019000601000000 REX.W movq r10,0x106001990
0x302f342840fc    9c  498b32         REX.W movq rsi,[r10]
0x302f342840ff    9f  49c70200000000 REX.W movq [r10],0x0
0x302f34284106    a6  49ba001a000601000000 REX.W movq r10,0x106001a00
0x302f34284110    b0  49c70200000000 REX.W movq [r10],0x0
0x302f34284117    b7  c3             retl
0x302f34284118    b8  48c7c700000000 REX.W movq rdi,0x0
0x302f3428411f    bf  48c7c600000000 REX.W movq rsi,0x0
0x302f34284126    c6  48ba0000000601000000 REX.W movq rdx,0x106000000
0x302f34284130    d0  4989e2         REX.W movq r10,rsp
0x302f34284133    d3  4883ec08       REX.W subq rsp,0x8
0x302f34284137    d7  4883e4f0       REX.W andq rsp,0xf0
0x302f3428413b    db  4c891424       REX.W movq [rsp],r10
0x302f3428413f    df  48b8b0aa430101000000 REX.W movq rax,0x10143aab0
0x302f34284149    e9  40f6c40f       testb rsp,0xf
0x302f3428414d    ed  7401           jz 0x302f34284150  <+0xf0>
0x302f3428414f    ef  cc             int3l
0x302f34284150    f0  ffd0           call rax
0x302f34284152    f2  488b2424       REX.W movq rsp,[rsp]
0x302f34284156    f6  49bab019000601000000 REX.W movq r10,0x1060019b0
0x302f34284160   100  498b32         REX.W movq rsi,[r10]
0x302f34284163   103  49bad019000601000000 REX.W movq r10,0x1060019d0
0x302f3428416d   10d  498b22         REX.W movq rsp,[r10]
0x302f34284170   110  49bac819000601000000 REX.W movq r10,0x1060019c8
0x302f3428417a   11a  498b2a         REX.W movq rbp,[r10]
0x302f3428417d   11d  4885f6         REX.W testq rsi,rsi
0x302f34284180   120  7404           jz 0x302f34284186  <+0x126>
0x302f34284182   122  488975f8       REX.W movq [rbp-0x8],rsi
0x302f34284186   126  49bab819000601000000 REX.W movq r10,0x1060019b8
0x302f34284190   130  498b3a         REX.W movq rdi,[r10]
0x302f34284193   133  49bac019000601000000 REX.W movq r10,0x1060019c0
0x302f3428419d   13d  498b12         REX.W movq rdx,[r10]
0x302f342841a0   140  488d7c175f     REX.W leaq rdi,[rdi+rdx*1+0x5f]
0x302f342841a5   145  ffe7           jmp rdi


RelocInfo (size = -1377915637)

(lldb) expr Code::cast(isolate->heap()->code_stubs()->ValueAt(451))->entry()
(byte *) $427 = 0x0000302f34284060
```
These are not the same stubs, the opcodes match but not the entry addresses


```c++
IGNITION_HANDLER(StackCheck, InterpreterAssembler) {
  Label ok(this), stack_check_interrupt(this, Label::kDeferred);

  Node* interrupt = StackCheckTriggeredInterrupt();
  Branch(interrupt, &stack_check_interrupt, &ok);

  BIND(&ok);
  Dispatch();

  BIND(&stack_check_interrupt);
  {
    Node* context = GetContext();
    CallRuntime(Runtime::kStackGuard, context);
    Dispatch();
  }
}
```
```c++
void MacroAssembler::CallRuntime(const Runtime::Function* f,
                                 int num_arguments,
                                 SaveFPRegsMode save_doubles) {
  // If the expected number of arguments of the runtime function is
  // constant, we check that the actual number of arguments match the
  // expectation.
  CHECK(f->nargs < 0 || f->nargs == num_arguments);

  // TODO(1236192): Most runtime routines don't need the number of
  // arguments passed in because it is constant. At some point we
  // should remove this need and make the runtime routine entry code
  // smarter.
  Set(rax, num_arguments);
  LoadAddress(rbx, ExternalReference(f, isolate()));
  CEntryStub ces(isolate(), f->result_size, save_doubles);
  CallStub(&ces);
}
```
But, normally these will be called at compile time. 
My thinking is that these are all callstubs (builtins/runtime), that are generated at compile time and then made availalbe
in memory somewhere allowing V8 to jump to the address of the first instruction and start executing it.
But how are these loaded into memory?  
This is done as Isolate initialization. Lets set the following break point we can follow this process:
```console
(lldb) br s -n Isolate::Init
```
```console
(lldb) br s -f isolate.cc -l 2703
```

```c++
bool Isolate::Init(StartupDeserializer* des) {
  ...
  isolate_addresses_[IsolateAddressId::kHandlerAddress] = reinterpret_cast<Address>(handler_address());
  isolate_addresses_[IsolateAddressId::kCEntryFPrAddress] = reinterpret_cast<Address>(centry_fp_address());
  isolate_addresses_[IsolateAddressId::kFunctionAddress] = reinterpret_cast<Address>(function_address());
  isolate_addresses_[IsolateAddressId::kContextAddress] = reinterpret_cast<Address>(context_address());
  isolate_addresses_[IsolateAddressId::kPendingExceptionAddress] = reinterpret_cast<Address>(pending_exception_address());
  isolate_addresses_[IsolateAddressId::kPendingHandlerContextAddress] = reinterpret_cast<Address>(pending_handler_context_address());
  isolate_addresses_[IsolateAddressId::kPendingHandlerCodeAddress] = reinterpret_cast<Address>(pending_handler_code_address());
  isolate_addresses_[IsolateAddressId::kPendingHandlerOffsetAddress] = reinterpret_cast<Address>(pending_handler_offset_address());
  isolate_addresses_[IsolateAddressId::kPendingHandlerFPAddress] = reinterpret_cast<Address>(pending_handler_fp_address());
  isolate_addresses_[IsolateAddressId::kPendingHandlerSPAddress] = reinterpret_cast<Address>(pending_handler_sp_address());
  isolate_addresses_[IsolateAddressId::kExternalCaughtExceptionAddress] = reinterpret_cast<Address>(external_caught_exception_address());
  isolate_addresses_[IsolateAddressId::kJSEntrySPAddress] = reinterpret_cast<Address>(js_entry_sp_address());
  ...
  InitializeThreadLocal();
  ...
  if (!create_heap_objects) des->DeserializeInto(this);
```
The functions, like handler_address(), can be found in v8/src/isolate.h: 
```c++
inline Address* handler_address() { return &thread_local_top_.handler_; }
```
Notice that when the isolate_addresses_ array is populated and when InitializeThreadLocal is call the heap is still empty. What is 
happening is that pointers are being setup. When `DeserializeInto(this)` is called is when the heap is populated:
```c++
void StartupDeserializer::DeserializeInto(Isolate* isolate) {
  Initialize(isolate);
  BuiltinDeserializer builtin_deserializer(isolate, builtin_data_);
  ...
  builtin_deserializer.DeserializeEagerBuiltins();
  ...
  CodeStub::GenerateFPStubs(this);
  StoreBufferOverflowStub::GenerateFixedRegStubsAheadOfTime(this); 
}
```
DeserializeEagerBuiltins will populate the builtins_ array which is part of Builtins which is a member of Isolate.
There seems to be codestubs that have to be generated, they cannot be serialized into the snapshot. 

`CodeStub::GenerateFPStubs(this)`, it is here that CEntryStub is generated:
```c++
void CodeStub::GenerateFPStubs(Isolate* isolate) {
  // Generate if not already in cache.
  SaveFPRegsMode mode = kSaveFPRegs;
  CEntryStub(isolate, 1, mode).GetCode();
  StoreBufferOverflowStub(isolate, mode).GetCode();
}
```

We can check that check the various data structures before and after using:
```console
(lldb) expr builtins_
(lldb) expr heap->roots_
(lldb) expr isolate_addresses_
(lldb) expr thread_local_top_
```
For `isolate_addresses_` which are pointers we can inspect them like this:
```console
(lldb) expr isolate_addresses_[11]
(v8::internal::Address) $194 = 0x0000000104808820 ""
(lldb) memory read -f x -c 1 -s 8 0x0000000104808820
0x104808820: 0x0000000000000000
```

```console
(lldb) expr CodeStub::Major::CEntry
(int) $120 = 4
```


```c++
Local<Value> result = script.ToLocalChecked()->Run();
```

If we back down through the call frame again, we will be in `execution.cc` and see this now familar code:
```c++
    typedef Object* (*JSEntryFunction)(Object* new_target, Object* target,
                                     Object* receiver, int argc,
                                     Object*** args);
    Handle<Code> code = is_construct
       ? isolate->factory()->js_construct_entry_code()
       : isolate->factory()->js_entry_code();
    ...
  
    // start of the function identified by code->entry() address.
    JSEntryFunction stub_entry = FUNCTION_CAST<JSEntryFunction>(code->entry());

    // Call the function through the right JS entry stub.
    Object* orig_func = *new_target;
    Object* func = *target;
    Object* recv = *receiver;
    Object*** argv = reinterpret_cast<Object***>(args);
    if (FLAG_profile_deserialization && target->IsJSFunction()) {
      PrintDeserializedCodeInfo(Handle<JSFunction>::cast(target));
    }
    RuntimeCallTimerScope timer(isolate, &RuntimeCallStats::JS_Execution);
    value = CALL_GENERATED_CODE(isolate, stub_entry, orig_func, func, recv, argc, argv);
  }
```
Notice that `code` will be what `isolate->factory()->js_entry_code()` returns:
```console
(lldb) p isolate->factory()->js_entry_code()
```

You can verify that it is in fact bootstrap.js that `func` represents by issueing:
```console
(lldb) job func
```

`Handle<Code> code represents generated machine code, and we can see that this instance is on Kind `STUB`:
```console
(lldb) p code->kind()
(v8::internal::Code::Kind) $32 = STUB
```

Next, we have `CALL_GENERATED_CODE` which can be found in `src/x64/simulator-x64.h`:
```c++
// Since there is no simulator for the x64 architecture the only thing we can
// do is to call the entry directly.
// TODO(X64): Don't pass p0, since it isn't used?
#efine CALL_GENERATED_CODE(isolate, entry, p0, p1, p2, p3, p4) \
  (entry(p0, p1, p2, p3, p4))
```
So this will be an call that looks like this:
```c++
entry(orig_func, func, recv, argc, argv);
```
```console
    0x100e381b4 <+1348>: movq   -0x118(%rbp), %rdx                  // move the value of address of code->entry() into rdx
    0x100e381bb <+1355>: movq   -0x120(%rbp), %rdi                  // first argument which is orig_func
->  0x100e381c2 <+1362>: movq   -0x128(%rbp), %rsi                  // second argument which is func
    0x100e381c9 <+1369>: movq   -0x130(%rbp), %rcx                  // third argument which is recv
    0x100e381d0 <+1376>: movl   -0x30(%rbp), %eax                   // fourth arg which is argc 
    0x100e381d3 <+1379>: movq   -0x138(%rbp), %r8                   // fifth argument which is argv
    0x100e381da <+1386>: movq   %rdx, -0x1c0(%rbp)                  // move code->entry from rdx into local varialbe 
    0x100e381e1 <+1393>: movq   %rcx, %rdx                          // move recv into rdx
    0x100e381e4 <+1396>: movl   %eax, %ecx                          // move argc into ecx
    0x100e381e6 <+1398>: movq   -0x1c0(%rbp), %r9                   // move code-entry into register r9
    0x100e381ed <+1405>: callq  *%r9                                // call address in register 9

(lldb) memory read -f x -c 1 -s 8 `$rbp - 0x118`
0x7fff5fbfd828: 0x0000302f34084060
(lldb) expr code->entry()
(byte *) $186 = 0x0000302f34084060

(lldb) memory read -f x -c 1 -s 8 `$rbp - 0x120`
0x7fff5fbfd820: 0x00001d64e5d822e1
(lldb) expr orig_func
(v8::internal::Object *) $209 = 0x00001d64e5d822e1

(lldb) memory read -f x -c 1 -s 8 `$rbp - 0x128`
0x7fff5fbfd818: 0x00001d6417d30669
(lldb) expr func
(v8::internal::Object *) $211 = 0x00001d6417d30669

(lldb) memory read -f x -c 1 -s 8 `$rbp - 0x130`
0x7fff5fbfd810: 0x00001d64f0a82239
(lldb) expr recv
(v8::internal::Object *) $213 = 0x00001d64f0a82239

(lldb) memory read -f x -c 1 -s 4 `$rbp - 0x30`
0x7fff5fbfd910: 0x00000000
(lldb) expr argc
(int) $216 = 0

(lldb) memory read -f x -c 1 -s 8 `$rbp - 0x138`
0x7fff5fbfd808: 0x0000000000000000
(lldb) expr argv
(v8::internal::Object ***) $219 = 0x0000000000000000

(lldb) memory read -f x -c 1 -s 8 `$rbp - 0x1c0`
0x7fff5fbfd780: 0x0000302f34084060
```

The last instruction `callq *%r9` will call into JSEntryStub:
```console
->  0x302f34084060: pushq  %rbp                                  // push callers base frame pointer saving it so we can restore it
    0x302f34084061: movq   %rsp, %rbp                            // mov the current value of rsp to into rbp which will be the frame pointer for this function
    0x302f34084064: pushq  $0x2                                  // this is pushing an immediate value 2 onto the stack. Where does this come from, the following:
(lldb) p v8::internal::StackFrame::TypeToMarker(static_cast<v8::internal::StackFrame::Type>(v8::internal::StackFrame::Type::ENTRY))
(int32_t) $262 = 2
    0x302f34084066: movabsq $0x106001990, %r10                   // move the context address into r10
(lldb) expr isolate->isolate_addresses_[IsolateAddressId::kContextAddress]
(v8::internal::Address) $230 = 0x0000000106001990 
    0x302f34084070: movq   (%r10), %r10                          // the context address is a pointer, this will dereference it
(lldb) memory read -f x -c 1 -s 8 `isolate->isolate_addresses_[IsolateAddressId::kContextAddress]`
0x106001990: 0x00001d6417d03a59
(lldb) register read r10
     r10 = 0x00001d6417d03a59
     0x302f34084073: pushq  %r10                                // push the dereferences context onto the stack
     0x302f34084075: pushq  %r12                                // r12 must be preserved accross function calls so save it and pop it later before returning
     0x302f34084077: pushq  %r13                                // r13 must be preserved accross function calls so save it and pop it later before returning
     0x302f34084079: pushq  %r14                                // r14 must be preserved accross function calls so save it and pop it later before returning
     0x302f3408407b: pushq  %r15                                // r14 must be preserved accross function calls so save it and pop it later before returning
     0x302f3408407d: pushq  %rbx                                // rbx must be preserved accross function calls so save it and pop it later before returning
     0x302f3408407e: movabsq $0x106000048, %r13                 // move the value of the roots_array_start into r13
(lldb) expr isolate->heap()->roots_array_start()
(v8::internal::Object **) $233 = 0x0000000106000048
     0x302f34084088: addq   $0x80, %r13                         // addp(kRootRegister, Immediate(kRootRegisterBias)); kRootRegisterBias is 128
     0x302f3408408f: movabsq $0x106001a00, %r10                 // move he CEntryFPAddress into r10
(lldb) memory read -f x -c 1 -s 8 isolate->isolate_addresses_[IsolateAddressId::kCEntryFPAddress]
0x106001a00: 0x0000000000000000
    0x302f34084099: pushq  (%r10)                               // dereference CEntryAddress and push onto the stack
(lldb) memory read -f x -c 1 -s 8 0x0000000106001a00
0x106001a00: 0x0000000000000000
    0x302f3408409c: movabsq 0x106001a20, %rax                   // move JSEntrySPAddress into rax
(lldb) memory read -f x -c 1 -s 8 isolate->isolate_addresses_[IsolateAddressId::kJSEntrySPAddress]
0x106001a20: 0x0000000000000000 
    0x302f340840a6: testq  %rax, %rax                           // is rax zero? if so there is an outer js call
(lldb) register read rax
     rax = 0x0000000000000000
    0x302f340840af: pushq  $0x2                                 // v8::internal::StackFrame::OUTERMOST_JSENTRY_FRAME))
    0x302f340840b1: movq   %rbp, %rax                           // move this functions base pointer into rax 
    0x302f340840b4: movabsq %rax, 0x106001a20                   // store the base pointer in isolate_addresses_[IsolateAddressId::kJSEntrySPAddress]
(lldb) memory read -f x -c 1 -s 8 `isolate->isolate_addresses_[IsolateAddressId::kJSEntrySPAddress]`
0x106001a20: 0x0000000000000000
after
(lldb) memory read -f x -c 1 -s 8 `isolate->isolate_addresses_[IsolateAddressId::kJSEntrySPAddress]`
0x106001a20: 0x00007fff5fbfd760
(lldb) register read rbp
     rbp = 0x00007fff5fbfd940
    0x302f340840be: jmp    0x302f340840c5
    0x302f340840c5: jmp    0x302f340840e0
    0x302f340840e0: movabsq $0x106001a08, %r10                  // move isolate_addresses_[IsolateAddressId::kHandlerAddress]` into r10
(lldb) memory read -f x -c 1 -s 8 `isolate->isolate_addresses_[IsolateAddressId::kHandlerAddress]`
0x106001a08: 0x0000000000000000
    0x302f340840ea: pushq  (%r10)                               // push the dereferenced handler address
    0x302f340840ed: movabsq $0x106001a08, %r10                  // move isolate_addresses_[IsolateAddressId::kHandlerAddress]` into r10
    0x302f340840f7: movq   %rsp, (%r10)                         // move the value of the current stack pointer into the object pointed to be r10
(lldb) memory read -f x -c 1 -s 8 0x106001a08
0x106001a08: 0x00007fff5fbfd710
(lldb) register read rsp
     rsp = 0x00007fff5fbfd710
    0x302f340840fa: callq  0x302f341418c0                       // call JSEntryTrampoline builtin
(lldb) expr isolate->builtins()->builtin_handle(Builtins::Name::kJSEntryTrampoline)->entry()
(byte *) $255 = 0x0000302f341418c0 
      
```
In `deps/v8/src/builtins/x64/builtins-x64.cc` we can find `Generate_JSEntryTrampolineHelper` which is what generates the builtin. As this is done
a compile time we can put a break point in it (you can debug mksnapshot though which is done elsewhere in this document).
```console
    0x302f341418c0: movq   %rdi, %r11                           // move the orig_fun into r11
    0x302f341418c3: movq   %rsi, %rdi                           // move func into rdi
    0x302f341418c6: xorl   %esi, %esi                           // zero out esi
    0x302f341418c8: pushq  %rbp                                 // EnterFrame (deps/v8/src/x64/macro-assembler-x64.cc)
    0x302f341418c9: movq   %rsp, %rbp                           // 
    0x302f341418cc: pushq  $0x1c                                // push 0x1c (decimal 28)
(lldb) p v8::internal::StackFrame::TypeToMarker(static_cast<v8::internal::StackFrame::Type>(v8::internal::StackFrame::Type::INTERNAL))
(int32_t) $256 = 28
    0x302f341418ce: movabsq $0x302f34141861, %r10               // CodeObject() what is this, seems like it is Handle<HeapObject> which will be patched later
                                                                // I think this might be the register file?
(lldb) memory read -f x -c 1 -s 8 0x302f34141861
0x302f34141861: 0x6900001d64ceb827
    0x302f341418d8: pushq  %r10                                 // push the HandleHeapObject into the stack
    0x302f341418da: movabsq $0x1d64e5d822e1, %r10               // move undefined value into r10 
(lldb) expr *isolate->factory()->undefined_value()
(v8::internal::Oddball *) $264 = 0x00001d64e5d822e1
    0x302f341418e4: cmpq   %r10, (%rsp)
    0x302f341418e8: jne    0x302f341418fa                       // last of EnterFrame if we don't abort that is
    0x302f341418fa: movabsq $0x106001990, %r10                  // move the ContextAdress into r10 (the scratch register for x86)
(lldb) memory read -f x -c 1 -s 8 `isolate->isolate_addresses_[IsolateAddressId::kContextAddress]`
0x106001990: 0x00001d6417d03a59
    0x302f34141904: movq   (%r10), %rsi                         // deref and move into context into rsi
    0x302f34141907: pushq  %rdi                                 // push function onto the stack
    0x302f34141908: pushq  %rdx                                 // push recv onto the stack (this was moved above with 0x302f34141908: pushq  %rdx)
    0x302f34141909: movq   %rcx, %rax                           // move argc into rax (this was moved above with 0x100e381e4 <+1396>: movl   %eax, %ecx)
    0x302f3414190c: movq   %r8, %rbx                            // move argv into rbx
    0x302f3414190f: movq   %r11, %rdx                           // move orig_func into rdx
// Generate_CheckStackOverflow  TODO: got through this as I struggled to understand/map the generated instructions to the source code :( 
    0x302f34141912: movq   0xd08(%r13), %r10                    // 
    0x302f34141919: movq   %rsp, %rcx
    0x302f3414191c: subq   %r10, %rcx
    0x302f3414191f: movq   %rax, %r11
    0x302f34141922: shlq   $0x3, %r11
    0x302f34141926: cmpq   %r11, %rcx
    0x302f34141929: jg     0x302f34141940
// Generate_JSEntryTrampolineHelper
    0x302f34141940: xorl   %ecx, %ecx                           // zero out ecx, for the following loop
    0x302f34141942: jmp    0x302f3414194f                       // jump to entry label
    0x302f3414194f: cmpq   %rax, %rcx                           // compare argc with rcx (this was moved above). 
    0x302f34141952: jne    0x302f34141944                       // entry the loop if. This is pushing the argv values onto the stack in a loop, but we don't have any so we fall through
    0x302f34141954: callq  0x302f3413c400                       //
(lldb) expr isolate->builtins()->Call(static_cast<ConvertReceiverMode>(ConvertReceiverMode::kAny))->entry()
(byte *) $279 = 0x0000302f3413c400 "@\xfffffff6\xffffffc7\x01\x0f\xffffff84F"
// for the full object with assembly language instructions:
(lldb) job *isolate->builtins()->Call(static_cast<ConvertReceiverMode>(ConvertReceiverMode::kAny))
// Builtins::Generate_Call_ReceiverIsAny 'deps/v8/src/builtins/builtins-call-gen.cc':
// void Builtins::Generate_Call_ReceiverIsAny(MacroAssembler* masm) {
//   Generate_Call(masm, ConvertReceiverMode::kAny);
// }
// Builtins::Generate_Call `deps/v8/src/builtins/x64/builtins-x64.cc`
    0x302f3413c400: testb  $0x1, %dil                           // move the immediate 1 into the low 8 bits or rdi  // __ JumpIfSmi(rdi, &non_callable); which consists of CheckSmi
    0x302f3413c404: je     0x302f3413c450                       // if ZF = 1 then jump. This would happen if rdi was a SMI// __ JumpIfSmi(rdi, &non_callable: which after CheckSmi will jump. rdi is the target and not a smi in our case
// recall that rdi is the function:
(lldb) register read rdi
     rdi = 0x00001d6417d30669
(lldb) expr func
(v8::internal::Object *) $280 = 0x00001d6417d30669
// Now I think that func is/was of type JSFunction (deps/v8/src/objects.h) as it was cast to Object* by:
// auto fun = i::Handle<i::JSFunction>::cast(Utils::OpenHandle(this));
// class JSFunction: public JSObject {
 public:
  // [prototype_or_initial_map]:
  DECL_ACCESSORS(prototype_or_initial_map, Object)
    0x302f3413c40a: movq   -0x1(%rdi), %rcx                      // move the map into rcx. CmpObjectType(rdi, JS_FUNCTION_TYPE, rcx). HeapObject::kMapOffset which is the first field in JSFunction:
(lldb) memory read -f x -c 1 -s 8 `$rdi - 1`
0x1d6417d30668: 0x00001d6433c82521
(lldb) expr JSFunction::cast(func)->prototype_or_initial_map()
(v8::internal::Object *) $286 = 0x00001d64e5d82321
    0x302f3413c40e: cmpb   $-0x1, 0xb(%rcx)                      // CmpObjectType(rdi, JS_FUNCTION_TYPE, rcx) calls CmpInstanceType. Check if the HeapObject is a JSFunction
    0x302f3413c412: je     0x302f3413be20                        // __ j(equal, masm->isolate()->builtins()->CallFunction(mode), RelocInfo::CODE_TARGET);
                                                                 // I was not sure where to find this `j` function but it is in src/x64/assembler-x64.cc (Assembler::j but note
                                                                 // that there are multiple overloaded j functions so make sure  you are looking at the correct one. There is 
                                                                 // as section that discusses Assembler::j in detail later in this document.
                                                                 // so what are we jumping to? We can back up in the debugger and find out:
                                                                 // (lldb) up 2
                                                                 // (lldb job *isolate->builtins()->CallFunction(static_cast<ConvertReceiverMode>(ConvertReceiverMode::kAny))
                                                                 // This will be a builtin named CallFunction_ReceiverIsAny, and being a builtin you'll find it in 
                                                                 // `src/builtins/builtins-definitions.h`. The implementation will be in `src/builtins/builtins-call.cc` and
                                                                 // will result in `return builtin_handle(kCallFunction_ReceiverIsAny)`. This code repsonsible for generating
                                                                 // is Builtins::Generate_CallFunction and in our case that means `src/builtins/x64/builtins-x64.cc`
(lldb) expr isolate->builtins()->CallFunction(static_cast<ConvertReceiverMode>(ConvertReceiverMode::kAny))->entry()
(byte *) $317 = 0x0000302f3413be20
// Builtins::Generate_CallFunction (deps/v8/src/builtins/x64/builtins-x64.cc)
    0x302f3413be20: testb  $0x1, %dil                            // AssertFunction (deps/v8/src/x64/macro-assembler-x64.cc) check if rdi is of type smi (testb(object, Immediate(kSmiTagMask));)
                                                                 // rdi is the function to call:
(lldb) register read rdi
     rdi = 0x00001d6417d30669
(lldb) expr JSFunction::cast(func)
(v8::internal::JSFunction *) $320 = 0x00001d6417d30669
    0x302f3413be24: jne    0x302f3413be36                        // this is also generated by AssertFunciton and the call to Assembler::j If not equal this code will fall through
    0x302f3413be36: pushq  %rdi                                  // AssertFunction still, push rdi (the function) onto the stack
    0x302f3413be37: movq   -0x1(%rdi), %rdi                      // move the map into rdi. CmpObjectType(object, JS_FUNCTION_TYPE, object). HeapObject::kMapOffset which is the first field in JSFunction
    0x302f3413be3b: cmpb   $-0x1, 0xb(%rdi)                      // CmpObjectType(rdi, JS_FUNCTION_TYPE, rcx) calls CmpInstanceType. Check if the HeapObject is a JSFunction
    0x302f3413be3f: popq   %rdi                                  // pop function from stack into rdi again
    0x302f3413be40: je     0x302f3413be52                        // jump if equal will jump to a L label and return from the Check call and then return from AssertFunction
// Builtins::Generate_CallFunction (deps/v8/src/builtins/x64/builtins-x64.cc) 
    0x302f3413be52: movq   0x1f(%rdi), %rdx                      // move the SharedFunctionInfo into rdx:
(lldb) memory read -f x -c 1 -s 8 `$rdi + 0x1f`
0x1d6417d30688: 0x00001d6417d2dc21
(lldb) expr JSFunction::cast(func)->shared()
(v8::internal::SharedFunctionInfo *) $325 = 0x00001d6417d2dc21
    0x302f3413be56: testb  $-0x20, 0x87(%rdx)                    //  testl(FieldOperand(rdx, SharedFunctionInfo::kCompilerHintsOffset)
(lldb) expr JSFunction::cast(func)->shared()->compiler_hints()
(int) $332 = 1056770
(lldb) memory read -f dec -c 1 -s 4 `($rdx + 0x87)`
0x1d6417d2dca8: 1056770
    0x302f3413be5d: jne    0x302f3413bfbc                        // __ j(not_zero, &class_constructor);
    0x302f3413be63: movq   0x27(%rdi), %rsi                      // move the functions context info rsi:
(lldb) memory read -f x -c 1 -s 8 `$rdi + 0x27`
0x1d6417d30690: 0x00001d6417d03a59
(lldb) expr JSFunction::cast(func)->context()
(v8::internal::Context *) $345 = 0x00001d6417d03a59
    0x302f3413be67: testb  $0x3, 0x87(%rdx)                      // check SharedFunctionInfo::IsNativeBit::kMask | SharedFunctionInfo::IsStrictBit::kMask 
    0x302f3413be6e: jne    0x302f3413bf14                        // 
    0x302f3413bf14: movslq 0x73(%rdx), %rbx                      // __ movsxlq(rbx, FieldOperand(rdx, SharedFunctionInfo::kFormalParameterCountOffset))
    0x302f3413bf18: movabsq $0x105300b92, %r10                   // move hook_on_function_call_address into scratch register; part of CheckDebugHook
(lldb) expr isolate->debug()->hook_on_function_call_address()
(v8::internal::Address) $353 = 0x0000000105300b92
    0x302f3413bf22: cmpb   $0x0, (%r10)                          // how is this generate? In CheckDebugHook I can only find cmpb(debug_hook_active_operand, Immediate(0)); but not he previous moveabsq
    0x302f3413bf26: je     0x302f3413bfa4                        // will jump to the label at the end of CheckDebugHook
// MacroAssembler::InvokeFunction
    0x302f3413bfa4: movq   -0x60(%r13), %rdx                     // move the UndefinedValueRootIndex into rdx generated by LoadRoot(rdx, Heap::kUndefinedValueRootIndex);
(lldb) memory read -f x -c 1 -s 8 `$r13 - 0x60`
0x106000068: 0x00001d64e5d822e1
lldb) expr isolate->heap()->roots_[Heap::RootListIndex::kUndefinedValueRootIndex]
(v8::internal::Object *) $354 = 0x00001d64e5d822e1
// InvokePrologue(expected, actual, &done, &definitely_mismatches, flag, Label::kNear)
    0x302f3413bfa8: cmpq   %rax, %rbx                            // Set(rax, actual.immediate()); 
    0x302f3413bfab: je     0x302f3413bfb2                        // will return from InvokePrologue
    0x302f3413bfb2: movq   0x37(%rdi), %rcx                      // move the function code into rcx (movp(rcx, FieldOperand(function, JSFunction::kCodeOffset)))
(lldb) memory read -f x -c 1 -s 8 `$rdi + 0x37`
0x1d6417d306a0: 0x0000302f34144281
(lldb) expr JSFunction::cast(func)->code()
(v8::internal::Code *) $358 = 0x0000302f34144281
    0x302f3413bfb6: addq   $0x5f, %rcx                           // addp(rcx, Immediate(Code::kHeaderSize - kHeapObjectTag)); the instruction start follows the Code object header
(lldb) job JSFunction::cast(func)->code()
0x302f34144281: [Code]
yind = BUILTIN
name = InterpreterEntryTrampoline
compiler = unknown
Instructions (size = 1004)
0x302f341442e0     0  488b5f2f       REX.W movq rbx,[rdi+0x2f]
(lldb) memory read -f x -c 1 -s8 `$rcx + 0x5f`
0x302f341442e0: 0x075b8b482f5f8b48
// notice that rcx now point to the first instruction.
    0x302f3413bfba: jmpq   *%rcx                                // now jump to the first instruction of function :) 

    0x302f341442e0: movq   0x2f(%rdi), %rbx                     // move the feedback vector into rbx
(lldb) memory read -f x -c 1 -s 8 `$rdi + 0x2f`
0x1d6417d30698: 0x00001d6417d306e9
(lldb) expr JSFunction::cast(func)->feedback_vector()
(v8::internal::FeedbackVector *) $363 = 0x00001d6417d306a9
(lldb) job JSFunction::cast(func)->feedback_vector()
0x1d6417d306a9: [FeedbackVector] in OldSpace
 - length: 1
 SharedFunctionInfo: 0x1d6417d2dc21 <SharedFunctionInfo>
 Optimized Code: 0
 Invocation Count: 0
 Profiler Ticks: 0
 Slot #0 kCreateClosure
  [0]: 0x1d6417d306d9 <Cell value= 0x1d64e5d822e1 <undefined>>

    302f341442e4: movq   0x7(%rbx), %rbx                     // move Slot[0] from the feedback vector into rbx
(lldb) memory read -f x -c 1 -s 8 `$rbx + 0x7`
0x1d6417d306f0: 0x00001d6417d306a9
// MaybeTailCallOptimizedCodeSlot(masm, feedback_vector, rcx, r14, r15);
// MaybeTailCallOptimizedCodeSlot(MacroAssembler* masm, Register feedback_vector, Register scratch1, Register scratch2, Register scratch3)
// Register closure = rdi;
// Register optimized_code_entry = rcx;
    0x302f341442e8: movq   0xf(%rbx), %rcx                     // __ movp(optimized_code_entry, FieldOperand(feedback_vector, FeedbackVector::kOptimizedCodeOffset));
(lldb) memory read -f x -c 1 -s 8 `$rbx + 0xf`
0x1d6417d306b8: 0x0000000000000000
(lldb) expr JSFunction::cast(func)->feedback_vector()->optimized_code()
(v8::internal::Code *) $367 = 0x0000000000000000
    0x302f341442ec: testb  $0x1, %cl                           // smi test against low 8 bit of rcx called from JumpIfNotSmi -> CheckSmi
    0x302f341442ef: jne    0x302f34144486                      // JumpIfNotSmi -> Assembler::j 
    0x302f341442f5: testb  $0x1, %cl                           // test again but this time from MacroAssembler::SmiCompare and its call to AssertSmi which calls CheckSmi
    0x302f341442f8: je     0x302f3414430a                      // SmiCompare -> AssertSmi -> Check
    

   



RelocInf  (size = 10) 0x302f341418d0  embedded object  (0x302f34141861 <Code BUILTIN>)
0x302f341418dc  embedded object  (0x1d64e5d822e1 <undefined>)
0x302f341418f5  code target (BUILTIN)  (0x302f341659c0)
0x302f341418fc  external reference (Isolate::context_address)  (0x106001990)
0x302f34141933  external reference (Runtime::ThrowStackOverflow)  (0x101438e80)
0x302f3414193c  code target (STUB)  (0x302f34084740)
0x302f34141955  code target (BUILTIN)  (0x302f3413c400)
0x302f3414196b  code target (BUILTIN)  (0x302f341659c0)
```

### SharedFunctionInfo::kCompilerHintsOffset
                                SHARED_FUNCTION_INFO_FIELDS)

### JSEntryStub 
```console
Process 568 stopped
* thread #1: tid = 0xf3b301, 0x000020819e104060, queue = 'com.apple.main-thread', stop reason = instruction step into
    frame #0: 0x000020819e104060
->  0x20819e104060: pushq  %rbp
    0x20819e104061: movq   %rsp, %rbp
    0x20819e104064: pushq  $0x2
    0x20819e104066: movabsq $0x104803190, %r10        ; imm = 0x104803190
```
I've previously mapped the assembly with the void `JSEntryStub::Generate` function but I did not cover:
```console
0x20819e1040fa    9a  e8c1d70b00     call 0x20819e1c18c0  (JSEntryTrampoline)    ;; code: BUILTIN
```
This  matches the assembly generated in `Generate_JSEntryTrampolineHelper(MacroAssembler* masm, bool is_construct)` 
in `src/builtins/x64/builtins-x64.cc`:
```console
    frame #0: 0x000020819e1c18c0
->  0x20819e1c18c0: movq   %rdi, %r11               // __ movp(r11, rdi);
    0x20819e1c18c3: movq   %rsi, %rdi               // __ movp(rdi, rsi);
    0x20819e1c18c6: xorl   %esi, %esi               // __ Set(rsi, 0);
    0x20819e1c18c8: pushq  %rbp                     // FrameScope scope(masm, StackFrame::INTERNAL);
    0x20819e1c18c9: movq   %rsp, %rbp               // FrameScope scope(masm, StackFrame::INTERNAL);
    0x20819e1c18cc: pushq  $0x1c                    // Is this also generated by the above scope?
    0x20819e1c18ce: movabsq $0x20819e1c1861, %r10   // Is this also generated by the above scope?
    ...
->  0x20819e1c1904: movq   (%r10), %rsi             // 
    0x20819e1c1907: pushq  %rdi                     // __ Push(rdi); func onto the stack
    0x20819e1c1908: pushq  %rdx                     // __ Push(rdx); recv onto the stack
    0x20819e1c1909: movq   %rcx, %rax               // __ movp(rax, rcx); argc
    0x20819e1c190c: movq   %r8, %rbx                // __ movp(rbx, r8); pointer to args
    0x20819e1c190f: movq   %r11, %rdx               // __ movp(rdx, r11); new target into rdx
    0x20819e1c1912: movq   0xd08(%r13), %r10        // Generate_CheckStackOverflow(masm, kRaxIsUntaggedInt);
    0x20819e1c1919: movq   %rsp, %rcx               // Generate_CheckStackOverflow(masm, kRaxIsUntaggedInt);
    0x20819e1c191c: subq   %r10, %rcx               // Generate_CheckStackOverflow(masm, kRaxIsUntaggedInt);
    0x20819e1c191f: movq   %rax, %r11               // Generate_CheckStackOverflow(masm, kRaxIsUntaggedInt);
    0x20819e1c1922: shlq   $0x3, %r11               // Generate_CheckStackOverflow(masm, kRaxIsUntaggedInt);
    0x20819e1c1926: cmpq   %r11, %rcx               // Generate_CheckStackOverflow(masm, kRaxIsUntaggedInt);
    0x20819e1c1929: jg     0x20819e1c1940           // jump if alright
    0x20819e1c192f: xorl   %eax, %eax               // handle stack overflow
    0x20819e1c1931: movabsq $0x1014259b0, %rbx      // handle stack overflow
    0x20819e1c193b: callq  0x20819e104740           // handle stack overflow
->  0x20819e1c1940: xorl   %ecx, %ecx               // __ Set(rcx, 0);  // Set loop variable to 0.
    0x20819e1c1942: jmp    0x20819e1c194f           // __ jmp(&entry, Label::kNear);
->  0x20819e1c194f: cmpq   %rax, %rcx               // both are zero at this stage
->  0x20819e1c1952: jne    0x20819e1c1944
                                                    // Handle<Code> builtin = is_construct
                                                    //     ? BUILTIN_CODE(masm->isolate(), Construct)
                                                    //     : masm->isolate()->builtins()->Call();
    0x20819e1c1954: callq  0x20819e1bc400           //__ Call(builtin, RelocInfo::CODE_TARGET);
```
Lets take a look `Call` in `src/x64/macro-assembler-x64.h`:
```c++
void Call(Handle<Code> code_object, RelocInfo::Mode rmode);
```
And the implementation look like this:
```c++
void TurboAssembler::Call(Handle<Code> code_object, RelocInfo::Mode rmode) {
#ifdef DEBUG
  int end_position = pc_offset() + CallSize(code_object);
#endif
  DCHECK(RelocInfo::IsCodeTarget(rmode));
  call(code_object, rmode);
#ifdef DEBUG
  DCHECK_EQ(end_position, pc_offset());
#endif
}
```
I think that `call` will end up in `src/x64/assembler-x64.cc`:
```c++
void Assembler::call(Handle<Code> target, RelocInfo::Mode rmode) {
  EnsureSpace ensure_space(this);
  // 1110 1000 #32-bit disp.
  emit(0xE8);
  emit_code_target(target, rmode);
}
```
`src/x64/assembler-x64-inl.h`
emit(0xE8)

### RUNTIME_FUNCTION
```c++
RUNTIME_FUNCTION(Runtime_InterpreterNewClosure) {
  HandleScope scope(isolate);
  DCHECK_EQ(4, args.length());
  CONVERT_ARG_HANDLE_CHECKED(SharedFunctionInfo, shared, 0);
  CONVERT_ARG_HANDLE_CHECKED(FeedbackVector, vector, 1);
  CONVERT_SMI_ARG_CHECKED(index, 2);
  CONVERT_SMI_ARG_CHECKED(pretenured_flag, 3);
  Handle<Context> context(isolate->context(), isolate);
  FeedbackSlot slot = FeedbackVector::ToSlot(index);
  Handle<Cell> vector_cell(Cell::cast(vector->Get(slot)), isolate);
  return *isolate->factory()->NewFunctionFromSharedFunctionInfo(
      shared, context, vector_cell,
      static_cast<PretenureFlag>(pretenured_flag));
}
```
`deps/v8/src/arguments.h` we find the `RUNTIME_FUNCTION` macro:
```c++
#define RUNTIME_FUNCTION_RETURNS_TYPE(Type, Name)                             \
  static INLINE(Type __RT_impl_##Name(Arguments args, Isolate* isolate));     \
                                                                              \
  V8_NOINLINE static Type Stats_##Name(int args_length, Object** args_object, \
                                       Isolate* isolate) {                    \
    RuntimeCallTimerScope timer(isolate, &RuntimeCallStats::Name);            \
    TRACE_EVENT0(TRACE_DISABLED_BY_DEFAULT("v8.runtime"),                     \
                 "V8.Runtime_" #Name);                                        \
    Arguments args(args_length, args_object);                                 \
    return __RT_impl_##Name(args, isolate);                                   \
  }                                                                           \
                                                                              \
  Type Name(int args_length, Object** args_object, Isolate* isolate) {        \
    DCHECK(isolate->context() == nullptr || isolate->context()->IsContext()); \
    CLOBBER_DOUBLE_REGISTERS();                                               \
    if (V8_UNLIKELY(FLAG_runtime_stats)) {                                    \
      return Stats_##Name(args_length, args_object, isolate);                 \
    }                                                                         \
    Arguments args(args_length, args_object);                                 \
    return __RT_impl_##Name(args, isolate);                                   \
  }                                                                           \
                                                                              \
  static Type __RT_impl_##Name(Arguments args, Isolate* isolate)

#define RUNTIME_FUNCTION(Name) RUNTIME_FUNCTION_RETURNS_TYPE(Object*, Name)
```
So lets see what that expands to:
```c++
  static INLINE(Type __RT_impl_Runtime_InterpreterNewClosure(Arguments args, Isolate* isolate));

  V8_NOINLINE static Object* Stats_InterpreterNewClosure(int args_length, Object** args_object, Isolate* isolate) {                    
    RuntimeCallTimerScope timer(isolate, &RuntimeCallStats::Name);            
    TRACE_EVENT0(TRACE_DISABLED_BY_DEFAULT("v8.runtime"), "V8.Runtime_" InterpreterNewClosure);                                        
    Arguments args(args_length, args_object);                                 
    return __RT_impl_InterpreterNewClosure(args, isolate);                                   
  }                                                                           

  Object* Runtime_InterpreterNewClosure(int args_length, Object** args_object, Isolate* isolate) {        
    DCHECK(isolate->context() == nullptr || isolate->context()->IsContext()); 
    CLOBBER_DOUBLE_REGISTERS();                                               
    if (V8_UNLIKELY(FLAG_runtime_stats)) {                                    
      return Stats_Runtime_InterpreterNewClosure(args_length, args_object, isolate);                 
    }                                                                         
    Arguments args(args_length, args_object);                                 
    return __RT_impl_Runtime_InterpreterNewClosure(args, isolate);                                   
  }                                                                           

  static Object* __RT_impl_Runtime_InterpreterNewClosure(Arguments args, Isolate* isolate)
    HandleScope scope(isolate);
    DCHECK_EQ(4, args.length());
    CONVERT_ARG_HANDLE_CHECKED(SharedFunctionInfo, shared, 0);
    CONVERT_ARG_HANDLE_CHECKED(FeedbackVector, vector, 1);
    CONVERT_SMI_ARG_CHECKED(index, 2);
    CONVERT_SMI_ARG_CHECKED(pretenured_flag, 3);
    Handle<Context> context(isolate->context(), isolate);
    FeedbackSlot slot = FeedbackVector::ToSlot(index);
    Handle<Cell> vector_cell(Cell::cast(vector->Get(slot)), isolate);
    return *isolate->factory()->NewFunctionFromSharedFunctionInfo(
      shared, context, vector_cell,
      static_cast<PretenureFlag>(pretenured_flag));
  }
```
Notice that `Runtime_InterpreterNewClosure` is called.

NewFunctionFromSharedFunctionInfo will call NewFunction which will create a new function:
```c++
  function->initialize_properties();
  function->initialize_elements();
  function->set_shared(*info);
  function->set_code(info->code());
  function->set_context(*context_or_undefined);
  function->set_prototype_or_initial_map(*the_hole_value());
  function->set_feedback_vector_cell(*undefined_cell());
  isolate()->heap()->InitializeJSObjectBody(*function, *map, JSFunction::kSize);
  return function;
```
Notice the call `function->set_code(info->code())`. This is the assembly code for `InterpreterEntryTrampoline`.


value = CALL_GENERATED_CODE(isolate, stub_entry, orig_func, func, recv,
   147                                 argc, argv);
So we are back here. Value is of type Function and is the function for node_bootstrap.js. I'm confused as I thought that the above call
to `CALL_GENEREATED_CODE` would run the script. The actuall calling of the functions is done in :

 auto ret = f->Call(env->context(), Null(env->isolate()), 1, &arg);

(lldb) dis -f
node`v8::internal::Runtime_InterpreterNewClosure:

```c++
Handle<JSFunction> Factory::NewFunctionFromSharedFunctionInfo(
    Handle<Map> initial_map, Handle<SharedFunctionInfo> info,
    Handle<Object> context_or_undefined, Handle<Cell> vector,
    PretenureFlag pretenure) {
    
    Handle<JSFunction> result =
      NewFunction(initial_map, info, context_or_undefined, pretenure);
```
```console
(lldb) job *info
0x2e6ef602e159: [SharedFunctionInfo] in OldSpace
 - name = 0x2e6ea0e02441 <String[0]: >
 - kind = [ NormalFunction ]
 - function_map_index = 129
 - formal_parameter_count = 1
 - expected_nof_properties = 10
 - language_mode = strict
 - instance class name = #Object
 - code = 0x20819e1c4281 <Code BUILTIN>
 - bytecode_array = 0x2e6ef6030501
 - source code = (process) {
  let internalBinding;
  const exceptionHandlerState = { captureFn: null };

  function startup() {
    const EventEmitter = NativeModule.require('events');

    const origProcProto = Object.getPrototypeOf(process);
    Object.setPrototypeOf(origProcProto, EventEmitter.prototype);

    EventEmitter.call(process);

    setupProcessObject();
    ...

  startup();
}
 - anonymous expression
 - function token position = 300
 - start position = 308
 - end position = 21943
 - no debug info
 - length = 1
 - feedback_metadata = 0x2e6ef6030759: [FeedbackMetadata] in OldSpace
 - length: 15
 - slot_count: 83
```
So we can see this is indeed `bootstrap.js`

```c++
Handle<JSFunction> Factory::NewFunction(Handle<Map> map,
                                        Handle<SharedFunctionInfo> info,
                                        Handle<Object> context_or_undefined,
                                        PretenureFlag pretenure) {
  AllocationSpace space = pretenure == TENURED ? OLD_SPACE : NEW_SPACE;
  Handle<JSFunction> function = New<JSFunction>(map, space);
  DCHECK(context_or_undefined->IsContext() ||
         context_or_undefined->IsUndefined(isolate()));

  function->initialize_properties();
  function->initialize_elements();
  function->set_shared(*info);
  function->set_code(info->code());
  function->set_context(*context_or_undefined);
  function->set_prototype_or_initial_map(*the_hole_value());
  function->set_feedback_vector_cell(*undefined_cell());
  isolate()->heap()->InitializeJSObjectBody(*function, *map, JSFunction::kSize);
  return function;
}
```
Lets now take a look at `info->code()`
```console
(lldb) job info->code()
0x20819e1c4281: [Code]
kind = BUILTIN
name = InterpreterEntryTrampoline
compiler = unknown
Instructions (size = 1133)
0x20819e1c42e0     0  488b5f2f       REX.W movq rbx,[rdi+0x2f]
0x20819e1c42e4     4  488b5b07       REX.W movq rbx,[rbx+0x7]
0x20819e1c42e8     8  488b4b0f       REX.W movq rcx,[rbx+0xf]
0x20819e1c42ec     c  f6c101         testb rcx,0x1
0x20819e1c42ef     f  0f8512020000   jnz 0x20819e1c4507  (InterpreterEntryTrampoline)
0x20819e1c42f5    15  f6c101         testb rcx,0x1
0x20819e1c42f8    18  7410           jz 0x20819e1c430a  (InterpreterEntryTrampoline)
...
```

Setting through again and we will be back in:
```c++
value = CALL_GENERATED_CODE(isolate, stub_entry, orig_func, func, recv,
                           argc, argv);
```
```console
(lldb) job value
0x2e6e9e80ea49: [Function]
 - map = 0x2e6e8f082521 [FastProperties]
 - prototype = 0x2e6ef60043d1
 - elements = 0x2e6ea0e02251 <FixedArray[0]> [HOLEY_ELEMENTS]
 - initial_map =
 - shared_info = 0x2e6ef602e159 <SharedFunctionInfo>
 - name = 0x2e6ea0e02441 <String[0]: >
 - formal_parameter_count = 1
 - kind = [ NormalFunction ]
 - context = 0x2e6ef6003af9 <FixedArray[281]>
 - code = 0x20819e1c4281 <Code BUILTIN>
 - interpreted
 - bytecode = 0x2e6ef6030501
 - source code = (process) {
  let internalBinding;
  ...
}
 - properties = 0x2e6ea0e02251 <FixedArray[0]> {
    #length: 0x2e6ea9c034f9 <AccessorInfo> (const accessor descriptor)
    #name: 0x2e6ea9c03569 <AccessorInfo> (const accessor descriptor)
    #prototype: 0x2e6ea9c035d9 <AccessorInfo> (const accessor descriptor)
 }

 - feedback vector: 0x2e6ef60308f1: [FeedbackVector] in OldSpace
 - length: 83
 SharedFunctionInfo: 0x2e6ef602e159 <SharedFunctionInfo>
 Optimized Code: 0
 Invocation Count: 0
 Profiler Ticks: 0
...
```

This will then return to node.cc:
```c++
Local<Value> result = script.ToLocalChecked()->Run();
if (result.IsEmpty()) {
  ReportException(env, try_catch);
  exit(4);
}
```

So the generated code will be entered. When it gets around to processing:
```javascript
const EventEmitter = NativeModule.require('events');
```
This will result in another call to Invoke and the contents of func will be lib/event.js.


### Code
Is a class in `src/objects.h` which represents generated machine code.

```c++
DECL_PRINTER(Code)
```
This macro looks like this:
```c++
#ifdef OBJECT_PRINT
#define DECL_PRINTER(Name) void Name##Print(std::ostream& os);  // NOLINT
#else
#define DECL_PRINTER(Name)
#endif
```
So if OBJECT_PRINT is defined there will be a function named:
```c++
void CodePrint(std::ostream& os);
```

Anything that extends v8::internal::Object can also have a `Print` function:
```c++
(lldb) expr recv->Print()
0x343ceb998159: [JS_API_OBJECT_TYPE]
 - map = 0x343c184c7d01 [FastProperties]
 - prototype = 0x343cbcce6ee9
 - elements = 0x343cad402251 <FixedArray[0]> [HOLEY_ELEMENTS]
 - embedder fields: 1
 - properties = 0x343ceb99a599 <PropertyArray[3]> {
    #close: 0x343c78ce8bd1 <JSFunction JSStreamWrap.handle.close (sfi = 0x343c78ce83f1)> (const data descriptor)
    #isClosing: 0x343cbccf8271 <JSFunction isClosing (sfi = 0x343c78ca3a49)> (const data descriptor)
    #onreadstart: 0x343cbccf82b1 <JSFunction onreadstart (sfi = 0x343c78ca3af9)> (const data descriptor)
    #onreadstop: 0x343cbccf82f1 <JSFunction onreadstop (sfi = 0x343c78ca3ba9)> (const data descriptor)
    #onshutdown: 0x343cbccf8331 <JSFunction onshutdown (sfi = 0x343c78ca3c59)> (const data descriptor)
    #onwrite: 0x343cbccf8371 <JSFunction onwrite (sfi = 0x343c78ca3d09)> (const data descriptor)
    #owner: 0x343ceb998719 <Socket map = 0x343c184c7cb1> (data field 0) properties[0]
    #onread: 0x343cbccb7619 <JSFunction onread (sfi = 0x343cb163e541)> (const data descriptor)
    #reading: 0x343cad402381 <true> (data field 1) properties[1]
 }
 - embedder fields = {
    0x105d053f0
 }
```
### CHECK
These macros Can be found in src/util.h.

```c++
#define LIKELY(expr) __builtin_expect(!!(expr), 1)
#define UNLIKELY(expr) __builtin_expect(!!(expr), 0)

#define CHECK(expr)                                                           \
  do {                                                                        \
    if (UNLIKELY(!(expr))) {                                                  \
      static const char* const args[] = { __FILE__, STRINGIFY(__LINE__),      \
                                          #expr, PRETTY_FUNCTION_NAME };      \
      node::Assert(&args);                                                    \
    }                                                                         \
  } while (0)

#define CHECK_EQ(a, b) CHECK((a) == (b))
```
So take the following expression:
```c++
CHECK_EQ(false, try_catch.IsVerbose());
```
it would expand to:
```c++
if (__builtin_expect(!!(false == try_catch.IsVerbose()), 0)) {
  static const char* const args[] = { __FILE__, STRINGIFY(__LINE__), #expr, __PRETTY_FUNCTION_NAME__ };
  node::Assert(args);
}
```
`__builtin_expect` expects its parameters to be of type long and not bool, so there is a need to cast.
```console
(lldb) expr !!(false == try_catch.IsVerbose())
(bool) $122 = true 
```
This is already bool but the macro can be used with other types. 


OutputStackCheck();
`OutputStackCheck` is generated using a macro:
```c++
#define DEFINE_BYTECODE_OUTPUT(name, ...)                             \
  template <typename... Operands>                                     \
  BytecodeNode BytecodeArrayBuilder::Create##name##Node(              \
      Operands... operands) {                                         \
    return BytecodeNodeBuilder<Bytecode::k##name, __VA_ARGS__>::Make( \
        this, operands...);                                           \
  }                                                                   \
                                                                      \
  template <typename... Operands>                                     \
  void BytecodeArrayBuilder::Output##name(Operands... operands) {     \
    BytecodeNode node(Create##name##Node(operands...));               \
    Write(&node);                                                     \
  }                                                                   \
                                                                      \
  template <typename... Operands>                                     \
  void BytecodeArrayBuilder::Output##name(BytecodeLabel* label,       \
                                          Operands... operands) {     \
    DCHECK(Bytecodes::IsJump(Bytecode::k##name));                     \
    BytecodeNode node(Create##name##Node(operands...));               \
    WriteJump(&node, label);                                          \
    LeaveBasicBlock();                                                \
  }
BYTECODE_LIST(DEFINE_BYTECODE_OUTPUT)
#undef DEFINE_BYTECODE_OUTPUT
```
`BYTECODE_LIST` is a macro defined in `src/interpreter/bytecodes.h`:
```c++
// The list of bytecodes which are interpreted by the interpreter.
// Format is V(<bytecode>, <accumulator_use>, <operands>).
#define BYTECODE_LIST(V)
 ...
 V(StackCheck, AccumulatorUse::kNone)                                         \

```
```c++
  template <typename... Operands>                                     
  BytecodeNode BytecodeArrayBuilder::CreateStackCheckNode(Operands... operands) {
    return BytecodeNodeBuilder<Bytecode::kStackCheck, __VA_ARGS__>::Make(this, operands...);
  }                                                                   
                                                                      
  template <typename... Operands>                                     
  void BytecodeArrayBuilder::OutputStackCheck(Operands... operands) {     
    BytecodeNode node(CreateStackCheckNode(operands...));               
    Write(&node);                                                     
  }                                                                   
                                                                      
  template <typename... Operands>                                     
  void BytecodeArrayBuilder::OutputStackCheck(BytecodeLabel* label,       
                                          Operands... operands) {     
    DCHECK(Bytecodes::IsJump(Bytecode::kStackCheck));                     
    BytecodeNode node(CreateStackCheckNode(operands...));               
    WriteJump(&node, label);                                          
    LeaveBasicBlock();                                                
  }
```
Our call does not have any parameters so the first `OutputStackCheck` will be called in this case which will create
the BytecodeNode and then call `Write(&node)`
```c++
(lldb) p *node
(v8::internal::interpreter::BytecodeNode) $218 = {
  bytecode_ = kStackCheck
  operands_ = ([0] = 0, [1] = 0, [2] = 0, [3] = 0, [4] = 0)
  operand_count_ = 0
  operand_scale_ = kSingle
  source_info_ = (position_type_ = kExpression, source_position_ = 0)
}
```

compiler.cc:803
```c++
 Handle<SharedFunctionInfo> shared_info =
   801       isolate->factory()->NewSharedFunctionInfoForLiteral(parse_info->literal(),
   802                                                           parse_info->script());
```
```console
(lldb) job *shared_info
```

I'm not showing the whole output but this is the contents of node_bootstrap.js

interpreter.cc:211
```console
 Handle<BytecodeArray> bytecodes =
211       generator()->FinalizeBytecode(isolate(), parse_info()->script());


-> 225   compilation_info()->SetBytecodeArray(bytecodes);
   226   compilation_info()->SetCode(
   227       BUILTIN_CODE(compilation_info()->isolate(), InterpreterEntryTrampoline));
   228   return SUCCEEDED;
```

So `BUILTIN_CODE` will expand to:
```c++
isolate->builtins()->builtin_handle(Builtins::kInterpreterEntryTrampoline);
```
So that line will become:
```c++
compilation_info()->SetCode(isolate->builtins()->builtin_handle(Builtins::kInterpreterEntryTrampoline));
```
To recap, `builtins/builtins.h' has a macro that creates an enum entry for all of the builtins listed in 
`src/builtins/builtins-definitions.h`. And we are then calling builtin_handle with that enum value:
```c++
V8_EXPORT_PRIVATE Handle<Code> builtin_handle(int index);
```
So what does this function do?  
src/builtins/builtins.cc
```c++
Handle<Code> Builtins::builtin_handle(int index) {
  DCHECK(IsBuiltinId(index));
  return Handle<Code>(reinterpret_cast<Code**>(builtin_address(index)));
}
```

### How builtins get initialized

But how does the builtins_ array get?

`src/builtins/builtins.h`:
```c++
BUILTIN_LIST(IGNORE_BUILTIN, IGNORE_BUILTIN, DECLARE_TF, DECLARE_TF,
               DECLARE_TF, DECLARE_TF, DECLARE_ASM)
```
Where `DECLARE_ASM` is defined as:
```c++
#define DECLARE_ASM(Name, ...) \
  static void Generate_##Name(MacroAssembler* masm);
```
So there will be a function named Generate_InterpreterEntryTrampoline.

But we want to know where the builtins_ array is populated. Well, it 
```c++
Object* builtins_[builtin_count];
```
And `builtins_count` is a member of the Name enum:
```c++
  enum Name : int32_t {
#define DEF_ENUM(Name, ...) k##Name,
    BUILTIN_LIST_ALL(DEF_ENUM)
#undef DEF_ENUM
        builtin_count
  };
```
It seems like these entries will get populated when the snapshot is deserialized:
(snapshot/builtin-deserializer.cc)
```c++
builtins->set_builtin(i, DeserializeBuiltin(i));
```

Lets take a look at the stack when running generated code from V8. In src/globals.h we find:
```c++
typedef byte* Address;
...
#define FOR_EACH_ISOLATE_ADDRESS_NAME(C)                \
  C(Handler, handler)                                   \
  C(CEntryFP, c_entry_fp)                               \
  C(CFunction, c_function)                              \
  C(Context, context)                                   \
  C(PendingException, pending_exception)                \
  C(PendingHandlerContext, pending_handler_context)     \
  C(PendingHandlerCode, pending_handler_code)           \
  C(PendingHandlerOffset, pending_handler_offset)       \
  C(PendingHandlerFP, pending_handler_fp)               \
  C(PendingHandlerSP, pending_handler_sp)               \
  C(ExternalCaughtException, external_caught_exception) \
  C(JSEntrySP, js_entry_sp)

enum IsolateAddressId {
#define DECLARE_ENUM(CamelName, hacker_name) k##CamelName##Address,
  FOR_EACH_ISOLATE_ADDRESS_NAME(DECLARE_ENUM)
#undef DECLARE_ENUM
      kIsolateAddressCount
};
```
Will expand to:
```c++
enum IsolateAddressId {
  kHandlerAddress,
  ...
};
```
Notice that `hacker_name` parameter is not used in this call.
In isolate.cc when an Isolate is initialized by `bool Isolate::Init(StartupDeserializer* des)`:
```c++
#define ASSIGN_ELEMENT(CamelName, hacker_name)                  \
  isolate_addresses_[IsolateAddressId::k##CamelName##Address] = \
      reinterpret_cast<Address>(hacker_name##_address());
  FOR_EACH_ISOLATE_ADDRESS_NAME(ASSIGN_ELEMENT)
#undef ASSIGN_ELEMENT
```
```c++
isolate_addresses_[IsolateAddressId::HandlerAddress] = reinterpret_cast<Address>(handler_address());
```
`isolate_addresses_` can be found in isolate.h:
```c++
Address isolate_addresses_[kIsolateAddressCount + 1];
```
So where does `handler_address()` come from?  
This is defined in `isolate.h`:
```c++
inline Address* c_entry_fp_address() { return &thread_local_top_.c_entry_fp_; }
inline Address* handler_address() { return &thread_local_top_.handler_; }
inline Address* js_entry_sp_address() { return &thread_local_top_.js_entry_sp_; }
inline Address* c_function_address() { return &thread_local_top_.c_function_; }
Context** context_address() { return &thread_local_top_.context_; }
Address pending_message_obj_address() { return reinterpret_cast<Address>(&thread_local_top_.pending_message_obj_); }

THREAD_LOCAL_TOP_ADDRESS(Context*, pending_handler_context)
THREAD_LOCAL_TOP_ADDRESS(Code*, pending_handler_code)
THREAD_LOCAL_TOP_ADDRESS(intptr_t, pending_handler_offset)
THREAD_LOCAL_TOP_ADDRESS(Address, pending_handler_fp)
THREAD_LOCAL_TOP_ADDRESS(Address, pending_handler_sp)


#define THREAD_LOCAL_TOP_ADDRESS(type, name) \
  type* name##_address() { return &thread_local_top_.name##_; }

Context* pending_handler_context_address() { return &thread_local_top_.pending_handler_context_;
```

When is `Isolate::Init` called?  
It is called from node::Start

```console
Isolate* const isolate = Isolate::New(params);
```
This will invoke `return IsolateNewImpl(isolate, params);`


```console
(lldb) expr target->Print()
(lldb) expr v8::internal::JSFunction::cast(*target)
(v8::internal::JSFunction *) $1115 = 0x000017771f4b05f1
(lldb) expr v8::internal::JSFunction::cast(*target)->code()
(v8::internal::Code *) $1116 = 0x0000262dec344281

(lldb) job v8::internal::JSFunction::cast(*target)->code()
```
So that target functions byte code is of type InterpreterEntryTrampoline

### JSEntryStub
So stub_entry is from x64/code-stubs-x64.cc` by `void JSEntryStub::Generate`:
(lldb) p stub_entry
(JSEntryFunction) $1126 = 0x0000262dec284060

(lldb) job *code
0x262dec284001: [Code]
kind = STUB
major_key = JSEntryStub
compiler = unknown
Instructions (size = 232)
0x262dec284060     0  55             push rbp
0x262dec284061     1  4889e5         REX.W movq rbp,rsp
0x262dec284064     4  6a02           push 0x2

Notice that `0x0000262dec284060` from `stub_entry` matches the first instruction in code.

0x262dec2840ed    8d  49ba088c800501000000 REX.W movq r10,0x105808c08    ;; external reference (Isolate::handler_address)
0x262dec2840f7    97  498922         REX.W movq [r10],rsp
0x262dec2840fa    9a  e8c1d70b00     call 0x262dec3418c0  (JSEntryTrampoline)    ;; code: BUILTIN


(lldb) dis -s 0x262dec3418c0
    0x262dec3418c0: movq   %rdi, %r11
    0x262dec3418c3: movq   %rsi, %rdi
    0x262dec3418c6: xorl   %esi, %esi
    0x262dec3418c8: pushq  %rbp
    0x262dec3418c9: movq   %rsp, %rbp
    0x262dec3418cc: pushq  $0x1c
    0x262dec3418ce: movabsq $0x262dec341861, %r10     ; imm = 0x262DEC341861
    0x262dec3418d8: pushq  %r10

### Runtime_CompileLazy
There is a V8 builtin named CompileLazy which when called can compile the function being called (runtime/runtime-compiler.cc):
(See RUNTIME_FUNCTION for details regarding the RUNTIME_FUNCTION macro) 
```c++
RUNTIME_FUNCTION(Runtime_CompileLazy) {
  ...
  if (!Compiler::Compile(function, Compiler::KEEP_EXCEPTION)) {
    return isolate->heap()->exception();
  }
  DCHECK(function->is_compiled());
  return function->code();
```

First time, what is compiled is the entire node_bootstrap.js:
InterpreterCompilationJob::Status InterpreterCompilationJob::FinalizeJobImpl()
(deps/v8/src/interpreter/interpreter.cc)
```console
(lldb) br s -f interpreter.cc -l 201
```
```c++
  Handle<BytecodeArray> bytecodes = generator()->FinalizeBytecode(isolate(), parse_info()->script());
```
```console
(lldb) job *bytecodes
0x18d63b2dfe1: [BytecodeArray] in OldSpaceParameter count 1
Frame size 8
    0 E> 0x18d63b2e01a @    0 : 93                StackCheck
  299 S> 0x18d63b2e01b @    1 : 6f 00 00 00       CreateClosure [0], [0], #0
         0x18d63b2e01f @    5 : 1e fb             Star r0
21946 S> 0x18d63b2e021 @    7 : 97                Return
Constant pool (size = 1)
0x18d63b2dfc9: [FixedArray] in OldSpace
 - map = 0x18ddb3022f1 <Map(HOLEY_ELEMENTS)>
 - length: 1
           0: 0x18d63b2df19 <SharedFunctionInfo>
Handler Table (size = 16)
```
Notice that the byte code for this is pretty short. If you look at node_bootstrap.js you see that it is a function. I think
this is why the CreateClosure operator is for. 
```console
(lldb) job  *shared_info
0x1606df4adc91: [SharedFunctionInfo] in OldSpace
 - name = 0x160691002441 <String[0]: >
 - kind = [ NormalFunction ]
 - function_map_index = 129
 - formal_parameter_count = 0
 - expected_nof_properties = 10
 - language_mode = strict
 - instance class name = #Object
 - code = 0x2066fa144281 <Code BUILTIN>
 - bytecode_array = 0x1606df4adfe1
 - source code = // Hello, and welcome to hacking node.js!
//
// This file is invoked by node::LoadEnvironment in src/node.cc, and is
// responsible for bootstrapping the node.js core. As special caution is given
// to the performance of the startup process, many dependencies are invoked
// lazily.
```
Notice that `source code` contains the entire content of bootstrap_node.js.

If you later look at `0x18d63b2df19` (from bytecodes above) we can see that this is the inner function:
```console
(lldb) job  *inner_shared_info
0x1606df4adf19: [SharedFunctionInfo] in OldSpace
 - name = 0x160691002441 <String[0]: >
 - kind = [ NormalFunction ]
 - function_map_index = 129
 - formal_parameter_count = 1
 - expected_nof_properties = 10
 - language_mode = strict
 - instance class name = #Object
 - code = 0x2066fa1450e1 <Code BUILTIN>
 - source code = (process) {
  let internalBinding;
  const exceptionHandlerState = { captureFn: null };

  function startup() {
```
And notice that this in now the anonymous function that takes the process object. This is what will be 
called later.

Next, we have:
```c++
compilation_info()->SetBytecodeArray(bytecodes);
compilation_info()->SetCode(BUILTIN_CODE(compilation_info()->isolate(), InterpreterEntryTrampoline));
```

InstallUnoptimizedCode will later use this Code and set that as on the SharedFunctionInfo:
```c++
shared->set_code(*compilation_info->code());
```

The inner_shared_info will also be compiled by `FinalizeUnoptimizedCode` and 
the bytecode for it will much longer as expected:
```console
(lldb) job *bytecodes
```
Next this is compiled by `FinalizeJobImpl`:
```c++
compilation_info()->SetBytecodeArray(bytecodes);
compilation_info()->SetCode(BUILTIN_CODE(compilation_info()->isolate(), InterpreterEntryTrampoline));
```
Again notice that the code is set to `InterpreterEntryTrampoline`.

After having done the function will return success.

This will now return and we will end up back in node.cc:
```c++
Local<Value> result = script.ToLocalChecked()->Run();
```
This will land us in `v8::Script::Run` and the following line:
```c++
auto fun = i::Handle<i::JSFunction>::cast(Utils::OpenHandle(this));
```
Now if we print this JSFunction using `job *fun` we can see the complete object including the source. 
We can also take a look at the code and the bytecode:
```console
(lldb) job  fun->abstract_code()
0x1606df4adfe1: [BytecodeArray] in OldSpaceParameter count 1
Frame size 8
    0 E> 0x1606df4ae01a @    0 : 93                StackCheck
  299 S> 0x1606df4ae01b @    1 : 6f 00 00 00       CreateClosure [0], [0], #0
         0x1606df4ae01f @    5 : 1e fb             Star r0
21946 S> 0x1606df4ae021 @    7 : 97                Return
Constant pool (size = 1)
0x1606df4adfc9: [FixedArray] in OldSpace
 - map = 0x1606023022f1 <Map(HOLEY_ELEMENTS)>
 - length: 1
           0: 0x1606df4adf19 <SharedFunctionInfo>
Handler Table (size = 16)
```
And a snipped of the code:
```console
(lldb) job  fun->code()
0x2066fa144281: [Code]
kind = BUILTIN
name = InterpreterEntryTrampoline
compiler = unknown
Instructions (size = 1004)
0x2066fa1442e0     0  488b5f2f       REX.W movq rbx,[rdi+0x2f]
0x2066fa1442e4     4  488b5b07       REX.W movq rbx,[rbx+0x7]
0x2066fa1442e8     8  488b4b0f       REX.W movq rcx,[rbx+0xf]
0x2066fa1442ec     c  f6c101         testb rcx,0x1
0x2066fa1442ef     f  0f8591010000   jnz 0x2066fa144486  (InterpreterEntryTrampoline)
0x2066fa1442f5    15  f6c101         testb rcx,0x1
```
Once again we will be back in `Invoke` and this following line:
```c++
Handle<Code> code = is_construct ? isolate->factory()->js_construct_entry_code() : isolate->factory()->js_entry_code();
```
In this case `is_construct` is false so Handle\<Code\> will be what ever `isolate->factory()->js_entry_code()` returns. And it
returns:
```console
(lldb) job *code
0x2066fa084001: [Code]
kind = STUB
major_key = JSEntryStub
```
Lets take a closer look at the call `isolate->factory()->js_entry_code()`. In src/factory.h we can find:
```c++
#define ROOT_ACCESSOR(type, name, camel_name) inline Handle<type> name();
  ROOT_LIST(ROOT_ACCESSOR)
#undef ROOT_ACCESSOR
```
And we can find `ROOT_LIST` in `src/heap/heap.h`:
```c++
#define ROOT_LIST(V)  \
  STRONG_ROOT_LIST(V) \
  SMI_ROOT_LIST(V)    \
  V(StringTable, string_table, StringTable)
```
`STRONG_ROOT_LIST(V)` contains (among others):
```c++
  /* JS Entries */                                                             \
  V(Code, js_entry_code, JsEntryCode)                                          \
  V(Code, js_construct_entry_code, JsConstructEntryCode)
```
And in `factory-inl.h` we have:
```c++
#define ROOT_ACCESSOR(type, name, camel_name)                         \
  Handle<type> Factory::name() {                                      \
    return Handle<type>(bit_cast<type**>(                             \
        &isolate()->heap()->roots_[Heap::k##camel_name##RootIndex])); \
  }
ROOT_LIST(ROOT_ACCESSOR)
#undef ROOT_ACCESSOR
```
So, for `js_entry_code` the following would be expanded by the preprocessor:
```c++
  Handle<JsEntryCode> Factory::js_entry_code() {  
    return Handle<JsEntryCode>(bit_cast<JsEntryCode**>(&isolate()->heap()->roots_[Heap::kJsEntryCodeRootIndex]));
  }
```

Ok so back to the `Invoke` function:
```c++
JSEntryFunction stub_entry = FUNCTION_CAST<JSEntryFunction>(code->entry());
...
value = CALL_GENERATED_CODE(isolate, stub_entry, orig_func, func, recv, argc, argv);
```
Now, we know that `stub_entry` is the `JSEntryStub` (src/x64/code-stubs-x64.cc) and `func` is the `InterpreterEntryTrampoline` (src/builtins/x64/builtins-x64.cc). 

```console
->  0xd099dd84060: pushq  %rbp                      // function prologue
    0xd099dd84061: movq   %rsp, %rbp                // function prologue                                                                  Stack
    0xd099dd84064: pushq  $0x2                      // the stack frame marker type                                                        2 (Marker Type ConstructEntryFrame)
    0xd099dd84066: movabsq $0x105808b90, %r10       // mov the value of the current context address
    0xd099dd84070: movq   (%r10), %r10              // dereference so the address is in r10
    0xd099dd84073: pushq  %r10                      // push the context address onto the stack                                            address to current context
    0xd099dd84075: pushq  %r12                      // store callee saved registers which need to be preserved                            whatever was in r12
    0xd099dd84077: pushq  %r13                      // as above                                                                           whatever was in r13
    0xd099dd84079: pushq  %r14                      // as above                                                                           whatever was in r14
    0xd099dd8407b: pushq  %r15                      // as above                                                                           whatever was in r15
    0xd099dd8407d: pushq  %rbx                      // as above                                                                           whatever was in rbx
    0xd099dd8407e: movabsq $0x105807248, %r13       // InitializeRootRegister (x64/macro-assembler-x64.h)
    0xd099dd84088: addq   $0x80, %r13               // also part of InitializeRootRegister
    0x16e271f840f: movabsq $0x105808c00, %r10       // mov the frame pointer address into r10
    0x16e271f8409: pushq  (%r10)                    // dereferense so that the address on the stack                                       address to frame pointer descriptor 
    0x16e271f840C: movabsq 0x105808c20, %rax        // mov the js_entry_sp (stack pointer) into rax
    0x16e271f840a6: testq  %rax, %rax               // test 
    0x16e271f840a9: jne    0x16e271f840c3           // and jump if zero flag is 0 (rax is not all zeros) which is not the case
    0xd099dd840af: pushq  $0x2                      // StackFrame::OUTERMOST_JSENTRY_FRAME))
    0xd099dd840b1: movq   %rbp, %rax                // move the current frame base pointer to rax
    0xd099dd840b4: movabsq %rax, 0x105808c20        // move the base frame pointer to memory location 
    0xd099dd840be: jmp    0xd099dd840c5             // unconditional jump (jmp &cont)
    0xd099dd840c5: jmp    0xd099dd840e0             // unconditional jump (jmp &invok)
    0xd099dd840e0: movabsq $0x105808c08, %r10       // move handler_address to r10 (PushStackHandler src/x64/macro-assembler-x64.cc)
    0xd099dd840ea: pushq  (%r10)                    // push the dereferenced address of handler_address onto the stack                    address to handler
    0xd099dd840ed: movabsq $0x105808c08, %r10       // move the address again. not sure why this is done again through
    0xd099dd840f7: movq   %rsp, (%r10)              // set the stack pointer to be that of the handler. This will be the I
    0xd099dd840fa: callq  0xd099de418c0             // __ Call(BUILTIN_CODE(isolate(), JSEntryTrampoline), RelocInfo::CODE_TARGET);

    `Generate_JSEntryTrampolineHelper(MacroAssembler* masm, bool is_construct)` in `src/builtins/x64/builtins-x64.cc`:
    0x16e2720418c0: movq   %rdi, %r11               // mov new_target into r11 
    0x16e2720418c3: movq   %rsi, %rdi               // move func into rdi
    0x16e2720418c6: xorl   %esi, %esi               // __ Set(rsi, 0);
    0x16e2720418c8: pushq  %rbp                     // function prologue                                                                  previous base frame pointer
    0x16e2720418c9: movq   %rsp, %rbp               // function prologue
    0x16e2720418cc: pushq  $0x1c                    // the stack frame marker type                                                        1c (28)
    0x16e2720418ce: movabsq $0x16e272041861, %r10   // move the CodeObject address into r10??
    0x16e2720418d8: pushq  %r10                     // push the CodeObject onto the stack                                                 code object
    0x16e2720418da: movabsq $0x37b3a2c022e1, %r10   // emit_debug_code
    0x16e2720418e4: cmpq   %r10, (%rsp)             // emit_debug_code
    0x16e2720418e8: jne    0x16e2720418fa           // emit_debug_code
    0x16e2720418fa: movabsq $0x105808b90, %r10
    0x16e272041904: movq   (%r10), %rsi
    0x16e272041907: pushq  %rdi                                                                                                           function
    0x16e272041908: pushq  %rdx                                                                                                           receiver
    0x16e272041909: movq   %rcx, %rax
    0x16e27204190c: movq   %r8, %rbx
    0x16e27204190f: movq   %r11, %rdx
    Generate_CheckStackOverflow(masm, kRaxIsUntaggedInt):
    0x16e272041912: movq   0xd08(%r13), %r10         // LoadRoot (x64/macro-assembler-x64.cc) (index << kPointerSizeLog2) - kRootRegisterBias))
    0x16e272041919: movq   %rsp, %rcx                // 
    0x16e27204191c: subq   %r10, %rcx
    0x16e27204191f: movq   %rax, %r11
    0x16e272041922: shlq   $0x3, %r11
    0x16e272041926: cmpq   %r11, %rcx
    0x16e272041929: jg     0x16e272041940
    0x16e272041940: xorl   %ecx, %ecx                // __ Set(rcx, 0);  Start loop to copy args onto the stack
    0x16e272041942: jmp    0x16e27204194f
    0x16e27204194f: cmpq   %rax, %rcx                // __ cmpp(rcx, rax);
    0x16e272041954: callq  0x16e27203c400            // __ Call(masm->isolate()->builtins()->Call(), RelocInfo::CODE_TARGET)
    
    
The last `Call` I think will land in (x64/macro-assembler-x64.cc line 2003):
```c++
void TurboAssembler::Call(Handle<Code> code_object, RelocInfo::Mode rmode) {
  call(code_object, rmode);
#endif
}
```
`call` is declared in `src/x64/assembler-x64.h`:
```c++
void call(Handle<Code> target, RelocInfo::Mode rmode = RelocInfo::CODE_TARGET);
```
The definition can be found in `src/x64/assembler-x64.cc`:
```c++
void Assembler::call(Address entry, RelocInfo::Mode rmode) {
  DCHECK(RelocInfo::IsRuntimeEntry(rmode));
  EnsureSpace ensure_space(this);
  // 1110 1000 #32-bit disp.
  emit(0xE8);
  emit_runtime_entry(entry, rmode);
}
```
If we look again at the callq:
```console
    0x16e272041954: callq  0x16e27203c400
```
And display the opcode for callq:
```console
(lldb) dis -f -b
->  0x16e272041954: e8 a7 aa ff ff                 callq  0x16e27203c400
```
We can see match the opcode emitted by `emit(0xE8)` to `e8.
But what is returned by the call to `masm->isolate()-builtins()->call()`?  
In src/isolate.h we have:
```c++
Builtins* builtins() { return &builtins_; }
```
If I back up 2 frames and try to execute isolate()->builtins()->Call() I get:
```console
(lldb) job *isolate->builtins()->Call(static_cast<ConvertReceiverMode>(ConvertReceiverMode::kAny))
```
In `src/builtins/builtins.h` we can find the `Call` function with has a default value for the mode
parameter:
```c++
Handle<Code> Call(ConvertReceiverMode = ConvertReceiverMode::kAny);
```
If we take a look in `src/builtins/builtins-call.cc` we find the definition of `Builtins::Call`:
```c++
Handle<Code> Builtins::Call(ConvertReceiverMode mode) {
  switch (mode) {
    case ConvertReceiverMode::kNullOrUndefined:
      return builtin_handle(kCall_ReceiverIsNullOrUndefined);
    case ConvertReceiverMode::kNotNullOrUndefined:
      return builtin_handle(kCall_ReceiverIsNotNullOrUndefined);
    case ConvertReceiverMode::kAny:
      return builtin_handle(kCall_ReceiverIsAny);
  }
  UNREACHABLE();
}
```
And if we look in src/builtins/builtins-definitions.h we can find Call_ReceiverIsAny`:
```c++
ASM(Call_ReceiverIsAny)
```
And we should be able to use `ConvertReceiverMode::kAny` with the `Call` function to find out
what is going to be called (the code_object in `call(code_object, rmode);`:
```console
(lldb) job *isolate->builtins()->Call(static_cast<ConvertReceiverMode>(ConvertReceiverMode::kAny))
0x16e27203c3a1: [Code]
kind = BUILTIN
name = Call_ReceiverIsAny
compiler = unknown
Instructions (size = 178)
0x16e27203c400     0  40f6c701       testb rdi,0x1
0x16e27203c404     4  0f8446000000   jz 0x16e27203c450  (Call_ReceiverIsAny)
0x16e27203c40a     a  488b4fff       REX.W movq rcx,[rdi-0x1]
0x16e27203c40e     e  80790bff       cmpb [rcx+0xb],0xff
0x16e27203c412    12  0f8408faffff   jz 0x16e27203be20  (CallFunction_ReceiverIsAny)    ;; code: BUILTIN
0x16e27203c418    18  80790bfe       cmpb [rcx+0xb],0xfe
0x16e27203c41c    1c  0f845efcffff   jz 0x16e27203c080  (CallBoundFunction)    ;; code: BUILTIN
0x16e27203c422    22  f6410c02       testb [rcx+0xc],0x2
0x16e27203c426    26  0f8424000000   jz 0x16e27203c450  (Call_ReceiverIsAny)
0x16e27203c42c    2c  80790bb7       cmpb [rcx+0xb],0xb7
0x16e27203c430    30  0f8505000000   jnz 0x16e27203c43b  (Call_ReceiverIsAny)
0x16e27203c436    36  e9e5000000     jmp 0x16e27203c520  (CallProxy)    ;; code: BUILTIN
0x16e27203c43b    3b  48897cc408     REX.W movq [rsp+rax*8+0x8],rdi
0x16e27203c440    40  488b7e27       REX.W movq rdi,[rsi+0x27]
0x16e27203c444    44  488bbfff000000 REX.W movq rdi,[rdi+0xff]
0x16e27203c44b    4b  e970f7ffff     jmp 0x16e27203bbc0  (CallFunction_ReceiverIsNotNullOrUndefined)    ;; code: BUILTIN
0x16e27203c450    50  55             push rbp
0x16e27203c451    51  4889e5         REX.W movq rbp,rsp
0x16e27203c454    54  6a1c           push 0x1c
0x16e27203c456    56  49baa1c30372e2160000 REX.W movq r10,0x16e27203c3a1  (Call_ReceiverIsAny)    ;; object: 0x16e27203c3a1 <Code BUILTIN>
0x16e27203c460    60  4152           push r10
0x16e27203c462    62  49bae122c0a2b3370000 REX.W movq r10,0x37b3a2c022e1    ;; object: 0x37b3a2c022e1 <undefined>
0x16e27203c46c    6c  4c391424       REX.W cmpq [rsp],r10
0x16e27203c470    70  7510           jnz 0x16e27203c482  (Call_ReceiverIsAny)
0x16e27203c472    72  48ba0000000009000000 REX.W movq rdx,0x900000000
0x16e27203c47c    7c  e83f950200     call 0x16e2720659c0  (Abort)    ;; code: BUILTIN
0x16e27203c481    81  cc             int3l
0x16e27203c482    82  57             push rdi
0x16e27203c483    83  b801000000     movl rax,0x1
0x16e27203c488    88  48bb9002430101000000 REX.W movq rbx,0x101430290    ;; external reference (Runtime::ThrowCalledNonCallable)
0x16e27203c492    92  e8a982f4ff     call 0x16e271f84740     ;; code: STUB, CEntryStub, minor: 8
0x16e27203c497    97  48837df81c     REX.W cmpq [rbp-0x8],0x1c
0x16e27203c49c    9c  7410           jz 0x16e27203c4ae  (Call_ReceiverIsAny)
0x16e27203c49e    9e  48ba000000004f000000 REX.W movq rdx,0x4f00000000
0x16e27203c4a8    a8  e813950200     call 0x16e2720659c0  (Abort)    ;; code: BUILTIN
0x16e27203c4ad    ad  cc             int3l
0x16e27203c4ae    ae  488be5         REX.W movq rsp,rbp
0x16e27203c4b1    b1  5d             pop rbp


RelocInfo (size = 11)
0x16e27203c414  code target (BUILTIN)  (0x16e27203be20)
0x16e27203c41e  code target (BUILTIN)  (0x16e27203c080)
0x16e27203c437  code target (BUILTIN)  (0x16e27203c520)
0x16e27203c44c  code target (BUILTIN)  (0x16e27203bbc0)
0x16e27203c458  embedded object  (0x16e27203c3a1 <Code BUILTIN>)
0x16e27203c464  embedded object  (0x37b3a2c022e1 <undefined>)
0x16e27203c47d  code target (BUILTIN)  (0x16e2720659c0)
0x16e27203c48a  external reference (Runtime::ThrowCalledNonCallable)  (0x101430290)
0x16e27203c493  code target (STUB)  (0x16e271f84740)
0x16e27203c4a9  code target (BUILTIN)  (0x16e2720659c0)
```

To find out what generated this code we can look in `src/builtins/builtins-call-gen.cc`: 
```c++
void Builtins::Generate_Call_ReceiverIsAny(MacroAssembler* masm) {
  Generate_Call(masm, ConvertReceiverMode::kAny);
}
```
And an implementation can be found in `src/builtins/x64/builtins-x64.cc`:
```c++
void Builtins::Generate_Call(MacroAssembler* masm, ConvertReceiverMode mode) {
}
```
Now, the actual produced assembly code looks like this:
(lldb) register read rax rdi
     rax = 0x0000000000000000                 // number of args
     rdi = 0x000037b34d8b06d9                 // the target to call

```console
0x16e27203c400: testb  $0x1, %dil            // __ JumpIfSmi(rdi, &non_callable); which consists of CheckSmi. dil is the low 8 bits of rdi
0x16e27203c404: je     0x16e27203c450        // __ JumpIfSmi(rdi, &non_callable: which after CheckSmi will jump. rdi is the target and not a smi in our case
0x16e27203c40a: movq   -0x1(%rdi), %rcx      // __ CmpObjectType(rdi, JS_FUNCTION_TYPE, rcx); which does a movp
0x16e27203c40e: cmpb   $-0x1, 0xb(%rcx)      // __ CmpObjectType(rdi, JS_FUNCTION_TYPE, rcx); which does a cmpb
0x16e27203c412: je     0x16e27203be20        // __ j(equal, masm->isolate()->builtins()->CallFunction(mode), RelocInfo::CODE_TARGET);
                                             // I was not sure where to find this `j` function but it is in src/x64/assembler-x64.cc
                                             // so what are we jumping to? We can back up in the debugger and find out:
                                             // (lldb) up 2
                                             // (lldb job *isolate->builtins()->CallFunction(static_cast<ConvertReceiverMode>(ConvertReceiverMode::kAny))
                                             // This will be a builtin named CallFunction_ReceiverIsAny, and being a builtin you'll find it in 
                                             // `src/builtins/builtins-definitions.h`. The implementation will be in `src/builtins/builtins-call.cc` and
                                             // will result in `return builtin_handle(kCallFunction_ReceiverIsAny)`. This code repsonsible for generating
                                             // is Builtins::Generate_CallFunction and in our case that means `src/builtins/x64/builtins-x64.cc`

Builtins::Generate_Call(MacroAssembler* masm, ConvertReceiverMode mode) (src/builtins/x64/builtins-x64.cc)
0x16e27203be20: testb  $0x1, %dil            // __ JumpIfSmi(rdi, &non_callable);
0x16e27203be24: jne    0x16e27203be36        // __ JumpIfSmi(rdi, &non_callable);

0x16e27203be36: pushq  %rdi                  // __ Push(rdi);
__ CallRuntime(Runtime::kThrowCalledNonCallable); in `src/x64/macro-assembler-x64.h` and 


...
0x16e27203be63: movq   0x27(%rdi), %rsi      //__ movp(rsi, FieldOperand(rdi, JSFunction::kContextOffset));
0x16e27203be67: testb  $0x3, 0x87(%rdx)      // __ testl(FieldOperand(rdx, SharedFunctionInfo::kCompilerHintsOffset), Immediate(SharedFunctionInfo::IsNativeBit::kMask | SharedFunctionInfo::IsStrictBit::kMask));
0x16e27203be6e: jne    0x16e27203bf14        // __ j(not_zero, &done_convert);
                                             // rax = nr args, rdx = SharedFunctionInfo, rdi = target function, rsi = the function context
0x16e27203bf14: movslq 0x73(%rdx), %rbx      // __ movsxlq(rbx, FieldOperand(rdx, SharedFunctionInfo::kFormalParameterCountOffset));
0x16e27203bf18: movabsq $0x1050057f2, %r10   
0x16e27203bf18: movabsq $0x1050057f2, %r10   // __ InvokeFunctionCode(rdi, no_reg, expected, actual, JUMP_FUNCTION);
0x16e27203bfa4: movq   -0x60(%r13), %rdx     // LoadRoot(rdx, Heap::kUndefinedValueRootIndex);
0x16e27203bfa8: cmpq   %rax, %rbx            // cmpp(expected.reg(), actual.reg());
0x16e27203bfb2: movq   0x37(%rdi), %rcx      // movp(rcx, FieldOperand(function, JSFunction::kCodeOffset));
0x16e27203bfb6: addq   $0x5f, %rcx           // addp(rcx, Immediate(Code::kHeaderSize - kHeapObjectTag));
0x16e27203bfba: jmpq   *%rcx                 // jmp(rcx); 

```
Once again I got lost :( 

I do know that if I set through I'll end up in `RUNTIME_FUNCTION(Runtime_InterpreterNewClosure` in `runtime/runtime-interpreter.cc` which has this line:
```c++
 return *isolate->factory()->NewFunctionFromSharedFunctionInfo(shared, context, vector_cell,static_cast<PretenureFlag>(pretenured_flag));
```
This call will delegate to another `NewFunctionFromSharedFunctionInfo` and:
```c++
Handle<JSFunction> result = NewFunction(initial_map, info, context_or_undefined, pretenure);
```
`NewFunction` will
```c++
Handle<JSFunction> function = New<JSFunction>(map, space);
function->initialize_properties();
function->initialize_elements();
function->set_shared(*info);
function->set_code(info->code());
function->set_context(*context_or_undefined);
function->set_prototype_or_initial_map(*the_hole_value());
function->set_feedback_vector_cell(*undefined_cell());
```
`code` will be the `InterpreterEntryTrampoline` and it looks like it was already been compiled:
```console
(lldb) expr info->is_compiled()
(bool) $483 = true
```

This Handle<JSFunction> is then returned `RUNTIME_FUNCTION(Runtime_InterpreterNewClosure`:
```console
    0x101437967 <+263>: movq   %rax, -0x8(%rbp)                               // store the value return value Handle<JSFunction> on the stack
    0x10143796b <+267>: movq   -0x8(%rbp), %rax                               // move it into rax which is the register for return value
    0x10143796f <+271>: addq   $0x50, %rsp                                    // clean up local stack variables
    0x101437973 <+275>: popq   %rbp                                           // pop the previous functions base frame pointer
    0x101437974 <+276>: retq                                                  // transfer control to the return address on the stack (placed there by call)
    0x101437975 <+277>: nopw   %cs:(%rax,%rax)

    (after retq):
    0x16e271f847a4: cmpq   0x88(%r13), %rax
    0x16e271f847ab: je     0x16e271f847f8

    0x16e271f847b1: movq   -0x58(%r13), %r14
    0x16e271f847b5: movabsq $0x105808ba0, %r10
    0x16e271f847bf: cmpq   (%r10), %r14
    0x16e271f847c2: je     0x16e271f847c5

    0x16e271f847c5: movq   0x8(%rbp), %rcx
    0x16e271f847c9: movq   (%rbp), %rbp
    0x16e271f847cd: leaq   0x8(%r15), %rsp
    0x16e271f847d1: pushq  %rcx
    0x16e271f847d2: movabsq $0x105808b90, %r10
    0x16e271f847dc: movq   (%r10), %rsi
    0x16e271f847df: movq   $0x0, (%r10)
    0x16e271f847e6: movabsq $0x105808c00, %r10        ; imm = 0x105808C00
    0x16e271f847f0: movq   $0x0, (%r10)
    0x16e271f847f7: retq

    0x16e271fd6cfd: movq   %rax, %rdx
    0x16e271fd6d00: movq   %rax, -0x28(%rbp)
    0x16e271fd6d04: movq   %rsp, %rax
    0x16e271fd6d07: cmpq   -0x38(%rbp), %rax
    0x16e271fd6d0b: je     0x16e271fd6d42

    0x16e271fd6d42: movq   -0x20(%rbp), %r12
    0x16e271fd6d46: addq   $0x4, %r12
    0x16e271fd6d4a: movq   -0x18(%rbp), %rdx
    0x16e271fd6d4e: movq   -0x18(%rdx), %r14
    0x16e271fd6d52: movzbl (%r12,%r14), %eax
    0x16e271fd6d57: movabsq $0x100000000, %r10        ; imm = 0x100000000
    0x16e271fd6d61: cmpq   %rax, %r10
    0x16e271fd6d64: jae    0x16e271fd6d76

    0x16e271fd6d76: movq   -0x10(%rbp), %r15
    0x16e271fd6d7a: movq   (%r15,%rax,8), %rbx
    0x16e271fd6d7e: movq   (%rbp), %rbp
    0x16e271fd6d82: movq   0x38(%rsp), %rax

    0x16e271fd6d87: addq   $0x68, %rsp
    0x16e271fd6d8b: jmpq   *%rbx

    0x16e271fbdde0: movsbq 0x1(%r14,%r12), %rbx
    0x16e271fbdde6: movq   %rbp, %rdx
    0x16e271fbdde9: movq   %rax, (%rdx,%rbx,8)
    0x16e271fbdded: addq   $0x2, %r12
    0x16e271fbddf1: movzbl (%r12,%r14), %ebx
    0x16e271fbddf6: movabsq $0x100000000, %r10        ; imm = 0x100000000
    0x16e271fbde00: cmpq   %rbx, %r10
    0x16e271fbde03: jae    0x16e271fbde15

    0x16e271fbde15: movq   (%r15,%rbx,8), %rbx
    0x16e271fbde19: jmpq   *%rbx
 
    0x16e271fdca60: movq   %rbp, %rbx
    0x16e271fdca63: movl   $0x0, -0x20(%rbx)
    0x16e271fdca6a: movl   %r12d, -0x1c(%rbx)
    0x16e271fdca6e: movl   %r12d, %edx
    0x16e271fdca71: movabsq $0x100000000, %r10        ; imm = 0x100000000
    0x16e271fdca7b: cmpq   %rdx, %r10
    0x16e271fdca7e: jae    0x16e271fdca90

    0x16e271fdca90: subl   $0x39, %edx
    0x16e271fdca93: movabsq $0x100000000, %r10        ; imm = 0x100000000
    0x16e271fdca9d: cmpq   %rdx, %r10
    0x16e271fdcaa0: jae    0x16e271fdcab2
    
    0x16e271fdcab2: cmpl   $0x0, %edx
    0x16e271fdcab5: jl     0x16e271fdcb0b
    0x16e271fdcab7: movl   0x33(%r14), %ecx
    0x16e271fdcabb: movabsq $0x100000000, %r10        ; imm = 0x100000000
    0x16e271fdcac5: cmpq   %rcx, %r10
    0x16e271fdcac8: jae    0x16e271fdcada

    0x16e271fdcada: subl   $0x1, %ecx
    0x16e271fdcadd: movabsq $0x100000000, %r10        ; imm = 0x100000000
    0x16e271fdcae7: cmpq   %rcx, %r10
    0x16e271fdcaea: jae    0x16e271fdcafc

    0x16e271fdcafc: subl   %edx, %ecx
    0x16e271fdcafe: cmpl   $0x0, %ecx
    0x16e271fdcb01: jl     0x16e271fdcb1c

    0x16e271fdcb03: movq   -0x18(%rbx), %rbx
    0x16e271fdcb07: movl   %ecx, 0x33(%rbx)
    0x16e271fdcb0a: retq

    0x16e272044654: movq   -0x18(%rbp), %r14
    0x16e272044658: movq   -0x20(%rbp), %r12
    0x16e27204465c: shrq   $0x20, %r12
    0x16e272044660: movzbl (%r14,%r12), %ebx
    0x16e272044665: cmpb   $-0x69, %bl
    0x16e272044668: je     0x16e2720446a3

    0x16e2720446a3: movq   -0x18(%rbp), %rbx
    0x16e2720446a7: movl   0x2b(%rbx), %ebx
    0x16e2720446aa: leave                              // will leave the function 

    0x16e2720446ab: popq   %rcx
    0x16e2720446ac: addq   %rbx, %rsp
    0x16e2720446af: pushq  %rcx
    0x16e2720446b0: retq

    0x16e272041959: cmpq   $0x1c, -0x8(%rbp)
    0x16e27204195e: je     0x16e272041970

    0x16e272041970: movq   %rbp, %rsp
    0x16e272041973: popq   %rbp
    0x16e272041974: retq

    0x16e271f840ff: movabsq $0x105808c08, %r10        ; imm = 0x105808C08
    0x16e271f84109: popq   (%r10)
    0x16e271f8410c: addq   $0x0, %rsp
    0x16e271f84110: popq   %rbx
    0x16e271f84111: cmpq   $0x2, %rbx
    0x16e271f84115: jne    0x16e271f8412c
    0x16e271f8411b: movabsq $0x105808c20, %r10        ; imm = 0x105808C20
    0x16e271f84125: movq   $0x0, (%r10)
    0x16e271f8412c: movabsq $0x105808c00, %r10        ; imm = 0x105808C00
    0x16e271f84136: popq   (%r10)
    0x16e271f84139: popq   %rbx                       // pop callee saved registers
    0x16e271f8413a: popq   %r15
    0x16e271f8413c: popq   %r14
    0x16e271f8413c: popq   %r14
    0x16e271f8413e: popq   %r13
    0x16e271f84140: popq   %r12
    0x16e271f84142: addq   $0x10, %rsp
    0x16e271f84146: popq   %rbp
    0x16e271f84147: retq

    value = CALL_GENERATED_CODE(isolate, stub_entry, orig_func, func, recv, argc, argv);
```
`value` is the function of type JSFunction:
```console
(lldb) job JSFunction::cast(value)->code()
(lldb) job JSFunction::cast(value)->abstract_code()
(lldb) job JSFunction::cast(value)->feedback_vector()
```
Just to recall, we called this function:
```c++
Local<Value> f_value = ExecuteString(env, MainSource(env), script_name);
```
To compile boostrap_node.js and the returned value if a Function that has been compiled. This is 
later called:
```c++
auto ret = f->Call(env->context(), Null(env->isolate()), 1, &arg);
```
This will once again land us in `Invoke` and:
```c++
value = CALL_GENERATED_CODE(isolate, stub_entry, orig_func, func, recv, argc, argv);
```
This is a function call so we will first entry the JSEntryStub, then the InterpreterTrampoline
which will delegate to `Generate_Call`
```console
    0x3db5b98c1954: callq  0x3db5b98bc400

    0x3db5b98bc400: testb  $0x1, %dil         // __ JumpIfSmi(rdi, &non_callable);
    0x3db5b98bc404: je     0x3db5b98bc450     // __ JumpIfSmi(rdi, &non_callable);
    0x3db5b98bc40a: movq   -0x1(%rdi), %rcx   // __ CmpObjectType(rdi, JS_FUNCTION_TYPE, rcx);
    0x3db5b98bc40e: cmpb   $-0x1, 0xb(%rcx)   // CmpInstanceType(map, type) called by CmpObjectType
    0x3db5b98bc412: je     0x3db5b98bbe20     // __ j(equal, masm->isolate()->builtins()->CallFunction(mode), RelocInfo::CODE_TARGET);
                                              // So the next instruction should match that from CallFunction.

    0x3db5b98bbe20: testb  $0x1, %dil         // 

```


### Assembler:j(Condition cc, Handle\<Code\> target, RelocInfo::Mode rmode) 
Can be found in `src/x64/assembler-x64.cc` (line 1367):
```c++
void Assembler::j(Condition cc, Handle<Code> target, RelocInfo::Mode rmode) {
  EnsureSpace ensure_space(this);
  DCHECK(is_uint4(cc));
  // 0000 1111 1000 tttn #32-bit disp.
  emit(0x0F);
  emit(0x80 | cc);
  emit_code_target(target, rmode);
}
``
Take this expression as an example:
```c++
__ j(equal, masm->isolate()->builtins()->CallFunction(mode), RelocInfo::CODE_TARGET);
```
`Condition` is an enum found in `src/x64/assembler-x64.h`:
```c++
enum Condition {
  // any value < 0 is considered no_condition
  no_condition  = -1,

  overflow      =  0,
  no_overflow   =  1,
  below         =  2,
  above_equal   =  3,
  equal         =  4,
  not_equal     =  5,
  below_equal   =  6,
  above         =  7,
  negative      =  8,
  positive      =  9,
  parity_even   = 10,
  parity_odd    = 11,
  less          = 12,
  greater_equal = 13,
  less_equal    = 14,
  greater       = 15,

  // Fake conditions that are handled by the
  // opcodes using them.
  always        = 16,
  never         = 17,
  // aliases
  carry         = below,
  not_carry     = above_equal,
  zero          = equal,
  not_zero      = not_equal,
  sign          = negative,
  not_sign      = positive,
  last_condition = greater
};
```
The jump instruction generated for this would look like:
```console
(lldb) dis -f -b
->  0x16e27203c412: 0f 84 08 fa ff ff  je     0x16e27203be20

  emit(0x0F);
  emit(0x80 | cc);
  emit_code_target(target, rmode);
```
So we can see that `0f` matches the `emit(0x0F)` call.
And `84` matches `emit(0x80 | 4)` which is 84 in hex. (4 is Condition::equal)
I was wondering about `(0x80 | cc)` and what that does. Well this will determin the type of jmp
opcode to emit. A near jmp opcode consists of two bytes, the first being with `0F` and the second
varies depending on the type of jmp. So lets take Condition::not_equal which is 5:
```console
(lldb) expr -f hex -- `0x80 | 5`
(int) $304 = 0x00000085
```

The last line, `emit_code_target(target, rmode)` which can be found in `src/x64/assembler-x64-inl.h`.
```c++
int current = static_cast<int>(code_targets_.size());

```
```console
expr isolate->builtins()->Call(static_cast<ConvertReceiverMode>(ConvertReceiverMode::kAny))->address()
(v8::internal::Address) $306 = 0x0000302f3413c3a0
```


### x64 jump instructions
Instruction Description                         Flags   short jump opcode   near jump opcodes 
* JO        Jump if overflow                    ZF = 1  70                  0F 80
* JE/JZ     Jump if equal/Jump if zero          ZF = 1  74                  0F 84
* JNE/JNZ   Jump if not equal/Jump if no zero   ZF = 1  75                  0F 85

#### Types of Jumps
A short `jmp` is encoded as two bytes 74 and the number of bytes +/- relative to the instruction pointer. The operand can
only be a 8-bit operand. So this can jump -126 to +129 bytes.

A near `jmp` allows for jumps in the current segment and uses 0F 84 and the operand is a 16-bit operand 


So a function is translated into bytecode by the ByteCodeGenerator which will visit the AST and emit bytecodes for 
each AST node. The bytecodes are set on the SharedFunctionInfo object and the code entry address is set to the
InterpreterEntryTrampoline builtin stub. InterpreterEntryTrampoline will set up the stack frame and then dispatch
to the interpreter's bytecode handler for the functions first bytecode. I think this is what the last call is 
doing above. So what is the first bytecode in our case?

```console
(lldb) up 2
(lldb) job v8::internal::JSFunction::cast(func)->abstract_code()
0x37b34d8adfe1: [BytecodeArray] in OldSpaceParameter count 1
Frame size 8
    0 E> 0x37b34d8ae01a @    0 : 93                StackCheck
  299 S> 0x37b34d8ae01b @    1 : 6f 00 00 00       CreateClosure [0], [0], #0
         0x37b34d8ae01f @    5 : 1e fb             Star r0
21946 S> 0x37b34d8ae021 @    7 : 97                Return
Constant pool (size = 1)
0x37b34d8adfc9: [FixedArray] in OldSpace
 - map = 0x37b31cc022f1 <Map(HOLEY_ELEMENTS)>
 - length: 1
           0: 0x37b34d8adf19 <SharedFunctionInfo>
Handler Table (size = 16)
```
So at this point we have the JSEntryStub which will take information from the isolate->isolate->thread_local_top_
and make it available to the rest of the code that will be called. This is something that has to be done each
time a function is entered. Now, the first InterpreterEntryTrampoline is called and hopefully we'll be able
to verify that this goes through the func->abstract_code() and compiles it. 



```c++
(lldb) job *isolate->builtins()->CallFunction(static_cast<ConvertReceiverMode>(ConvertReceiverMode::kAny))        0x3db5b98bbdc1: [Code]
kind = BUILTIN
name = CallFunction_ReceiverIsAny
compiler = unknown
Instructions (size = 510)
0x3db5b98bbe20     0  40f6c701       testb rdi,0x1                                      // __ AssertFunction
0x3db5b98bbe24     4  7510           jnz 0x3db5b98bbe36  (CallFunction_ReceiverIsAny)
0x3db5b98bbe26     6  48ba0000000036000000 REX.W movq rdx,0x3600000000
0x3db5b98bbe30    10  e88b9b0200     call 0x3db5b98e59c0  (Abort)    ;; code: BUILTIN
0x3db5b98bbe35    15  cc             int3l
0x3db5b98bbe36    16  57             push rdi
0x3db5b98bbe37    17  488b7fff       REX.W movq rdi,[rdi-0x1]
0x3db5b98bbe3b    1b  807f0bff       cmpb [rdi+0xb],0xff
0x3db5b98bbe3f    1f  5f             pop rdi
0x3db5b98bbe40    20  7410           jz 0x3db5b98bbe52  (CallFunction_ReceiverIsAny)
0x3db5b98bbe42    22  48ba000000003b000000 REX.W movq rdx,0x3b00000000
0x3db5b98bbe4c    2c  e86f9b0200     call 0x3db5b98e59c0  (Abort)    ;; code: BUILTIN
0x3db5b98bbe51    31  cc             int3l                                             // end __ AssertFunction

0x3db5b98bbe52    32  488b571f       REX.W movq rdx,[rdi+0x1f]
```


### Codestubs
Are declared in `src/code-stubs.h`. Lets take a look at two `CEntry` and `JSEntry`:
```c++
#define CODE_STUB_LIST_ALL_PLATFORMS(V)       \
  /* --- PlatformCodeStubs --- */             \
  ...
  V(CEntry)                                   \
  ...
  V(JSEntry)                                  \
```
CodeStub extends ZoneObject. It has an enum named major with all the code stubs:
```c++
enum Major {
  NoCache = 0,
  ...
  CEntry,
  ...,
  JSEntry,
};
```
`Handle<Code> GetCode()` will return the Code for this code stub and generate it if needed.
If the code is not in the cache then the following will be called to generate the code:
```c++
Handle<Code> new_object = GenerateCode();
```

### Zone
Is very well documented:
```c++
// The Zone supports very fast allocation of small chunks of
// memory. The chunks cannot be deallocated individually, but instead
// the Zone supports deallocating all chunks in one fast
// operation. The Zone is used to hold temporary data structures like
// the abstract syntax tree, which is deallocated after compilation.
```

### ZoneObject 
Is an object that exist in a zone and is indented to be extended (like CodeStub does).


### Script::Compile
Will delegate to ScriptCompiler::Compile, and then to ScriptCompiler::CompileUnboundInternal, and then to ScriptCompiler::CompileUnboundInternal.
```c++
i::MaybeHandle<i::SharedFunctionInfo> maybe_function_info = i::Compiler::GetSharedFunctionInfoForScript(
-> 2332            str, name_obj, line_offset, column_offset, source->resource_options,
   2333            source_map_url, isolate->native_context(), NULL, &script_data,
   2334            options, i::NOT_NATIVES_CODE, host_defined_options);
```
```c++
  ParseInfo parse_info(script);
  Zone compile_zone(isolate->allocator(), ZONE_NAME);
  ...
  maybe_result = CompileToplevel(&parse_info, isolate);
```

### CompileToplevel
```c++
  if (parse_info->literal() == nullptr && !parsing::ParseProgram(parse_info, isolate)) {
  ...
  std::unique_ptr<CompilationJob> outer_function_job(GenerateUnoptimizedCode(parse_info, isolate, &inner_function_jobs));
...
```

### GenerateUnoptimizedCode
`src/compiler.cc`
```c++
  Compiler::EagerInnerFunctionLiterals inner_literals;
  if (!Compiler::Analyze(parse_info, &inner_literals)) {
    return std::unique_ptr<CompilationJob>();
  }
  std::unique_ptr<CompilationJob> outer_function_job(
      PrepareAndExecuteUnoptimizedCompileJob(parse_info, parse_info->literal(), isolate));
```

### PrepareAndExecuteUnoptimizedCompileJob
`src/compiler.cc` 
```console
(lldb) br s -f compiler.cc -l 385
```
```c++
std::unique_ptr<CompilationJob> job(interpreter::Interpreter::NewCompilationJob(parse_info, literal, isolate));
  if (job->PrepareJob() == CompilationJob::SUCCEEDED && job->ExecuteJob() == CompilationJob::SUCCEEDED) {
    return job;
  }
```
Lets take a look at `parse_info`:
```console
(lldb) job parse_info->script_->source()
"'use strict';\x0a\x0a(function frogger(process) {\x0a  process._rawDebug('entry function');\x0a\x0a  function startup() {\x0a  process._rawDebug('startup function');\x0a  return true;\x0a  }\x0a\x0a  startup();\x0a});\x0a"
```
Notice that this is the complete contents of bootstrap_node.js:
```console
(lldb) job parse_info->script_->name()
"bootstrap_node.js"
```
`PrepareJob` will print the AST if configured with `--print-ast`:
```console
[generating bytecode for function: ]
--- AST ---
FUNC at 0
. KIND 0
. SUSPEND COUNT 0
. NAME ""
. INFERRED NAME ""
. EXPRESSION STATEMENT at 284
. . LITERAL "use strict"
. EXPRESSION STATEMENT at 299
. . ASSIGN at -1
. . . VAR PROXY local[0] (0x10702f338) (mode = TEMPORARY) ".result"
. . . FUNC LITERAL at 300
. . . . NAME
. . . . INFERRED NAME
. . . . PARAMS
. . . . . VAR (0x10701cd48) (mode = VAR) "process"
. RETURN at -1
. . VAR PROXY local[0] (0x10702f338) (mode = TEMPORARY) ".result"
```
`VAR PROXY` indicates that this scope resolution will connect these nodes declaring VAR nodes.
This is all `PrepareJob` does.

`ExecuteJob()` will call:
```c++
return UpdateState(ExecuteJobImpl(), State::kReadyToFinalize);
```
The call to `ExecuteJobImpl` will end up in interpreter.cc:191 in our case. In this function we find the following:
```c++
  generator()->GenerateBytecode(stack_limit());
```
Now we are getting closer to figuring out how the bytecode is generated. This will land us in bytecode-generator.cc:894.

`GenerateBytecode`:
```c++
  InitializeAstVisitor(stack_limit);
  ContextScope incoming_context(this, closure_s