# Neovim + Neovide 双击文件“单窗口多标签打开”部署手册（C:\Tools 版）

> **目标**：在全新 Windows 机器上，让双击关联文件时由 `launcherNeoVide.exe` 接管——若 Neovide/Neovim 已在运行，则在**同一窗口的新标签页**打开文件；若未运行，则冷启动 Neovide 并打开文件。  
> **统一安装根目录**：`C:\Tools`  
> **目录约定**：
> - Neovim：`C:\Tools\Neovim\bin\nvim.exe`
> - Neovide：`C:\Tools\Neovide\neovide.exe`
> - 启动器：`C:\Tools\NeovideLauncher\launcherNeoVide.exe`

---

## 0. 原理简述

1. **Neovim 端**：`init.lua` 在 Neovide 中执行 `vim.fn.serverstart("\\\\.\\pipe\\nvim-server")`，创建命名管道服务器。
2. **启动器端**：`launcherNeoVide.exe` 检测该管道：
   - **在线**：执行 `nvim --headless --server "\\.\pipe\nvim-server" --remote-tab "文件"`，在已有实例中新标签打开。
   - **离线**：执行 `neovide.exe "文件"`，冷启动新窗口。
3. **注册表**：把启动器加入“打开方式”，再设置默认应用，实现双击接管。
4. **聚焦**：通过 `NeovideFocus` 及 `TabEnter` / `VimEnter` 自动命令，确保新标签打开后窗口前置。

---

## 1. 环境准备

### 1.1 安装 Neovim
- 将 Neovim 解压/安装到 `C:\Tools\Neovim\`，确保存在：
  ```text
  C:\Tools\Neovim\bin\nvim.exe
  ```
- 将 `C:\Tools\Neovim\bin` 加入系统 `PATH`，命令行执行 `nvim --version` 能正常输出版本。

### 1.2 安装 Neovide
- 将 Neovide 放置到 `C:\Tools\Neovide\`，确保存在：
  ```text
  C:\Tools\Neovide\neovide.exe
  ```
- 手动双击 `neovide.exe`，确认能正常启动。

### 1.3 安装 MinGW-w64 / g++
- 命令行执行 `g++ --version` 能正常输出版本。若没有，安装 MSYS2/MinGW-w64，并把其 `bin` 目录加入 `PATH`。

### 1.4 准备启动器目录
创建目录：
```text
C:\Tools\NeovideLauncher\
```

### 1.5 备份注册表
导入 `.reg` 前，建议先备份：
- `Win + R` → `regedit` → 文件 → 导出 → 备份 `HKEY_CLASSES_ROOT` 相关项或整个注册表。

---

## 2. 配置 Neovim 的命名管道服务器

### 2.1 找到 Neovim 配置目录
在 Neovim 中执行 `:echo stdpath('config')`，Windows 通常为：
```text
C:\Users\<用户名>\AppData\Local\nvim
```

### 2.2 在 `init.lua` 中加入多标签所需片段
将下文 **“附录 A：init.lua 片段”** 的内容追加到你现有的 `init.lua` 末尾（若新机器没有 `init.lua`，直接新建并写入该片段即可）。

> 说明：这段是“多标签打开文件”功能所需的**最小片段**。其余配色、快捷键、插件、临时目录等配置与本功能无关，按个人习惯自行配置。

### 2.3 验证管道是否创建
用 Neovide 正常启动 Neovim，在 Neovim 中执行：
```vim
:echo serverlist()
```
应能看到类似：
```text
['\\.\pipe\nvim-server']
```
如果没有，说明 `init.lua` 未加载或 `vim.g.neovide` 判断未通过。先解决此问题，否则“新标签打开”不会生效。

---

## 3. 编译启动器

将下文 **“附录 B：launcherNeoVide.cpp”** 保存到：
```text
C:\Tools\NeovideLauncher\launcherNeoVide.cpp
```
源文件使用 **UTF-8 编码** 保存即可（含中文注释不影响编译）。

在 `C:\Tools\NeovideLauncher\` 目录下打开命令行，执行：
```bash
g++ -O2 -s -o launcherNeoVide.exe launcherNeoVide.cpp -mwindows -static
```

编译成功后确认存在：
```text
C:\Tools\NeovideLauncher\launcherNeoVide.exe
```

---

## 4. 导入注册表关联

### 4.1 准备 `.reg` 文件
将下文 **“附录 C：neovide_LauncherNeoVide_register.reg”** 的内容保存为：
```text
C:\Tools\NeovideLauncher\neovide_LauncherNeoVide_register.reg
```
> 保存时请使用“ANSI”或“Unicode（UTF-16 LE）”编码（**不要**用 UTF-8），否则注册表编辑器可能识别异常。

### 4.2 导入
双击该 `.reg` 文件，UAC 确认。若提示权限不足，右键 → 以管理员身份运行 `regedit`，再通过“文件 → 导入”导入。

导入后效果：
- 注册 `Neovide.cppLauncher` ProgID；
- 在所有文件的“打开方式”候选列表中加入 **Neovide (cppLauncher)**；
- **不会自动修改默认打开程序**。

---

## 5. 设置默认打开方式（让双击生效）

以 `.txt` 为例：

1. 右键任意 `.txt` 文件；
2. 选择 **打开方式** → **选择其他应用**；
3. 若列表中有 **Neovide (cppLauncher)**，选择它；
4. 若没有，点击 **更多应用** → **在这台电脑上查找其他应用**，浏览到：
   ```text
   C:\Tools\NeovideLauncher\launcherNeoVide.exe
   ```
5. 勾选 **始终使用此应用打开 .txt 文件**；
6. 点击 **确定**。

对其他需要关联的扩展名重复上述操作。  
> 不建议把所有文件类型的默认程序都设为启动器，只对需要的扩展名设置即可。

---

## 6. 功能验证

### 6.1 冷启动测试
- 关闭所有 Neovide 窗口；
- 双击一个已关联的 `.txt` 文件；
- 预期：启动一个新的 Neovide 窗口，并打开该文件。

### 6.2 新标签打开测试（核心）
- 保持上一步的 Neovide 窗口不关闭；
- 双击**另一个** `.txt` 文件；
- 预期：在**同一个 Neovide 窗口**中新建标签页打开，并自动聚焦。

### 6.3 启动器自身测试
- 直接双击 `launcherNeoVide.exe`（不通过文件关联）；
- 若已有 Neovide 在运行：预期激活已有窗口；
- 若没有：预期启动空白 Neovide。

### 6.4 管道检查
在 Neovim 中执行：
```vim
:echo serverlist()
```
应包含：
```text
\\.\pipe\nvim-server
```

---

## 7. 故障排查

| 现象 | 可能原因 | 处理 |
|---|---|---|
| 双击文件总是新开窗口 | 管道未建立或地址不一致 | 检查 `init.lua` 中 `serverstart` 与 C++ `PIPE_ADDR`；执行 `:echo serverlist()` |
| 双击文件无反应 | 注册表 exe 路径错误；默认应用未设置 | 检查 `.reg` 中 command 路径；重新设置默认应用 |
| 新标签打开但窗口不前置 | `TabEnter` 聚焦未生效 | 检查 `focus_neovide` 与 `NeovideFocus` 命令；确认 `TabEnter` 自动命令已注册 |
| 启动器闪退/无窗口 | 编译未加 `-mwindows`；或被杀毒拦截 | 用给定命令重新编译；加入杀毒白名单 |
| 中文路径乱码 | 使用了旧 ANSI 版本 | 使用本 Unicode 版 `launcherNeoVide.cpp` 编译 |
| Neovide 冷启动失败 | `PATH` 中没有 `nvim.exe`；`NEOVIDE_PATH` 错误 | 命令行验证 `nvim --version`；检查宏路径 |
| 管道检测在线但打开失败 | `NVIM_PATH` 错误；权限不一致 | 检查 `nvim.exe` 绝对路径；Neovide 与启动器用同一用户权限运行 |
| 导入 reg 后“打开方式”无项 | 未以管理员导入；路径写错 | 管理员导入；检查 `HKEY_CLASSES_ROOT\*\OpenWithProgIDs` |
| 快速双击多个文件时启动了多个 Neovide | 管道尚未创建（Neovide 正在冷启动） | 等第一次启动完成后再双击；或视为已知限制 |

**关键一致性检查清单：**
- `init.lua`：`serverstart("\\\\.\\pipe\\nvim-server")`
- C++：`PIPE_ADDR L"\\\\.\\pipe\\nvim-server"`
- 两者实际字符串均为 `\\.\pipe\nvim-server`
- `.reg` 中 command 指向 `C:\Tools\NeovideLauncher\launcherNeoVide.exe`
- 系统 `PATH` 中能找到 `C:\Tools\Neovim\bin\nvim.exe`
- 已为对应扩展名设置默认应用

---

## 8. 卸载与恢复

1. 将下文 **“附录 D：neovide_unLauncherNeoVide_register.reg”** 保存为同名 `.reg` 文件并导入。
2. 手动把 `.txt` 等扩展名的默认应用改回原程序（如记事本）。
3. 删除 `C:\Tools\NeovideLauncher\` 目录。
4. 若出现异常，使用第 1.5 步备份的注册表恢复。

---

## 9. 最小部署清单

- [ ] Neovim 已放到 `C:\Tools\Neovim\`，`nvim.exe` 在 `PATH` 中
- [ ] Neovide 已放到 `C:\Tools\Neovide\`，可手动启动
- [ ] `init.lua` 中已加入附录 A 片段
- [ ] `:echo serverlist()` 能看到 `\\.\pipe\nvim-server`
- [ ] `launcherNeoVide.cpp` 中三个宏指向 `C:\Tools` 下路径
- [ ] 已用 `g++ -O2 -s -o launcherNeoVide.exe launcherNeoVide.cpp -mwindows -static` 编译
- [ ] `.reg` 已导入，exe 路径为 `C:\Tools\NeovideLauncher\launcherNeoVide.exe`
- [ ] 已为 `.txt` 等扩展名设置默认应用
- [ ] 冷启动、新标签打开、启动器激活三项测试通过

---

## 附录 A：init.lua 片段（仅多标签相关）

将以下内容追加到你的 `init.lua` 末尾（或作为独立的 `init.lua` 使用）：

```lua
-- ============================ Neovide 专属配置（多标签核心） ============================
-- 依赖 launcherNeoVide.exe 配合，实现双击文件在同一 Neovide 窗口新标签打开
if vim.g.neovide then
    -- 1. 命名管道通信（与 C++ 启动器交互）
    --    地址必须与 launcherNeoVide.cpp 中的 PIPE_ADDR 完全一致
    vim.fn.serverstart("\\\\.\\pipe\\nvim-server")

    -- 2. 窗口聚焦辅助函数
    --    NeovideFocus 为 Neovide 提供的命令，用于将窗口带到前台
    local focus_neovide = function(delay)
        vim.defer_fn(function()
            pcall(vim.cmd, "NeovideFocus") -- pcall 容错：命令失败不报错
        end, delay)
    end

    -- 3. 标签页切换时聚焦（新标签打开时窗口前置）
    vim.api.nvim_create_autocmd("TabEnter", {
        callback = function() focus_neovide(10) end,
        desc = "标签页切换时聚焦 Neovide 窗口"
    })

    -- 4. 首次启动时聚焦（冷启动时窗口前置）
    vim.api.nvim_create_autocmd("VimEnter", {
        callback = function() focus_neovide(200) end,
        desc = "Neovide 启动后聚焦窗口"
    })
end
```

---

## 附录 B：launcherNeoVide.cpp

```cpp
/**
 * launcherNeoVide.cpp - Neovide 文件启动器（C:\Tools 版）
 *
 * 编译输出：launcherNeoVide.exe
 * 编译命令（静态链接，无控制台窗口）：
 *   g++ -O2 -s -o launcherNeoVide.exe launcherNeoVide.cpp -mwindows -static
 *
 * 功能：
 *   1. 双击关联文件 → 共用同一 Neovide 窗口，新标签页打开，自动聚焦。
 *   2. 双击启动器本身 → 若 Neovide 已运行，尝试激活窗口（不启动新实例，不报错）；
 *      若未运行，启动空白 Neovide 并获得焦点。
 *
 * 技术要点：
 *   - 命名管道 (\\.\pipe\nvim-server) 直接检测服务端状态，零额外进程。
 *   - 热启动时通过 nvim --remote-expr 执行 NeovideFocus 激活窗口。
 *   - 所有路径使用栈缓冲区，无堆分配，启动极速。
 *   - 全面支持 Unicode（中文路径无乱码），使用安全字符串函数防止溢出。
 *   - 入口使用标准 WinMain + 内部宽字符 API，无需 -municode 链接选项。
 *
 * 环境依赖：
 *   - init.lua 中需启动命名管道服务器：vim.fn.serverstart('\\\\.\\pipe\\nvim-server')
 *   - 路径需与代码中宏定义一致，按实际安装位置修改。
 */

#include <windows.h>
#include <strsafe.h>   // 提供安全字符串函数 StringCchPrintfW / StringCchCopyW
#include <shellapi.h>  // 提供 CommandLineToArgvW

// ======================== 可配置宏（C:\Tools 统一目录） ========================
#define NVIM_PATH      L"C:\\Tools\\Neovim\\bin\\nvim.exe"
#define NEOVIDE_PATH   L"C:\\Tools\\Neovide\\neovide.exe"
#define PIPE_ADDR      L"\\\\.\\pipe\\nvim-server"

// ======================== 辅助函数 ==============================================

/**
 * IsNvimServerOnline - 检测命名管道服务端是否在线
 * @param pipeAddr  管道地址（宽字符串）
 * @return TRUE 表示服务端存在且可通信，FALSE 表示离线
 *
 * 错误判断：除 FILE_NOT_FOUND 外，ERROR_PIPE_BUSY 和 ERROR_ACCESS_DENIED
 * 也视为在线（服务端存在但暂时无法访问），避免误判。
 */
BOOL IsNvimServerOnline(LPCWSTR pipeAddr)
{
    HANDLE hPipe = CreateFileW(pipeAddr, GENERIC_READ | GENERIC_WRITE, 0,
                               NULL, OPEN_EXISTING, 0, NULL);
    if (hPipe != INVALID_HANDLE_VALUE) {
        CloseHandle(hPipe);
        return TRUE;
    }

    DWORD err = GetLastError();
    if (err == ERROR_PIPE_BUSY || err == ERROR_ACCESS_DENIED) {
        return TRUE;
    }
    return FALSE;
}

/**
 * LaunchNvimRemoteCommand - 向已运行的 Neovim 服务端发送远程命令
 * @param nvimPath   nvim.exe 路径
 * @param pipeAddr   管道地址
 * @param command    远程命令（如 "--remote-tab \"file\"" 或 "--remote-expr ..."）
 * @param hideWindow 是否隐藏 nvim 控制台（通常 TRUE）
 * @return TRUE 成功启动进程，FALSE 失败
 */
BOOL LaunchNvimRemoteCommand(LPCWSTR nvimPath, LPCWSTR pipeAddr,
                             LPCWSTR command, BOOL hideWindow)
{
    WCHAR cmdLine[1024];
    HRESULT hr = StringCchPrintfW(cmdLine, _countof(cmdLine),
                                  L"\"%s\" --headless --server \"%s\" %s",
                                  nvimPath, pipeAddr, command);
    if (FAILED(hr)) {
        return FALSE;
    }

    STARTUPINFOW si = { sizeof(si) };
    PROCESS_INFORMATION pi = {};
    DWORD creationFlags = hideWindow ? CREATE_NO_WINDOW : 0;

    if (!CreateProcessW(NULL, cmdLine, NULL, NULL, FALSE,
                        creationFlags, NULL, NULL, &si, &pi)) {
        return FALSE;
    }

    CloseHandle(pi.hProcess);
    CloseHandle(pi.hThread);
    return TRUE;
}

/**
 * LaunchNeovideWithFile - 启动新的 Neovide 实例并打开指定文件
 */
BOOL LaunchNeovideWithFile(LPCWSTR neovidePath, LPCWSTR filePath)
{
    WCHAR cmdLine[1024];
    HRESULT hr = StringCchPrintfW(cmdLine, _countof(cmdLine),
                                  L"\"%s\" \"%s\"", neovidePath, filePath);
    if (FAILED(hr)) {
        return FALSE;
    }

    STARTUPINFOW si = { sizeof(si) };
    PROCESS_INFORMATION pi = {};
    if (!CreateProcessW(NULL, cmdLine, NULL, NULL, FALSE,
                        0, NULL, NULL, &si, &pi)) {
        return FALSE;
    }

    AllowSetForegroundWindow(pi.dwProcessId);
    CloseHandle(pi.hProcess);
    CloseHandle(pi.hThread);
    return TRUE;
}

/**
 * LaunchBlankNeovide - 启动空白 Neovide（无文件参数）
 */
BOOL LaunchBlankNeovide(LPCWSTR neovidePath)
{
    WCHAR cmdLine[512];
    HRESULT hr = StringCchPrintfW(cmdLine, _countof(cmdLine),
                                  L"\"%s\"", neovidePath);
    if (FAILED(hr)) {
        return FALSE;
    }

    STARTUPINFOW si = { sizeof(si) };
    PROCESS_INFORMATION pi = {};
    if (!CreateProcessW(NULL, cmdLine, NULL, NULL, FALSE,
                        0, NULL, NULL, &si, &pi)) {
        return FALSE;
    }

    AllowSetForegroundWindow(pi.dwProcessId);
    CloseHandle(pi.hProcess);
    CloseHandle(pi.hThread);
    return TRUE;
}

// ======================== WinMain 入口 ==========================================

int WINAPI WinMain(HINSTANCE hInstance, HINSTANCE hPrevInstance,
                   LPSTR lpCmdLine, int nCmdShow)
{
    // ---------- 1. 使用宽字符 API 获取命令行参数（支持中文路径） ----------
    WCHAR filePath[MAX_PATH] = L"";

    int argc;
    LPWSTR* argv = CommandLineToArgvW(GetCommandLineW(), &argc);
    if (argv != NULL) {
        if (argc > 1) {
            StringCchCopyW(filePath, _countof(filePath), argv[1]);
        }
        LocalFree(argv);
    }

    // ---------- 2. 无文件参数：双击启动器本身 ----------
    if (filePath[0] == L'\0') {
        if (IsNvimServerOnline(PIPE_ADDR)) {
            // 已有实例：激活窗口（不启动新实例，不报错）
            LaunchNvimRemoteCommand(NVIM_PATH, PIPE_ADDR,
                                    L"--remote-expr \"vim.cmd('NeovideFocus')\"",
                                    TRUE);
        } else {
            // 未运行：启动空白 Neovide
            LaunchBlankNeovide(NEOVIDE_PATH);
        }
        return 0;
    }

    // ---------- 3. 有文件参数：双击关联文件 ----------
    if (IsNvimServerOnline(PIPE_ADDR)) {
        // 服务端在线：新标签页打开文件（聚焦由 TabEnter 自动命令接管）
        WCHAR command[1024];
        HRESULT hr = StringCchPrintfW(command, _countof(command),
                                      L"--remote-tab \"%s\"", filePath);
        if (SUCCEEDED(hr)) {
            LaunchNvimRemoteCommand(NVIM_PATH, PIPE_ADDR, command, TRUE);
        }
    } else {
        // 服务端不在线：启动新实例打开文件
        LaunchNeovideWithFile(NEOVIDE_PATH, filePath);
    }

    return 0;
}
```

---

## 附录 C：neovide_LauncherNeoVide_register.reg

> 保存时使用 **ANSI** 或 **UTF-16 LE** 编码（**不要** UTF-8）。

```reg
Windows Registry Editor Version 5.00

; ==================== 注册启动器 ProgID ====================
[HKEY_CLASSES_ROOT\Neovide.cppLauncher]
@="Neovide (cppLauncher)"

[HKEY_CLASSES_ROOT\Neovide.cppLauncher\shell\open\command]
@="\"C:\\Tools\\NeovideLauncher\\launcherNeoVide.exe\" \"%1\""

; ==================== 加入所有文件的“打开方式”候选列表 ====================
[HKEY_CLASSES_ROOT\*\OpenWithProgIDs]
"Neovide.cppLauncher"=""
```

---

## 附录 D：neovide_unLauncherNeoVide_register.reg

> 保存时使用 **ANSI** 或 **UTF-16 LE** 编码（**不要** UTF-8）。

```reg
Windows Registry Editor Version 5.00

; ==================== 删除 ProgID 主项 ====================
[-HKEY_CLASSES_ROOT\Neovide.cppLauncher]

; ==================== 从“打开方式”候选列表中移除 ====================
[-HKEY_CLASSES_ROOT\*\OpenWithProgIDs\Neovide.cppLauncher]
```

---

### 变更点 / 校对说明

| 项目 | 校对结论 |
|---|---|
| `init.lua` 片段 | 已按“只保留多标签相关”要求精简；仅含 `serverstart` + 聚焦逻辑 |
| 路径一致性 | `init.lua` 与 `launcherNeoVide.cpp` 中管道地址实际字符串均为 `\\.\pipe\nvim-server` ✅ |
| cpp 中路径宏 | 已统一改为 `C:\Tools\Neovim\bin\nvim.exe` / `C:\Tools\Neovide\neovide.exe` ✅ |
| `.reg` 中路径 | 已统一为 `C:\Tools\NeovideLauncher\launcherNeoVide.exe` ✅ |
| 文字描述 | 之前手册中“R: 盘临时目录 / 配色 / 快捷键”等内容与本功能无关，已全部删除 |
| `.reg` 编码提示 | 新增“使用 ANSI 或 UTF-16 LE”提示，避免中文乱码或导入失败 |
| 快速双击多文件 | 在故障排查中补充为“已知限制”，不视为缺陷 |

按以上步骤操作，即可在 `C:\Tools` 统一目录下实现“双击文件在 Neovide 同一窗口的新标签页打开”的功能。
