# Entity Schema — Bidirectional Relationships

This describes the `@data-client/rest` Entity schema definitions that cause the stack overflow during denormalization when all entities are in the cache store.

## Entity Types

| Entity | JSON:API type | Role |
|--------|--------------|------|
| Building | `buildings` | Primary entity (~3,000) |
| Department | `departments` | Primary entity (~3,000) |
| Room | `rooms` | Join table linking Building ↔ Department (~6,000) |
| Document | `documents` | Associated data (~9,000) |

## Schema Definitions

```typescript
import { Entity } from '@data-client/rest';

class Building extends Entity {
  // ... attributes ...

  static key = 'Building';
  static get schema() {
    return {
      departments: [Department],        // → Department[]
      rooms: [Room],                    // → Room[]
      documents: [Document],            // → Document[]
    };
  }
}

class Department extends Entity {
  // ... attributes ...

  static key = 'Department';
  static get schema() {
    return {
      buildings: [Building],            // → Building[]  ← BACK-REFERENCE
      parent: Department,               // → Department (self-referential)
      children: [Department],           // → Department[] (self-referential)
      operationalLead: Department,      // → Department (self-referential)
      regionalLead: Department,         // → Department (self-referential)
      rooms: [Room],                    // → Room[]
      documents: [Document],            // → Document[]
    };
  }
}

class Room extends Entity {
  // ... attributes ...

  static key = 'Room';
  static get schema() {
    return {
      building: Building,               // → Building  ← BACK-REFERENCE
      department: Department,            // → Department ← BACK-REFERENCE
      documents: [Document],            // → Document[]
    };
  }
}

class Document extends Entity {
  // ... attributes ...

  static key = 'Document';
  static get schema() {
    return {
      buildings: [Building],            // → Building[]  ← BACK-REFERENCE
      departments: [Department],        // → Department[] ← BACK-REFERENCE
      rooms: [Room],                    // → Room[]  ← BACK-REFERENCE
    };
  }
}
```

## Bidirectional Cycles

The schemas create these cycles during denormalization:

```
Building → departments → Department → buildings → Building → ...
Building → rooms → Room → building → Building → ...
Department → rooms → Room → department → Department → ...
Building → documents → Document → buildings → Building → ...
```

With ~20,000 unique entities in the store (all loaded via progressive pagination), each entity hop visits a **different pk**, so `GlobalCache`'s same-pk cycle detection does not prevent the recursion. The denormalization traverses the full bidirectional graph until the call stack overflows.

## Data Volume

Each JSON file represents one page of 500 primary entities with included relationships:

- `buildings_p[1-6].json` — 6 pages × 500 buildings + ~3,300 included entities each
- `departments_p[1-6].json` — 6 pages × 500 departments + ~4,100 included entities each

Total: **~20,850 unique entities** across all pages.

## How to Reproduce

1. Load all `buildings_p*.json` pages into the data-client store (simulating progressive pagination)
2. Load all `departments_p*.json` pages into the store
3. Trigger denormalization of any building or department entity
4. Stack overflow occurs as denormalization traverses bidirectional relationships across ~20k entities
