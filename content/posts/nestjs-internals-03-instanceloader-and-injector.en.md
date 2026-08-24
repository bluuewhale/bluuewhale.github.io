+++
title = '[Diving into NestJS] 03. InstanceLoader and Injector'
date = '2022-11-28T11:47:49+09:00'
draft = false
translationKey = 'nestjs-internals-03-instanceloader-and-injector'
slug = 'nestjs-internals-03-instanceloader-and-injector-en'
aliases = ['/posts/nestjs-internals-03-instanceloader-and-injector-en/']
description = 'How InstanceWrapper and Injector create prototypes, walk resolveConstructorParams(), and produce and inject instances of every dependency object.'
tags = ['NestJS', 'TypeScript', 'DI', 'Node.js']
categories = ['NestJS']
+++

## Intro
Hi everyone. Last time, we covered how NestJS registers metadata for modules and their dependency objects. This time, let's look at `InstanceLoader` and `Injector`, the two classes that manage the lifecycle (creation, injection, teardown) of the instances registered in each module.

## Dependency Injection in NestJS
Dependency Injection (DI) is a programming approach where you declare the dependencies between instances up front, and hand off the work of parsing those relationships and creating instances to an IoC container, usually managed by the framework. A full explanation of DI is out of scope here.

Implementing DI well depends on parsing an object graph correctly and coordinating creation order. NestJS supports two styles: constructor-based and property-based. In the example below, `CatController` depends on `CatService`. So building a `CatController` instance requires first building a `CatService` instance and injecting it in. `InstanceLoader` and `Injector` direct and coordinate this process in NestJS.

```typescript
@Controller
class CatController {
  // property-based
  @Inject('HTTP_OPTIONS')
  private readonly httpClient: T;

  // constructor-based
  constructor(private readonly catService: CatService) {}
}
```


## InstanceLoader
`InstanceLoader` does exactly what its name suggests: it parses a module's metadata and creates its dependency objects (`provider`, `injectable`, `controller`). Its job splits into two phases: creating prototypes, then creating instances. `InstanceLoader` builds the prototype objects for dependencies first, then uses them to build instances. Look at the implementation and you'll notice `InstanceLoader` delegates most of its real work to `Injector`.


```typescript
// packages/core/injector/instance-loader.ts
export class InstanceLoader {
  ...
  
  // ========================================================
  public async createInstancesOfDependencies(
    modules: Map<string, Module> = this.container.getModules(),
  ) {
    
    this.createPrototypes(modules);
    await this.createInstances(modules);  
  }
  // ========================================================

  private createPrototypes(modules: Map<string, Module>) {
    modules.forEach(moduleRef => {
      this.createPrototypesOfProviders(moduleRef);
      this.createPrototypesOfInjectables(moduleRef);
      this.createPrototypesOfControllers(moduleRef);
    });
  }

  private async createInstances(modules: Map<string, Module>) {
    await Promise.all(
      [...modules.values()].map(async moduleRef => {
        await this.createInstancesOfProviders(moduleRef);
        await this.createInstancesOfInjectables(moduleRef);
        await this.createInstancesOfControllers(moduleRef);

        const { name } = moduleRef.metatype;
        this.isModuleWhitelisted(name) &&
          this.logger.log(MODULE_INIT_MESSAGE`${name}`);
      }),
    );
  }

  private createPrototypesOfProviders(moduleRef: Module) {
    const { providers } = moduleRef;
    providers.forEach(wrapper =>
      this.injector.loadPrototype<Injectable>(wrapper, providers),
    );
  }

  private async createInstancesOfProviders(moduleRef: Module) {
    const { providers } = moduleRef;
    const wrappers = [...providers.values()];
    await Promise.all(
      wrappers.map(item => this.injector.loadProvider(item, moduleRef)),
    );
  }

  private createPrototypesOfControllers(moduleRef: Module) {
    const { controllers } = moduleRef;
    controllers.forEach(wrapper =>
      this.injector.loadPrototype<Controller>(wrapper, controllers),
    );
  }

  private async createInstancesOfControllers(moduleRef: Module) {
    const { controllers } = moduleRef;
    const wrappers = [...controllers.values()];
    await Promise.all(
      wrappers.map(item => this.injector.loadController(item, moduleRef)),
    );
  }

  private createPrototypesOfInjectables(moduleRef: Module) {
    const { injectables } = moduleRef;
    injectables.forEach(wrapper =>
      this.injector.loadPrototype(wrapper, injectables),
    );
  }

  private async createInstancesOfInjectables(moduleRef: Module) {
    const { injectables } = moduleRef;
    const wrappers = [...injectables.values()];
    await Promise.all(
      wrappers.map(item => this.injector.loadInjectable(item, moduleRef)),
    );
  }

  private isModuleWhitelisted(name: string): boolean {
    return name !== InternalCoreModule.name;
  }
```

## InstanceWrapper
Before we get to `Injector`, let's touch briefly on `InstanceWrapper`. Each `InstanceWrapper` holds the metadata a dependency object needs during DI, and manages that instance's lifecycle according to its `Context` and `Scope`. Internally, NestJS wraps every dependency instance in an `InstanceWrapper`.

```typescript
// packages/core/injector/instance-wrapper.ts
export class InstanceWrapper<T = any> {
  public readonly name: any;
  public readonly token: InstanceToken;
  public readonly async?: boolean;
  public readonly host?: Module;
  public readonly isAlias: boolean = false;

  public scope?: Scope = Scope.DEFAULT;
  public metatype: Type<T> | Function;
  public inject?: FactoryProvider['inject'];
  public forwardRef?: boolean;
  public durable?: boolean;

  private readonly values = new WeakMap<ContextId, InstancePerContext<T>>();
  private transientMap?:
    | Map<string, WeakMap<ContextId, InstancePerContext<T>>>
    | undefined;
  private isTreeStatic: boolean | undefined;
  private isTreeDurable: boolean | undefined;
  private readonly [INSTANCE_METADATA_SYMBOL]: InstanceMetadataStore = {};
  private readonly [INSTANCE_ID_SYMBOL]: string;
  
  private static logger: LoggerService = new Logger(InstanceWrapper.name);

  public createPrototype(contextId: ContextId) {
    const host = this.getInstanceByContextId(contextId);
    if (!this.isNewable() || host.isResolved) {
      return;
    }
    return Object.create(this.metatype.prototype);
  }
}
```

Trace through how a dependency object gets registered into a module and you'll find `InstanceWrapper` right there. Last time, we covered how `DependenciesScanner` parses a module's metadata and registers the metadata for its dependencies onto the `Module` object. Follow that code far enough and it eventually calls `Module.addProvider()`, which is where the `InstanceWrapper` object gets created.

```typescript
// packages/core/scanner.ts
export class DependenciesScanner {
  
  public async scan(module: Type<any>) {
    await this.registerCoreModule();
    await this.scanForModules(module);    
    await this.scanModulesForDependencies(); // <<<<<<<<<<<<< (1)
    this.calculateModulesDistance();

    this.addScopedEnhancersMetadata();
    this.container.bindGlobalScope();
  }
  
  public async scanModulesForDependencies(
    modules: Map<string, Module> = this.container.getModules(),
  ) {
    for (const [token, { metatype }] of modules) {
      await this.reflectImports(metatype, token, metatype.name);
      this.reflectProviders(metatype, token); // <<<<<<<<<<<<<< (2)
      this.reflectControllers(metatype, token);
      this.reflectExports(metatype, token);
    }
  }
  
  public reflectProviders(module: Type<any>, token: string) {
    const providers = [
      ...this.reflectMetadata(MODULE_METADATA.PROVIDERS, module),
      ...this.container.getDynamicMetadataByToken(
        token,
        MODULE_METADATA.PROVIDERS as 'providers',
      ),
    ];
    providers.forEach(provider => {
      this.insertProvider(provider, token); // <<<<<<<<<<<<<< (3)
      this.reflectDynamicMetadata(provider, token);
    });
  }
  
 public insertProvider(provider: Provider, token: string) {
    const isCustomProvider = this.isCustomProvider(provider);
    if (!isCustomProvider) {
      return this.container.addProvider(provider as Type<any>, token); // <<<<<<<<<<<<< (4)
    }
    const applyProvidersMap = this.getApplyProvidersMap();
    const providersKeys = Object.keys(applyProvidersMap);
    const type = (
      provider as
        | ClassProvider
        | ValueProvider
        | FactoryProvider
        | ExistingProvider
    ).provide;

    if (!providersKeys.includes(type as string)) {
      return this.container.addProvider(provider as any, token); // <<<<<<<<<<<<< (4)
    }
  
    // =================================================================
    // below is for global injectables(ex, interceptor, guards..) 
    // registered using nestjs-provided constants (ex, APP_INTERCEPTOR)
    // =================================================================
    const providerToken = `${
      type as string
    } (UUID: ${randomStringGenerator()})`;

    let scope = (provider as ClassProvider | FactoryProvider).scope;
    if (isNil(scope) && (provider as ClassProvider).useClass) {
      scope = getClassScope((provider as ClassProvider).useClass);
    }
    this.applicationProvidersApplyMap.push({
      type,
      moduleKey: token,
      providerKey: providerToken,
      scope,
    });

    const newProvider = {
      ...provider,
      provide: providerToken,
      scope,
    } as Provider;

    const factoryOrClassProvider = newProvider as
      | FactoryProvider
      | ClassProvider;
    if (this.isRequestOrTransient(factoryOrClassProvider.scope)) {
      return this.container.addInjectable(newProvider, token); // <<<<<<<<<<<<< (4)
    }
    this.container.addProvider(newProvider, token); // <<<<<<<<<<<<<< (4)
  }
}
```

```typescript
// packages/core/injector/container.ts
export class NestContainer {
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
    return moduleRef.addProvider(provider); // <<<<<<<<<<<<< (5)
  }
}
```

```typescript
// packages/core/injector/module.ts
export class Module {
  ...
  public addProvider(provider: Provider) {
    if (this.isCustomProvider(provider)) {
      return this.addCustomProvider(provider, this._providers);
    }
    this._providers.set(
      provider,
      new InstanceWrapper({ // <<<<<<<<<<<<<< (6)
        token: provider,
        name: (provider as Type<Injectable>).name,
        metatype: provider as Type<Injectable>, // prototype object
        instance: null, // registered without any instance
        isResolved: false,
        scope: getClassScope(provider),
        durable: isDurable(provider),
        host: this,
      }),
    );
    return provider as Type<Injectable>;
  }
}
```

## Injector
Now back to `Injector`. In this post, I want to focus on the two methods `InstanceLoader` calls on it: `loadPrototype` and `loadProvider`.

#### Injector.loadPrototype()
`Injector.loadPrototype()` creates an empty object by calling `Object.create()`. It's technically already an instance, but since it never went through a constructor call, this is a safe way to build objects that depend on other objects that don't exist yet. Because it hasn't received any injected dependencies, most of its properties are still `undefined` at this point. Concretely, this method registers the instance by updating the `instance` property on the `InstanceWrapper` that `DependenciesScanner` already created.

```typescript
// packages/core/injector/injector.ts
export class Injector {
  public loadPrototype<T>(
    { token }: InstanceWrapper<T>,
    collection: Map<InstanceToken, InstanceWrapper<T>>,
    contextId = STATIC_CONTEXT,
  ) {
    if (!collection) {
      return;
    }
    const target = collection.get(token);
    const instance = target.createPrototype(contextId);
    if (instance) {
      const wrapper = new InstanceWrapper({
        ...target,
        instance, // initialized with instance for static context
      });
      collection.set(token, wrapper);
    }
  }
}
```

```typescript
// packages/core/injector/instance-wrapper.ts
export class InstanceWrapper<T = any> {
  ...
  set instance(value: T) {
    this.values.set(STATIC_CONTEXT, { instance: value });
  }

  get instance(): T {
    const instancePerContext = this.getInstanceByContextId(STATIC_CONTEXT);
    return instancePerContext.instance;
  }
}
```

#### Injector.loadInstance()
Next, `InstanceLoader` calls `Injector.loadInstance()`. Building instances across multiple modules can create conflicts, so `Injector` coordinates that process by toggling flags (`isPending`, `isResolved`, `donePromise`) on the `InstancePerContext` object.

```typescript
// packages/core/injector/injector.ts
export class Injector {
  public async loadInstance<T>(
    wrapper: InstanceWrapper<T>,
    collection: Map<InstanceToken, InstanceWrapper>,
    moduleRef: Module,
    contextId = STATIC_CONTEXT,
    inquirer?: InstanceWrapper,
  ) {
    const inquirerId = this.getInquirerId(inquirer);
    const instanceHost = wrapper.getInstanceByContextId(
      this.getContextId(contextId, wrapper),
      inquirerId,
    );
    if (instanceHost.isPending) {
      return instanceHost.donePromise.then((err?: unknown) => {
        if (err) {
          throw err;
        }
      });
    }
    const done = this.applyDoneHook(instanceHost);
    const token = wrapper.token || wrapper.name;

    const { inject } = wrapper;
    const targetWrapper = collection.get(token);
    if (isUndefined(targetWrapper)) {
      throw new RuntimeException();
    }
    if (instanceHost.isResolved) {
      return done();
    }      
    ...
  }
  
  public applyDoneHook<T>(
    wrapper: InstancePerContext<T>,
  ): (err?: unknown) => void {
    let done: (err?: unknown) => void;
    wrapper.donePromise = new Promise<unknown>((resolve, reject) => {
      done = resolve;
    });
    wrapper.isPending = true;
    return done;
  }
}
```

From here, actual instance creation through dependency injection begins. Metadata registered via decorators like `@Inject` and `@Optional` comes into play at this stage.

1. It calls `resolveConstructorParams()`, which parses the dependency info declared in the constructor and fetches the matching objects.

2. Inside `resolveConstructorParams()`, it invokes the `callback` function you passed in. `callback` does the central DI work: it injects the constructor's dependency objects to build the instance, then injects any property-based dependencies on top of that.

Once this finishes, instance creation is complete for every dependency object (`Provider`, `Controller`, and so on) registered in `NestContainer`.

```typescript
// packages/core/injector/injector.ts
export class Injector {
  public async loadInstance<T>(
    wrapper: InstanceWrapper<T>,
    collection: Map<InstanceToken, InstanceWrapper>,
    moduleRef: Module,
    contextId = STATIC_CONTEXT,
    inquirer?: InstanceWrapper,
  ) {
    ...
    // instantiation
    try {
      const callback = async (instances: unknown[]) => {
        const properties = await this.resolveProperties(
          wrapper,
          moduleRef,
          inject as InjectionToken[],
          contextId,
          wrapper,
          inquirer,
        );
        const instance = await this.instantiateClass(
          instances,
          wrapper,
          targetWrapper,
          contextId,
          inquirer,
        );
        this.applyProperties(instance, properties);
        done();
      };
      await this.resolveConstructorParams<T>(
        wrapper,
        moduleRef,
        inject as InjectionToken[],
        callback,
        contextId,
        wrapper,
        inquirer,
      );
    } catch (err) {
      done(err);
      throw err;
    }
  }
}
```

#### Injector.resolveConstructorParams()

```typescript
export class Injector {
  public async resolveConstructorParams<T>(
    wrapper: InstanceWrapper<T>,
    moduleRef: Module,
    inject: InjectorDependency[],
    callback: (args: unknown[]) => void | Promise<void>,
    contextId = STATIC_CONTEXT,
    inquirer?: InstanceWrapper,
    parentInquirer?: InstanceWrapper,
  ) {
    // 1. skip if it is a redundant execution
    let inquirerId = this.getInquirerId(inquirer);
    const metadata = wrapper.getCtorMetadata();

    if (metadata && contextId !== STATIC_CONTEXT) {
      const deps = await this.loadCtorMetadata(
        metadata,
        contextId,
        inquirer,
        parentInquirer,
      );
      return callback(deps);
    }

    // 2. parse param types in constructor
    const isFactoryProvider = !isNil(inject);
    const [dependencies, optionalDependenciesIds] = isFactoryProvider
      ? this.getFactoryProviderDependencies(wrapper)
      : this.getClassDependencies(wrapper);

    // 3. resolve individual parameters
    let isResolved = true;
    const resolveParam = async (param: unknown, index: number) => {
      try {
        if (this.isInquirer(param, parentInquirer)) {
          return parentInquirer && parentInquirer.instance;
        }
        if (inquirer?.isTransient && parentInquirer) {
          inquirer = parentInquirer;
          inquirerId = this.getInquirerId(parentInquirer);
        }
        const paramWrapper = await this.resolveSingleParam<T>(
          wrapper,
          param,
          { index, dependencies },
          moduleRef,
          contextId,
          inquirer,
          index,
        );
        const instanceHost = paramWrapper.getInstanceByContextId(
          this.getContextId(contextId, paramWrapper),
          inquirerId,
        );
        if (!instanceHost.isResolved && !paramWrapper.forwardRef) {
          isResolved = false;
        }
        return instanceHost?.instance;
      } catch (err) {
        const isOptional = optionalDependenciesIds.includes(index);
        if (!isOptional) {
          throw err;
        }
        return undefined;
      }
    };
    const instances = await Promise.all(dependencies.map(resolveParam));
    isResolved && (await callback(instances));
  }
}
```

## Wrap-up

This post gave a broad tour of `InstanceLoader` and `Injector`, the two classes at the heart of instance creation and dependency injection in NestJS. Next time, I'll cover `NestApplication` itself.
