## Summary

This PR fixes MongoDB `server.address` for the Go tracer so spans report the configured hostname instead of falling back to the raw socket IP when that hostname is available.

The change follows the same overall pattern used by the SQL instrumentation:

- harden DWARF member scanning so unnamed or non-constant members do not abort offset discovery
- add hostname plumbing to the MongoDB event path
- register the MongoDB driver offsets needed to walk `driver.Operation -> Deployment -> topology.Topology -> cfg -> SeedList`
- extract `SeedList[0]` in the eBPF path and carry it into the userspace span transform
- assert `server.address: mongo` in the Go MongoDB OATS coverage

One caveat is that `SeedList[0]` is the configured seed address, not necessarily the exact server selected for a replica set or `mongodb+srv://` deployment. That matches the existing SQL behavior in this repository and still improves the current MongoDB output from IP-only to hostname-aware for the supported test cases.

## Validation

- [x] I have read and followed the [contributing guidelines](https://github.com/open-telemetry/opentelemetry-ebpf-instrumentation/blob/main/CONTRIBUTING.md)
- [x] If this enhances / fixes / changes a core feature, I have updated the [features documentation](https://github.com/open-telemetry/opentelemetry-ebpf-instrumentation/blob/main/devdocs/features.md) and [support matrix](https://github.com/open-telemetry/opentelemetry-ebpf-instrumentation/blob/main/SUPPORT_MATRIX.md) as needed.

Validation plan from the implementation notes:

- `go test -run 'TestReadMembers|TestGoOffsetsFromDwarf' ./pkg/internal/goexec/`
- `go test -run Mongo ./pkg/ebpf/common/`
- `go test -run TestRawTraceDispatcher ./pkg/ebpf/common/`
- `make compile`
- `make oats-test-mongo`
- `make verify`
- `make build`
- `make lint`
- `make test`

The current planning document records implementation work as complete and final validation as still in progress.

## Generative AI Disclosure

Generative AI was used to compare the MongoDB path against the existing MySQL hostname-logging path and help identify the changes needed for MongoDB to behave the same way. Claude Code was also used to design the full planning document and implementation plan for this issue.

When the work hit roadblocks such as stack-size overlap concerns and unexpected errors during debugging, Generative AI was used as a troubleshooting aid to narrow down likely causes and next checks. The resulting plan, affected files, caveats, and validation steps were then reviewed against the repository code and local findings before submission.
