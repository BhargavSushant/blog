---
title: "How to Install a C Compiler on Windows (GCC with MSYS2) and Use It in VS Code"
date: 2026-10-22T16:00:00+05:30
draft: false
author: "Sushant Bhargav"
categories: ["Training"]
series: ["C Language Training"]
tags: ["C", "C Language", "GCC", "MSYS2", "Windows", "VS Code", "Beginners"]
summary: "A step-by-step guide to installing the GCC C compiler on a Windows machine using MSYS2 and winget, setting up PATH, and running your first C program from VS Code."
ShowToc: true
TocOpen: true
weight: 10
---

This is the first lesson in the **C Language Training** series. Before writing any C code, you need a compiler on your computer. A compiler turns your `.c` source file into an `.exe` that Windows can run.

On Windows there is no C compiler built in, so we will install **GCC** (the GNU Compiler Collection) using **MSYS2**.

## What is MSYS2?

MSYS2 is a software distribution for Windows. It gives you a Unix-style terminal and a package manager called `pacman` (the same one used by Arch Linux). The main reason we use it is simple: it is the easiest way to get a real, up-to-date GCC compiler on Windows. One command installs the whole toolchain, and one command updates it later.

## What you need

- A Windows 10 or Windows 11 PC
- An internet connection
- [Visual Studio Code](https://code.visualstudio.com/) installed

## Step 1: Install MSYS2

### Option A: Using winget (fastest)

Open **PowerShell** or **Command Prompt** and run:

```powershell
winget install --id MSYS2.MSYS2 -e --source winget
```

This installs MSYS2 to `C:\msys64`.

### Option B: Manual download

If winget fails (for example with a network error such as `0x80072efe`), download the installer yourself:

1. Open [https://www.msys2.org](https://www.msys2.org).
2. Download the installer (`msys2-x86_64-xxxxxxxx.exe`).
3. Run it and keep the default install folder `C:\msys64`.
4. Finish the installation.

> **Tip:** If downloads keep dropping, turn off any VPN or proxy, or try a different network such as a mobile hotspot.

## Step 2: Install the GCC toolchain

This is the step where many beginners get stuck. The `pacman` command **only works inside the MSYS2 terminal**. It will say "not recognized" in PowerShell or Command Prompt.

1. Press the Windows key and search for **MSYS2 UCRT64**.
2. Open it. This is a separate terminal window, not PowerShell.
3. Run this command:

```bash
pacman -S --needed base-devel mingw-w64-ucrt-x86_64-toolchain
```

4. When it shows `Enter a selection (default=all)`, press **Enter**.
5. When it asks to proceed with the installation, type **Y** and press Enter.

If you see messages saying packages are "up to date -- skipping", that is fine. It means they are already installed.

## Step 3: Add the compiler to your Windows PATH

Right now GCC works only inside the MSYS2 window. To use it from PowerShell and VS Code, add it to your PATH:

1. Press the Windows key and search for **Environment Variables**.
2. Click **Edit environment variables for your account**.
3. Under **User variables**, select **Path** and click **Edit**.
4. Click **New** and paste:

```
C:\msys64\ucrt64\bin
```

5. Click **OK** on every window to save.

## Step 4: Verify the installation

Close all open terminals and VS Code windows. PATH changes are not picked up by windows that were already open.

Open a **new** PowerShell window and run:

```powershell
gcc --version
```

You should see output that starts with something like:

```
gcc.exe (Rev..., Built by MSYS2 project) 16.2.0
```

You can also check the debugger and make tool:

```powershell
gdb --version
mingw32-make --version
```

## Step 5: Set up VS Code

1. Open VS Code.
2. Go to Extensions (`Ctrl + Shift + X`).
3. Search for **C/C++** (published by Microsoft) and click **Install**.
4. Optional: install **Code Runner** if you want a one-click run button.

## Step 6: Write and run your first C program

Create a folder for your practice files, open it in VS Code, and create a file named `hello.c`:

```c
#include <stdio.h>

int main(void) {
    printf("Hello, World!\n");
    return 0;
}
```

Open the VS Code terminal (``Ctrl + ` ``) and run:

```powershell
gcc hello.c -o hello
.\hello.exe
```

Output:

```
Hello, World!
```

What the compile command means:

| Part | Meaning |
|------|---------|
| `gcc` | The C compiler |
| `hello.c` | Your source file |
| `-o hello` | Name the output file `hello` (becomes `hello.exe` on Windows) |

## Practice program: print a pyramid

Once Hello World works, try this program. It prints a centered pyramid of stars.

```c
#include <stdio.h>

int main(void) {
    int rows;

    printf("Enter number of rows: ");
    scanf("%d", &rows);

    for (int i = 1; i <= rows; i++) {
        for (int j = 1; j <= rows - i; j++) {
            printf(" ");
        }
        for (int k = 1; k <= 2 * i - 1; k++) {
            printf("*");
        }
        printf("\n");
    }

    return 0;
}
```

Output for 5 rows:

```
    *
   ***
  *****
 *******
*********
```

## Troubleshooting

**`pacman` is not recognized**
You are in PowerShell or Command Prompt. Open **MSYS2 UCRT64** from the Start menu and run it there.

**`gcc` is not recognized**
The PATH entry is missing or the terminal is old. Check that `C:\msys64\ucrt64\bin` is in your User PATH, then open a new terminal.

**winget error `0x80072efe`**
A network connection problem. Retry with `--source winget`, disable VPN or proxy, switch networks, or use the manual installer from msys2.org.

**Red squiggles under `#include` in VS Code**
Press `Ctrl + Shift + P`, choose **C/C++: Edit Configurations (UI)**, and set the compiler path to `C:\msys64\ucrt64\bin\gcc.exe`.

**Using the wrong MSYS2 window**
MSYS2 installs several shortcuts (MSYS, UCRT64, MINGW64, CLANG64). For this guide always use **UCRT64**.

## What's next

In the next lesson, [Hello, World and the Shape of a C Program](/training/c-language/hello-world/), we write a first program, look at it line by line, and learn to read a compiler error.

Happy coding!
