# Flexible Grid Controller v1.0.1 Documentation

## Table of Contents
1. [Introduction](#introduction)
2. [Getting Started](#getting-started)
3. [Architecture Overview](#architecture-overview)
4. [Scene Setup](#scene-setup)
5. [Samples](#samples)
6. [Contact and Support](#contact-and-support)

## Introduction
The Flexible Grid Controller is a Unity tool designed to seamlessly integrate various input sources, such as keyboard or gamepad, into a wide array of grid-like structures in games. Whether it's a map in a turn-based game, board in tabletop game, a puzzle, or sequence of units, this tool will adapt to your existing setup to enhance interactivity. With its clean architecture, the Flexible Grid Controller can be easily integrated into different projects, without disturbing the original codebase. 

This documentation offers comprehensive guidance on the architecture and integrating the Flexible Grid Controller into your project, facilitating both basic setups and complex scenarios.

## Getting Started
### Prerequisites
- Unity version 2021.3 or higher

### Installation
Once purchased, there are two ways to add the package into your project:
- Open the Flexible Grid Controller page in the Asset Store, click "Open in Unity" button to download
- Open the "Package manager" in Unity and search for Flexible Grid Controller in "Your assets" tab to import 

## Architecture Overview
The Flexible Grid Controller was designed to support a broad spectrum of use cases and integrate with existing projects without changing the original codebase. It uses generic programming extensively, a technique enhancing code flexibility and reusability, but also introducing complexity. To address it, non-generic implementations are added to cover basic scenarios.

### Key Concepts
Let's outline the critical parts of the Flexible Grid Controller:

- **Interactable**: Fundamental element within the grid that users interact with. These include map tiles, a units in the game, ui elements, or anything else representable as a grid element. Basic operation, like selecting and clicking, are performed on interactables.
- **Controllable**: Manages all the interactable object within the grid. It can access individual elements of the grid to select element at given index and determine how the transition between interactables work.
- **Grid Controller**: The intermediary between the user input and the grid, handles the logic of executing commands on interactables
- **Grid Index**: A value indicating interactable's location within the grid, varying from simple 1D, 2D, or 3D coordinates to more complex key-based indexing for more complex grid structures.
- **Command**: Represents actions executed on interactables, including selecting or clicking
- **Input Provider**: Monitors user inputs (from keyboard, gamepad, etc.) and translates them into commands for the grid controller to process.

The diagram below shows how the system's parts work together.
```
  +-----+           
  |User |           
  +-----+           
     |              
     | (provides input)      
     v              
+-------------+     
|InputProvider|     
+-------------+     
     |              
     | (translates input to commands)      
     v              
  +--------+      
  |Command |      
  +--------+      
     |              
     | (executes on)        
     v              
+----------------+      +-------------+  
| GridController |      |Controllable |  
+----------------+      +-------------+  
     |                         |        
     | (manages)               | (aggregates)   
     v                         v        
+-------------+ <--------------+        
|Interactable |                       
+-------------+                       
     |                                 
     | (implements)                   
     v                                 
  (Click/Select/etc.)                  

```

### Detailed Component Description
#### Interactable
The Interactable is one of the basic concepts in the project that needs to be implemented in your project. The Interactable is defined as a generic interface presented below
```c#
public interface IInteractable<TIndex, TControllable, TInteractable>
    where TIndex : IGridIndex
    where TControllable : Controllable<TIndex, TControllable, TInteractable>
    where TInteractable : IInteractable<TIndex, TControllable, TInteractable>
{
    TIndex GetGridIndex(TControllable controllable);
}
```
The Interactable takes three type parameters: `TIndex` and `TControllable` define the type of Grid Index and Controllable the Interactable can work with. The last type parameter, `TInteractable` is a reference to its own type, which is a Recursive Generics design pattern. This pattern enhances type safety and flexibility, allowing for more precise method signatures and reducing the need for type casting

The interface defines a single method, `GetGridIndex`, which purpose it to return the index of the object on the grid. By itself, the interface doesn't define any more behaviour.

Building upon the `IInteractable` interface there are the `IClickable` and `ISelectable` interfaces defining basic operations: clicking and selecting.
```c#
public interface IClickable<TIndex, TControllable, TInteractable> : IInteractable<TIndex, TControllable, TInteractable>
        where TIndex : IGridIndex
        where TControllable : Controllable<TIndex, TInteractable, TControllable>
        where TInteractable : IClickable<TIndex, TControllable, TInteractable>
{
    UnityEvent<TInteractable, TControllable> Clicked { get; set; }
    void OnClick(TControllable controllable);
}
```
```c#
public interface ISelectable<TIndex, TControllable, TInteractable> : IInteractable<TIndex, TControllable, TInteractable>
        where TIndex : IGridIndex
        where TControllable : Controllable<TIndex, TInteractable, TControllable>
        where TInteractable : ISelectable<TIndex, TControllable, TInteractable>
{
    UnityEvent<TInteractable, TControllable> Selected { get; set; }
    UnityEvent<TInteractable, TControllable> Deselected { get; set; }

    void OnSelected(TControllable controllable);
    void OnDeselected(TControllable controllable);
}
```
The interfaces provide events related to their respective operations - clicking and selecting the grid element. Callback methods are defined for the click operation and object selection and deselection. These functions are where the actual behaviour of specific interactables should be implemented.

Next is the `IBasicInteractable` interface implementing both `IClickable` and `ISelectable`, combining basic operations in one entity.
```c#
public interface IBasicInteractable<TIndex, TControllable, TInteractable> : IInteractable<TIndex, TControllable, TInteractable>,
        ISelectable<TIndex, TControllable, TInteractable>,
        IClickable<TIndex, TControllable, TInteractable>
    where TIndex : IGridIndex
    where TControllable : Controllable<TIndex, TInteractable, TControllable>
    where TInteractable : IBasicInteractable<TIndex, TControllable, TInteractable>
{
```
Lastly, there are three non-generic interfaces defining entities that can be clicked and selected in 1D, 2D and 3D environments. These interfaces cover basic use cases and are most likely to be used in your projects. The interface below implements `IBasicInteractable` with `GridIndex2D` as the Grid Index parameter, and `BasicControllable2D` as the controllable parameter, making it usable in 2D scenarios. Finally, `IBasicControllable2D` itself is provided as the last type parameter, following the Recursive Generics pattern.
```c#
public interface IBasicInteractable2D : IBasicInteractable<GridIndex2D, BasicControllable2D, IBasicInteractable2D>
{
}
// 1D and 3D alternatives skipped for brevity
```
The class implementing this interface needs to include OnClicked, OnSelected and OnDeselected methods. These methods define the behaviour of the object when subject to the respective operations. Often this behaviour is already coded for mouse interaction in the original object, and the implementations can be just delegated to these methods. 

#### Controllable
The Controllable manages all interactables within the grid. You can think of the Controllable as the grid that you will be controlling. The project contains three non-generic implementations of Controllable for 1D, 2D and 3D environments. In basic use cases there's no need to do any coding for the Controllable.

The Controllable is implemented as an abstract class, inheriting from `MonoBehaviour`. It takes three type parameters, defining what `GridIndex` and `IInteractable` it can work with, and the `Controllable` type itself, following the Recursive Generics pattern.

```c#
public abstract class Controllable<TIndex, TInteractable, TControllable> : MonoBehaviour
    where TIndex : IGridIndex
    where TControllable : Controllable<TIndex, TInteractable, TControllable>
    where TInteractable : IInteractable<TIndex, TControllable, TInteractable>
{
    // Event indicating that the Controllable is ready to take input, e.g when the game starts
    public UnityEvent ControllableReady;
    // Event indicating that the Controllable is not accepting input, e.g when the game ends or is paused
    public UnityEvent ControllableNotReady;

    // Returns the interactable at the origin of the grid.
    public abstract TInteractable GetInteractableAtOrigin();
    // Returns the interactable at given index
    public abstract TInteractable GetInteractableAt(TIndex gridIndex);

    // Method returns all interactables within the grid
    public abstract IEnumerable<TInteractable> GetAllInteractables();
    // Method defines the transition from one grid element to the other
    public abstract TIndex GetNextPosition(TIndex currentIndex, TIndex indexDelta);
}
```
In the project there are three general-purpose Controllables, `BasicControllable1D`, `BasicControllable2D` and `BasicControllable3D`. These classes define the transitions between grid indices, and fetching interactables at given index. While not optimal, this implementation is universal and good enough to get started.
```c#
public abstract class BasicControllable2D : BasicControllableControllable<GridIndex2D, IBasicInteractable2D, BasicControllable2D>
{
    public override IBasicInteractable2D GetInteractableAt(GridIndex2D gridIndex)
    {
        var allInteractables = GetAllInteractables() ?? throw new InvalidOperationException("Implementation error: GetAllInteractables() returned null");
        var interactable = allInteractables.FirstOrDefault(c => c.GetGridIndex(this).Equals(gridIndex)) ?? throw new ArgumentException("No interactable at given grid index");
        return interactable;
    }

    public override IBasicInteractable2D GetInteractableAtOrigin()
    {
        var originIndex = new GridIndex2D(0, 0);
        return GetInteractableAt(originIndex);
    }

    public override GridIndex2D GetNextPosition(GridIndex2D currentIndex, GridIndex2D indexDelta)
    {
        var maxGridX = GetAllInteractables().Where(c => c.GetGridIndex(this).Y.Equals(currentIndex.Y)).Max(c => c.GetGridIndex(this).X);
        var maxGridY = GetAllInteractables().Where(c => c.GetGridIndex(this).X.Equals(currentIndex.X)).Max(c => c.GetGridIndex(this).Y);

        var minGridX = GetAllInteractables().Where(c => c.GetGridIndex(this).Y.Equals(currentIndex.Y)).Min(c => c.GetGridIndex(this).X);
        var minGridY = GetAllInteractables().Where(c => c.GetGridIndex(this).X.Equals(currentIndex.X)).Min(c => c.GetGridIndex(this).Y);

        var adjustedX = currentIndex.X - minGridX;
        var adjustedY = currentIndex.Y - minGridY;

        var maxX = maxGridX - minGridX + 1;
        var maxY = maxGridY - minGridY + 1;

        var newX = (adjustedX + indexDelta.X + maxX) % maxX;
        var newY = (adjustedY + indexDelta.Y + maxY) % maxY;

        newX += minGridX;
        newY += minGridY;

        return new GridIndex2D(newX, newY);
    }
```
Finally, there are three non-abstract classes `GridControllable1D`, `GridControllable2D` and `GridControllable3D`. These implementations assume that all interactables are available in the scene and will be assigned to the `_interactables` field in the editor. The script then selects the `IBasicInteractable` component from all the interactables, and invokes the `ControllableReady` event, indicating that the grid is ready to recieve input.
```c#
public class GridControllable2D : BasicControllable2D
{
    [SerializeField] private List<MonoBehaviour> _interactables;
    private IEnumerable<IBasicInteractable2D> _interactableInstances;

    private void Start()
    {
        // Collects all IBasicInteractable2D instances.
        _interactableInstances = _interactables.Select(t => t.GetComponent<IBasicInteractable2D>());
        
        // Once the interactables are collected, notify that the controllable is ready for interaction.
        ControllableReady.Invoke();
    }

    public override IEnumerable<IBasicInteractable2D> GetAllInteractables()
    {
        return _interactableInstances;
    }
}
```
#### Grid Controller
The Grid Controller is the intermediary between user inputs and the Controllable, processing commands and managing interactions. It is implemented as a generic abstract class inheriting from `MonoBehaviour`. The generic parameters `TIndex`, `TControllable` and `TInteractable` defines the type of Controllable, Interactable and Grid Index that the controller can work with.
```c#
public abstract class GridController<TIndex, TControllable, TInteractable> : MonoBehaviour
    where TIndex : IGridIndex
    where TControllable : Controllable<TIndex, TControllable, TInteractable>
    where TInteractable : IInteractable<TIndex, TControllable, TInteractable>
{
    // Currently "active" interactable. Commands are executed on the active interactable
    protected TInteractable _currentInteractable;
    [SerializeField] protected TControllable _controllable;
    [SerializeField] protected InputProvider<TIndex, TControllable, TInteractable> _inputProvider;
    
    protected virtual void Awake()
    {
        Assert.IsNotNull(_controllable, $"Controllable not set on {gameObject.name}");
        Assert.IsNotNull(_inputProvider, $"Input provider not set on {gameObject.name}");

        _controllable.ControllableReady.AddListener(InitializeInteraction);
        _controllable.ControllableNotReady.AddListener(StopInteraction);
    }
    
    protected virtual void InitializeInteraction()
    {
        _currentInteractable = _controllable.GetInteractableAtOrigin();
        _inputProvider.OnCommand.AddListener(ExecuteCommand);
    }
    
    protected virtual void StopInteraction()
    {
        _inputProvider.OnCommand.RemoveListener(ExecuteCommand);
    }

    // Executes the Command that arrived form the InputProvider
    protected virtual void ExecuteCommand(IInputCommand<TIndex, TControllable, TInteractable> command)
    {
        _currentInteractable = command.Execute(_currentInteractable, _controllable);
    }

    public virtual void SetInputProvider(InputProvider<TIndex, TControllable, TInteractable> inputProvider)
    {
        if(inputProvider == null)
        {
            throw new System.ArgumentNullException(nameof(inputProvider), $"{nameof(inputProvider)} parameter cannot be null");
        }

        this._inputProvider.OnCommand.RemoveListener(ExecuteCommand);
        this._inputProvider = inputProvider;

        this._inputProvider.OnCommand.AddListener(ExecuteCommand);
    }

    protected virtual void OnDestroy()
    {
        _inputProvider.OnCommand.RemoveListener(ExecuteCommand);
        _controllable.ControllableReady.RemoveListener(InitializeInteraction);
        _controllable.ControllableNotReady.RemoveListener(StopInteraction);
    }
```
The project contains three non-generic implementations of `GridController` that work in their respective environments, `BasicGridController1D`, `BasicGridController2D` and `BasicGridController3D`.
#### Grid Index
The Grid Index is a value indicating the position of the Interactable on the grid. It is implemented as a marker interface and as such does not define any methods or properties. The project includes implementation for 1D, 2D and 3D indices.
```c#
public interface IGridIndex
{
}
```
```c#
public readonly struct GridIndex2D : IGridIndex
{
    public int X { get; }
    public int Y { get; }

    public GridIndex2D(int x, int y)
    {
        X = x;
        Y = y;
    }
}
```
#### Command
The Command represents task to be executed on Interactable and Controllable. It is implemented as a generic interface, with `TIndex`, `TControllable` and `TInteractable` parameters defining the Grid Index, Controllable and Interactable that the Command can work with. The interface includes a single method, `Execute`, defining what happens when the command is activated.
```c#
public interface IInputCommand<TIndex, TControllable, TInteractable>
    where TIndex : IGridIndex
    where TControllable : Controllable<TIndex, TControllable, TInteractable>
    where TInteractable : IInteractable<TIndex, TControllable, TInteractable>
{
    TInteractable Execute(TInteractable interactable, TControllable controllable);
}
```
The project contains implementations for two commands, `ClickCommand` and `MoveSelectionCommand`, alongside their non-generic versions.

#### Input Provider
The Input Provider monitors user input and dispatches corresponding Commands in response. It is implemented as a generic abstract class, with `TIndex`, `TControllable` and `TInteractable` parameters defining the Grid Index, Controllable and Interactable that the Command can work with. Implementing the InputProvider class allows for integration of various input methods into the game, including keyboard, gamepad using Unity's legacy or new input systems.

The project includes implementations for both legacy and new input systems. The snippet below shows a sample Input Provider
```c#
public class BasicInputProvider2D : BasicInputProvider<GridIndex2D, BasicControllable2D, IBasicInteractable2D>
{
    public float DirectionalInputCooldownTime = 0.2f;
    protected float _lastInputTime;

    protected override void Update()
    {
        base.Update();
        float horizontal = Mathf.RoundToInt(UnityEngine.Input.GetAxisRaw("Horizontal"));
        float vertical = Mathf.RoundToInt(UnityEngine.Input.GetAxisRaw("Vertical"));

        var direction = new Vector2(horizontal, vertical);

        if (Time.time >= _lastInputTime + DirectionalInputCooldownTime && direction != Vector2.zero)
        {
            OnCommand?.Invoke(new MoveSelectionCommand2D(new GridIndex2D((int)direction.x, (int)direction.y)));
            _lastInputTime = Time.time;
        }

        if (UnityEngine.Input.GetButtonDown("Fire1"))
        {
            OnCommand?.Invoke(new ClickCommand2D());
        }
    }
}
```
## Scene Setup
Integrating the Flexible Grid Controller into your project is a straightforward process.  This chapter covers implementing the appropriate Interactable interface and configuring the Controllable, Grid Controller, and Input Provider.

### Interactable
The first step is to identify your scene's interactable elements. These might include tiles, units, UI elements, or anything else you want the player to interact with within the grid. Next, consider how these elements are arranged and the actions you want them to support (e.g., selection, clicking).

For common scenarios, the project provides ready-to-use interfaces that support clicking and selection within different grid layouts:
- `IBasicInteractable1D`: Ideal for 1D sequences of elements, such as a list of units.
- `IBasicInteractable2D`: Suited for classic 2D grid layouts, like tile maps.
- `IBasicInteractable3D`: Designed for 3D grid structures (e.g., voxel-based environments).

To represent interactable elements in your game, you'll have a class implement one of these interfaces. This class can either be a MonoBehaviour component attached directly to your grid elements, or a separate class managed by another object in your scene – the choice depends on your project's specific setup.

### Controllable
The initial choice of Interactable type determines subsequent options. The Controllable must be compatible with the selected Interactable type.  The project includes three Controllable types to align with common grid scenarios:
- `GridControllable1D`: Use this for interacting with `IBasicInteractable1D` elements, ideal for 1D sequences.
- `GridControllable2D`: Designed to work with `IBasicInteractable2D` elements, best suited for 2D grid layouts.
- `GridControllable3D`: Compatible with `IBasicInteractable3D` elements, fitting for 3D grid structures.

Add the appropriate Controllable component to a Game Object within your scene. In the Inspector, populate the "Interactables" field to establish the connection between the Controllable and your interactible elements.

### Input Provider
Input Provider is the component that listens to user input and translates it to actions. As before, earlier choices will guide the Input Provider selection. The project includes several Input Provider implementations compatible with BasicInteractable and GridControllable:
- `BasicInputProvider1D`, `BasicInputProvider2D`, `BasicInputProvider3D`: These support basic keyboard and legacy gamepad input.
- `BasicNewInputProvider1D`, `BasicNewInputProvider1D`, `BasicNewInputProvider1D`: Designed for utilizing Unity's New Input System, offering greater input mapping flexibility.

Add the selected Input Provider to an empty Game Object in the scene.

### Grid Controller
Finally, the Grid Controller component that connects all the components together. Earlier choices will determine the compatible Grid Controller. Here are the options included in the project:

- `BasicGridController1D`: Compatible with `BasicInteractable1D` and `GridControllable1D`.
- `BasicGridController2D`: Compatible with `BasicInteractable2D` and `GridControllable2D`.
- `BasicGridController3D`: Compatible with `BasicInteractable3D` and `GridControllable3D`.

Create a dedicated GameObject in your scene and attach the corresponding Grid Controller component. Within the Inspector, be sure to assign references to your previously configured Controllable and Input Provider.

The basic setup of the Flexible Grid Controller is ready. To explore its potential, the next chapter offers illustrative examples showcasing various grid-based interactions.

## Samples
Included in the package are five sample scenes designed to showcase the system's capabilities. The first scene is a mouse-controlled baseline, offering a simplistic representation of a turn-based game. This is followed by four tutorial scenes, each progressively introducing more advanced concepts, extending the baseline scene with keyboard and controller input support. This chapter provides detailed descriptions of all scenes.

The samples are located in the Assets/Examples folder. The baseline folder contains the common codebase, and each of the samples introduces it's own set of scripts, required for the integration.

### Baseline
The baseline scene consists of a 5x5 grid of tiles and four moveable units. It is controlled exclusively with the mouse, clicking on a unit selects it and clicking on a tile moves the unit to the destination. Visual cues aid in situation awareness. The scene focuses on interaction with grid elements, omitting pathfinding, combat, and other customary turn-based mechanics.

Presented below are the relevant scripts

The Tile is the basic building block of the grid, designed to react to mouse interactions. `OnHighlighted`, `OnDehighlighted` and `OnClicked` methods are triggered by corresponding mouse events. In response, the Tile invokes it's own set of events and applies visual cues.
```c#
public class Tile : MonoBehaviour
    {
        // The x and y coordinates of this Tile within the grid.
        private Vector2 _coords;
        // Reference to the <see cref="Unit"/> that is currently occupying the Tile.
        private Unit _currentUnit;
       
        public Vector2 Coords { get { return _coords; } set { _coords = value; } }
        public Unit CurrentUnit { get { return _currentUnit; } set { _currentUnit = value; } }

        public UnityEvent<Tile> TileClicked;
        public UnityEvent<Tile> TileHighlighted;
        public UnityEvent<Tile> TileDehighlighted;

        // Unity callback for the mouse enter event.
        protected void OnMouseEnter()
        {
            OnHighlighted();
        }

        // Unity callback for the mouse exit event.
        protected void OnMouseExit()
        {
            OnDehighlighted();
        }

        // Unity callback for the mouse click event.
        protected void OnMouseDown()
        {
            OnClicked();
        }

        // Called when the tile is highlighted (typically due to mouse events).
        public void OnHighlighted()
        {
            // Apply visual cue (ommited)
            TileHighlighted.Invoke(this);
        }

        // Called when the tile is dehighlighted (typically due to mouse events).
        public void OnDehighlighted()
        {
            // Apply visual cue (ommited)
            TileDehighlighted.Invoke(this);
        }

        // Called when the tile is clicked (typically due to mouse events).
        public void OnClicked()
        {
            // Apply visual cue (ommited)
            TileClicked.Invoke(this);
        }

        // Rest of the members ommited for brevity
    }
```
The Unit class represents a unit in the game, it can be clicked on, selected and moved to another tile in the grid. `OnClicked` method is calld in response to the OnMouseDown callback, and the Unit invokes it's own `UnitClidked` event in response.
```c#
public class Unit : MonoBehaviour
    {
        // The current Tile that the unit is occupying.
        [SerializeField] private Tile _currentTile;

        // Event triggered when the unit is clicked.
        public UnityEvent<Unit> UnitClicked;
        
        private bool _isSelected = false;

        // Unity callback for the mouse down event.
        public void OnMouseDown()
        {
            OnClicked();
        }

        // Moves the unit to a specified tile.
        public void MoveToTile(Tile tile)
        {
            // ...
            // Implementation skipped here for brevity
        }

        // Called when the unit is clicked (typically due to mouse events).
        public void OnClicked()
        {
            UnitClicked.Invoke(this);
        }

        // Called when the unit is selected.
        public void OnUnitSelected()
        {
            _isSelected = true;
            // Applying visual cue (ommited)
        }

        // Called when the unit is deselected.
        public void OnUnitDeselected()
        {
            _isSelected = false;
            // Applying visual cue (ommited)
        }

        // Rest of the members ommited for brevity
    }
```

Finally, there is the `GameController` script, the core of the game's logic. It responds to player interactions by listening for click events on tiles and units, managing unit selection, and enforcing movement rules.
```c#
public class GameController : MonoBehaviour
    {
        [SerializeField] private List<Unit> _units;
        [SerializeField] private List<Tile> _tiles;

        // Reference to the currently selected unit.
        private Unit _currentlySelectedUnit;

        // Initializes the game by setting up click listeners for tiles and units.
        private void Start()
        {
            foreach (Tile tile in _tiles)
            {
                tile.TileClicked.AddListener((tile) => OnTileClicked(tile));
            }

            foreach (Unit unit in _units)
            {
                unit.UnitClicked.AddListener((unit) => OnUnitClicked(unit));
            }
        }

        // Handles the event when a unit is clicked. Selects the unit if not already selected,
        // and deselects the previous one if a new unit is clicked.
        private void OnUnitClicked(Unit clickedUnit)
        {
            if (_currentlySelectedUnit == clickedUnit)
            {
                return;
            }

            if (_currentlySelectedUnit != null)
            {
                _currentlySelectedUnit.OnUnitDeselected();
            }

            _currentlySelectedUnit = clickedUnit;
            _currentlySelectedUnit.OnUnitSelected();
        }

        // Handles the event when a tile is clicked. If a unit is currently selected and the tile is empty,
        // moves the unit to that tile.
        private void OnTileClicked(Tile clickedTile)
        {
            if (_currentlySelectedUnit != null && clickedTile.CurrentUnit == null)
            {
                _currentlySelectedUnit.MoveToTile(clickedTile);
            }
        }
    }
```

### Sample 1
The first sample scene centers on fundamentals - grid navigation. The key decision here is choosing the right `Interactable` interface to implement. Focus on two factors: the grid layout and the actions you want to enable on your tiles. For this scene, with our 2D grid and tiles supporting selection and clicking, the `IBasicInteractable2D` interface fits perfectly!

The interface is implemented as a separate component - no need to edit the existing codebase. The implemented methods, `OnClick`, `OnSelected` and `OnDeselected` simply forward the actions to methods already existing on the underlying Tile.  The `GetGridIndex` method uses the Tile's Coords field to determine its position within the grid. The new component is attached to the original Tile prefab.
```c#
public class TileInteractable : MonoBehaviour, IBasicInteractable2D
{
    // Reference to the underlying Tile
    [SerializeField] private Tile _tileReference;

    public UnityEvent<IBasicInteractable2D, BasicControllable2D> Selected { get; set; } = new UnityEvent<IBasicInteractable2D, BasicControllable2D>();
    public UnityEvent<IBasicInteractable2D, BasicControllable2D> Deselected { get; set; } = new UnityEvent<IBasicInteractable2D, BasicControllable2D>();
    public UnityEvent<IBasicInteractable2D, BasicControllable2D> Clicked { get; set; } = new UnityEvent<IBasicInteractable2D, BasicControllable2D>();

    // Provides the grid index for this tile.
    public GridIndex2D GetGridIndex(BasicControllable2D controllable)
    {
        // The index is given by referencing the Coords field in the underlying tile
        return new GridIndex2D((int)_tileReference.Coords.x, (int)_tileReference.Coords.y);
    }

    // Delegates the click action to the underlying Tile.
    public void OnClick(BasicControllable2D controllable)
    { 
        _tileReference.OnClicked();
    }

    // Delegates the deselect action to the underlying Tile.
    public void OnDeselected(BasicControllable2D controllable)
    {
        _tileReference.OnDehighlighted();
    }

    // Delegates the select action to the underlying Tile.
    public void OnSelected(BasicControllable2D controllable)
    {
        _tileReference.OnHighlighted();
    }
}
```
Implementing the `IBasicInteractable2D` interface prepared the tiles for grid navigation. The choice of this interface dictates subsequent choices: `GridControllable2D` and subsequently `BasicGridController2D`. The `BasicInputProvider2D` complements the setup for input handling. Scene configuration of these components finalizes the integration process.

To summarize, integrating the Flexible Grid Controller required adding a new component, configuring scene elements, and minimal code changes.

### Sample 2
The second sample demonstrates an alternative approach to integrating `IBasicInteractable2D`. Instead of using a separate component, the interface can be implemented directly within the `Tile` class or, as shown here, a class derived from `Tile`. This method can be advantageous in specific cases. For example, if the necessary methods within your Tile-like class are not public, direct implementation allows access without exposing private members. Additionally, encapsulation and potential reduction in overhead might be appealing factors in smaller projects.
```c#
public class CustomTile : Tile, IBasicInteractable2D
    {
        public UnityEvent<IBasicInteractable2D, BasicControllable2D> Selected { get; set; } = new UnityEvent<IBasicInteractable2D, BasicControllable2D>();
        public UnityEvent<IBasicInteractable2D, BasicControllable2D> Deselected { get; set; } = new UnityEvent<IBasicInteractable2D, BasicControllable2D>();
        public UnityEvent<IBasicInteractable2D, BasicControllable2D> Clicked { get; set; } = new UnityEvent<IBasicInteractable2D, BasicControllable2D>();

        // Provides the grid index for this tile.
        public GridIndex2D GetGridIndex(BasicControllable2D controllable)
        {
            // Coords can be accessed directly here
            return new GridIndex2D((int)Coords.x, (int)Coords.y);
        }

        // Delegates the click action directly to the Tile.
        public void OnClick(BasicControllable2D controllable)
        {
            OnClicked();
        }

        // Delegates the click action directly to the Tile.
        public void OnDeselected(BasicControllable2D controllable)
        {
            OnDehighlighted();
        }

        // Delegates the click action directly to the Tile.
        public void OnSelected(BasicControllable2D controllable)
        {
            OnHighlighted();
        }
    }
```
The rest of the scene setup remains the same as in Sample 1.

### Sample 3
The third sample extends on the previous by introducing unit handling. Here units are not treated as separate Interactables, rather a element of the Tile. With minor modification, the `TileInteractable` component easily accommodates this:
```c#
public class TileInteractable : MonoBehaviour, IBasicInteractable2D
{
    /// ... The rest of the code remains unchanged
    
    public void OnClick(BasicControllable2D controllable)
    {
        if(_tileReference.CurrentUnit != null) 
        {
            _tileReference.CurrentUnit.OnClicked();
        }
        else
        {
            _tileReference.OnClicked();
        }
    }
}
```
Notice the addition of a conditional check within the OnClick method.  If a unit is present on the tile, the unit's OnClicked method is called; otherwise, the default tile clicking  behavior remains. The rest of the scene setup does not change.

### Sample 4
The final sample demonstrates a common gameplay pattern in gamepad-controlled strategy games: cycling through units.  Instead of selecting tiles to interact with a unit, a button press can cycle through available units back and forth. We'll achieve this by creating a new grid controller, while the setup for tile navigation and clicking interactions remains the same.

To make units interactable, we need to choose an appropriate interface. Units, like tiles, support selection and clicking. However, they aren't inherently ordered like tiles within a grid. We can conceptually view units as a sequence,  making a 1D grid a good fit. The IBasicInteractable1D is ideal for this scenario.

`UnitInteractable` components is shown below

```c#
public class UnitInteractable : MonoBehaviour, IBasicInteractable1D
{
    [SerializeField] private Unit _unitReference;

    public UnityEvent<IBasicInteractable1D, BasicControllable1D> Selected { get; set; } = new UnityEvent<IBasicInteractable1D, BasicControllable1D>();
    public UnityEvent<IBasicInteractable1D, BasicControllable1D> Deselected { get; set; } = new UnityEvent<IBasicInteractable1D, BasicControllable1D>();
    public UnityEvent<IBasicInteractable1D, BasicControllable1D> Clicked { get; set; } = new UnityEvent<IBasicInteractable1D, BasicControllable1D>();

    public GridIndex1D GetGridIndex(BasicControllable1D controllable)
    {
        // Referencing another component to determine the Unit order
        return new GridIndex1D(GetComponent<UnitOrder>().Index);
    }

    public void OnClick(BasicControllable1D controllable)
    {
        _unitReference.OnMouseDown();
    }

    public void OnSelected(BasicControllable1D controllable)
    {
        _unitReference.OnMouseDown();
    }
    public void OnDeselected(BasicControllable1D controllable)
    {
    }
}
```
```c#
public class UnitOrder : MonoBehaviour
{
    public int Index;
}
```
Note that `OnClick` and `OnSelected` are equivalent and trigger the corresponding Unit method. `OnDeselected`  is empty  as there's no deselection behavior in this  example. The `GetGridIndex` method uses a separate `UnitOrder` component to provide an order value, assigned manually in the editor.

The choice of `IBasicInteractable1D` dictates the subsequent selections: the `GridControllable1D` and `BasicGridController1D` components to manage a 1D sequence. To provide a distinct control scheme for navigating tiles and cycling through units, let's create a new InputProvider using the new Unity Input System. This avoids keybind clashes with the existing InputProvider2D used for tile navigation::

```c#
public class SoliderInputProvider : InputProvider<GridIndex1D, BasicControllable1D, IBasicInteractable1D>
{
    public void Awake()
    {
        CycleInputActions inputActions = new CycleInputActions();
        inputActions.Grid.Enable();

        // Use Left / Right gamepad trigger or "Q" / "E" keys on the keyboard
        inputActions.Grid.Cycle.performed += (context) =>
        {
            var inputValue = Mathf.RoundToInt(context.ReadValue<float>());
            OnCommand.Invoke(new MoveSelectionCommand1D(new GridIndex1D(inputValue)));
        };
    }
}
```
With this setup, we retain intuitive tile navigation while streamlining unit selection, creating a smoother player experience.

## Contact and Support
If you have any questions, feedback, or need assistance with the Flexible Grid Controller, feel free to reach out. You can contact me directly via email at crookedhead@outlook.com for specific queries or suggestions. Additionally, for broader community support and discussions, join the [Flexible Grid Controller Discord server](https://discord.gg/h5x8MAFrCF).
