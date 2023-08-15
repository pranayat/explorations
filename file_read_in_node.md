# User code
app.js
```js
const fs = require('fs');

fs.readFile('file.txt', 'utf8', (err, data) => {
    if (err) {
        console.error('Error reading file:', err);
        return;
    }
    console.log('File content:', data);
});
```

```
node app.js
```

- The `node` binary initializes the v8 engine
- V8 JIT compiler compiles the JS to bytecode, this is then interpreted and executed by the V8 interpreter. The interpretor makes the required OS system calls.
- V8 optimizing compiler 'Turbofan' compiles part of the code to CPU architecture specific machine code which is directly executed by the CPU.

# JS wrapper
- The `fs.readFile` function imported above is in the [node/lib/fs.js](https://github.com/nodejs/node/blob/3d5e7cd8f049f1d2bd041974fbde87f71cbbbf31/lib/fs.js#L380https://github.com/nodejs/node/blob/3d5e7cd8f049f1d2bd041974fbde87f71cbbbf31/lib/fs.js#L380) file
- calls the C++ code via bindings TODO: how do these work
```js
function readFile(path, options, callback) {
  callback = maybeCallback(callback || options);
  options = getOptions(options, { flag: 'r' });
  const ReadFileContext = require('internal/fs/read/context');
  const context = new ReadFileContext(callback, options.encoding);
  context.isUserFd = isFd(path); // File descriptor ownership

  if (options.signal) {
    context.signal = options.signal;
  }
  if (context.isUserFd) {
    process.nextTick(function tick(context) {
      ReflectApply(readFileAfterOpen, { context }, [null, path]);
    }, context);
    return;
  }

  if (checkAborted(options.signal, callback))
    return;

  const flagsNumber = stringToFlags(options.flag, 'options.flag');
  path = getValidatedPath(path);

  const req = new FSReqCallback();
  req.context = context;
  req.oncomplete = readFileAfterOpen;
  binding.open(pathModule.toNamespacedPath(path),
               flagsNumber,
               0o666,
               req);
}
```

# C++ implementation
- The implementation seems to be in [node/src/node_file.cc](https://github.com/nodejs/node/blob/3d5e7cd8f049f1d2bd041974fbde87f71cbbbf31/src/node_file.cc#L1964)
- makes requests to `libuv` in calls like `uv_fs_open` and `uv_fs_read`
```c++
static void ReadFileSync(const FunctionCallbackInfo<Value>& args) {
  Environment* env = Environment::GetCurrent(args);
  auto isolate = env->isolate();

  CHECK_GE(args.Length(), 2);

  BufferValue path(env->isolate(), args[0]);
  CHECK_NOT_NULL(*path);

  CHECK(args[1]->IsInt32());
  const int flags = args[1].As<Int32>()->Value();

  if (CheckOpenPermissions(env, path, flags).IsNothing()) return;

  uv_fs_t req;
  auto defer_req_cleanup = OnScopeLeave([&req]() { uv_fs_req_cleanup(&req); });

  FS_SYNC_TRACE_BEGIN(open);
  uv_file file = uv_fs_open(nullptr, &req, *path, flags, 438, nullptr);
  FS_SYNC_TRACE_END(open);
  if (req.result < 0) {
    // req will be cleaned up by scope leave.
    Local<Value> out[] = {
        Integer::New(isolate, req.result),       // errno
        FIXED_ONE_BYTE_STRING(isolate, "open"),  // syscall
    };
    return args.GetReturnValue().Set(Array::New(isolate, out, arraysize(out)));
  }
  uv_fs_req_cleanup(&req);

  auto defer_close = OnScopeLeave([file]() {
    uv_fs_t close_req;
    CHECK_EQ(0, uv_fs_close(nullptr, &close_req, file, nullptr));
    uv_fs_req_cleanup(&close_req);
  });

  std::string result{};
  char buffer[8192];
  uv_buf_t buf = uv_buf_init(buffer, sizeof(buffer));

  FS_SYNC_TRACE_BEGIN(read);
  while (true) {
    auto r = uv_fs_read(nullptr, &req, file, &buf, 1, -1, nullptr);
    if (req.result < 0) {
      FS_SYNC_TRACE_END(read);
      // req will be cleaned up by scope leave.
      Local<Value> out[] = {
          Integer::New(isolate, req.result),       // errno
          FIXED_ONE_BYTE_STRING(isolate, "read"),  // syscall
      };
      return args.GetReturnValue().Set(
          Array::New(isolate, out, arraysize(out)));
    }
    uv_fs_req_cleanup(&req);
    if (r <= 0) {
      break;
    }
    result.append(buf.base, r);
  }
  FS_SYNC_TRACE_END(read);

  args.GetReturnValue().Set(String::NewFromUtf8(env->isolate(),
                                                result.data(),
                                                v8::NewStringType::kNormal,
                                                result.size())
                                .ToLocalChecked());
}
```

# libuv - a library written in C
- [File operations](https://docs.libuv.org/en/v1.x/fs.html) are not epollable and run on the libuv thread pool
- If they were epollable, libuv would have made the epoll() system call to poll the OS for file I/O
- work requests to the threadpool are made using the [uv_queue_work](https://github.com/nodejs/node/blob/6f9d6f277b8dc973ce8bdbb5e135fd22e88b205b/deps/uv/src/threadpool.c#L368) call via the wrapper [here](https://github.com/nodejs/node/blob/6f9d6f277b8dc973ce8bdbb5e135fd22e88b205b/src/threadpoolwork-inl.h#L37)
  
