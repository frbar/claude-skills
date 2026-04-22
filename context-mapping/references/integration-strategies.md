# Integration Strategies Reference

Technical implementation patterns for each context mapping relationship type.

## Shared Kernel Implementation

### Strategy: Shared Library/Package

```text
┌─────────────────┐     ┌─────────────────┐
│   Context A     │     │   Context B     │
│                 │     │                 │
│   References    │     │   References    │
│       │         │     │       │         │
│       ▼         │     │       ▼         │
│   ┌───────────────────────────┐         │
│   │     SharedKernel.dll      │         │
│   │  (NuGet internal feed)    │         │
│   └───────────────────────────┘         │
└─────────────────┘     └─────────────────┘
```

**Implementation:**

```xml
<!-- SharedKernel.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <PackageId>Company.SharedKernel</PackageId>
    <Version>1.0.0</Version>
  </PropertyGroup>
</Project>
```

```csharp
// SharedKernel contents
namespace Company.SharedKernel;

// Shared value objects
public record CustomerId(Guid Value);
public record Money(decimal Amount, Currency Currency);

// Shared base classes
public abstract class Entity<TId>
{
    public TId Id { get; protected set; }
}
```

**Best Practices:**

- Keep kernel small (value objects, base classes)
- Version semantically (breaking changes = major version)
- Both teams must approve changes
- Single repository for kernel code

---

## Partnership Implementation

### Strategy: Bidirectional Events

```text
┌─────────────────┐                     ┌─────────────────┐
│   Context A     │                     │   Context B     │
│                 │──── Events A ────►  │                 │
│                 │                     │                 │
│                 │◄─── Events B ────   │                 │
└─────────────────┘                     └─────────────────┘
              │                                   │
              └───────────┬───────────────────────┘
                          │
                   ┌──────▼──────┐
                   │  Event Bus  │
                   │ (MediatR/   │
                   │  MassTransit)│
                   └─────────────┘
```

**Implementation with MediatR:**

```csharp
// Context A publishes
public class OrderContext
{
    public async Task CompleteOrder(OrderId id)
    {
        // ... business logic
        await _mediator.Publish(new OrderCompletedEvent(id));
    }
}

// Context B handles and may respond
public class ShippingContext : INotificationHandler<OrderCompletedEvent>
{
    public async Task Handle(OrderCompletedEvent notification, CancellationToken ct)
    {
        // Create shipment
        var shipment = CreateShipment(notification.OrderId);

        // Publish back
        await _mediator.Publish(new ShipmentCreatedEvent(shipment.Id));
    }
}
```

---

## Customer/Supplier Implementation

### Strategy: Internal API

```text
┌─────────────────┐                     ┌─────────────────┐
│   Downstream    │                     │   Upstream      │
│   (Customer)    │──── API Call ────►  │   (Supplier)    │
│                 │                     │                 │
│                 │◄─── Response ────   │                 │
└─────────────────┘                     └─────────────────┘
```

**Implementation with Typed HTTP Client:**

```csharp
// Upstream: InventoryContext exposes internal API
[ApiController]
[Route("api/internal/inventory")]
public class InternalInventoryController : ControllerBase
{
    [HttpGet("products/{productId}/availability")]
    public async Task<ActionResult<AvailabilityDto>> GetAvailability(Guid productId)
    {
        var availability = await _service.CheckAvailability(productId);
        return Ok(availability.ToDto());
    }
}

// Downstream: OrderContext consumes via typed client
public interface IInventoryClient
{
    Task<Availability> GetAvailability(ProductId productId);
}

public class InventoryClient : IInventoryClient
{
    private readonly HttpClient _client;

    public async Task<Availability> GetAvailability(ProductId productId)
    {
        var response = await _client.GetFromJsonAsync<AvailabilityDto>(
            $"api/internal/inventory/products/{productId}/availability");
        return response.ToDomain();
    }
}

// Registration
services.AddHttpClient<IInventoryClient, InventoryClient>(client =>
{
    client.BaseAddress = new Uri(config["Services:Inventory"]);
});
```

---

## Conformist Implementation

### Strategy: Direct Model Adoption

```text
┌─────────────────┐                     ┌─────────────────┐
│   Downstream    │                     │   Upstream      │
│   (Conforms)    │──── Uses directly───│   Model        │
│                 │                     │                 │
│   Uses upstream │                     │                 │
│   DTOs as-is    │                     │                 │
└─────────────────┘                     └─────────────────┘
```

**Implementation:**

```csharp
// Upstream model (e.g., data warehouse schema)
namespace DataWarehouse.Contracts;

public class SalesFactDto
{
    public DateTime SaleDate { get; set; }
    public string ProductSku { get; set; }
    public decimal Amount { get; set; }
    public string Region { get; set; }
}

// Downstream uses directly - no translation
namespace Reporting.Application;

public class SalesReportService
{
    private readonly IDataWarehouseClient _warehouse;

    public async Task<SalesReport> GenerateReport(DateRange range)
    {
        // Use upstream model directly
        var facts = await _warehouse.GetSalesFacts(range.Start, range.End);

        // Work with their schema
        var byRegion = facts.GroupBy(f => f.Region);

        return new SalesReport(byRegion);
    }
}
```

**When Appropriate:**

- Upstream model is well-designed
- Minimal friction with domain language
- Upstream is stable
- Translation cost exceeds value

---

## Anti-Corruption Layer Implementation

### Strategy: Adapter Pattern

```text
┌─────────────────────────────────────┐
│           Downstream                │
│                                     │
│   ┌─────────────────────────────┐   │
│   │      Domain Model           │   │
│   │   (PaymentRequest, etc.)    │   │
│   └───────────────▲─────────────┘   │
│                   │                 │
│   ┌───────────────┴─────────────┐   │
│   │    Anti-Corruption Layer    │   │
│   │  ┌───────────────────────┐  │   │
│   │  │   Gateway Adapter     │  │   │
│   │  │   (translates both    │  │   │
│   │  │    directions)        │  │   │
│   │  └───────────────────────┘  │   │
│   └───────────────▲─────────────┘   │
│                   │                 │
└───────────────────┼─────────────────┘
                    │
          ┌─────────▼─────────┐
          │   External API    │
          │   (Foreign model) │
          └───────────────────┘
```

**Implementation:**

```csharp
// Domain model (what we want to work with)
namespace Payment.Domain;

public record PaymentRequest(
    Money Amount,
    PaymentMethod Method,
    OrderReference Order
);

public record PaymentResult(
    bool Success,
    TransactionId? TransactionId,
    PaymentFailureReason? FailureReason
);

// External model (what the gateway uses)
namespace Payment.Infrastructure.External;

public class GatewayChargeRequest
{
    public decimal amount { get; set; }
    public string currency { get; set; }
    public string payment_token { get; set; }
    public string idempotency_key { get; set; }
}

public class GatewayChargeResponse
{
    public string id { get; set; }
    public string status { get; set; } // "approved", "declined"
    public string decline_code { get; set; }
}

// ACL: The adapter that translates
namespace Payment.Infrastructure.AntiCorruption;

public class PaymentGatewayAdapter : IPaymentGateway
{
    private readonly ExternalGatewayClient _client;

    public async Task<PaymentResult> ProcessPayment(PaymentRequest request)
    {
        // Translate TO external format
        var externalRequest = TranslateToExternal(request);

        // Call external service
        var externalResponse = await _client.ChargeAsync(externalRequest);

        // Translate FROM external format
        return TranslateFromExternal(externalResponse);
    }

    private GatewayChargeRequest TranslateToExternal(PaymentRequest request)
    {
        return new GatewayChargeRequest
        {
            amount = request.Amount.Value,
            currency = request.Amount.Currency.Code,
            payment_token = request.Method.Token,
            idempotency_key = request.Order.Id.ToString()
        };
    }

    private PaymentResult TranslateFromExternal(GatewayChargeResponse response)
    {
        return new PaymentResult(
            Success: response.status == "approved",
            TransactionId: response.status == "approved"
                ? new TransactionId(response.id)
                : null,
            FailureReason: response.status != "approved"
                ? MapDeclineCode(response.decline_code)
                : null
        );
    }

    private PaymentFailureReason MapDeclineCode(string code) => code switch
    {
        "insufficient_funds" => PaymentFailureReason.InsufficientFunds,
        "card_expired" => PaymentFailureReason.CardExpired,
        "fraud_detected" => PaymentFailureReason.FraudSuspected,
        _ => PaymentFailureReason.Unknown
    };
}
```

---

## Open Host Service Implementation

### Strategy: Formal REST/gRPC API

```text
┌─────────────────────────────────────┐
│           Upstream (Host)           │
│                                     │
│   ┌─────────────────────────────┐   │
│   │       Domain Model          │   │
│   └───────────────▲─────────────┘   │
│                   │                 │
│   ┌───────────────┴─────────────┐   │
│   │      Application Layer      │   │
│   └───────────────▲─────────────┘   │
│                   │                 │
│   ┌───────────────┴─────────────┐   │
│   │    Open Host Service API    │   │
│   │   - REST Controllers        │   │
│   │   - OpenAPI Spec            │   │
│   │   - Versioning              │   │
│   └───────────────▲─────────────┘   │
└───────────────────┼─────────────────┘
                    │
    ┌───────────────┼───────────────┐
    │               │               │
┌───▼───┐     ┌─────▼────┐    ┌─────▼────┐
│Client │     │ Client B │    │ Client C │
│   A   │     │          │    │          │
└───────┘     └──────────┘    └──────────┘
```

**Implementation:**

```csharp
// API versioning setup
services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true;
});

// Versioned controller
[ApiController]
[ApiVersion("1.0")]
[Route("api/v{version:apiVersion}/products")]
public class ProductsController : ControllerBase
{
    [HttpGet("{id}")]
    [ProducesResponseType(typeof(ProductDto), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<ActionResult<ProductDto>> GetProduct(Guid id)
    {
        var product = await _service.GetById(new ProductId(id));
        if (product is null) return NotFound();
        return Ok(product.ToDto());
    }
}
```

---

## Published Language Implementation

### Strategy: OpenAPI/AsyncAPI Contracts

```yaml
# products-api.yaml (Published Language)
openapi: 3.0.3
info:
  title: Product Catalog API
  version: 1.0.0
  description: Official API contract for Product Catalog

paths:
  /products/{productId}:
    get:
      operationId: getProduct
      parameters:
        - name: productId
          in: path
          required: true
          schema:
            type: string
            format: uuid
      responses:
        '200':
          description: Product details
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Product'

components:
  schemas:
    Product:
      type: object
      required:
        - id
        - name
        - price
      properties:
        id:
          type: string
          format: uuid
        name:
          type: string
        price:
          $ref: '#/components/schemas/Money'

    Money:
      type: object
      required:
        - amount
        - currency
      properties:
        amount:
          type: number
          format: decimal
        currency:
          type: string
          pattern: '^[A-Z]{3}$'
```

**Client Generation:**

```bash
# Generate typed client from OpenAPI spec
dotnet openapi add url https://catalog.internal/openapi.json \
  --output-file ProductCatalogClient.cs
```

---

## Pattern-to-Implementation Quick Reference

| Pattern | Primary Strategy | .NET Technology |
| ------- | --------------- | --------------- |
| Shared Kernel | Shared NuGet | Internal NuGet feed |
| Partnership | Bidirectional events | MediatR, MassTransit |
| Customer/Supplier | Internal API | Typed HttpClient |
| Conformist | Direct usage | Reference DTOs directly |
| ACL | Adapter pattern | Interface + Adapter class |
| OHS | Formal API | ASP.NET + API versioning |
| PL | Contract-first | OpenAPI, Protobuf |
| Separate Ways | None | No integration code |
