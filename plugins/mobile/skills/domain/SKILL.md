---
name: domain
description: "Domain layer patterns for repository interfaces, use cases, domain exceptions, and DI wiring in domain and use_case. Use when adding repositories, use cases, or modifying business logic."
---

## Related Guidelines

- `@dart` - Dart language best practices, naming, modern features
- `@clean` - Layer separation, dependency rules
- `@di` - get_it, injectable, micropackage setup
- `@data` - Data source patterns domain depends on

## Package Dependency Rules (enforce strictly)

```
entity     → no dependencies
data       → entity
domain     → entity, data (data source interfaces only)
use_case   → domain, entity
screens    → domain, use_case, entity
```

Reject upward dependencies (e.g. domain importing `screens`).

## Repository Layer (`domain/src/<domain>/`)

### Folder Structure

```
domain/src/<domain>/
├── <domain>.dart                     # Barrel — exports interface + exceptions
├── <domain>_repository.dart          # abstract interface only
└── exception/
    ├── exception.dart                # Barrel — exports all exception classes
    └── <domain>_exception.dart       # domain exceptions
```

> Implementation lives in `data/src/repository/` — see `@data`.

### Barrel Contents

Only export symbols consumed outside the folder. Do not export internal helpers or types only used within the same directory.

```dart
// <domain>.dart — exported because consumers import repository interface and exceptions
export '<domain>_repository.dart';
export 'exception/exception.dart';

// exception/exception.dart — exported because exceptions are caught by callers outside
export '<domain>_exception.dart';
```

### Import

```dart
// Consume from other packages:
import 'package:domain/<domain>/<domain>.dart';
```

### Interface

- `abstract interface class <Domain>Repository`
- Methods return entity types or primitives — never DTOs/models/transport types
- Document every exception: `/// Throws [XxxException] when ...`

```dart
abstract interface class AuthenticationRepository {
  /// - Throws [LoginException] when login fails.
  /// - Throws [IncorrectCredentialException] when credentials are wrong.
  Future<void> login({required String email, required String password});
}
```

## Use Case Layer (`use_case/<domain>/<feature>/`)

### Folder Structure

```
use_case/<domain>/
├── <domain>.dart                  # Barrel — exports all feature barrels
└── <feature>/
    ├── <use_case_name>_uc.dart        # abstract interface
    ├── <use_case_name>_uc_impl.dart   # implementation
    └── <feature>.dart                 # Barrel
```

### Barrel Contents

Only export symbols consumed outside the folder. Do not export impl — callers depend on the interface only.

```dart
// use_case/<domain>/<domain>.dart — re-exports feature barrels for consumers
export '<feature>/<feature>.dart';

// use_case/<domain>/<feature>/<feature>.dart — interface only; impl is internal
export '<use_case_name>_uc.dart';
```

### Import

```dart
import 'package:app/use_case/<domain>/<domain>.dart';
```

### Interface

- `abstract interface class <Action><Domain>UseCase` with single `call()` method
- Invoke as `_useCase()` not `_useCase.execute()`

```dart
abstract interface class DetermineAccountUseCase {
  Future<void> call();
}
```

### Implementation

- `@Injectable(as: <Action><Domain>UseCase)` on `final class <Action><Domain>UseCaseImpl`
- Inject only **repositories** or **other use cases** — never data sources, APIs, local sources
- One use case = one business operation
- Validation and branching logic lives here; raw data access in repositories
- Propagate or wrap domain exceptions from repositories

```dart
@Injectable(as: DetermineAccountUseCase)
final class DetermineAccountUseCaseImpl implements DetermineAccountUseCase {
  const DetermineAccountUseCaseImpl({
    required UserRepository userRepository,
    required SynchronizeInformationUseCase synchronizeInformationUseCase,
  })  : _userRepository = userRepository,
        _synchronizeInformationUseCase = synchronizeInformationUseCase;

  final UserRepository _userRepository;
  final SynchronizeInformationUseCase _synchronizeInformationUseCase;

  @override
  Future<void> call() async {
    final user = await _userRepository.getInformation();
    if (user.children.isNotEmpty) {
      await _userRepository.setMultipleUsersFlag(value: true);
      await _userRepository.removeCachedUser();
      return;
    }
    await _synchronizeInformationUseCase();
    await _userRepository.setMultipleUsersFlag(value: false);
  }
}
```

## Domain Exceptions (`domain/src/<domain>/exception/`)

- One exception class per failure mode
- Extend base `Exception` or shared base class
- `String message` field — user-safe, no stack traces/tokens/raw payloads
- Export via exception barrel and repository's public barrel

## Dependency Injection

- `@Injectable(as: Interface)` on every implementation — build_runner generates registrations
- External bindings → `<domain>_module.dart` with `@module abstract class`
- Export repository **interfaces** (not impls) from domain package barrel
- Export use case **interfaces** from `use_case/<domain>/<feature>/<feature>.dart`
- Run `fvm dart run build_runner build --delete-conflicting-outputs` (or `dart run ...` if `.fvmrc` is absent or `fvm` is unavailable) after annotation changes

## NEVER

- Import `data` implementations, Retrofit types, or `DioException` in domain/use case code
- Put UI logic, `BuildContext`, or Flutter widgets in domain layer
- Call data sources or APIs directly from use cases — always through a repository
- Expose internal models or DTOs — return entity types or primitives
- Leak `DataSourceException` above the domain boundary
- Swallow exceptions silently — translate or rethrow with context
