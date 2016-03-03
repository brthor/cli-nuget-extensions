# Intro

## Overview
Currently the dotnet cli supports the resolution of "commands" through references to a package under the "tools" node in a project.json file. The implementation for this lives in the `dotnet-restore` command.

The purpose of this doc is to detail how the switch for this logic to be place in NuGet might be designed. 
The intention is to capture any contracts which are created between the dotnet cli and NuGet in the creation of this scenario, call out any open questions, and detail a proposed design. It is intended to be technical from a high level.

## Concepts used in the doc
`dotnet extensions`: refers to commands named like `dotnet-foo` which come from outside the cli. This is not intended to be a proposition for the final name (although it seems fitting), but just a consistent convention in this doc

`dotnet extension packages`: NuGet packages which contain dotnet extension. This is the core topic of this doc

`consumer project`: The project which references a dotnet extension like `dotnet-foo` with the intention of being able to call `dotnet foo` from the command line

## Core Scenarios
- An extension package with a single tool which runs cross platform
- Packages with a different name than the tool

#### Potential Scenarios (not v1)
- Metapackages which reference multiple extension packages (transitive tools)
- Platform Specific Tools
- Non-Managed Tools

## Quick Reference

### Consumer Project.json Example (Including Aliases)
```
{
    "version": "1.0.0-*",

    "dependencies": {
        "NETStandard.Library": "1.0.0-*"
    },

    "frameworks": {
        "netstandardapp1.5": { }
    },

    "tools": {
        "dotnet-tool-package": "1.0.0"
        "another-tool-package": {
            "version": "1.0.0", 
            "alias": "foo"
        }
    }
}
```

### Tool Project.json Example
```
{
    "version": "1.0.0-*",

    "dependencies": {
        "NETStandard.Library": "1.0.0-*"
    },

    "frameworks": {
        "netstandardapp1.5": { }
    }
}
```


### Extension Hive Location, and artifacts
Base Path: `~/.nuget/packages/.tools`

ToolPaths: 
`~/.nuget/packages/.tools/{package_name}/{version}/{tfm}/project.lock.json`

# Detailed Explanations

## Restoration of Extension Package Flow (NuGet side)
The flow of an extension package restoration begins with `dotnet restore` invocation.
The basic flow is described in the diagram below.

![](https://github.com/brthor/cli-nuget-extensions/blob/master/nugetflow.png)

NuGet will add an additional logic to recognize the `tools` top level node in project.json. The basic entry in a **consumer project** would look like:

```
{
    "version": "1.0.0-*",

    "dependencies": {
        "NETStandard.Library": "1.0.0-*"
    },

    "frameworks": {
        "netstandardapp1.5": { }
    },

    "tools": {
        "dotnet-tool-package": "1.0.0"
    }
}
```
The `"dotnet-tool-package"` is a regular depdendency node, supporting all the parameters a dependency node under `netstandardapp1.5` might.

For each of the sub nodes in `tools`, an independent package restore will be kicked off. This restore will follow the same logic of any other package restore of a top level project. The extension is essentially an independent project, with independent dependencies and is treated as such. It does not affect the package graph of the consumer project in any way.

The dependencies of the tools packages will be determined from the nuspec generated for the package like they would for any regular package. 

Imports information for any TfM of tool will be explicitly *not included*. Any tool which relies on `imports` to perform a successful restore must manually pack the dlls from the packages that require `imports` as a part of it's own package.

After the restore finished, the project.lock.json of the tool package would be saved to the package location in the tools hive. The deps file of this package would be copied from the package hive to the extensions hive. 

More specifically, the project.lock.json and deps file would be placed in `~/.nuget/packages/.tools/dotnet-tool-package/1.0.0/{tfm}`.

Summary of NuGet changes:
- [ ] Recognize "tools" node in project.json
- [ ] Kick off independent restores for each tool package node in `tool` node
- [ ] After each of the independent restores, save the project.lock.json in `~/.nuget/packages/.tools/{package_name}/{version}/{tfm}`
- [ ] `pack` must include the deps file in a tool package (See open questions)

## Invocation of Extension Command (dotnet cli side)

The dotnet driver will use a command resolution strategy for finding commands from the tools hive.

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

    "tools": {
        "dotnet-tool-package": "1.0.0"
    }
}
```

The General flow is described in the diagram:

![](https://github.com/brthor/cli-nuget-extensions/blob/master/cliflow.png)

The changes to the driver are almost purely changing the search strategy it uses to find the extension .dll file.
It is very reliant on nuget to place things correctly, so it's worth calling out the specific contracts:

1. dotnet driver expects that a restore of an extensions package will place a project.lock.json and deps file in (`~/.nuget/packages/.tools/{package_name}/1.0.0/{tfm}`)
2. dotnet driver expects that only one tfm directory will be present in `~/.nuget/packages/.tools/{package_name}/1.0.0`

Changes to `dotnet`
- [ ] Change Extension Resolution Strategy to search the tools hive for a deps file and project.lock.json.
- [ ] Change Extension Resolution Strategy to use the saved project.lock.json to find a dll matching the invoked tool name. (ex. user invokes `dotnet foo`, driver looks for `dotnet-foo.dll`)
- [ ] Add understanding of `tools` top level node in project.json

## Additional Scenarios

** Removed until further discussion **

# Meeting Notes

## 2016-??-??

- Tools do not flow between P2P dependencies.
- Tool Packages must use `netstandardapp1.5` or a compatible TFM.
- If a tool package has more than one TfM some rules will be followed to pick only one for restoration (see open questions).

## 2016-03-03

- First stage of NuGet work is just in the project.json format and restore code. We will not think about the UI (and marking a package as a tool) quite yet.
- For now, the TFM to use in the `.tools` directory is hard coded to be `netstandardapp1.5`. The TFM itself may be renamed before release.
- We will only support the version string under the `tools` node (not a complex object like the `dependencies` node) or an object with only the `version` property. No other properties are supported yet.
- NuGet can generate the project.lock.json file that goes into the `.tools` directory by building the project.json model in memory based off of the tool's .nuspec file.

# Open Questions
- Does pack always pack a deps file, or only for tool packages?
- What precedence rules do we use to determine the TFM for a tool package which defines more than one?
  - For now we are hard coding `netstandardapp1.5`.
- If an extension is using a different framework than the project, can it still run?
- Does the extension need to pull the project dependencies or not?
- Can an extension define extra dependencies (not sure if we have a scenario for that)
- How does pack know we are packing an extension? 
  - Is there a project.json definition to say we are building an extension package?
  - Maybe an argument passed to pack?
- What is the source of truth for the location of the packages folder? This is very important for the driver.
