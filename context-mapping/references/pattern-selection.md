# Context Mapping Pattern Selection Guide

Decision framework for choosing the right context mapping pattern based on team dynamics, technical requirements, and organizational context.

## Decision Flow

### Step 1: Do the Contexts Share Code?

```text
Do the contexts share actual code (libraries, packages)?
├── Yes → Consider Shared Kernel
│   └── Is coordination manageable?
│       ├── Yes → Shared Kernel [SK]
│       └── No → Separate or use ACL
└── No → Continue to Step 2
```

### Step 2: Is There Data/Service Flow?

```text
Does data or service calls flow between contexts?
├── No → Do teams need to collaborate?
│   ├── Yes → Partnership [P]
│   └── No → Separate Ways [SW]
└── Yes → Continue to Step 3
```

### Step 3: Determine Direction

```text
Which context provides and which consumes?
├── Neither (bidirectional) → Partnership [P]
└── Clear direction exists → Identify Upstream/Downstream
    └── Continue to Step 4
```

### Step 4: Assess Downstream Influence

```text
Can downstream team influence upstream API/model?
├── No (external, legacy, or no leverage)
│   └── Does upstream model fit downstream needs?
│       ├── Yes → Conformist [CF]
│       └── No → Anti-Corruption Layer [ACL]
└── Yes → Customer/Supplier [C/S]
    └── Continue to Step 5
```

### Step 5: Assess Upstream API Strategy

```text
Does upstream serve multiple consumers?
├── Yes → Open Host Service [OHS]
│   └── Need formal contract?
│       └── Yes → Add Published Language [PL]
└── No → Basic Customer/Supplier sufficient
```

## Quick Selection Matrix

| Scenario | Pattern | Rationale |
| -------- | ------- | --------- |
| Same team, shared entities | Shared Kernel | Minimize duplication |
| Two teams co-developing feature | Partnership | Equal collaboration |
| Internal service dependency | Customer/Supplier | Negotiated API |
| Third-party API integration | ACL | Protect domain |
| Vendor API with good design | Conformist | Embrace their model |
| Platform team serving others | OHS + PL | Standardized access |
| No meaningful dependency | Separate Ways | Avoid coupling |

## Factor Analysis

### Team Dynamics Factors

| Factor | Low | High |
| ------ | --- | ---- |
| **Collaboration frequency** | Separate Ways, ACL | SK, Partnership |
| **Trust level** | ACL, Conformist | SK, Partnership, C/S |
| **Downstream leverage** | Conformist, ACL | C/S, Partnership |
| **Coordination capacity** | Separate Ways | SK, Partnership |

### Technical Factors

| Factor | Favors | Avoids |
| ------ | ------ | ------ |
| **Model compatibility** | Conformist, SK | ACL |
| **API stability** | Conformist, OHS | Tight coupling |
| **Performance critical** | Direct integration | Heavy translation |
| **Multiple consumers** | OHS, PL | Point-to-point |

### Organizational Factors

| Factor | Pattern Guidance |
| ------ | ---------------- |
| **Same team** | SK or no boundary |
| **Same org, different teams** | C/S, Partnership |
| **Different orgs** | OHS, PL, ACL |
| **Vendor/external** | ACL or Conformist |

## Pattern Trade-off Analysis

### Coupling vs Autonomy

```text
High Coupling ◄────────────────────────────► High Autonomy

Shared Kernel → Partnership → C/S → Conformist → ACL → Separate Ways
```

### Integration Effort vs Protection

```text
Low Effort ◄────────────────────────────► High Protection

Conformist → C/S → SK → Partnership → OHS → ACL
```

### Maintenance Cost

| Pattern | Initial Cost | Ongoing Cost |
| ------- | ------------ | ------------ |
| Shared Kernel | Low | Medium-High |
| Partnership | Medium | Medium |
| Customer/Supplier | Medium | Low-Medium |
| Conformist | Low | Low |
| ACL | High | Medium |
| OHS | High | Medium-High |
| PL | High | Medium |
| Separate Ways | None | None |

## Anti-Patterns to Avoid

### Wrong Pattern Selection

| Situation | Wrong Choice | Why It Fails | Better Choice |
| --------- | ------------ | ------------ | ------------- |
| External API | Conformist | Model leaks, breaks on change | ACL |
| Unstable upstream | Partnership | Assumes stability | C/S or ACL |
| No real dependency | SK | Artificial coupling | Separate Ways |
| Multiple consumers | Point-to-point C/S | Doesn't scale | OHS |

### Pattern Smell Indicators

**Shared Kernel Growing:**

- More than 5 entities in kernel
- Multiple teams blocked by kernel changes
- Kernel has its own release cycle

**Solution:** Split kernel or convert to C/S

**Partnership Without Collaboration:**

- One team always waiting on other
- Decisions made unilaterally
- No joint planning sessions

**Solution:** Convert to C/S with clear upstream

**ACL Without Value:**

- Translation is 1:1 mapping
- No domain concepts differ
- ACL code is trivial

**Solution:** Consider Conformist instead

## Decision Record Template

When selecting a pattern, document:

```markdown
## Context Relationship Decision

**Contexts:** ContextA → ContextB
**Pattern Selected:** [Pattern]
**Date:** YYYY-MM-DD

### Factors Considered
- Team dynamics: [description]
- Technical requirements: [description]
- Organizational context: [description]

### Alternatives Rejected
- [Pattern]: [reason not chosen]

### Trade-offs Accepted
- [Trade-off 1]
- [Trade-off 2]

### Review Triggers
- If [condition], reconsider pattern
```

## Evolution Paths

Patterns can evolve as circumstances change:

```text
Conformist ────────► ACL
(when model friction increases)

Partnership ────────► Customer/Supplier
(when direction clarifies)

Customer/Supplier ──► Open Host Service
(when more consumers appear)

Shared Kernel ──────► Customer/Supplier
(when kernel becomes burden)

Separate Ways ──────► Any Pattern
(when integration need emerges)
```

## Red Flags Checklist

Before finalizing pattern selection:

- [ ] Have you identified clear upstream/downstream (if asymmetric)?
- [ ] Have you assessed team leverage and collaboration capacity?
- [ ] Have you considered model compatibility?
- [ ] Have you evaluated future evolution needs?
- [ ] Have you documented the decision rationale?
- [ ] Have you identified review triggers?
