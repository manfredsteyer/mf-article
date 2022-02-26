# Module Federation

_Manfred Steyer, GDE_

Nowadays, Micro Frontend architectures are highly discussed. Webpack's Module Federation provides a solid way for building such architectures. As the Angular CLI uses webpack, Module Federation can also be easily used in Angular-based solutions.

This article first gives an overview on Micro Frontends and Module Federation. We also outline some recommendations about it. Then, we describe how to use Module Federation together with Angular and the CLI.

The [example](https://github.com/manfredsteyer/mf-angular-showcase) used in this article can be found [here](https://github.com/manfredsteyer/mf-angular-showcase).

## Micro Frontend Architectures

The idea behind Micro Frontend architectures is to split a huge frontend into several individual ones. These resulting small and hence less complex frontends are called Micro Frontends. For instance, an eProcurement system could consist of the following Micro Frontends:

![Example: Micro Frontends for an eProcurement system](overview.png)

The main purpose for using such an architecture to scale teams: Now, a small team can be assigned to each Micro Frontend. These teams can work in an autonomous way and do their own decisions. Most importantly, they can deploy new versions of their Micro Frontend without waiting for or coordinating with other teams. Or, to put it in another way: This architecture helps getting the agility of small teams for large-scale software projects.

To allow the individual teams to work in an autonomous way, each micro frontend typically represents a sub-domain of the business the software system is built for. The borders between these domains should be chosen in a way that prevents use cases overlapping several domains. While preventing this is not always possible, such a situation should be rather the exception than the rule.

Another advantage of Micro Frontend architectures we want to strike out here is the possibility to dramatically increase build times. As each team only modifies its own micro frontend, they only need to compile their own slice of the overall system. Together with build caches provides by products like [Bazel](https://bazel.build/) or [Nx](https://nx.dev/), this allows to further improve build times.

## Runtime Integration

Splitting large-scale frontends into several smaller ones is only one side of the coin: We also need to find a way to provide an integrated user experience. However, as micro frontends are deployed individually by different teams, integration needs to happen at runtime.

An easy and also solid approach towards runtime integration is connecting several applications via hyperlinks. We find this approach in product suites like Office 365 or the Google platform. Unfortunately, this approach also defeats the advantages of Single Page Applications (SPAs). Hence, hyper-link integration is mainly used in situations where the user sticks for quite a long time with one (Micro) Frontend.

Another approach is providing a shell -- an application that loads the individual Micro Frontends on demand. While this looks a lot like traditional lazy loading, the main difference is that the loaded Micro Frontends are not known at compile time. 

Traditionally, several technical approaches have been used for building shells. Using iframes, loading further SPAs, and loading web components on demand are some popular examples. However, besides some other obvious drawbacks, these approaches demand for a lot of non-trivial infrastructure code for loading and orchestrating different Micro Frontends. Also, they dramatically increase the bundle sizes.

Fortunately, webpack's Module Federation discussed in the next section provides a solid solution for building a shell. 

## Module Federation for Runtime Integration

Module Federation, introduced with webpack 5, helps with runtime integration by allowing to load code from a separately built and deployed application. Basically, it allows to do something like this:

```typescript
const Component = await import('other-app/component');

const NgModule = await import('other-app/ng-module');
```

While we are loading ECMAScript modules from separately built and deployed applications, from Angular's perspective, this looks like lazy loading. That means Angular doesn't even recognize that something special is going on. Module Federation takes care of the heavy lifting underneath the covers. 

This seems to be the biggest advantage of Module Federation: As Angular is not aware of dealing with separately built Micro Frontends, we can use Angular as it is intended to be used. We don't need to tweak Angular in unsupported ways, nor do we need any additional infrastructure code for orchestrating different Micro Frontends.

To make all of this possible, Module Federation defines two types of applications:

* **Remotes** are applications that offer some of their ECMAScript modules to other applications. In our case, a Micro Frontend would be such a remote.
  
* **Hosts** load ECMAScript Modules provided by remotes. In our case, the shell is the host.

Also, Module Federation allows to share libraries at runtime. That means even though the shell and the Micro Frontends are separately compiled, we only need to load Angular, RxJS, and other libraries once. 

Obviously, when sharing libraries at runtime, there can be version mismatches. Fortunately, Webpack Module Federation comes with several strategies for dealing with such situations. However, as a later section will explain, we recommend to prevent them upfront by using a respective code organization.

Unfortunately but also understandably, shared libraries are not tree-shakable: As Micro Frontends are loaded at runtime, we cannot predict upfront which parts of our shared libraries they will use. This will increase the overall bundle size. 

Also, there is another drawback of runtime integration and hence also of module federation: They turn compile time dependencies into runtime dependencies. Hence, we don't know at compile time, if individual Micro Frontends will work together at runtime. This makes E2E tests even more important.

## Monorepos for Organizing Source Code

In general, the Angular team recomments organizing your source code in a monorepo -- one code repository that contains all your source code in the form of applications and libraries. While libraries can be reused across different applications they also allow to substructure a huge system in smaller parts.

As all the libraries are co-located in the same repository, they don't need to be versioned and published via a registry. Also, there is always exactly one version of each library. This _ever green version policy_ not only reduces the effort for managing the provided source code but also prevents version conflicts. 

Also, several monorepo implementations allow for restricting yourself to just installing one version per third-party npm package. This self-restriction also prevents version conflicts.

This all not only sounds interesting in theory but is also an approach heavily and successfully used at Google where there is just one monorepo containing all the source code. While this may not be (easily) possible at each company, using a monorepo to substructure and organize one sole project, a product suite, or several related projects already gives you a fair amount of the advantages outlined. This is something, the author sees in a lot in companies he is consulting. 

However, to succeed with monorepos you need two things: respective processes and respective tooling. The processes, for instance, define how to deal with breaking changes that affect several applications and hence several teams. The tooling helps with tasks like controlling which parts of the monorepo are allowed to access which other parts in order to enforce loosely coupling. 

While loosely coupled libraries are in general vital for an architecture that can evolve, it's one of the fundamental ideas of Micro Frontends: They must not depend on each other in order to allow individual teams to work autonomously. 

Besides this, for succeeding with a mono repo, we also need tooling that allows for incremental builds and tests: As building and testing all applications and libraries contained in a mono repo can be far too time-consuming, such tooling only rebuilds and retests code that has been affected by changes. 

The build system [Nx](https://nx.dev) -- an initiative started by former Angular core team members -- provides all these features and more. As Nx is built on top of the Angular CLI, it feels quite natural to Angular developers: All the known commands like ``ng generate`` or ``ng build`` are still in place and provide additional possibilities needed for enterprise-scale development and monorepos.

Also, Google open-sourced a version of its build tool for managing the huge internal monorepo. While this open version called [Bazel](https://bazel.build/) even allows for more fine grained control over code units compiled separately, it is also more complicated to configure. Also, unlike Nx, it cannot be seen as a drop-in replacement for the Angular CLI.

## Recommendations: When and how to use Micro Frontends and Module Federation?

While building a modular monolith is quite easy with a mono repo, we also see that the Micro Frontend approach provides several advantages for large-scale applications. This architecture is especially interesting when

* different teams need to work in an autonomous way
* compilation times needs to be reduced

In these cases Module Federation is a quite tempting technology. However, in order to prevent version mismatches upfront, the Angular team strongly advises to stick with a monorepo. 

While in general, Module Federation also allows to integrate applications from different repositories and comes with strategies for compensating for version mismatches, we think the possibility of having version mismatches is too risky in the case of Single Page Applications that are downloaded into the browser. This situation is different for Micro Services and server-side rendered web applications: In these cases the concreate frameworks and versions are hidden behind an URL and don't need to be loaded nor executed locally. 

> While the author of this document has supported companies using both, mono repos and multiple repos, he advices to make sure to know about the consequences but also to define measures for dealing with version conflicts before moving to a multi-repo scenario. 

Even though several Micro Frontends are managed in a monorepo, they can be built and deployed separately. If they share libraries, it's, however, important to always deploy all the changed applications together. If there is a release branch, you at least need to deploy all micro frontends that have been changed there. Otherwise, version conflicts may occur. 

Tools like Nx provide ways to find out about all applications affected by changes. Also, the CI/CD process might use Nx to get this information and to only deploy the respective micro frontends. As Nx allows for incremental builds, the CI/CD process only needs to build and test affected applications.


![](example.png)

![](loading-via-mf.png)

![](monorepo-structure.png)

# Adding Module Federation

ng add ...

```javascript
// projects/catalog/webpack.config.js

const ModuleFederationPlugin = require("webpack/lib/container/ModuleFederationPlugin");
const mf = require("@angular-architects/module-federation/webpack");
const path = require("path");
const share = mf.share;

const sharedMappings = new mf.SharedMappings();
sharedMappings.register(
  path.join(__dirname, '../../tsconfig.json'),
  []);

module.exports = {
  output: {
    uniqueName: "catalog",
    publicPath: "auto"
  },
  optimization: {
    runtimeChunk: false
  },   
  resolve: {
    alias: {
      ...sharedMappings.getAliases(),
    }
  },
  experiments: {
    outputModule: true
  },
  plugins: [
    new ModuleFederationPlugin({
        library: { type: "module" },

        name: "catalog",
        filename: "remoteEntry.js",
        exposes: {
            './CatalogModule': './projects/catalog/src/app/catalog/catalog.module.ts',
        },        

        shared: share({
          "@angular/core": { singleton: true, strictVersion: true, requiredVersion: 'auto' }, 
          "@angular/common": { singleton: true, strictVersion: true, requiredVersion: 'auto' }, 
          "@angular/common/http": { singleton: true, strictVersion: true, requiredVersion: 'auto' }, 
          "@angular/router": { singleton: true, strictVersion: true, requiredVersion: 'auto' },

          ...sharedMappings.getDescriptors()
        })
        
    }),
    sharedMappings.getPlugin()
  ],
};
```


```javascript
// projects/approval/webpack.config.js

const ModuleFederationPlugin = require("webpack/lib/container/ModuleFederationPlugin");
const mf = require("@angular-architects/module-federation/webpack");
const path = require("path");
const share = mf.share;

const sharedMappings = new mf.SharedMappings();
sharedMappings.register(
  path.join(__dirname, '../../tsconfig.json'),
  []);

module.exports = {
  output: {
    uniqueName: "approval",
    publicPath: "auto"
  },
  optimization: {
    runtimeChunk: false
  },   
  resolve: {
    alias: {
      ...sharedMappings.getAliases(),
    }
  },
  experiments: {
    outputModule: true
  },
  plugins: [
    new ModuleFederationPlugin({
        library: { type: "module" },

        // For remotes (please adjust)
        name: "approval",
        filename: "remoteEntry.js",
        exposes: {
            './ApprovalModule': './projects/approval/src/app/approval/approval.module.ts',
        },        
        
        shared: share({
          "@angular/core": { singleton: true, strictVersion: true, requiredVersion: 'auto' }, 
          "@angular/common": { singleton: true, strictVersion: true, requiredVersion: 'auto' }, 
          "@angular/common/http": { singleton: true, strictVersion: true, requiredVersion: 'auto' }, 
          "@angular/router": { singleton: true, strictVersion: true, requiredVersion: 'auto' },

          ...sharedMappings.getDescriptors()
        })
        
    }),
    sharedMappings.getPlugin()
  ],
};
```

```javascript
// projects/shell/webpack.config.js

const ModuleFederationPlugin = require("webpack/lib/container/ModuleFederationPlugin");
const mf = require("@angular-architects/module-federation/webpack");
const path = require("path");
const share = mf.share;

const sharedMappings = new mf.SharedMappings();
sharedMappings.register(
  path.join(__dirname, '../../tsconfig.json'),
  []);

module.exports = {
  output: {
    uniqueName: "shell",
    publicPath: "auto"
  },
  optimization: {
    runtimeChunk: false
  },   
  resolve: {
    alias: {
      ...sharedMappings.getAliases(),
    }
  },
  experiments: {
    outputModule: true
  },
  plugins: [
    new ModuleFederationPlugin({
        library: { type: "module" },

        shared: share({
          "@angular/core": { singleton: true, strictVersion: true, requiredVersion: 'auto' }, 
          "@angular/common": { singleton: true, strictVersion: true, requiredVersion: 'auto' }, 
          "@angular/common/http": { singleton: true, strictVersion: true, requiredVersion: 'auto' }, 
          "@angular/router": { singleton: true, strictVersion: true, requiredVersion: 'auto' },

          ...sharedMappings.getDescriptors()
        })
        
    }),
    sharedMappings.getPlugin()
  ],
};
```

```typescript
// projects/shell/src/app/app-routing.module.ts

const routes: Routes = [
  {
    path: '',
    redirectTo: 'home',
    pathMatch: 'full'
  },
  {
    path: 'home',
    component: HomeComponent
  },
  {
    path: 'catalog',
    loadChildren: () => loadRemoteModule({
      type: 'module',
      remoteEntry: 'http://localhost:4201/remoteEntry.js',
      exposedModule: './CatalogModule'
    }).then(esm => esm.CatalogModule)
  },
  {
    path: 'approval',
    loadChildren: () => loadRemoteModule({
      type: 'module',
      remoteEntry: 'http://localhost:4202/remoteEntry.js',
      exposedModule: './ApprovalModule'
    }).then(esm => esm.ApprovalModule)
  },  
  {
    path: 'about',
    component: AboutComponent
  }
];

@NgModule({
  imports: [RouterModule.forRoot(routes)],
  exports: [RouterModule]
})
export class AppRoutingModule { }
```

## Communication


![](login01.png)

![](login02.png)


```typescript
// projects/auth/src/lib/auth.service.ts

import { Injectable } from '@angular/core';

@Injectable({
  providedIn: 'root'
})
export class AuthService {

  currentUser = '';

  constructor() { }

  login(userName: string, password: string): void {
    this.currentUser = userName;
    // No password check for the sake of demonstration
    // "Login for honest users TM"
  }

  logout(): void {
    this.currentUser = '';
  }
}
```

```typescript
// projects/auth/src/public-api.ts

export * from './lib/auth.service';
```

```javascript   
{
  [...]
  "compilerOptions": {
    "paths": {
      "@demo/auth": [
        "projects/auth/src/public-api.ts"
      ]
    },
    [...]
  },
  [...]
}
```

```javascript
// all webpack.config.js files
const sharedMappings = new mf.SharedMappings();
sharedMappings.register(
  path.join(__dirname, '../../tsconfig.json'),
  ['@demo/auth']);
```

