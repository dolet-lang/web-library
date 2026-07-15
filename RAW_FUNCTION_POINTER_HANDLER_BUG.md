# Raw Function Pointer Route Handler Bug

## Summary

The `web` package currently relies on the old route handler API:

```dlt
WebRouter.add("GET", "/", home_handler)
WebApp.run(port)
```

This is the desired API and should remain supported. However, it currently crashes at runtime because named functions used as route handlers are stored as raw `i64` values and later cast back to `fun(Request) -> str`.

The issue appears to be in the compiler/codegen support for raw function pointers or named function references, not in the `web` package HTTP parser, socket code, `Response.html`, `Response.render`, MySQL, or the HTML template.

## Current `web` Package Behavior

Relevant code path:

```dlt
# router.dlt
static fun add(method: str, path: str, handler: i64):
    ...
    Memory.write_i64(WebRouter.routes + offset + 16, handler)

# webapp.dlt
handler: i64 = WebRouter.find_route(req.method, req.path)
handler_fn: fun(Request) -> str = handler as fun(Request) -> str
response: str = handler_fn(req)
```

The crash happens when `handler_fn(req)` is called.

## Observed Runtime Failure

On Windows, opening the route can produce an Application Error similar to:

```text
The instruction at 0x00007FF... referenced memory at 0xFFFFFFFFFFFFFFFF.
The memory could not be read.
```

Sometimes the client sees a timeout or connection reset because the server process crashes or gets stuck while trying to invoke the handler.

## Minimal Reproduction

```dlt
import web

port: i32 = 25568

fun ok_handler(req: Request) -> str:
    return Response.html("ok")

print(@"router repro running at http://localhost:{port}")

WebRouter.add("GET", "/", ok_handler)
WebApp.run(port)
```

Compile and run:

```powershell
doletc index.dlt -o app.exe
.\app.exe
curl.exe http://127.0.0.1:25568/
```

Expected:

```text
ok
```

Actual:

- Runtime crash, connection reset, or timeout.
- Windows error references invalid memory, often `0xFFFFFFFFFFFFFFFF`.

## Smaller Compiler-Level Reproduction

This isolates the problem away from sockets and router matching:

```dlt
import web

fun ok_handler(req: Request) -> str:
    print("inside handler")
    return Response.html("ok")

req: Request = Request()
handler: i64 = ok_handler
handler_fn: fun(Request) -> str = handler as fun(Request) -> str
response: str = handler_fn(req)
print(response)
```

Observed compile/codegen failure included:

```text
Error: MLIR translation failed
index.mlir:...: error: @ identifier expected to start with letter or '_'
    %94 = llvm.mlir.addressof @89 : !llvm.ptr
```

This strongly suggests named function references are being lowered incorrectly when coerced through `i64` / raw function pointer form.

## What Was Tested

A manual socket loop that does not use `WebRouter` and does not call a handler through a raw pointer works.

A route handler stored as a heap closure also worked in testing:

```dlt
WebRouter.add("GET", "/", fun(req: Request) -> str:
    return ok_handler(req)
)
```

But that changes the public API and is not the desired final fix. The desired API is still:

```dlt
WebRouter.add("GET", "/", ok_handler)
```

## Likely Root Cause

The compiler currently supports closures (`fun(...) -> ...`) through the closure ABI, but direct named function pointer support is incomplete or broken.

The web package's old API depends on one of these being valid:

```dlt
handler: i64 = ok_handler
```

or:

```dlt
handler_fn: fun(Request) -> str = handler as fun(Request) -> str
```

The generated MLIR/LLVM must preserve a valid function address and call it with the correct ABI.

## Desired Compiler Fix

Keep the `web` package API as-is and fix compiler/codegen so named functions can safely be used as raw function references.

Requirements:

1. `WebRouter.add("GET", "/", home_handler)` must compile.
2. Storing the handler in an `i64` route table must preserve a valid function address.
3. Casting the stored `i64` back to `fun(Request) -> str` must produce a callable function pointer.
4. The indirect call ABI must match direct named functions with signature `fun(Request) -> str`.
5. This should not break closure support or `@heap fun(...)` support.

## Suggested Tests To Add

### 1. Named Function To Raw Pointer Roundtrip

```dlt
import web

fun ok_handler(req: Request) -> str:
    return Response.html("ok")

req: Request = Request()
handler: i64 = ok_handler
handler_fn: fun(Request) -> str = handler as fun(Request) -> str
print(handler_fn(req))
```

### 2. WebRouter Handler Invocation

```dlt
import web

port: i32 = 25568

fun ok_handler(req: Request) -> str:
    return Response.html("ok")

WebRouter.add("GET", "/", ok_handler)
WebApp.run(port)
```

Then request `/` and assert the body is `ok`.

### 3. Template Rendering Through Router

```dlt
import web

indexPath: str = "index.html"
port: i32 = 25568

fun home_handler(req: Request) -> str:
    ctx: i64 = Context.create()
    Context.set(ctx, "title", "home")
    return Response.render(indexPath, ctx)

WebRouter.add("GET", "/", home_handler)
WebApp.run(port)
```

Expected: `{{title}}` renders as `home` and the process does not crash.

## Repository History Notes

Before any attempted workaround, `packages/web` was pushed and was already up to date.

A temporary workaround commit changed route handlers to heap closures:

```text
6ff912f fix: store web route handlers as heap closures
```

That commit was reverted to preserve the old API:

```text
5d825d1 Revert "fix: store web route handlers as heap closures"
```

So the current `web` package intentionally remains on the old raw handler API and awaits the compiler fix.