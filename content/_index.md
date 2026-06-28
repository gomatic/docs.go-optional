---
title: go-optional
---

`go-optional` is the gomatic ecosystem's generic implementation of the [**functional-options pattern**](https://dave.cheney.net/2014/10/17/functional-options-for-friendly-apis) for Go. It provides a single generic interface — [`Option[T]`](https://pkg.go.dev/github.com/gomatic/go-optional#Option) — that any value can satisfy by knowing how to `Apply(*T)` to a target, plus the constructors ([`New`](https://pkg.go.dev/github.com/gomatic/go-optional#New), [`NewWithDefault`](https://pkg.go.dev/github.com/gomatic/go-optional#NewWithDefault)) that compose those options into a finished `T`. Despite its name, it is **not** a `Some`/`None`/`Maybe`/`Option` monad: it is the composable-configuration pattern for building friendly constructor APIs, not a nullable/optional value type.

- **Source:** [`gomatic/go-optional`](https://github.com/gomatic/go-optional)
- **API reference:** [pkg.go.dev/github.com/gomatic/go-optional](https://pkg.go.dev/github.com/gomatic/go-optional)

The import path is `github.com/gomatic/go-optional`; the package name is `optional`. Requires Go 1.26+.

## Install

```sh
go get github.com/gomatic/go-optional
```

## Functional options, not a Maybe monad

The functional-options pattern lets a constructor take a variadic list of composable, self-describing configuration values instead of a long positional argument list or a mutable config struct. Each option knows how to mutate the value being built; the constructor applies them in order, last-write-wins. The result is a friendly, extensible API where callers pass only the options they care about and defaults fill in the rest.

`go-optional` owns the **mechanism only** — it ships no concrete options. An [`Option[T]`](https://pkg.go.dev/github.com/gomatic/go-optional#Option) is anything that implements `Apply(*T)`; every consumer declares its own option types for its own struct:

```go
import optional "github.com/gomatic/go-optional"

type Thing struct{ name string }

// Name is a functional option for Thing — a value that configures a *Thing.
type Name string

func (n Name) Apply(t *Thing) { optional.Must(t).name = string(n) }
```

## Usage

### Build a value from options

[`New`](https://pkg.go.dev/github.com/gomatic/go-optional#New) starts from the zero value of `T` and applies each option on top; [`NewWithDefault`](https://pkg.go.dev/github.com/gomatic/go-optional#NewWithDefault) starts from a default `T` you provide instead.

```go
package main

import (
	"fmt"

	optional "github.com/gomatic/go-optional"
)

type Thing struct{ name string }

type Name string

func (n Name) Apply(t *Thing) { optional.Must(t).name = string(n) }

func main() {
	thing := optional.New[Thing](Name("example"))
	withDefault := optional.NewWithDefault(Thing{name: "default"}, Name("override"))

	fmt.Printf("%+v\n", thing)       // {name:example}
	fmt.Printf("%+v\n", withDefault) // {name:override}
}
```

A `nil` option in the slice is skipped, and options are applied left-to-right so later options win over earlier ones and over the default.

### Define options as `var` or `const`

Because an option is just a typed value, it can be declared either way and passed straight to [`New`](https://pkg.go.dev/github.com/gomatic/go-optional#New):

```go
varName := Name("a var Name")
const constName = Name("a const Name")

optional.New[Thing](varName)   // {name:a var Name}
optional.New[Thing](constName) // {name:a const Name}
```

### Wrap the constructor (optional)

When a constructor does nothing but apply options over a default, you do not strictly need it — callers can reach for [`New`](https://pkg.go.dev/github.com/gomatic/go-optional#New) or [`NewWithDefault`](https://pkg.go.dev/github.com/gomatic/go-optional#NewWithDefault) directly. A thin wrapper is still convenient for baking in package defaults:

```go
func NewThing(opts ...optional.Option[Thing]) Thing {
	return optional.NewWithDefault(Thing{name: "simplifies defaults"}, opts...)
}

NewThing()                  // {name:simplifies defaults}
NewThing(Name("like first")) // {name:like first}
```

### `Must` — guard against a nil receiver

[`Must`](https://pkg.go.dev/github.com/gomatic/go-optional#Must) returns a non-nil `*T`: hand it a `nil` and you get a fresh `*T` back, otherwise you get the pointer untouched. Using it at the top of an `Apply` method makes the option a no-op against a `nil` target instead of panicking. If you would rather have the panic, skip `Must` and let the usual `nil` dereference happen.

```go
func (n Name) Apply(t *Thing) { optional.Must(t).name = string(n) }

aThing := optional.Must[Thing](nil) // &Thing{} — never nil
aThing.name = "private pointer instance"
```

### Configuration maps

For dynamic, string-keyed configuration, [`Configuration`](https://pkg.go.dev/github.com/gomatic/go-optional#Configuration) is an option-applied `map[string]any` that decodes into a struct via [mapstructure](https://github.com/go-viper/mapstructure). Declare typed [`ConfigurationKey[T]`](https://pkg.go.dev/github.com/gomatic/go-optional#ConfigurationKey) keys, turn them into options with `.Opt(value)` (or [`Enabled`](https://pkg.go.dev/github.com/gomatic/go-optional#Enabled) for a `bool` flag), build the map with [`NewConfiguration`](https://pkg.go.dev/github.com/gomatic/go-optional#NewConfiguration), and decode it with [`Decode`](https://pkg.go.dev/github.com/gomatic/go-optional#Decode):

```go
package main

import (
	"fmt"
	"math"

	optional "github.com/gomatic/go-optional"
)

const (
	userKey    optional.ConfigurationKey[string] = "user"
	enabledKey optional.ConfigurationKey[bool]   = "enabled"
	countKey   optional.ConfigurationKey[int]    = "count"
	amountKey  optional.ConfigurationKey[float64] = "amount"
)

type Config struct {
	User    string  `mapstructure:"user"`
	Enabled bool    `mapstructure:"enabled"`
	Count   int     `mapstructure:"count"`
	Amount  float64 `mapstructure:"amount"`
}

func main() {
	cfg, err := optional.Decode[Config](optional.NewConfiguration(
		optional.Enabled(enabledKey),
		userKey.Opt("test-string"),
		countKey.Opt(99),
		amountKey.Opt(math.Pi),
	))
	fmt.Printf("%v %#v\n", err, cfg)
	// <nil> &main.Config{User:"test-string", Enabled:true, Count:99, Amount:3.141592653589793}
}
```

[`KV`](https://pkg.go.dev/github.com/gomatic/go-optional#KV) is a shorthand for building an [`Opt`](https://pkg.go.dev/github.com/gomatic/go-optional#Opt) from a raw string key and value without declaring a typed `ConfigurationKey` first. A failed decode returns an error that matches the sentinel [`ErrDecode`](https://pkg.go.dev/github.com/gomatic/go-optional#ErrDecode) under [`errors.Is`](https://pkg.go.dev/errors#Is), wrapping the underlying mapstructure failure with `%w`.

### Generic option-applied maps

[`Map[K, V]`](https://pkg.go.dev/github.com/gomatic/go-optional#Map) is the fully generic form behind `Configuration`: an option-applied `map[K]V` whose option is a [`Pair[K, V]`](https://pkg.go.dev/github.com/gomatic/go-optional#Pair) carrying one key and value. Use it when you need a configurable map keyed by something other than `string`.

```go
m := optional.New[optional.Map[string, int]](
	optional.Pair[string, int]{Key: "port", Value: 8080},
)
```

## Design

- **One interface, everything else is a value.** [`Option[T]`](https://pkg.go.dev/github.com/gomatic/go-optional#Option) is the entire contract — `Apply(*T)`. Options are ordinary values, safe to declare as `const`, copy, and reuse.
- **Mechanism only, no concrete options.** The package ships the constructors and the `Configuration`/`Map` helpers; each consumer declares the option types for its own structs.
- **Nil-safe by construction.** [`NewWithDefault`](https://pkg.go.dev/github.com/gomatic/go-optional#NewWithDefault) skips `nil` options, and [`Must`](https://pkg.go.dev/github.com/gomatic/go-optional#Must) lets an `Apply` method tolerate a `nil` target.
- **Constant/sentinel errors.** Every error the package emits is a const of the `Error` string type, matchable with [`errors.Is`](https://pkg.go.dev/errors#Is); [`ErrDecode`](https://pkg.go.dev/github.com/gomatic/go-optional#ErrDecode) wraps the underlying cause with `%w`.
- **Dependency-light.** The core options machinery depends only on the standard library; `Configuration` decoding pulls in [mapstructure](https://github.com/go-viper/mapstructure).

## Who uses it

`go-optional` underpins the friendly constructor APIs across the gomatic Go projects — see the other [`gomatic/go-*`](https://github.com/orgs/gomatic/repositories?q=go-) libraries.
