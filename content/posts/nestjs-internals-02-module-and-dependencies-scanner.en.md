+++
title = '[Diving into NestJS] 02. @Module and DynamicModule'
date = '2022-11-18T15:38:40+09:00'
draft = false
translationKey = 'nestjs-internals-02-module-and-dependencies-scanner'
slug = 'nestjs-internals-02-module-and-dependencies-scanner-en'
aliases = ['/posts/nestjs-internals-02-module-and-dependencies-scanner-en/']
description = 'How the @Module decorator attaches metadata via Reflect, and how DependenciesScanner and NestContainer register both StaticModule and DynamicModule.'
tags = ['NestJS', 'TypeScript', 'DI', 'Node.js']
categories = ['NestJS']
+++

Hi everyone.
Last time, we covered how `NestFactory` builds `NestApplication`. This time, let's look at how `Module`, one of NestJS's core building blocks, gets registered into your application.


## @Module
NestJS declares a module with the `@Module` decorator. The official docs describe `@Module` as the way Nest gathers the metadata it needs to organize the application's structure. (Note: "module" here means something different from the internal `Module` class NestJS uses under the hood.)

> A module is a class annotated with a @Module() decorator. The @Module() decorator provides metadata that Nest makes use of to organize the application structure.

Look at the internal implementation of the `@Module` decorator in the NestJS source, and you'll see it attaches the data you passed as parameters (`imports` and the rest) onto the target class as metadata.

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

`Reflect` is a global object built into JavaScript that enables metaprogramming: attaching arbitrary metadata to any object and its properties at runtime. You can read more in the [proposal](https://rbuckton.github.io/reflect-metadata/#introduction) and the [API docs](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Reflect).


## DependenciesScanner
`DependenciesScanner` uses the metadata registered through `@Module` to record each module's relationships (`imports`) and dependencies (`providers`, `controllers`, and so on). You'll find this logic in its two core methods, `scanForModules()` and `scanModulesForDependencies()`.

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
Beyond `@Module`, NestJS also offers `DynamicModule`, a way to configure a module dynamically at registration time. A full explanation of `DynamicModule` is out of scope for this post, but you can read more [here](https://docs.nestjs.com/fundamentals/dynamic-modules).

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

Look at how `DynamicModule` is implemented, and you'll see it's a way to declare additional dependency metadata on a registered module object. A module registered through `@Module` (let's call it a `StaticModule` from here on) stores its dependency info (`imports`, `controllers`, and so on) as metadata via `Reflect`. A `DynamicModule`, on the other hand, stores extra information as instance properties on top of whatever a `StaticModule` already carries in its metadata. That means registering a `DynamicModule` requires an extra parsing step.

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
To see how a `DynamicModule` gets registered, let's revisit `NestContainer` from the last post. `NestContainer` is where module data lives. Registering a module internally means calling `NestContainer.addModule()`, and `ModuleCompiler` handles parsing the metadata for any dynamically registered module inside it.


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

From here, the parsed `DynamicModule` metadata gets stored in the `dynamicModuleMetadata` property, as shown below. Notice, too, that any modules the `DynamicModule` imports get registered recursively.

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

## Wrap-up
This post covered how `StaticModule` and `DynamicModule` metadata gets registered internally. But metadata only describes relationships: module-to-module, or module-to-dependency. Performing dependency injection needs something more: creating instances of those dependency objects and managing their lifecycle. Next time, I'll dig into `InstanceLoader` and `Injector`, the two classes responsible for that in NestJS.
