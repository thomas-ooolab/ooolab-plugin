# Layer Responsibilities

## Contents
- [Presentation Layer](#1-presentation-layer-lib)
- [Domain Layer — Use Cases](#2-domain-layer)
- [Domain Layer — Entities](#entities-packagesentity)
- [Data Layer — Repositories](#3-data-layer-packages)
- [Data Layer — Data Package](#data-package-packagesdata)

---

## 1. Presentation Layer (`lib/`)

**Location**: `lib/screens/`, `lib/widgets/`, `lib/components/`

**Responsibilities**:
- Display data to users
- Handle user interactions
- Manage UI state using BLoC/Cubit
- Navigate between screens
- NO business logic (delegate to use cases)
- NO direct repository calls (use through cubits and use cases)

**Structure**:
```
lib/screens/[feature_name]/
├── cubit/
│   ├── [feature]_cubit.dart      # State management logic
│   └── [feature]_state.dart      # State definitions
├── models/                        # View models (presentation layer only)
│   └── [model_name].dart
├── route/
│   └── [feature]_route.dart      # Navigation routes
├── views/
│   └── [feature]_screen.dart     # UI implementation
└── [feature].dart                # Barrel file
```

**Guidelines**:
- Screens should be stateless widgets when possible
- Cubits should extend `Cubit<State>` and use `CubitMixin` for safe emissions
- Always inject dependencies through constructor (repositories, use cases)
- Use `BlocProvider` to instantiate cubits at screen level
- Resolve dependencies via `sl<T>()` inside `BlocProvider.create`
- Cubits should call use cases or repositories, never services directly
- Handle UI state transitions using `FormStatus` and `DataLoadStatus`

**Example Cubit**:
```dart
@injectable
class LoginCubit extends Cubit<LoginState> with CubitMixin<LoginState> {
  LoginCubit({
    required AuthenticationRepository authenticationRepository,
    required UserRepository userRepository,
    required DetermineAccountUseCase determineAccountUseCase,
  }) : _authenticationRepository = authenticationRepository,
       _userRepository = userRepository,
       _determineAccountUseCase = determineAccountUseCase,
       super(LoginState());

  final AuthenticationRepository _authenticationRepository;
  final UserRepository _userRepository;
  final DetermineAccountUseCase _determineAccountUseCase;

  Future<void> login() async {
    safeEmit(state.copyWith(status: FormStatus.submitting));
    try {
      await _authenticationRepository.login(
        email: state.email,
        password: state.password,
      );
      await _determineAccountUseCase();
      safeEmit(state.copyWith(status: FormStatus.success));
    } catch (e) {
      safeEmit(state.copyWith(
        status: FormStatus.failure,
        errorMessage: e.toString(),
      ));
    }
  }
}
```

---

## 2. Domain Layer

### Use Cases (`lib/use_case/`)

**Responsibilities**:
- Contain business logic and orchestration
- Coordinate between multiple repositories
- Single responsibility (one use case = one business operation)
- Platform-independent business rules
- NO UI dependencies
- NO direct service calls

**Structure**:
```
lib/use_case/[domain]/[feature]/
├── [use_case_name]_uc.dart       # Abstract interface
├── [use_case_name]_uc_impl.dart  # Implementation
└── [feature].dart                 # Barrel file
```

**Guidelines**:
- Define abstract interface with `call()` method
- Use `@Injectable(as: Interface)` for dependency injection
- Inject only repositories (never services or APIs)
- Keep use cases focused and composable
- Handle complex business logic and validations
- Coordinate multiple repository calls when needed

**Example Use Case**:
```dart
/// Abstract interface
abstract interface class DetermineAccountUseCase {
  Future<void> call();
}

/// Implementation
@Injectable(as: DetermineAccountUseCase)
final class DetermineAccountUseCaseImpl implements DetermineAccountUseCase {
  const DetermineAccountUseCaseImpl({
    required UserRepository userRepository,
    required SynchronizeInformationUseCase synchronizeInformationUseCase,
  }) : _userRepository = userRepository,
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

### Entities (`packages/entity/`)

**Responsibilities**:
- Define domain models
- Pure Dart classes with business logic
- Immutable data structures
- NO dependencies on other layers
- Use with json_serializable for serialization

**Structure**:
```
packages/entity/lib/src/[domain]/
├── entity.dart                    # Barrel file
├── [entity_name].dart            # Domain entity
├── [enum_name].dart              # Domain enums
└── [type_name].dart              # Domain types
```

**Guidelines**:
- Keep entities immutable (use `@freezed` or `copyWith`)
- Define domain-specific types and enums
- Include entity-level business logic only
- Use `@JsonSerializable()` for JSON conversion
- Export through `entity.dart` package

---

## 3. Data Layer (`packages/`)

### Repositories (`packages/domain/`)

**Responsibilities**:
- Abstract data access interface
- Coordinate between remote and local data sources
- Convert service exceptions to repository exceptions
- Handle data caching strategies
- Map service models to domain entities
- NO business logic

**Structure**:
```
packages/domain/lib/src/[domain]/
├── [domain]_repository.dart          # Abstract interface
├── [domain]_repository_impl.dart     # Implementation
├── [domain]_module.dart              # DI module
└── exception/
    └── exception.dart                 # Repository exceptions
```

**Guidelines**:
- Define abstract interface using `abstract interface class`
- Implementation uses `@Injectable(as: Interface)`
- Inject data source interfaces (remote and local) from `packages/data`
- Convert `DataSourceException` to domain-specific exceptions
- Document exceptions in interface method comments
- Keep repository focused on data access only

**Example Repository**:
```dart
/// Abstract interface
abstract interface class AuthenticationRepository {
  /// Login
  ///
  /// - Throws [LoginException] when logging in fails.
  /// - Throws [IncorrectCredentialException] when logging with incorrect credential.
  Future<void> login({required String email, required String password});
}

/// Implementation
@Injectable(as: AuthenticationRepository)
final class AuthenticationRepositoryImpl implements AuthenticationRepository {
  AuthenticationRepositoryImpl({
    required AuthenticationRemoteDataSource authenticationRemoteDataSource,
    required AuthenticationLocalDataSource authenticationLocalDataSource,
  }) : _authenticationRemoteDataSource = authenticationRemoteDataSource,
       _authenticationLocalDataSource = authenticationLocalDataSource;

  final AuthenticationRemoteDataSource _authenticationRemoteDataSource;
  final AuthenticationLocalDataSource _authenticationLocalDataSource;

  @override
  Future<void> login({required String email, required String password}) async {
    try {
      final tokens = await _authenticationRemoteDataSource.login(
        email: email,
        password: password,
      );
      await _authenticationLocalDataSource.save(
        accessToken: tokens.accessToken,
        refreshToken: tokens.refreshToken,
        tokenType: tokens.tokenType,
      );
    } on DataSourceException catch (e) {
      if (e.error?.name == ErrorName.unauthorized) {
        throw IncorrectCredentialException('Incorrect credential error!');
      }
      throw LoginException(e.toString());
    }
  }
}
```

### Data Package (`packages/data/`)

Unified data source package containing both remote (API) and local (persistence) sources.

**Responsibilities**:
- Handle remote API calls (remote source)
- Handle local data persistence (local source)
- Handle data transformations between API and domain
- Convert HTTP errors to DataSourceException
- Manage shared preferences, secure storage, Hive
- NO business logic
- NO direct entity exposure (use models)

**Structure**:
```
packages/data/lib/
├── data.dart                               # Barrel file (exports DataModule, local, remote)
└── src/
    ├── data_module.dart                    # Top-level DI module
    ├── remote/                             # Remote data sources
    │   ├── remote.dart                     # Remote barrel file
    │   ├── api/                            # Retrofit API definitions
    │   │   └── [domain]/
    │   │       └── [domain]_api.dart
    │   ├── datasource/                     # Remote data source layer
    │   │   └── [domain]/
    │   │       ├── [domain]_remote_datasource.dart       # Abstract interface
    │   │       ├── [domain]_remote_datasource_impl.dart  # Implementation
    │   │       ├── model/                                 # Request/Response models
    │   │       │   └── [model_name].dart
    │   │       └── datasource.dart                        # Barrel file
    │   └── network/                        # Network configuration
    └── local/                              # Local data sources
        ├── local.dart                      # Local barrel file
        ├── datasource/                     # Local data source layer
        │   ├── authentication/             # Token persistence
        │   ├── user/                       # User preferences/cache
        │   └── [feature]/                  # Feature-specific cache
        └── model/                          # Local storage models
```

**Remote Source Guidelines**:
- Define remote data source interface with `abstract interface class`
- Implementation injects only API classes
- All methods throw `DataSourceException` on failure
- Convert API models to entities
- Use Retrofit for API definitions
- Handle HTTP errors and convert to DataSourceException

**Example Remote Data Source**:
```dart
/// Abstract interface
abstract interface class AuthenticationRemoteDataSource {
  /// Login
  ///
  /// Throws [DataSourceException] if the api fails for some reason.
  Future<Tokens> login({required String email, required String password});
}

/// Implementation
final class AuthenticationRemoteDataSourceImpl implements AuthenticationRemoteDataSource {
  AuthenticationRemoteDataSourceImpl(this._api);

  final AuthenticationApi _api;

  @override
  Future<Tokens> login({
    required String email,
    required String password,
  }) async {
    try {
      final response = await _api.login(
        LoginRequestModel(email: email, password: password).toJson(),
      );
      return TokensModel.fromJson(response.data as Map<String, dynamic>);
    } catch (e) {
      throw DataSourceException.from(e);
    }
  }
}
```

**Local Data Source Guidelines**:
- Define local data source interface
- Implement using Hive, SharedPreferences, or SecureStorage
- Keep models separate from domain entities
- Handle serialization/deserialization
- Throw exceptions on storage failures

**Example Local Data Source**:
```dart
abstract interface class AuthenticationLocalDataSource {
  /// Save the tokens to local storage.
  ///
  /// Throws [Exception] if failure happened for any reason.
  Future<void> save({
    required String tokenType,
    required String accessToken,
    required String refreshToken,
  });

  /// Get the tokens from local storage.
  ///
  /// Return [Null] if failure happened for any reason.
  Future<Tokens?> get();

  /// Delete the tokens which were saved in local storage.
  ///
  /// Throws [Exception] if failure happened for any reason.
  Future<void> delete();
}
```
