# Upstream Build System Improvements for Native Windows (MinGW-w64)

Hi devs, just dumping what I found trying to make the builder script. Have fun (or don't, I'm probably the only idiot using Ardour on Windows for live productions).

# ardour.keys breaks native Windows builds
In gtk2_ardour/wscript, a_rule uses `>${TGT}`. On Windows, Waf runs commands with redirects through cmd.exe. The default value for --windows-key in root wscript is `Mod4><Super`. In cmd.exe, double quotes do not escape redirects, so `><` gets parsed as invalid syntax and crashes the build with: `< was unexpected at this time`.

Fix: Default --windows-key to Mod4 when --dist-target=mingw is set, or run fmt-bindings.py directly as a Python method task instead of using shell redirection.

# Waf auto-detects MSVC over GCC when --dist-target=mingw is passed
On Windows, Waf checks msvc before gcc. If Visual Studio is installed on the machine, Waf binds cl.exe even when the user explicitly passed --dist-target=mingw. cl.exe then chokes on MinGW flags like -mstackrealign and fails immediately.

Fix: If --dist-target=mingw is passed, force Waf to check gcc and g++ first so users do not have to manually pass --check-c-compiler=gcc and --check-cxx-compiler=g++ on the command line.

# SIMD assembly check tests the compiler binary name
In root wscript and libs/ardour/wscript, the build checks if 'x86_64-w64' is in str(conf.env['CC']) to enable 64-bit Windows AVX/SSE assembly. In a native MSYS2 environment, the compiler is often invoked simply as gcc.exe. This fails the regex and silently drops all 64-bit SIMD optimizations from the build.

Fix: Check the target architecture or the `__x86_64__` preprocessor macro directly instead of regex matching the compiler executable name.

# CC and CXX without .exe breaks Waf on Windows
Setting CC=gcc or CC=x86_64-w64-mingw32-gcc in MSYS2 causes Waf to check for a file literally named gcc. It fails with "Program ['gcc'] is not executable" because it does not append .exe when verifying the binary on Windows.

# MSYS2 PortAudio with ASIO
MSYS2 provides mingw-w64-x86_64-portaudio-asio in their official repos. It includes pa_asio.h and working ASIO binaries out of the box, which avoids needing to download Steinberg SDK headers manually.