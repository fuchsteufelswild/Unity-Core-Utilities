# Unity Core Utilities

A comprehensive collection of robust, modular, and easy to extend and use for Unity. This package provides the foundational architecture for any game genre, featuring a data-driven core, performance-oriented pooling, a powerful procedural animation system, and more.

> This is not a standalone game but a library of core systems. For examples of real-world implementation, see the main project: [Unity FPS Foundation](https://github.com/fuchsteufelswild/Unity-FPS-Foundation).

# Dependencies
Unity Editor Toolbox: Used to provide enhanced editor functionality along with Editor Coroutine Utils. (Included/Referenced in the project).

Unity's New Input System: Required for the Input system module.

# Features & System Overview

---

## Data Architecture
A type-safe, automatically assigned unique ID based system for defining any game data (e.g Items, Surfaces, Categories).

- **Definition\<T>:** Base **ScriptableObject** class for all data definitions, automatically guaranteed to have unique ID.
- **DefinitionRegistry\<T>:** A central lazily-initialized database that tracks all instances of a definition type, queryable by ID or Name.
- **DefinitionReference\<T>:** A robust struct that serializes a reference to any **Definition**, with a custom property drawer for easy assignment
in the inspector.
- **CRTP Pattern:** Enforces strict 1-to-1 relationships between categories and their members at compile-time.

## Object Pooling
A high-performance easy-to-use pooling system for both plain C# object and Unity **GameObject**s. 

- **ObjectPoolingModule:** The central brain and registry for all pools, acquiring any element of type *T* or an element that was created from a specific prefab template is done through this class as well.
- **ScenePoolController:** Manages lifetime and categorization of pooled objects per scene.
- **Zero-Intrusion Pooling:** Unity objects are pooled in **ManagedSceneObjectPool** via a wrapper component (**PooledSceneObject**), which is like a component that acts as an interface for pooling system, eliminating the need to modify existing custom **MonoBehaviour** scripts.
- **IUnityObjectPoolListener:** Allows any component on the same **GameObject** to react to an object's pool lifecyle events (Acquire, Release).

## Procedural Animation System
A physics-based system for smooth, dynamic animations that can be used for Cameras, UI, FPS characters, and more.

- **Physically-Based Spring Motion:** Core calculations use damped harmonic oscillation for natural-feeling motion (**PhysicsSpring.cs**).
- **Force Types:** **SpringForce** (constant) or **CurveForce** (Animation Curve-based) motions.
- **Motion:** Standalone motions that do not change depending on the game state, or ones that are used accumulate and manage force motions.
- **DataMotion:** Data-based motions that are based on specific *type* of **MotionData** which can be changed depending on the game state (e.g aiming, swimming, having weapon). These motions set their used **MotionData** when receiving call from **MotionDataBroadcaster** through implemented interface of **IMotionDataReceiver**.
- **MotionDataBroadcaster** & **MotionDataReceiver:** System for overriding motion data based on game state (e.g aiming, swimming). Works through **MotionDataPreset**, last added preset takes priority, or can be set *data-override* which act as a data that takes full priority over a specific *type* of **MotionData**.
- **MotionMixer:** Blends multiple active motions using fixed-step interpolation for seamless, fluid results.
- **Data-Driven Effect System:** Uses **AnimationData** and **DataBasedAnimator** to play specific effects when specific animation parameters change.

## Input System
A structured abstraction layer built for Unity's New Input System.

- **InputModule:** The central manager, **InputProfile**s are added to here and also controls the stack of *cancel action*s.
- **InputProfile:** Constrains which **InputHandler** *types* can be active, enabling easy context switching and control (e.g gameplay, building, menus).
- **InputHandler:** Modular components that can be extended to handle specific input actions.

## Saving & Loading System
A complete save game solution supporting complex object hierarchies and prefab instantiation.

- **SaveModule:** Central controller with configurable serialization logic and file system backends.
- **SaveableGameObject** & **ISaveableComponent:** Mark any **GameObject** and its components for saving via **SaveableGameObject** component, and by implementing **ISaveableComponent** any component can take part in the saving process. Handles both scene objects and prefab instances.
- **Dual GUID System:** Tracks objects by both instance ID and prefab ID for correct loading and instantiation.
- **SaveablePrefabRegistry:** Database of saveable prefabs.
- **Separate Metadata & Scene Data:** Save files are split for efficient browsing of save games without loading full scene data.

## Post-Processing Animation System
Animate Unity Post-Processing volume parameters (e.g Depth of Field, Vignette) for gameplay effects.

- **PostFXModule:** Core manager taking requests of animating **PostFXAnimationPreset** by passing it into pooled **PostFXAnimator**s.
- **PostFXAnimationPreset:** Collection of **PostFXAnimation\<T>**.
- **PostFXAnimation\<VolumeComponent>:** Collection of **PostFXAnimationParameter\<T>** that acts on the specific **VolumeComponent** of Post-Processing stack.
- **PostFXParameterAnimation\<T>:** Animation that acts on the specific value of type *T* of specific **VolumeComponent** used by **PostFXAnimation**.

## General Utilities
A vital toolkit for any Unity project.

- **GameModule:** A base class for creating singleton-like, pre-initialized manager systems as **ScriptableObject** assets.
- **Options<T>:** A generic system for creating automatically saved settings (in-game or developer options), works through generic **Option<T>** fields.
- **Parent-Child Behaviour System:** (**IParentBehaviour**, **IChildBehaviour**) enables O(1) lookup of child behaviours from a parent without direct dependencies.
- **Extension Methods:** Comprehensive utility classes and extensions for **Math**, **Physics**, and general Unity operations.
- **AudioModule:** A simple manager for playing 2D/3D audio via a pooled set of **AudioSource** components.
- **Editor Utilities:** A set of utility classes that ease the development of editor classes.
- **Experimental Tweening System:** A curiosity-driven implementation similar to DOTween.

# Getting Started

## Create a Data Definition

> Make sure created **ScriptableObject** asset is placed under path *Resources/Definitions/YourClass* (YourClassDefinition automatically stripped to YourClass in directory searching for a more logic search in the folder structre).

1. Create a new class inheriting from **Definition\<YourClass>**.
2. Your asset will automatically be registered and lazily-used by **DefinitionRegistry\<YourClass>**.
3. Use **DefinitionReference\<YourClass>** in any **MonoBehaviour** to reference it as a field.
4. If you want to test **CRTP** pattern, create two classes that both derive from **Definition\<T>** and refer to each other like the following:
```csharp
public sealed class YourMember<T, U> : 
  Definition<T> 
  where T : YourMember<T, U> 
  where U : YourCategory<U, T>
{}

public sealed class YourCategory<U, T> :
  Definition<U>
  where U : YourCategory<U, T>
  where T : YourMember<T, U>
{}
```

## Setting Up Object Pooling

1. Add **PooledSceneObject** component to the prefab you want to pool.
2. In code, create pool calling in two ways, one for creating pool for specific type *T*, and one for creating pool for a specific template prefab that is of type *T*:
```csharp
ObjectPoolingModule.Instance.RegisterPool(new ManagedSceneObjectPool<YourComponent>(gameObject.scene, YourPoolCategory.YourCategory, minSize, maxSize);
ObjectPoolingModule.Instance.RegisterPool(yourPrefab,
new ManagedSceneObjectPool<YourComponent>(yourPrefab, gameObject.scene, YourPoolCategory.YourCategory, minSize, maxSize);
```
3. Then in code request an instance:
```csharp
YourObject obj = ObjectPoolingModule.Instance.GetElement(yourPrefab);
```
4. Return it to the pool:
```csharp
obj.Release(delay);
```

# Links

- Example use in project: [**Unity FPS Foundation**](https://github.com/fuchsteufelswild/Unity-FPS-Foundation)
- [**Unity Editor Toolbox**](https://github.com/arimger/Unity-Editor-Toolbox)

# License

This project is licensed under MIT license.
