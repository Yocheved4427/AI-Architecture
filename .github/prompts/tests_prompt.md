# Prompt for Generating Unit & Integration Tests for WebApiShop

## Framework & Tools
- xUnit with `[Fact]` attributes
- Moq 4.20.72 for mocking dependencies
- Moq.EntityFrameworkCore for mocking `DbSet` operations (`.ReturnsDbSet(...)`)
- `AutoMapper` with a real `MapperConfiguration` in service-level tests — **do not mock `IMapper`**
- Async/await for every async operation

---

## Naming Convention
Follow the pattern: `MethodName_ShouldExpectedResult_WhenCondition`

Examples:
- `GetUsers_ShouldReturnAllUsers`
- `GetUsers_ShouldReturnEmptyList_WhenNoUsersExist`
- `Login_ShouldReturnNull_WhenPasswordIsInvalid`
- `AddOrder_ShouldReturnNull_WhenSumIsIncorrect`

---

## Unit Tests — Repository Level

### Structure
1. Create `var mockContext = new Mock<ApiShopContext>();`
2. Use `.Setup(x => x.DbSet).ReturnsDbSet(testData)` from Moq.EntityFrameworkCore
3. For single-entity lookups use `.Setup(x => x.DbSet.FindAsync(...)).ReturnsAsync(...)`
4. Follow the AAA pattern (Arrange / Act / Assert)
5. Cover both happy paths and edge cases (empty list, null, not-found)

### Example
```csharp
[Fact]
public async Task GetUsers_ShouldReturnAllUsers()
{
    // Arrange
    var users = new List<User>
    {
        new User { UserId = 1, FirstName = "Alice", Email = "alice@test.com", Password = "abc" },
        new User { UserId = 2, FirstName = "Bob",   Email = "bob@test.com",   Password = "xyz" }
    };
    var mockContext = new Mock<ApiShopContext>();
    mockContext.Setup(x => x.Users).ReturnsDbSet(users);
    var repository = new UserRepository(mockContext.Object);

    // Act
    var result = await repository.GetUsers();

    // Assert
    Assert.NotNull(result);
    Assert.Equal(2, result.Count());
}

[Fact]
public async Task GetUsers_ShouldReturnEmptyList_WhenNoUsersExist()
{
    var mockContext = new Mock<ApiShopContext>();
    mockContext.Setup(x => x.Users).ReturnsDbSet(new List<User>());
    var repository = new UserRepository(mockContext.Object);

    var result = await repository.GetUsers();

    Assert.NotNull(result);
    Assert.Empty(result);
}
```

---

## Unit Tests — Service Level

### Structure
1. Mock `IXxxRepository` dependencies
2. Build a real `IMapper` using `MapperConfiguration` — **never mock `IMapper`**
3. Mock `ILogger<T>` where the service requires it
4. Verify repository calls with `.Verify(x => x.Method(It.IsAny<T>()), Times.Once)`
5. Test business-logic rules (price sums, password validation, uniqueness checks)

### Example
```csharp
public class OrdersServicesTests
{
    private readonly Mock<IOrdersRepository> _ordersRepoMock;
    private readonly Mock<IProductsRepository> _productsRepoMock;
    private readonly IMapper _mapper;

    public OrdersServicesTests()
    {
        _ordersRepoMock   = new Mock<IOrdersRepository>();
        _productsRepoMock = new Mock<IProductsRepository>();

        var config = new MapperConfiguration(cfg =>
        {
            cfg.CreateMap<Order, OrderDTO>().ReverseMap();
            cfg.CreateMap<OrderItem, OrderItemDTO>().ReverseMap();
        });
        _mapper = config.CreateMapper();
    }

    [Fact]
    public async Task AddOrder_ShouldReturnOrderDTO_WhenSumIsCorrect()
    {
        // Arrange
        var product = new Product { ProductId = 1, Price = 50 };
        _productsRepoMock.Setup(x => x.GetProductById(1)).ReturnsAsync(product);

        var orderDto = new OrderDTO(
            OrderId: 0,
            UserId: 1,
            OrderDate: DateOnly.FromDateTime(DateTime.Now),
            TotalPrice: 50,
            Items: new List<OrderItemDTO> { new OrderItemDTO(0, 0, 1, 1, 50) }
        );

        _ordersRepoMock.Setup(x => x.AddOrder(It.IsAny<Order>())).ReturnsAsync(new Order());
        var service = new OrdersServices(_ordersRepoMock.Object, _productsRepoMock.Object, _mapper);

        // Act
        var result = await service.AddOrder(orderDto);

        // Assert
        Assert.NotNull(result);
        _ordersRepoMock.Verify(x => x.AddOrder(It.IsAny<Order>()), Times.Once);
    }

    [Fact]
    public async Task AddOrder_ShouldReturnNull_WhenSumIsIncorrect()
    {
        var product = new Product { ProductId = 1, Price = 50 };
        _productsRepoMock.Setup(x => x.GetProductById(1)).ReturnsAsync(product);

        var orderDto = new OrderDTO(
            OrderId: 0,
            UserId: 1,
            OrderDate: DateOnly.FromDateTime(DateTime.Now),
            TotalPrice: 999,   // wrong total
            Items: new List<OrderItemDTO> { new OrderItemDTO(0, 0, 1, 1, 50) }
        );

        var service = new OrdersServices(_ordersRepoMock.Object, _productsRepoMock.Object, _mapper);
        var result  = await service.AddOrder(orderDto);

        Assert.Null(result);
        _ordersRepoMock.Verify(x => x.AddOrder(It.IsAny<Order>()), Times.Never);
    }
}
```

---

## Integration Tests

### Structure
1. Implement `IDisposable` for cleanup
2. Use `DatabaseFixture` which creates a unique SQL Server database per test class
3. Seed the database directly via `_dbContext` before calling the repository
4. Call `_dbContext.ChangeTracker.Clear()` before re-querying to avoid stale EF tracking
5. Use `Assert.Contains(...)` for collection membership; `Assert.Equal` for exact value checks
6. Clean up with `_fixture.Dispose()` in `Dispose()`

### Example
```csharp
public class UserRepositoryIntegrationTests : IDisposable
{
    private readonly DatabaseFixture _fixture;
    private readonly ApiShopContext  _dbContext;
    private readonly UserRepository  _userRepository;

    public UserRepositoryIntegrationTests()
    {
        _fixture        = new DatabaseFixture();
        _dbContext      = _fixture.Context;
        _userRepository = new UserRepository(_dbContext);
    }

    public void Dispose() => _fixture.Dispose();

    [Fact]
    public async Task Update_ShouldPersistChangesInDatabase()
    {
        // Arrange
        var user = new User { FirstName = "Before", Email = "before@test.com", Password = "123" };
        await _dbContext.Users.AddAsync(user);
        await _dbContext.SaveChangesAsync();
        _dbContext.ChangeTracker.Clear();          // detach so EF doesn't return cached entity

        // Act
        user.FirstName = "After";
        await _userRepository.Update(user.UserId, user);
        _dbContext.ChangeTracker.Clear();

        // Assert
        var updated = await _dbContext.Users.FindAsync(user.UserId);
        Assert.Equal("After", updated!.FirstName);
    }
}
```

---

## Test Coverage Areas

### Each Repository
| Scenario | Unit test | Integration test |
|---|---|---|
| GetAll — returns data | Yes | Yes |
| GetAll — returns empty list | Yes | Yes |
| GetById / Login — returns entity | Yes | Yes |
| GetById — returns null | Yes | Yes |
| Add — persists & returns entity | Yes | Yes |
| Update — modifies entity | Yes | Yes |
| Delete — removes entity | Yes | Yes |

### Each Service
- Business-logic validation (order totals, password matching, unique email)
- DTO mapping produces correct field values
- Repository methods called with correct arguments (`.Verify`)
- Returns `null` on invalid input; returns DTO on success

---

## Key Conventions

| Rule | Detail |
|---|---|
| DB context class | `ApiShopContext` |
| Primary key names | `UserId`, `ProductId`, `CategoryId`, `OrderId` |
| DTOs are C# records | Construct with positional syntax `new OrderDTO(0, ...)` |
| Date fields | `DateOnly.FromDateTime(DateTime.Now)` |
| Async verification | `.Verify(x => x.Method(It.IsAny<T>()), Times.Once)` |
| EF tracking reset | `_dbContext.ChangeTracker.Clear()` before re-querying in integration tests |
| Do not mock `IMapper` | Always create a real `MapperConfiguration` in service tests |
| Assertions | `Assert.Equal`, `Assert.NotNull`, `Assert.Null`, `Assert.Empty`, `Assert.Contains` |
