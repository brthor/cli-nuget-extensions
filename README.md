# Intro

## Overview
Currently the dotnet cli supports the resolution of "commands" through references to a package under the "tools" node in a project.json file. The implementation for this lives in the `dotnet-restore` command.

The purpose of this doc is to detail how the switch for this logic to be place in NuGet might be designed. 
The intention is to capture any contracts which are created between the dotnet cli and NuGet in the creation of this scenario, call out any open questions, and detail a proposed design. It is primarily technical but not to the level of implementation.

## Terminology (just to avoid confusion)
`dotnet extensions`: refers to commands named like `dotnet-foo` which come from outside the cli. This is not intended to be a proposition for the final name (although it seems fitting), but just a consistent convention in this doc

`dotnet extension packages`: NuGet packages which contain dotnet extension. This is the core topic of this doc

# Core Scenario
- An extension package with a single tool which runs cross platform

# Interesting Scenarios
- Metapackages which reference multiple extension packages (transitive tools)
- Platform Specific Tools
- Non-Managed Tools
- Packages with a different name than the tool

## Project File

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
        "dotnet-tool-package": { "version": "1.0.0", "target": "package" }
    }
}
```

OR 

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
        "dotnet-tool-package": "1.0.0"
    }
}
```

## Design for an Extension Package with a single tool

The flow of this design begins with `dotnet restore`. `dotnet restore` 
