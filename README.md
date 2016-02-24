# Intro

## Overview
Currently the dotnet cli supports the resolution of "commands" through references to a package under the "tools" node in a project.json file. The implementation for this lives in the `dotnet-restore` command.

The purpose of this doc is to detail how the switch for this logic to be place in NuGet might be designed. 
The intention is to capture any contracts which are created between the dotnet cli and NuGet in the creation of this scenario, call out any open questions, and detail a proposed design. It is intended to be technical from a high level.

## Concepts used in the doc
`dotnet extensions`: refers to commands named like `dotnet-foo` which come from outside the cli. This is not intended to be a proposition for the final name (although it seems fitting), but just a consistent convention in this doc

`dotnet extension packages`: NuGet packages which contain dotnet extension. This is the core topic of this doc

`consumer project`: The project which references a dotnet extension like `dotnet-foo` with the intention of being able to call `dotnet foo` from the command line

## Core Scenario
- An extension package with a single tool which runs cross platform

#### Interesting Scenarios
- Metapackages which reference multiple extension packages (transitive tools)
- Platform Specific Tools
- Non-Managed Tools
- Packages with a different name than the tool

## Restoration of Extension Package Flow (NuGet side)
The flow of an extension package restoration begins with `dotnet restore` invocation.
The basic flow is described in the diagram below.

![](https://github.com/brthor/cli-nuget-extensions/blob/master/basicflow.png)

NuGet will add an additional logic to recognize the `dotnet-extensions` top level node in project.json. The entries would look like:

```
{
    "version": "1.0.0-*",

    "dependencies": {
        "NETStandard.Library": "1.0.0-*"
    },

    "frameworks": {
        "dnxcore50": { }
    },

    "dotnet-extensions": {
        "dotnet-extension-package": "1.0.0"
    }
}
```
The `"dotnet-extension-package"` is a regular depdendency node, supporting all the parameters a dependency node under `dnxcore50` might.

For each of the sub nodes in `dotnet-extensions`, an independent package restore will be kicked off. This restore will follow the same logic of any other package restore of a top level project. The extension is essentially an independent project, with independent dependencies and is treated as such.

The dependencies of the tools packages will be determined from the nuspec generated for the package like they would for any regular package. 
Imports information per TfM of the tool would need to be included to solve the issue of a extension package project constructed like so:
```
{
    "version": "1.0.0-*",
    "name": "dotnet-extension-package"

    "dependencies": {
        "NETStandard.Library": "1.0.0-*"
    },

    "frameworks": {
        "dnxcore50": { "imports": "portable-net452+win81" }
    }
}
```

In this case we need to preserve the imports statement or the restore will fail.

The NuSpec Could add a top level node like:
```
...
<dotnet-extension-data>
    <targetframework>
        <tfm>dnxcore50</tfm>
        <imports>portable-net45+win81</imports>
    </targetframework>
</dotnet-extension-data>
```

After the restore finished, the extension package project.lock.json would be copied from the `packages` hive to the `dotnet-extensions` hive. More specifically, the file `~/.nuget/packages/dotnet-extension-package/1.0.0/project.lock.json` would be copied to `~/.nuget/dotnet-extensions/dotnet-extension-package/1.0.0`. This is to create a safe location where the driver can load a project context and generate a deps file (addressed later) without mutating the package cache.

Summary of NuGet changes:
- [ ] Recognize "dotnet-extensions" node in project.json
- [ ] Kick off independent restores for each extension package node in `dotnet-extensions` node
- [ ] After each of the independent restores, copy the top level extension package to the special `dotnet-extensions` package hive
- [ ] Add understanding of "dotnet-extension-data" node in NuSpec of extension package
- [ ] Include imports from "dotnet-extension-data" in restore of each targetframework
- [ ] Add "dotnet-extension-data" nuspec output to `nuget pack`

## Invocation of Extension Command (dotnet cli side)

The dotnet driver will use a command resolution strategy for finding commands from the dotnet extensions package hive.

For a consumer project like so:
```
{
    "version": "1.0.0-*",

    "dependencies": {
        "NETStandard.Library": "1.0.0-*"
    },

    "frameworks": {
        "dnxcore50": { }
    },

    "dotnet-extensions": {
        "dotnet-extension-package": "1.0.0"
    }
}
```

The General flow is described in the diagram:

![](https://github.com/brthor/cli-nuget-extensions/blob/master/cli-extensions.png)

The changes to the driver are almost purely changing the search strategy it uses to find the extension .dll file.
It is very reliant on nuget to place things correctly, so it's worth calling out the specific contracts:

1. dotnet driver expects that a restore of an extensions package will have the unpacked package directory (`~/.nuget/packages/dotnet-extension-package/1.0.0`) copied to the dotnet-extensions package hive (`~/.nuget/dotnet-extensions/dotnet-extension-package/1.0.0`)
2. dotnet driver expects that a project.lock.json for the `dotnet-extension-package` will exist in the dotnet-extensions package hive (`~/.nuget/dotnet-extensions/dotnet-extension-package/1.0.0/project.lock.json`)
3. dotnet driver expects that the dotnet-extension-package package will have a runtime export with filename `dotnet-extension-package.dll`

Changes to `dotnet`
- [ ] Change Extension Resolution Strategy to search the dotnet-extensions package hive
- [ ] Add understanding of `dotnet-extensions` top level node in project.json


## Additional Scenarios

### Non-Managed extensions
A non-managed extension is any executable which is invokeable via `Process.Start` and named like `dotnet-extension`.

Changes to support this are purely on the dotnet side (assuming above changes to NuGet had been made). In this case, NuGet would restore the package just as if it were a managed extension package. 

The command extension itself could be included in the "native" export of the .nupkg. This may not be quite right but seems like a reasonable candidate.

The driver would have an additional resolution strategy for searching the native exports of the package. This would be a fallback.

To support easy creation this package we'd need to introduce a method of including native assets in `dotnet pack`.

Summary:
- [ ] Fallback Extension Resolution strategy in the dotnet driver for searching native exports of an extension package
- [ ] Method of including Native (?) Assets in a nuget package via dotnet pack

### Extension Aliases, Consumer and Extension Package sides

#### Command Extension Packages which Provide a tool with a different name than the package

This would allow a package like `Microsoft.DotNet.FooProduct.CliExtension` to include a tool `dotnet-fooproduct`

To support this, the logic of the driver command resolution strategy would need to change to do a constrained search through the `dotnet-extensions` package hive for the command being invoked. This search would be constrained to the package names and versions defined in the `dotnet-extensions` node of the project.json of the consumer project. If a .dll (or executable if non-managed commands were supported) matching the invoked command name was found, the driver would invoke that.

For example, given a consumer project detailed like so:
```
{
    "version": "1.0.0-*",
    "command": "foo",

    "dependencies": {
        "NETStandard.Library": "1.0.0-*"
    },

    "frameworks": {
        "dnxcore50": { }
    },

    "dotnet-extensions": {
        "tools": {
            "Microsoft.DotNet.FooProduct.CliExtension": "1.0.0"
        }
    }
}
```

If `dotnet fooproduct` was invoked from the command line, the driver would load the ProjectContext of the project.lock.json in `~/.nuget/dotnet-extensions/Microsoft.DotNet.FooProduct.CliExtension/1.0.0/` and search it's exports for dll matching `dotnet-fooproduct.dll`. The same logic is used for resolving a default in the case of multiples.

In the case of multiple nodes under `dotnet extensions` in the consumer project, each package would be searched (in order) and the first matching dll invoked.

EDIT: Nuspec Changes
```
<dotnet-extension-data>
    <command>
        <name>foo</name>
        <assembly>lib/runtimes/any/dotnet.fooextension.dll</assembly>
    </command>
    <targetframework>
        <tfm>dnxcore50</tfm>
        <imports>portable-net45+win81</imports>
    </targetframework>
</dotnet-extension-data>
```

Summary:
- [ ] Add Command resolution logic to search through `dotnet-extensions` package ProjectContexts for a .dll matching the invoked command


#### Consumer Project Which references multiple extension packages providing the same tool name
```
{
    "version": "1.0.0-*",

    "dependencies": {
        "NETStandard.Library": "1.0.0-*"
    },

    "frameworks": {
        "dnxcore50": { }
    },

    "dotnet-extensions": {
        "tools": {
            "Microsoft.DotNet.FooProduct": "1.0.0",
            "Microsoft.DotNet.FooProduct2": "1.0.0"
        },
        "aliases": {
            "package": "Microsoft.DotNet.FooProduct",
            "alias": "bar"
        }
    }
}
```

`dotnet bar` maps to `dotnet-foo` command in `Microsoft.DotNet.FooProduct`
`dotnet foo` maps to `dotnet-foo` command in `Microsoft.DotNet.FooProduct2`

### Metapackages which bring in multiple dotnet extensions

This would allow a package like `dotnet-powertools` to bring in multiple extensions with a single reference.

This could work by NuGet doing a restore as detailed above, looking for the <dotnet-extension-data> node in the NuSpecs of any dependencies of the metapackage. Whenever it finds this, it will kick off another independent extension restore of that package (basically recursive).

With extension aliases being supported (see above) the only change needed here would be to expand the Command resolution strategy to search the dependencies of packages in the `dotnet-extensions` node which exist in the dotnet extensions package hive. In other words, the alias based search detailed above would flow only to dependencies which are also extension packages.

Summary
- [ ] NuGet would transitively kick off independent extension restores for dependencies of a metapackage which are also extension packages
- [ ] Extend Alias based Command Resolution logic in the driver to flow through to package dependencies which are also extension packages


# Open Questions
- If an extension package defines more than one TfM, the driver will just use the first one. Should this be configurable. Can't think of any scenarios for this right now.
- If an extension is using a different framework than the project, can it still run?
- Does the extension need to pull the project dependencies or not?
- Can an extension define extra dependencies (not sure if we have a scenario for that)
- How does pack know we are packing an extension? Is there a project.json definition to say we are building an extension package?

- What is the source of truth for the location of the packages folder? This is very important for the driver.
