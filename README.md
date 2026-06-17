# su26-ai301-contribution

# Contribution 1: Try to determine the hostname for MongoDB connections

**Contribution Number:** 1

**Student:** Yuyan Ke 

**Issue:** https://github.com/open-telemetry/opentelemetry-ebpf-instrumentation/issues/1192

**Status:** Phase I Complete

---

## Why I Chose This Issue

[1-2 paragraphs explaining why this issue interests you, how it matches your skills/learning goals, what you hope to learn]

This issue deals with mongo database configuration, and given my background with system and recently starting to work more with Mongo DB at work, this issue offers an opportunity to dig into the Mongo configuration. Since this will be my very first open-source contribution, I'm hoping to learn and experience the end-to-end process first as a stepping stone. 

---

## Understanding the Issue

### Problem Description

[In your own words, what's broken or missing?]

Mongo connection detail does not include hostname, so this issue is to determine the Mongo database hostname from the connection string. Hostname is more descriptive and human-friendly compared to IP addresses or random strings. 

### Expected Behavior

[What should happen?]

Upon successfully completion of this issue, Mongo DB hostname can be determined and logged based on connection string.

### Current Behavior

[What actually happens?]

Only Mongo conenction string is available, no hostname.

### Affected Components

[Which parts of the codebase are involved?]

Code change in `go_mongo.c` , references available in `go_sql.c`, specifically the `read_mysql_hostname_from_mysqlconn()`.

---

## Reproduction Process

### Environment Setup

[Notes on setting up your local development environment - challenges you faced, how you solved them]
Setting up the local development environment on Windows 10 required a few extra steps since eBPF is a Linux kernel technology and cannot run natively on Windows.

**Challenge 1: eBPF does not run on Windows**

eBPF is a Linux kernel feature. Attempting to run the instrumentation directly on Windows 10 is not possible.

*Solution:* Install WSL2 (Windows Subsystem for Linux 2), which provides a real Linux kernel on Windows. WSL2 is available on Windows 10 build 19041+.

```powershell
# Run in PowerShell as Administrator
wsl --install -d Ubuntu
```

After rebooting, verify the kernel version inside WSL2:
```bash
uname -r
# Should show 5.15.x or higher
```

**Challenge 2: `docker compose -f` flag not recognized**

Running the test stack with `docker compose -f docker-compose-go-mongo.yml up --build` failed because the machine only had the older standalone `docker-compose` binary (v1.25.9), not the Docker Compose v2 plugin. The v2 plugin is what provides the `docker compose` (with a space) subcommand.

*Solution:* This pointed toward using the v1 binary (`docker-compose`) instead, but that surfaced a new problem (see Challenge 3).

**Challenge 3: `docker-compose` v1 rejects the compose file**

Switching to `docker-compose -f docker-compose-go-mongo.yml up --build` failed with:

```
ERROR: The Compose file './docker-compose-go-mongo.yml' is invalid because:
Unsupported config option for services: 'autoinstrumenter'
```

The compose file uses two features that v1 does not support:
- No `version:` field (modern Compose Spec format) — v1 misinterprets the file structure entirely
- `pid: "service:testserver"` — sharing a PID namespace with another service is v2-only

*Solution:* Docker Compose v2 is required. There is no workaround using v1.

**[WIP] Challenge 4: `docker-compose-plugin` not available via apt on Ubuntu 20.04**

Attempting to install the v2 plugin through apt failed:

```
E: Unable to locate package docker-compose-plugin
```

This is because Ubuntu 20.04 (Focal) does not include `docker-compose-plugin` in its default package repositories.

*Solution:* Install the Docker Compose v2 binary directly from GitHub releases into the Docker CLI plugins directory:

```bash
DOCKER_CONFIG=${DOCKER_CONFIG:-$HOME/.docker}
mkdir -p $DOCKER_CONFIG/cli-plugins
curl -SL https://github.com/docker/compose/releases/download/v2.27.0/docker-compose-linux-x86_64 \
  -o $DOCKER_CONFIG/cli-plugins/docker-compose
chmod +x $DOCKER_CONFIG/cli-plugins/docker-compose
```

### Steps to Reproduce [Expected]

**Step 1 — Start the test stack**

From the repo root in a WSL2 terminal:
```bash
cd internal/test/oats/mongo
docker compose -f docker-compose-go-mongo.yml up --build
```

This starts three services: a MongoDB instance (port 27017), a Go test HTTP server (port 8080), and the OBI eBPF auto-instrumenter attached to the test server's process.

**Step 2 — Trigger MongoDB operations**

In a second WSL2 terminal:
```bash
curl http://localhost:8080/mongo
```

This runs InsertOne → FindOne → UpdateOne → FindOne → DeleteOne against the `testdb.items` collection.

**Step 3 — Observe the span output**

```bash
docker compose -f docker-compose-go-mongo.yml logs autoinstrumenter
```


### Reproduction Evidence

- **Commit showing reproduction:** [Link to commit in your fork]
- **Screenshots/logs:** [If applicable]
- **My findings:** [What you discovered during reproduction]

---

## Solution Approach

### Analysis

[Your analysis of the root cause - what's causing the issue?]

### Proposed Solution

[High-level description of your fix approach]

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** 
MongoDB spans produced by the eBPF instrumentation are missing the `server.address` attribute. The connection string (e.g., `mongodb://mongo:27017`) contains the hostname, but `go_mongo.c` never reads or stores it — the `mongo_go_client_req_t` struct has no hostname field, so the user-space transform has nothing to emit as `server.address`.

**Match:**
`go_sql.c` solves the identical problem for MySQL and PostgreSQL. It reads a hostname string by dereferencing a pointer chain from the driver connection struct into an internal config field (`mysqlConn.cfg.Addr`, `pgx.Conn.config.Host`). The offset for that field is registered in `go_offsets.h`, injected at eBPF load time, and the string is extracted with `read_go_str()`. The MongoDB equivalent pointer chain starts from the `Operation` struct (already available in `obi_uprobe_mongo_op_execute`) via its `Deployment` interface field, which holds a `*topology.Topology` whose `servers` map entry contains a `description.Server` with an `Addr` string field.

**Plan:**
1. **[bpf/common/common.h]** — Add `unsigned char hostname[k_sql_hostname_max_len]` field to `mongo_go_client_req_t` (mirror the same constant used by `sql_request_trace_t`).
2. **[bpf/gotracer/go_offsets.h]** — Add a new offset entry (e.g., `_mongo_server_addr_pos`) for the address string field inside the MongoDB driver's topology server struct.
3. **[bpf/gotracer/go_mongo.c]** — In `obi_uprobe_mongo_op_execute`, after reading `req->db`, dereference `Operation.Deployment` → `topology.Topology` → server address string and store it in `req->hostname` using `read_go_str()`.
4. **[pkg/ebpf/common/mongo_detect_transform.go]** — Read `req.hostname` from the eBPF event and set it as the `server.address` span attribute (mirror how the SQL transform handles `trace.hostname`).
5. **[internal/test/oats/mongo/yaml/oats_go_mongo.yaml]** — Add `server.address: 'mongo'` to the expected attributes on the MongoDB span assertion.

**Implement:** [Link to your branch/commits as you work]

**Review:**
- [ ] New struct field does not break struct layout alignment (padding may be needed)
- [ ] Offset key follows the existing `_mongo_*` naming convention in `go_offsets.h`
- [ ] `read_go_str` call is guarded with a `bpf_dbg_printk` on failure, matching the style of other reads in `go_mongo.c`
- [ ] Transform sets `server.address` only when the hostname field is non-empty (avoid emitting blank attribute)
- [ ] Contribution follows CONTRIBUTING.md — DCO sign-off on commits

**Evaluate:**
Run the OATS test locally after the fix:
​```bash
cd internal/test/oats/mongo
docker compose -f docker-compose-go-mongo.yml up --build
curl http://localhost:8080/mongo
docker compose -f docker-compose-go-mongo.yml logs autoinstrumenter
​```
The span for `insert testdb.items` should now include `server.address: mongo`. The OATS assertion in `oats_go_mongo.yaml` will also catch regressions automatically in CI.

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
