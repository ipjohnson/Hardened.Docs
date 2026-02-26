# Mocking

The `[Mock]` attribute provides seamless NSubstitute mock creation and registration within Hardened tests. Apply it to a test method parameter and the framework creates an NSubstitute mock, registers it in the DI container as a singleton, and injects it into your test.

**Package:** `Hardened.Shared.Testing` (namespace `Hardened.Shared.Testing.Attributes`)

---

## [Mock] Attribute

### Definition

```csharp
using NSubstitute;

namespace Hardened.Shared.Testing.Attributes;

[AttributeUsage(AttributeTargets.Parameter)]
public class MockAttribute : Attribute, IHardenedParameterProviderAttribute {
    // Creates NSubstitute.Substitute.For() of the parameter type
    // Registers the mock as a singleton in the test DI container
}
```

### How It Works

When the test framework encounters a parameter decorated with `[Mock]`:

1. It calls `NSubstitute.Substitute.For()` with the parameter's type
2. It registers the mock as a **singleton** in the test's `IServiceCollection`
3. The mock replaces any existing registration for that type
4. The mock instance is injected into the test method parameter

Because the mock is registered as a singleton, any service that depends on the mocked interface will receive the same mock instance.

---

## Basic Usage

```csharp
using Hardened.Shared.Testing.Attributes;
using NSubstitute;
using Xunit;

public class OrderServiceTests {
    [HardenedTest]
    public async Task CreateOrder_CallsRepository(
        IOrderService orderService,
        [Mock] IOrderRepository repository) {
        // Arrange: configure the mock
        repository.Save(Arg.Any<Order>())
            .Returns(Task.CompletedTask);

        // Act
        var result = await orderService.Create(new CreateOrderRequest {
            CustomerId = "cust-1"
        });

        // Assert: verify the mock was called
        await repository.Received(1).Save(Arg.Any<Order>());
    }
}
```

In this example:

- `IOrderService` is resolved from the real DI container (the actual implementation)
- `IOrderRepository` is an NSubstitute mock that replaces the real repository
- When `orderService.Create` calls its injected `IOrderRepository`, it hits the mock

---

## Configuring Mock Behavior

Since mocks are NSubstitute proxies, you use the full NSubstitute API to configure them.

### Return Values

```csharp
[HardenedTest]
public async Task GetOrder_ReturnsFromRepository(
    IOrderService orderService,
    [Mock] IOrderRepository repository) {
    var expected = new Order { Id = "ord-1", CustomerId = "cust-1" };

    repository.GetById("ord-1")
        .Returns(Task.FromResult<Order?>(expected));

    var result = await orderService.GetById("ord-1");

    Assert.NotNull(result);
    Assert.Equal("ord-1", result.Id);
}
```

### Throwing Exceptions

```csharp
[HardenedTest]
public async Task GetOrder_HandlesRepositoryError(
    IOrderService orderService,
    [Mock] IOrderRepository repository) {
    repository.GetById(Arg.Any<string>())
        .ThrowsAsync(new TimeoutException("DB timeout"));

    await Assert.ThrowsAsync<ServiceException>(
        () => orderService.GetById("ord-1"));
}
```

### Argument Matching

```csharp
[HardenedTest]
public async Task CreateOrder_SavesWithCorrectCustomer(
    IOrderService orderService,
    [Mock] IOrderRepository repository) {
    await orderService.Create(new CreateOrderRequest {
        CustomerId = "cust-42"
    });

    await repository.Received(1).Save(
        Arg.Is<Order>(o => o.CustomerId == "cust-42"));
}
```

### Callbacks

```csharp
[HardenedTest]
public async Task CreateOrder_GeneratesId(
    IOrderService orderService,
    [Mock] IOrderRepository repository) {
    Order? savedOrder = null;

    repository.Save(Arg.Any<Order>())
        .Returns(Task.CompletedTask)
        .AndDoes(info => savedOrder = info.Arg<Order>());

    await orderService.Create(new CreateOrderRequest {
        CustomerId = "cust-1"
    });

    Assert.NotNull(savedOrder);
    Assert.False(string.IsNullOrEmpty(savedOrder.Id));
}
```

---

## Multiple Mocks

You can mock multiple dependencies in the same test:

```csharp
[HardenedTest]
public async Task ProcessOrder_CoordinatesServices(
    IOrderProcessor processor,
    [Mock] IOrderRepository repository,
    [Mock] IPaymentGateway payment,
    [Mock] INotificationService notifications) {

    // Configure mocks
    repository.GetById("ord-1")
        .Returns(Task.FromResult<Order?>(
            new Order { Id = "ord-1", Total = 99.99m }));

    payment.Charge(99.99m, Arg.Any<string>())
        .Returns(Task.FromResult(new PaymentResult { Success = true }));

    // Act
    await processor.Process("ord-1");

    // Verify interactions
    await payment.Received(1).Charge(99.99m, Arg.Any<string>());
    await notifications.Received(1).Send(
        Arg.Any<string>(), Arg.Any<string>());
}
```

---

## Mock Overrides Real Registrations

When `[Mock]` is used, the mock replaces any existing DI registration for that type. This means your real service implementations are swapped out transparently:

```csharp
// In the application:
[Expose(typeof(IEmailSender))]
public class SmtpEmailSender : IEmailSender {
    public Task Send(string to, string body) {
        // Real SMTP call -- you do NOT want this in tests
    }
}

// In tests:
[HardenedTest]
public async Task Registration_SendsEmail(
    IRegistrationService registration,
    [Mock] IEmailSender emailSender) {
    // SmtpEmailSender is replaced by the mock
    // No real emails are sent

    await registration.Register(new User { Email = "test@example.com" });

    await emailSender.Received(1).Send(
        "test@example.com",
        Arg.Any<string>());
}
```

---

## Mixing Mocks and Real Services

A key benefit of `[Mock]` is selective mocking. You can mock external dependencies while keeping internal services real:

```csharp
[HardenedTest]
public async Task FullWorkflow_WithMockedExternals(
    IOrderWorkflow workflow,           // Real implementation
    IOrderValidator validator,          // Real implementation (auto-resolved)
    [Mock] IPaymentGateway payment,    // Mocked
    [Mock] IEmailSender email) {       // Mocked

    payment.Charge(Arg.Any<decimal>(), Arg.Any<string>())
        .Returns(Task.FromResult(new PaymentResult { Success = true }));

    // The real workflow calls real validator, but mocked payment and email
    var result = await workflow.Execute(new OrderRequest {
        CustomerId = "cust-1",
        Total = 50.00m
    });

    Assert.True(result.Success);
    await payment.Received(1).Charge(50.00m, Arg.Any<string>());
    await email.Received(1).Send(Arg.Any<string>(), Arg.Any<string>());
}
```

---

## NSubstitute Reference

`[Mock]` uses NSubstitute under the hood. Here is a quick reference of commonly used NSubstitute patterns:

| Pattern | Purpose |
|---|---|
| `mock.Method().Returns(value)` | Configure return value |
| `mock.Method().ThrowsAsync(ex)` | Configure exception |
| `mock.Received(n).Method()` | Verify call count |
| `mock.DidNotReceive().Method()` | Verify not called |
| `Arg.Any<T>()` | Match any argument |
| `Arg.Is<T>(predicate)` | Match argument with predicate |
| `.AndDoes(info => ...)` | Execute callback on call |

For the full NSubstitute API, see the [NSubstitute documentation](https://nsubstitute.github.io/help.html).

---

## Related Pages

- [HardenedTest](hardened-test.md) -- test attribute and DI bootstrap
- [Web Testing](web-testing.md) -- HTTP endpoint testing with `ITestWebApp`
- [Custom Attributes](custom-attributes.md) -- building your own parameter providers
