# Architecture Documentation: R3, VContainer & MVP

> This document records the architectural decisions and learning notes for this Unity project.

---

## Overview

This project adopts the following libraries and patterns:

| Component | Choice | Purpose |
|-----------|--------|---------|
| DI Framework | **VContainer** | Dependency Injection, object lifecycle management |
| Reactive Library | **R3** | Reactive programming, event stream handling |
| Architecture Pattern | **MVP** | Separation of concerns |

---

## Why These Choices?

### VContainer over Zenject

- **5-10x faster** resolution speed
- **Minimal GC allocation** during resolution
- **Smaller codebase** with fewer internal types
- **Immutable containers** for thread safety
- Clean and transparent API

### R3 over UniRx

- R3 is the **official successor** to UniRx by Cysharp/neuecc
- **Cross-platform support** (not Unity-only)
- **Modern API** with better performance
- Better integration with async/await

### MVP over MVC

- **Complete decoupling** between View and Model
- **Better testability** (Views can be mocked)
- **Perfect fit** with R3's ReactiveProperty
- Clear data flow and responsibilities

---

## MVP Pattern Explained

```
┌─────────────────────────────────────────────────────┐
│                                                     │
│    ┌─────────┐                ┌─────────┐          │
│    │  Model  │                │  View   │          │
│    │  (Data) │                │  (UI)   │          │
│    └─────────┘                └─────────┘          │
│         ▲                         ▲                │
│         │                         │                │
│         │  Read/Write        Update UI             │
│         │                         │                │
│         ▼                         ▼                │
│    ┌─────────────────────────────────────┐         │
│    │            Presenter                │         │
│    │     (Mediator / Logic Handler)      │         │
│    └─────────────────────────────────────┘         │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### Responsibilities

| Layer | Role | Unity Example |
|-------|------|---------------|
| **Model** | Pure data, no UI awareness | `ReactiveProperty<int> Health` |
| **View** | Pure display, no data logic | `Slider`, `Text`, MonoBehaviours |
| **Presenter** | Connects M↔V, handles logic | Subscribes to Model, updates View |

---

## Recommended Project Structure

```
Assets/
├── Scripts/
│   ├── Models/           ← Pure data classes
│   │   ├── PlayerModel.cs
│   │   ├── InventoryModel.cs
│   │   └── GameStateModel.cs
│   │
│   ├── Views/            ← MonoBehaviours, UI only
│   │   ├── PlayerHealthView.cs
│   │   ├── InventoryView.cs
│   │   └── GameOverView.cs
│   │
│   ├── Presenters/       ← Logic connecting M and V
│   │   ├── PlayerPresenter.cs
│   │   ├── InventoryPresenter.cs
│   │   └── GamePresenter.cs
│   │
│   └── LifetimeScopes/   ← VContainer configuration
│       ├── RootLifetimeScope.cs
│       └── GameSceneLifetimeScope.cs
│
└── Docs/                 ← Documentation
    └── Architecture-R3-VContainer-MVP.md
```

---

## Code Examples

### Model (Pure Data)

```csharp
using R3;

public class PlayerModel
{
    public ReactiveProperty<int> Health { get; } = new(100);
    public ReactiveProperty<int> MaxHealth { get; } = new(100);
    public ReactiveProperty<bool> IsAlive { get; } = new(true);
    
    public void TakeDamage(int damage)
    {
        Health.Value = Math.Max(0, Health.Value - damage);
        if (Health.Value <= 0)
            IsAlive.Value = false;
    }
}
```

### View (Pure Display)

```csharp
using UnityEngine;
using UnityEngine.UI;

public class PlayerHealthView : MonoBehaviour
{
    [SerializeField] private Text healthText;
    [SerializeField] private Slider healthSlider;
    
    public void UpdateHealth(int current, int max)
    {
        healthText.text = $"{current}/{max}";
        healthSlider.value = (float)current / max;
    }
}
```

### Presenter (Connector)

```csharp
using R3;
using VContainer;
using VContainer.Unity;

public class PlayerHealthPresenter : IStartable, IDisposable
{
    private readonly PlayerModel _model;
    private readonly PlayerHealthView _view;
    private readonly CompositeDisposable _disposables = new();
    
    public PlayerHealthPresenter(PlayerModel model, PlayerHealthView view)
    {
        _model = model;
        _view = view;
    }
    
    public void Start()
    {
        _model.Health
            .CombineLatest(_model.MaxHealth, (current, max) => (current, max))
            .Subscribe(t => _view.UpdateHealth(t.current, t.max))
            .AddTo(_disposables);
    }
    
    public void Dispose() => _disposables.Dispose();
}
```

### LifetimeScope (DI Configuration)

```csharp
using VContainer;
using VContainer.Unity;

public class GameLifetimeScope : LifetimeScope
{
    [SerializeField] private PlayerHealthView playerHealthView;
    
    protected override void Configure(IContainerBuilder builder)
    {
        builder.Register<PlayerModel>(Lifetime.Singleton);
        builder.RegisterComponent(playerHealthView);
        builder.RegisterEntryPoint<PlayerHealthPresenter>();
    }
}
```

---

## Key Concepts Quick Reference

| Concept | One-liner |
|---------|-----------|
| **ReactiveProperty** | A value that notifies subscribers when changed |
| **Observable** | A stream of events over time |
| **Subscribe** | Listen to changes and react |
| **AddTo(this)** | Auto-dispose when GameObject is destroyed |
| **LifetimeScope** | VContainer's composition root |
| **IStartable** | VContainer's entry point interface |

---

## Resources

- [VContainer Documentation](https://vcontainer.hadashikick.jp/)
- [R3 GitHub Repository](https://github.com/Cysharp/R3)
- [Cysharp (neuecc)](https://github.com/Cysharp)

---

*Document created: 2024*  
*Architecture decision: MVP with R3 + VContainer*

