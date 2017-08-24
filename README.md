# Running VM VirtualBox in background (Headless Start)

1. Choose which VM `{vm_name}` or `{vm_id}` you want to run on headless start `(e.g. {vm_name} -> "Centos 7" or {vm_id} -> "a278a531-6768-4eda-a2b2-cb7445723e97")`

2. Locate your VirtualBox `(e.g. "C:\Program Files\Oracle\VirtualBox")`

   Create a batch file `(e.g. "C:\path\to\batch\centos7.bat")`
   ```
   cd "c:\Program Files\Oracle\VirtualBox\"
   VBoxHeadless.exe -s "{vm_name}" -v on
   ```
   or
   ```
   cd "C:\Program Files\Oracle\VirtualBox"
   VBoxHeadless.exe --startvm "{vm_id}"
   ```

3. Run `(win + r)`
   ```
   shell:startup
   ```

   Create vbs file on your startup directory `(e.g. centos7.vbs)`
   ```
   Set centos7 = WScript.CreateObject("WScript.Shell")
   obj = centos7.Run("C:\path\to\batch\centos7.bat", 0)
   set centos7 = Nothing
   ```

4. Restart your computer or execute vbs script directly to test and your done!
