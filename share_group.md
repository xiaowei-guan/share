# Understanding the Share Group
In OpenGL, a Context holds the entire state of the machine (which textures are bound, which shaders are active, etc.).By default, resources created in Context A are invisible to Context B.

A Share Group is a link between two or more contexts that allows them to share Object Handles.
What is Shared?
 - Textures and Renderbuffers
 - Buffer Objects (VBOs, EBOs, etc.)
 - Shader and Program Objects
   
What is NOT Shared?
 - Container Objects: Specifically VAOs (Vertex Array Objects) and FBOs (Framebuffer Objects). These are considered "state containers" and must be created per-context.
 - The State Machine: Changing the clear color or binding a texture in the IO thread won't affect the Render thread.
 
Expert Tip: While the handles are shared, you must use Sync Objects ($glFenceSync$) to ensure the IO thread has finished writing the data before the Render thread tries to draw with it.

## 1. The Core Concept: EGL Context Sharing
In EGL, the "Share Group" is established at the moment of context creation. By passing an existing EGLContext into the share_context parameter of eglCreateContext, the driver links their internal resource tables (handles for textures, shaders, and VBOs).

Function Signature
```C++
EGLContext eglCreateContext(
    EGLDisplay display,
    EGLConfig config,
    EGLContext share_context, // The "Master" context to share with
    const EGLint *attrib_list
);
```
## 2. Implementation: Setting Up Two Contexts
To verify that the OpenGL backend uses two threads for PSO (Pipeline State Object) setup, you need a Render Context (Main Thread) and an IO Context (Worker Thread).

```C++
// 1. Initialize EGL Display and Config (Standard Boilerplate)
EGLDisplay display = eglGetDisplay(EGL_DEFAULT_DISPLAY);
eglInitialize(display, NULL, NULL);
EGLConfig config; 
// ... choose your config using eglChooseConfig ...

// 2. Create the Primary Render Context
EGLint contextAttribs[] = { EGL_CONTEXT_CLIENT_VERSION, 3, EGL_NONE };
EGLContext renderContext = eglCreateContext(display, config, EGL_NO_CONTEXT, contextAttribs);

// 3. Create the Secondary IO Context (The Share Group magic)
// We pass 'renderContext' as the 3rd argument.
EGLContext ioContext = eglCreateContext(display, config, renderContext, contextAttribs);

if (ioContext == EGL_NO_CONTEXT) {
    // Error: Driver does not support sharing or config mismatch
}
```
## 3. Handling the IO Thread "Surface"
A common mistake when moving away from GLFW is forgetting that a context needs a Surface to be made "Current," even if it’s just for background loading.

Since the IO thread doesn't draw to the screen, you should use a Pbuffer Surface (Pixel Buffer) or the Surfaceless extension if supported.

```C++
// Inside IO Thread
glfwMakeContextCurrent(ioContext);

GLuint program = glCreateProgram();
// ... Attach shaders, Link program ...

// CRITICAL: Insert a sync fence so the Render thread knows we are done
GLsync uploadFence = glFenceSync(GL_SYNC_GPU_COMMANDS_COMPLETE, 0);
glFlush(); // Force commands to the GPU
```
## 4. Verifying the PSO Setup
To confirm your backend is truly utilizing the second thread for PSOs (Shader compilation/linking), you can monitor the GL_COMPLETION_STATUS. This query allows the Render thread to check if the IO thread has finished the heavy lifting without blocking the main loop.

Verification Strategy:
IO Thread: Calls glLinkProgram(programHandle).

Render Thread: Periodically checks:

```C++
GLint linked = 0;
glGetProgramiv(programHandle, GL_COMPLETION_STATUS_KHR, &linked); 
if (linked == GL_TRUE) {
    // PSO is ready to use without stuttering the frame
}
```
Would you like me to provide a complete C++ class structure that manages this thread-safe handoff between the IO and Render contexts?
