# Resolver Interface Redesign

**Status:** Proposal
**Scope:** `pkg/resolver` (CEL, Unstructured, Starlark) and `internal/family.go`

## 1. Summary

The current `Resolver` interface has two kinds of problems: (1) it can only
return flat `map[string]string` data, so lists and multi-sample output get
smuggled through string tricks in map keys, and sanitization rules live
outside the interface as an unwritten convention; and (2) Starlark doesn't
fit the interface at all, so it's bolted on as a separate struct field with
its own duplicated code path instead of being a real resolver.

This document proposes a `Result` type that returns data as what it actually
is (scalar, list, or map) instead of encoding it into strings, plus a small
`FamilyGenerator` interface that lets Starlark — and any future
whole-family-at-once resolver — plug into the same dispatch and the same
sample-writing code as everyone else, instead of needing its own branch and
its own copy of the logic.

## 2. Problems with the current design

### Problem 1 — Starlark isn't really a `Resolver`

`FamilyType` has a dedicated field, separate from the generic
`resolver.Resolver` interface:

```go
starlarkResolver *resolver.StarlarkResolver
```

`buildMetricString` checks this field directly before doing anything else:

```go
switch {
case f.starlarkResolver != nil:
    metricStr, sampleCount = f.buildMetricStringFromStarlark(unstructured)
default:
    // generic path
}
```

And the one factory function that is supposed to build *any* resolver
explicitly refuses to build Starlark:

```go
case v1alpha1.ResolverTypeStarlark:
    return nil, fmt.Errorf("starlark resolver requires starlark config...")
```

In plain terms: Starlark is treated as a completely different kind of thing
from CEL and Unstructured, even though conceptually it's still "a way to turn
a Kubernetes object into metric labels and values." Nothing in the type
system says "Starlark belongs to this family" — a new engineer reading
`Resolver` would never guess Starlark exists.

### Problem 2 — Starlark duplicates the sample-writing logic

Because Starlark is special-cased, it can't reuse the pipeline the other two
resolvers already go through:

```
resolveLabels → sanitizeKey → sortLabels → writeMetricSamplesWithCount
                                              (handles list expansion)
```

`buildMetricStringFromStarlark` instead does its own smaller version of this
inline: its own loop, its own call to `sanitizeKey`, its own call to
`sortLabels`, and it calls `writeMetricTo` directly instead of
`writeMetricSamplesWithCount`. Two consequences:

- Starlark samples never go through the list-expansion machinery
  (`resolvedExpandedLabelSet`, `expandedValueSentinel`) even though a real
  scripting language could plausibly need it.
- Every future bug fix to "how we write a sample" has to be made in two
  places by hand, and it's easy to fix one and forget the other.

### Problem 3 — `map[string]string` forces composite data into string tricks

`Resolve(query, obj) map[string]string` can only return flat scalars. So when
a query resolves to a *list*, the code fakes it by writing synthetic keys
with `#0`, `#1`, ... suffixes (`collectIndexedResolvedValues`,
`listIndexRegex`), and stashes the corresponding metric values under a
NUL-byte sentinel key (`expandedValueSentinel`) — because a NUL byte can
never appear in a real Prometheus label name, so it's "safe" to use as a
marker. The code's own comment admits this is not clean:

> "we do not want resolver-specific logic making its way into
> non-resolver-specific code, however, this is general enough that it can be
> reasonably justified as an implementation detail"

In plain terms: the interface's return type can't say "this is a list," so
the caller has to guess it from the *shape of the map keys* instead of being
told directly.

### Problem 4 — Sanitization is a floating convention, not part of the contract

`sanitizeKey` is called twice, independently, in two different places:

- once inside `resolveLabels` for the generic path
- once inside `buildMetricStringFromStarlark`

It isn't part of the `Resolver` interface at all. A person writing a new
resolver has no way of knowing this step is expected, and no way to opt out
or ask for different sanitization rules — it "just works" today only because
both existing call sites happen to remember to call it.

## 3. Proposed design

### 3.1 A `Result` type that says what shape it is

Instead of always returning `map[string]string`, a resolver returns a small
tagged value that says up front whether it's a single string, a list of
strings, or a real key→value map:

```go
package resolver

type Kind int

const (
    KindScalar Kind = iota
    KindList
    KindMap
)

// Result is what a Resolver returns for one query. Exactly one of the
// accessor methods below is meaningful, based on Kind().
type Result struct {
    kind   Kind
    scalar string
    list   []string
    mmap   map[string]string
}

func NewScalarResult(s string) Result { return Result{kind: KindScalar, scalar: s} }
func NewListResult(l []string) Result { return Result{kind: KindList, list: l} }
func NewMapResult(m map[string]string) Result { return Result{kind: KindMap, mmap: m} }

func (r Result) Kind() Kind            { return r.kind }
func (r Result) Scalar() string        { return r.scalar }
func (r Result) List() []string        { return r.list }
func (r Result) Map() map[string]string { return r.mmap }

// Empty reports whether the query produced no usable data (e.g. an empty
// list from a guarded expression). Callers use this instead of checking
// len() on a map.
func (r Result) Empty() bool {
    switch r.kind {
    case KindScalar:
        return r.scalar == ""
    case KindList:
        return len(r.list) == 0
    case KindMap:
        return len(r.mmap) == 0
    }
    return true
}
```

Note `List()` is a real `[]string`, in order — not a map with `#0`/`#1` keys.
That's the piece that removes the string-trick encoding.

### 3.2 The `Resolver` interface, unchanged in spirit

```go
type Capabilities interface {
    HasTimeout() bool
    UnderscoreExpansionSupported() bool
    IsKeySanitize() bool
}

type Resolver interface {
    Capabilities
    Resolve(query string, obj map[string]any) (Result, error)
}
```

CEL and Unstructured implement this exactly as before, just returning a
`Result` instead of a `map[string]string`.

### 3.3 An optional interface for resolvers that generate whole families

Starlark doesn't answer one query at a time — it turns a whole object into a
batch of pre-labeled samples in one call. Rather than force it into
`Resolve(query, obj)` with a meaningless `query` argument, it gets its own
small interface:

```go
// FamilyGenerator is implemented by resolvers that produce complete,
// already-labeled metric samples directly, instead of answering one
// label/value query at a time.
type FamilyGenerator interface {
    Capabilities
    GenerateFamilies(obj map[string]any) ([]ResolvedFamily, error)
}
```

`StarlarkResolver` implements `FamilyGenerator`, not `Resolver`. It is no
longer excluded from the resolver factory — it's just a different kind of
resolver, and the factory can construct it like any other:

```go
func (f *FamilyType) resolver(t v1alpha1.ResolverType) (any, error) {
    switch t {
    case v1alpha1.ResolverTypeUnstructured:
        return resolver.NewUnstructuredResolver(f.logger), nil
    case v1alpha1.ResolverTypeCEL:
        return resolver.NewCELResolver(...), nil
    case v1alpha1.ResolverTypeStarlark:
        return resolver.NewStarlarkResolver(...), nil // no longer an error
    default:
        return nil, fmt.Errorf("unknown resolver %q", t)
    }
}
```

`buildMetricString` then dispatches on what the constructed value actually
implements, instead of on a struct field that only `FamilyType` knows about:

```go
resolverInstance, err := f.resolver(metric.Resolver)
if err != nil {
    logger.V(1).Error(err, "skipping")
    return "", 0
}

switch r := resolverInstance.(type) {
case resolver.FamilyGenerator:
    families, err := r.GenerateFamilies(unstructured.Object)
    // convert each sample into (keys, values, value string) and feed it
    // into the SAME writeMetricSamplesWithCount used below — see 3.4
case resolver.Resolver:
    // existing generic per-label path, unchanged
}
```

### 3.4 One sample writer for everyone

The fix for duplicated sample-writing isn't a type change — it's making sure
`FamilyGenerator` output goes through the *same* function as everything
else. Each sample coming out of `GenerateFamilies` gets converted into the
same `(labelKeys []string, labelValues []string, value string)` shape the
generic path already builds, and handed to the existing
`writeMetricSamplesWithCount`. No second implementation of "how do we write
one Prometheus sample line" exists anywhere in the codebase after this
change.

### 3.5 Sanitization becomes part of the contract

`IsKeySanitize()` on `Capabilities` is checked once, in one place — inside
the shared label-processing code — rather than being called independently
from two different functions:

```go
if resolverInstance.IsKeySanitize() {
    key = metricutil.SanitizeKey(key)
}
```

`metricutil.SanitizeKey` also replaces the two separate implementations that
exist today (`family.go`'s `sanitizeKey` using `strcase.ToSnake`, and
`cel.go`'s `metricutil.SanitizeLabelKey` used inside `labelPrefixBinding`) —
one function, in the shared `metricutil` package, used everywhere.

## 4. `family.go` before vs. after

### 4.1 The `FamilyType` struct

**Before** — a special field just for Starlark:

```go
type FamilyType struct {
    v1alpha1.Family
    logger              klog.Logger
    celCostLimit        uint64
    celTimeout          time.Duration
    celEvaluations      *prometheus.CounterVec
    managedRMMNamespace string
    managedRMMName      string
    createdAt           time.Time
    cutoff              atomic.Bool
    starlarkResolver    *resolver.StarlarkResolver // <-- special case
}
```

**After** — the field is gone. Starlark is just another resolver the
factory can build:

```go
type FamilyType struct {
    v1alpha1.Family
    logger              klog.Logger
    celCostLimit        uint64
    celTimeout          time.Duration
    celEvaluations      *prometheus.CounterVec
    managedRMMNamespace string
    managedRMMName      string
    createdAt           time.Time
    cutoff              atomic.Bool
}
```

### 4.2 The resolver factory

**Before** — Starlark is explicitly refused:

```go
case v1alpha1.ResolverTypeStarlark:
    return nil, fmt.Errorf("starlark resolver requires starlark config in family %q", f.Name)
```

**After** — Starlark is built like anything else:

```go
case v1alpha1.ResolverTypeStarlark:
    return resolver.NewStarlarkResolver(f.logger, f.StarlarkConfig), nil
```

### 4.3 `buildMetricString`

**Before** — a top-level `switch` on a struct field decides which of two
*entirely separate* code paths (one full loop each) to run:

```go
func (f *FamilyType) buildMetricString(unstructured *unstructured.Unstructured) (string, int64) {
    var metricStr string
    var sampleCount int64

    switch {
    case f.starlarkResolver != nil:
        metricStr, sampleCount = f.buildMetricStringFromStarlark(unstructured)
    default:
        familyRawBuilder := getBuilder()
        defer putBuilder(familyRawBuilder)

        for i := range f.Metrics {
            // ~30 lines: resolve labels, resolve value, write samples
        }
        metricStr = familyRawBuilder.String()
    }
    // cutoff handling...
    return metricStr, sampleCount
}

// A second, separate ~40-line function exists ONLY for Starlark,
// reimplementing sanitizeKey, sortLabels, and sample writing by hand.
func (f *FamilyType) buildMetricStringFromStarlark(unstr *unstructured.Unstructured) (string, int64) {
    ...
}
```

**After** — one loop over metrics. Each metric's resolver is asked what it
is; both kinds feed the *same* sample writer at the end:

```go
func (f *FamilyType) buildMetricString(unstructured *unstructured.Unstructured) (string, int64) {
    logger := f.logger.WithValues("family", f.Name)

    familyRawBuilder := getBuilder()
    defer putBuilder(familyRawBuilder)

    var sampleCount int64

    for i := range f.Metrics {
        metric := &f.Metrics[i]

        resolverInstance, err := f.resolver(metric.Resolver)
        if err != nil {
            logger.V(1).Error(err, "skipping")
            continue
        }

        var samples int64

        switch r := resolverInstance.(type) {
        case resolver.FamilyGenerator:
            samples = f.writeGeneratedFamilies(familyRawBuilder, r, unstructured, logger)
        case resolver.Resolver:
            samples = f.writeResolvedMetric(familyRawBuilder, r, metric, unstructured, logger)
        }

        sampleCount += samples
    }

    if f.IsCutoff() {
        logger.V(1).Info("Family is cut off due to cardinality limits, suppressing metric output")
        return "", sampleCount
    }

    return familyRawBuilder.String(), sampleCount
}
```

The old "generic path" body moves, unchanged, into `writeResolvedMetric`.
The old Starlark-only function is replaced by a small adapter that reuses
the *same* sample writer as everything else:

```go
// writeGeneratedFamilies converts FamilyGenerator output into the same
// (keys, values, value) shape the generic path uses, then writes it with
// the one shared writer — no separate sample-writing logic for Starlark.
func (f *FamilyType) writeGeneratedFamilies(
    builder *strings.Builder, r resolver.FamilyGenerator,
    unstr *unstructured.Unstructured, logger klog.Logger,
) int64 {
    generated, err := r.GenerateFamilies(unstr.Object)
    if err != nil {
        logger.V(1).Error(err, "resolver generation failed")
        return 0
    }

    var sampleCount int64

    for _, genFamily := range generated {
        for _, sample := range genFamily.Samples {
            keys, values := labelMapToKeysValues(sample.Labels, r.IsKeySanitize())
            sortLabels(keys, values)

            value := strconv.FormatFloat(sample.Value, 'f', -1, 64)

            n, err := writeMetricSamplesWithCount(
                builder, f.Name, f.kind(), unstr, keys, values, nil, value, logger,
            )
            if err != nil {
                continue
            }
            sampleCount += n
        }
    }

    return sampleCount
}
```

### 4.4 `resolveLabels` and `resolveMetricValue`

**Before** — the caller has to guess "is this a list?" from key names, using
a regex and a NUL-byte sentinel:

```go
resolvedLabelset := resolverInstance.Resolve(label.Value, obj)
if val, ok := resolvedLabelset[label.Value]; ok {
    // scalar
} else {
    // must be a list or map — figure it out from key suffixes
    resolvedExpandedLabelSet[sanitizedName] = append(..., collectIndexedResolvedValues(resolvedLabelset)...)
    for k, v := range resolvedLabelset {
        if listIndexRegex.MatchString(k) {
            continue
        }
        // ... map handling
    }
}
```

**After** — the resolver tells you what it returned; no guessing:

```go
result, err := resolverInstance.Resolve(label.Value, obj)
if err != nil {
    logger.V(1).Error(err, "skipping label")
    continue
}

switch result.Kind() {
case resolver.KindScalar:
    resolvedLabelValues = append(resolvedLabelValues, result.Scalar())
    resolvedLabelKeys = append(resolvedLabelKeys, sanitize(label.Name, resolverInstance))
case resolver.KindList:
    resolvedExpandedLabelSet[sanitize(label.Name, resolverInstance)] = result.List()
case resolver.KindMap:
    for k, v := range result.Map() {
        resolvedLabelValues = append(resolvedLabelValues, v)
        resolvedLabelKeys = append(resolvedLabelKeys, sanitize(k, resolverInstance))
    }
}
```

`listIndexRegex`, `collectIndexedResolvedValues`, and `expandedValueSentinel`
are deleted — there's nothing left for them to do once `Result` carries its
own shape.

## 5. How this solves each problem

**Problem 1 (Starlark isn't a real Resolver).** Starlark is no longer a
special field checked with `!= nil`. It's a resolver that implements
`FamilyGenerator` — a real, named interface in the same package as
`Resolver`. The factory can build it like any other resolver, and the
dispatch in `buildMetricString` is a type switch on *what the resolver can
do*, not on *which concrete struct type `FamilyType` happens to hold*. If a
fourth resolver type shows up later that also generates whole families
(instead of one label at a time), it plugs into the exact same
`FamilyGenerator` path with zero changes to `FamilyType`.

**Problem 2 (duplicated sample-writing).** By converting `FamilyGenerator`
output into the same `(keys, values, value)` shape used everywhere else, and
routing it through the one shared `writeMetricSamplesWithCount`, there is
only one place where "write a Prometheus sample line" is implemented. A bug
fix there now automatically applies to CEL, Unstructured, and Starlark
output — nobody has to remember to fix it twice.

**Problem 3 (string-trick encoding for composite data).** `Result` says
directly whether it holds a scalar, a list, or a map — `Kind()` tells you,
and `List()` gives you a real, ordered `[]string`. There's no more need for
`#0`/`#1` suffix keys or a NUL-byte sentinel to smuggle list data through a
flat map. The caller asks "what kind of thing did I get back" instead of
reverse-engineering it from key names.

**Problem 4 (sanitization is a floating convention).** `IsKeySanitize()` is
part of `Capabilities`, which every resolver (including Starlark, through
`FamilyGenerator`) must implement. Sanitization is applied exactly once, in
the shared label code, and it's driven by something the resolver itself
declares — not by two call sites separately remembering to do it. A future
resolver that needs different rules can simply return `false` and handle its
own sanitization, instead of silently getting (or missing) rules baked into
unrelated code.

## 6. What doesn't change

- CEL's and Unstructured's actual resolution logic is untouched — only their
  return type changes from `map[string]string` to `Result`.
- `writeMetricSamplesWithCount`, `sortLabels`, and the list-expansion
  machinery for the generic path stay as they are.
- The public behavior of existing metrics (label names, sample output) is
  meant to be identical before and after this change.

## 7. Shared logic lives in one place, once

Some logic — key sanitization, sorting labels, formatting a float value —
isn't really "resolver logic" or "family logic." It's used by both sides
today, sometimes with two separate copies (e.g. `family.go`'s `sanitizeKey`
vs. `cel.go`'s `labelPrefixBinding` calling its own version). Instead of
each package rolling its own, this logic moves into the existing
`pkg/metricutil` package as plain functions. Resolvers and `family.go` call
into it if they need it — nobody is forced to, but nobody has to reimplement
it either.

```go
// pkg/metricutil/labels.go
package metricutil

func SanitizeKey(s string) string {
    return strcase.ToSnake(nonWordRegex.ReplaceAllString(s, "_"))
}

func SortLabels(keys []string, parallel ...[]string) { /* moved here, unchanged */ }
```

```go
// used from family.go
key := metricutil.SanitizeKey(label.Name)

// used from resolver/cel.go, same function, no local copy
sanitized := metricutil.SanitizeKey(k)
```

## 8. Open questions

1. **Are `HasTimeout()` and `UnderscoreExpansionSupported()` actually
   consumed anywhere?** `IsKeySanitize()` has a clear, single call site in
   the shared label code. It's not yet clear what in `family.go` (or
   elsewhere) is supposed to branch on the other two capability flags. If
   nothing does, they're decorative — an interface promising behavior that
   no caller ever asks about — and should either be wired up to something
   real (e.g. `HasTimeout()` gating a per-resolver timeout wrapper) or
   dropped from `Capabilities` until there's an actual consumer.
