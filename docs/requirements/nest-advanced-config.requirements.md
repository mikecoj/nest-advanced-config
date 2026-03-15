# NestJS Advanced Config Module – Requirements Specification

## 1. Overview

This document defines the requirements for a **modern configuration module for NestJS** designed to replace or extend the capabilities of `@nestjs/config`.

The module must provide:

* **Full type safety**
* **Zod schema validation**
* **Strong developer experience**
* **High performance**
* **Security features**
* **Compatibility with DDD and Hexagonal Architecture**

The module will be designed as a **NestJS Dynamic Module** and support both **application-wide configuration** and **feature-level configuration**.

The configuration system must support:

* Environment variables
* Secret managers
* File-based configuration
* Programmatic loaders

The configuration must be **validated once during bootstrap** and treated as **immutable afterward**.

---

# 2. Design Principles

## 2.1 Type Safety

Configuration access must be fully typed using **TypeScript inference from Zod schemas**.

Example:

```ts
config.get('database.url')
```

Must infer:

```ts
string
```

The developer must never need to use:

```
as string
```

or any manual type assertions.

---

## 2.2 Single Source of Truth

All configuration definitions must be derived from a **Zod schema**.

The schema must define:

* type
* validation rules
* default values
* optional values
* transformations

Example:

```ts
z.object({
  url: z.string().url(),
  poolSize: z.number().default(10),
})
```

---

## 2.3 Immutable Configuration

Once validated, the configuration must be **deep frozen**.

The configuration must be:

* read-only
* not modifiable during runtime

This prevents accidental mutation and ensures predictable behavior.

---

## 2.4 Minimal Boilerplate

Configuration definition should require **very little code**.

Target developer experience:

```ts
defineConfig({
  namespace: 'database',
  schema: databaseSchema,
  load: ({ env }) => ({
    url: env.getString('DB_URL'),
    poolSize: env.getNumber('DB_POOL', 10),
  })
})
```

---

## 2.5 Domain-Oriented Configuration

The system must support **distributed configuration ownership**.

Example project structure:

```
src

modules
  auth
    auth.config.ts

  payments
    payments.config.ts

infrastructure
  database
    database.config.ts
```

Each domain module must own its configuration schema.

---

## 2.6 High Performance

The configuration module must:

* validate configuration **once during bootstrap**
* avoid runtime parsing
* use **O(1)** property access
* avoid repeated string path parsing

---

## 2.7 Security

The module must provide built-in mechanisms for:

* secret loading
* secret masking
* strict configuration validation
* safe config printing

---

# 3. Functional Requirements

## 3.1 Configuration Definition

Configuration definitions must be declared using a `defineConfig` helper.

### API

```ts
function defineConfig<
  Namespace extends string,
  Schema extends ZodSchema
>(options: ConfigDefinitionOptions<Namespace, Schema>): ConfigDefinition
```

### ConfigDefinitionOptions

```ts
interface ConfigDefinitionOptions<
  Namespace extends string,
  Schema extends ZodSchema
> {
  namespace: Namespace
  schema: Schema
  load?: ConfigLoader<Schema>
}
```

---

## 3.2 Namespace Support

Every configuration definition must belong to a **namespace**.

Example:

```ts
namespace: 'database'
```

Resulting configuration access:

```
database.url
database.poolSize
```

Namespaces must be **merged into the global configuration object**.

---

## 3.3 Configuration Loader

The configuration loader is responsible for mapping external sources into the internal configuration structure.

### Loader Signature

```ts
type ConfigLoader<Schema> = (
  ctx: LoadContext
) => Partial<z.infer<Schema>>
```

The loader executes during application bootstrap.

---

## 3.4 Load Context

The loader receives a context object containing configuration sources.

### API

```ts
interface LoadContext {
  env: EnvSource
  secrets: SecretSource
  files: FileSource
}
```

---

# 4. Environment Source API

The environment source must provide safe access to environment variables.

### Interface

```ts
interface EnvSource {

  getString(key: string, defaultValue?: string): string

  getNumber(key: string, defaultValue?: number): number

  getBoolean(key: string, defaultValue?: boolean): boolean

  getOptionalString(key: string): string | undefined

  getOptionalNumber(key: string): number | undefined

  getOptionalBoolean(key: string): boolean | undefined
}
```

### Responsibilities

The environment source must:

* parse values
* validate existence
* provide helpful error messages

---

# 5. Secret Source API

The configuration system must support retrieving secrets from secret management systems.

### Interface

```ts
interface SecretSource {

  get(key: string): Promise<string>

  getOptional(key: string): Promise<string | undefined>
}
```

Possible implementations:

* Azure Key Vault
* AWS Secrets Manager
* Hashicorp Vault
* Kubernetes secrets

---

# 6. File Source API

The module must support configuration from files.

### Interface

```ts
interface FileSource {

  json(path: string): unknown

  yaml(path: string): unknown

  text(path: string): string
}
```

---

# 7. ConfigModule Dynamic Module

The configuration system must be implemented as a **NestJS dynamic module**.

### API

```ts
ConfigModule.forRoot(options: ConfigModuleOptions)
```

### ConfigModuleOptions

```ts
interface ConfigModuleOptions {

  schemas: ConfigDefinition[]

  loaders?: SourceLoader[]

  strict?: boolean

  cache?: boolean
}
```

---

## 7.1 Feature Module Registration

Modules must be able to register configuration locally.

### API

```ts
ConfigModule.forFeature(configDefinition)
```

Example:

```ts
@Module({
  imports: [ConfigModule.forFeature(authConfig)]
})
export class AuthModule {}
```

---

# 8. ConfigService API

The configuration must be accessible via `ConfigService`.

### Constructor

```ts
class ConfigService<TConfig>
```

---

## 8.1 Get Configuration Value

```ts
get<K extends Path<TConfig>>(key: K): PathValue<TConfig, K>
```

Example:

```ts
config.get('database.url')
```

Return type:

```
string
```

---

## 8.2 Namespace Access

Developers must be able to retrieve entire namespaces.

### API

```ts
namespace<K extends keyof TConfig>(name: K): TConfig[K]
```

Example:

```ts
const db = config.namespace('database')

db.url
db.poolSize
```

---

# 9. Validation Flow

Bootstrap sequence:

1. Load environment variables
2. Initialize source loaders
3. Execute configuration loaders
4. Merge configuration namespaces
5. Validate configuration with Zod
6. Apply default values
7. Deep freeze configuration
8. Register ConfigService

---

# 10. Strict Mode

Strict mode ensures only declared configuration variables are allowed.

### Behavior

If enabled:

```
strict: true
```

The system must throw an error when:

* unknown environment variables exist
* required configuration values are missing

---

# 11. Configuration Freezing

After validation the configuration must be frozen.

### Implementation

Use:

```ts
Object.freeze
```

or recursive deep freeze.

This prevents:

* runtime mutation
* accidental overrides

---

# 12. Logging and Debugging

The module must support safe configuration inspection.

### API

```ts
config.printSafe()
```

Secrets must be masked.

Example output:

```
database.url = ********
database.poolSize = 10
```

---

# 13. Testing Support

Tests must be able to override configuration values.

### API

```ts
ConfigModule.forRoot({
  schemas: [...],
  overrides: {
    database: {
      url: "test"
    }
  }
})
```

---

# 14. Performance Requirements

The module must satisfy the following:

| Requirement       | Target        |
| ----------------- | ------------- |
| Config validation | once per boot |
| Config lookup     | O(1)          |
| Runtime parsing   | none          |
| Memory overhead   | minimal       |

---

# 15. Developer Experience Requirements

The module must provide:

* automatic type inference
* minimal configuration code
* clear error messages
* predictable configuration structure
* IDE autocomplete support

---

# 16. Error Handling

The system must provide detailed error messages for:

* missing environment variables
* invalid schema values
* loader failures
* secret retrieval failures

Errors must include:

* namespace
* field name
* source variable

Example:

```
Config validation failed:

database.url
Expected: valid URL
Received: "not-a-url"

Source: DB_URL
```

---

# 17. Example Configuration Definition

```ts
export const databaseConfig = defineConfig({

  namespace: 'database',

  schema: z.object({
    url: z.string().url(),
    poolSize: z.number().default(10),
    ssl: z.boolean().default(false),
  }),

  load: ({ env }) => ({
    url: env.getString('DB_URL'),
    poolSize: env.getNumber('DB_POOL', 10),
    ssl: env.getBoolean('DB_SSL', false),
  })
})
```

---

# 18. Example Usage

```ts
@Module({
  imports: [
    ConfigModule.forRoot({
      schemas: [
        databaseConfig,
        authConfig,
        appConfig
      ]
    })
  ]
})
export class AppModule {}
```

Service usage:

```ts
@Injectable()
export class DatabaseService {

  constructor(private config: ConfigService) {}

  connect() {
    const url = this.config.get('database.url')
  }
}
```

---

# 19. Future Extensions

Possible future features include:

* configuration hot reload
* configuration snapshots
* distributed configuration sources
* secret rotation support
* CLI configuration validation
* configuration schema generation

---

# 20. Summary

The proposed configuration module provides:

* strict validation
* full type safety
* high performance
* modular configuration ownership
* security features
* excellent developer experience

It is designed to support **large-scale NestJS applications using Clean Architecture, Hexagonal Architecture, and Domain-Driven Design**.
