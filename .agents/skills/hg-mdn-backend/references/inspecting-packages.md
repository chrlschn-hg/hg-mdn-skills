# Inspecting Packages and Assemblies

Look an API up instead of guessing its shape.  `dotnet-inspect` reads NuGet packages and local assemblies without a project: signatures, overloads, versions, and API differences.  Run it through `dnx` (no install):

```shell
# Versions of a package
dnx dotnet-inspect -y -- package Microsoft.Extensions.Logging.Abstractions --versions

# Find a type by pattern
dnx dotnet-inspect -y -- find '*Bounded*' --package System.Threading.Channels

# A type's shape, and one member's overloads with signatures
dnx dotnet-inspect -y -- type JsonSerializer --package System.Text.Json
dnx dotnet-inspect -y -- member Channel CreateBounded --package System.Threading.Channels

# What changed between versions (breaking changes only)
dnx dotnet-inspect -y -- diff --package System.Text.Json@9.0.0..10.0.0 --breaking
```

- `--package <Name>[@version]` for NuGet, `--project <path>` for a project's own references, `--library <dll>` for a built assembly.
- Add `--table` or `--shape` for compact views; `dnx dotnet-inspect skill` prints an agent skill describing every command.
- Use it **before** writing code against a library you have not used (recent versions differ from what you remember), when a call does not compile, and before an upgrade (`diff --breaking`).
- `dnx dotnet-inspect -y -- --help` lists the rest.
