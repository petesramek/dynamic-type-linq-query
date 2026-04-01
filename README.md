# Dynamic Type LINQ Query

Sample project demonstrating how to use `IDynamicMetaObjectProvider` and `DynamicMetaObject` to decorate a concrete C# type with **dynamic dispatch behavior**, enabling both property-name and indexer-style access inside LINQ expressions.

Created as an answer to the Stack Overflow question:
[How to force "dynamic" behavior in a LINQ expression with DynamicObject?](https://stackoverflow.com/questions/57336485/how-to-force-dynamic-behavior-in-a-linq-expression-with-dynamicobject)

---

## Problem

When you assign objects to an `IEnumerable<dynamic>` and filter with `.Where(m => m.SomeProperty ...)`, the C# compiler emits a **dynamic call-site** that is resolved at runtime via the DLR (Dynamic Language Runtime). By default, a concrete class has no idea how to handle those call-sites, so accessing a computed or virtual property (e.g., `Age` derived from `BirthDate`, or a composed `FullName`) simply fails.

The challenge is to intercept these dynamic member and indexer accesses and map them to real method calls or expression trees — without changing the LINQ query itself.

---

## Solution Overview

`Person` implements `IDynamicMetaObjectProvider`, which lets the DLR ask the object how to handle every dynamic operation. The implementation returns a custom `PersonMetaObject` (a `DynamicMetaObject` subclass) that translates each requested member name into a concrete `Expression` tree:

| Dynamic access | What `PersonMetaObject` returns |
|---|---|
| `m.Age` / `m["Age"]` | `Expression.Call` invoking `Person.GetAge()` |
| `m.FullName` / `m["FullName"]` | `String.Join(" ", FirstName, LastName)` expression |
| Any real property name | `Expression.Property` accessor |
| Unknown name | `Expression.Default(typeof(object))` — returns `null` |

A separate `ExpressionFactory` builds **composable lambda expressions** for `IQueryable<dynamic>`, using `Expression.Dynamic` with a `CSharpBinder` so the same dynamic dispatch works when passed to `.Where(...)` on a queryable.

---

## Key Classes

### `Person` – `IDynamicMetaObjectProvider`

A regular POCO that opts in to dynamic dispatch by implementing `IDynamicMetaObjectProvider.GetMetaObject`:

```csharp
public sealed class Person : IDynamicMetaObjectProvider {
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public DateTime BirthDate { get; set; }

    public int GetAge() => DateTime.Today.Year - BirthDate.Year;

    DynamicMetaObject IDynamicMetaObjectProvider.GetMetaObject(Expression parameter)
        => new PersonMetaObject(parameter, this);
}
```

### `PersonMetaObject` – `DynamicMetaObject`

Overrides both `BindGetMember` (property-style: `m.Age`) and `BindGetIndex` (indexer-style: `m["Age"]`) and routes them to the same internal resolver:

```csharp
// Property-style: m.Age
public override DynamicMetaObject BindGetMember(GetMemberBinder binder)
    => GetDynamicMetaObject(GetSelfExpression(), binder.Name);

// Indexer-style: m["Age"]
public override DynamicMetaObject BindGetIndex(GetIndexBinder binder, DynamicMetaObject[] indexes)
    => GetDynamicMetaObject(GetSelfExpression(), indexes.First().Value.ToString());
```

The resolver maps names to expression trees:
- **`Age`** → calls `Person.GetAge()` via reflection + `Expression.Call`
- **`FullName`** → `String.Join(" ", FirstName, LastName)` via `Expression.Call`
- **Known property** → `Expression.Property` accessor
- **Unknown** → `Expression.Default(typeof(object))`

### `ExpressionFactory` – composable predicates for `IQueryable<dynamic>`

Builds `Expression<Func<dynamic, bool>>` lambdas using `Expression.Dynamic` with a `CSharpBinder`, so the dynamic dispatch is baked into the expression tree itself and can be passed to `.Where(...)` on an `IQueryable<dynamic>`:

```csharp
// Predicate using property access binder: m.Age > 59
ExpressionFactory.PropertyGreaterThanPredicate("Age", 59)

// Predicate using index access binder: m["Age"] < 60
ExpressionFactory.IndexLessThanPredicate("Age", 60)
```

---

## Usage Examples (`Program.cs`)

```csharp
IEnumerable<dynamic> enumerable = [
    new Person { BirthDate = new DateTime(1975, 8, 14), FirstName = "Alan",    LastName = "Smith"   },
    new Person { BirthDate = new DateTime(2006, 1, 26), FirstName = "Elisa",   LastName = "Ridley"  },
    new Person { BirthDate = new DateTime(1993, 12, 1), FirstName = "Randy",   LastName = "Knowles" },
    new Person { BirthDate = new DateTime(1946, 5, 8),  FirstName = "Melissa", LastName = "Fincher" }
];

// 1. Property-style access on IEnumerable<dynamic>
var youngsters = enumerable.Where(m => m.Age < 21);

// 2. Indexer-style access on IEnumerable<dynamic>
var adults = enumerable.Where(m => m["Age"] >= 21 && m["Age"] <= 59);

// 3. Composed lambda for IQueryable<dynamic> – property binder
var seniors = enumerable.AsQueryable()
    .Where(ExpressionFactory.PropertyGreaterThanPredicate("Age", 59));

// 4. Composed lambda for IQueryable<dynamic> – index binder
var under60 = enumerable.AsQueryable()
    .Where(ExpressionFactory.IndexLessThanPredicate("Age", 60));
```

> **Note:** The `ExpressionFactory` predicates work with `IQueryable<dynamic>` backed by in-memory collections. They are **not** compatible with database-backed LINQ providers (e.g., Entity Framework), as those providers cannot translate dynamic call-sites into SQL.

---

## Project Structure

```
.
└── src/
    ├── Data/
    │   ├── Meta/
    │   │   └── PersonMetaObject.cs     # DynamicMetaObject – resolves dynamic member/index access to expressions
    │   └── Person.cs                   # Concrete type implementing IDynamicMetaObjectProvider
    ├── Expressions/
    │   └── ExpressionFactory.cs        # Builds composable Func<dynamic,bool> lambdas using Expression.Dynamic
    └── Program.cs                      # Entry point – demonstrates all four access patterns
```

---

## Key Concepts

- **`IDynamicMetaObjectProvider`** – interface that a type implements to take full control of how the DLR handles dynamic operations against it.
- **`DynamicMetaObject`** – returned by `GetMetaObject`; override `BindGetMember` / `BindGetIndex` to return `Expression` trees for each operation.
- **`Expression.Dynamic`** – wraps a `CallSiteBinder` in an expression tree node, allowing dynamic dispatch to be embedded in a composable lambda.
- **`CSharpBinder.GetMember` / `GetIndex`** – C# runtime binders that perform the same member/index lookup the C# compiler would generate for a `dynamic` call-site.
- **`BindingRestrictions.GetTypeRestriction`** – caches the meta-object per type so the DLR does not re-bind on every call.

---

## Additional Resources

- [Stack Overflow question](https://stackoverflow.com/questions/57336485/how-to-force-dynamic-behavior-in-a-linq-expression-with-dynamicobject)
- [IDynamicMetaObjectProvider Interface](https://learn.microsoft.com/en-us/dotnet/api/system.dynamic.idynamicmetaobjectprovider)
- [DynamicMetaObject Class](https://learn.microsoft.com/en-us/dotnet/api/system.dynamic.dynamicmetaobject)

---

## Prerequisites

- [.NET SDK](https://dotnet.microsoft.com/download) (project targets .NET — see `DynamicTypeLinqQuery.csproj`)
- No external NuGet packages required

## Running the Project

```bash
cd src
dotnet run
```