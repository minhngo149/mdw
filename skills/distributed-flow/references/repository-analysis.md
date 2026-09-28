# Repository Analysis

This reference covers orienting in an unfamiliar repository and tracing one flow from its
trigger to its last side effect without reading the whole codebase.

## Contents

1. Orient: cheapest sources first
2. Find the entry point by its registration
3. Trace the path
4. Check reachability
5. Weigh the sources
6. Repository shapes

## 1. Orient: cheapest sources first

Stop orienting once you can name the process that owns the entry point and the infrastructure
clients it constructs. Everything else comes from tracing.

| Question | Look at |
|---|---|
| Language and framework | `go.mod`, `package.json`, `pom.xml`, `build.gradle(.kts)`, `pyproject.toml`, `requirements*.txt`, `Gemfile`, `Cargo.toml`, `composer.json`, `*.csproj`. The dependency list names the web framework, ORM, broker client, cache client, and HTTP client. |
| What runs (deployables) | `cmd/*/main.go`, `main.*`, `apps/*`, `services/*`, `if __name__ == "__main__"`, `Dockerfile*`, `docker-compose*.yml`, Kubernetes or Helm manifests, `Procfile`, `serverless.yml`, `Makefile`, `package.json` `scripts` |
| Infrastructure | compose services, `.env.example`, `config/*`, `application.yml`, `settings.py`, and the client constructors in `main` or bootstrap code |
| Wiring | `main`, bootstrap, and the DI container: which implementation is constructed and passed to what |
| Conventions | how one existing feature is laid out. Other features usually follow the same layout. |

Useful first searches (use the Grep and Glob tools, or `rg`):

```bash
git ls-files | grep -Ei '(^|/)(main\.[a-z]+|Dockerfile[^/]*|docker-compose[^/]*|Procfile|serverless\.ya?ml)$'
git ls-files | grep -Ei '\.(proto|graphql|gql)$|/migrations?/|/k8s/|/helm/|/deploy/'
rg -l --no-messages -i 'amqp|rabbit|kafka|nats|sqs|pubsub|redis|celery|sidekiq|bullmq|asynq'
```

Budget: read the files on the flow's path in full, and skim everything else with search.

## 2. Find the entry point by its registration

Search for how the trigger is **registered**, then open the handler it points to.

| Trigger | Registration patterns |
|---|---|
| HTTP / REST | Go: `.POST(`, `.Post(`, `HandleFunc(`, `mux.Handle("POST /path"`. Node: `router.post(`, `app.post(`, `@Post(` (NestJS), `fastify.post(`, Next.js `app/**/route.ts` exporting `POST`. JVM: `@PostMapping`, `@RequestMapping(method = POST)`, JAX-RS `@POST @Path`. Python: `@app.post` (FastAPI), `methods=["POST"]` (Flask), `urls.py` and `@api_view` (Django/DRF). Ruby: `config/routes.rb` to `Controller#action`. PHP: `Route::post(`. .NET: `[HttpPost]`, `app.MapPost(` |
| GraphQL | SDL `type Mutation` or `type Query` in `.graphql` files, resolver maps, gqlgen `*.resolvers.go`, `@Resolver`/`@Mutation` (NestJS), `@MutationMapping` (Spring), graphene or strawberry `Mutation` classes |
| gRPC | `.proto` `service X { rpc Y }`, then `RegisterXServer(`, `addService(`, `@GrpcService`, then the method implementation |
| CLI | cobra `&cobra.Command{Use:`, urfave/cli, click `@click.command`, argparse `add_parser`, commander `.command(`, Rake tasks, Django `BaseCommand`, Spring `CommandLineRunner` |
| Scheduled / cron | `cron.AddFunc(`, gocron, `@Scheduled(`, node-cron, agenda, Celery `beat_schedule`, sidekiq-cron, Quartz, Kubernetes `kind: CronJob`, crontab files, serverless `schedule:` or EventBridge rules |
| Webhook (inbound) | an HTTP route plus signature verification (`X-Hub-Signature`, `Stripe-Signature`, HMAC compare) plus a switch on event type |
| Message consumer | RabbitMQ: `Consume(`, `QueueBind(`, `@RabbitListener`, `channel.consume`. Kafka: `@KafkaListener`, sarama `ConsumerGroup`, kafka-go `NewReader`, kafkajs `consumer.run`, confluent `subscribe(`. NATS: `Subscribe(`, `QueueSubscribe(`, JetStream `PullSubscribe` or `Consumer(`. SQS: `ReceiveMessage` |
| Background job | Sidekiq `perform`, Celery `@task`/`@shared_task`, BullMQ `new Worker(`, asynq `HandleFunc`, Hangfire, and in-process `go func`, `@Async`, executors, `asyncio.create_task` |
| Internal business operation | find every caller of the business function, then each caller's entry point. One operation can have several triggers; list them and scope to the one requested. |

Routes, topics, queues, subjects, and job names are often constants or config keys. Search for
the literal string **and** the constant name, and resolve config keys to values in in-repo config
files.

## 3. Trace the path

Go depth-first along the flow. Read each function on the path in full and note every call that
leaves it. Keep a running hop list as you go, so you never need to re-read a file:

```text
n. <file> <Symbol>: <what it does>  [boundary? PROTOCOL MODE]
```

| Obstacle | Technique |
|---|---|
| Interface or abstract type | Find the implementations (`func (x *T) Method(`, `implements`, `class X(Base)`), then find which one this process's bootstrap or DI container wires in (wire, fx, dig, Spring `@Bean`/`@Primary`/`@Profile`, Nest providers, Guice, .NET `services.Add*`). One wired implementation is `CONFIRMED`. A choice made by runtime config is `INFERRED`; cite the config key. |
| Middleware, interceptors, guards, aspects | Read them in execution order. They can end the request (auth, rate limit) or change behavior (transaction per request, retry, idempotency). |
| Behavior-changing annotations | `@Transactional`, `@Retryable`, `@Async`, `@Cacheable`, `@CircuitBreaker`, `@TransactionalEventListener`, and Python or TS decorators. Read what they configure. |
| Hidden I/O | ORM lazy loading, model hooks and callbacks (`after_commit`, `@PrePersist`, GORM or Sequelize hooks), database triggers in migrations, listeners registered elsewhere |
| Dispatch by name | event buses, command buses, and strategy maps keyed by string. Find where the map is populated. |
| Generated code | proto stubs, ORM models, OpenAPI clients. Read the interface, not the generated internals. |
| Concurrency | `go`, executors, `Promise.all`, `errgroup`, `CompletableFuture`. These change mode and failure semantics (see `sync-flow.md`, `async-flow.md`). |
| Receiver in another repository | Stop there. Record the contract (URL and path, topic, payload type) and mark the receiver's internals `UNKNOWN`. |

Stop tracing a branch when it:

- reaches a datastore, broker, or external system (record the operation and go no further)
- reaches framework or library code (record the behavior as an `INFERRED` default)
- leaves the flow's business scope (generic logging, metrics)
- reaches the repository boundary

## 4. Check reachability

Before including code in the flow, confirm that it runs on this path. It must be registered or
called from the path, not guarded by a disabled flag or profile, not test-only, and not a TODO
stub. Code that is present but not wired, and that documentation describes as in use, is
**documentation drift**. Report it that way, not as part of the flow.

## 5. Weigh the sources

| Source | Weight |
|---|---|
| Production code on the path | Primary. Supports `CONFIRMED`. |
| In-repo config, migrations, schemas, IaC and deployment manifests | `CONFIRMED` for what they declare; deployed values may differ |
| Tests | Supporting evidence of intended behavior. A mock shows what the author expected a dependency to do, not what it does. |
| Library or framework defaults | `INFERRED`. State only defaults you are sure of; otherwise `UNKNOWN`. |
| README, ADRs, wikis, comments, commit messages | Leads to verify. If code contradicts them, record drift. |
| Names of directories, packages, classes, variables | Leads only |

## 6. Repository shapes

| Shape | What changes |
|---|---|
| Monolith | One process plus infrastructure. The value is in the in-process path, the transactions, and the failure handling. Do not promote modules to services. |
| Modular monolith | Modules may talk through in-process events or shared tables. It is still one process; check whether event dispatch is synchronous. |
| Monorepo with several deployables | Both ends of a call may be in the repo. Trace across and link the caller's client to the receiver's route or consumer. |
| One service of many (polyrepo) | Most receivers are external to the repo. Record contracts; mark their internals `UNKNOWN`. |
| Serverless | Each function is its own deployable. Triggers are declared in `serverless.yml`, SAM, CDK, or Terraform. |
