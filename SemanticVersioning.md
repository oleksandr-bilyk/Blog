[Software Versioning](https://en.wikipedia.org/wiki/Software_versioning) and [Semantic Versioning](https://semver.org/) may be implemented using GitVersion that allows to have consistent versioning strategy cross build script, .NET projects and AzureDevOps pipeline.

GitVersion is powerful tool but good understanding of it may need through a learning path. 

1. [Semantic Versioning](https://semver.org/)
Given a version number MAJOR.MINOR.PATCH. Semantic Versioning provides version numbers and the way they change convey meaning about the underlying code and what has been modified from one version to the next.
1. [Control your GitHub releases with GitVersion and GitReleaseManager](https://www.youtube.com/watch?v=SlM02V1tkSc) 
This vides is related to GitHub but provides good expertise overview about GitVersion related topics.
1. [Install and use a .NET local tool using the .NET CLI](https://docs.microsoft.com/en-us/dotnet/core/tools/local-tools-how-to-use)
As prerequisite for GitVersion it is useful to have basic understanding of .NET local tools.
1. GitVersion site provides detailed documentation https://gitversion.net/ 
    1. [GitVersion .NET local tool installation](https://gitversion.net/docs/usage/cli/installation) 
GitVersion's command line interface can be installed and consumed in many different ways.
    1. [MSBuild Task](https://gitversion.net/docs/usage/msbuild)
The MSBuild Task for GitVersion — GitVersion.MsBuild — is a simple solution if you want to version your assemblies without writing any command line scripts or modifying your build process.
    1. [GitVersion Configuration](https://gitversion.net/docs/reference/configuration)
GitVersion is mainly powered by configuration. In configuration you may bump the next version explicitly, set Semantic Version mode and many other settings to control versioning strategy.
    1. [Version Variables](https://gitversion.net/docs/reference/variables)
Version variables are quite useful if you need different formats of the version number.
    1. [Version Incrementing](https://gitversion.net/docs/reference/version-increments)
GitVersion works with several workflows. This page is split up into two sections, first is all about understanding the approach GitVersion uses by default, and the second is how you can manually increment the version.
    1. [Versioning Modes](https://gitversion.net/docs/reference/modes/) 
Version supports a few different versioning modes. They are described in detail.

# Query current GitVersion from git repository for build scripting.
Build script may query current GitVersion JSON to generate some build artifacts. 
Current state of GitVersion [version variables](https://gitversion.net/docs/reference/variables) after [installation](https://gitversion.net/docs/usage/cli/installation) may be queried from command line.
Initialize .NET Tools once in source code folder.
``` dotnet tool restore ```
Call GitVersion as local DotNet tool.
``` dotnet tool run dotnet-gitversion ```
GitVersion tool console output may be parsed and used by any build or utility scripting in F#, C# or Powershell script. F#5 interactive [provides excellent scripting experience](https://docs.microsoft.com/en-us/dotnet/fsharp/tools/fsharp-interactive/). F# may call .NET base class library or any Nuget package to parses Major, Minor, Patch and SemVer from GitVersion.
``` fsharp
#r "nuget: FSharp.Json, 0.4.0"

open FSharp.Json
open System
open System.Diagnostics


type GitVersionResult = {
    Major: Int16
    Minor: Int16
    Patch: Int16
    SemVer: string
}

let getGitVersionResult(): GitVersionResult =
    let startInfo = 
        ProcessStartInfo(
            FileName = "dotnet", 
            Arguments = "tool run dotnet-gitversion",
            UseShellExecute = false,
            RedirectStandardOutput = true,
            CreateNoWindow = true)
    use gitVersionToolProcess = new Process(StartInfo = startInfo)
    gitVersionToolProcess.Start() |> ignore 
    let gitVersionToolOutput = gitVersionToolProcess.StandardOutput.ReadToEnd()
    Json.deserialize gitVersionToolOutput

let gitVersion = getGitVersionResult()
Console.WriteLine("SemVer: " + gitVersion.SemVer)
```

# Query current GitVersion from git repository for NPM and WebPack.
JS Single Page Application (SPA) developed with NPM package manager and WebPack bundler may get semantic version from GitVersion.
1. Get GitVersion semantic version in webpack.config.js
``` javascript
var child = require('child_process').execSync('dotnet tool run dotnet-gitversion');
var gitVersionResult = JSON.parse(child.toString())
console.log("DotNet-GitVersion-SemVer: " + gitVersionResult.SemVer)
```
2. Add DefinePlugin to WebPack configuration.
``` javascript
const config = {
    ...
    plugins: [
        ...
        new webpack.DefinePlugin({ __SemVer__: JSON.stringify(gitVersionResult.SemVer) })
    ]
    ...
};
```
3. Use defined global variable from application.
``` typescript
declare var __SemVer__: string;

function MyComponent() {
    return <Text>{"SPA version: " + __SemVer__}</Text> 
}   
```
