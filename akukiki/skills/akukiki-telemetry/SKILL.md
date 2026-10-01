---
name: akukiki-telemetry
description: Connects a Node.js, Python or Go project to akukiki's early-access storage of logs, traces and errors, only with the person's consent, and answers "what broke?" from that data. Use when the user asks to use or install the akukiki-telemetry skill, to send their app's logs or errors to akukiki, or asks what broke in a project already connected.
metadata:
  version: "13"
---

# akukiki-telemetry

This guide was written by akukiki (https://akukiki.com) for you, the person's AI agent. In short: it connects the person's running app so that akukiki keeps its logs, request traces and errors, and then lets you answer "what broke?" from that data. The project changes only after the person's explicit yes to the exact changes you show. What leaves is the running app's own logs, traces and errors, with common secret formats masked — never the code or files. This guide never downloads or runs other files from akukiki.com; besides registering the project, its only request is a once-a-day look at the public list of versions.

You are helping someone who built a product with an AI agent and does not know DevOps terms. With their consent, connect their running app to akukiki so its logs, traces and errors are kept, and later use that data to tell them what broke and where in the code.

## Before you start — is this the latest version?

Once a day, compare this file's version (`metadata.version` above) with the site:

```bash
[ -n "$(find ~/.config/akukiki/akukiki-telemetry.checked -mtime -1 2>/dev/null)" ] && echo checked-today || curl -fsS https://akukiki.com/.well-known/skills/index.json
```

- `checked-today`, or no answer (no network, an error, the command refused): go on with this file without a word — never ask the person about the version check, and do not retry it in this conversation.
- An answer where you can read the `version` of `akukiki-telemetry`: mark the check done with `mkdir -p ~/.config/akukiki && touch ~/.config/akukiki/akukiki-telemetry.checked`. If that version is higher than this file's, tell the person in one sentence that a newer version of this guide is out and how to get it, then go on with this file. If it was installed with `npx skills add bookops/skills`: `npx skills update` in a terminal. If it is the Claude Code plugin: update the akukiki plugin — in a terminal, `claude plugin marketplace update bookops`, then `claude plugin update akukiki@bookops`.

## Rules

- Nothing happens without an explicit yes. Ask twice: first whether data may be sent to akukiki at all (step 1), then whether you may make the exact changes you list (step 4). No or silence means you stop and change nothing.
- Answer in the user's language — every message and every question. Write the questions yourself; never paste English sentences from this skill into a conversation in another language.
- Use plain words: "your app will send its error messages", not "OTLP exporter".
- Tokens:
  - the write token (`ak_w_…`) goes only into the environment of the production app. You never commit it to git and never put it in code that runs in the browser;
  - the read token (`ak_r_…`) stays on this machine in `~/.config/akukiki/<project id>.token` with `chmod 600`, outside the repository. You never show it in chat.
- Never send code or file contents. The only things that leave this machine are the registration request and, after setup, what the running app itself sends.

## Step 0 — Already connected?

If the repository has a `.akukiki-project` file and `~/.config/akukiki/$(cat .akukiki-project).token` exists, this project is connected: go to step 6 to answer questions, or step 7 if a token leaked. Otherwise start at step 1.

## Step 1 — Tell, then ask

Explain in plain words, in the user's language, what will leave and why:

- every request the app answers (HTTP and gRPC): when it came, how long it took, whether it failed — so they see when their services get slower or start failing;
- the lines the app writes to its log — to see what happened before an error;
- when the app crashes or a request fails with an error: where and why;
- for Node.js and Python, also a few basic numbers such as memory use;
- what is masked: common formats of emails, tokens, keys, card numbers and passwords are replaced with `[MASKED]` before anything is stored — not every possible secret, so lines that print secrets should go (step 2).

Nothing disappears from the app's console. What does not leave: the code and the files.

Then say where and for how long: on akukiki's server in Uzbekistan; logs, traces and metrics 7 days, error groups 30 days; up to 50 MB a day, free in early access; the full terms are on akukiki.com/<lang>/privacy. And how to stop and delete: remove the changes (step 4 says how) and write to the address on that privacy page.

Do not list the code changes here: step 4 shows them before anything changes.

Ask whether they want this. Go on only after an explicit yes.

## Step 2 — Look at the project

1. Stack. Supported: Node.js, Python and Go apps that run as their own server process (on a server, in a container, on a PaaS) — not functions on a serverless platform. Node.js and Python are connected by changing how the app starts; Go by a small change in its code, and you say so before step 3. For any other stack say it is not supported yet, and stop without changing anything.
2. Secrets in logs. Search the code for logging calls that print environment variables, request headers or bodies, passwords, tokens or keys (for example `console.log(process.env`, `print(request.headers`, `logger.info(password`). Show each place you find, file and line, and explain that such lines would send secrets out; the masking catches common formats, not all. Offer to remove them as part of step 4.
3. Where the production app gets its environment: a systemd unit (`Environment=` or `EnvironmentFile=`), a `.env` file on the server that is not in git, a Docker Compose file on the server, or a hosting dashboard. If you cannot reach production from here, you will give the person the exact lines to add there.

## Step 3 — Register

Ask for their email if you don't know it. Then register so that the answer, which holds both tokens, goes straight into a private file and never shows in any output:

```bash
(umask 077; mkdir -p ~/.config/akukiki && curl -sS -o ~/.config/akukiki/registration.json -w '%{http_code}\n' \
  -X POST https://otel.akukiki.com/v1/projects -H 'Content-Type: application/json' \
  -d '{"email": "<email>", "name": "<short project name>", "lang": "<en | pt | ru | uz>", "agent": "<claude-code | hermes | other>", "code": "<the referral code from the person's message, if it had 'code X', 'código X', 'код X' or 'kod X' — just X; otherwise empty>"}')
```

- 201: take what you need from the file with `sed`, never by printing it:

  ```bash
  P=$(sed -n 's/.*"project_id":"\([^"]*\)".*/\1/p' ~/.config/akukiki/registration.json)
  [ -n "$P" ] || { echo "could not read project_id from the registration answer" >&2; exit 1; }
  (umask 077; sed -n 's/.*"read_token":"\([^"]*\)".*/\1/p' ~/.config/akukiki/registration.json > ~/.config/akukiki/$P.token)
  printf '%s\n' "$P" > .akukiki-project
  ```

  `.akukiki-project` holds only the project id, which is not a secret: commit it — it tells step 0 which token belongs to this project. The write token stays in `registration.json` until step 4 puts it into the production environment. Tell the person to open the email from akukiki and click the link: nothing is accepted until they do.
- 400: fix the field named in the answer and ask again before resending.
- 403 "no seats left": early access is full; tell them and stop.
- 429 or 5xx: tell them it did not go through and suggest trying later.

On anything but 201 delete `~/.config/akukiki/registration.json`.

## Step 4 — Show the changes, then make them

List every change before touching anything.

Node.js:

- install in the project: `npm install @opentelemetry/api @opentelemetry/auto-instrumentations-node`
- unless the app already handles SIGTERM itself (`git grep -n SIGTERM` in its code — a plain `grep -rn` also matches `node_modules`, where many packages mention SIGTERM, and would wrongly say the app handles it), add a file `otel-exit.cjs`:

  ```js
  // Added by akukiki-telemetry. The OpenTelemetry register flushes data on SIGTERM but does not stop the app;
  // this ends the process 3 s later, as SIGTERM did before. Not needed if the app handles SIGTERM itself.
  process.on("SIGTERM", () => setTimeout(() => process.exit(0), 3000).unref());
  ```

  Explain why in plain words: without it a restart by systemd, pm2 or Docker hangs until the process is killed, and telemetry silently stops.
- start the app with both preloads, using the *absolute* path to `otel-exit.cjs` — resolve it when you write the line, e.g. `$(pwd)/otel-exit.cjs`, and put that resolved path in the file, not the `$(pwd)` expression itself: a relative `./otel-exit.cjs` breaks under systemd, which usually runs the unit without a `WorkingDirectory`. `node --require @opentelemetry/auto-instrumentations-node/register --require <absolute path>/otel-exit.cjs server.js` — in the start command, or as `NODE_OPTIONS` where the environment is set before node starts (systemd `Environment=`, Docker `environment:`). `NODE_OPTIONS` in a `.env` file that node reads with `--env-file` does not work: it is read too late.

Python (inside the app's virtualenv, if it has one):

- install: `pip install opentelemetry-distro opentelemetry-exporter-otlp`, then `opentelemetry-bootstrap -a install`
- start the app through `opentelemetry-instrument`, for example `opentelemetry-instrument gunicorn app:app`
- add `OTEL_PYTHON_LOGGING_AUTO_INSTRUMENTATION_ENABLED=true` to the environment

Go (there is no preload, so the change is in the code):

1. Remember the `go` line of `go.mod`. Install the modules with the command below — these exact versions are tested together. If the `go` line changed, the modules need a newer Go than the project uses: put `go.mod` and `go.sum` back as they were, tell the person that updating Go is a separate job, and stop without changing anything else.
2. Add the file `akukiki_otel.go` below, unchanged.
3. In `main`, after the app sets up `log` (`log.SetFlags`, `log.SetOutput`) — or at its start if it does not: `flush := akukikiStart(true)` when `main` itself waits for SIGTERM (`signal.Notify`), then call `flush()` right after the signal arrives; otherwise `akukikiStart(false)`, and it handles SIGTERM itself.
4. Wrap the handler every HTTP server serves: `http.ListenAndServe(addr, akukikiHTTP(handler))` or `http.Server{Handler: akukikiHTTP(handler)}`; a `nil` handler becomes `akukikiHTTP(http.DefaultServeMux)`.
5. Add the options to every gRPC server: `grpc.NewServer(akukikiGRPC()...)`, or `grpc.NewServer(append(opts, akukikiGRPC()...)...)` when it already has options.
6. Run `go mod tidy`, then `go build -o /dev/null ./...` — it leaves no binary behind. If it fails, put every file back as it was and say the change did not work.

Tell the person what this connects and what it does not: the standard `log` output, every HTTP and gRPC request, and panics in them. It does not connect panics in background goroutines or other loggers (zap, zerolog, logrus, `log.New(...)`). The console output stays exactly as it was. The change reaches production after their next deploy, as usual.

<!-- akukiki-go-modules-start -->
```bash
go get go.opentelemetry.io/otel@v1.46.0 go.opentelemetry.io/otel/sdk@v1.46.0 go.opentelemetry.io/otel/log@v0.22.0 go.opentelemetry.io/otel/sdk/log@v0.22.0 go.opentelemetry.io/contrib/exporters/autoexport@v0.71.0 go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp@v0.71.0 go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc@v0.71.0
```
<!-- akukiki-go-modules-end -->

`akukiki_otel.go`, next to `main.go`, in package `main`:

<!-- akukiki-go-start -->
```go
// Added by akukiki-telemetry: sends this app's logs, request traces and errors to akukiki.
// Nothing is sent unless OTEL_EXPORTER_OTLP_ENDPOINT is set. To remove: delete this file and the
// akukiki… calls in main, then run go mod tidy.
package main

import (
	"context"
	"fmt"
	"io"
	"log"
	"net/http"
	"os"
	"os/signal"
	"runtime/debug"
	"strings"
	"sync"
	"syscall"
	"time"

	"go.opentelemetry.io/contrib/exporters/autoexport"
	"go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc"
	"go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"
	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/attribute"
	otellog "go.opentelemetry.io/otel/log"
	"go.opentelemetry.io/otel/propagation"
	sdklog "go.opentelemetry.io/otel/sdk/log"
	sdktrace "go.opentelemetry.io/otel/sdk/trace"
	"google.golang.org/grpc"
)

var (
	akukikiTraces *sdktrace.TracerProvider
	akukikiLogs   *sdklog.LoggerProvider
	akukikiLogger otellog.Logger
)

// akukikiStart connects the app and returns what sends the rest before it exits. Call it at the start of
// main, after log.SetFlags and log.SetOutput. appHandlesSIGTERM: main waits for SIGTERM itself and calls the
// returned function; otherwise akukikiStart does it and exits as SIGTERM would.
func akukikiStart(appHandlesSIGTERM bool) (flush func()) {
	if os.Getenv("OTEL_EXPORTER_OTLP_ENDPOINT") == "" {
		return func() {}
	}
	ctx := context.Background()
	spans, err := autoexport.NewSpanExporter(ctx)
	if err != nil {
		log.Printf("akukiki: not connected: %v", err)
		return func() {}
	}
	logs, err := autoexport.NewLogExporter(ctx)
	if err != nil {
		log.Printf("akukiki: not connected: %v", err)
		return func() {}
	}
	akukikiTraces = sdktrace.NewTracerProvider(sdktrace.WithBatcher(spans))
	akukikiLogs = sdklog.NewLoggerProvider(sdklog.WithProcessor(sdklog.NewBatchProcessor(logs)))
	akukikiLogger = akukikiLogs.Logger("akukiki")
	otel.SetTracerProvider(akukikiTraces)
	otel.SetTextMapPropagator(propagation.TraceContext{})
	// The console keeps the same lines; each one is also sent as a log record.
	log.SetOutput(io.MultiWriter(log.Writer(), akukikiLogWriter{}))

	var once sync.Once
	flush = func() {
		once.Do(func() {
			ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
			defer cancel()
			akukikiTraces.Shutdown(ctx)
			akukikiLogs.Shutdown(ctx)
		})
	}
	if !appHandlesSIGTERM {
		go func() {
			stop := make(chan os.Signal, 1)
			signal.Notify(stop, syscall.SIGTERM)
			<-stop
			flush()
			// Die by SIGTERM as before, not by an exit code: systemd counts only the signal as a clean stop.
			signal.Reset(syscall.SIGTERM)
			if self, err := os.FindProcess(os.Getpid()); err == nil {
				self.Signal(syscall.SIGTERM) // on Windows this is a no-op, and SIGTERM never comes there anyway
			}
		}()
	}
	return flush
}

type akukikiLogWriter struct{}

func (akukikiLogWriter) Write(p []byte) (int, error) {
	var r otellog.Record
	r.SetTimestamp(time.Now())
	r.SetSeverity(otellog.SeverityInfo)
	r.SetBody(attribute.StringValue(strings.TrimRight(string(p), "\n")))
	akukikiLogger.Emit(context.Background(), r)
	return len(p), nil
}

// akukikiHTTP wraps the app's HTTP handler: one trace per request, and panics recorded.
func akukikiHTTP(h http.Handler) http.Handler {
	if akukikiTraces == nil {
		return h
	}
	return otelhttp.NewHandler(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		defer akukikiRecord(r.Context())
		h.ServeHTTP(w, r)
	}), "http", otelhttp.WithSpanNameFormatter(func(_ string, r *http.Request) string { return r.Method + " " + r.URL.Path }))
}

// akukikiGRPC returns the options for grpc.NewServer: one trace per call, and panics recorded.
func akukikiGRPC() []grpc.ServerOption {
	if akukikiTraces == nil {
		return nil
	}
	return []grpc.ServerOption{
		grpc.StatsHandler(otelgrpc.NewServerHandler()),
		grpc.ChainUnaryInterceptor(func(ctx context.Context, req any, _ *grpc.UnaryServerInfo, h grpc.UnaryHandler) (any, error) {
			defer akukikiRecord(ctx)
			return h(ctx, req)
		}),
		grpc.ChainStreamInterceptor(func(srv any, ss grpc.ServerStream, _ *grpc.StreamServerInfo, h grpc.StreamHandler) error {
			defer akukikiRecord(ss.Context())
			return h(srv, ss)
		}),
	}
}

// akukikiRecord sends a panic with its stack, then lets it go on exactly as it would without akukiki.
func akukikiRecord(ctx context.Context) {
	v := recover()
	if v == nil {
		return
	}
	if v != http.ErrAbortHandler {
		var r otellog.Record
		r.SetTimestamp(time.Now())
		r.SetSeverity(otellog.SeverityError)
		r.SetBody(attribute.StringValue(fmt.Sprintf("panic: %v", v)))
		r.AddAttributes(
			attribute.String("exception.type", fmt.Sprintf("%T", v)),
			attribute.String("exception.message", fmt.Sprint(v)),
			attribute.String("exception.stacktrace", string(debug.Stack())),
		)
		akukikiLogger.Emit(ctx, r)
		flushCtx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
		akukikiLogs.ForceFlush(flushCtx)
		cancel()
	}
	panic(v)
}
```
<!-- akukiki-go-end -->

All three, in the production environment (never in a file under git):

```
OTEL_SERVICE_NAME=<project name>
OTEL_EXPORTER_OTLP_ENDPOINT=https://otel.akukiki.com
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
OTEL_EXPORTER_OTLP_HEADERS=Authorization=Bearer%20<the write token from registration.json>
OTEL_TRACES_EXPORTER=otlp
OTEL_LOGS_EXPORTER=otlp
OTEL_METRICS_EXPORTER=otlp
```

Write the token line without printing the token, for example into a local env file. First make sure git ignores that file — `git check-ignore -q .env` — and if it does not, add `.env` to `.gitignore` as part of the changes you show. Remove any existing `OTEL_EXPORTER_OTLP_HEADERS=` line before adding the new one, so a later re-run (token rotation, step 7) replaces it instead of leaving a second, dead line:

```bash
sed -i.bak '/^OTEL_EXPORTER_OTLP_HEADERS=/d' .env 2>/dev/null; rm -f .env.bak
sed -n 's/.*"write_token":"\([^"]*\)".*/OTEL_EXPORTER_OTLP_HEADERS=Authorization=Bearer%20\1/p' ~/.config/akukiki/registration.json >> .env
```

If production is out of reach, the person has to paste that line there: say so, and show it only then.

Plus the removal of any secret-printing lines from step 2 the person agreed to.

If the answer is no: delete `~/.config/akukiki/$(cat .akukiki-project).token` first, then `.akukiki-project` and `~/.config/akukiki/registration.json`, and stop — otherwise step 0 would treat this project as already connected.

Wait for an explicit yes, then make exactly these changes and nothing else. Restart the app and check that it starts and answers as before. If it does not, roll back at once. Then:

- check that no token is in git or in a file git would take: `git grep --untracked -nE 'ak_[wr]_[A-Z2-7]{20}'` must find nothing (ignored files such as `.env` are skipped);
- delete `~/.config/akukiki/registration.json`.

Tell the person how to roll back: remove the packages, `otel-exit.cjs` and the `--require` or `opentelemetry-instrument` part of the start command, delete the `OTEL_…` lines from the environment, restart. For Go: delete `akukiki_otel.go` and the `akukiki…` calls in `main`, run `go mod tidy`, deploy as usual.

## Step 5 — Check that data arrives

When they have clicked the link and the app has run a few minutes:

```bash
curl -sS https://otel.akukiki.com/api/v1/usage -H "Authorization: Bearer $(cat ~/.config/akukiki/$(cat .akukiki-project).token)"
```

- `status` is `active` and `today` has bytes: tell them it works, and that they can see their projects, errors and usage themselves in the cabinet at https://my.akukiki.com — they sign in with the project's email and a code from the mail.
- 403 "confirm your email": remind them about the link.
- Nothing after 5 minutes: check that the app was restarted with the new environment, on the right server, that the server may make outgoing HTTPS requests, and look in the app's own output for 401 answers (wrong token).

## Step 6 — "What broke?"

```bash
curl -sS "https://otel.akukiki.com/api/v1/errors?since=24h" -H "Authorization: Bearer $(cat ~/.config/akukiki/$(cat .akukiki-project).token)"
```

Use the period the person asks about (`since=1h`, `7d` is written `168h`). For the newest groups, say in plain words what went wrong, how many times and since when. Find the place in the local code from the stack: match the file name and function, as paths on the server may differ. Show the file and line and what to change. A group of type `HTTP 500` without a stack is an answer the app sent after catching an error itself: find the route and look at what it catches.

If the group has a `trace_id`, look at the request it happened in: `GET https://otel.akukiki.com/api/v1/traces/<trace_id>` (slow steps, a failing call to another service). For context, search the logs: `GET https://otel.akukiki.com/api/v1/logs?q=<text>&since=1h` (at most 7 days back). Use the same `Authorization` header. Don't dump raw stacks or logs on the person; explain.

## Step 7 — A token leaked

If `git grep --untracked -nE 'ak_[wr]_[A-Z2-7]{20}'` finds a token in a file under git, if one is in code that runs in the browser, or the person says it leaked: tell them, and offer a replacement.

For a leaked write token, ask whether the new one can go into production right now:

- If yes, or if it's a leaked read token instead (it never touches production): ask for one yes for both steps together — the replacement and the edit of the production environment — because the old token stops at once. Replace and edit straight away with `"urgent": true`.
- If not: use `"urgent": false` instead. The old token keeps working for 24 hours, so telemetry does not go dark while they get to it — damage from a leaked write token is bounded by the daily quota either way.

```bash
(umask 077; curl -sS -o ~/.config/akukiki/rotation.json -w '%{http_code}\n' -X POST https://otel.akukiki.com/api/v1/tokens \
  -H "Authorization: Bearer $(cat ~/.config/akukiki/$(cat .akukiki-project).token)" \
  -H 'Content-Type: application/json' -d '{"kind": "write", "urgent": true}')
```

Put the new token (`"token"` in `rotation.json`) into `OTEL_EXPORTER_OTLP_HEADERS` the same way as in step 4, restart the app, remove the old token from the file or browser code where it was found, check `git grep --untracked -nE 'ak_[wr]_[A-Z2-7]{20}'` again and delete `rotation.json`. A token in git history is dead after the replacement; rewriting history is the person's choice. A leaked read token: the same request with `"kind": "read"`, then `(umask 077; sed -n 's/.*"token":"\([^"]*\)".*/\1/p' ~/.config/akukiki/rotation.json > ~/.config/akukiki/$(cat .akukiki-project).token)`.

## Step 8 — Delete the project

If the person asks to delete the project from akukiki, start the deletion:

```bash
curl -sS -X DELETE https://otel.akukiki.com/api/v1/project -H "Authorization: Bearer $(cat ~/.config/akukiki/$(cat .akukiki-project).token)"
```

On `202` tell them: an email with a link went to the project's address; nothing is deleted until they open it, sign in to the cabinet and confirm with the project's name. Never say the project is already deleted. After they confirm, offer to roll back the changes from step 4 and delete `~/.config/akukiki/$(cat .akukiki-project).token` and `.akukiki-project`.
