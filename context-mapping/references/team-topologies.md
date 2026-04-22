# Team Topologies and Context Mapping

Guidance for aligning team structures with bounded context relationships.

## Team Topologies Overview

From Matthew Skelton and Manuel Pais's Team Topologies:

| Team Type | Purpose | Context Mapping Relevance |
| --------- | ------- | ------------------------- |
| **Stream-aligned** | End-to-end flow of value | Owns bounded context(s) |
| **Platform** | Internal services for others | OHS/PL provider |
| **Enabling** | Helps others adopt practices | Supports multiple contexts |
| **Complicated Subsystem** | Deep specialist knowledge | May own complex context |

## Mapping Patterns to Team Structures

### Shared Kernel

**Team Structure:** Same team or tightly coupled teams

```text
┌─────────────────────────────────────┐
│           Single Team               │
│  ┌─────────────┐ ┌─────────────┐    │
│  │  Context A  │ │  Context B  │    │
│  │     │       │ │      │      │    │
│  │     └───────┼─┼──────┘      │    │
│  │         SK  │ │             │    │
│  └─────────────┘ └─────────────┘    │
└─────────────────────────────────────┘
```

**Recommendation:**

- Single stream-aligned team owns both contexts
- Or: Tight partnership with shared ownership
- Shared kernel should be small enough for single team

---

### Partnership

**Team Structure:** Two stream-aligned teams with interaction mode X-as-a-Service

```text
┌─────────────────┐     ┌─────────────────┐
│  Team A         │◄───►│  Team B         │
│  (Stream)       │     │  (Stream)       │
│                 │     │                 │
│  ┌───────────┐  │     │  ┌───────────┐  │
│  │ Context A │  │     │  │ Context B │  │
│  └───────────┘  │     │  └───────────┘  │
└─────────────────┘     └─────────────────┘
      Collaboration Mode
```

**Recommendation:**

- Both teams stream-aligned
- Collaboration interaction mode
- Regular joint planning sessions
- Consider merging if collaboration too frequent

---

### Customer/Supplier

**Team Structure:** Provider team and consumer team

```text
┌─────────────────┐
│  Consumer Team  │
│  (Stream)       │
│                 │
│  ┌───────────┐  │      Feature
│  │ Context A │  │◄─── Requests
│  └─────┬─────┘  │
└────────┼────────┘
         │ API
         │
┌────────▼────────┐
│  Provider Team  │
│  (Stream or     │
│   Platform)     │
│  ┌───────────┐  │
│  │ Context B │  │
│  └───────────┘  │
└─────────────────┘
```

**Recommendation:**

- Provider: Stream-aligned or Platform team
- Consumer: Stream-aligned team
- X-as-a-Service interaction mode
- Provider roadmap influenced by consumer needs

---

### Conformist

**Team Structure:** Consumer team accepting external model

```text
┌─────────────────┐
│  Consumer Team  │
│  (Stream)       │
│                 │
│  ┌───────────┐  │
│  │ Context A │  │
│  │ (conforms)│  │
│  └─────┬─────┘  │
└────────┼────────┘
         │ Uses as-is
         │
┌────────▼────────┐
│  External/      │
│  Vendor System  │
│  (No team       │
│   influence)    │
└─────────────────┘
```

**Recommendation:**

- Consumer team accepts external model
- No negotiation overhead
- Consider ACL if model friction high

---

### Anti-Corruption Layer

**Team Structure:** Consumer team with protective boundary

```text
┌─────────────────────────────────┐
│  Consumer Team (Stream)         │
│                                 │
│  ┌───────────┐   ┌───────────┐  │
│  │ Context A │◄──│    ACL    │  │
│  │  (pure)   │   │ (adapter) │  │
│  └───────────┘   └─────┬─────┘  │
└────────────────────────┼────────┘
                         │ Translates
                         │
              ┌──────────▼──────────┐
              │   External System   │
              │   (foreign model)   │
              └─────────────────────┘
```

**Recommendation:**

- Consumer team owns ACL code
- Team maintains translation layer
- Enabling team may help with patterns

---

### Open Host Service

**Team Structure:** Platform team serving multiple consumers

```text
         ┌─────────────────┐
         │  Platform Team  │
         │                 │
         │  ┌───────────┐  │
         │  │  Context  │  │
         │  │  + OHS    │  │
         │  └─────┬─────┘  │
         └────────┼────────┘
                  │ Formal API
      ┌───────────┼───────────┐
      │           │           │
┌─────▼─────┐ ┌───▼───┐ ┌─────▼─────┐
│  Team A   │ │Team B │ │  Team C   │
│ (Stream)  │ │(Stream)│ │ (Stream) │
└───────────┘ └───────┘ └───────────┘
```

**Recommendation:**

- Provider is Platform team
- Consumers are Stream-aligned teams
- X-as-a-Service interaction mode
- API versioning essential

---

### Published Language

**Team Structure:** Platform team with formal contracts

```text
┌─────────────────────────────────┐
│  Platform Team                  │
│                                 │
│  ┌───────────┐   ┌───────────┐  │
│  │  Context  │   │  OpenAPI  │  │
│  │           │◄──│   Spec    │  │
│  └───────────┘   │  (PL)     │  │
│                  └─────┬─────┘  │
└────────────────────────┼────────┘
                         │ Contract-first
                         │
              Consumers generate clients
              from published spec
```

**Recommendation:**

- Contract owned by Platform team
- Contract-first development
- Breaking changes managed carefully

---

## Interaction Modes by Pattern

| Pattern | Recommended Mode | Frequency |
| ------- | ---------------- | --------- |
| Shared Kernel | Collaboration | High |
| Partnership | Collaboration | Medium-High |
| Customer/Supplier | X-as-a-Service | Medium |
| Conformist | X-as-a-Service | Low |
| ACL | X-as-a-Service | Low |
| OHS | X-as-a-Service | Low |
| PL | X-as-a-Service | Low |
| Separate Ways | None | None |

## Cognitive Load Considerations

### Context-to-Team Mapping

**Rule of Thumb:** One stream-aligned team should own 1-3 bounded contexts maximum.

**Signs of Overload:**

- Team context-switching between unrelated domains
- Slow delivery due to too many responsibilities
- Knowledge silos within team

**Solutions:**

- Split team along context boundaries
- Create platform team for shared infrastructure
- Use enabling team to reduce complexity

### Pattern Complexity Impact

| Pattern | Cognitive Load | Team Capacity Needed |
| ------- | -------------- | -------------------- |
| Shared Kernel | Medium | High coordination |
| Partnership | High | High collaboration |
| Customer/Supplier | Medium | Medium |
| Conformist | Low | Low |
| ACL | High | High translation |
| OHS | High | High API design |
| PL | High | High contract mgmt |
| Separate Ways | Low | Independent |

## Organizational Restructuring Signals

### When to Split Teams

- Shared Kernel growing too large
- Partnership causing delays
- Too many bounded contexts per team

### When to Merge Teams

- Frequent Partnership interactions
- Small Shared Kernel with little coordination
- Related contexts with aligned goals

### When to Create Platform Team

- Multiple OHS consumers
- Common infrastructure needs
- Reducing duplication across stream teams

## Anti-Patterns

| Anti-Pattern | Problem | Solution |
| ------------ | ------- | -------- |
| Single team, many contexts | Cognitive overload | Split by context boundary |
| Shared Kernel across teams | Coordination overhead | Convert to C/S or split team |
| Platform without consumers | Overhead without value | Merge into stream team |
| Conformist by default | Model pollution | Evaluate ACL value |

## Decision Template

When aligning teams with contexts:

```markdown
## Team-Context Alignment Decision

**Context:** [Context Name]
**Pattern:** [Mapping Pattern]
**Team Type:** [Stream/Platform/Enabling/Complicated]
**Interaction Mode:** [Collaboration/X-as-a-Service/Facilitating]

### Rationale
[Why this alignment]

### Cognitive Load Assessment
- Team capacity: [evaluation]
- Context complexity: [evaluation]
- Sustainable: [yes/no]

### Evolution Path
[How this might change]
```
