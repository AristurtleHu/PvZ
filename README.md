# How to Play
Run "PvZ" in "build/bin" (Linux) (or build from cmake file in other os)

# PvZ Game (C++ with CMake)

This is a **Plants vs Zombies** (PvZ)-inspired game built using **C++** and **CMake**. The game features basic plant and zombie interactions, including animations for actions like shooting and movement (e.g., Pole Vaulting Zombie).

## Features

- **Game Objects**: Plants, zombies, and bullets are all modeled as `GameObject`s. Each object is created with distinct behaviors and deleted when their state is considered "dead".
- **Core Components**: 
  - `BasePlant`, `BaseZombie`, and `BaseBullet` are the foundational classes, extending from `BaseGameObject` to encapsulate common functionality.
  - Animations are included for actions like shooting and movement to enhance the gaming experience.

More details in design - [guide](DesignDoc.md)

## Architecture

- **Object-Oriented Design**: All entities in the game (plants, zombies, bullets, etc.) are derived from the `BaseGameObject` class.
- **Memory Management**: GameObjects are automatically deleted when they reach the "dead" state, ensuring efficient resource usage.
