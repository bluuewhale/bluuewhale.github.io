+++
title = '[NestJS 파헤치기] 01. NestFactory'
date = '2022-11-15T15:17:34+09:00'
draft = false
translationKey = 'nestjs-internals-01-nestfactory'
slug = 'nestjs-internals-01-nestfactory'
description = 'NestFactory.create() 내부를 따라가며 HttpAdapter, ApplicationConfig, NestContainer가 생성되고 NestApplication이 완성되는 과정을 살펴봅니다.'
tags = ['NestJS', 'TypeScript', 'DI', 'Node.js']
categories = ['NestJS']
+++

안녕하세요. 최근 여러 가지 일로 바쁜 나날을 보내다 보니 오랜만에 포스팅을 하게 되었습니다.
이번에는 NestJS 파헤치기라는 주제로 글을 작성해보려 합니다.
최근 업무상 NestJS를 새롭게 사용하게 되었는데, 그 과정에서 NestJS 코어와 관련한 지식이 부족하여 기능을 구현하는 데 어려움을 느끼는 일이 잦아졌습니다. 그래서 개인적인 공부를 위해 NestJS 프레임워크를 개괄적으로 살펴보는 시간을 가져보고자 마음먹었습니다.

이번에는 그 첫 시간으로 모든 NestJS 어플리케이션의 진입점에 해당하는 `NestFactory` 클래스에 대해 살펴보도록 하겠습니다.

## NestFactory
NestJS 공식문서를 살펴보면, 처음 마주하는 튜토리얼에서 다음과 같은 코드를 찾아볼 수 있습니다.

```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(3000);
}

bootstrap();
```

이를 통해 `NestFactory`가 `AppModule`을 읽어서, 서버에 전달된 요청을 처리하는 모든 로직을 담고 있는 `NestApplication` 인스턴스를 생성하는 역할을 한다는 것을 확인할 수 있습니다. 이번 포스팅의 목표는 `NestFactory.create()` 안에서 과연 무슨 일들이 일어나고 있는지 이해하는 것입니다.

## NestFactory.create()

`NestFactory.create()`를 들여다보면 다음과 같이 구현되어 있습니다. 지금부터 한 줄 한 줄 차근차근 자세히 살펴보도록 하겠습니다.
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
처음 세 줄을 살펴보면, `NestFactory`는 내부적으로 주입받은 `HttpAdapter`를 그대로 사용하거나, 그렇지 않으면 기본값으로 새로운 `HttpAdapter` 인스턴스를 생성합니다.

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

`HttpAdapter`는 NestJS Core에서 정의하고 있는 `AbstractHttpAdapter` 인터페이스를 구현한 클래스에 해당합니다. `AbstractHttpAdapter`는 http 요청을 처리하는 데 필요한 인터페이스를 정의하고 있는 `HttpServer` 인터페이스의 일부를 구현한 추상 클래스입니다.

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

한 가지 재미있는 점은, `AbstractHttpAdapter`가 일부 메서드(예: `use`, `get`, `post`)의 구현을 `any` 타입으로 주입받은 `instance`에 의존하고 있다는 점입니다. 이미 메서드 이름이 낯익다는 것을 눈치채신 분들도 있을 것 같습니다.

그렇습니다. 해당 메서드는 `express`에서 미들웨어나 핸들러를 등록할 때 사용하는 메서드명과 일치합니다. 실제로 `NestFactory`는 `express`에 의존하여 `AbstractHttpAdapter` 인터페이스를 구현한 `ExpressAdapter`를 기본값으로 사용하고 있습니다. 이러한 관점에서 보면 `NestJS`는 일종의 `express-wrapper` 프레임워크라고도 볼 수 있습니다. `express`에 익숙하신 분들이라면 정말 반가운 일이 아닐 수 없습니다.

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
이제 다음 줄로 넘어가보도록 하겠습니다. `NestFactory`는 이후 `ApplicationConfig` 인스턴스를 생성합니다.

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

`ApplicationConfig`는 `NestJS`에서 사용되는 글로벌 미들웨어들을 저장하고 있는 비교적 단순한 데이터 객체에 해당합니다. 처음에는 대부분 빈 값으로 초기화되어 있지만, 우리가 `NestApplication`을 생성하는 과정에서 모든 글로벌 미들웨어들은 내부적으로 `ApplicationConfig`에 등록됩니다.

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

다음으로 `NestFactory`는 `NestJS`의 의존성 주입(`DI`)에서 핵심적인 역할을 수행하는 `NestContainer` 객체를 생성합니다. `NestContainer`는 `NestApplication`이 동작하기 위해 필수적인 모듈(`Module`)을 저장하고 있는 객체입니다.

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

`NestContainer`는 내부적으로 `Module`, `ModuleTokenFactory`, `ModuleCompiler`, `ModulesContainer`와 같은 클래스들을 가지고 있습니다. `NestContainer`에는 이후 `DependenciesScanner`를 통해 모듈이 등록됩니다. 이와 관련된 보다 자세한 내용은 뒤에 이어서 다루도록 하겠습니다.

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

`Module` 클래스는 하나의 모듈에 포함되어 있는 모든 의존성(예: `Provider`, `Controller`)을 정의하고 있는 핵심 객체입니다. 우리가 일반적으로 `@Module` 데코레이터 혹은 `forRoot()` 메서드를 통해 `DynamicModule` 형태로 등록한 모듈들은 `NestJS` 프레임워크 안에서는 `Module` 클래스로 관리됩니다.

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

`ModuleTokenFactory`는 간단히 말해 모듈을 등록·관리할 때 키로 사용되는 토큰(`hash`)을 생성하는 클래스입니다.

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

`NestContainer`에서는 `DynamicModule`의 의존성 관련 정보를 따로 분리하여 관리합니다. `ModuleCompiler`는 이를 위해 `Module`의 정보를 적절히 파싱하는 기능을 수행하는 일종의 유틸리티 클래스에 해당한다고 볼 수 있습니다.

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

`ModulesContainer`는 모든 `Module` 정보가 저장되는 비교적 단순한 `Map<string, Module>` 타입의 클래스입니다. 키 값으로는 `ModuleTokenFactory`에서 생성된 토큰을 사용하게 됩니다.

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
다음으로 `NestFactory`는 간단한 설정을 마친 후 `initialize()` 메서드를 호출합니다. `initialize()` 메서드의 핵심 역할은 크게 `DependenciesScanner`에 의한 모듈 등록 과정과 `InstanceLoader`에 의한 의존성 객체 생성 과정으로 나눌 수 있습니다.

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

`DependenciesScanner`의 가장 중요한 역할은 모듈 트리를 순회하며 `NestContainer`에 모듈들을 등록하는 것입니다. 그뿐만 아니라 `DependenciesScanner`는 모듈 간의 연결 관계(`edge`)와 관련된 정보를 등록하고, 모듈에서 관리하는 의존성(예: `providers`) 등의 메타 정보를 관리하며, 이후 `NestApplication`에서 모듈을 순회할 때(예: `Module Lifecycle Hooks`) 필요한 위상 정렬을 위해 모듈의 깊이를 계산하는 등 다양한 기능을 추가로 수행합니다.

```typescript
// packages/core/scanner.ts
export class DependenciesScanner {
  ...
  public async scan(module: Type<any>) {
    await this.registerCoreModule(); // NestContainer에 InternalCoreModule 등록
    await this.scanForModules(module); // Module Tree를 재귀적으로 탐색하여 모든 모듈을 NestContainer에 등록 
    await this.scanModulesForDependencies(); // 메타데이터 파싱을 통해 등록된 모듈의 의존성(imports, providers, controllers, exports) 관련 정보를 추가 
    this.calculateModulesDistance(); // Module Tree를 순회하며 모듈을 위상 정렬(Topological Sort)하기 위해 필요한 depth 정보를 등록

    this.addScopedEnhancersMetadata(); // Provider의 Scope 관련 메타데이터를 Controller에 추가 등록
    this.container.bindGlobalScope(); // Global Module을 import 하고 있는 모듈들에 연결 정보 추가
  }
```

### InstanceLoader
`InstanceLoader`는 `NestContainer`에 등록된 모듈의 `Provider`, `Injectable`, `Controller`의 프로토타입 객체와 인스턴스를 순서대로 생성하는 역할을 합니다. 프로토타입을 생성하는 내부 과정은 단순하지만, 복잡한 의존성 주입 관계를 갖고 있는 인스턴스를 생성하는 과정은 비교적 쉽지 않습니다. 해당 기능은 `Injector`라 불리는 내부 클래스가 실행합니다. `Injector`는 의존성 객체의 위계를 파악하고 콜백을 활용하여 순서를 조율하며, 이를 적절히 주입하여 모든 의존성 객체의 인스턴스를 생성하는 등 `DI`에서 핵심적인 역할을 수행합니다. 이번 포스팅에서 다루기에는 내용이 많으므로 `Injector`는 추후 다른 포스팅에서 자세히 알아보도록 하겠습니다.

```typescript
// packages/core/injector/instance-loader.ts
export class InstanceLoader {
  protected readonly injector = new Injector();

  public async createInstancesOfDependencies(
    modules: Map<string, Module> = this.container.getModules(),
  ) {
    this.createPrototypes(modules); // 의존성 객체의 prototype 생성
    await this.createInstances(modules); // 의존성 객체의 instance 생성
  }
  ...
}
```

## NestApplication
드디어 마지막 과정입니다. `NestFactory`는 모듈과 의존성 객체 등록 과정을 모두 마친 후 `NestApplication` 인스턴스를 생성합니다. 이후 에러 핸들링, 메서드 체이닝, fallback 등 일부 기능을 위해 `Proxy` 객체로 감싸는 과정을 끝으로 `NestFactory`의 역할이 마무리됩니다.

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

## 마무리
이번 포스팅에서는 `NestFactory`를 통해 모든 서비스 로직을 담고 있는 `NestApplication`이 생성되는 과정을 살펴보았습니다. 다음 포스팅에서는 `NestApplication`이 어떻게 서버에 전달된 요청을 처리하는지와 관련된 내용을 다루어 보도록 하겠습니다. 감사합니다.


## 참고자료
- [(Github) nestjs/nest](https://github.com/nestjs/nest)
- [[Nest.js] Nest.js의 OnModuleInit 인터페이스와 Instance Loader의 동작 방식](https://m.blog.naver.com/biud436/222571147763)
