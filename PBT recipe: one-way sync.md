# Applicability

You want to synchronize some local state to a remote store.

You could synchronously go and update the remote state whenever the local state is updated, but this has some limitations:
- Changes to local state might be slow, as it contains a request-time callout to the remote store
- There is a pit-of-failure for concurrency issues, two concurrent updates to local state might clobber each other remotely
- Your remote store must have at least the capacity of your local store (there's no opportunity to throttle/batch changes)
- Your remote store must not be subject to any changes from any other actor
- Two local changes that cancel out must be sent as two seperate commands to the remote store

Given all these constraints, then chances are it's more viable to write a background worker that diffs the local state and remote state, and comes up with a patchset to apply to the remote store.

# Description

We want to test the function:

```typescript
type CalculateDifference = <LocalState, RemoteState, Patchset>(local: LocalState, remote: RemoteState) => Patchset
```

For the sake of simplicity, let's assume that our remote state is a list of immutable objects - we only care about adds/deletes to that list, not updates. The remote state must have some primary key that correlates remote state with local state.

For example, our remote state might be a list of all our registered users' emails. If a user activates their account, that causes an addition remotely. Deactivates = deletion. Changes email = addition + deletion.

# Generators

TODO: Specify common generators so they can be referenced in properties described below

# Properties

## It advises additions for all local objects that do not exist remotely

Simply adding something locally should replicate that change remotely.

### Generate

A local state and a remote state. The set of local primary keys must be distinct from the set of remote primary keys. This can be achieved by making one generator depend on the other, or simply by giving the ID generator sufficient entropy (e.g. it's a guid generator).

### Test

The patchset contains addition instructions forall objects in the local state. The patchset might contain other instructions, but the test should filter those out and just observe the create obstructions.

### Alternative

Don't generate a remote state, just use the empty local state for the test (e.g. an empty list). That way, we would expect all instructions to be addition instructions. However, the small amount of complexity for generating an arbitrary remote state and filtering only the addition instructions is probably worth the extra coverage over feature interaction bugs.

## It advises deletions for all remote objects that do not exist locally

Simply deleting something locally should replicate that change remotely.

This is mostly the same as the addition case described above, but instead of testing creates, we're testing deletes.

### Generate

A local state and a remote state. The set of local primary keys must be distinct from the set of remote primary keys. This can be achieved by making one generator depend on the other, or simply by giving the ID generator sufficient entropy (e.g. it's a guid generator).

### Test

The patchset contains delete instructions forall objects in the remote state. The patchset might contain other instructions, but the test should filter those out and just observe the delete obstructions.

### Alternative

Don't generate a local state, just use the empty remote state for the test (e.g. an empty list). That way, we would expect all instructions to be deletion instructions. However, the small amount of complexity for generating an arbitrary local state and filtering only the delete instructions is probably worth the extra coverage over feature interaction bugs.

## It advises deletions for duplicate remote objects

Your process might need to be resilient in resolving remote duplicates. If the remote store actually has a unique constraint on the primary key, then this is not necessary. If the remote store is pretty dumb, then the extra effort required to handle duplicates is probably worth the future operational concerns saved. Besides, the function can already advise delete instructions, and the process can already execute those instructions. 

The remote state which might contain duplicates must have a local representation, otherwise deleting duplicates might not be discrimateable from standard deletions because the local doesn't exist.

### Generate

1. One local object
2. For that local object, a remote state which contains `n` remote objects that represent the local object (`n >= 1`)

### Test

The patchest contains delete instructions for `n - 1` remote objects. That is, all but one of the duplicates are deleted.

### Alternatives

There are many different approaches for testing this behaviour. Some might empower the test scenario of many local objects, many remote objects, and one or many duplicates of one or many local objects. I found the complexity required in this test was not worth the more sophisticated examples that might provide better test coverage.

## It does not advise changes when local and remote state are practically the same

There might be many cases where local state and remote state are strictly different, but practically the same.

In this example of a list of objects, those objects might appear in a different order between different calls, but the results should be the same. Therefore, our one property to implement this requirement for this example might be:

**It is resilient to order of local and remote objects**

One scenario in which this is very important is deleting duplicates. We should always be advising deleting the same duplicates. Else, two instances of this process might treat the opposite object as the duplicate, depending on an indeterministic order.

### Generate

1. A local state
2. A remote state

### Test

1. Get the patchset for those states
2. Get the patchset for the reverse of those states
3. Assert those patchsets are structurally equal

### Alternatives

If your generation framework has a shuffle generator built-in, use this for more sophisticated examples (rather than simply reversing the list in the test).
