---
name: clean
description: "Clean Architecture layer separation, dependency rules, repositories, data sources, and use cases for Flutter apps. Use when implementing features, reviewing layer boundaries, adding repositories, or understanding data flow between layers."
---

## Related Guidelines

- `@dart` - Dart language features used to enforce layer boundaries
- `@presentation` - Presentation layer structure and screen patterns
- `@domain` - Repository interfaces and use case patterns
- `@data` - Data source and repository implementation patterns
- `@di` - Dependency injection wiring across layers
- `@state` - State management within the presentation layer

# Clean Architecture Guidelines

## Architecture Overview

This project follows **Clean Architecture** with a modular monorepo. Layers have clear responsibilities and unidirectional dependency flow.

```
┌─────────────────────────────────────────────────────────────┐
│                    Presentation Layer                       │
│  screens/, widgets/, components/                            │
└─────────────────────────────────────────────────────────────┘
                            ↓ depends on
┌─────────────────────────────────────────────────────────────┐
│                      Domain Layer                           │
│  use_case/, entity/                                         │
└─────────────────────────────────────────────────────────────┘
                            ↓ depends on
┌─────────────────────────────────────────────────────────────┐
│                       Data Layer                            │
│  domain/, data/                                             │
└─────────────────────────────────────────────────────────────┘
```

**Dependency rule**: always inward. Data → Domain ← Presentation.

## Reference

| Topic | File |
|-------|------|
| Layer responsibilities (full detail + examples) | [reference/layers.md](reference/layers.md) |

## Naming Conventions

### Files
- Screens: `[feature]_screen.dart`
- Cubits: `[feature]_cubit.dart` / `[feature]_state.dart`
- Use Cases: `[action]_uc.dart` / `[action]_uc_impl.dart`
- Repositories: `[domain]_repository.dart` / `[domain]_repository_impl.dart`
- Remote data sources: `[domain]_remote_datasource.dart` / `[domain]_remote_datasource_impl.dart`
- Local data sources: `[domain]_local_datasource.dart` / `[domain]_local_datasource_impl.dart`
- Routes: `[feature]_route.dart`

### Classes
- `[Feature]Screen`, `[Feature]Cubit`, `[Feature]State`
- `[Action][Domain]UseCase` / `[Action][Domain]UseCaseImpl`
- `[Domain]Repository` / `[Domain]RepositoryImpl`
- `[Domain]RemoteDataSource` / `[Domain]RemoteDataSourceImpl`
- `[Domain]LocalDataSource` / `[Domain]LocalDataSourceImpl`

## Data Flow Example

```
User taps login → LoginScreen (Widget)
  ↓ calls
LoginCubit.login()
  ↓ calls
AuthenticationRepository.login()
  ↓ calls
AuthenticationRemoteDataSourceImpl → _api.login()  (Retrofit)
AuthenticationLocalDataSourceImpl  → save tokens

Response flows back: API → DataSource → Repository → Cubit → UI
```

## Package Dependencies

- `entity` — no dependencies
- `data` — depends on `entity`
- `domain` — depends on `entity`, `data`
- `use_case` — depends on `domain`
- `screens` — depends on `domain`, `use_case`, `entity`

## Error Handling

- Data sources throw `DataSourceException`
- Repositories translate to domain-specific exceptions
- Use cases propagate or wrap repository exceptions
- Cubits catch all, emit failure state

## Creating New Feature (Checklist)

1. Define entity in `entity/`
2. Create remote data source in `data/src/remote/datasource/`
3. Create local data source in `data/src/local/datasource/` (if persistence needed)
4. Create repository in `domain/src/<domain>/`
5. Create use case in `use_case/` (if orchestration needed)
6. Create screen + cubit in `screens/`
7. Register dependencies in package DI modules

## Package Structure

```
├── entity/       # Domain entities (no deps)
├── data/         # Remote + local data sources
├── domain/       # Repository interfaces & impls
├── ui_kit/       # Reusable UI components
└── shared_lint/  # Shared lint rules
```

## Key Files

- `core/di/di.dart` — DI configuration
- `app.dart` — App initialization
- `core/state_management/` — State management utilities (`DataLoadStatus`)
- `data/src/remote/network/` — Network config
- `entity/` — All domain models
@
## Tools & Commands

> Use `fvm` prefix only if `.fvmrc` exists in the project root AND `fvm` command is available; otherwise call `dart`/`flutter` directly.

- **Code generation**: `fvm dart run build_runner build -d` (or `dart run build_runner build -d`)
- **Tests**: `fvm flutter test --coverage --branch-coverage --fail-fast -r failures-only --test-randomize-ordering-seed random`
- **Translations**: run the project's translation generation script directly
- **Run app**: `fvm flutter run --flavor development`

## References

- Clean Architecture: https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html
- BLoC Library: https://bloclibrary.dev
