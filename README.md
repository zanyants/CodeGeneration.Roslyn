# Roslyn-based Code Generation

[![Build status](https://ci.appveyor.com/api/projects/status/y81yha5rm3wvlycv/branch/master?svg=true)](https://ci.appveyor.com/project/AArnott/codegeneration-roslyn/branch/master)
[![NuGet package](https://img.shields.io/nuget/v/CodeGeneration.Roslyn.svg)][NuPkg]

Assists in performing Roslyn-based code generation during a build.
This includes design-time support, such that code generation can respond to
changes made in hand-authored code files by generating new code that shows
up to Intellisense as soon as the file is saved to disk.

## How to write your own code generator

In this walkthrough, we will define a code generator that replicates any class
your code generation attribute is applied to, but with a suffix appended to its name.

### Define code generator

This must be done in a library that targets netstandard1.3 or net46 (or later).
Your generator cannot be defined in the same project that will have code generated
for it because code generation runs *before* the receiving project is itself compiled.

Install the [CodeGeneration.Roslyn][NuPkg] NuGet Package.
You may need to define this property in your .NET SDK netstandard project to
workaround the problem with the `Microsoft.Composition` NuGet package:

```xml
<PackageTargetFallback>$(PackageTargetFallback);portable-net45+win8+wp8+wpa81;</PackageTargetFallback>
```

Define the generator class:

```csharp
using CodeGeneration.Roslyn;
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.CSharp;
using Microsoft.CodeAnalysis.CSharp.Syntax;
using System;
using System.Threading;
using System.Threading.Tasks;
using Validation;

public class DuplicateWithSuffixGenerator : ICodeGenerator
{
    private readonly string suffix;

    public DuplicateWithSuffixGenerator(AttributeData attributeData)
    {
        Requires.NotNull(attributeData, nameof(attributeData));

        this.suffix = (string)attributeData.ConstructorArguments[0].Value;
    }

    public Task<SyntaxList<MemberDeclarationSyntax>> GenerateAsync(TransformationContext context, IProgress<Diagnostic> progress, CancellationToken cancellationToken)
    {
        var results = SyntaxFactory.List<MemberDeclarationSyntax>();

        // Our generator is applied to any class that our attribute is applied to.
        var applyToClass = (ClassDeclarationSyntax)context.ProcessingMember;

        // Apply a suffix to the name of a copy of the class.
        var copy = applyToClass
            .WithIdentifier(SyntaxFactory.Identifier(applyToClass.Identifier.ValueText + this.suffix));

        // Return our modified copy. It will be added to the user's project for compilation.
        results = results.Add(copy);
        return Task.FromResult<SyntaxList<MemberDeclarationSyntax>>(results);
    }
}
```

### Define attribute

To activate your code generator, you need to define an attribute that can be
applied to the class to be copied. This attribute may be defined in the same
assembly as defines your code generator, but since your code generator must
be defined in a netstandard1.3 or net46 library, this may limit which projects
can apply your attribute. So define your attribute in another assembly if
it must be applied to projects that target older platforms.

If your attributes are in their own project, you must install the
[CodeGeneration.Roslyn.Attributes][AttrNuPkg] package to your attributes project.

Define your attribute class.
For this walkthrough, we will assume that the attributes are defined in the same
netstandard1.3 project that defines the generator which allows us to use the more
convenient `typeof` syntax when declaring the code generator type.
If the attributes and code generator classes were in separate assemblies, you must
specify the assembly-qualified name of the generator type as a string instead.


```csharp
using CodeGeneration.Roslyn;
using System;
using System.Diagnostics;
using Validation;

[AttributeUsage(AttributeTargets.Class, Inherited = false, AllowMultiple = true)]
[CodeGenerationAttribute(typeof(DuplicateWithSuffixGenerator))]
[Conditional("CodeGeneration")]
public class DuplicateWithSuffixAttribute : Attribute
{
    public DuplicateWithSuffixAttribute(string suffix)
    {
        Requires.NotNullOrEmpty(suffix, nameof(suffix));

        this.Suffix = suffix;
    }

    public string Suffix { get; }
}
```

The `[Conditional("CodeGeneration")]` attribute is not necessary, but it will prevent
the attribute from persisting in the compiled assembly that consumes it, leaving it
instead as just a compile-time hint to code generation, and allowing you to not ship
with a dependency on your code generation assembly.

## Apply code generation

The attribute may not be applied in the same assembly that defines the generator.
This is because the code generator must be compiled in order to execute before compiling
the project that applies the attribute.

Applying code generation is incredibly simple. Just add the attribute on any type
or member supported by the attribute and generator you wrote. Note you will need to
add a project reference to the project that defines the attribute.

```csharp
[DuplicateWithSuffix("A")]
public class Foo
{
}
```

Install the [CodeGeneration.Roslyn.BuildTime][BuildTimeNuPkg] package into the
project that uses your attribute. You may set `PrivateAssets="all"` on this reference
because this is a build-time only package.

You can then consume the generated code at design-time:

```csharp
[Fact]
public void SimpleGenerationWorks()
{
    var foo = new Foo();
    var fooA = new FooA();
}
```

You should see Intellisense help you in all your interactions with `FooA`.
If you execute Go To Definition on it, Visual Studio will open the generated code file
that actually defines `FooA`, and you'll notice it's exactly like `Foo`, just renamed
as our code generator defined it to be.

### Shared Projects

When using shared projects and partial classes across the definitions of your class in shared and platform projects:

* The code generation attributes should be applied only to the files in the shared project
  (or in other words, the attribute should only be applied once per type to avoid multiple generator invocations).
* The MSBuild:GenerateCodeFromAttributes custom tool must be applied to every file we want to auto generate code from.

## Developing your code generator

Your code generator can be defined in a project in the same solution as the solution with
the project that consumes it. You can edit your code generator and build the solution
to immediately see the effects of your changes on the generated code.

## Packaging up your code generator for others' use

You can also package up your code generator as a NuGet package for others to install
and use. Your NuGet package should include a dependency on the `CodeGeneration.Roslyn.BuildTime`
that matches the version of `CodeGeneration.Roslyn` that you used to produce your generator.
For example, if you used version 0.4.6 of this project, your .nuspec file would include this tag:

```xml
<dependency id="CodeGeneration.Roslyn.BuildTime" version="0.4.6" />
```

In addition to this dependency, your NuGet package should include a `build` folder with an
MSBuild file (either a .props or a .targets file) that defines an `GeneratorAssemblySearchPaths`
MSBuild item pointing to the folder containing your code generator assembly and its dependencies.
For example your package should have a `build\MyPackage.targets` file with this content:

```xml
<?xml version="1.0" encoding="utf-8" ?>
<Project ToolsVersion="14.0" DefaultTargets="Build" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
  <ItemGroup>
    <GeneratorAssemblySearchPaths Include="$(MSBuildThisFileDirectory)..\tools" />
  </ItemGroup>
</Project>
```

Then your package should also have a `tools` folder that contains your code generator and any of the runtime
dependencies it needs *besides those delivered by the `CodeGeneration.Roslyn.BuildTime` package*.

Your attributes assembly should be placed under your package's `lib` folder` so consuming projects
can apply those attributes.

Your consumers should depend on your package, and the required dotnet CLI tool,
so that the MSBuild Task can invoke the `dotnet codegen` command line tool:

```xml
<ItemGroup>
  <PackageReference Include="YourCodeGenPackage" Version="1.2.3" PrivateAssets="all" />
  <DotNetCliToolReference Include="dotnet-codegen" Version="0.4.6" />
</ItemGroup>
```

Also, any consumer must have a nuget.config file with at least this content:

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <add key="api.nuget.org" value="https://api.nuget.org/v3/index.json" />
    <add key="corefxlab" value="https://dotnet.myget.org/F/dotnet-corefxlab/api/v3/index.json" />
  </packageSources>
</configuration>
```

Make sure that the DotNetCliToolReference version matches the version of the
`CodeGeneration.Roslyn` package your package depends on.

[NuPkg]: https://nuget.org/packages/CodeGeneration.Roslyn
[BuildTimeNuPkg]: https://nuget.org/packages/CodeGeneration.Roslyn.BuildTime
[AttrNuPkg]: https://nuget.org/packages/CodeGeneration.Roslyn.Attributes
