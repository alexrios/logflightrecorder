# logflightrecorder

[![CI](https://github.com/alexrios/logflightrecorder/actions/workflows/ci.yml/badge.svg)](https://github.com/alexrios/logflightrecorder/actions/workflows/ci.yml)
[![Go Reference](https://pkg.go.dev/badge/github.com/alexrios/logflightrecorder.svg)](https://pkg.go.dev/github.com/alexrios/logflightrecorder)
[![Go Report Card](https://goreportcard.com/badge/github.com/alexrios/logflightrecorder)](https://goreportcard.com/report/github.com/alexrios/logflightrecorder)

A `slog.Handler` that keeps the last N log records in a fixed-size circular buffer.
Zero external dependencies -- stdlib only.

Primary use case: exposing recent logs via health-check or admin HTTP endpoints.
Inspired by `runtime/trace.FlightRecorder`, it can also act as a **black box recorder**
that flushes context-rich logs on error.

## Install

```bash
go get github.com/alexrios/logflightrecorder
```

## Quick start

```go
package main

import (
	"encoding/json"
	"log/slog"
	"net/http"

	lfr "github.com/alexrios/logflightrecorder"
)

func main() {
	ring := lfr.New(500, nil)
	logger := slog.New(ring)
	slog.SetDefault(logger)

	http.HandleFunc("GET /debug/logs", func(w http.ResponseWriter, r *http.Request) {
		data, err := ring.JSON()
		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}
		w.Header().Set("Content-Type", "application/json")
		w.Write(data)
	})

	slog.Info("server starting", "port", 8080)
	http.ListenAndServe(":8080", nil)
}
```

## API overview

| Function / Method | Description |
|---|---|
| `New(size, opts)` | Create a handler with buffer capacity `size` |
| `Handle(ctx, record)` | Store a record (implements `slog.Handler`); triggers flush if `FlushOn` threshold is met |
| `WithAttrs(attrs)` | Return a handler with additional attributes (shared buffer) |
| `WithGroup(name)` | Return a handler with a group prefix (shared buffer) |
| `Records()` | Snapshot of stored records, oldest to newest (respects `MaxAge`) |
| `All()` | `iter.Seq[slog.Record]` iterator over stored records (respects `MaxAge`) |
| `JSON()` | Marshal records as a JSON array (respects `MaxAge`) |
| `WriteTo(w)` | Stream records as JSON to an `io.Writer` (implements `io.WriterTo`) |
| `Len()` | Number of records physically stored (ignores `MaxAge`) |
| `Capacity()` | Total buffer capacity |
| `Clear()` | Remove all records |

### Options

| Field | Type | Description |
|---|---|---|
| `Level` | `slog.Leveler` | Minimum level stored (default: `INFO`) |
| `FlushOn` | `slog.Leveler` | Level that triggers flush to `FlushTo` |
| `FlushTo` | `slog.Handler` | Destination for flushed records |
| `MaxAge` | `time.Duration` | Exclude records older than this from reads; `0` = no filter |

## Black box pattern

Keep a ring buffer of recent logs and flush them to stderr when an error occurs:

```go
ring := lfr.New(500, &lfr.Options{
	FlushOn: slog.LevelError,
	FlushTo: slog.NewJSONHandler(os.Stderr, nil),
	MaxAge:  5 * time.Minute,
})
logger := slog.New(ring)

logger.Info("request started", "path", "/api/users")
logger.Info("db query", "rows", 42)
// ... when an error happens, all recent logs are flushed to stderr
logger.Error("query failed", "err", err)
```

Serve the ring buffer over HTTP with `WriteTo`:

```go
http.HandleFunc("GET /debug/logs", func(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")
	ring.WriteTo(w) // implements io.WriterTo
})
```

## Benchmarks

Measured on an Intel Core i9-14900K (32 threads):

| Benchmark | ns/op | B/op | allocs/op |
|---|---:|---:|---:|
| Handle | 91 | 0 | 0 |
| Handle_Parallel | 271 | 0 | 0 |
| Handle_WithFlush | 86 | 0 | 0 |
| Handle_FlushTrigger | 42,135 | 32,768 | 1 |
| Records (1000) | 182,295 | 294,913 | 1 |
| All (1000) | 126,891 | 294,912 | 1 |
| JSON (100 records, 5 attrs) | 380,854 | 140,814 | 1,804 |
| WriteTo | 386,905 | 140,843 | 1,804 |
| WithAttrs (5 attrs) | 257 | 288 | 2 |
| WithGroup | 47 | 16 | 1 |
| Records_WithMaxAge | 245,142 | 294,912 | 1 |

## License

[MIT](LICENSE)
