# Pre-requisites
Windows 7 SP1 VM:
- x86 Visual C++ Redistributable 2015

Windows 11 Build VM toolchain:
- Visual Studio Build Tools 2022
- CMake
- Python 3.15.2
  - PyYAML
  - Mako >= 0.8.0
  - pyparsing
  - setuptools
  - packaging
  - MarkupSafe
- Microsoft Visual Studio Installer
  - Microsoft Visual C++ v14 Redistributable (x86) - 14.51.36247
- Meson Build System
- Windows Software Development Kit - Windows 10.0.19041.5609
- Windows Software Development Kit - Windows 10.0.22621.5040
- Windows Software Development Kit - Windows 10.26100.7705
- Flex/Bison - Latest release

# Build LLVM x86
```
cd /d C:\llvm-13.0.1

cmake -B build-win32 ^
  -G Ninja ^
  -DCMAKE_BUILD_TYPE=Release ^
  -DCMAKE_INSTALL_PREFIX=C:/LLVM13-x86 ^
  -DLLVM_TARGETS_TO_BUILD=X86 ^
  -DLLVM_ENABLE_PROJECTS="" ^
  -DLLVM_BUILD_TOOLS=ON ^
  -DLLVM_INCLUDE_TESTS=OFF ^
  -DLLVM_INCLUDE_EXAMPLES=OFF ^
  -DLLVM_ENABLE_TERMINFO=OFF ^
  -DLLVM_ENABLE_ZLIB=OFF ^
  -DLLVM_USE_CRT_RELEASE=MT

cmake --build build-win32 --parallel 4
cmake --install build-win32
```

# Build Mesa3D softpipe and llvmpipe
1. Open up Developer Command Prompt for Visual Studio
2. Unpack the latest Mesa3D library
`tar -xvf mesa-21.3.9.tar.xf`
3. If compiling llvmpipe, set LLVM to be visible
```
set "PATH=%PATH%;C:\msys64\usr\bin"
set LLVM=C:\LLVM13-x86
set CMAKE_PREFIX_PATH=C:\LLVM13-x86
```
4. Then configure Mesa to have the renderers of softpipe and llvmpipe
```
cd /d C:\mesa-26.2.5
meson setup build-softpipe-new ^
  -Dplatforms=windows ^
  -Dgallium-drivers=softpipe ^
  -Dvulkan-drivers= ^
  -Dllvm=disabled ^
  -Db_vscrt=mt

meson configure build-softpipe -Db_vscrt=mt

cd build-softpipe
ninja
```
5. Verify the dependencies with dumpbin and find their path:
```
Get-ChildItem "C:\Users\TYGE\Downloads\mesa-21.3.9\build-llvmpipe" -Recurse -Include *.dll,*.exe | Select-Object Name,DirectoryName

dumpbin /dependents PATH_TO_DLL
dumpbin /headers PATH_TO_DLL
```

6. Test the executables on Windows 7 SP1 with a script or the product

Test script example:
```

#include <windows.h>
#include <GL/gl.h>
#include <cstdio>

int main() {
    HDC dc = GetDC(nullptr);

    PIXELFORMATDESCRIPTOR pfd = {};
    pfd.nSize = sizeof(pfd);
    pfd.nVersion = 1;
    pfd.dwFlags = PFD_DRAW_TO_WINDOW | PFD_SUPPORT_OPENGL;
    pfd.iPixelType = PFD_TYPE_RGBA;
    pfd.cColorBits = 24;
    pfd.cDepthBits = 24;
    pfd.iLayerType = PFD_MAIN_PLANE;

    HWND window = CreateWindowA(
        "STATIC", "Mesa Test", WS_OVERLAPPEDWINDOW,
        0, 0, 640, 480, nullptr, nullptr,
        GetModuleHandle(nullptr), nullptr
    );

    if (!window) return 1;

    ReleaseDC(nullptr, dc);
    dc = GetDC(window);

    int format = ChoosePixelFormat(dc, &pfd);
    if (!format || !SetPixelFormat(dc, format, &pfd)) {
        printf("Pixel format failed: %lu\n", GetLastError());
        return 1;
    }

    HGLRC context = wglCreateContext(dc);
    if (!context || !wglMakeCurrent(dc, context)) {
        printf("OpenGL context failed: %lu\n", GetLastError());
        return 1;
    }

    printf("Vendor:   %s\n", glGetString(GL_VENDOR));
    printf("Renderer: %s\n", glGetString(GL_RENDERER));
    printf("Version:  %s\n", glGetString(GL_VERSION));

    wglMakeCurrent(nullptr, nullptr);
    wglDeleteContext(context);
    ReleaseDC(window, dc);
    DestroyWindow(window);

    return 0;
}

```

Save as MesaTest.cpp

Then compile it statically:
```
cl /MT /EHsc /O2 MesaTest.cpp opengl32.lib gdi32.lib user32.lib /Fe:MesaTest.exe
```

# If need to cleanup builddir on meson:
Replace `build-win7-x86` with the directory specified in `meson setup`
```
ninja -C build-win7-x86 -t clean
ninja -C build-win7-x86
```
