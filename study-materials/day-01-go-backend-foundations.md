# Day 1: Go Backend Foundations

This lesson is the detailed companion to Day 1 of the
[backend learning roadmap](../BACKEND_LEARNING_ROADMAP.md#day-1-go-as-a-backend-language).
It is self-contained: you can complete the required lesson without searching for
another tutorial. External references are optional sources for deeper study.

## Learning outcomes

By the end of this lesson, you should be able to:

- create and navigate a Go module;
- explain packages, exported names, zero values, structs, and methods;
- choose deliberately between values and pointers;
- recognize when slices and maps share mutable storage;
- define a small interface at the point where it is consumed;
- create, wrap, inspect, and preserve errors;
- use `defer` to release an acquired resource;
- pass cancellation and deadlines through `context.Context`;
- protect shared state with a mutex;
- distinguish data-race freedom from business correctness;
- implement and test a concurrency-safe in-memory product store.

Budget about three focused hours. Type the examples instead of copying them when
possible. The small mistakes you make while typing are useful practice with the
compiler and test output.

## 1. Modules, packages, files, and names

A **module** is the versioned unit of Go source code. Its root contains `go.mod`,
which declares the module path and the Go language version. A module contains one
or more **packages**. A package is a group of `.go` files in the same directory
that declare the same package name.

Initialize this repository from its root:

```bash
go mod init be-standard
```

For this local learning project, `be-standard` is a sufficient module path. If the
repository is later published, it can use its canonical repository path.

Create application packages below `internal/`. Go's `internal` rule prevents code
outside the parent module tree from importing them. That makes the intended API
boundary explicit without a framework.

```text
internal/
`-- product/
    |-- store.go
    `-- store_test.go
```

Every Go file begins with a package declaration:

```go
package product
```

Names beginning with an uppercase letter are exported from their package. Names
beginning with a lowercase letter are package-private.

```go
type Product struct { // Product is exported.
    SKU  string       // SKU is exported.
    name string       // name is visible only inside package product.
}
```

Export only what a caller needs. A smaller package surface leaves fewer behaviors
that must remain compatible.

## 2. Zero values, structs, and methods

Every Go variable has a value immediately. If no value is assigned explicitly,
Go uses the type's **zero value**.

| Type | Zero value |
| --- | --- |
| `bool` | `false` |
| integers and floats | `0` |
| `string` | `""` |
| pointer, slice, map, function, interface, channel | `nil` |
| struct | each field's zero value |

Good Go types are useful in their zero state when that is practical. For example,
`sync.Mutex` is ready to use without a constructor. A map is different: reading a
nil map is safe, but writing to it panics, so a type that owns a map normally needs
a constructor.

```go
type Product struct {
    ID   string
    SKU  string
    Name string
}

type Store struct {
    products map[string]Product
}

func NewStore() *Store {
    return &Store{products: make(map[string]Product)}
}
```

A method is a function with a receiver:

```go
func (p Product) DisplayName() string {
    return p.SKU + " - " + p.Name
}
```

The receiver appears between `func` and the method name. A value receiver receives
a copy of the value. A pointer receiver receives an address through which the
method can modify the original value.

## 3. Values, pointers, slices, and maps

### Choose values by default for small data

Use a value when:

- the data is small and copying it is clear;
- the function should not mutate the caller's value;
- absence is represented separately, often by an error or boolean;
- value semantics make the API easier to reason about.

```go
func Rename(p Product, name string) Product {
    p.Name = name
    return p
}
```

Use a pointer when:

- the function or method must mutate the original value;
- copying would be expensive or incorrect, such as copying a type containing a
  mutex;
- identity matters independently of field equality;
- `nil` has a deliberate, documented meaning.

```go
func (p *Product) Rename(name string) {
    p.Name = name
}
```

Do not use pointers automatically merely because the code is in a backend. Values
often make ownership and mutation much clearer.

### A slice is a view over an array

A slice contains metadata pointing to an underlying array. Assigning or passing a
slice copies the metadata, not necessarily the elements. Two slices can therefore
observe mutations to the same storage.

```go
names := []string{"tea", "coffee"}
copyOfHeader := names
copyOfHeader[0] = "water"
// names[0] is now "water" too.
```

Make an independent copy at an ownership boundary:

```go
independent := append([]string(nil), names...)
```

### A map is shared mutable state

Assigning a map does not copy its entries. Maps are not safe for concurrent writes,
or for a read concurrent with a write. A store that owns a map must prevent callers
from accessing that map directly and synchronize all access.

Returning a new slice from `List` protects the store's collection from callers.
Because the exercise's `Product` contains only strings, copying each `Product`
value is enough. If it later contains slices, maps, or pointers, decide whether a
deeper copy is required.

## 4. Interfaces describe required behavior

An interface is a set of method signatures. A type implements an interface
implicitly by having those methods; there is no `implements` declaration.

```go
type ProductReader interface {
    Get(sku string) (Product, error)
}

func PrintProduct(reader ProductReader, sku string) error {
    product, err := reader.Get(sku)
    if err != nil {
        return err
    }

    fmt.Println(product.DisplayName())
    return nil
}
```

The code that needs to read a product should declare the smallest interface it
needs. This is called defining the interface on the consumer side. It avoids a
large provider-owned interface that forces every consumer and test double to
depend on unrelated methods.

Do not introduce an interface just because one implementation exists. Introduce
one where a caller needs substitution, isolation in a test, or a real boundary.

You can ask the compiler to verify an implementation:

```go
var _ ProductReader = (*Store)(nil)
```

This declaration allocates nothing. It fails compilation if `*Store` does not
implement `ProductReader`.

## 5. Errors are values with identity

Go functions commonly return a useful result plus an error:

```go
product, err := store.Get("TEA-001")
if err != nil {
    // Handle or return the error.
}
```

Use sentinel errors when callers need to distinguish a stable condition:

```go
var (
    ErrDuplicateSKU = errors.New("duplicate SKU")
    ErrNotFound     = errors.New("product not found")
)
```

Add context while preserving the original error with `%w`:

```go
return Product{}, fmt.Errorf("get product %q: %w", sku, ErrNotFound)
```

The wrapped message is useful to a human, while `errors.Is` follows the wrapping
chain and retains machine-checkable identity:

```go
if errors.Is(err, ErrNotFound) {
    // Map this condition deliberately, such as to HTTP 404 later.
}
```

Use `errors.As` when you need a particular error type and its fields:

```go
var validationErr *ValidationError
if errors.As(err, &validationErr) {
    fmt.Println(validationErr.Field)
}
```

Avoid comparing error text. Messages are for people and may change; identities and
types are contracts. Handle an error once: either recover from it meaningfully or
return it with useful context.

## 6. Resource ownership and `defer`

Whoever successfully acquires a resource is normally responsible for releasing
it. Place the `defer` immediately after confirming acquisition succeeded:

```go
file, err := os.Open("products.json")
if err != nil {
    return err
}
defer file.Close()
```

Deferred calls run when the surrounding function returns, in last-in-first-out
order. Typical backend uses include closing response bodies and query rows,
rolling back a transaction unless it commits, unlocking a mutex, and stopping a
timer.

Do not put an unbounded number of defers inside a long-running loop: they remain
pending until the surrounding function returns. Extract one iteration into a
small function when each iteration owns a resource.

For resources where close errors matter, such as buffered writers, explicitly
check the close or flush error instead of discarding it.

## 7. Context carries cancellation across boundaries

`context.Context` communicates cancellation, deadlines, and request-scoped values
across API boundaries.

```go
func LoadProduct(ctx context.Context, store ProductReader, sku string) (Product, error) {
    select {
    case <-ctx.Done():
        return Product{}, ctx.Err()
    default:
        return store.Get(sku)
    }
}
```

Rules to follow:

- accept `context.Context` as the first parameter;
- do not store it in a struct;
- pass it to downstream calls that support cancellation;
- do not pass `nil`; use `context.Background()` when no parent exists;
- call the returned cancel function when creating a timeout or cancellation scope;
- use context values only for request-scoped metadata such as trace identity, not
  ordinary optional function parameters.

```go
ctx, cancel := context.WithTimeout(parent, 500*time.Millisecond)
defer cancel()
```

Cancellation is cooperative. Calling `cancel` closes `ctx.Done()`, but work stops
only if the receiving code observes the signal or calls an API that does.

The in-memory store does not perform blocking I/O, so its first version does not
need context parameters. PostgreSQL and HTTP operations added later will.

## 8. Concurrency: goroutines, synchronization, and correctness

A goroutine is a function executing concurrently with other goroutines:

```go
done := make(chan struct{})

go func() {
    defer close(done)
    // Perform bounded work.
}()

<-done
```

The channel gives the goroutine a termination signal that the caller can observe.
Every goroutine should have a clear owner, a stopping condition, and a way for its
completion or error to be observed. Starting a goroutine and forgetting it risks a
leak.

### Channels and mutexes solve different ownership problems

Use a channel when goroutines need to transfer values, signal completion, or hand
off ownership. Use a mutex when multiple goroutines need short, synchronized access
to shared in-memory state.

```go
type Store struct {
    mu       sync.RWMutex
    products map[string]Product
}
```

A read lock permits other readers. A write lock excludes readers and writers:

```go
s.mu.RLock()
defer s.mu.RUnlock()

s.mu.Lock()
defer s.mu.Unlock()
```

Keep critical sections small, but large enough to preserve the complete invariant.
For duplicate prevention, checking and inserting must occur under one write lock:

```go
s.mu.Lock()
defer s.mu.Unlock()

if _, exists := s.products[product.SKU]; exists {
    return ErrDuplicateSKU
}
s.products[product.SKU] = product
```

If the check and insertion used separate lock sections, two goroutines could both
observe that the SKU is absent before either inserts it.

### Atomics and happens-before

An atomic operation safely reads or writes one supported value without a data race.
Atomics are useful for simple counters and flags, but they do not automatically
protect a rule involving several values or steps. Prefer a mutex when expressing
the invariant with atomics would be difficult to explain.

The Go memory model defines **happens-before** relationships: synchronization such
as unlocking a mutex before another goroutine locks it makes preceding writes
visible to the later goroutine. Without synchronization, assumptions based on
source-code order or wall-clock timing are invalid.

### Data-race freedom is not business correctness

A data race is unsynchronized concurrent access to the same memory where at least
one access writes. The race detector can find many such execution paths.

A program can be race-free and still be wrong. Consider inventory stored behind
properly locked methods:

```go
if store.Quantity(sku) > 0 {
    store.Decrement(sku)
}
```

Each call may be individually race-free, but two callers can both observe quantity
`1`, then both decrement. The business operation requires one atomic check-and-
decrement operation. Synchronize the whole invariant, not merely individual field
accesses. Day 3 applies the same principle with database transactions and locks.

## 9. Guided exercise: in-memory product store

Create `internal/product/store_test.go` first. Tests define the behavior before the
implementation exists.

### Required public behavior

Use this product model:

```go
type Product struct {
    ID   string
    SKU  string
    Name string
}
```

Implement this store API:

```go
func NewStore() *Store
func (s *Store) Create(product Product) error
func (s *Store) Get(sku string) (Product, error)
func (s *Store) List() []Product
```

Required invariants:

1. Each SKU identifies at most one product.
2. `Create` returns an error matching `ErrDuplicateSKU` for an existing SKU.
3. `Get` returns an error matching `ErrNotFound` for an unknown SKU.
4. Concurrent calls must not race or corrupt the map.
5. `List` returns a newly allocated snapshot so callers cannot change the store's
   collection.
6. `List` uses a deterministic order so tests and later API behavior are stable.

For Day 1, order the list lexicographically by SKU using `sort.Slice` or
`slices.SortFunc`. Stable order matters because map iteration order is deliberately
unspecified.

### Step 1: write the basic tests

Write tests for:

- creating and retrieving one product;
- retrieving an unknown SKU;
- creating the same SKU twice;
- listing products in SKU order;
- changing the returned list without changing the next returned list.

A useful test shape is:

```go
func TestStoreCreateAndGet(t *testing.T) {
    store := NewStore()
    want := Product{ID: "product-1", SKU: "TEA-001", Name: "Green tea"}

    if err := store.Create(want); err != nil {
        t.Fatalf("Create() error = %v", err)
    }

    got, err := store.Get(want.SKU)
    if err != nil {
        t.Fatalf("Get() error = %v", err)
    }
    if got != want {
        t.Fatalf("Get() = %#v, want %#v", got, want)
    }
}
```

Run the test now. Compilation failure is expected because the implementation does
not exist yet:

```bash
go test ./internal/product
```

This is the red step of red-green-refactor: observe the test fail for the expected
reason, add the smallest correct implementation, and then improve structure while
the tests remain green.

### Step 2: implement the sequential behavior

Create `store.go` with the model, sentinel errors, `Store`, and its methods. Use a
map keyed by SKU. Add deterministic sorting in `List`.

Run:

```bash
go test ./internal/product
```

Do not continue until the basic tests pass.

### Step 3: add concurrent tests

Start a fixed number of goroutines. Each must signal completion, and the test must
wait for all of them. `sync.WaitGroup` is appropriate:

```go
func TestStoreConcurrentCreateAndGet(t *testing.T) {
    store := NewStore()
    const workers = 100

    var wg sync.WaitGroup
    wg.Add(workers)

    for i := 0; i < workers; i++ {
        i := i
        go func() {
            defer wg.Done()

            sku := fmt.Sprintf("SKU-%03d", i)
            product := Product{ID: sku, SKU: sku, Name: "Product " + sku}
            if err := store.Create(product); err != nil {
                t.Errorf("Create(%q) error = %v", sku, err)
                return
            }
            if _, err := store.Get(sku); err != nil {
                t.Errorf("Get(%q) error = %v", sku, err)
            }
        }()
    }

    wg.Wait()

    if got := len(store.List()); got != workers {
        t.Fatalf("List() length = %d, want %d", got, workers)
    }
}
```

The number of goroutines is explicitly bounded. Every goroutine executes
`wg.Done()` with `defer`, including early-return paths, so the test has a clear
termination condition.

Now run the race detector:

```bash
go test -race ./internal/product
```

If you initially omit the mutex, expect a failure similar to:

```text
WARNING: DATA RACE
Read at ...
Previous write at ...
```

Do not silence the test. Add one `sync.RWMutex` owned by `Store`, use its write lock
for the entire duplicate-check-and-insert operation, and use its read lock while
getting or copying products.

### Step 4: test a business-level concurrency rule

Have many goroutines attempt to create the same SKU. Exactly one call must succeed;
every other result must match `ErrDuplicateSKU`. Protect only the test's result
counters with a separate mutex or use a results channel.

This test proves more than the absence of a data race: it proves that the uniqueness
invariant remains true under contention.

### Step 5: complete the verification loop

Run the repository's Day 1 checks:

```bash
gofmt -w .
go test ./...
go test -race ./...
go vet ./...
```

Interpret them as different evidence:

- `gofmt` makes source formatting deterministic;
- `go test` checks the specified examples and invariants;
- `go test -race` instruments memory access to detect exercised data races;
- `go vet` detects suspicious constructs that compile but are commonly erroneous.

Passing these commands does not prove the program is bug-free. It does mean the
current implementation passed four useful, repeatable checks.

## 10. Optional stretch: fuzz a validator

After the required work, write a small SKU parser or validator and fuzz it. A fuzz
test supplies generated inputs to find panics and invariant violations.

For example, define a rule that a SKU is 3-32 ASCII uppercase letters, digits, or
hyphens, starts with a letter, and ends with a letter or digit. Seed the fuzz test
with valid and invalid examples, then assert only properties that must always hold:
the validator never panics, accepted input respects every rule, and normalization
does not change already valid input.

Run:

```bash
go test -fuzz=FuzzValidateSKU -fuzztime=20s ./internal/product
```

Keep fuzzing optional on the first pass. The product store and required verification
commands are the priority.

## 11. Knowledge check

Answer these without looking back:

1. What is the difference between a module and a package?
2. Why can a nil map be read but not written?
3. When is a pointer receiver required?
4. Why can modifying one slice affect another slice?
5. How does a type declare that it implements an interface?
6. Why should the consumer usually own an interface?
7. What information does `%w` preserve?
8. When should `errors.Is` be used instead of comparing error messages?
9. Where should `defer` appear after resource acquisition?
10. Why does calling a context cancel function not forcibly stop a goroutine?
11. What must every goroutine have to avoid becoming an unowned leak?
12. Why can code pass the race detector and still violate an inventory rule?

### Expected answers

1. A module is the versioned source unit declared by `go.mod`; a package is a
   directory-level compilation and API unit within a module.
2. A nil map has no allocated table. Lookup is defined to return the zero value,
   but assignment has nowhere to store an entry and panics.
3. Use one when the method must mutate the original, copying is unsafe or costly,
   identity matters, or nil is intentional.
4. A slice copies a descriptor that can still point to the same underlying array.
5. It does not declare implementation; its method set satisfies the interface
   implicitly.
6. The consumer can describe only the behavior it needs, keeping coupling small.
7. It adds context while preserving the wrapped error in the error chain.
8. Use `errors.Is` when behavior depends on a stable error identity.
9. Immediately after checking that acquisition succeeded.
10. Cancellation is cooperative; the goroutine or downstream API must observe it.
11. An owner, a bounded or cancellable stopping condition, and observable completion
    or error handling.
12. The detector finds memory races on executed paths, not multi-step business
    invariants such as atomic check-and-decrement.

## 12. Completion evidence

Day 1 is complete when all of the following are true:

- the four verification commands pass;
- the duplicate-SKU contention test proves exactly one success;
- every started goroutine has a bounded count and a completion path;
- you can explain value versus pointer semantics without relying on "performance" as
  the only reason;
- you can explain implicit interface satisfaction;
- you can demonstrate that `errors.Is` recognizes a sentinel through a `%w` wrapper;
- you record one mistake, why it occurred, and the test or design rule that now
  prevents it.

Then continue to Day 2. Do not add HTTP handlers during this lesson; keeping the
exercise in memory makes the language, ownership, errors, and concurrency behavior
easier to see.
