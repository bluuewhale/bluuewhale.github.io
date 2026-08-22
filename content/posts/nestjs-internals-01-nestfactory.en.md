+++
title = '[Diving into NestJS] 01. NestFactory'
date = '2022-11-15T15:17:34+09:00'
draft = false
translationKey = 'nestjs-internals-01-nestfactory'
slug = 'nestjs-internals-01-nestfactory-en'
aliases = ['/posts/nestjs-internals-01-nestfactory-en/']
description = 'A walk through NestFactory.create(), tracing how the HttpAdapter, ApplicationConfig, and NestContainer come together to build a NestApplication.'
tags = ['NestJS', 'TypeScript', 'DI', 'Node.js']
categories = ['NestJS']
+++

Hi everyone. Things have kept me busy lately, so it's been a while since my last post.

I want to start a series called "Diving into NestJS." I picked up NestJS recently for work, and I kept running into friction because I didn't understand its core well enough. So I decided to spend some time studying the framework from the inside out, mostly for my own benefit.

For this first post, let's look at `NestFactory`, the entry point for every NestJS application.

## NestFactory
Open the NestJS docs and the very first tutorial you hit shows code like this:

```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(3000);
}

bootstrap();
```

From this, you can tell that `NestFactory` reads `AppModule` and builds a `NestApplication` instance, the object that holds all the logic for handling incoming requests. This post's goal is to understand exactly what happens inside `NestFactory.create()`.

## NestFactory.create()

Here's how `NestFactory.create()` is actually implemented. Let's go through it line by line.
```typescript
// packages/core/nest-factory.ts
public async create<T extends INestApplication = INestApplication>(
    module: any,
    serverOrOptions?: AbstractHttpAdapter | NestApplicationOptions,
    options?: NestApplicationOptions,
  ): Promise<T> {
    const [httpServer, appOptions] = this.isHttpServer(serverOrOptions)
      ? [serverOrOptions, options]
      : [this.createHttpAdapter(), serverOrOptions];

    const applicationConfig = new ApplicationConfig();
    const container = new NestContainer(applicationConfig);
    this.setAbortOnError(serverOrOptions, options);
    this.registerLoggerConfiguration(appOptions);

    await this.initialize(module, container, applicationConfig, httpServer);

    const instance = new NestApplication(
      container,
      httpServer,
      applicationConfig,
      appOptions,
    );
    const target = this.createNestInstance(instance);
    return this.createAdapterProxy<T>(target, httpServer);
  }
```

## HttpAdapter
Look at the first three lines: `NestFactory` either reuses an `HttpAdapter` you passed in, or, in the common case, creates a fresh `HttpAdapter` instance by default.

```typescript
// packages/core/nest-factory.ts
export class NestFactory {
  public async create<T extends INestApplication = INestApplication>(
    module: any,
    serverOrOptions?: AbstractHttpAdapter | NestApplicationOptions,
    options?: NestApplicationOptions,
  ): Promise<T> {
    // =======================================================
    const [httpServer, appOptions] = this.isHttpServer(serverOrOptions)
      ? [serverOrOptions, options]
      : [this.createHttpAdapter(), serverOrOptions];
    // =======================================================

	const applicationConfig = new ApplicationConfig();
    const container = new NestContainer(applicationConfig);  
    this.setAbortOnError(serverOrOptions, options);
    this.registerLoggerConfiguration(appOptions);

	await this.initialize(module, container, applicationConfig, httpServer);

    const instance = new NestApplication(
      container,
      httpServer,
      applicationConfig,
      appOptions,
    );
    const target = this.createNestInstance(instance);
    return this.createAdapterProxy<T>(target, httpServer);
  }
}
```

`HttpAdapter` is a class that implements `AbstractHttpAdapter`, an interface defined in NestJS Core. `AbstractHttpAdapter` is an abstract class that implements part of `HttpServer`, the interface that describes everything needed to handle an HTTP request.

```typescript
// packages/core/adapters/http-adapters.ts
export abstract class AbstractHttpAdapter<
  TServer = any,
  TRequest = any,
  TResponse = any,
> implements HttpServer<TRequest, TResponse>
{
  protected httpServer: TServer;

  constructor(protected instance?: any) {}

  public use(...args: any[]) {
    return this.instance.use(...args);
  }

  public get(...args: any[]) {
    return this.instance.get(...args);
  }

  public post(...args: any[]) {
    return this.instance.post(...args);
  } 
  ...
  abstract close();
  abstract status(response: any, statusCode: number);
  abstract reply(response: any, body: any, statusCode?: number);
  abstract end(response: any, message?: string);
  abstract render(response: any, view: string, options: any);
  abstract redirect(response: any, statusCode: number, url: string);
}
```

Here's something interesting: `AbstractHttpAdapter` implements some of these methods (`use`, `get`, `post`) by just forwarding to an `instance` typed as `any`. You've probably already noticed those method names look familiar.

They should. They're the exact method names `express` uses for registering middleware and handlers. `NestFactory` actually depends on `express` by default, using `ExpressAdapter`, an implementation of `AbstractHttpAdapter` built on top of it. From this angle, NestJS is basically an `express` wrapper framework. If you already know `express`, that should feel reassuring.

```typescript
// packages/platform-express/adapters/express-adapter.ts
export class ExpressAdapter extends AbstractHttpAdapter {
  ...
  constructor(instance?: any) {
    super(instance || express()); // << express!!
  }
}
```

```typescript
// packages/core/nest-factory.ts
export class NestFactoryStatic {
  ...
  private createHttpAdapter<T = any>(httpServer?: T): AbstractHttpAdapter {
    const { ExpressAdapter } = loadAdapter(
      '@nestjs/platform-express',
      'HTTP',
      () => require('@nestjs/platform-express'),
    );
    return new ExpressAdapter(httpServer);
  }
}
```

## ApplicationConfig
Moving to the next line: `NestFactory` creates an `ApplicationConfig` instance.

```typescript
// packages/core/nest-factory.ts
export class NestFactory {
  public async create<T extends INestApplication = INestApplication>(
    module: any,
    serverOrOptions?: AbstractHttpAdapter | NestApplicationOptions,
    options?: NestApplicationOptions,
  ): Promise<T> {
    const [httpServer, appOptions] = this.isHttpServer(serverOrOptions)
      ? [serverOrOptions, options]
      : [this.createHttpAdapter(), serverOrOptions];

	// =======================================================
    const applicationConfig = new ApplicationConfig();
    // =======================================================

    const container = new NestContainer(applicationConfig);  
    this.setAbortOnError(serverOrOptions, options);
    this.registerLoggerConfiguration(appOptions);

	await this.initialize(module, container, applicationConfig, httpServer);

    const instance = new NestApplication(
      container,
      httpServer,
      applicationConfig,
      appOptions,
    );
    const target = this.createNestInstance(instance);
    return this.createAdapterProxy<T>(target, httpServer);
  }
}
```

`ApplicationConfig` is a fairly plain data object that stores the global middleware used across the NestJS app. Most of its fields start out empty, but as `NestApplication` gets built, every global middleware you register ends up recorded in this object.

```typescript
// packages/core/application-config.ts
export class ApplicationConfig {
  private globalPrefix = '';
  private globalPrefixOptions: GlobalPrefixOptions<ExcludeRouteMetadata> = {};
  private globalPipes: Array<PipeTransform> = [];
  private globalFilters: Array<ExceptionFilter> = [];
  private globalInterceptors: Array<NestInterceptor> = [];
  private globalGuards: Array<CanActivate> = [];
  private versioningOptions: VersioningOptions;
  ...
}
```

## NestContainer

Next, `NestFactory` creates a `NestContainer`, the object that plays the central role in NestJS's dependency injection (DI). `NestContainer` holds every `Module` that `NestApplication` needs to run.

```typescript
// packages/core/nest-factory.ts
export class NestFactory {
  public async create<T extends INestApplication = INestApplication>(
    module: any,
    serverOrOptions?: AbstractHttpAdapter | NestApplicationOptions,
    options?: NestApplicationOptions,
  ): Promise<T> {
    const [httpServer, appOptions] = this.isHttpServer(serverOrOptions)
      ? [serverOrOptions, options]
      : [this.createHttpAdapter(), serverOrOptions];

    const applicationConfig = new ApplicationConfig();

	// =======================================================
    const container = new NestContainer(applicationConfig);
	// =======================================================

    this.setAbortOnError(serverOrOptions, options);
    this.registerLoggerConfiguration(appOptions);

	await this.initialize(module, container, applicationConfig, httpServer);

    const instance = new NestApplication(
      container,
      httpServer,
      applicationConfig,
      appOptions,
    );
    const target = this.createNestInstance(instance);
    return this.createAdapterProxy<T>(target, httpServer);
  }
}
```

Internally, `NestContainer` holds classes like `Module`, `ModuleTokenFactory`, `ModuleCompiler`, and `ModulesContainer`. Modules get registered into it later, through `DependenciesScanner`, which I'll cover in more detail below.

```typescript
// packages/core/injector/factory.ts
export class NestContainer {
  private readonly globalModules = new Set<Module>();
  private readonly moduleTokenFactory = new ModuleTokenFactory();
  private readonly moduleCompiler = new ModuleCompiler(this.moduleTokenFactory);
  private readonly modules = new ModulesContainer();
  private readonly dynamicModulesMetadata = new Map<string, Partial<DynamicModule>>();
  ...
  
  public addProvider(
    provider: Provider,
    token: string,
  ): string | symbol | Function {
    const moduleRef = this.modules.get(token);
    if (!provider) {
      throw new CircularDependencyException(moduleRef?.metatype.name);
    }
    if (!moduleRef) {
      throw new UnknownModuleException();
    }
    return moduleRef.addProvider(provider);
  }
}
```

### Module

`Module` is the core object that defines every dependency (`Provider`, `Controller`, and so on) contained in a single module. Whatever you register with the `@Module` decorator, or as a `DynamicModule` via `forRoot()`, gets managed internally as a `Module` instance inside the framework.

```typescript
// packages/core/injector/module.ts
export class Module {
  private readonly _id: string;
  private readonly _imports = new Set<Module>();
  private readonly _providers = new Map<
    InstanceToken,
    InstanceWrapper<Injectable>
  >();
  private readonly _injectables = new Map<
    InstanceToken,
    InstanceWrapper<Injectable>
  >();
  private readonly _middlewares = new Map<
    InstanceToken,
    InstanceWrapper<Injectable>
  >();
  private readonly _controllers = new Map<
    InstanceToken,
    InstanceWrapper<Controller>
  >();
  ...
}
```

### ModuleTokenFactory

`ModuleTokenFactory` does one simple job: it generates the token (a hash) used as the key when registering and looking up modules.

```typescript
// packages/core/injector/module-token-factory.ts
export class ModuleTokenFactory {
  public create(
    metatype: Type<unknown>,
    dynamicModuleMetadata?: Partial<DynamicModule> | undefined,
  ): string {
    const moduleId = this.getModuleId(metatype);
    const opaqueToken = {
      id: moduleId,
      module: this.getModuleName(metatype),
      dynamic: this.getDynamicMetadataToken(dynamicModuleMetadata),
    };
    return hash(opaqueToken, { ignoreUnknown: true });
  }
  ...
}
```

### ModuleCompiler

`NestContainer` keeps a `DynamicModule`'s dependency information separate from the rest. `ModuleCompiler` is the utility class that handles parsing a `Module`'s data for that purpose.

```typescript
// packages/core/injector/compiler.ts
export class ModuleCompiler {
  public async compile(
    metatype: Type<any> | DynamicModule | Promise<DynamicModule>,
  ): Promise<ModuleFactory> {
    const { type, dynamicMetadata } = this.extractMetadata(await metatype);
    const token = this.moduleTokenFactory.create(type, dynamicMetadata);
    return { type, dynamicMetadata, token };
  }
  ...
}
```

### ModulesContainer

`ModulesContainer` is a fairly simple `Map<string, Module>` that stores every `Module`. It keys entries by the token generated in `ModuleTokenFactory`.

```typescript
// packages/core/injector/modules-container.ts
export class ModulesContainer extends Map<string, Module> {
  private readonly _applicationId = uuid();

  get applicationId(): string {
    return this._applicationId;
  }
}
```

## initialize()
After a bit more setup, `NestFactory` calls `initialize()`. Its core work splits into two parts: `DependenciesScanner` registering modules, and `InstanceLoader` creating dependency instances.

```typescript
// packages/core/nest-factory.ts
export class NestFactory{
  ...
  private async initialize(
    module: any,
    container: NestContainer,
    config = new ApplicationConfig(),
    httpServer: HttpServer = null,
      ) {
        const instanceLoader = new InstanceLoader(container);
        const metadataScanner = new MetadataScanner();
        const dependenciesScanner = new DependenciesScanner(
          container,
          metadataScanner,
          config,
        );
        container.setHttpAdapter(httpServer);

        const teardown = this.abortOnError === false ? rethrow : undefined;
        await httpServer?.init();
        try {
          this.logger.log(MESSAGES.APPLICATION_START);

          await ExceptionsZone.asyncRun(
            async () => {
              await dependenciesScanner.scan(module);
              await instanceLoader.createInstancesOfDependencies();
              dependenciesScanner.applyApplicationProviders();
            },
            teardown,
            this.autoFlushLogs,
          );
        } catch (e) {
          this.handleInitializationError(e);
        }
    }
}
```

### DependenciesScanner

`DependenciesScanner`'s main job is walking the module tree and registering each module into `NestContainer`. It also records the edges between modules, tracks metadata for each module's dependencies (`providers` and friends), and computes each module's depth, which `NestApplication` later needs to topologically sort modules for things like module lifecycle hooks.

```typescript
// packages/core/scanner.ts
export class DependenciesScanner {
  ...
  public async scan(module: Type<any>) {
    await this.registerCoreModule(); // register InternalCoreModule into NestContainer
    await this.scanForModules(module); // recursively walk the module tree and register every module into NestContainer
    await this.scanModulesForDependencies(); // parse metadata to add dependency info (imports, providers, controllers, exports) for each registered module
    this.calculateModulesDistance(); // walk the module tree and record depth, needed to topologically sort modules

    this.addScopedEnhancersMetadata(); // attach a provider's scope-related metadata onto its controller
    this.container.bindGlobalScope(); // wire up modules that import a global module
  }
```

### InstanceLoader
`InstanceLoader` creates, in order, the prototype objects and then the instances for a module's `Provider`, `Injectable`, and `Controller` entries. Building the prototype is simple, but building instances that sit inside a nontrivial dependency graph is not. That work belongs to an internal class called `Injector`, which figures out the dependency hierarchy, coordinates the order of creation through callbacks, and injects everything correctly to produce instances for all the dependency objects. It plays a central role in DI. There's too much to cover here, so I'll dedicate a future post to `Injector`.

```typescript
// packages/core/injector/instance-loader.ts
export class InstanceLoader {
  protected readonly injector = new Injector();

  public async createInstancesOfDependencies(
    modules: Map<string, Module> = this.container.getModules(),
  ) {
    this.createPrototypes(modules); // create prototypes for dependency objects
    await this.createInstances(modules); // create instances for dependency objects
  }
  ...
}
```

## NestApplication
And finally, the last step. Once module and dependency registration wrap up, `NestFactory` creates the `NestApplication` instance. It wraps that instance in a `Proxy` to handle error handling, method chaining, and fallbacks, and with that, `NestFactory`'s job is done.

```typescript
// packages/core/nest-factory.ts
export class NestFactory {
  public async create<T extends INestApplication = INestApplication>(
    module: any,
    serverOrOptions?: AbstractHttpAdapter | NestApplicationOptions,
    options?: NestApplicationOptions,
  ): Promise<T> {
    const [httpServer, appOptions] = this.isHttpServer(serverOrOptions)
      ? [serverOrOptions, options]
      : [this.createHttpAdapter(), serverOrOptions];

    const applicationConfig = new ApplicationConfig();
    const container = new NestContainer(applicationConfig);
    this.setAbortOnError(serverOrOptions, options);
    this.registerLoggerConfiguration(appOptions);

	await this.initialize(module, container, applicationConfig, httpServer);

	// =======================================================
    const instance = new NestApplication(
      container,
      httpServer,
      applicationConfig,
      appOptions,
    );
    const target = this.createNestInstance(instance);
    return this.createAdapterProxy<T>(target, httpServer);
    // =======================================================
  }
}
```

## Wrap-up
This post traced how `NestFactory` builds `NestApplication`, the object that holds all of a service's logic. Next time, I'll cover how `NestApplication` actually handles an incoming request. Thanks for reading.


## References
- [(GitHub) nestjs/nest](https://github.com/nestjs/nest)
- [[Nest.js] How the OnModuleInit interface and Instance Loader work](https://m.blog.naver.com/biud436/222571147763)
