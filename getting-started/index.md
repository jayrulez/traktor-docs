---
layout: default
permalink: /getting-started/
title: Getting Started
nav_order: 2
has_children: true
---

# Getting Started with Traktor

This guide walks you through getting Traktor, creating your first project, and making a simple scene. By the end, you'll have a working development environment and understand the basic workflow for making games with Traktor.

## System Requirements

Before you start, make sure your system meets these requirements:

**Graphics card:** You need a Vulkan 1.3 capable GPU. Most modern graphics cards from NVIDIA, AMD, or Intel support this. If you're unsure, check your GPU manufacturer's specifications.

**Operating system:** Traktor's editor and build tools run on **Windows** or **Linux**. macOS is not currently supported for development (though you can build games for macOS as a deployment target).

**Build tools:** On Windows, you need **Visual Studio 2022 or newer** with C++ development tools installed. On Linux, you need **GCC** or **Clang** with C++17 support.

---

## Installation

You have two options for getting Traktor: download prebuilt binaries or build from source.

### Download Prebuilt Binaries

The fastest way to try Traktor is to download the latest binary release from [GitHub Releases](https://github.com/apistol78/traktor/releases). Download the appropriate package for your platform, extract it, and you're ready to run the editor.

**Important:** Prebuilt releases may lag behind the latest source code. If you encounter bugs or want the newest features, build from source instead.

### Build from Source (Recommended)

Building from source gives you the latest code, bug fixes, and features. It also lets you modify the engine and contribute improvements.

**1. Clone the repository:**

```bash
git clone --recursive https://github.com/apistol78/traktor.git
cd traktor
```

The `--recursive` flag is important. It fetches all submodules (third-party libraries) that Traktor depends on.

**2. Build for your platform:**

Follow the platform-specific instructions:
- **[Build on Windows](./build-windows/)** - Using Visual Studio
- **[Build on Linux](./build-linux/)** - Using GCC or Clang

These guides walk you through configuring your build environment, compiling the engine, and running the editor for the first time.

---

## Next Steps

Once you have Traktor built and the editor running, follow these tutorials in order:

1. **[Tutorial 00: Create Your First Project](tutorial-00-first-project/)** - Create a workspace and understand the project structure
2. **[Tutorial 01: Create Your First Scene](tutorial-01-first-scene/)** - Build a lit scene with geometry
3. **[Tutorial 02: Add Your First Script](tutorial-02-first-script/)** - Write Lua code to bring your scene to life

---

## Continue Learning

After completing the tutorials above, here's where to go next:

**Learn the workflow** - Read the [Editor Documentation](../editor/) to understand the Database, Pipeline, Scene Editor, and deployment tools. The editor is your primary workspace, so mastering it will make development much faster.

**Understand the architecture** - The [Architecture](../engine/architecture/) guide explains how Traktor is organized. The module system, memory management, and core concepts. This helps you understand how everything fits together.

**Add gameplay logic** - The [Scripting](../engine/scripting/) guide teaches you how to write Lua scripts for gameplay. You'll learn about components, the update loop, input handling, and more.

**Explore the systems** - Dive into the [Engine Documentation](../engine/) to learn about rendering, physics, audio, animation, AI, and all the other systems available.

**See examples** - Check the [Example Projects](./example-projects/) page for sample projects that demonstrate various features.

**Get help** - Join the [Discord community](https://discord.gg/fSMrww2B7C) to ask questions, share projects, and connect with other Traktor developers. The community is friendly and helpful for both beginners and experienced users.
