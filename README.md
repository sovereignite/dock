# endpoint

A small Go library for publishing, validating, and resolving **boot-scoped service
endpoint records** on a single host.

`endpoint` is a leaf library extracted from
[github.com/sovereignite/sovereignite](https://github.com/sovereignite/sovereignite)
(`internal/endpoint`). It has no dependencies outside the Go standard library.

## What it does

A service publishes its loopback address to a JSON record so that other local
processes can discover where to reach it. Records are scoped to a specific boot
and process instance, so a stale record from a previous boot or a crashed PID
is rejected rather than trusted.

By convention the record for a service lives at:

```
/run/sovereignite/<service>/endpoint.json
```

Records are written atomically (temp file + rename + fsync) with owner-only
permissions, and reads reject symlinks and malformed JSON.

## API

- `Publish(dir, record)` — atomically write a record to `dir/endpoint.json`,
  creating the directory owner-only if needed. Returns the loopback
  `net.Addr` the caller should listen on.
- `PublishSerialised(dir, record)` — `Publish` under a process-wide lock.
- `Validate(record, bootID, pid, service)` — reject stale boot IDs, wrong
  services, non-loopback addresses, PID mismatches, and malformed records.
- `ValidateRecord(dir, bootID, pid, service)` — read + validate in one step.
- `ReadRecord(dir)` / `RecordFromFile(path)` — read a record, rejecting
  symlinks and malformed JSON.
- `Cleanup(dir, record)` — remove a record only if it still matches the given
  service and instance nonce.
- `GenerateNonce()` — random instance nonce.
- `NewServiceDir(base, service)` — join a base directory and service name.

The record schema is versioned via `RecordVersion` (currently 1).

## Example

```go
package main

import (
	"log"
	"os"

	"github.com/sovereignite/dock"
)

func main() {
	dir := endpoint.NewServiceDir("/run/sovereignite", "myservice")
	rec := endpoint.EndpointRecord{
		Service:       "myservice",
		BootID:        "boot-uuid",
		InstanceNonce: endpoint.GenerateNonce(),
		PID:           os.Getpid(),
		Network:       "tcp",
		Address:       "127.0.0.1",
		Port:          8080,
	}

	addr, err := endpoint.PublishSerialised(dir, rec)
	if err != nil {
		log.Fatalf("publish: %v", err)
	}
	_ = addr // net.Addr to listen on
}
```

## License

GPL-2.0-only. See [LICENSE.md](LICENSE.md).
