# Context Mapping Pattern Catalog

Complete reference for all 8 DDD context mapping patterns with examples, use cases, and trade-offs.

## Symmetric Patterns

### Shared Kernel (SK)

**Definition:** Two bounded contexts share a common subset of the domain model, including code and possibly database schema.

**Visual:**

```text
┌─────────────────┐     ┌─────────────────┐
│   Context A     │     │   Context B     │
│                 │     │                 │
│    ┌───────┐    │     │    ┌───────┐    │
│    │Shared │◄───┼─────┼───►│Shared │    │
│    │Kernel │    │     │    │Kernel │    │
│    └───────┘    │     │    └───────┘    │
└─────────────────┘     └─────────────────┘
```

**CML Syntax:**

```cml
ContextA [SK]<->[SK] ContextB
```

**When to Use:**

- Two contexts need identical business rules
- Shared value objects or entities
- Tight team collaboration exists
- Changes to shared code can be coordinated

**Trade-offs:**

| Pros | Cons |
| ---- | ---- |
| Eliminates duplication | Tight coupling |
| Consistent business rules | Coordination overhead |
| Single source of truth | Risk of growing too large |
| Simpler integration | Both teams must agree on changes |

**Example:**

```csharp
// Shared Kernel: CustomerInfo value object
// Owned by CustomerContext, used by OrderContext and ShippingContext
namespace SharedKernel;

public record CustomerInfo(
    CustomerId Id,
    string Name,
    Address ShippingAddress,
    Email Email
);
```

**Warning Signs It's Too Big:**

- More than 2-3 entities/value objects
- Frequent merge conflicts
- One team waiting on another for changes
- Shared kernel grows faster than individual contexts

---

### Partnership (P)

**Definition:** Two contexts have mutual dependency with no clear upstream/downstream. Teams collaborate closely and co-evolve their models.

**Visual:**

```text
┌─────────────────┐     ┌─────────────────┐
│   Context A     │◄───►│   Context B     │
│                 │     │                 │
│  (mutual deps)  │     │  (mutual deps)  │
└─────────────────┘     └─────────────────┘
```

**CML Syntax:**

```cml
ContextA [P]<->[P] ContextB
```

**When to Use:**

- Neither team is clearly upstream or downstream
- Bidirectional data/event flow
- Teams have equal leverage and influence
- Joint development on shared features

**Trade-offs:**

| Pros | Cons |
| ---- | ---- |
| Flexible collaboration | Requires strong communication |
| Equal team voice | No single owner for decisions |
| Natural for peer services | Can become chaotic without discipline |
| Supports innovation | Harder to scale beyond 2 teams |

**Example:**

```cml
/* Order and Shipping co-evolve the fulfillment process
 * Both teams contribute to the shipping rules
 */
OrderContext [P]<->[P] ShippingContext
```

**Anti-Pattern Warning:**

If one team consistently waits on another, it's not a true Partnership - consider Customer/Supplier instead.

---

## Asymmetric Patterns

### Customer/Supplier (C/S)

**Definition:** Upstream context (Supplier) provides services/data to downstream context (Customer). The customer has some influence over the supplier's priorities.

**Visual:**

```text
┌─────────────────┐     ┌─────────────────┐
│   Upstream      │     │   Downstream    │
│   (Supplier)    │────►│   (Customer)    │
│                 │     │                 │
│   Provides      │     │   Consumes      │
└─────────────────┘     └─────────────────┘
```

**CML Syntax:**

```cml
DownstreamContext [D,C]<-[U,S] UpstreamContext
```

**When to Use:**

- Clear provider/consumer relationship
- Downstream team can negotiate API changes
- Same organization, reasonable collaboration
- Internal service communication

**Trade-offs:**

| Pros | Cons |
| ---- | ---- |
| Clear ownership | Requires negotiation process |
| Customer voice heard | Slower API evolution |
| Predictable dependencies | Customer depends on supplier roadmap |
| Natural internal API pattern | Conflicts if priorities differ |

**Example:**

```cml
/* Order consumes inventory levels
 * Inventory team accepts feature requests from Order team
 */
OrderContext [D,C]<-[U,S] InventoryContext
```

---

### Conformist (CF)

**Definition:** Downstream context adopts upstream model wholesale, with no translation layer. The downstream team has no influence over upstream design.

**Visual:**

```text
┌─────────────────┐     ┌─────────────────┐
│   Upstream      │     │   Downstream    │
│   (Dictates)    │────►│   (Conforms)    │
│                 │     │                 │
│   No negotiation│     │   Accepts model │
└─────────────────┘     └─────────────────┘
```

**CML Syntax:**

```cml
DownstreamContext [D,CF]<-[U] UpstreamContext
```

**When to Use:**

- Third-party system you can't change
- Upstream model is well-designed for your needs
- Translation cost exceeds conformance cost
- Rapid integration needed

**Trade-offs:**

| Pros | Cons |
| ---- | ---- |
| Fast integration | Upstream model leaks into domain |
| No translation overhead | Vulnerable to upstream changes |
| Simple implementation | Model may not fit domain language |
| Good for stable APIs | Limited domain expression |

**Example:**

```cml
/* Analytics conforms to the data warehouse schema
 * No point translating - schema is designed for our reports
 */
AnalyticsContext [D,CF]<-[U] DataWarehouseContext
```

**When NOT to Conform:**

- Upstream model conflicts with your domain language
- Upstream changes frequently
- Model impedance causes significant friction
- You have resources for an ACL

---

### Anti-Corruption Layer (ACL)

**Definition:** Downstream context creates a translation layer to isolate itself from an incompatible upstream model.

**Visual:**

```text
┌─────────────────┐     ┌─────────┐     ┌─────────────────┐
│   Upstream      │     │   ACL   │     │   Downstream    │
│   (Foreign)     │────►│Translate│────►│   (Protected)   │
│                 │     │         │     │                 │
│   External model│     │ Adapter │     │   Domain model  │
└─────────────────┘     └─────────┘     └─────────────────┘
```

**CML Syntax:**

```cml
DownstreamContext [D,ACL]<-[U] UpstreamContext
```

**When to Use:**

- Upstream model doesn't match your domain language
- Integrating with legacy systems
- External third-party APIs
- Protecting domain purity is important

**Trade-offs:**

| Pros | Cons |
| ---- | ---- |
| Domain model stays clean | Translation code to maintain |
| Isolates from upstream changes | Performance overhead |
| Clear boundary | More moving parts |
| Enables domain evolution | Duplication of concepts |

**Example:**

```csharp
// ACL: Translates external payment gateway to domain model
namespace Payment.Infrastructure.AntiCorruption;

public class PaymentGatewayAdapter : IPaymentService
{
    private readonly ExternalPaymentClient _client;

    public async Task<PaymentResult> ProcessPayment(PaymentRequest request)
    {
        // Translate domain model to external format
        var externalRequest = new GatewayPaymentRequest
        {
            Amount = request.Amount.Value,
            Currency = request.Amount.Currency.Code,
            CardToken = request.PaymentMethod.Token
        };

        // Call external service
        var response = await _client.ChargeAsync(externalRequest);

        // Translate response back to domain model
        return new PaymentResult(
            Success: response.Status == "approved",
            TransactionId: new TransactionId(response.Id),
            FailureReason: response.DeclineReason
        );
    }
}
```

---

### Open Host Service (OHS)

**Definition:** Upstream context provides a formal, well-documented API for consumers. Multiple downstream contexts can integrate through this standardized interface.

**Visual:**

```text
┌─────────────────┐     ┌─────────────────┐
│   Upstream      │     │   Downstream A  │
│   (Host)        │────►│                 │
│                 │     └─────────────────┘
│   ┌─────────┐   │     ┌─────────────────┐
│   │   API   │───┼────►│   Downstream B  │
│   │(formal) │   │     └─────────────────┘
│   └─────────┘   │     ┌─────────────────┐
│                 │────►│   Downstream C  │
└─────────────────┘     └─────────────────┘
```

**CML Syntax:**

```cml
DownstreamContext [D]<-[U,OHS] UpstreamContext
```

**When to Use:**

- Multiple consumers need the same data/service
- API versioning and stability matter
- External integrations planned
- Team capacity for API design

**Trade-offs:**

| Pros | Cons |
| ---- | ---- |
| Standardized access | API design overhead |
| Decoupled consumers | Versioning complexity |
| Scalable integrations | Documentation burden |
| Clear contract | Change management needed |

---

### Published Language (PL)

**Definition:** A standardized, documented language/schema for cross-context communication. Often combined with Open Host Service.

**CML Syntax:**

```cml
DownstreamContext [D]<-[U,OHS,PL] UpstreamContext
```

**Common Formats:**

- OpenAPI/Swagger specifications
- Protocol Buffers (gRPC)
- JSON Schema
- AsyncAPI (for events)
- GraphQL Schema

**When to Use:**

- Cross-team or cross-organization communication
- Contractual integration requirements
- Multiple programming languages involved
- Long-term API stability needed

**Example:**

```yaml
# OpenAPI spec as Published Language
openapi: 3.0.0
info:
  title: Inventory API
  version: 1.0.0
paths:
  /products/{id}/availability:
    get:
      summary: Get product availability
      responses:
        '200':
          description: Availability information
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Availability'
```

---

## No Integration Pattern

### Separate Ways (SW)

**Definition:** Two contexts have no meaningful relationship and evolve independently. Integration would add complexity without proportional value.

**Visual:**

```text
┌─────────────────┐     ┌─────────────────┐
│   Context A     │     │   Context B     │
│                 │     │                 │
│   Independent   │ ✗   │   Independent   │
│                 │     │                 │
└─────────────────┘     └─────────────────┘
```

**CML Syntax:**

Not typically represented in CML (absence of relationship).

**When to Use:**

- No data or process dependencies exist
- Integration cost exceeds value
- Contexts serve different business capabilities
- Duplicate data is acceptable (or required by compliance)

**Trade-offs:**

| Pros | Cons |
| ---- | ---- |
| Complete autonomy | Potential duplication |
| No coordination needed | No shared learning |
| Simple deployment | Manual sync if needed later |
| Fast independent evolution | Harder to integrate later |

**Example:**

Marketing analytics context has no relationship with internal HR system - they serve completely different purposes and duplicating employee counts is fine.

---

## Pattern Combinations

Patterns are often combined:

| Combination | Use Case |
| ----------- | -------- |
| `[U,S,OHS]` | Supplier provides formal API |
| `[U,OHS,PL]` | Formal API with documented schema |
| `[D,C,ACL]` | Customer with translation layer |
| `[D,ACL]<-[U,OHS,PL]` | Protected consumer of formal API |

**Example Combined:**

```cml
/* Payment gateway: formal API with schema, consumer uses ACL */
PaymentContext [D,ACL]<-[U,OHS,PL] ExternalPaymentGateway
```

---

## Quick Reference Table

| Pattern | Direction | Team Dynamics | Technical Impl |
| ------- | --------- | ------------- | -------------- |
| Shared Kernel | Symmetric | Tight collaboration | Shared library |
| Partnership | Symmetric | Equal peers | Bidirectional events |
| Customer/Supplier | Asymmetric | Negotiated | Internal API |
| Conformist | Asymmetric | Accept as-is | Direct model use |
| Anti-Corruption Layer | Asymmetric | Protected | Adapter pattern |
| Open Host Service | Asymmetric | Standardized | REST/gRPC API |
| Published Language | Asymmetric | Contracted | OpenAPI/Protobuf |
| Separate Ways | None | Independent | No integration |
