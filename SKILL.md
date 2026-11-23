---
name: Golang Backend Style
description: Write and edit Go backend code in a layered, opinionated idiom — chi + pgx + raw SQL, sentinel errors, a normalize/validate split, and table-driven tests. Use it whenever a Go service follows, or should follow, this architecture: new endpoints, handlers, repository functions, models, middlewares, schema, or tests.
---

# Go Backend Style Guide

This guide defines a complete, opinionated idiom for Go backend
services: a layered `internal/` architecture, strict error discipline,
a fixed validation flow, a distinctive vertical format, and a
table-driven testing culture. It was distilled from a production code
base and is written to be adopted by any service: new code should
speak this idiom natively, expressing each feature fresh rather than
cloning existing code.

The skeletons below use a placeholder entity (`Thing`) on purpose:
they demonstrate the form of each layer without tying the guide to
any real domain.

All **code, comments, and error messages are written in English.**

## Stack

The idiom assumes this toolchain; substitute only with good reason:

- Routing: `go-chi/chi/v5`
- Database: `jackc/pgx/v5` through a global `pgxpool`
- Auth: `golang-jwt/jwt/v5` (HS256) + `golang.org/x/crypto/bcrypt`
- Config: environment variables loaded with `joho/godotenv`
- Tests: stdlib `testing` + `httptest`; database-backed tests behind
  an `integration` build tag

## Core principles

1. **GoDoc comment on every exported identifier**, explaining *why*
   it is designed that way rather than merely what it does. Write
   multi-line prose comments in the guide's voice; reserve the list
   format for rule lists (e.g. `//   - 3-20 characters,`).
2. **One package doc comment per package**
   (`// Package auth implements ...`), placed above the `package`
   clause in exactly one file.
3. **Sentinel errors + `%w` wrapping + package prefix** (detailed
   below).
4. **Security never leaks information:** a missing resource and a
   denied access produce the same response; which step failed is
   never disclosed.
5. **normalize and validate are separate**, and both live in
   helpers.go.
6. **Simplicity first:** no interfaces, no dependency injection;
   repository functions are package-level and use the global
   `store.DB`.
7. Code is always `gofmt`-clean: `gofmt -l ./internal` must return
   nothing.

## Layer architecture

```
internal/
├── config/        → environment variables (dotenv file, GetEnv)
├── store/         → pgxpool connection (global DB), schema bootstrap, shared pgx helpers
├── models/        → JSON request/response types, JWT claims. No logic.
├── repository/    → SQL queries, sentinel errors. Never sees HTTP.
├── handlers/      → router.go + per-domain packages
│   └── <domain>/  → handlers.go (HTTP) + helpers.go (normalize/validate)
└── middlewares/   → chi middlewares (Auth), context keys
```

Dependencies flow one way: `handlers → repository → store`, and
`handlers → models ← repository`. Handlers never write SQL; the
repository never writes HTTP. Shared types live in `internal/models`
with the `XxxReq` suffix (`CreateNewThingReq`); list-row response
types also live there (e.g. `OrganizationTeam`).

## Adding a new endpoint

1. `internal/models/<domain>.go`: add the `XxxReq` type with a doc
   comment.
2. Store package: add the required table or index to the schema
   bootstrap with `IF NOT EXISTS`.
3. `internal/repository/<domain>.go`: add the function and its
   sentinel errors.
4. `internal/handlers/<domain>/helpers.go`: constants, normalize,
   validate.
5. `internal/handlers/<domain>/handlers.go`: the `XxxHandler`.
6. `internal/handlers/router.go`: mount the route.
7. Unit tests (`*_test.go`) and, where needed,
   `*_integration_test.go`.

When opening a new domain package, use the same file layout:
`handlers.go`, `helpers.go`, `handlers_test.go`, `helpers_test.go`,
`handlers_integration_test.go`.

## Handler recipe

The handler doc comment also names the status codes:

```go
// CreateNewThingHandler validates the creation body and stores a
// thing inside the caller's organization, answering 201 with the new
// id. The caller must be the organization's owner (404 otherwise)
// and the identifier must be free (409 otherwise).
func CreateNewThingHandler(
	w http.ResponseWriter,
	r *http.Request,
) {
	w.Header().Set("Content-Type", "application/json")

	claims, ok := middlewares.UserFromContext(r.Context())

	if !ok {
		w.WriteHeader(http.StatusUnauthorized)

		json.NewEncoder(w).Encode(map[string]any{
			"error": "unauthorized",
		})

		return
	}

	thingID, err := strconv.ParseInt(
		chi.URLParam(r, "thing_id"), 10, 64,
	)

	if err != nil {
		w.WriteHeader(http.StatusBadRequest)

		json.NewEncoder(w).Encode(map[string]any{
			"error": "invalid_thing_id",
		})

		return
	}

	if thingID <= 0 {
		// ... same invalid_thing_id 400 block
	}

	var req models.CreateNewThingReq

	err = json.NewDecoder(r.Body).Decode(&req)

	if err != nil {
		w.WriteHeader(http.StatusBadRequest)

		json.NewEncoder(w).Encode(map[string]any{
			"error": "invalid_request",
		})

		return
	}

	normalizeCreateNewThingReq(&req)

	err = validateCreateNewThingReq(req)

	if err != nil {
		w.WriteHeader(http.StatusBadRequest)

		json.NewEncoder(w).Encode(map[string]any{
			"error": err.Error(),
		})

		return
	}

	id, err := repository.CreateNewThing(
		r.Context(),
		req.Name,
		claims.ID,
	)

	if err != nil {
		if errors.Is(err, repository.ErrThingAlreadyExists) {
			w.WriteHeader(http.StatusConflict)

			json.NewEncoder(w).Encode(map[string]any{
				"error": "thing_already_exists",
			})

			return
		}

		w.WriteHeader(http.StatusInternalServerError)

		json.NewEncoder(w).Encode(map[string]any{
			"error": "internal_error",
		})

		return
	}

	w.WriteHeader(http.StatusCreated)

	json.NewEncoder(w).Encode(map[string]any{
		"message": "thing_created",
		"thingID": id,
	})
}
```

Fixed conventions:

- `w.Header().Set("Content-Type", "application/json")` is the first
  line of every handler.
- Step order: content-type → user context → URL param → body decode →
  normalize → validate → repository → error mapping → response.
- **Use `http.StatusXxx` constants** (`http.StatusCreated`); never
  bare numbers.
- Exactly one blank line between `w.WriteHeader(...)` and
  `json.NewEncoder(w).Encode(...)`, after every error block's closing
  `}`, and between logical steps (the vertical "paragraph" format).
- Every error body is `{"error": "..."}` written through
  `map[string]any`; response bodies are built the same way.
- Error keys are short snake_case machine identifiers
  (`invalid_request`, `thing_already_exists`); validation errors pass
  the validator's message verbatim (`err.Error()`).
- URL parameters are snake_case: `{organization_id}`, `{thing_id}`,
  `{member_id}`. Unparseable or `<= 0` ids answer 400 with
  `invalid_*_id`.
- Unexpected repository failures answer 500 `internal_error`; never
  leak the message to the client, and do not add logging — a silent
  500 is the established behavior.
- The JWT identity (`claims.ID`, `claims.Username`) travels through
  `r.Context()` via `middlewares.UserFromContext`.

## Repository recipe

Repository functions are package-level, take `ctx context.Context` as
the first parameter, and return multiple scalar values
(`(int64, error)`); no struct parameters. Names start with a verb and
read as a complete action: `InsertNewUser`, `GetUserByEmail`,
`CreateNewThing`, `DeleteThing`, `ListOrganizationThings`.

**Declare the sentinel error directly above the function that returns
it:**

```go
// ErrThingNotFound is returned by DeleteThing when no thing with the
// id belongs to the given organization. The "not the owner" case is
// indistinguishable on purpose.
var ErrThingNotFound = errors.New("repository: thing not found")
```

Query skeleton:

```go
func ListThings(
	ctx context.Context,
	organizationID int64,
	userID int64,
) ([]models.Thing, error) {
	var isMember bool

	err := store.DB.QueryRow(
		ctx,
		`
		SELECT EXISTS (
			SELECT 1
			FROM organization_members
			WHERE organization_id = $1 AND user_id = $2
		);
		`,
		organizationID,
		userID,
	).Scan(&isMember)

	if err != nil {
		return nil, fmt.Errorf("repository: check organization membership: %w", err)
	}

	if !isMember {
		return nil, ErrOrganizationNotFound
	}

	rows, err := store.DB.Query(
		ctx,
		`
		SELECT id, name
		FROM things
		WHERE organization_id = $1;
		`,
		organizationID,
	)

	if err != nil {
		return nil, fmt.Errorf("repository: list things: %w", err)
	}

	defer rows.Close()

	// Start with an empty slice so an empty result encodes as []
	// instead of null.
	things := []models.Thing{}

	for rows.Next() {
		var thing models.Thing

		err = rows.Scan(&thing.ID, &thing.Name)
		if err != nil {
			return nil, fmt.Errorf("repository: scan thing: %w", err)
		}

		things = append(things, thing)
	}

	if err := rows.Err(); err != nil {
		return nil, fmt.Errorf("repository: list things: %w", err)
	}

	return things, nil
}
```

Fixed conventions:

- **SQL is a multi-line raw string in backticks**; keywords UPPERCASE,
  body indented with tabs, statement ends with `;`. No ORM, no string
  builders.
- Parameters are `$1, $2, ...` and **each argument sits on its own
  line** at the call.
- `errors.Is(err, pgx.ErrNoRows)` → the not-found sentinel;
  `store.IsUniqueErr(err)` → the "already exists" sentinel. Never
  hardcode the `23505` SQLSTATE yourself — use the shared helper in
  the store package.
- After `Exec`, `RowsAffected() == 0` → the "not found" sentinel.
- Access control happens inside the query: `SELECT EXISTS(...)` or
  several `EXISTS/COALESCE` columns resolved in one `SELECT` — e.g.
  the caller's privilege and the target's role in a single query. A
  missing resource and an unauthorized caller produce the same error —
  deliberately.
- One blank line after the `var` block; one after every error block.
- Wrapping format: `fmt.Errorf("repository: <action>: %w", err)`. The
  prefix matches the package: `repository:`, `store:`, `config:`,
  `auth:`.

Transactions, for operations that perform multiple writes:

```go
	tx, err := store.DB.Begin(ctx)
	if err != nil {
		return 0, fmt.Errorf("repository: begin create thing: %w", err)
	}

	// Rollback is a no-op after a successful commit, so every early
	// return below safely discards the partial work.
	defer tx.Rollback(ctx)

	// ... tx.QueryRow / tx.Exec ...

	if err := tx.Commit(ctx); err != nil {
		return 0, fmt.Errorf("repository: commit create thing: %w", err)
	}
```

Single-use values such as refresh tokens merge the read and the
delete into one statement:
`DELETE ... WHERE ... RETURNING user_id;` — a consumed token can never
be replayed, even if the later steps fail.

## helpers.go recipe

All normalization and validation of a domain package lives in
helpers.go. Handlers call `normalizeXxxReq(&req)` by **pointer** and
`validateXxxReq(req)` by **value**.

```go
const (
	minThingNameLength = 1
	// maxThingNameLength bounds the display name, like organization
	// names.
	maxThingNameLength = 80
)

var thingIdentifierRe = regexp.MustCompile(`^[a-z0-9-]+$`)

// allowedThingRoles lists the roles an add-thing-member request can
// assign. The 'creator' role belongs to the thing's creator alone
// and is deliberately not assignable.
var allowedThingRoles = map[string]struct{}{
	"member": {},
	"admin":  {},
}

// normalizeCreateNewThingReq trims the text fields and lowercases the
// identifier so the exact values that pass validation are the ones
// stored. The password-like fields are deliberately left untouched:
// leading/trailing spaces can be part of them.
func normalizeCreateNewThingReq(req *models.CreateNewThingReq) {
	req.Name = strings.TrimSpace(req.Name)
	req.Identifier = strings.ToLower(strings.TrimSpace(req.Identifier))
}

// validateCreateNewThingReq checks that the request can be stored
// as-is. Messages are written for the client: they flow verbatim into
// the 400 {"error": ...} response.
func validateCreateNewThingReq(req models.CreateNewThingReq) error {
	if req.Name == "" {
		return errors.New("name is required")
	}

	if len(req.Name) > maxThingNameLength {
		return fmt.Errorf("name must be at most %d characters", maxThingNameLength)
	}

	return nil
}
```

Rules:

- Every bound and TTL is a named constant; no magic numbers.
  Non-trivial constants get a short "why" comment.
- Sets are `map[string]struct{}` (reserved names, allowed roles); no
  slice + `slices.Contains`.
- **Password-like fields are never trimmed or normalized** — state
  this in both the code and the doc comment.
- On "does it exist?" flows (login, add-member), validate only
  presence, not format: a value that cannot match stored data is
  already eliminated by the query. Document this deliberate decision
  in the doc comment.
- URL-safe identifier rules: `[a-z0-9-_]`, starts with a letter, no
  trailing separator, no consecutive separators.
- Secret values are generated with `crypto/rand` and stored only as
  their SHA-256 hash, so a database leak cannot be replayed against
  the API; justify this in the doc comment.

## Router

chi; domain routes live inside
`router.Route("/api/v1/<domain>", ...)`, standalone routes use
`router.With(middlewares.Auth).Get(...)`. Every protected route takes
`r.With(middlewares.Auth)`; the middleware is mounted per route, never
per group. `/health` always exists:

```go
	router.Route("/api/v1/things", func(r chi.Router) {
		r.With(middlewares.Auth).Post("/", things.CreateNewThingHandler)
		r.With(middlewares.Auth).Get("/", things.ListThingsHandler)
		r.With(middlewares.Auth).Delete("/{thing_id}", things.DeleteThingHandler)
	})

	router.Get("/health", func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		w.WriteHeader(http.StatusOK)

		json.NewEncoder(w).Encode(map[string]string{"status": "ok"})
	})
```

Each route group gets a one-line comment above it, e.g.
`// things`. Handler names end with the `Handler` suffix (legacy
handlers may lack it; keep the suffix in new code).

## Store / schema

- `store.DB *pgxpool.Pool` is global; `ConnectDatabase` connects,
  pings, and on failure closes the pool and leaves `DB = nil`.
- The schema bootstrap runs on every startup: everything is `CREATE
  TABLE IF NOT EXISTS` / `CREATE INDEX IF NOT EXISTS` / `ALTER TABLE
  ... ADD COLUMN IF NOT EXISTS`. Existing tables are never rewritten;
  when a column must be backfilled into older databases, add it as
  nullable (NOT NULL cannot be enforced on pre-existing rows) and
  enforce uniqueness with a separate index — justify the patch in a
  comment above the SQL.
- Schema conventions: `id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY
  KEY`, `TIMESTAMPTZ NOT NULL DEFAULT NOW()`, foreign keys with
  `REFERENCES ...(id) ON DELETE CASCADE`, pairs with `UNIQUE (a, b)`,
  conditional uniqueness via a partial unique index
  (`WHERE status = 'pending'`).
- Reusable pgx helpers live in one place (like `IsUniqueErr`); the
  SQLSTATE code lives in exactly one named constant.

## Middleware / context

Context keys are unexported struct types, immune to collisions:

```go
type userContextKey struct{}
```

Access goes through `UserFromContext(ctx) (User, bool)`; `ok == false`
means the request never passed through the middleware and the handler
answers 401. The middleware's 401 body is also
`{"error":"invalid_token"}` and **never explains why the token is
invalid** — mirroring how login treats bad credentials. JWT
verification pins the algorithm inside the keyfunc: only HMAC is
accepted, so a forged `alg` header cannot pass.

## Testing recipe

### Unit tests

- Tests use `t.Helper()` fixtures and named table cases:

```go
func TestValidateThingReq(t *testing.T) {
	cases := []struct {
		name    string
		req     models.CreateNewThingReq
		wantErr bool
	}{
		{"valid", models.CreateNewThingReq{Name: "X"}, false},
		{"missing name", models.CreateNewThingReq{}, true},
		{"name too long", models.CreateNewThingReq{Name: strings.Repeat("x", 81)}, true},
	}

	for _, c := range cases {
		t.Run(c.name, func(t *testing.T) {
			err := validateCreateNewThingReq(c.req)
			if (err != nil) != c.wantErr {
				t.Errorf("validateCreateNewThingReq(%+v) = %v, wantErr %v", c.req, err, c.wantErr)
			}
		})
	}
}
```

- Failure messages are short got/want patterns: `"x = %q, want %q"`,
  `"got %v, want nil"`. No external mock framework — only the standard
  `testing` package. Use `t.Setenv` when the environment matters.
- Model tests live in an external test package (`package models_test`,
  importing `internal/models`); handler and helper tests live in the
  internal test package and call unexported functions directly.

### Integration tests

- The file starts with the `//go:build integration` build tag and a
  comment block explaining how to run it:

```go
//go:build integration

// End-to-end tests for the thing handlers backed by a real PostgreSQL
// instance. Run them with:
//
//	TEST_DATABASE_URL=postgres://user:pass@localhost/myapp_test \
//		go test -tags=integration ./...
//
// The database must be disposable: every test truncates the tables.
```

- `TestMain`: when `TEST_DATABASE_URL` is set, it assigns the DSN and
  the test secret, calls `ConnectDatabase` + the schema bootstrap, and
  closes the pool at the end. Without a database the tests **skip**;
  they never fail.
- A `requireDB(t *testing.T)` helper: `t.Helper()`, `t.Skipf` when no
  DB is configured, otherwise `TRUNCATE ... RESTART IDENTITY CASCADE`
  on every table. Every integration test starts with it.
- Fixtures are inserted through the repository layer, never through
  handlers; bcrypt uses `bcrypt.MinCost` in fixtures for speed.
- Handler tests drive the real router via `httptest.NewRequest` +
  `httptest.NewRecorder`.

## JSON contract

- Model fields are snake_case (`created_at`, `organization_id`);
  auth token responses are camelCase (`accessToken`, `refreshToken`).
- Empty lists must encode as `[]`: initialize with `[]models.X{}` so
  JSON never emits `null`.
- No one-off response structs: single-use fields are dumped through
  `map[string]any`; recurring list types live in models.

## Import order

Two groups: the standard library first, a blank line, then project
and third-party imports **together**:

```go
import (
	"errors"
	"net/http"

	"github.com/yourorg/myapp/internal/repository"
	"github.com/go-chi/chi/v5"
)
```

## Pre-flight checklist

```sh
gofmt -l ./internal        # must print nothing
go vet ./internal/...
go build ./...
go test ./...

# integration (when a database is available):
TEST_DATABASE_URL=postgres://... go test -tags=integration ./...
```

Before finishing, also verify: does every exported identifier have a
doc comment? Are sentinel errors declared above the functions that
return them? Are all error messages in English? Are response bodies
`{"error": ...}`-shaped? Are missing resources indistinguishable from
denied access? Are password-like fields left untouched? Are
`http.Status*` constants used?
