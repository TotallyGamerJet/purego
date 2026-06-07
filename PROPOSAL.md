# Background

Go currently relies on Cgo for foreign function calls. The cgo tool is quite powerful in that it can generate type safe C to Go translations without being able to parse C itself. However, it is annoying to work with because it requires a dependency on an external C toolchain every time a build happens, causes slow build times due to invoking the C compiler, and has increased call overhead due to stack switching and scheduler coordination.

This proposal does not aim to address the performance point as that has been worked on plenty before ([#68587](https://github.com/golang/go/issues/68587), [#58336](https://github.com/golang/go/issues/58336), [#60961](https://github.com/golang/go/issues/60961)).

Alternative approaches to Cgo have emerged with [purego](github.com/ebitengine/purego) and [goffi](github.com/go-webgpu/goffi) being the most popular, which demonstrate that it is already possibles (with some hacks) to call a C function without a C compiler and more importantly that there is a true desire from the community for this feature.

On Windows, there is the [syscall](https://pkg.go.dev/syscall?GOOS=windows) package which provides SyscallN but it has the limitation of only being able to call functions with uintptr sized arguments which does not meet all libraries' needs. Additionally, other platform APIs and standard libraries (e.g., libc, CoreFoundation.framework) are not accessible by a similar package. The windows syscall package also introduces inefficiencies through extra indirection, module handling overhead, as documented in this [blog post](https://blog.kowalczyk.info/a-3g9f/optimizing-calling-windows-dll-functions-in-go.html).

Go already contains nearly all machinery required to call foreign ABI functions. The remaining gap is a compiler-supported way to invoke a function pointer using the platform ABI. This proposal fills that gap without requiring C parsing, header processing, code generation during builds, or changes to the Go type system.

# Proposal

Introduce a new compiler directive: `//go:cgo_call localname [abi]`. This directive binds a Go function declaration to a function pointer stored in a uintptr, typically populated via `//go:cgo_import_dynamic` or runtime lookup.

The `go:cgo_call` directive must be followed by a function declaration with no body. It specifies that the function should call the C function provided by the localname uintptr variable using the abi listed or defaults to "system" abi if none is provided.

For example,

```go
//go:cgo_import_dynamic mypkg.putsPtr puts "libc.so.6" 
var putsPtr uintptr

//go:cgo_call putsPtr system
func puts(s *byte) int32 

puts(&[]byte("hello from go\x00")[0]) 
```

Another example which assigns the variable,

```go
//go:cgo_import_dynamic mypkg.vkGetInstanceProcAddrPtr vkGetInstanceProcAddr "vulkan-1.dll"
var vkGetInstanceProcAddrPtr uintptr

//go:cgo_call vkGetInstanceProcAddrPtr
func vkGetInstanceProcAddr(instance VkInstance, pName *byte) uintptr

var vkCreateInstancePtr = vkGetInstanceProcAddr(nil, &[]byte("vkCreateInstance\x00")[0]);

//go:cgo_call vkCreateInstancePtr
func vkCreateInstance(
    pCreateInfo *VkInstanceCreateInfo,
    pAllocator  *VkAllocationCallbacks,
    pInstance   *VkInstance,
) VkResult
```

And one for macOS,

```go
//go:cgo_import_dynamic _ _ "/System/Library/Frameworks/CoreFoundation.framework/CoreFoundation"
//go:cgo_import_dynamic mypkg.CFStringCreateWithCStringPtr CFStringCreateWithCString ""
var CFStringCreateWithCStringPtr uintptr

// Define needed CoreFoundation types
type CFAllocatorRef uintptr
type CFStringRef uintptr
type CFStringEncoding uint32

const kCFStringEncodingUTF8 CFStringEncoding = 0x08000100
const kCFAllocatorDefault CFAllocatorRef = 0

//go:cgo_call CFStringCreateWithCStringPtr
func CFStringCreateWithCString(
    alloc CFAllocatorRef,
    cStr *byte,
    encoding CFStringEncoding,
) CFStringRef

cfStr := CFStringCreateWithCString(kCFAllocatorDefault, &[]byte("Hello from Go\x00")[0], kCFStringEncodingUTF8)
```

Note, these examples exclude types and proper pinning of strings for brevity.

### **Supported Types**

To simplify ABI handling and ensure correctness, the initial implementation should support:

*   **Fixed-size integers**: int8 (byte), int16, int32, int64, uint8, uint16, uint32, uint64

*   **Floating point**: float32, float64

*   **Pointer types**: unsafe.Pointer, uintptr, \*T (treated as pointer)


It purposely excludes structs, complex numbers, variadics, slices, maps, channels and strings as they complicate the ABI and memory handling needed to correctly pass them into C.

The optional ABI argument may be one of "system" (default) or "raw" (runtime only).

For the system ABI, the directive calls are routed through the existing runtime mechanism. The compiler generates a function that maps the arguments to the system's C calling convention (SysV, Microsoft x64 calling convention, etc.) and then calls the function stored in the uintptr variable. This reuses the same machinery currently used by cgo (`runtime.cgocall`).

The "raw" ABI performs a direct call without `runtime.cgocall`. This means that there are no stack growth checks, no scheduler coordination, or no GC safepoints. This ABI option would only be allowed inside the runtime for use in implementing `runtime/cgo` package in a pure Go way.

The initial implementation would be supported on amd64 & arm64 for windows, linux and darwin as that covers the most popular platforms.

A `GOEXPERIMENT=cgocall` build option can be added to conditionally enable the feature while its implementation and feasible are tested.

# Rationale

The ease of cross compiling with a pure Go program is one of the most popular things about Go. At the same time, it is necessary to provide some way to interface with C since it is not possible to rewrite everything in Go even if that’d be preferred. Therefore, it is unfortunate that the ease of cross-compilation is completely thrown out once any attempt is made to interact with C.

The Go ecosystem is accustomed to doing work outside of the compiler to avoid slowing it down. That is why `go:generate` must be ran manually and the same should be true for interacting with C. The bindings can easily be generated by tools that support actually parsing C files like [cc/v3](https://pkg.go.dev/modernc.org/cc/v3) instead of parsing the error messages of specific C compilers. The cost of reading and generating bindings would be done once by the library maintainer and then users that import the package no longer need to worry about making sure they have the correct C compiler installed.

The reason for the indirection of a uintptr variable is because there is no defined way to cast a uintptr to a function of a certain ABI. Adding the indirection means that APIs that rely on returning function pointers can still be called (OpenGL, Vulkan, or plain dlopen). The annoyance of this should be limited as it is managed by library maintainers.

Another annoyance that this proposal resolves is when importing symbols in the C package namespace. LSPs give up on checking if a C symbol exists as that would require invoking the C compiler. With this method the entire file is made solely of Go code so type checking is simplified and works as expected.

# The Cost

Every feature has a cost and this one is no different. It adds an entirely new way to call into C which confuses the choice for users. Should they choose the old Cgo or the new one? It also increases the complexity of the runtime to support multiple ABIs for each differing calling convention. In addition it is not able to completely replace the current Cgo implementation as it does not support a way to statically link C into the Go binary . This means it is only really useful for linking against system libraries guaranteed to be present on the system or requiring distributors to bundle the shared library with their binary.

# Potential Future Work

### Support ABI-aware struct passing and return values

The current suggested implementation excludes structs to simplify the initial implementation. Adding it would bring it on par with what is supported by Cgo. Structs would want to use `structs.HostLayout` to ensure that the memory correctly aligns with what is expected on the C side.

### Additional Platforms

The current suggested implementation is limited to the most utilized platforms (windows, linux, darwin). Once an initial implementation is created other operating systems and architectures could be added.

### **Additional ABIs**

Most obvious would be Windows, with its many ABI flavors: stdcall, cdecl, and fastcall. These would most likely need to be limited to their respective GOOS/GOARCH pairs.

### Port runtime/cgo

This proposal does not require the `runtime/cgo` package be ported to Go as it is currently possible to implement it outside the stdlib (see [internal/fakcgo](https://github.com/ebitengine/purego/tree/main/internal/fakecgo)). However, it would be nice to avoid exposing these runtime symbols to the world ([#67401](https://github.com/golang/go/issues/67401), [#79702](https://github.com/golang/go/issues/79702)).

### Callbacks

Some APIs require passing a function into C. This proposal does not provide a way to get a function pointer that matches the C ABI. Again, purego does handle this but adding directly into the compiler would be best and most performant as purego relies on reflection.

### Dynamically Pull in Variables

It would be nice to provide a way to dynamically link to variables from shared libraries. This isn't required for initial release because all symbols can be grabbed using dlsym which is made possible by this proposal.

### Support additional argument / return values

The proposal also ignores complex numbers, strings, maps, interfaces which can be sent into C land and back out. I am not sure how likely it is people use these types with Cgo though.

### Bundling in C code

Adding in a way to bundle pre-compiled C code would make it possible the use external C libraries in a single binary. However, shipping pre-compiled code is looked down upon from a security standpoint so this may not happen. Instead prefer to port the code to Go using the dynamic linking provided in this proposal to utilize system libraries.

* * *

## Other Resources

C compiler requirement leads to hacks solution: [https://stoolap.io/blog/2026/04/08/calling-a-rust-library-from-go-with-cgo-disabled/](https://stoolap.io/blog/2026/04/08/calling-a-rust-library-from-go-with-cgo-disabled/)

### Related Issues:

proposal to export `cgo_import_static`: [https://github.com/golang/go/issues/75473](https://github.com/golang/go/issues/75473)

> [cherrymui](https://github.com/cherrymui): I think there is a general demand for calling C functions in precompiled (static or dynamic) objects, which I think is definitely worth considering. But I think it needs a more complete solution, e.g. a general mechanism to call a C function. Also, on some platforms, it may require the program to be initialized in certain way, e.g. using pthread to create threads, instead of direct syscalls.

wasm import global: [https://github.com/golang/go/issues/59149](https://github.com/golang/go/issues/59149)

wasm import: [https://github.com/golang/go/issues/38248](https://github.com/golang/go/issues/38248)

code execution caused by Cgo: [https://github.com/golang/go/issues/23672](https://github.com/golang/go/issues/23672)