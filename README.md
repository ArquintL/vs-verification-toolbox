# VS Verification Toolbox

[![Test Status](https://github.com/viperproject/vs-verification-toolbox/workflows/NPM%20Tests/badge.svg?branch=master)](https://github.com/viperproject/vs-verification-toolbox/actions?query=workflow%3A%22NPM+Tests%22+branch%3Amaster)
[![License: MPL 2.0](https://img.shields.io/badge/License-MPL%202.0-brightgreen.svg)](./LICENSE)

This module provides several useful tools for writing VS Code extensions that verify code.

As it is not yet published on NPM, in order to use this module, you'll have to use some special syntax in your package.json:

```json
"vs-verification-toolbox": "https://github.com/viperproject/vs-verification-toolbox.git"
```

If `npm install` still complains about the package not being in the registry, try deleting package-lock.json to regenerate it.

## Features

### General Utilities

The `util` package offers several convenient utilities and wrappers.

#### File System Locations

A `Location` represents a directory on the file system, providing a simple API for navigating through the hierarchy. Example usage:

```typescript
const root = new Location("/tmp/folder");
root.basePath; // "/tmp/folder"
root.enclosingFolder; // Location(/tmp)
root.child("sub", "path"); // Location(/tmp/folder/sub/path)
root.path("sub", "path"); // "/tmp/folder/sub/path"
root.executable("run"); // unix-based: "/tmp/folder/run"; windows: "/tmp/folder/run.exe"
await root.mkdir(); // ensures the folder exists
await root.exists(); // true
```

#### Platform Checking

`Platform` is a simple enum representing the three major platforms: `Linux`, `Windows`, `Mac`.

It also provides a constant `currentPlatform` equal to the current platform, or `null` if it's not one of those three.

#### Progress Reporting

The `ProgressListener` type describes a callback for reporting progress from long-running asynchronous tasks. It takes the fraction of the task that is complete, as well as a user-facing description of the current step.

Additionally, there is a very convenient function `withProgressInWindow` that runs an asynchronous task, forwarding the progress it reports (to the given `ProgressListener`) to the VS Code window as a notification with a nice progress bar.

### Dependency Installation

The `dependencies` package provides a way to define your dependencies declaratively and install them asynchronously.

An example dependency might be defined as follows:

```typescript
const myDependency = new Dependency<"remote" | "local">(
    path.join(context.globalStoragePath, "myDependency"), // path to install folder
    ["remote",
        new InstallerSequence([
            new FileDownloader("https://remote.com/file.zip"),
            new ZipExtractor("unzipped"), // name for the unzipped folder
        ])
    ],
    ["local", new LocalReference("/path/to/external/local/installation")]
);
```

This defines a dependency with two sources (`remote` and `local`) which is installed within a folder named `myDependency` within the global storage folder VS Code provides to your extension, e.g. `~/Library/Application Support/Code/User/globalStorage/<your extension slug>/myDependency`. Of course, you could specify any old path, but this one is probably a good choice for your use case.

If you then run `await myDependency.ensureInstalled("remote")`, it will give you a `Location` referencing the location of the local installation from the `remote` source. If it's already installed, it won't do any extra work; otherwise it will download the file at https://remote.com/file.zip and unzip it into a folder named `unzipped`. Either way, you get back the location of the `unzipped` folder. You can force a reinstall even if it already exists by calling `update` instead of `ensureInstalled`.

If you instead run `await myDependency.ensureInstalled("local")` (or `update`, `install`, etc.), it will simply give you back a reference to the folder at /path/to/external/local/installation, after ensuring that folder actually exists.

