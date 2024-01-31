---
title: NDK静态编译SDL2+Imgui
date: 2023-10-07 12:34:00 +0800
categories: [Android]
tags: [Android]     # TAG names should always be 
---

# NDK静态编译SDL2+Imgui

有些时候为了兼容性只能CPU写像素实现绘制,想画出好看的图形和UI就得用上Imgui.

Imgui提供了一个SDL2软渲染的实现,软渲染使用CPU的计算能力来执行图形渲染任务，因此不需要特殊的硬件支持,兼容性十分好.

可以用SDL2+Imgui渲染出离屏缓冲,把像素拿出来就能达到目的.

## 编译SDL2

SDL版本 2.28.4

NDK版本 r26

### 修改CMakeList.txt

把

```
cmake_minimum_required(VERSION 3.0.0...3.5)
project(SDL2 C CXX)
```

替换成下面的,路径自行替换

```
set(CMAKE_POLICY_DEFAULT_CMP0091 NEW)

set(CMAKE_C_COMPILER /home/www/Desktop/toolchains/toolchain-arm64-21/bin/aarch64-linux-android33-clang)
set(CMAKE_CXX_COMPILER /home/www/Desktop/toolchains/toolchain-arm64-21/bin/aarch64-linux-android33-clang++)
set(CMAKE_SYSROOT /home/www/Desktop/toolchains/toolchain-arm64-21/sysroot)

cmake_minimum_required(VERSION 3.0.0...3.5)
project(SDL2 C CXX)

set(CMAKE_INSTALL_PREFIX /home/www/Desktop/sdl/install)
set(SDL_STATIC ON)
set(SDL_SHARED OFF)
```

编译安装

```bash
mkdir build
cmake ../sdl2
make
make install
```

## 编译Imgui

我们只需要Imgui源码的这几个文件,创建src和include文件夹,把文件放进对应的文件夹里面

- src
  - imgui.cpp
  - imgui_draw.cpp
  - imgui_impl_sdl2.cpp
  - imgui_impl_sdlrenderer.cpp
  - imgui_internal.h
  - imgui_tables.cpp
  - imgui_widgets.cpp
  - imstb_rectpack.h
  - imstb_textedit.h
  - imstb_truetype.h
- include
  - imconfig.h
  - imgui.h
  - imgui_impl_sdl2.h
  - imgui_impl_sdlrenderer.h

替换imgui_impl_sdlrenderer.cpp成以下内容

```cpp
#include "imgui.h"
#include "imgui_impl_sdl2.h"

// SDL
#include <SDL.h>
#include <SDL_syswm.h>
#if defined(__APPLE__)
#include <TargetConditionals.h>
#endif

#if SDL_VERSION_ATLEAST(2,0,4) && !defined(__EMSCRIPTEN__) && !defined(__ANDROID__) && !(defined(__APPLE__) && TARGET_OS_IOS) && !defined(__amigaos4__)
#define SDL_HAS_CAPTURE_AND_GLOBAL_MOUSE    1
#else
#define SDL_HAS_CAPTURE_AND_GLOBAL_MOUSE    0
#endif
#define SDL_HAS_VULKAN                      SDL_VERSION_ATLEAST(2,0,6)

// SDL Data
struct ImGui_ImplSDL2_Data
{
    void*     c;
    SDL_Renderer*   Renderer;
    Uint64          Time;
    Uint32          MouseWindowID;
    int             MouseButtonsDown;
    SDL_Cursor*     MouseCursors[ImGuiMouseCursor_COUNT];
    SDL_Cursor*     LastMouseCursor;
    int             PendingMouseLeaveFrame;
    char*           ClipboardTextData;
    bool            MouseCanUseGlobalState;

    ImGui_ImplSDL2_Data()   { memset((void*)this, 0, sizeof(*this)); }
};

static ImGui_ImplSDL2_Data* ImGui_ImplSDL2_GetBackendData()
{
    return ImGui::GetCurrentContext() ? (ImGui_ImplSDL2_Data*)ImGui::GetIO().BackendPlatformUserData : nullptr;
}

static const char* ImGui_ImplSDL2_GetClipboardText(void*)
{
    return "";
}

static void ImGui_ImplSDL2_SetClipboardText(void*, const char* text)
{
    return;
}

static void ImGui_ImplSDL2_SetPlatformImeData(ImGuiViewport*, ImGuiPlatformImeData* data)
{
    if (data->WantVisible)
    {
        SDL_Rect r;
        r.x = (int)data->InputPos.x;
        r.y = (int)data->InputPos.y;
        r.w = 1;
        r.h = (int)data->InputLineHeight;
        SDL_SetTextInputRect(&r);
        SDL_StartTextInput();
    }
    else
    {
        SDL_StopTextInput();
    }
}

static ImGuiKey ImGui_ImplSDL2_KeycodeToImGuiKey(int keycode)
{
    switch (keycode)
    {
        case SDLK_TAB: return ImGuiKey_Tab;
        case SDLK_LEFT: return ImGuiKey_LeftArrow;
        case SDLK_RIGHT: return ImGuiKey_RightArrow;
        case SDLK_UP: return ImGuiKey_UpArrow;
        case SDLK_DOWN: return ImGuiKey_DownArrow;
        case SDLK_PAGEUP: return ImGuiKey_PageUp;
        case SDLK_PAGEDOWN: return ImGuiKey_PageDown;
        case SDLK_HOME: return ImGuiKey_Home;
        case SDLK_END: return ImGuiKey_End;
        case SDLK_INSERT: return ImGuiKey_Insert;
        case SDLK_DELETE: return ImGuiKey_Delete;
        case SDLK_BACKSPACE: return ImGuiKey_Backspace;
        case SDLK_SPACE: return ImGuiKey_Space;
        case SDLK_RETURN: return ImGuiKey_Enter;
        case SDLK_ESCAPE: return ImGuiKey_Escape;
        case SDLK_QUOTE: return ImGuiKey_Apostrophe;
        case SDLK_COMMA: return ImGuiKey_Comma;
        case SDLK_MINUS: return ImGuiKey_Minus;
        case SDLK_PERIOD: return ImGuiKey_Period;
        case SDLK_SLASH: return ImGuiKey_Slash;
        case SDLK_SEMICOLON: return ImGuiKey_Semicolon;
        case SDLK_EQUALS: return ImGuiKey_Equal;
        case SDLK_LEFTBRACKET: return ImGuiKey_LeftBracket;
        case SDLK_BACKSLASH: return ImGuiKey_Backslash;
        case SDLK_RIGHTBRACKET: return ImGuiKey_RightBracket;
        case SDLK_BACKQUOTE: return ImGuiKey_GraveAccent;
        case SDLK_CAPSLOCK: return ImGuiKey_CapsLock;
        case SDLK_SCROLLLOCK: return ImGuiKey_ScrollLock;
        case SDLK_NUMLOCKCLEAR: return ImGuiKey_NumLock;
        case SDLK_PRINTSCREEN: return ImGuiKey_PrintScreen;
        case SDLK_PAUSE: return ImGuiKey_Pause;
        case SDLK_KP_0: return ImGuiKey_Keypad0;
        case SDLK_KP_1: return ImGuiKey_Keypad1;
        case SDLK_KP_2: return ImGuiKey_Keypad2;
        case SDLK_KP_3: return ImGuiKey_Keypad3;
        case SDLK_KP_4: return ImGuiKey_Keypad4;
        case SDLK_KP_5: return ImGuiKey_Keypad5;
        case SDLK_KP_6: return ImGuiKey_Keypad6;
        case SDLK_KP_7: return ImGuiKey_Keypad7;
        case SDLK_KP_8: return ImGuiKey_Keypad8;
        case SDLK_KP_9: return ImGuiKey_Keypad9;
        case SDLK_KP_PERIOD: return ImGuiKey_KeypadDecimal;
        case SDLK_KP_DIVIDE: return ImGuiKey_KeypadDivide;
        case SDLK_KP_MULTIPLY: return ImGuiKey_KeypadMultiply;
        case SDLK_KP_MINUS: return ImGuiKey_KeypadSubtract;
        case SDLK_KP_PLUS: return ImGuiKey_KeypadAdd;
        case SDLK_KP_ENTER: return ImGuiKey_KeypadEnter;
        case SDLK_KP_EQUALS: return ImGuiKey_KeypadEqual;
        case SDLK_LCTRL: return ImGuiKey_LeftCtrl;
        case SDLK_LSHIFT: return ImGuiKey_LeftShift;
        case SDLK_LALT: return ImGuiKey_LeftAlt;
        case SDLK_LGUI: return ImGuiKey_LeftSuper;
        case SDLK_RCTRL: return ImGuiKey_RightCtrl;
        case SDLK_RSHIFT: return ImGuiKey_RightShift;
        case SDLK_RALT: return ImGuiKey_RightAlt;
        case SDLK_RGUI: return ImGuiKey_RightSuper;
        case SDLK_APPLICATION: return ImGuiKey_Menu;
        case SDLK_0: return ImGuiKey_0;
        case SDLK_1: return ImGuiKey_1;
        case SDLK_2: return ImGuiKey_2;
        case SDLK_3: return ImGuiKey_3;
        case SDLK_4: return ImGuiKey_4;
        case SDLK_5: return ImGuiKey_5;
        case SDLK_6: return ImGuiKey_6;
        case SDLK_7: return ImGuiKey_7;
        case SDLK_8: return ImGuiKey_8;
        case SDLK_9: return ImGuiKey_9;
        case SDLK_a: return ImGuiKey_A;
        case SDLK_b: return ImGuiKey_B;
        case SDLK_c: return ImGuiKey_C;
        case SDLK_d: return ImGuiKey_D;
        case SDLK_e: return ImGuiKey_E;
        case SDLK_f: return ImGuiKey_F;
        case SDLK_g: return ImGuiKey_G;
        case SDLK_h: return ImGuiKey_H;
        case SDLK_i: return ImGuiKey_I;
        case SDLK_j: return ImGuiKey_J;
        case SDLK_k: return ImGuiKey_K;
        case SDLK_l: return ImGuiKey_L;
        case SDLK_m: return ImGuiKey_M;
        case SDLK_n: return ImGuiKey_N;
        case SDLK_o: return ImGuiKey_O;
        case SDLK_p: return ImGuiKey_P;
        case SDLK_q: return ImGuiKey_Q;
        case SDLK_r: return ImGuiKey_R;
        case SDLK_s: return ImGuiKey_S;
        case SDLK_t: return ImGuiKey_T;
        case SDLK_u: return ImGuiKey_U;
        case SDLK_v: return ImGuiKey_V;
        case SDLK_w: return ImGuiKey_W;
        case SDLK_x: return ImGuiKey_X;
        case SDLK_y: return ImGuiKey_Y;
        case SDLK_z: return ImGuiKey_Z;
        case SDLK_F1: return ImGuiKey_F1;
        case SDLK_F2: return ImGuiKey_F2;
        case SDLK_F3: return ImGuiKey_F3;
        case SDLK_F4: return ImGuiKey_F4;
        case SDLK_F5: return ImGuiKey_F5;
        case SDLK_F6: return ImGuiKey_F6;
        case SDLK_F7: return ImGuiKey_F7;
        case SDLK_F8: return ImGuiKey_F8;
        case SDLK_F9: return ImGuiKey_F9;
        case SDLK_F10: return ImGuiKey_F10;
        case SDLK_F11: return ImGuiKey_F11;
        case SDLK_F12: return ImGuiKey_F12;
    }
    return ImGuiKey_None;
}

static void ImGui_ImplSDL2_UpdateKeyModifiers(SDL_Keymod sdl_key_mods)
{
    ImGuiIO& io = ImGui::GetIO();
    io.AddKeyEvent(ImGuiMod_Ctrl, (sdl_key_mods & KMOD_CTRL) != 0);
    io.AddKeyEvent(ImGuiMod_Shift, (sdl_key_mods & KMOD_SHIFT) != 0);
    io.AddKeyEvent(ImGuiMod_Alt, (sdl_key_mods & KMOD_ALT) != 0);
    io.AddKeyEvent(ImGuiMod_Super, (sdl_key_mods & KMOD_GUI) != 0);
}

bool ImGui_ImplSDL2_ProcessEvent(const SDL_Event* event)
{
    ImGuiIO& io = ImGui::GetIO();
    ImGui_ImplSDL2_Data* bd = ImGui_ImplSDL2_GetBackendData();

    switch (event->type)
    {
        case SDL_MOUSEMOTION:
        {
            ImVec2 mouse_pos((float)event->motion.x, (float)event->motion.y);
            io.AddMousePosEvent(mouse_pos.x, mouse_pos.y);
            return true;
        }
        case SDL_MOUSEWHEEL:
        {
            //IMGUI_DEBUG_LOG("wheel %.2f %.2f, precise %.2f %.2f\\n", (float)event->wheel.x, (float)event->wheel.y, event->wheel.preciseX, event->wheel.preciseY);
#if SDL_VERSION_ATLEAST(2,0,18) // If this fails to compile on Emscripten: update to latest Emscripten!
            float wheel_x = -event->wheel.preciseX;
            float wheel_y = event->wheel.preciseY;
#else
            float wheel_x = -(float)event->wheel.x;
            float wheel_y = (float)event->wheel.y;
#endif
#ifdef __EMSCRIPTEN__
            wheel_x /= 100.0f;
#endif
            io.AddMouseWheelEvent(wheel_x, wheel_y);
            return true;
        }
        case SDL_MOUSEBUTTONDOWN:
        case SDL_MOUSEBUTTONUP:
        {
            int mouse_button = -1;
            if (event->button.button == SDL_BUTTON_LEFT) { mouse_button = 0; }
            if (event->button.button == SDL_BUTTON_RIGHT) { mouse_button = 1; }
            if (event->button.button == SDL_BUTTON_MIDDLE) { mouse_button = 2; }
            if (event->button.button == SDL_BUTTON_X1) { mouse_button = 3; }
            if (event->button.button == SDL_BUTTON_X2) { mouse_button = 4; }
            if (mouse_button == -1)
                break;
            io.AddMouseButtonEvent(mouse_button, (event->type == SDL_MOUSEBUTTONDOWN));
            bd->MouseButtonsDown = (event->type == SDL_MOUSEBUTTONDOWN) ? (bd->MouseButtonsDown | (1 << mouse_button)) : (bd->MouseButtonsDown & ~(1 << mouse_button));
            return true;
        }
        case SDL_TEXTINPUT:
        {
            io.AddInputCharactersUTF8(event->text.text);
            return true;
        }
        case SDL_KEYDOWN:
        case SDL_KEYUP:
        {
            ImGui_ImplSDL2_UpdateKeyModifiers((SDL_Keymod)event->key.keysym.mod);
            ImGuiKey key = ImGui_ImplSDL2_KeycodeToImGuiKey(event->key.keysym.sym);
            io.AddKeyEvent(key, (event->type == SDL_KEYDOWN));
            io.SetKeyEventNativeData(key, event->key.keysym.sym, event->key.keysym.scancode, event->key.keysym.scancode); // To support legacy indexing (<1.87 user code). Legacy backend uses SDLK_*** as indices to IsKeyXXX() functions.
            return true;
        }
    }
    return false;
}

static bool ImGui_ImplSDL2_Init(SDL_Window* window, SDL_Renderer* renderer)
{
    ImGuiIO& io = ImGui::GetIO();
    IM_ASSERT(io.BackendPlatformUserData == nullptr && "Already initialized a platform backend!");

    bool mouse_can_use_global_state = false;
#if SDL_HAS_CAPTURE_AND_GLOBAL_MOUSE
    const char* sdl_backend = SDL_GetCurrentVideoDriver();
    const char* global_mouse_whitelist[] = { "windows", "cocoa", "x11", "DIVE", "VMAN" };
    for (int n = 0; n < IM_ARRAYSIZE(global_mouse_whitelist); n++)
        if (strncmp(sdl_backend, global_mouse_whitelist[n], strlen(global_mouse_whitelist[n])) == 0)
            mouse_can_use_global_state = true;
#endif

    // Setup backend capabilities flags
    ImGui_ImplSDL2_Data* bd = IM_NEW(ImGui_ImplSDL2_Data)();
    io.BackendPlatformUserData = (void*)bd;
    io.BackendPlatformName = "imgui_impl_sdl2";
    io.BackendFlags |= ImGuiBackendFlags_HasMouseCursors;       // We can honor GetMouseCursor() values (optional)
    io.BackendFlags |= ImGuiBackendFlags_HasSetMousePos;        // We can honor io.WantSetMousePos requests (optional, rarely used)

    bd->c = window;
    bd->Renderer = renderer;
    bd->MouseCanUseGlobalState = mouse_can_use_global_state;

    io.SetClipboardTextFn = ImGui_ImplSDL2_SetClipboardText;
    io.GetClipboardTextFn = ImGui_ImplSDL2_GetClipboardText;
    io.ClipboardUserData = nullptr;
    io.SetPlatformImeDataFn = ImGui_ImplSDL2_SetPlatformImeData;

    // Load mouse cursors
    bd->MouseCursors[ImGuiMouseCursor_Arrow] = SDL_CreateSystemCursor(SDL_SYSTEM_CURSOR_ARROW);
    bd->MouseCursors[ImGuiMouseCursor_TextInput] = SDL_CreateSystemCursor(SDL_SYSTEM_CURSOR_IBEAM);
    bd->MouseCursors[ImGuiMouseCursor_ResizeAll] = SDL_CreateSystemCursor(SDL_SYSTEM_CURSOR_SIZEALL);
    bd->MouseCursors[ImGuiMouseCursor_ResizeNS] = SDL_CreateSystemCursor(SDL_SYSTEM_CURSOR_SIZENS);
    bd->MouseCursors[ImGuiMouseCursor_ResizeEW] = SDL_CreateSystemCursor(SDL_SYSTEM_CURSOR_SIZEWE);
    bd->MouseCursors[ImGuiMouseCursor_ResizeNESW] = SDL_CreateSystemCursor(SDL_SYSTEM_CURSOR_SIZENESW);
    bd->MouseCursors[ImGuiMouseCursor_ResizeNWSE] = SDL_CreateSystemCursor(SDL_SYSTEM_CURSOR_SIZENWSE);
    bd->MouseCursors[ImGuiMouseCursor_Hand] = SDL_CreateSystemCursor(SDL_SYSTEM_CURSOR_HAND);
    bd->MouseCursors[ImGuiMouseCursor_NotAllowed] = SDL_CreateSystemCursor(SDL_SYSTEM_CURSOR_NO);

    // Set platform dependent data in viewport
    // Our mouse update function expect PlatformHandle to be filled for the main viewport
    ImGuiViewport* main_viewport = ImGui::GetMainViewport();
    main_viewport->PlatformHandleRaw = nullptr;

#ifdef SDL_HINT_MOUSE_FOCUS_CLICKTHROUGH
    SDL_SetHint(SDL_HINT_MOUSE_FOCUS_CLICKTHROUGH, "1");
#endif

    // From 2.0.18: Enable native IME.
    // IMPORTANT: This is used at the time of SDL_CreateWindow() so this will only affects secondary windows, if any.
    // For the main window to be affected, your application needs to call this manually before calling SDL_CreateWindow().
#ifdef SDL_HINT_IME_SHOW_UI
    SDL_SetHint(SDL_HINT_IME_SHOW_UI, "1");
#endif

    // From 2.0.22: Disable auto-capture, this is preventing drag and drop across multiple windows (see #5710)
#ifdef SDL_HINT_MOUSE_AUTO_CAPTURE
    SDL_SetHint(SDL_HINT_MOUSE_AUTO_CAPTURE, "0");
#endif

    return true;
}

bool ImGui_ImplSDL2_InitForOpenGL(SDL_Window* window, void* sdl_gl_context)
{
    IM_UNUSED(sdl_gl_context); // Viewport branch will need this.
    return ImGui_ImplSDL2_Init(window, nullptr);
}

bool ImGui_ImplSDL2_InitForVulkan(SDL_Window* window)
{
#if !SDL_HAS_VULKAN
    IM_ASSERT(0 && "Unsupported");
#endif
    return ImGui_ImplSDL2_Init(window, nullptr);
}

bool ImGui_ImplSDL2_InitForD3D(SDL_Window* window)
{
#if !defined(_WIN32)
    IM_ASSERT(0 && "Unsupported");
#endif
    return ImGui_ImplSDL2_Init(window, nullptr);
}

bool ImGui_ImplSDL2_InitForMetal(SDL_Window* window)
{
    return ImGui_ImplSDL2_Init(window, nullptr);
}

bool ImGui_ImplSDL2_InitForSDLRenderer(SDL_Window* window, SDL_Renderer* renderer)
{
    return ImGui_ImplSDL2_Init(window, renderer);
}

void ImGui_ImplSDL2_Shutdown()
{
    ImGui_ImplSDL2_Data* bd = ImGui_ImplSDL2_GetBackendData();
    IM_ASSERT(bd != nullptr && "No platform backend to shutdown, or already shutdown?");
    ImGuiIO& io = ImGui::GetIO();

    if (bd->ClipboardTextData)
        SDL_free(bd->ClipboardTextData);
    for (ImGuiMouseCursor cursor_n = 0; cursor_n < ImGuiMouseCursor_COUNT; cursor_n++)
        SDL_FreeCursor(bd->MouseCursors[cursor_n]);
    bd->LastMouseCursor = NULL;

    io.BackendPlatformName = nullptr;
    io.BackendPlatformUserData = nullptr;
    IM_DELETE(bd);
}

extern void SDLEXT_GetBufferSize(int *w,int *h);

void ImGui_ImplSDL2_NewFrame()
{
    ImGui_ImplSDL2_Data* bd = ImGui_ImplSDL2_GetBackendData();
    IM_ASSERT(bd != nullptr && "Did you call ImGui_ImplSDL2_Init()?");
    ImGuiIO& io = ImGui::GetIO();

    // Setup display size (every frame to accommodate for window resizing)
    int w, h;
    int display_w, display_h;

    SDLEXT_GetBufferSize(&w,&h);
    SDLEXT_GetBufferSize(&display_w,&display_h);
        
    io.DisplaySize = ImVec2((float)w, (float)h);
    if (w > 0 && h > 0)
        io.DisplayFramebufferScale = ImVec2((float)display_w / w, (float)display_h / h);

    // Setup time step (we don't use SDL_GetTicks() because it is using millisecond resolution)
    static Uint64 frequency = SDL_GetPerformanceFrequency();
    Uint64 current_time = SDL_GetPerformanceCounter();
    io.DeltaTime = bd->Time > 0 ? (float)((double)(current_time - bd->Time) / frequency) : (float)(1.0f / 60.0f);
    bd->Time = current_time;

    if (bd->PendingMouseLeaveFrame && bd->PendingMouseLeaveFrame >= ImGui::GetFrameCount() && bd->MouseButtonsDown == 0)
    {
        bd->MouseWindowID = 0;
        bd->PendingMouseLeaveFrame = 0;
        io.AddMousePosEvent(-FLT_MAX, -FLT_MAX);
    }
}
```

编译打包

```bash
set INCS=-Iinclude -I../SDL2/include/SDL2
set SRCS=src/imgui.cpp src/imgui_demo.cpp src/imgui_draw.cpp src/imgui_impl_sdl2.cpp src/imgui_impl_sdlrenderer.cpp src/imgui_tables.cpp src/imgui_widgets.cpp
%CXX% -c %SRCS% %INCS%

del lib\\* /q
mkdir lib
move *.o lib > nul
cd lib

%AR% rcs libimgui.a *.o
del *.o /q
cd ..
```

## 集成进可执行文件

由于没使用ndk-build,使用的时候加上这个源文件解决linker报错

```bash
#include <stdio.h>
#include <SDL2/SDL.h>
#include <unistd.h>

extern "C" void Android_ActivityMutex_Lock_Running()
{
	return;
}
extern "C" void Android_ActivityMutex_Unlock()
{
	return;
}
extern "C" SDL_bool SDL_IsAndroidTablet(void)
{
	return SDL_FALSE;
}
extern "C" const char *SDL_AndroidGetInternalStoragePath(void)
{
	return "/sdcard/";
}
extern "C" int Android_JNI_FileOpen(SDL_RWops *ctx,
                         const char *fileName, const char *mode)
						 {
							 return 0;
						 }

extern "C" Sint64 Android_JNI_FileSize(SDL_RWops *ctx)
{
	return 0;
}
extern "C" Sint64 Android_JNI_FileSeek(SDL_RWops *ctx, Sint64 offset, int whence)
{
	return 0;
}
extern "C" int Android_JNI_FileClose(SDL_RWops *ctx)
{
	return 0;
}
extern "C" size_t Android_JNI_FileRead(SDL_RWops *ctx, void *buffer,
                            size_t size, size_t maxnum)
{
	return 0;
}
extern "C" size_t Android_JNI_FileWrite(SDL_RWops *ctx, const void *buffer,
                             size_t size, size_t num)
							 {
								 return 0;
							 }
extern "C" int SDL_GetAndroidSDKVersion(void)
{
    return 33;
}

extern "C" void Android_JNI_GetManifestEnvironmentVariables(void)
{
	return;
}
extern "C" SDL_bool Android_JNI_ShouldMinimizeOnFocusLoss()
{
	return SDL_FALSE;
}
extern "C" int PLATFORM_hid_init(void)
{
	return 0;
}
extern "C" int PLATFORM_hid_exit(void)
{
	return 0;
}
extern "C" void* PLATFORM_hid_enumerate(unsigned short vendor_id, unsigned short product_id)
{
	return NULL;
}
extern "C" void PLATFORM_hid_free_enumeration(struct hid_device_info *devs)
{
	return;
}
extern "C" void *PLATFORM_hid_open(unsigned short vendor_id, unsigned short product_id, const wchar_t *serial_number)
{
	// TODO: Implement
	return NULL;
}
extern "C" void* PLATFORM_hid_open_path(const char *path, int bExclusive)
{
	return NULL;
}
extern "C" void* PLATFORM_hid_write(void *device, const unsigned char *data, size_t length)
{
	return NULL;
}
extern "C" void* PLATFORM_hid_read_timeout()
{
	return NULL;
}

extern "C" void* PLATFORM_hid_read()
{
	return NULL;
}
extern "C" void* PLATFORM_hid_set_nonblocking()
{
	return NULL;
}
extern "C" void* PLATFORM_hid_send_feature_report()
{
	return NULL;
}
extern "C" void* PLATFORM_hid_get_feature_report()
{
	return NULL;
}
extern "C" void* PLATFORM_hid_close()
{
	return NULL;
}
extern "C" void* PLATFORM_hid_get_manufacturer_string()
{
	return NULL;
}
extern "C" void* PLATFORM_hid_get_product_string()
{
	return NULL;
}
extern "C" void* PLATFORM_hid_get_serial_number_string()
{
	return NULL;
}
extern "C" void* PLATFORM_hid_get_indexed_string()
{
	return NULL;
}
extern "C" void* PLATFORM_hid_error()
{
	return NULL;
}
extern "C" void* Android_JNI_SetupThread()
{
	return NULL;
}
```

添加源码,设置imgui获取到的屏幕大小

```cpp
void SDLEXT_GetBufferSize(int *w,int *h)
{
	*w=g_width;
	*h=g_heigh;
}
```

例子

```cpp
int g_width=1240;
int g_heigh=2772;
int main(int argc, char* argv[])
{
	SDL_SetMainReady();
	if (SDL_Init(SDL_INIT_VIDEO) < 0) {
        printf("SDL could not initialize! SDL_Error: %s\\n", SDL_GetError());
        return 1;
	}

	SDL_Surface *surface = SDL_CreateRGBSurfaceWithFormat(0, g_width, g_heigh, 32, SDL_PIXELFORMAT_ARGB8888);
	SDL_Renderer *renderer = SDL_CreateSoftwareRenderer(surface);
	if (renderer == NULL)
	{
		return 0;
	}
	
	IMGUI_CHECKVERSION();
	ImGui::CreateContext();
	ImGuiIO &io = ImGui::GetIO();

	ImGui::StyleColorsDark();

	ImGui_ImplSDL2_InitForSDLRenderer(0, renderer);
	ImGui_ImplSDLRenderer_Init(renderer);
	uint32_t *imgBuffer = (uint32_t *)malloc(g_width*g_heigh*4);

	bool show_demo_window = true;
	bool show_another_window = false;
	ImVec4 clear_color = ImVec4(0.00f, 0.00f, 0.00f, 0.00f);
	
	while(1)
	{
		ImGui_ImplSDLRenderer_NewFrame();
		ImGui_ImplSDL2_NewFrame();
		ImGui::NewFrame();
		
		SDL_RenderSetScale(renderer, io.DisplayFramebufferScale.x, io.DisplayFramebufferScale.y);
		SDL_SetRenderDrawColor(renderer, (Uint8)(clear_color.x * 255), (Uint8)(clear_color.y * 255), (Uint8)(clear_color.z * 255), (Uint8)(clear_color.w * 255));
		SDL_RenderClear(renderer);

		ImDrawList *list = ImGui::GetForegroundDrawList();
		ImGui::Begin("huaji0901.cn");
		ImGui::Text("huajihuajihuaji");
		ImGui::End();

		ImGui::Render();
		ImGui_ImplSDLRenderer_RenderDrawData(ImGui::GetDrawData());

		uint32_t *pixes = (uint32_t *)surface->pixels;
		printf("pixes = %p\\r\\n",pixes);
		printf("pitch = %d\\r\\n",surface->pitch);
		fflush(stdout);
		for(int i=0;i<g_heigh;i++)
		{
			uint32_t * srcRowPixes = (uint32_t *)((uint8_t*)pixes + (surface->pitch * i));
			uint32_t * dstRowPixes = (uint32_t *)((uint8_t*)imgBuffer + (g_width * 4 * i));
			memcpy(dstRowPixes,srcRowPixes,g_width * 4);
		}
		std::this_thread::sleep_for(std::chrono::milliseconds(10));
	}
```