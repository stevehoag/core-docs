---
title: Implementing a dispose method
description: Implementing a dispose method
keywords: .NET, .NET Core
author: shoag
manager: wpickett
ms.date: 08/16/2016
ms.topic: article
ms.prod: .net-core
ms.technology: .net-core-technologies
ms.devlang: dotnet
ms.assetid: eca6cdc3-6a14-4296-86fb-1eb2f21455b0
---

# Implementing a dispose method

You implement a [Dispose](xref:System.IDisposable#System_IDisposable_Dispose) method to release unmanaged resources used by your application. The .NET garbage collector does not allocate or release unmanaged memory. 

The pattern for disposing an object, referred to as a dispose pattern, imposes order on the lifetime of an object. The dispose pattern is used only for objects that access unmanaged resources, such as file and pipe handles, registry handles, wait handles, or pointers to blocks of unmanaged memory. This is because the garbage collector is very efficient at reclaiming unused managed objects, but it is unable to reclaim unmanaged objects.

The dispose pattern has two variations:

* You wrap each unmanaged resource that a type uses in a safe handle (that is, in a class derived from [System.Runtime.InteropServices.SafeHandle](xref:System.Runtime.InteropServices.SafeHandle)). In this case, you implement the [IDisposable](xref:System.IDisposable) interface and an additional `Dispose(Boolean)` method. This is the recommended variation and doesn't require overriding the [Object.Finalize](xref:System.Object#System_Object_Finalize) method. 

> [!NOTE]
> The [Microsoft.Win32.SafeHandles](xref:Microsoft.Win32.SafeHandles) namespace provides a set of classes derived from [SafeHandle](xref:System.Runtime.InteropServices.SafeHandle), which are listed in the [Using safe handles](#Using-safe-handles) section. If you can't find a class that is suitable for releasing your unmanaged resource, you can implement your own subclass of [SafeHandle](xref:System.Runtime.InteropServices.SafeHandle). 
 
* You implement the [IDisposable](xref:System.IDisposable) interface and an additional `Dispose(Boolean`) method, and you also override the [Object.Finalize](xref:System.Object#System_Object_Finalize) method. You must override [Finalize](xref:System.Object#System_Object_Finalize) to ensure that unmanaged resources are disposed of if your [IDisposable.Dispose](xref:System.IDisposable#System_IDisposable_Dispose) implementation is not called by a consumer of your type. If you use the recommended technique discussed in the previous bullet, the [System.Runtime.InteropServices.SafeHandle](xref:System.Runtime.InteropServices.SafeHandle) class does this on your behalf. 

To help ensure that resources are always cleaned up appropriately, a [Dispose](xref:System.IDisposable#System_IDisposable_Dispose) method should be callable multiple times without throwing an exception. 

The code example provided for the [GC.KeepAlive](xref:System.GC#System_GC_KeepAlive_System_Object_) method shows how aggressive garbage collection can cause a finalizer to run while a member of the reclaimed object is still executing. It is a good idea to call the [KeepAlive](xref:System.GC#System_GC_KeepAlive_System_Object_) method at the end of a lengthy `Dispose` method.

## Dispose() and Dispose(Boolean)

The [IDisposable](xref:System.IDisposable) interface requires the implementation of a single parameterless method, [Dispose](xref:System.IDisposable#System_IDisposable_Dispose). However, the dispose pattern requires two `Dispose` methods to be implemented: 

* A public non-virtual (`NonInheritable` in Visual Basic) [IDisposable.Dispose](xref:System.IDisposable#System_IDisposable_Dispose) implementation that has no parameters.

* A protected virtual (`Overridable` in Visual Basic) `Dispose` method whose signature is:

  ```cs
  protected virtual void Dispose(bool disposing)
  ```

  ```vb
  Protected Overridable Sub Dispose(disposing As Boolean)
  ```

### The Dispose() overload

Because the public, non-virtual (`NonInheritable` in Visual Basic), parameterless `Dispose` method is called by a consumer of the type, its purpose is to free unmanaged resources and to indicate that the finalizer, if one is present, doesn't have to run. Because of this, it has a standard implementation:

```cs
public void Dispose()
{
   // Dispose of unmanaged resources.
   Dispose(true);
   // Suppress finalization.
   GC.SuppressFinalize(this);
}
```

```vb
Public Sub Dispose() _
           Implements IDisposable.Dispose
   ' Dispose of unmanaged resources.
   Dispose(True)
   ' Suppress finalization.
   GC.SuppressFinalize(Me)
End Sub
```

The `Dispose` method performs all object cleanup, so the garbage collector no longer needs to call the objects' [Object.Finalize](xref:System.Object#System_Object_Finalize) override. Therefore, the call to the [GC.SuppressFinalize](xref:System.GC.System_GC_SuppressFinalize_System_Object_) method prevents the garbage collector from running the finalizer. If the type has no finalizer, the call to [SuppressFinalize](xref:System.GC.System_GC_SuppressFinalize_System_Object_) has no effect. Note that the actual work of releasing unmanaged resources is performed by the second overload of the `Dispose` method.

### The Dispose(Boolean) overload

In the second overload, the *disposing* parameter is a [Boolean](xref:System.Boolean) that indicates whether the method call comes from a [Dispose](xref:System.IDisposable#System_IDisposable_Dispose) method (its value is `true`) or from a finalizer (its value is `false`). 

The body of the method consists of two blocks of code: 

* A block that frees unmanaged resources. This block executes regardless of the value of the *disposing* parameter. 

* A conditional block that frees managed resources. This block executes if the value of *disposing* is `true`. The managed resources that it frees can include: 

    **Managed objects that implement IDisposable**. The conditional block can be used to call their [Dispose](xref:System.IDisposable#System_IDisposable_Dispose) implementation. If you have used a safe handle to wrap your unmanaged resource, you should call the [SafeHandle.Dispose(Boolean](xref:System.Runtime.InteropServices.SafeHandle.html#System_Runtime_InteropServices_SafeHandle_Dispose_System_Boolean_) implementation here. 

    **Managed objects that consume large amounts of memory or consume scarce resources.** Freeing these objects explicitly in the `Dispose` method releases them faster than if they were reclaimed non-deterministically by the garbage collector. 


If the method call comes from a finalizer (that is, if *disposing* is `false`), only the code that frees unmanaged resources executes. Because the order in which the garbage collector destroys managed objects during finalization is not defined, calling this `Dispose` overload with a value of `false` prevents the finalizer from trying to release managed resources that may have already been reclaimed. 

## Implementing the dispose pattern for a base class

If you implement the dispose pattern for a base class, you must provide the following: 

> [!IMPORTANT]
> You should implement this pattern for all base classes that implement [IDisposable](xref:System.IDisposable) and are not `sealed`. 
 
* A [Dispose](xref:System.IDisposable#System_IDisposable_Dispose) implementation that calls the `Dispose(Boolean)` method. 

* A `Dispose(Boolean)` method that performs the actual work of releasing resources. 

* Either a class derived from [SafeHandle](xref:System.Runtime.InteropServices.SafeHandle) that wraps your unmanaged resource (recommended), or an override to the [Object.Finalize](xref:System.Object#System_Object_Finalize) method. The [SafeHandle](xref:System.Runtime.InteropServices.SafeHandle)SafeHandle class provides a finalizer that frees you from having to code one. 

Here's the general pattern for implementing the dispose pattern for a base class that uses a safe handle. 

```cs
using Microsoft.Win32.SafeHandles;
using System;
using System.Runtime.InteropServices;

class BaseClass : IDisposable
{
   // Flag: Has Dispose already been called?
   bool disposed = false;
   // Instantiate a SafeHandle instance.
   SafeHandle handle = new SafeFileHandle(IntPtr.Zero, true);

   // Public implementation of Dispose pattern callable by consumers.
   public void Dispose()
   { 
      Dispose(true);
      GC.SuppressFinalize(this);           
   }

   // Protected implementation of Dispose pattern.
   protected virtual void Dispose(bool disposing)
   {
      if (disposed)
         return; 

      if (disposing) {
         handle.Dispose();
         // Free any other managed objects here.
         //
      }

      // Free any unmanaged objects here.
      //
      disposed = true;
   }
}
```

```vb
Imports Microsoft.Win32.SafeHandles
Imports System.Runtime.InteropServices

Class BaseClass : Implements IDisposable
   ' Flag: Has Dispose already been called?
   Dim disposed As Boolean = False
   ' Instantiate a SafeHandle instance.
   Dim handle As SafeHandle = New SafeFileHandle(IntPtr.Zero, True)

   ' Public implementation of Dispose pattern callable by consumers.
   Public Sub Dispose() _
              Implements IDisposable.Dispose
      Dispose(True)
      GC.SuppressFinalize(Me)           
   End Sub

   ' Protected implementation of Dispose pattern.
   Protected Overridable Sub Dispose(disposing As Boolean)
      If disposed Then Return

      If disposing Then
         handle.Dispose()
         ' Free any other managed objects here.
         '
      End If

      ' Free any unmanaged objects here.
      '
      disposed = True
   End Sub
End Class
```

> [!NOTE] 
> The previous example uses a [SafeFileHandle](xref:Microsoft.Win32.SafeHandles.SafeFileHandle) object to illustrate the pattern; any object derived from [SafeHandle](xref:System.Runtime.InteropServices.SafeHandle) could be used instead. Note that the example does not properly instantiate its [SafeFileHandle](xref:Microsoft.Win32.SafeHandles.SafeFileHandle) object. 
 
Here's the general pattern for implementing the dispose pattern for a base class that overrides [Object.Finalize](xref:System.Object#System_Object_Finalize). 

```cs
using System;

class BaseClass : IDisposable
{
   // Flag: Has Dispose already been called?
   bool disposed = false;

   // Public implementation of Dispose pattern callable by consumers.
   public void Dispose()
   { 
      Dispose(true);
      GC.SuppressFinalize(this);           
   }

   // Protected implementation of Dispose pattern.
   protected virtual void Dispose(bool disposing)
   {
      if (disposed)
         return; 

      if (disposing) {
         // Free any other managed objects here.
         //
      }

      // Free any unmanaged objects here.
      //
      disposed = true;
   }

   ~BaseClass()
   {
      Dispose(false);
   }
}
```

```vb
Class BaseClass : Implements IDisposable
   ' Flag: Has Dispose already been called?
   Dim disposed As Boolean = False

   ' Public implementation of Dispose pattern callable by consumers.
   Public Sub Dispose() _
              Implements IDisposable.Dispose
      Dispose(True)
      GC.SuppressFinalize(Me)           
   End Sub

   ' Protected implementation of Dispose pattern.
   Protected Overridable Sub Dispose(disposing As Boolean)
      If disposed Then Return

      If disposing Then
         ' Free any other managed objects here.
         '
      End If

      ' Free any unmanaged objects here.
      '
      disposed = True
   End Sub

   Protected Overrides Sub Finalize()
      Dispose(False)      
   End Sub
End Class
```

> [!NOTE]
> In C#, you override [Object.Finalize](xref:System.Object#System_Object_Finalize) by defining a `destructor`. 


## Implementing the dispose pattern for a derived class

A class derived from a class that implements the [IDisposable](xref:System.IDisposable) interface shouldn't implement [IDisposable](xref:System.IDisposable), because the base class implementation of [IDisposable.Dispose](xref:System.IDisposable#System_IDisposable_Dispose) is inherited by its derived classes. Instead, to implement the dispose pattern for a derived class, you provide the following: 

* A `protected Dispose(Boolean)` method that overrides the base class method and performs the actual work of releasing the resources of the derived class. This method should also call the `Dispose(Boolean)` method of the base class and pass it a value of `true` for the *disposing* argument. 

* Either a class derived from [SafeHandle](xref:System.Runtime.InteropServices.SafeHandle) that wraps your unmanaged resource (recommended), or an override to the [Object.Finalize](xref:System.Object#System_Object_Finalize) method. The [SafeHandle](xref:System.Runtime.InteropServices.SafeHandle) class provides a finalizer that frees you from having to code one. If you do provide a finalizer, it should call the `Dispose(Boolean)` overload with a *disposing* argument of `false`. 

Here's the general pattern for implementing the dispose pattern for a derived class that uses a safe handle: 

```cs
using Microsoft.Win32.SafeHandles;
using System;
using System.Runtime.InteropServices;

class DerivedClass : BaseClass
{
   // Flag: Has Dispose already been called?
   bool disposed = false;
   // Instantiate a SafeHandle instance.
   SafeHandle handle = new SafeFileHandle(IntPtr.Zero, true);

   // Protected implementation of Dispose pattern.
   protected override void Dispose(bool disposing)
   {
      if (disposed)
         return; 

      if (disposing) {
         handle.Dispose();
         // Free any other managed objects here.
         //
      }

      // Free any unmanaged objects here.
      //

      disposed = true;
      // Call base class implementation.
      base.Dispose(disposing);
   }
}
```

```vb
Imports Microsoft.Win32.SafeHandles
Imports System.Runtime.InteropServices

Class DerivedClass : Inherits BaseClass 
   ' Flag: Has Dispose already been called?
   Dim disposed As Boolean = False
   ' Instantiate a SafeHandle instance.
   Dim handle As SafeHandle = New SafeFileHandle(IntPtr.Zero, True)

   ' Protected implementation of Dispose pattern.
   Protected Overrides Sub Dispose(disposing As Boolean)
      If disposed Then Return

      If disposing Then
         handle.Dispose()
         ' Free any other managed objects here.
         '
      End If

      ' Free any unmanaged objects here.
      '
      disposed = True

      ' Call base class implementation.
      MyBase.Dispose(disposing)
   End Sub
End Class
```

> [!NOTE] 
> The previous example uses a [SafeFileHandle](xref:Microsoft.Win32.SafeHandles.SafeFileHandle) object to illustrate the pattern; any object derived from [SafeHandle](xref:System.Runtime.InteropServices.SafeHandle) could be used instead. Note that the example does not properly instantiate its [SafeFileHandle](xref:Microsoft.Win32.SafeHandles.SafeFileHandle) object. 

Here's the general pattern for implementing the dispose pattern for a derived class that overrides [Object.Finalize](xref:System.Object#System_Object_Finalize):

```cs
using System;

class DerivedClass : BaseClass
{
   // Flag: Has Dispose already been called?
   bool disposed = false;

   // Protected implementation of Dispose pattern.
   protected override void Dispose(bool disposing)
   {
      if (disposed)
         return; 

      if (disposing) {
         // Free any other managed objects here.
         //
      }

      // Free any unmanaged objects here.
      //
      disposed = true;

      // Call the base class implementation.
      base.Dispose(disposing);
   }

   ~DerivedClass()
   {
      Dispose(false);
   }
}
```

```vb
Class DerivedClass : Inherits BaseClass
   ' Flag: Has Dispose already been called?
   Dim disposed As Boolean = False

   ' Protected implementation of Dispose pattern.
   Protected Overrides Sub Dispose(disposing As Boolean)
      If disposed Then Return

      If disposing Then
         ' Free any other managed objects here.
         '
      End If

      ' Free any unmanaged objects here.
      '
      disposed = True

      ' Call the base class implementation.
      MyBase.Dispose(disposing)
   End Sub

   Protected Overrides Sub Finalize()
      Dispose(False)
   End Sub 
End Class
```

> [!NOTE]
> In C#, you override [Object.Finalize](xref:System.Object#System_Object_Finalize) by defining a `destructor`.

## Using safe handles

Writing code for an object's finalizer is a complex task that can cause problems if not done correctly. Therefore, we recommend that you construct [System.Runtime.InteropServices.SafeHandle](xref:System.Runtime.InteropServices.SafeHandle) objects instead of implementing a finalizer.

Classes derived from the [System.Runtime.InteropServices.SafeHandle](xref:System.Runtime.InteropServices.SafeHandle) class simplify object lifetime issues by assigning and releasing handles without interruption. They contain a critical finalizer that is guaranteed to run while an application domain is unloading. The following derived classes in the [Microsoft.Win32.SafeHandles](xref:Microsoft.Win32.SafeHandles) namespace provide safe handles: 

* The [SafeFileHandle](xref:Microsoft.Win32.SafeHandles.SafeFileHandle), [SafeMemoryMappedFileHandle](xref:Microsoft.Win32.SafeHandles.SafeMemoryMappedFileHandle), and [SafePipeHandle](xref:Microsoft.Win32.SafeHandles.SafePipeHandle) class, for files, memory mapped files, and pipes. 

* The [SafeMemoryMappedViewHandle](xref:Microsoft.Win32.SafeHandles.SafeMemoryMappedViewHandle) class, for memory views. 

* The [SafeNCryptKeyHandle](xref:Microsoft.Win32.SafeHandles.SafeNCryptKeyHandle), [SafeNCryptProviderHandle](xref:Microsoft.Win32.SafeHandles.SafeNCryptProviderHandle), and [SafeNCryptSecretHandle](xref:Microsoft.Win32.SafeHandles.SafeNCryptSecretHandle) classes, for cryptography constructs. 

* The [SafeRegistryHandle](xref:Microsoft.Win32.SafeHandles.SafeRegistryHandle) class, for registry keys. 

* The [SafeWaitHandle](xref:Microsoft.Win32.SafeHandles.SafeWaitHandle) class, for wait handles. 

## Using a safe handle to implement the dispose pattern for a base class

The following example illustrates the dispose pattern for a base class, `DisposableStreamResource`, that uses a safe handle to encapsulate unmanaged resources. It defines a `DisposableResource` class that uses a [SafeFileHandle](xref:Microsoft.Win32.SafeHandles.SafeFileHandle) to wrap a [Stream](xref:System.IO.Stream) object that represents an open file. The `DisposableResource` method also includes a single property, `Size`, that returns the total number of bytes in the file stream. 

```cs
using Microsoft.Win32.SafeHandles;
using System;
using System.IO;
using System.Runtime.InteropServices;

public class DisposableStreamResource : IDisposable
{
   // Define constants.
   protected const uint GENERIC_READ = 0x80000000;
   protected const uint FILE_SHARE_READ = 0x00000001;
   protected const uint OPEN_EXISTING = 3;
   protected const uint FILE_ATTRIBUTE_NORMAL = 0x80;
   protected IntPtr INVALID_HANDLE_VALUE = new IntPtr(-1);
   private const int INVALID_FILE_SIZE = unchecked((int) 0xFFFFFFFF);

   // Define Windows APIs.
   [DllImport("kernel32.dll", EntryPoint = "CreateFileW", CharSet = CharSet.Unicode)]
   protected static extern IntPtr CreateFile (
                                  string lpFileName, uint dwDesiredAccess, 
                                  uint dwShareMode, IntPtr lpSecurityAttributes, 
                                  uint dwCreationDisposition, uint dwFlagsAndAttributes, 
                                  IntPtr hTemplateFile);

   [DllImport("kernel32.dll")]
   private static extern int GetFileSize(SafeFileHandle hFile, out int lpFileSizeHigh);

   // Define locals.
   private bool disposed = false;
   private SafeFileHandle safeHandle; 
   private long bufferSize;
   private int upperWord;

   public DisposableStreamResource(string filename)
   {
      if (filename == null)
         throw new ArgumentNullException("The filename cannot be null.");
      else if (filename == "")
         throw new ArgumentException("The filename cannot be an empty string.");

      IntPtr handle = CreateFile(filename, GENERIC_READ, FILE_SHARE_READ,
                                 IntPtr.Zero, OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL,
                                 IntPtr.Zero);
      if (handle != INVALID_HANDLE_VALUE)
         safeHandle = new SafeFileHandle(handle, true);
      else
         throw new FileNotFoundException(String.Format("Cannot open '{0}'", filename));

      // Get file size.
      bufferSize = GetFileSize(safeHandle, out upperWord); 
      if (bufferSize == INVALID_FILE_SIZE)
         bufferSize = -1;
      else if (upperWord > 0) 
         bufferSize = (((long)upperWord) << 32) + bufferSize;
   }

   public long Size 
   { get { return bufferSize; } }

   public void Dispose()
   {
      Dispose(true);
      GC.SuppressFinalize(this);
   }           

   protected virtual void Dispose(bool disposing)
   {
      if (disposed) return;

      // Dispose of managed resources here.
      if (disposing)
         safeHandle.Dispose();

      // Dispose of any unmanaged resources not wrapped in safe handles.

      disposed = true;
   }  
}
```

```vb
Imports Microsoft.Win32.SafeHandles
Imports System.IO

Public Class DisposableStreamResource : Implements IDisposable
   ' Define constants.
   Protected Const GENERIC_READ As UInteger = &H80000000ui
   Protected Const FILE_SHARE_READ As UInteger = &H0000000i
   Protected Const OPEN_EXISTING As UInteger = 3
   Protected Const FILE_ATTRIBUTE_NORMAL As UInteger = &H80
   Protected INVALID_HANDLE_VALUE As New IntPtr(-1)
   Private Const INVALID_FILE_SIZE As Integer = &HFFFFFFFF

   ' Define Windows APIs.
   Protected Declare Function CreateFile Lib "kernel32" Alias "CreateFileA" (
                                         lpFileName As String, dwDesiredAccess As UInt32, 
                                         dwShareMode As UInt32, lpSecurityAttributes As IntPtr, 
                                         dwCreationDisposition As UInt32, dwFlagsAndAttributes As UInt32, 
                                         hTemplateFile As IntPtr) As IntPtr
   Private Declare Function GetFileSize Lib "kernel32" (hFile As SafeFileHandle, 
                                                        ByRef lpFileSizeHigh As Integer) As Integer

   ' Define locals.
   Private disposed As Boolean = False
   Private safeHandle As SafeFileHandle 
   Private bufferSize As Long 
   Private upperWord As Integer

   Public Sub New(filename As String)
      If filename Is Nothing Then
         Throw New ArgumentNullException("The filename cannot be null.")
      Else If filename = ""
         Throw New ArgumentException("The filename cannot be an empty string.")
      End If

      Dim handle As IntPtr = CreateFile(filename, GENERIC_READ, FILE_SHARE_READ,
                                        IntPtr.Zero, OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL,
                                        IntPtr.Zero)
      If handle <> INVALID_HANDLE_VALUE Then
         safeHandle = New SafeFileHandle(handle, True)
      Else
         Throw New FileNotFoundException(String.Format("Cannot open '{0}'", filename))
      End If

      ' Get file size.
      bufferSize = GetFileSize(safeHandle, upperWord) 
      If bufferSize = INVALID_FILE_SIZE Then
         bufferSize = -1
      Else If upperWord > 0 Then 
         bufferSize = (CLng(upperWord) << 32) + bufferSize
      End If     
   End Sub

   Public ReadOnly Property Size As Long
      Get
         Return bufferSize
      End Get
   End Property

   Public Sub Dispose() _
              Implements IDisposable.Dispose
      Dispose(True)
      GC.SuppressFinalize(Me)
   End Sub           

   Protected Overridable Sub Dispose(disposing As Boolean)
      If disposed Then Exit Sub

      ' Dispose of managed resources here.
      If disposing Then
         safeHandle.Dispose()
      End If

      ' Dispose of any unmanaged resources not wrapped in safe handles.

      disposed = True
   End Sub  
End Class
```

## Using a safe handle to implement the dispose pattern for a derived class

The following example illustrates the dispose pattern for a derived class, `DisposableStreamResource2`, that inherits from the `DisposableStreamResource` class presented in the previous example. The class adds an additional method, `WriteFileInfo`, and uses a [SafeFileHandle](xref:Microsoft.Win32.SafeHandles.SafeFileHandle) object to wrap the handle of the writable file. 

```cs
using Microsoft.Win32.SafeHandles;
using System;
using System.IO;
using System.Runtime.InteropServices;
using System.Threading;

public class DisposableStreamResource2 : DisposableStreamResource
{
   // Define additional constants.
   protected const uint GENERIC_WRITE = 0x40000000; 
   protected const uint OPEN_ALWAYS = 4;

   // Define additional APIs.
   [DllImport("kernel32.dll")]   
   protected static extern bool WriteFile(
                                SafeFileHandle safeHandle, string lpBuffer, 
                                int nNumberOfBytesToWrite, out int lpNumberOfBytesWritten,
                                IntPtr lpOverlapped);

   // Define locals.
   private bool disposed = false;
   private string filename;
   private bool created = false;
   private SafeFileHandle safeHandle;

   public DisposableStreamResource2(string filename) : base(filename)
   {
      this.filename = filename;
   }

   public void WriteFileInfo()
   { 
      if (! created) {
         IntPtr hFile = CreateFile(xref:".\FileInfo.txt", GENERIC_WRITE, 0, 
                                   IntPtr.Zero, OPEN_ALWAYS, 
                                   FILE_ATTRIBUTE_NORMAL, IntPtr.Zero);
         if (hFile != INVALID_HANDLE_VALUE)
            safeHandle = new SafeFileHandle(hFile, true);
         else
            throw new IOException("Unable to create output file.");

         created = true;
      }

      string output = String.Format("{0}: {1:N0} bytes\n", filename, Size);
      int bytesWritten;
      bool result = WriteFile(safeHandle, output, output.Length, out bytesWritten, IntPtr.Zero);                                     
   }

   protected new virtual void Dispose(bool disposing)
   {
      if (disposed) return;

      // Release any managed resources here.
      if (disposing)
         safeHandle.Dispose();

      disposed = true;

      // Release any unmanaged resources not wrapped by safe handles here.

      // Call the base class implementation.
      base.Dispose(true);
   }
}
```

```vb
Imports Microsoft.Win32.SafeHandles
Imports System.IO

Public Class DisposableStreamResource2 : Inherits DisposableStreamResource
   ' Define additional constants.
   Protected Const GENERIC_WRITE As Integer = &H40000000 
   Protected Const OPEN_ALWAYS As Integer = 4

   ' Define additional APIs.
   Protected Declare Function WriteFile Lib "kernel32.dll" (
                              safeHandle As SafeFileHandle, lpBuffer As String, 
                              nNumberOfBytesToWrite As Integer, ByRef lpNumberOfBytesWritten As Integer,
                              lpOverlapped As Object) As Boolean

   ' Define locals.
   Private disposed As Boolean = False
   Private filename As String
   Private created As Boolean = False
   Private safeHandle As SafeFileHandle

   Public Sub New(filename As String)
      MyBase.New(filename)
      Me.filename = filename
   End Sub

   Public Sub WriteFileInfo() 
      If Not created Then
         Dim hFile As IntPtr = CreateFile(".\FileInfo.txt", GENERIC_WRITE, 0, 
                                          IntPtr.Zero, OPEN_ALWAYS, 
                                          FILE_ATTRIBUTE_NORMAL, IntPtr.Zero)
         If hFile <> INVALID_HANDLE_VALUE Then
            safeHandle = New SafeFileHandle(hFile, True)
         Else
            Throw New IOException("Unable to create output file.")
         End If
         created = True
      End If
      Dim output As String = String.Format("{0}: {1:N0} bytes {2}", filename, Size, 
                                           vbCrLf)
      WriteFile(safeHandle, output, output.Length, 0&, Nothing)                                     
   End Sub

   Protected Overloads Overridable Sub Dispose(disposing As Boolean)
      If disposed Then Exit Sub

      ' Release any managed resources here.
      If disposing Then
         safeHandle.Dispose()
      End If

      disposed = True
      ' Release any unmanaged resources not wrapped by safe handles here.

      ' Call the base class implementation.
      MyBase.Dispose(True)
   End Sub
End Class
```

## See Also

[SuppressFinalize](xref:System.GC#System_GC_SuppressFinalize_System_Object_)

[IDisposable](xref:System.IDisposable)

[IDisposable.Dispose](xref:System.IDisposable#System_IDisposable_Dispose)

[Microsoft.Win32.SafeHandles](xref:Microsoft.Win32.SafeHandles)

[System.Runtime.InteropServices.SafeHandle](xref:System.Runtime.InteropServices.SafeHandle)

[IDisposable.Dispose](xref:System.IDisposable#System_IDisposable_Dispose)
