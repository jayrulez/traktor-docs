---
layout: default
permalink: /engine/
title: Engine
nav_order: 3
has_children: true
---

# Engine Documentation

Welcome to the Traktor engine documentation. This section covers the technical architecture, core systems, and APIs you'll work with when building games with Traktor.

These docs are organized from foundational concepts to specialized systems. Start with **Architecture** to understand how everything fits together, then explore **Runtime** and **World** to learn the application model and entity-component system. From there, dive into the specific systems your game needs: rendering, physics, animation, audio, scripting, and more.

Each page explains not just *how* to use a system, but *why* it works that way. You'll find practical code examples in both C++ and Lua, best practices learned from real projects, and references to the actual source code when you need to go deeper.

## Core Systems

- [Architecture](architecture/) - Engine design and core systems
- [Runtime](runtime/) - Application framework and lifecycle
- [World](world/) - Entity-component system
- [Resources](resources/) - Asset loading and management

## Game Systems

- [Physics](physics/) - Rigid body dynamics and character controllers
- [Scripting](scripting/) - Lua integration and gameplay programming
- [Input](input/) - Keyboard, mouse, and gamepad handling
- [AI](ai/) - Navigation mesh and pathfinding

## Graphics and Audio

- [Rendering](render/) - Vulkan-based renderer with ray tracing
- [Animation](animation/) - Skeletal animation and state graphs
- [Theater](theater/) - Cinematic cutscenes and scene animation
- [Particles](particles/) - GPU particle effects
- [Audio](audio/) - 3D spatial audio and music
- [UI](ui/) - User interface system
- [Terrain](terrain/) - Heightfield terrain
- [Weather](weather/) - Dynamic weather effects
- [Networking](networking/) - Multiplayer and online services
