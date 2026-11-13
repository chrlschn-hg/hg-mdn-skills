# Modern .NET Skills

Provides agents instructions to build modern .NET full stack applications

## Writing Guide

- Terse, direct, concise guidance
- Use code samples to convey best practices; callout with comments
- Format C# code using `csharpier` to ensure idiomatic guidance
- Avoid over exposition; keep it dense ASD-STE100 rules
- Ensure consistency when working; check other docs on related topics
- Cross link and reference skills when there is overlap and agent should read both
- Ask for independent review before finalizing
- Use good comment hygeine; idiomatic to the language because other agents will follow your cue.  Comments explain "why", cross reference ideas, cite sources (e.g. reference URLs, etc.)

## Tooling

- [Mise](https://mise.jdx.dev/llms.txt) provides the tools: `mise install` reads `.config/mise.toml` (.NET SDK, `csharpier`, `csharprepl`); run them as `csharpier check <dir>` or `mise exec -- <tool>`
- macOS: if `dotnet --version` dies with SIGTERM after the first `mise install`, re-sign the install ad hoc (`-H` because the install directory is a symlink):

  ```sh
  find -H "$(mise where dotnet)" -type f \( -name dotnet -o -name '*.dylib' \) -exec codesign --force --sign - {} \;
  ```

## Verifying

- Verify behaviors using file-based C# apps (single file `.cs` file) or `.csx` scripts
- Can also use `csharprepl`
- Do not publish unverified code; build in a temporary directory as needed to verify

## csharp (C#) and dotnet (.NET)

- Current C# is 15; current .NET is 11 both are GA
- Promote modern practices; terse, functional, fluent chaining