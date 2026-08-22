+++
title = '[NestJS 파헤치기] 03. InstanceLoader and Injector'
date = '2022-11-28T11:47:49+09:00'
draft = false
translationKey = 'nestjs-internals-03-instanceloader-and-injector'
slug = 'nestjs-internals-03-instanceloader-and-injector'
description = 'InstanceWrapper와 Injector가 프로토타입을 생성하는 단계, 그리고 resolveConstructorParams()를 거쳐 의존성 객체의 인스턴스를 생성하고 주입하는 단계를 다룹니다.'
tags = ['NestJS', 'TypeScript', 'DI', 'Node.js']
categories = ['NestJS']
+++

## Intro
안녕하세요. 이전 포스팅에서는 NestJS에서 모듈과 의존성 객체의 메타데이터가 어떤 과정을 거쳐 등록되는지 살펴보았습니다. 이번 포스팅에서는 모듈에 등록된 의존성 객체 인스턴스의 라이프사이클(`생성`, `주입`, `제거`)을 관리하는 `InstanceLoader`와 `Injector`에 대해 알아보도록 하겠습니다.

## Dependency Injection in NestJS
`Dependency Injection`(이하 의존성 주입)은 인스턴스들의 의존 관계를 선언적으로 표현하고, 의존 관계의 파싱과 인스턴스 생성은 일반적으로 프레임워크에서 관리하는 IoC 컨테이너에 위임하는 프로그래밍 방법론입니다. 의존성 주입과 관련된 자세한 내용은 이번 포스팅의 범위를 벗어나므로 생략하도록 하겠습니다.

의존성 주입을 구현할 때에는 객체들의 의존 관계를 정확히 파싱하고 이를 순서대로 조율해주는 과정이 매우 중요합니다. NestJS에서는 크게 생성자 기반(`constructor-based`) 방식과 프로퍼티 기반(`property-based`) 방식이 있습니다. 아래 예시에서 `CatController`는 `CatService`에 의존성을 가지고 있습니다. 따라서 `CatController`의 인스턴스를 생성하려면 먼저 `CatService`의 인스턴스를 생성해야 하고, 이 인스턴스를 `CatController`를 생성할 때 주입해주어야 합니다. NestJS에서는 `InstanceLoader`와 `Injector`가 이 과정을 실질적으로 주도하고 조율하는 역할을 한다고 볼 수 있습니다.

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
`InstanceLoader`는 그 이름처럼, 모듈의 메타데이터를 파싱하여 의존성 객체(`provider`, `injectable`, `controller`)를 생성하는 클래스입니다. `InstanceLoader`의 역할은 크게 프로토타입 생성과 인스턴스 생성으로 나뉠 수 있습니다. `InstanceLoader`는 먼저 의존성 객체의 프로토타입 객체를 생성한 후, 이를 활용하여 인스턴스를 생성합니다. 내부 구현을 살펴보면, `InstanceLoader`의 주요 기능 대부분이 `Injector`에 의존하고 있는 것을 확인할 수 있습니다.


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
`Injector`에 대해 살펴보기에 앞서, 먼저 `InstanceWrapper`에 대해 짧게 알고 넘어가도록 하겠습니다. `InstanceWrapper`는 하나의 의존성 객체가 의존성 주입 과정에서 필요한 다양한 메타데이터를 가지고 있으며, `Context`, `Scope`에 따라 인스턴스의 라이프사이클을 관리하는 역할을 합니다. NestJS는 내부적으로 의존성 객체의 인스턴스들을 `InstanceWrapper`라는 클래스로 감싸서 관리합니다.

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

모듈에 의존성 객체가 등록되는 과정을 자세히 살펴보면, `InstanceWrapper`의 존재를 찾을 수 있습니다. 이전 포스팅에서, `DependenciesScanner`가 모듈의 메타데이터를 파싱하여 `Module` 객체에 해당 모듈이 가지고 있는 의존성 객체의 메타데이터를 등록한다는 내용을 다루었습니다. 관련 코드를 따라가다 보면, 최종적으로 `Module.addProvider()`가 호출되어 `InstanceWrapper` 객체가 생성되는 것을 확인할 수 있습니다.

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
다시 `Injector`로 돌아와보도록 하겠습니다. 이번 포스팅에서는 `InstanceLoader`가 호출하는 `Injector`의 주요 메서드인 `loadPrototype`과 `loadProvider`를 중점적으로 다뤄보도록 하겠습니다.

#### Injector.loadPrototype()
먼저 `Injector.loadPrototype()` 메서드를 살펴보면, `Object.create()` 메서드를 호출하는 방식으로 빈 객체를 생성합니다. 실제로는 인스턴스를 생성하는 것이지만, 이렇게 생성된 객체는 생성자 메서드를 호출하지 않으므로 다른 객체에 의존하는 객체도 생성할 수 있습니다. 생성된 인스턴스는 자신이 의존하는 객체를 아직 주입받지 못했기 때문에, 대부분의 프로퍼티가 `undefined`인 상태입니다. 코드 구현을 살펴보면, 구체적으로는 `DependenciesScanner`가 생성한 `InstanceWrapper`에 `instance` 프로퍼티를 업데이트하는 방식으로 의존성 객체의 인스턴스를 등록합니다.

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
다음으로, `InstanceLoader`는 `Injector.loadInstance()` 메서드를 호출합니다. 여러 모듈에 등록된 의존성 객체 인스턴스 생성 과정에서 발생할 수 있는 충돌을 방지하기 위해 `Injector`는 `InstancePerContext` 객체의 프로퍼티(`isPending`, `isResolved`, `donePromise`)를 조작하여 인스턴스 생성 과정을 조율합니다.

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

이후 본격적으로 의존성 주입을 통해 인스턴스를 생성하는 절차가 진행됩니다. `@Inject`, `@Optional` 등의 데코레이터를 통해 등록된 메타데이터는 이 과정에서 활용됩니다.

1. `resolveConstructorParams()`를 호출하여 생성자에 등록된 의존성 정보들을 파싱한 후, 해당되는 객체들을 불러옵니다.

2. `resolveConstructorParams()` 내부에서 주어진 `callback` 함수를 호출합니다. `callback`은 생성자에 등록된 의존성 객체들을 주입하여 인스턴스를 생성하고, 추가적으로 프로퍼티에 등록된 의존성 객체들을 주입하는 등 의존성 주입에서 핵심적인 역할을 합니다.

위 과정을 마친 후에는 `NestContainer`에 등록된 모든 의존성 객체(ex, `Provider`, `Controller`)의 인스턴스 생성 과정이 마무리됩니다.

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

## 마무리

이번 포스팅에서는 NestJS에서 핵심이라고 할 수 있는 인스턴스 생성과 의존성 주입을 담당하는 `InstanceLoader`와 `Injector`에 대해 대략적으로 살펴보았습니다. 다음 포스팅에서는 `NestApplication`에 대해 다뤄보도록 하겠습니다.
