---
layout: default
permalink: /getting-started/build-windows/
title: Build for Windows
parent: Getting Started
nav_order: 2
---

## Build for Windows

### Prerequisites

Before you run the Traktor build scripts, make sure you have the following prerequisites installed and working.

- **Visual Studio 2022** (any edition) or later with the following workloads;
  - *Desktop development with C++*
  - *Game development with C++*
- **LunarG Vulkan SDK**. Latest version preferred.
  - This will be downloaded for you in the build step if you don't have a global installation, but it's recommended to install it yourself.

### Build

Building the engine and editor for Windows is very straightforward, requiring only three to four steps;

1. Run `scripts/update-3rdp.bat` to make sure all third-party dependencies are downloaded and set up correctly. Make sure you run this script *first* whenever you build the engine, as the solution generated in the following step won't build correctly if you don't.

> [!NOTE]
> `scripts/update-3rdp.bat` will call `scripts/config.bat` to check if your environment contains a global Vulkan SDK installation. If it does, that dependency will be skipped in favor of the global SDK. Otherwise, it will download the full Vulkan SDK and use it locally. Due to this, it's recommended you have the Vulkan SDK already installed and the `VULKAN_SDK` system environment variable set to its install path to avoid potential duplicate downloads of a rather heavy dependency.

2. After third-party dependencies are finished downloading, run `scripts/build-projects-vs2022-win64.bat` to generate the Traktor solution and projects.

3. Open the generated solution at `build/Win64/Traktor Win64.sln` in the version of Visual Studio you have installed.

> [!NOTE]
> Though the generated solutions are intended for use in Visual Studio 2022, they will work without any problem in Visual Studio 2026 (Insiders or otherwise) once you let VS updgrade them to the newer format, and will build without a hitch.

4. Make sure the startup application is set to **Traktor.Editor.App**, then build the solution and run the editor.
