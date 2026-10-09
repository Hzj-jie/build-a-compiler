# Building a Programming Language from 0.1 - b2style_self_test

Previously, whenever we added a test for a new language feature, we had to write a piece of code expected to run, compile that code into a .NET assembly (as a Base64-encoded string), write [http://vb.net](http://vb.net) code to compile it (decoding the Base64 string back into bytes/raw string), execute the compiled code on the simulator, and compare the output against expectations. Although none of these steps can be skipped, the [http://vb.net](http://vb.net) code at the end is essentially repetitive. So let's consider how to simplify this process. Some parts are still necessary, of course—such as compiling the test code into the .NET assembly, since I don't want the tests to depend on where files are located on disk. The streamlined workflow looks roughly like this:

- Write several snippets of code expected to run, containing assertions.
- Compile this code into the .NET assembly.
- Implement a generic compile, run, and test routine in [http://vb.net](http://vb.net), using the assertions in the code to verify test results.

That way, adding new test cases only requires writing code with assertions; the second step is just running a command, and the third step is independent of the test code itself.

Because running the tests requires iterating over all test files, the previous `genall` ([service/tests/compiler/b2style/codegen/run.cmd](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/b2style/codegen/run.cmd)) pattern no longer works, as it turns each file into a separate byte string:

```bat
setlocal
path %PATH%;..\..\..\..\resource\gen;
genall --pattern=*.txt _b2style_test_data > b2style_test_data.vb
move /Y b2style_test_data.vb ..\
endlocal

```

While we could use reflection, why bother? It is much simpler to use [service/resource/tar](https://github.com/Hzj-jie/osi.net/tree/master/service/resource/tar).

This `tar` utility is very similar to POSIX `tar`: it packs multiple files into one archive and has built-in features like compression. Originally intended to store large numbers of small files together to save space, its usage here is slightly different. Since the packed data will still be converted into a Base64 string stored in source code, we use an [in_mem_fs](https://github.com/Hzj-jie/osi.net/blob/master/service/resource/tar/fs.vb#L194) to replace the file-based [default_fs](https://github.com/Hzj-jie/osi.net/blob/master/service/resource/tar/fs.vb#L54) used previously. Those are internal implementation details of `tar` that we won't elaborate on here; in short, they implement the functionality in [tar/gen](https://github.com/Hzj-jie/osi.net/blob/master/service/resource/tar/gen.vb).

```text
Public Shared Function read() As MemoryStream
    Dim o As New MemoryStream()
    assert(writer.in_mem(size Or 10 * 1024 * 1024,
                         New selector() With {.pattern = pattern Or "*"},
                         o).dump())
    Return o
End Function

Public Shared Function reader_of(ByVal input As MemoryStream) As reader
    Return reader.in_mem(input)
End Function

Public Shared Sub dump(ByVal input As MemoryStream, ByVal root As String)
    reader_of(input).foreach(Sub(ByVal file As String, ByVal m As MemoryStream)
                                 assert(Not file Is Nothing)
                                 assert(Not m Is Nothing)
                                 Try
                                     Directory.GetParent(Path.Combine(root, file)).Create()
                                 Catch ex As Exception
                                     assert(False, ex)
                                 End Try
                                 IO.File.WriteAllBytes(Path.Combine(root, file), m.bytes())
                             End Sub)
End Sub

```

This includes two sets of APIs: `read` writes files into memory in preparation for Base64 encoding, while `dump` and `reader_of` read from a memory buffer—the former extracting files into a specific directory, and the latter letting `tar.reader` perform other operations such as reading strings directly.

This neatly serves two different needs: `dump` can pack the `bstyle` and [b2style libraries](https://github.com/Hzj-jie/osi.net/tree/master/service/compiler/b2style/lib) directly and extract them at runtime, while `reader_of` can iterate over each test file to run tests.

[service/compiler/b2style/rewriter/nodes/includes.vb#L28](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b2style/rewriter/nodes/includes.vb#L28)

```vb.net
Public NotInheritable Class default_includes
    Private Shared ReadOnly folder As String = create_inc_folder()

    Private Shared Function create_inc_folder() As String
        Dim folder As String = Path.Combine(temp_folder, "b2style-inc")
        tar.gen.dump(b2style_lib.data, folder)
        Return folder
    End Function

    Public NotInheritable Class folders
        Inherits __do(Of vector(Of String))

        Protected Overrides Function at() As vector(Of String)
            Return +include_folders
        End Function
    End Class

```

[service/tests/compiler/b2style/b2style_self_test.vb#L26](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/b2style/b2style_self_test.vb#L26)

```vb.net
<test>
Private Shared Sub run()
    Dim a As New vector(Of Action)()
    tar.gen.reader_of(b2style_self_test_cases.data).foreach(
        Sub(ByVal name As String,
            ByVal enc_precision As Double,
            ByVal content As StreamReader)
            Dim text As String = content.ReadToEnd()
            a.emplace_back(Sub()
                               execute(name, text)
                           End Sub)
        End Sub)
    assertions.of(a).not_empty()
    concurrency_runner.execute(+a)
End Sub

```

Next, we need to add a few helpers on the `b2style` side. The current implementation is very simple and just uses `stdout` markers: for instance, `b2style::testing:assert_true(true)` outputs `Success: no extra information`, and `b2style::testing::finished()` marks the end of the test. The VB.NET test runner then checks whether all outputs match expectations.

[osi.net/service/compiler/b2style/lib/testing at master · Hzj-jie/osi.net](https://github.com/Hzj-jie/osi.net/tree/master/service/compiler/b2style/lib/testing)

```cpp
void assert_true(bool v, string msg) {
  _assertion_count++;
  string prefix;
  if (v) {
    prefix = "Success: ";
  } else {
    prefix = "Failure: ";
  }
  ::b2style::std_out(prefix);
  ::b2style::std_out(msg);
  ::b2style::std_out("\n");
}

void finished() {
  ::b2style::std_out("Total assertions: ");
  ::b2style::std_out(_assertion_count);
  ::b2style::std_out("\n");
}

```

We could also add a `loaded_method` to handle more complex assertions, though it doesn't seem necessary for now.

A test file looks like this: [service/tests/compiler/b2style/self-test/self-health.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/b2style/self-test/self-health.txt)

```cpp
#include <b2style.h>
#include <testing.h>

void main() {
  b2style::testing::assert_true(true);
  b2style::testing::finished();
}

```

The changes are in [osi.net/commit/cdbc332d1c1023ae04138ea9fc020e498d70e866](https://github.com/Hzj-jie/osi.net/commit/cdbc332d1c1023ae04138ea9fc020e498d70e866).
