---
title: "How to simulate AutoMapper that works during the build time"
description: "Generate mapping code on build with Roslyn"
date: 2019-12-01T00:09:00+02:00
tags : ["dotnet", "C#", "roslyn", "AutoMapper", "code generation", "mapping"]
highlight: true
image: "splashscreen.jpg"
isBlogpost: true
---

Almost two years ago I created the very first version of [MappingGenerator](https://github.com/cezarypiatek/MappingGenerator). Since then, I've put a lot of work in this project, adding new functions and improving the mapping generation algorithm with __14 releases__ (__43 issues/feature requests__ closed) in the meantime. With over __5.5k downloads__ from the marketplace and __380 stars__ on Github, it looks like there is quite a market demand for this kind of tool (even though my [coffee button](https://www.buymeacoffee.com/tmAJLYvWy) statistics indicate something different). In the meantime, I got a couple of feature requests to implement some kind of mechanism that allows tracking changes of mapped classes and synchronize the mapping code in response to these changes. I was resisting for some time because it seemed to be a quite complicated problem but after a while, I decided to give it a try and make something that will somehow satisfy all those requirements. In this blog post I'm going to describe how to create a tool for generating code during the build process and how I used it to create auto-synchronizing mapping classes.

## How to generate code on build

Implementing a mechanism that tracks code changes and applies amendments to mapping code (especially when the modifications are allowed) seems to be very challenging, so the easiest option was to generate the whole mapping code during the build. This solves the problem of change tracking but confines the ability to apply custom code modifications. Anyway, the problem of generating code during the build stage seems to be very compelling, so besides the tradeoffs, I started working on it. 

There is a concept of [Source Generators](https://github.com/dotnet/roslyn/blob/master/docs/features/generators.md) described in `Roslyn` documentation. Unfortunately, the implementation of this compiler feature has not been finish so far. There are a couple of open-source projects such as
[Uno.SourceGeneration](https://github.com/unoplatform/Uno.SourceGeneration) and [CodeGeneration.Roslyn](https://github.com/AArnott/CodeGeneration.Roslyn) which are trying to simulate such functionality. They basically work in similar way by triggering external program from the MsBuild target, which is invoked before the actual compilation, and including files generated by this external program into `Compile` item group. I tried to use `CodeGeneration.Roslyn` in the first approach but I came across a problem with loading plugins that contain external dependencies. I even tried to propose a solution with PR but the response time from the maintainer was too long. After reviewing the whole project, I've decided to completely rewrite it making the following changes:

- Use `Microsoft.CodeAnalysis.MSBuild` to load the C# project instead of building compilation unit manually
- Use .NET Core 3.0 features for loading generator plugins ([link](https://docs.microsoft.com/en-us/dotnet/core/tutorials/creating-app-with-plugin-support))
- Remove caching mechanism which in my opinion was built based on the wrong assumptions (`CodeGeneration.Roslyn` is only using files that trigger generator as cache dependency, rather than tracking source of all symbols used in generated code)
- Simplify the plugins API
- Add parallelism for documents processing
- Create SDK that simplify the process of plugins development

The source code of the new solution is available on Github as [SmartCodeGenerator](https://github.com/cezarypiatek/SmartCodeGenerator) project. Developing plugins for `SmartCodeGenerator` is quite straightforward - you can find a short and concise instruction how to create and consume custom plugins in the project's readme.

## Generate mapping code on build
I used my `SmartCodeGenerator` engine to build a plugin that generates mapping code during the build - I called it `MappingGenerator.OnBuildGenerator`. Here's a quick instruction how to use it:

1. Install `SmartCodeGenerator.Engine` nuget package
2. Install `MappingGenerator.OnBuildGenerator` nuget package
3. Add the following snippet into your codebase:

```csharp
using System;
using System.Diagnostics;

namespace MappingGenerator.OnBuildGenerator
{
    [AttributeUsage(AttributeTargets.Interface)]
    [Conditional("CodeGeneration")]
    public class MappingInterface : Attribute
    {
    }
}
```

Since now, for every interface marked with `[MappingInterface]`, `MappingGenerator.OnBuildGenerator` will generate an implementation during the build. Sample usage can look as follows:

```csharp
namespace TestGenerator
{
    [MappingInterface]
    public interface IUserMapper
    {
        UserDTO Map(UserEntity entity);
        UserEntity Map(UserDTO dto);
    }
}

```
After rebuilding the project, you should be able to use in the codebase `UserMapper` class which implements `IUserMapper` interface (sometimes there is a need for solution reload to make the intellisense and syntax highlighting work correctly). You should be able to easily navigate to the source code of the generated class which should be located under `IntermediateOutputPath` (by default the `obj` directory). You can watch `MappingGenerator.OnBuildGenerator` in action on the following video:

<div class="video-container">
<iframe width="853" height="480" src="https://www.youtube.com/embed/43tRxSEa11Y?rel=0" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen></iframe>
</div>


All the generated methods are virtual, so if you need to make some adjustments to the mapping logic, you can achieve it by inheriting from the generated class and overriding given method by adding extra mapping logic:

```csharp
public class CustomUserMapper: UserMapper
{
    public override UserDTO MapFrom(UserEntity user)
    {
        var dto = base.MapFrom(user);
        //TODO: Make some adjustment to dto here
        return dto;
    }
}
```