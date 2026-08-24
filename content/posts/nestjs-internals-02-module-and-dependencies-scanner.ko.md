+++
title = '[NestJS 파헤치기] 02. @Module and DynamicModule'
date = '2022-11-18T15:38:40+09:00'
draft = false
translationKey = 'nestjs-internals-02-module-and-dependencies-scanner'
slug = 'nestjs-internals-02-module-and-dependencies-scanner'
description = '@Module 데코레이터가 Reflect로 메타데이터를 등록하는 방식과, DependenciesScanner·NestContainer가 StaticModule·DynamicModule을 처리하는 과정을 살펴봅니다.'
tags = ['NestJS', 'TypeScript', 'DI', 'Node.js']
categories = ['NestJS']
+++

안녕하세요.
이전 포스팅에서는 `NestFactory`가 `NestApplication`을 생성하는 과정을 다루었습니다. 이번 포스팅에서는 NestJS를 구성하는 핵심 요소 중 하나인 `Module`이 어떻게 애플리케이션에 등록되는지 알아보겠습니다.


## @Module
NestJS에서는 모듈을 선언할 때 `@Module` 데코레이터를 사용합니다. NestJS 공식 문서에서는 `@Module`이 Nest가 애플리케이션 구조를 관리하는 데 필요한 메타데이터를 다루는 용도로 사용된다고 설명합니다. (다만, 여기서 말하는 모듈은 NestJS가 내부적으로 사용하는 Module과 구별되는 개념입니다.)

> A module is a class annotated with a @Module() decorator. The @Module() decorator provides metadata that Nest makes use of to organize the application structure.

`NestJS` 소스코드에서 `@Module` 데코레이터의 내부 구현을 살펴보면, `target` 클래스에 `@Module` 데코레이터의 파라미터로 전달한 데이터(예: `imports`)를 메타데이터로 추가한다는 것을 알 수 있습니다.

```typescript
// packages/common/decorators/modules/module.decorator.ts
export function Module(metadata: ModuleMetadata): ClassDecorator {
  const propsKeys = Object.keys(metadata);
  validateModuleKeys(propsKeys);

  return (target: Function) => {
    for (const property in metadata) {
      if (metadata.hasOwnProperty(property)) {
        Reflect.defineMetadata(property, (metadata as any)[property], target);
      }
    }
  };
}
```


## Reflect

Reflect는 런타임에 모든 JavaScript 객체와 그 객체의 프로퍼티에 다양한 메타데이터를 추가하여 메타프로그래밍(metaprogramming)을 가능하게 해 주는, JavaScript에 내장된 전역 객체입니다. Reflect에 관한 더 자세한 내용은 [proposal](https://rbuckton.github.io/reflect-metadata/#introduction)과 [API 문서](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Reflect)에서 확인할 수 있습니다.


## DependenciesScanner
`@Module` 데코레이터를 통해 등록된 메타데이터는 `DependenciesScanner`가 각 모듈의 참조 관계(`imports`)와 의존성(예: `providers`, `controllers`) 정보를 등록할 때 활용됩니다. `DependenciesScanner`에서 핵심적인 역할을 수행하는 `scanForModules()`와 `scanModulesForDependencies()` 메서드에서 이를 확인할 수 있습니다.

#### `DependenciesScanner.scanModules()`

```typescript
// packages/core/scanner.ts
export class DependenciesScanner {
  ...
  public async scanForModules(
    moduleDefinition:
      | ForwardReference
      | Type<unknown>
      | DynamicModule
      | Promise<DynamicModule>,
    scope: Type<unknown>[] = [],
    ctxRegistry: (ForwardReference | DynamicModule | Type<unknown>)[] = [],
  ): Promise<Module[]> {
    const moduleInstance = await this.insertModule(moduleDefinition, scope);
    moduleDefinition =
      moduleDefinition instanceof Promise
        ? await moduleDefinition
        : moduleDefinition;
    ctxRegistry.push(moduleDefinition);

    if (this.isForwardReference(moduleDefinition)) {
      moduleDefinition = (moduleDefinition as ForwardReference).forwardRef();
    }

    // ===========================================================
    const modules = !this.isDynamicModule(
      moduleDefinition as Type<any> | DynamicModule,
    )

      ? this.reflectMetadata(
          MODULE_METADATA.IMPORTS, // <<<<<<<<<<<<<<<<<<
          moduleDefinition as Type<any>,
        )
      : [
          ...this.reflectMetadata(
            MODULE_METADATA.IMPORTS, // <<<<<<<<<<<<<<<<<<
            (moduleDefinition as DynamicModule).module,
          ),
          ...((moduleDefinition as DynamicModule).imports || []),
        ];
	// ===========================================================
	...
  }
    
  public reflectMetadata(metadataKey: string, metatype: Type<any>) {
    return Reflect.getMetadata(metadataKey, metatype) || [];
  }
}  
```

#### `DependenciesScanner.scanModulesForDependencies()`
```typescript
// packages/core/scanner.ts
export class DependenciesScanner {
  ...
  public async scanModulesForDependencies(
    modules: Map<string, Module> = this.container.getModules(),
  ) {
    for (const [token, { metatype }] of modules) {
      await this.reflectImports(metatype, token, metatype.name);
      this.reflectProviders(metatype, token); // <<<<<<<<<<<<<<<<<<
      this.reflectControllers(metatype, token);
      this.reflectExports(metatype, token);
    }
  }

  public reflectProviders(module: Type<any>, token: string) {
    const providers = [
      // =========================================================
      ...this.reflectMetadata(MODULE_METADATA.PROVIDERS, module),  // <<<<<<<<<<<<<<<<<<
      // =========================================================
      ...this.container.getDynamicMetadataByToken(
        token,
        MODULE_METADATA.PROVIDERS as 'providers', 
      ),
    ];
    providers.forEach(provider => {
      this.insertProvider(provider, token);
      this.reflectDynamicMetadata(provider, token);
    });
  }

  public reflectMetadata(metadataKey: string, metatype: Type<any>) {
    return Reflect.getMetadata(metadataKey, metatype) || [];  // <<<<<<<<<<<<<<<<<<
  }
}  
```

## DynamicModule
NestJS는 `@Module` 외에도 모듈을 동적으로 구성할 수 있게 해 주는 `DynamicModule` 기능을 제공합니다. `DynamicModule`에 관한 자세한 설명은 이 포스팅의 범위를 벗어나므로 생략하겠습니다. 관련 내용은 [링크](https://docs.nestjs.com/fundamentals/dynamic-modules)에서 확인할 수 있습니다.

```typescript
// example of a dynamic module
import { Module, DynamicModule } from '@nestjs/common';
import { createDatabaseProviders } from './database.providers';
import { Connection } from './connection.provider';

@Module({
  providers: [Connection],
})
export class DatabaseModule {
  static forRoot(entities = [], options?): DynamicModule {
    const providers = createDatabaseProviders(options, entities);
    return {
      module: DatabaseModule,
      providers: providers,
      exports: providers,
    };
  }
}
```

`DynamicModule`의 구현을 살펴보면, 등록된 모듈 객체에 의존성 관련 메타데이터를 추가로 선언하는 형태라는 것을 확인할 수 있습니다. `@Module` 데코레이터를 통해 등록된 모듈(이후 편의를 위해 `StaticModule`이라고 부르겠습니다)은 `imports`, `controllers`와 같이 의존성과 관련된 정보를 `Reflect`를 활용하여 객체의 메타데이터에 저장합니다. 반면 `DynamicModule`은 `StaticModule`이 메타데이터로 가지고 있는 의존성 정보 외에, 추가 정보를 모듈 객체 인스턴스의 프로퍼티 값으로 저장합니다. 따라서 `DynamicModule`을 등록하는 과정에서는 추가적인 파싱 작업이 이루어집니다.

```typescript
// packages/common/interfaces/modules/dynamic-module.interface.ts
export interface DynamicModule extends ModuleMetadata {
  module: Type<any>;
  global?: boolean;
}
```

```typescript
// packages/common/interfaces/modules/module-metadata.interface.ts
export interface ModuleMetadata {
  imports?: Array<Type<any> | DynamicModule | Promise<DynamicModule> | ForwardReference>;
  controllers?: Type<any>[];
  providers?: Provider[];
  exports?: Array<
    | DynamicModule
    | Promise<DynamicModule>
    | string
    | symbol
    | Provider
    | ForwardReference
    | Abstract<any>
    | Function
  >;
}
```

## NestContainer
`DynamicModule`이 어떻게 등록되는지 더 자세히 살펴보기 위해, 이전 포스팅에서 다루었던 `NestContainer`를 잠시 다시 짚어보겠습니다. `NestContainer`는 모듈 데이터가 실질적으로 저장되는 객체입니다. NestJS에서 모듈 등록은 내부적으로 `NestContainer.addModule()` 메서드를 호출하여 이루어집니다. `ModuleCompiler`는 `NestContainer` 안에서 동적으로 등록된 모듈의 메타데이터를 파싱하는 역할을 수행합니다.


```typescript
export class NestContainer {
  public async addModule(
    metatype: Type<any> | DynamicModule | Promise<DynamicModule>,
    scope: Type<any>[],
  ): Promise<Module | undefined> {
    if (!metatype) {
      throw new UndefinedForwardRefException(scope);
    }
      
    // ==============================================================================
    const { type, dynamicMetadata, token } = await this.moduleCompiler.compile(
      metatype,
    );
    // ==============================================================================
      
    if (this.modules.has(token)) {
      return this.modules.get(token);
    }
    const moduleRef = new Module(type, this);
    moduleRef.token = token;
    this.modules.set(token, moduleRef);

    await this.addDynamicMetadata(
      token,
      dynamicMetadata,
      [].concat(scope, type),
    );

    if (this.isGlobalModule(type, dynamicMetadata)) {
      this.addGlobalModule(moduleRef);
    }
    return moduleRef;
  }
}
```


```typescript
// packages/core/injector/compiler.ts
export class ModuleCompiler {
  constructor(private readonly moduleTokenFactory = new ModuleTokenFactory()) {}

  public async compile(
    metatype: Type<any> | DynamicModule | Promise<DynamicModule>,
  ): Promise<ModuleFactory> {
    const { type, dynamicMetadata } = this.extractMetadata(await metatype);
    const token = this.moduleTokenFactory.create(type, dynamicMetadata);
    return { type, dynamicMetadata, token };
  }

  public extractMetadata(metatype: Type<any> | DynamicModule): {
    type: Type<any>;
    dynamicMetadata?: Partial<DynamicModule> | undefined;
  } {
    if (!this.isDynamicModule(metatype)) {
      return { type: metatype };
    }
    const { module: type, ...dynamicMetadata } = metatype;
    return { type, dynamicMetadata };
  }

  public isDynamicModule(
    module: Type<any> | DynamicModule,
  ): module is DynamicModule {
    return !!(module as DynamicModule).module;
  }
}
```

이후 아래와 같이 `dynamicModuleMetadata` 프로퍼티에 파싱된 `DynamicModule`의 메타데이터를 등록합니다. 마지막으로, `DynamicModule`에서 `import`하고 있는 다른 모듈 또한 재귀적으로 등록되는 것을 확인할 수 있습니다.

```typescript
export class NestContainer {
  public async addModule(
    metatype: Type<any> | DynamicModule | Promise<DynamicModule>,
    scope: Type<any>[],
  ): Promise<Module | undefined> {
    if (!metatype) {
      throw new UndefinedForwardRefException(scope);
    }
      
    const { type, dynamicMetadata, token } = await this.moduleCompiler.compile(
      metatype,
    );
      
    if (this.modules.has(token)) {
      return this.modules.get(token);
    }
    const moduleRef = new Module(type, this);
    moduleRef.token = token;
    this.modules.set(token, moduleRef);
      
    // ==============================================================================
    await this.addDynamicMetadata(
      token,
      dynamicMetadata,
      [].concat(scope, type),
    );
    // ==============================================================================
   
    if (this.isGlobalModule(type, dynamicMetadata)) {
      this.addGlobalModule(moduleRef);
    }
    return moduleRef;
  }

  public async addDynamicMetadata(
    token: string,
    dynamicModuleMetadata: Partial<DynamicModule>,
    scope: Type<any>[],
  ) {
    if (!dynamicModuleMetadata) {
      return;
    }
    this.dynamicModulesMetadata.set(token, dynamicModuleMetadata);

    const { imports } = dynamicModuleMetadata;
    await this.addDynamicModules(imports, scope);
  }

  public async addDynamicModules(modules: any[], scope: Type<any>[]) {
    if (!modules) {
      return;
    }
    await Promise.all(modules.map(module => this.addModule(module, scope)));
  }
}
```

## 마무리
이번 포스팅에서는 `StaticModule`과 `DynamicModule`의 메타데이터가 내부적으로 등록되는 과정을 살펴보았습니다. 하지만 메타데이터는 `모듈-모듈` 관계 혹은 `모듈-의존성 객체` 사이의 관계를 설명할 뿐입니다. 실질적으로 의존성 주입이 이루어지려면 의존성 객체의 인스턴스를 생성하고 라이프사이클을 관리하는 기능이 필요합니다. 다음 포스팅에서는 NestJS에서 이러한 역할을 담당하는 `InstanceLoader`와 `Injector`에 대해 더 자세히 알아보겠습니다.
