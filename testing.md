# Testing Guidelines for Vietnam Coral Reef Monitoring System

## Core Testing Principles

### 1. Test-Driven Development (TDD)
- Write tests before implementing features
- Follow the Red-Green-Refactor cycle
- Use tests to drive design decisions

### 2. Test Coverage Goals
- **Services**: 90%+ code coverage
- **Controllers**: 85%+ code coverage
- **Repositories**: 80%+ code coverage
- **Common libraries**: 95%+ code coverage

### 3. Test Isolation
- Each test must be independent
- Tests should not depend on external services
- Mock all dependencies outside the unit being tested

## Test Organization

### File Structure
- Test files should be located alongside the files they test
- Use the `.spec.ts` extension for all test files
- Group related tests using nested `describe` blocks
- Follow a consistent naming pattern for test files

### Test Hierarchy
```
// Use the relative file path as the root describe name
describe('src/app/component-name/component-name.service.ts', () => {
  // Setup and teardown

  it('should be defined', () => {
    // Basic existence test
  });

  describe('methodName', () => {
    // Tests for a specific method

    it('should handle successful case', () => {
      // Test for success path
    });

    it('should handle error case', () => {
      // Test for error path
    });
  });
});
```

The root describe should always use the relative file path of the file being tested. This makes it easier to:
- Identify which file a test failure is coming from in test reports
- Find the test file for a specific implementation file
- Ensure unique describe block names across the project

## Writing Effective Tests

### 1. Test Suite Optimization
- **Initialize test objects only once per test suite** for performance
- Use `beforeAll` for one-time setup of test objects and spies
- Use `beforeEach` only for state that needs to be reset between tests
- Reset mocks between tests with `jest.clearAllMocks()`
- Initialize the NestJS testing module in `beforeAll` when possible

```typescript
// One-time initialization of objects and module
let service: YourService;
let repository: YourRepository;
let dependencyService: DependencyService;
let module: TestingModule;

// Create objects once per test suite
const repository = new YourRepository();
const dependencyService = new DependencyService();

// Set up spies once - ALWAYS store in variables for later use
const findSpy = jest.spyOn(repository, 'find');
const findOneSpy = jest.spyOn(repository, 'findOne');
const saveSpy = jest.spyOn(repository, 'save');
const performActionSpy = jest.spyOn(dependencyService, 'performAction');

beforeAll(async () => {
  // Initialize the module once
  module = await Test.createTestingModule({
    providers: [
      YourService,
      {
        provide: YourRepository,
        useValue: repository,
      },
      {
        provide: DependencyService,
        useValue: dependencyService,
      },
    ],
  }).compile();

  service = module.get<YourService>(YourService);
  repository = module.get<YourRepository>(YourRepository);
  dependencyService = module.get<DependencyService>(DependencyService);
});

// Reset mock implementations between tests
beforeEach(() => {
  jest.clearAllMocks();
  // Set up default mock implementations if needed
  findSpy.mockResolvedValue([]);
  findOneSpy.mockResolvedValue(null);
});

afterAll(async () => {
  // Clean up resources if needed
  await module.close();
});
```

### 2. Mocking Dependencies
- Create mock implementations for all dependencies
- **Always use `jest.spyOn()` for mocking** instead of jest.fn() when possible
- **Always store jest.spyOn results in variables** for better control and reuse
- Set up mock return values in individual tests, not in the global scope

```typescript
// Mock using jest.spyOn for existing objects - ALWAYS store in variables
const repository = new YourRepository();
const findSpy = jest.spyOn(repository, 'find').mockResolvedValue([mockEntity]);
const findOneSpy = jest.spyOn(repository, 'findOne').mockResolvedValue(mockEntity);
const saveSpy = jest.spyOn(repository, 'save').mockResolvedValue(mockEntity);

// For dependencies that need to be provided via DI
const mockRepositoryObj = {
  find: jest.fn(),
  findOne: jest.fn(),
  save: jest.fn(),
};

// Always prefer jest.spyOn when the object exists - ALWAYS store in variables
const service = new YourService();
const getByIdSpy = jest.spyOn(service, 'getById').mockResolvedValue(mockEntity);
const createSpy = jest.spyOn(service, 'create').mockResolvedValue(mockEntity);

// Set up mock return in a test
const getByIdSpy = jest.spyOn(mockService, 'getById');
getByIdSpy.mockResolvedValue(mockEntity);
```

#### Why Use jest.spyOn Instead of jest.fn()
- `jest.spyOn` creates a mock function similar to `jest.fn()` but tracks calls to the method on the original object
- It's more maintainable as it preserves the original implementation unless explicitly mocked
- Easier to restore the original implementation with `mockRestore()`
- Provides better integration with TypeScript type checking
- Makes refactoring safer as it will catch method signature changes

#### Why Store Spies in Variables
- Allows for easier reference in multiple test cases
- Enables changing the mock implementation between tests
- Makes tests more readable by clearly identifying what's being mocked
- Provides access to all mock functionality like `mockClear()`, `mockReset()`, etc.
- Helps TypeScript identify the correct types for the mock functions

### 3. Arrange-Act-Assert Pattern
- Arrange: Set up the test data and conditions
- Act: Execute the method being tested
- Assert: Verify the results meet expectations

```typescript
it('should create a garden successfully', async () => {
  // Arrange
  const createPayload = { /* test data */ };
  dependencyMethodSpy.mockResolvedValue(expectedResult);
  
  // Act
  const result = await service.create(createPayload);
  
  // Assert
  expect(dependencyMethodSpy).toHaveBeenCalledWith(expectedParams);
  expect(result).toEqual(expectedResult);
});
```

### 4. Test Naming
- Test names should clearly describe what is being tested
- Include the expected behavior in the test name
- Use consistent naming patterns

```typescript
// Good examples:
it('should return null when garden not found', async () => {});
it('should throw RpcException when site not found', async () => {});
it('should update a garden successfully', async () => {});
```

## Testing Different Components

### 1. Services
- Test business logic thoroughly
- Mock repositories and other injected services
- Test both success and error paths
- Verify correct repository method calls

### 2. Controllers
- Test endpoint handling
- Mock service methods that controllers depend on
- Verify proper transformation of service responses to HTTP responses
- Test route parameter and body extraction

### 3. Repositories
- Test custom repository methods
- Use TypeORM's testing utilities
- Mock the database connection

### 4. Exception Filters
- Test exception handling logic
- Verify proper exception transformation

## Testing Microservice Communication

### 1. Client Proxy Mocking
```typescript
import { of } from 'rxjs';

// Create a real ClientProxy or a minimal implementation
const clientProxy = {
  send: () => {},
  // other methods...
};

// Then spy on the methods you need to mock - STORE IN VARIABLE
const sendSpy = jest.spyOn(clientProxy, 'send');
sendSpy.mockImplementation(() => of({ success: true }));

// In your testing module
providers: [
  {
    provide: 'SERVICE_NAME',
    useValue: clientProxy,
  },
]
```

### 2. Event Pattern Testing
- Test event publishing and handling
- Mock the EventEmitter injected into services

## Integration Testing

- Test interactions between components
- Focus on service boundaries
- Use NestJS testing utilities for broader tests

```typescript
// Integration test example
describe('Integration Tests', () => {
  let app: INestApplication;
  
  beforeAll(async () => {
    const moduleFixture = await Test.createTestingModule({
      imports: [AppModule],
    })
    .overrideProvider(DatabaseService)
    .useValue(mockDatabaseService)
    .compile();
    
    app = moduleFixture.createNestApplication();
    await app.init();
  });
  
  afterAll(async () => {
    await app.close();
  });
  
  it('should perform an end-to-end operation', async () => {
    // Test code here
  });
});
```

## Testing Edge Cases

Always include tests for:
- Empty or null inputs
- Boundary values
- Error conditions
- Unexpected input types
- Race conditions (where applicable)
- Authentication/authorization scenarios

## External Libraries

### Libraries Not to Mock
- **Lodash**: Do not mock lodash utility functions. These are pure functions that are fast and reliable.
- **Standard JavaScript utilities**: Don't mock built-in JavaScript functions and utilities.
- **Simple data transformation libraries**: Libraries that simply transform data without side effects don't need mocking.

### Libraries to Mock
- **Database clients**: Always mock database connections and operations.
- **HTTP clients**: Mock axios, fetch, or other HTTP clients.
- **File system operations**: Mock fs operations to avoid touching the real file system.
- **External APIs**: Mock any calls to external services.
- **Time-related functions**: Mock Date, setTimeout, setInterval when testing time-dependent code.

## Entity Mocking Guidelines

When mocking entities, include all properties that are used in the tests:

```typescript
// Create an entity with all required properties and methods
const mockEntity = {
  id: 'entity-id',
  name: 'Entity Name',
  // Include all properties used in tests
  createdAt: new Date(),
  relations: [],
  // For methods, implement them directly
  getId: () => 'entity-id',
  setName: (name: string) => {},
};

// Then use jest.spyOn to mock the methods when needed - STORE IN VARIABLE
const getIdSpy = jest.spyOn(mockEntity, 'getId');
getIdSpy.mockReturnValue('mocked-id');
```

## Commands for Running Tests

```bash
# Run all tests
yarn test

# Run specific test file
yarn test specific-file.spec.ts

# Run tests in watch mode
yarn test:watch

# Run tests with coverage
yarn test:cov

# Run tests in debug mode
yarn test:debug

# Run e2e tests
yarn test:e2e
```

## Continuous Integration Testing

- All tests must pass before merging PRs
- Coverage reports are generated on CI
- Tests should run quickly (< 5 minutes in CI)

## Best Practices

1. **Initialize objects once per test suite** - Create objects and spies at the describe level
2. **Always use jest.spyOn for mocking** - Use jest.spyOn instead of jest.fn() when possible
3. **Always store spy results in variables** - Never use jest.spyOn without storing the result
4. **Don't mock pure utility libraries** - Never mock lodash or similar utility functions
5. **Don't test implementation details** - Test behavior, not how it's implemented
6. **Keep tests simple** - Each test should verify one specific behavior
7. **Use descriptive assertions** - Make failure messages clear
8. **Avoid test interdependence** - Tests should not rely on other tests
9. **Mock external services** - Don't rely on external APIs or databases
10. **Test failure cases** - Don't just test the happy path
11. **Refactor tests as code evolves** - Tests should be maintained like production code
12. **Use factory functions** - Create helpers to generate test data

## Example Test (Reference Implementation)

```typescript
import { Test, TestingModule } from '@nestjs/testing';
import { YourService } from './your.service';
import { YourRepository } from './your.repository';
import { DependencyService } from './dependency.service';

describe('src/your-module/your.service.ts', () => {
  // Define variables at the top
  let service: YourService;
  let repository: YourRepository;
  let dependencyService: DependencyService;
  let module: TestingModule;

  // Create objects once for the entire test suite
  // This improves performance by not recreating objects for each test
  const repository = new YourRepository();
  const dependencyService = new DependencyService();
  
  // Set up spies once - ALWAYS store in variables
  const findSpy = jest.spyOn(repository, 'find');
  const findOneSpy = jest.spyOn(repository, 'findOne');
  const saveSpy = jest.spyOn(repository, 'save');
  const performActionSpy = jest.spyOn(dependencyService, 'performAction');
  
  // For DI scenarios, objects can be created once too
  const mockRepository = {
    find: jest.fn(),
    findOne: jest.fn(),
    save: jest.fn()
  };
  
  const mockDependencyService = {
    performAction: jest.fn()
  };

  // One-time setup of the testing module
  beforeAll(async () => {
    module = await Test.createTestingModule({
      providers: [
        YourService,
        {
          provide: YourRepository,
          useValue: mockRepository,
        },
        {
          provide: DependencyService,
          useValue: mockDependencyService,
        },
      ],
    }).compile();

    service = module.get<YourService>(YourService);
    repository = module.get<YourRepository>(YourRepository);
    dependencyService = module.get<DependencyService>(DependencyService);
  });

  // Reset mocks before each test
  beforeEach(() => {
    jest.clearAllMocks();
    // Set default mock implementations for this test run
    findSpy.mockResolvedValue([]);
    findOneSpy.mockResolvedValue(null);
    saveSpy.mockResolvedValue(null);
    performActionSpy.mockResolvedValue('success');
  });
  
  // Clean up after all tests are done
  afterAll(async () => {
    await module.close();
  });

  it('should be defined', () => {
    expect(service).toBeDefined();
  });

  describe('findById', () => {
    it('should return an item when found', async () => {
      // Arrange
      const mockItem = { id: '1', name: 'Test' };
      // Create a new spy or reuse the existing one
      findOneSpy.mockResolvedValue(mockItem);
      // For DI mocks: mockRepository.findOne.mockResolvedValue(mockItem);

      // Act
      const result = await service.findById('1');

      // Assert
      expect(findOneSpy).toHaveBeenCalledWith({ id: '1' });
      // For DI mocks: expect(mockRepository.findOne).toHaveBeenCalledWith({ id: '1' });
      expect(result).toEqual(mockItem);
    });

    it('should return null when item not found', async () => {
      // Arrange
      findOneSpy.mockResolvedValue(null);
      // For DI mocks: mockRepository.findOne.mockResolvedValue(null);

      // Act
      const result = await service.findById('nonexistent');

      // Assert
      expect(findOneSpy).toHaveBeenCalledWith({ id: 'nonexistent' });
      // For DI mocks: expect(mockRepository.findOne).toHaveBeenCalledWith({ id: 'nonexistent' });
      expect(result).toBeNull();
    });
  });

  describe('create', () => {
    it('should create an item successfully', async () => {
      // Arrange
      const createDto = { name: 'New Item' };
      const savedItem = { id: '1', ...createDto };
      saveSpy.mockResolvedValue(savedItem);
      performActionSpy.mockResolvedValue('success');
      // For DI mocks:
      // mockRepository.save.mockResolvedValue(savedItem);
      // mockDependencyService.performAction.mockResolvedValue('success');

      // Act
      const result = await service.create(createDto);

      // Assert
      expect(performActionSpy).toHaveBeenCalled();
      expect(saveSpy).toHaveBeenCalledWith(createDto);
      // For DI mocks:
      // expect(mockDependencyService.performAction).toHaveBeenCalled();
      // expect(mockRepository.save).toHaveBeenCalledWith(createDto);
      expect(result).toEqual(savedItem);
    });
  });
});
```

These guidelines should be followed for all new tests in the Vietnam Coral Reef Monitoring System project. Adhering to these standards will ensure consistent, maintainable, and effective tests throughout the codebase.
