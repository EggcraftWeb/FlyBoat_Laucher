#define UNICODE
#define _UNICODE

#include <windows.h>
#include <commctrl.h>
#include <urlmon.h>
#include <shellapi.h>

#include <filesystem>
#include <string>
#include <vector>
#include <thread>
#include <fstream>
#include <atomic>
#include <cwctype>

#pragma comment(lib, "Comctl32.lib")
#pragma comment(lib, "Urlmon.lib")
#pragma comment(lib, "Shell32.lib")

namespace fs = std::filesystem;

// ============================================================
// CONFIGURATION
// ============================================================

static const wchar_t* MC_VERSION = L"1.5.2";
static const wchar_t* MC_USERNAME = L"Steve";

static const fs::path BASE =
L"C:\\Minecraft152Launcher";

static const fs::path ROOT =
BASE / L"minecraft";

static const fs::path LOG_FILE =
BASE / L"launcher_debug.txt";

static const fs::path MINECRAFT_LOG =
BASE / L"minecraft_console.log";

static const fs::path BATCH_FILE =
BASE / L"launch_minecraft.bat";

static const fs::path VERSION_DIR =
ROOT / L"versions" / L"1.5.2";

static const fs::path LIBRARIES =
ROOT / L"libraries";

static const fs::path NATIVES =
VERSION_DIR / L"natives";

static const fs::path ASSETS =
ROOT / L"assets";

static const fs::path CLIENT_JAR =
VERSION_DIR / L"1.5.2.jar";

// ============================================================
// DOWNLOAD URLS
// ============================================================

static const wchar_t* CLIENT_URL =
L"https://launcher.mojang.com/v1/objects/"
L"465378c9dc2f779ae1d6e8046ebc46fb53a57968/"
L"client.jar";

static const wchar_t* LAUNCHWRAPPER_URL =
L"https://libraries.minecraft.net/"
L"net/minecraft/launchwrapper/1.5/"
L"launchwrapper-1.5.jar";

static const wchar_t* JOPT_URL =
L"https://libraries.minecraft.net/"
L"net/sf/jopt-simple/jopt-simple/4.5/"
L"jopt-simple-4.5.jar";

static const wchar_t* ASM_URL =
L"https://libraries.minecraft.net/"
L"org/ow2/asm/asm-all/4.1/"
L"asm-all-4.1.jar";

static const wchar_t* JINPUT_URL =
L"https://libraries.minecraft.net/"
L"net/java/jinput/jinput/2.0.5/"
L"jinput-2.0.5.jar";

static const wchar_t* JUTILS_URL =
L"https://libraries.minecraft.net/"
L"net/java/jutils/jutils/1.0.0/"
L"jutils-1.0.0.jar";

static const wchar_t* LWJGL_URL =
L"https://libraries.minecraft.net/"
L"org/lwjgl/lwjgl/lwjgl/2.9.0/"
L"lwjgl-2.9.0.jar";

static const wchar_t* LWJGL_UTIL_URL =
L"https://libraries.minecraft.net/"
L"org/lwjgl/lwjgl/lwjgl_util/2.9.0/"
L"lwjgl_util-2.9.0.jar";

static const wchar_t* LWJGL_NATIVE_URL =
L"https://libraries.minecraft.net/"
L"org/lwjgl/lwjgl/lwjgl-platform/2.9.0/"
L"lwjgl-platform-2.9.0-natives-windows.jar";

static const wchar_t* JINPUT_NATIVE_URL =
L"https://libraries.minecraft.net/"
L"net/java/jinput/jinput-platform/2.0.5/"
L"jinput-platform-2.0.5-natives-windows.jar";

// ============================================================
// LIBRARY PATHS
// ============================================================

static const fs::path LAUNCHWRAPPER =
LIBRARIES /
L"net/minecraft/launchwrapper/1.5/"
L"launchwrapper-1.5.jar";

static const fs::path JOPT =
LIBRARIES /
L"net/sf/jopt-simple/4.5/"
L"jopt-simple-4.5.jar";

static const fs::path ASM =
LIBRARIES /
L"org/ow2/asm/asm-all/4.1/"
L"asm-all-4.1.jar";

static const fs::path JINPUT =
LIBRARIES /
L"net/java/jinput/jinput/2.0.5/"
L"jinput-2.0.5.jar";

static const fs::path JUTILS =
LIBRARIES /
L"net/java/jutils/jutils/1.0.0/"
L"jutils-1.0.0.jar";

static const fs::path LWJGL =
LIBRARIES /
L"org/lwjgl/lwjgl/lwjgl/2.9.0/"
L"lwjgl-2.9.0.jar";

static const fs::path LWJGL_UTIL =
LIBRARIES /
L"org/lwjgl/lwjgl/lwjgl_util/2.9.0/"
L"lwjgl_util-2.9.0.jar";

static const fs::path LWJGL_NATIVE =
LIBRARIES /
L"org/lwjgl/lwjgl/lwjgl-platform/2.9.0/"
L"lwjgl-platform-2.9.0-natives-windows.jar";

static const fs::path JINPUT_NATIVE =
LIBRARIES /
L"net/java/jinput/jinput-platform/2.0.5/"
L"jinput-platform-2.0.5-natives-windows.jar";

// ============================================================
// WINDOWS MESSAGES
// ============================================================

#define WM_APP_STATUS   (WM_APP + 1)
#define WM_APP_PROGRESS (WM_APP + 2)
#define WM_APP_FINISH   (WM_APP + 3)

// ============================================================
// GLOBAL UI
// ============================================================

HWND gWindow = nullptr;
HWND gStatus = nullptr;
HWND gProgress = nullptr;
HWND gInstall = nullptr;
HWND gPlay = nullptr;
HWND gOpenLog = nullptr;

std::atomic<bool> gBusy(false);

// ============================================================
// DIRECTORIES
// ============================================================

bool CreateDirectories()
{
    try
    {
        fs::create_directories(BASE);
        fs::create_directories(ROOT);
        fs::create_directories(VERSION_DIR);
        fs::create_directories(LIBRARIES);
        fs::create_directories(NATIVES);
        fs::create_directories(ASSETS);

        return true;
    }
    catch (...)
    {
        return false;
    }
}

// ============================================================
// LOGGING
// ============================================================

void Log(const std::wstring& text)
{
    try
    {
        fs::create_directories(BASE);

        std::wofstream file(
            LOG_FILE,
            std::ios::app
        );

        if (file)
        {
            file << text << L"\n";
        }
    }
    catch (...)
    {
    }
}

void StartLog()
{
    try
    {
        fs::create_directories(BASE);

        std::wofstream file(
            LOG_FILE,
            std::ios::trunc
        );

        if (file)
        {
            file
                << L"Minecraft 1.5.2 Launcher\n"
                << L"==========================\n"
                << L"Version: 1.5.2\n"
                << L"Username: Steve\n"
                << L"RAM: 2 GB\n"
                << L"Base: "
                << BASE.wstring()
                << L"\n\n";
        }
    }
    catch (...)
    {
    }
}

// ============================================================
// UI
// ============================================================

void SetStatus(const std::wstring& text)
{
    if (gStatus)
    {
        SetWindowTextW(
            gStatus,
            text.c_str()
        );
    }
}

void Progress(int value)
{
    if (!gProgress)
        return;

    if (value < 0)
        value = 0;

    if (value > 100)
        value = 100;

    SendMessageW(
        gProgress,
        PBM_SETPOS,
        static_cast<WPARAM>(value),
        0
    );
}

void PostStatus(
    HWND hwnd,
    const std::wstring& text)
{
    auto* copy =
        new std::wstring(text);

    PostMessageW(
        hwnd,
        WM_APP_STATUS,
        0,
        reinterpret_cast<LPARAM>(copy)
    );
}

void PostProgress(
    HWND hwnd,
    int value)
{
    PostMessageW(
        hwnd,
        WM_APP_PROGRESS,
        static_cast<WPARAM>(value),
        0
    );
}

// ============================================================
// DOWNLOAD CALLBACK
// ============================================================

class DownloadCallback :
    public IBindStatusCallback
{
private:

    HWND m_hwnd;

public:

    explicit DownloadCallback(HWND hwnd)
        : m_hwnd(hwnd)
    {
    }

    STDMETHOD(OnStartBinding)(
        DWORD,
        IBinding*)
    {
        return S_OK;
    }

    STDMETHOD(GetPriority)(
        LONG*)
    {
        return E_NOTIMPL;
    }

    STDMETHOD(OnLowResource)(
        DWORD)
    {
        return S_OK;
    }

    STDMETHOD(OnProgress)(
        ULONG current,
        ULONG maximum,
        ULONG,
        LPCWSTR)
    {
        if (maximum > 0)
        {
            int percent =
                static_cast<int>(
                    (
                        static_cast<unsigned long long>(
                            current
                            ) * 100ULL
                        ) /
                    maximum
                    );

            PostProgress(
                m_hwnd,
                percent
            );
        }

        return S_OK;
    }

    STDMETHOD(OnStopBinding)(
        HRESULT,
        LPCWSTR)
    {
        return S_OK;
    }

    STDMETHOD(GetBindInfo)(
        DWORD*,
        BINDINFO*)
    {
        return E_NOTIMPL;
    }

    STDMETHOD(OnDataAvailable)(
        DWORD,
        DWORD,
        FORMATETC*,
        STGMEDIUM*)
    {
        return S_OK;
    }

    STDMETHOD(OnObjectAvailable)(
        REFIID,
        IUnknown*)
    {
        return E_NOTIMPL;
    }

    STDMETHOD(QueryInterface)(
        REFIID riid,
        void** ppv)
    {
        if (!ppv)
            return E_POINTER;

        *ppv = nullptr;

        if (
            riid == IID_IUnknown ||
            riid == IID_IBindStatusCallback
            )
        {
            *ppv =
                static_cast<IBindStatusCallback*>(
                    this
                    );

            AddRef();

            return S_OK;
        }

        return E_NOINTERFACE;
    }

    STDMETHOD_(ULONG, AddRef)()
    {
        return 1;
    }

    STDMETHOD_(ULONG, Release)()
    {
        return 1;
    }
};

// ============================================================
// DOWNLOAD
// ============================================================

bool DownloadFile(
    HWND hwnd,
    const wchar_t* url,
    const fs::path& destination)
{
    try
    {
        fs::create_directories(
            destination.parent_path()
        );
    }
    catch (...)
    {
        Log(
            L"Could not create directory for:"
        );

        Log(
            destination.wstring()
        );

        return false;
    }

    Log(L"Downloading:");
    Log(std::wstring(url));
    Log(L"Destination:");
    Log(destination.wstring());

    DownloadCallback callback(hwnd);

    HRESULT hr =
        URLDownloadToFileW(
            nullptr,
            url,
            destination.c_str(),
            0,
            &callback
        );

    if (FAILED(hr))
    {
        Log(
            L"URLDownloadToFileW failed. HRESULT: " +
            std::to_wstring(
                static_cast<unsigned long>(hr)
            )
        );

        return false;
    }

    try
    {
        if (!fs::exists(destination))
        {
            Log(
                L"Downloaded file does not exist."
            );

            return false;
        }

        if (fs::file_size(destination) == 0)
        {
            Log(
                L"Downloaded file is empty."
            );

            return false;
        }
    }
    catch (...)
    {
        return false;
    }

    Log(L"Download successful.");

    return true;
}

// ============================================================
// DEPENDENCY
// ============================================================

bool EnsureFile(
    HWND hwnd,
    const wchar_t* name,
    const wchar_t* url,
    const fs::path& destination)
{
    try
    {
        if (
            fs::exists(destination) &&
            fs::file_size(destination) > 0
            )
        {
            Log(
                L"Already installed: " +
                std::wstring(name)
            );

            return true;
        }
    }
    catch (...)
    {
    }

    PostStatus(
        hwnd,
        L"Downloading " +
        std::wstring(name) +
        L"..."
    );

    PostProgress(
        hwnd,
        0
    );

    return DownloadFile(
        hwnd,
        url,
        destination
    );
}

// ============================================================
// JAVA 8
// ============================================================

bool IsJava8(
    const fs::path& path)
{
    if (!fs::exists(path))
        return false;

    std::wstring s =
        path.wstring();

    for (wchar_t& c : s)
        c = towlower(c);

    // Java 8
    if (
        s.find(L"jre1.8") == std::wstring::npos &&
        s.find(L"jdk1.8") == std::wstring::npos
        )
    {
        return false;
    }

    // Ignora Java instalado em Program Files (x86),
    // que normalmente é Java 32-bit.
    if (
        s.find(L"program files (x86)") !=
        std::wstring::npos
        )
    {
        return false;
    }

    return true;
}

bool FindJava8(
    fs::path& result)
{
    std::vector<fs::path> candidates;

    wchar_t javaHome[4096] = {};

    DWORD length =
        GetEnvironmentVariableW(
            L"JAVA_HOME",
            javaHome,
            4096
        );

    if (length > 0)
    {
        fs::path home(
            javaHome
        );

        candidates.push_back(
            home / L"bin" / L"java.exe"
        );

        candidates.push_back(
            home / L"jre" / L"bin" / L"java.exe"
        );
    }

    // SOMENTE Java instalado em Program Files,
    // evitando Java 32-bit de Program Files (x86).
    const fs::path roots[] =
    {
        L"C:\\Program Files\\Java"
    };

    for (const auto& root : roots)
    {
        try
        {
            if (!fs::exists(root))
                continue;

            for (
                const auto& entry :
                fs::directory_iterator(root))
            {
                if (!entry.is_directory())
                    continue;

                candidates.push_back(
                    entry.path() /
                    L"bin" /
                    L"java.exe"
                );
            }
        }
        catch (...)
        {
        }
    }

    for (const auto& candidate : candidates)
    {
        if (IsJava8(candidate))
        {
            result = candidate;

            Log(L"Java 8 64-bit found:");
            Log(result.wstring());

            return true;
        }
    }

    Log(L"Java 8 64-bit not found.");

    return false;
}

// ============================================================
// EXTRACT NATIVES
// ============================================================

bool ExtractNativeJar(
    HWND hwnd,
    const fs::path& jar)
{
    if (!fs::exists(jar))
    {
        Log(L"Native JAR missing:");
        Log(jar.wstring());

        return false;
    }

    try
    {
        fs::create_directories(
            NATIVES
        );
    }
    catch (...)
    {
        return false;
    }

    PostStatus(
        hwnd,
        L"Extracting native libraries..."
    );

    /*
        Minecraft's old native packages are ZIP/JAR files.

        Windows 10/11 normally includes tar.exe.
        It can extract ZIP/JAR archives.
    */

    std::wstring command =
        L"tar.exe -xf \"" +
        jar.wstring() +
        L"\" -C \"" +
        NATIVES.wstring() +
        L"\"";

    Log(L"Extraction command:");
    Log(command);

    std::vector<wchar_t> commandBuffer(
        command.begin(),
        command.end()
    );

    commandBuffer.push_back(
        L'\0'
    );

    STARTUPINFOW si = {};
    PROCESS_INFORMATION pi = {};

    si.cb =
        sizeof(si);

    BOOL created =
        CreateProcessW(
            nullptr,
            commandBuffer.data(),
            nullptr,
            nullptr,
            FALSE,
            CREATE_NO_WINDOW,
            nullptr,
            ROOT.c_str(),
            &si,
            &pi
        );

    if (!created)
    {
        DWORD error =
            GetLastError();

        Log(
            L"Could not start tar.exe."
        );

        Log(
            L"Windows error: " +
            std::to_wstring(error)
        );

        return false;
    }

    WaitForSingleObject(
        pi.hProcess,
        INFINITE
    );

    DWORD exitCode = 1;

    GetExitCodeProcess(
        pi.hProcess,
        &exitCode
    );

    CloseHandle(
        pi.hThread
    );

    CloseHandle(
        pi.hProcess
    );

    if (exitCode != 0)
    {
        Log(
            L"tar.exe exit code: " +
            std::to_wstring(exitCode)
        );

        return false;
    }

    Log(
        L"Native extraction successful."
    );

    return true;
}

// ============================================================
// CLEAN NATIVES
// ============================================================

void CleanNatives()
{
    try
    {
        for (
            const auto& entry :
            fs::recursive_directory_iterator(
                NATIVES
            ))
        {
            if (!entry.is_regular_file())
                continue;

            std::wstring name =
                entry.path()
                .filename()
                .wstring();

            bool isDll =
                name.size() >= 4 &&
                _wcsicmp(
                    name.c_str() +
                    name.size() - 4,
                    L".dll"
                ) == 0;

            if (!isDll)
            {
                fs::remove(
                    entry.path()
                );
            }
        }
    }
    catch (...)
    {
    }
}

// ============================================================
// INSTALL
// ============================================================

bool InstallDependencies(
    HWND hwnd)
{
    StartLog();

    Log(L"========================================");
    Log(L"INSTALL START");
    Log(L"========================================");

    if (!CreateDirectories())
    {
        PostStatus(
            hwnd,
            L"Could not create Minecraft folders."
        );

        Log(
            L"CreateDirectories failed."
        );

        return false;
    }

    fs::path java;

    if (!FindJava8(java))
    {
        PostStatus(
            hwnd,
            L"Java 8 not found."
        );

        MessageBoxW(
            hwnd,
            L"Java 8 was not found.\n\n"
            L"Minecraft 1.5.2 requires Java 8.\n\n"
            L"Install Java 8 and try again.",
            L"Java 8 Required",
            MB_OK |
            MB_ICONERROR
        );

        return false;
    }

    struct Dependency
    {
        const wchar_t* name;
        const wchar_t* url;
        const fs::path* path;
    };

    const Dependency dependencies[] =
    {
        {
            L"Minecraft 1.5.2",
            CLIENT_URL,
            &CLIENT_JAR
        },

        {
            L"LaunchWrapper",
            LAUNCHWRAPPER_URL,
            &LAUNCHWRAPPER
        },

        {
            L"JOpt Simple",
            JOPT_URL,
            &JOPT
        },

        {
            L"ASM",
            ASM_URL,
            &ASM
        },

        {
            L"JInput",
            JINPUT_URL,
            &JINPUT
        },

        {
            L"JUtils",
            JUTILS_URL,
            &JUTILS
        },

        {
            L"LWJGL",
            LWJGL_URL,
            &LWJGL
        },

        {
            L"LWJGL Util",
            LWJGL_UTIL_URL,
            &LWJGL_UTIL
        },

        {
            L"LWJGL Windows natives",
            LWJGL_NATIVE_URL,
            &LWJGL_NATIVE
        },

        {
            L"JInput Windows natives",
            JINPUT_NATIVE_URL,
            &JINPUT_NATIVE
        }
    };

    const int count =
        static_cast<int>(
            sizeof(dependencies) /
            sizeof(dependencies[0])
            );

    for (int i = 0; i < count; ++i)
    {
        if (!EnsureFile(
            hwnd,
            dependencies[i].name,
            dependencies[i].url,
            *dependencies[i].path))
        {
            PostStatus(
                hwnd,
                L"Dependency download failed."
            );

            Log(
                L"FAILED: " +
                std::wstring(
                    dependencies[i].name
                )
            );

            return false;
        }

        PostProgress(
            hwnd,
            ((i + 1) * 70) / count
        );
    }

    // LWJGL natives
    if (!ExtractNativeJar(
        hwnd,
        LWJGL_NATIVE))
    {
        PostStatus(
            hwnd,
            L"Could not extract LWJGL natives."
        );

        return false;
    }

    PostProgress(
        hwnd,
        85
    );

    // JInput natives
    if (!ExtractNativeJar(
        hwnd,
        JINPUT_NATIVE))
    {
        PostStatus(
            hwnd,
            L"Could not extract JInput natives."
        );

        return false;
    }

    CleanNatives();

    PostProgress(
        hwnd,
        100
    );

    PostStatus(
        hwnd,
        L"Dependencies installed successfully."
    );

    Log(L"INSTALL COMPLETE");

    return true;
}

// ============================================================
// CHECK INSTALLATION
// ============================================================

bool IsInstalled()
{
    try
    {
        return
            fs::exists(CLIENT_JAR) &&
            fs::exists(LAUNCHWRAPPER) &&
            fs::exists(JOPT) &&
            fs::exists(ASM) &&
            fs::exists(JINPUT) &&
            fs::exists(JUTILS) &&
            fs::exists(LWJGL) &&
            fs::exists(LWJGL_UTIL) &&
            fs::exists(LWJGL_NATIVE) &&
            fs::exists(JINPUT_NATIVE) &&
            fs::exists(NATIVES);
    }
    catch (...)
    {
        return false;
    }
}

// ============================================================
// CLASSPATH
// ============================================================

std::wstring BuildClasspath()
{
    std::wstring cp;

    auto Add =
        [&](const fs::path& file)
        {
            if (!cp.empty())
                cp += L";";

            cp +=
                L"\"" +
                file.wstring() +
                L"\"";
        };

    Add(LAUNCHWRAPPER);
    Add(JOPT);
    Add(ASM);
    Add(JINPUT);
    Add(JUTILS);
    Add(LWJGL);
    Add(LWJGL_UTIL);
    Add(CLIENT_JAR);

    return cp;
}

// ============================================================
// BUILD JAVA COMMAND
// ============================================================

std::wstring BuildCommand(
    const fs::path& java)
{
    std::wstring command;

    command +=
        L"\"" +
        java.wstring() +
        L"\" ";

    // 2 GB maximum heap
    command +=
        L"-Xms512M ";

    command +=
        L"-Xmx4048M ";

    // Native libraries
    command +=
        L"-Djava.library.path=\"" +
        NATIVES.wstring() +
        L"\" ";

    // Classpath
    command +=
        L"-cp " +
        BuildClasspath() +
        L" ";

    // LaunchWrapper
    command +=
        L"net.minecraft.launchwrapper.Launch ";

    // Minecraft parameters
    command +=
        L"--version 1.5.2 ";

    command +=
        L"--username Steve ";

    /*
        Offline session.

        This does not authenticate a Microsoft account.
    */

    command +=
        L"--session token:0 ";

    command +=
        L"--gameDir \"" +
        ROOT.wstring() +
        L"\" ";

    command +=
        L"--assetsDir \"" +
        ASSETS.wstring() +
        L"\"";

    return command;
}

// ============================================================
// LAUNCH
// ============================================================

bool LaunchMinecraft(
    HWND hwnd)
{
    StartLog();

    Log(L"========================================");
    Log(L"LAUNCH START");
    Log(L"========================================");

    if (!IsInstalled())
    {
        Log(
            L"Installation incomplete."
        );

        MessageBoxW(
            hwnd,
            L"Dependencies are not installed.\n\n"
            L"Click INSTALL to play derp.",
            L"Installation Required",
            MB_OK |
            MB_ICONWARNING
        );

        return false;
    }

    fs::path java;

    if (!FindJava8(java))
    {
        Log(
            L"Java 8 not found."
        );

        MessageBoxW(
            hwnd,
            L"Java 8 was not found.\n\n"
            L"Install Java 8 first.",
            L"Java Error",
            MB_OK |
            MB_ICONERROR
        );

        return false;
    }

    std::wstring command =
        BuildCommand(java);

    Log(L"Java:");
    Log(java.wstring());

    Log(L"Command:");
    Log(command);

    // ========================================================
    // CREATE BAT
    // ========================================================

    try
    {
        std::wofstream bat(
            BATCH_FILE,
            std::ios::trunc
        );

        if (!bat)
        {
            Log(
                L"Could not create BAT file."
            );

            return false;
        }

        bat
            << L"@echo off\r\n";

        bat
            << L"title Minecraft 1.5.2 Console\r\n";

        bat
            << L"echo ========================================\r\n";

        bat
            << L"echo Minecraft 1.5.2\r\n";

        bat
            << L"echo Username: Steve\r\n";

        bat
            << L"echo RAM: 2 GB\r\n";

        bat
            << L"echo ========================================\r\n";

        bat
            << L"echo.\r\n";

        bat
            << L"cd /d \"";

        bat
            << ROOT.wstring();

        bat
            << L"\"\r\n";

        bat
            << command;

        bat
            << L" 1>\"";

        bat
            << MINECRAFT_LOG.wstring();

        bat
            << L"\" 2>&1\r\n";

        bat
            << L"echo.\r\n";

        bat
            << L"echo ========================================\r\n";

        bat
            << L"echo Minecraft exited.\r\n";

        bat
            << L"echo Exit code: %ERRORLEVEL%\r\n";

        bat
            << L"echo ========================================\r\n";

        bat
            << L"pause\r\n";
    }
    catch (...)
    {
        Log(
            L"Exception while creating BAT."
        );

        return false;
    }

    Log(L"BAT:");
    Log(BATCH_FILE.wstring());

    Log(L"Minecraft console log:");
    Log(MINECRAFT_LOG.wstring());

    // ========================================================
    // START BAT
    // ========================================================

    std::wstring cmd =
        L"cmd.exe /c \"\"" +
        BATCH_FILE.wstring() +
        L"\"\"";

    std::vector<wchar_t> buffer(
        cmd.begin(),
        cmd.end()
    );

    buffer.push_back(
        L'\0'
    );

    STARTUPINFOW si = {};
    PROCESS_INFORMATION pi = {};

    si.cb =
        sizeof(si);

    BOOL created =
        CreateProcessW(
            nullptr,
            buffer.data(),
            nullptr,
            nullptr,
            FALSE,
            CREATE_NEW_CONSOLE,
            nullptr,
            BASE.c_str(),
            &si,
            &pi
        );

    if (!created)
    {
        DWORD error =
            GetLastError();

        Log(
            L"CreateProcess failed."
        );

        Log(
            L"Windows error: " +
            std::to_wstring(error)
        );

        wchar_t message[512] = {};

        FormatMessageW(
            FORMAT_MESSAGE_FROM_SYSTEM |
            FORMAT_MESSAGE_IGNORE_INSERTS,
            nullptr,
            error,
            0,
            message,
            512,
            nullptr
        );

        std::wstring text =
            L"Could not start Minecraft.\n\n"
            L"Windows error: " +
            std::to_wstring(error) +
            L"\n\n" +
            message;

        MessageBoxW(
            hwnd,
            text.c_str(),
            L"Launch Failed",
            MB_OK |
            MB_ICONERROR
        );

        return false;
    }

    Log(
        L"Minecraft console started."
    );

    CloseHandle(
        pi.hThread
    );

    // ========================================================
    // WAIT FOR PROCESS
    // ========================================================

    WaitForSingleObject(
        pi.hProcess,
        INFINITE
    );

    DWORD exitCode = 1;

    GetExitCodeProcess(
        pi.hProcess,
        &exitCode
    );

    CloseHandle(
        pi.hProcess
    );

    Log(
        L"Minecraft process exit code: " +
        std::to_wstring(exitCode)
    );

    // ========================================================
    // COPY CONSOLE LOG INTO DEBUG LOG
    // ========================================================

    try
    {
        if (fs::exists(MINECRAFT_LOG))
        {
            std::wifstream console(
                MINECRAFT_LOG
            );

            if (console)
            {
                Log(
                    L"========== MINECRAFT OUTPUT =========="
                );

                std::wstring line;

                while (
                    std::getline(
                        console,
                        line
                    ))
                {
                    Log(line);
                }

                Log(
                    L"========== END MINECRAFT OUTPUT =========="
                );
            }
        }
    }
    catch (...)
    {
        Log(
            L"Could not read Minecraft console log."
        );
    }

    // ========================================================
    // RESULT
    // ========================================================

    if (exitCode != 0)
    {
        PostStatus(
            hwnd,
            L"Minecraft failed. Check minecraft_console.log."
        );

        MessageBoxW(
            hwnd,
            L"Minecraft could not start.\n\n"
            L"The Java output was saved here:\n\n"
            L"C:\\Minecraft152Launcher\\"
            L"minecraft_console.log\n\n"
            L"Please send that file's contents "
            L"if the game still does not start.",
            L"Minecraft Launch Failed",
            MB_OK |
            MB_ICONERROR
        );

        return false;
    }

    PostStatus(
        hwnd,
        L"Minecraft closed normally."
    );

    return true;
}

// ============================================================
// THREADS
// ============================================================

void InstallThread(
    HWND hwnd)
{
    bool success =
        InstallDependencies(hwnd);

    PostMessageW(
        hwnd,
        WM_APP_FINISH,
        success ? 1 : 0,
        1
    );
}

void LaunchThread(
    HWND hwnd)
{
    bool success =
        LaunchMinecraft(hwnd);

    PostMessageW(
        hwnd,
        WM_APP_FINISH,
        success ? 1 : 0,
        2
    );
}

// ============================================================
// OPEN LOG
// ============================================================

void OpenLog()
{
    try
    {
        fs::create_directories(
            BASE
        );
    }
    catch (...)
    {
    }

    if (!fs::exists(LOG_FILE))
    {
        StartLog();
    }

    ShellExecuteW(
        nullptr,
        L"open",
        LOG_FILE.c_str(),
        nullptr,
        nullptr,
        SW_SHOWNORMAL
    );
}

// ============================================================
// WINDOW PROCEDURE
// ============================================================

LRESULT CALLBACK WindowProc(
    HWND hwnd,
    UINT message,
    WPARAM wParam,
    LPARAM lParam)
{
    switch (message)
    {
        // ========================================================
        // CREATE
        // ========================================================

    case WM_CREATE:
    {
        gWindow =
            hwnd;

        gStatus =
            CreateWindowW(
                L"STATIC",
                L"Ready.",
                WS_CHILD |
                WS_VISIBLE,
                20,
                15,
                550,
                25,
                hwnd,
                nullptr,
                nullptr,
                nullptr
            );

        gProgress =
            CreateWindowExW(
                0,
                PROGRESS_CLASSW,
                nullptr,
                WS_CHILD |
                WS_VISIBLE,
                20,
                45,
                550,
                22,
                hwnd,
                nullptr,
                nullptr,
                nullptr
            );

        SendMessageW(
            gProgress,
            PBM_SETRANGE,
            0,
            MAKELPARAM(0, 100)
        );

        Progress(0);

        gInstall =
            CreateWindowW(
                L"BUTTON",
                L"INSTALL",
                WS_CHILD |
                WS_VISIBLE |
                BS_PUSHBUTTON,
                20,
                85,
                175,
                45,
                hwnd,
                reinterpret_cast<HMENU>(1),
                nullptr,
                nullptr
            );

        gPlay =
            CreateWindowW(
                L"BUTTON",
                L"Play",
                WS_CHILD |
                WS_VISIBLE |
                BS_PUSHBUTTON,
                205,
                85,
                175,
                45,
                hwnd,
                reinterpret_cast<HMENU>(2),
                nullptr,
                nullptr
            );

        gOpenLog =
            CreateWindowW(
                L"BUTTON",
                L"Open FlyBoat Log",
                WS_CHILD |
                WS_VISIBLE |
                BS_PUSHBUTTON,
                390,
                85,
                180,
                45,
                hwnd,
                reinterpret_cast<HMENU>(3),
                nullptr,
                nullptr
            );

        EnableWindow(
            gPlay,
            IsInstalled()
        );

        return 0;
    }

    // ========================================================
    // BUTTONS
    // ========================================================

    case WM_COMMAND:
    {
        const int id =
            LOWORD(wParam);

        // ----------------------------------------------------
        // INSTALL
        // ----------------------------------------------------

        if (id == 1)
        {
            if (gBusy)
                return 0;

            gBusy = true;

            EnableWindow(
                gInstall,
                FALSE
            );

            EnableWindow(
                gPlay,
                FALSE
            );

            EnableWindow(
                gOpenLog,
                FALSE
            );

            SetStatus(
                L"Installing dependencies..."
            );

            Progress(0);

            std::thread(
                InstallThread,
                hwnd
            ).detach();

            return 0;
        }

        // ----------------------------------------------------
        // PLAY
        // ----------------------------------------------------

        if (id == 2)
        {
            if (gBusy)
                return 0;

            gBusy = true;

            EnableWindow(
                gInstall,
                FALSE
            );

            EnableWindow(
                gPlay,
                FALSE
            );

            EnableWindow(
                gOpenLog,
                FALSE
            );

            SetStatus(
                L"Starting Minecraft..."
            );

            std::thread(
                LaunchThread,
                hwnd
            ).detach();

            return 0;
        }

        // ----------------------------------------------------
        // LOG
        // ----------------------------------------------------

        if (id == 3)
        {
            OpenLog();

            return 0;
        }

        break;
    }

    // ========================================================
    // STATUS
    // ========================================================

    case WM_APP_STATUS:
    {
        auto* text =
            reinterpret_cast<std::wstring*>(
                lParam
                );

        if (text)
        {
            SetStatus(
                *text
            );

            delete text;
        }

        return 0;
    }

    // ========================================================
    // PROGRESS
    // ========================================================

    case WM_APP_PROGRESS:
    {
        Progress(
            static_cast<int>(
                wParam
                )
        );

        return 0;
    }

    // ========================================================
    // THREAD FINISHED
    // ========================================================

    case WM_APP_FINISH:
    {
        bool success =
            wParam != 0;

        int operation =
            static_cast<int>(
                lParam
                );

        gBusy = false;

        EnableWindow(
            gInstall,
            TRUE
        );

        EnableWindow(
            gOpenLog,
            TRUE
        );

        if (operation == 1)
        {
            EnableWindow(
                gPlay,
                success
            );

            if (success)
            {
                MessageBoxW(
                    hwnd,
                    L"Dependencies installed successfully.\n\n"
                    L"Minecraft: 1.5.2\n"
                    L"Username: Steve\n"
                    L"RAM: 2 GB\n\n"
                    L"Click PLAY 1.5.2 to launch.",
                    L"Installation Complete",
                    MB_OK |
                    MB_ICONINFORMATION
                );
            }
        }
        else
        {
            EnableWindow(
                gPlay,
                IsInstalled()
            );
        }

        return 0;
    }

    // ========================================================
    // CLOSE
    // ========================================================

    case WM_DESTROY:
    {
        PostQuitMessage(0);

        return 0;
    }
    }

    return DefWindowProcW(
        hwnd,
        message,
        wParam,
        lParam
    );
}

// ============================================================
// ENTRY POINT
// ============================================================

int WINAPI wWinMain(
    HINSTANCE hInstance,
    HINSTANCE,
    PWSTR,
    int nCmdShow)
{
    StartLog();

    Log(
        L"Launcher process started."
    );

    // ========================================================
    // COMMON CONTROLS
    // ========================================================

    INITCOMMONCONTROLSEX controls = {};

    controls.dwSize =
        sizeof(controls);

    controls.dwICC =
        ICC_PROGRESS_CLASS;

    InitCommonControlsEx(
        &controls
    );

    // ========================================================
    // WINDOW CLASS
    // ========================================================

    WNDCLASSW wc = {};

    wc.lpfnWndProc =
        WindowProc;

    wc.hInstance =
        hInstance;

    wc.lpszClassName =
        L"Minecraft152LauncherClass";

    wc.hCursor =
        LoadCursorW(
            nullptr,
            IDC_ARROW
        );

    wc.hbrBackground =
        reinterpret_cast<HBRUSH>(
            COLOR_WINDOW + 1
            );

    if (!RegisterClassW(&wc))
    {
        Log(
            L"RegisterClassW failed."
        );

        MessageBoxW(
            nullptr,
            L"Could not register launcher window.",
            L"Launcher Error",
            MB_OK |
            MB_ICONERROR
        );

        return 1;
    }

    // ========================================================
    // WINDOW
    // ========================================================

    HWND hwnd =
        CreateWindowW(
            L"Minecraft152LauncherClass",
            L"Minecraft 1.5.2 - FlyBoat Edition",
            WS_OVERLAPPED |
            WS_CAPTION |
            WS_SYSMENU |
            WS_MINIMIZEBOX,
            300,
            250,
            610,
            180,
            nullptr,
            nullptr,
            hInstance,
            nullptr
        );

    if (!hwnd)
    {
        Log(
            L"CreateWindowW failed."
        );

        MessageBoxW(
            nullptr,
            L"Could not create launcher window.",
            L"Launcher Error",
            MB_OK |
            MB_ICONERROR
        );

        return 1;
    }

    gWindow =
        hwnd;

    ShowWindow(
        hwnd,
        nCmdShow
    );

    UpdateWindow(
        hwnd
    );

    // ========================================================
    // MESSAGE LOOP
    // ========================================================

    MSG msg = {};

    while (
        GetMessageW(
            &msg,
            nullptr,
            0,
            0
        ) > 0
        )
    {
        TranslateMessage(
            &msg
        );

        DispatchMessageW(
            &msg
        );
    }

    Log(
        L"Launcher process ended."
    );

    return 0;
}
