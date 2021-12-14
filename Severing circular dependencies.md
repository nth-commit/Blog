# Severing Circular Dependencies

The simplest of circular dependencies is when `Module A` imports from `Module B`, and `Module B` imports from `Module A`. If you draw a graph of these modules, there is obviously a cycle.

This makes it really really hard for your string-concatenator to figure out in which order to load these things. We use Webpack, it doesn't strictly fall over on circular dependencies - which is not a desirable behavior, as you can be building a debt of circular dependencies without being aware, and as soon as you delete some innocuous-looking module import - the whole system breaks (because that import was forcing some load order that happened to resolve things in the right order).

I recently exercised a TypeScript/React project of all it's circular dependency demons. Here's a quick runbook on how you can do the same.
 
## 1. Analyze

Firstly, I installed the webpack plugin https://github.com/aackerman/circular-dependency-plugin into our build. Initially, for informational purposes. Eventually, I'd hope we'd be able to run it as part of the build process, and prevent circular dependencies from emerging in the future. This was obviously predicated on being able to eliminate the existing circular dependencies entirely.

![image](https://user-images.githubusercontent.com/6957926/146087141-91e8851f-92ec-4ab0-8faf-35daf2ada949.png)

### Metric 1: Number of errors

Our first run produced **74 errors**.

Note, the error count is not entirely indicative of the amount of work required to resolve. Unfortunately, some circular dependencies might be obscuring others (fix one, another pops up). Fortunately, severing a single dependency might resolve a bunch of errors. On that note;

### Metric 2: Distribution of cycle size

Simply put, circular dependencies over many modules are much harder to resolve than circular dependencies over only a few modules. This is because there are many more possible (not necessarily viable) solutions, and it's a much bigger cognitive load to unpick them.

If you have a lot of errors, you may want to parse and analyze this metric programmatically. Else just give it a good 'ol eyeball.

### Metric 3: Number of common edges

If your errors are repeatedly reporting the same edge, that might be a good place to start - independent of how large the cycle is.

![image](https://user-images.githubusercontent.com/6957926/146088297-4d64e33e-9c7b-46ba-93a9-8c1ecab9f14c.png)

In this case, the dependency `src\tools\feature.ts -> src\features\create.ts` was appearing many times. Severing that dependency would involve touching the least amount of files, least likely to cause you to want to flip the table and rearchitect everything from the ground up.

As above, if you have a lot of errors, it might be useful to detect the most common edges programmatically. Otherwise eyeballs.

## 2. Fix

### Pitfall 1: Avoid redesign

A lot of the fixes in this section involve a sleight of hand that might want to cause you to name an abstraction. If you have a decent amount of errors (e.g. 74), don't bother or we'll be here all week. We just want to get our project building and running again. Redesign later.

### Pitfall 2: One fell swoop

For one circular dependency, you might make a lot of mistakes that you'll want to quickly undo. Don't let the number of changes in your working directory stack up too much. Commit each circular dependency resolution as you go - even if they don't immediately resolve the root issue, one less circular dependency improves the system.

### Fix 1: Extract shared module

Given the modules:

```typescript
/// ClassA.ts
import ClassB from './ClassB';

export const NUMBER_OF_WONKIES = 5;

export default class ClassA { }
```

```typescript
/// ClassB.ts
import { NUMBER_OF_WONKIES } from './ClassA';

export default class ClassB { }
```

A simple fix is to extract the constant `NUMBER_OF_WONKIES` to another module (say `Constants.ts`) and import it from `ClassA.ts` and `ClassB.ts`.

### Fix 2: Inversion of control

The more you have modules being passed their runtime dependencies, rather than statically "reaching out" for them (i.e. by import), the more flexible they are and the more simple they are to understand. Building out the previous example further:

```typescript
/// ClassA.ts
import ClassB from './ClassB';

export const NUMBER_OF_WONKIES = 5;

export default class ClassA {
  constructor() {
    this.b = new ClassB();
  }
  
  calculateWonkyCost: (wonkyPrice: number) => wonkyPrice * NUMBER_OF_WONKIES;
}
```

```typescript
/// ClassB.ts
import ClassA, { NUMBER_OF_WONKIES } from './ClassA';

export default class ClassB {
  calculateWonkyCost: (wonkyPrice: number) => wonkyPrice * NUMBER_OF_WONKIES;
}
```

Both classes use the constant in their bodies. The dependency hierarchy is obviously `A -> B`, given `ClassA` create instances of `ClassB`. It is preferable to inject the dependency from `ClassA` to `ClassB` and minimize the amount of times `NUMBER_OF_WONKIES` is accessed statically i.e.:

```typescript
/// ClassA.ts
import ClassB from './ClassB';

export const NUMBER_OF_WONKIES = 5;

export default class ClassA {
  constructor() {
    this.b = new ClassB(NUMBER_OF_WONKIES);
  }
  
  calculateWonkyCost: (wonkyPrice: number) => wonkyPrice * NUMBER_OF_WONKIES;
}
```

```typescript
/// ClassB.ts
export default class ClassB {
  constructor(private readonly numberOfWonkies: number) { }

  calculateWonkyCost: (wonkyPrice: number) => wonkyPrice * this.numberOfWonkies;
}
```

### Fix 3: Synthetic imports

Sometimes you just need the type. There shouldn't be a runtime requirement to load the module if it's not actually required at runtime - it should not affect the load order. TypeScript gives you the language to express this with `import type`. Checkout this circular reference:

```typescript
/// Parent.ts
import Child from './Child';

export default class Parent {
  private readonly children: Child[] = [];

  addChild(): void {
    this.children.push(new Child(this));
  }
}
```

```typescript
/// Child.ts
import Parent from './Parent';

export default class Child {
  constructor(private readonly parent: Parent) { }
  
  getParent(): Parent {
    return this.parent;
  }
}
```

`Child` doesn't actually need `Parent` at runtime, it's just passing it through ignorantly. If you think about how this runs in (untyped) Javascript the `parent` parameter may be passed literally anything, and the child only needs to echo what it's passed - in fact the `Child` class could easily be generic over `Parent`. This is actually a fairly common source of circular dependencies when dealing with classes.

The fix is simply augmenting the `Child` module:

```typescript
/// Child.ts
import type Parent from './Parent';
```

### Fix 4: Eliminate constructs that require runtime checks

Sometimes, synthetic imports might not be possible.

TL;DR: Use discriminating tags > `instanceof`, use string unions > 'enum'.

**MORE AT 6**
