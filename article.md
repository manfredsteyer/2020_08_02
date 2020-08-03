# Dealing with Version Mismatch in Micro Frontends based upon Webpack Module Federation

Webpack Module Federation makes it easy to load separately compiled code like microfrontends. It even allows to share libraries among them. This prevents that the same library has to be loaded several times.

However, there might be situations where several microfrontends and the shell need different versions of a shared library. Also, these versions might not be compatible to each other.

For dealing with such cases, Module Federation provides several options. In this article I present these options by looking at different scenarios. The [source code](https://github.com/manfredsteyer/module_federation_shared_versions) for these scenarios can be found in my [GitHub account](https://github.com/manfredsteyer/module_federation_shared_versions).

> Big thanks to [Tobias Koppers](https://twitter.com/wSokra), founder of webpack, for answering several questions about this topic and for proof reading this article.


## Example Used Here

To demonstrate how Module Federation deals with different versions of shared libraries, I use a simple shell application known from the other parts of this article series. It is capable of loading microfrontends into its working area:

![Shell loading microfrontends](./static-all-1-0-0.png)

The microfrontend is framed with the red dashed line. 

For sharing libraries, both, the shell and the microfrontend use the following setting in their webpack configurations:

```javascript
new ModuleFederationPlugin({
   [...],
   shared: ["rxjs", "useless-lib"]
})
```

If you are new to Module Federation, you can find an explanation about it [here](https://www.angulararchitects.io/aktuelles/the-microfrontend-revolution-module-federation-in-webpack-5/). 

The package [useless-lib](https://www.npmjs.com/package/useless-lib) is a dummy package, I've published for this example. It's available in the versions ``1.0.0``, ``1.0.1``, ``1.1.0``, ``2.0.0``, ``2.0.1``, and ``2.1.0``. In the future, I might add further ones. These versions allow to simulate different kinds of version mismatches. 

To indicate the installed version, ``useless-lib`` exports a ``version`` constant. As you can see in the screenshot above, the shell and the microfrontend display this constant. In the shown constellation, both use the same version (``1.0.0``) and hence they can share it. Hence, ``useless-lib`` is only loaded once.

However, in the following sections we will examine what happens if there are version mismatches between the ``useless-lib`` used in the shell and the one used in the ``microfrontend``. This also allows me to explain different concepts Module Federation implements for dealing with such situations.


## Semantic Versioning by Default

For our first variation, let's assume our ``package.json`` is pointing to the following versions:

Lib          | Shell  | MFE1 
-------------|--------|-------
useless-lib  | ^1.0.0 | ^1.0.1

This leads to the following result:

![](static--^1-0-0--vs--1-0-1.png)

Module Federation decides to go with version ``1.0.1`` as this is the highest version compatible with both applications according to rules of semantic versioning (^1.0.0 means, we can also go with a higher minor and patch versions).


## Fallback Modules for Incompatible Versions

Now, let's assume we've adjusted our dependencies in ``package.json`` this way:

Lib          | Shell  | MFE1 
-------------|--------|-------
useless-lib  | ~1.0.0 | 1.1.0

Both versions are not compatible to each other (``~1.0.0`` means, that only a higher patch version but not a higher minor version is acceptable).

This leads to the following result:

![Using Fallback Module](static--~1-0-0--vs--1-1-0.png)

This shows, that Module Federation uses different versions for both applications. In our case, each application falls back to its own version which is also called the fallback module.

<!--
FRAGE: Wenn ich hier auf einer oder auf beiden Seiten ``strictVersion:false`` setze, ändert sich nichts am Verhalten. Ist das so gewollt?

Wenn ich hier auf einer oder auf beiden Seiten ``strictVersion:true`` setze, bekomme ich auch keine Warnung. Ist das so gewollt?
-->

## Differences With Dynamic Module Federation

Interestingly, Module Federation's behavior is a bit different when it comes do Dynamic Module Federation. The reason is that dynamic remotes are not known at program start and hence Module Federation cannot draw their versions into consideration during its initialization phase.

For explaining this difference, let's assume the shell is loading the microfrontend dynamically and that we have the following versions:

Lib          | Shell   | MFE1 
-------------|---------|-------
useless-lib  | ^1.0.0  | ^1.0.1

While in the case of classic (static) Module Federation, both applications would agree upon using version ``1.0.1`` during the initialization phase, here in the case of dynamic module federation, the shell does not even know of the microfrontend in this phase. Hence, it can only choose for its own version: 

![Dynamic Microfrontend falls back to own version](dynamic--1-0-0--vs--1-0-1.png)

If there were other static remotes (e. g. microfrontends), the shell could also choose for one of their versions according to semantic versioning, as shown above.

Unfortunately, when the dynamic microfrontend is loaded, module federation does not find an already loaded version which is compatible to ``1.0.1``. Hence, the microfrontend falls back to its own version ``1.0.1``.

On contrary, let's assume the shell has the highest compatible version:

Lib          | Shell   | MFE1 
-------------|---------|-------
useless-lib  | ^1.1.0  | ^1.0.1

In this case, the microfrontend would decide to use the already loaded one:

![Dynamic Microfrontend uses already loaded version](dynamic--1-1-0--vs--1-0-1.png)

To put it in a nutshell, in general, it's a good idea to make sure your shell provides the highest compatible versions when using dynamic remotes. 

## Singletons

Falling back to another version is not always the best solution: Using more than one version can lead to unforeseeable effects when we talk about libraries holding state. This seems to be always the case for your leading application framework/ library like Angular, React or Vue.

For such scenarios, Module Federation allows us to define libraries as **singletons**. Such a singleton is only loaded once. 

If there are only compatible versions, Module Federation will decide for the highest one as shown in the examples above. However, if there is a version mismatch, singletons prevent Module Federation from falling back to an further library version.

For this, let's consider the following version mismatch:

Lib          | Shell   | MFE1 
-------------|---------|-------
useless-lib  | ^2.0.0  | ^1.1.0

Let's also consider we've configured the ``useless-lib`` as a singleton:

```javascript
// Shell
shared: { 
  "rxjs": {}, 
  "useless-lib": {
    singleton: true,
  }
},
```

Here, we use an advanced configuration for defining singletons. Instead of a simple array, we go with an object where each key represents a package. 

If one library is used as a singleton, you will very likely set the singleton property in every configuration. Hence, I'm also adjusting the microfrontend's Module Federation configuration accordingly:

```javascript
// MFE1
shared: { 
    "rxjs": {},
    "useless-lib": {
        singleton: true
    } 
}
```

To prevent loading several versions of the singleton package, Module Federation decides for only loading the highest available library which it is aware of during the initialization phase. In our case this is version ``2.0.0``: 

![Module Federation only loads the highest version for singletons](static-singleton-warning.png)

However, as version ``2.0.0`` is not compatible to version ``1.1.0`` according to semantic versioning, we get a warning. If we are lucky, the federated application works even though we have this mismatch. However, if version ``2.0.0`` introduced braking changes we run into, our application might fail.

In the latter case, it might be beneficial to fail fast when detecting the mismatch by throwing an example. To make Module Federation behaving this way, we set ``strictVersion`` to ``true``: 


```javascript
// MFE1
shared: { 
  "rxjs": {},
  "useless-lib": {
    singleton: true,
    strictVersion: true
  } 
}
```

The result of this looks as follows at runtime:

![Version mismatches regarding singletons using strictVersion make the application fail](static-singleton-strict-error.png)


## Accepting a Version Range

There might be cases where you know that a higher major version is backwards compatible even though it doesn't need to be with respect to semantic versioning. In these scenarios, you can make Module Federation accepting a defined version range.

To explore this option, let's one more time assume the following version mismatch:

Lib          | Shell   | MFE1 
-------------|---------|-------
useless-lib  | ^2.0.0  | ^1.1.0

Now, we can use the ``requiredVersion`` option for the ``useless-lib`` when configuring the microfrontend:

```javascript
// MFE1
shared: { 
  "rxjs": {},
  "useless-lib": {
    singleton: true,
    strictVersion: true,
    requiredVersion: ">=1.1.0 <3.0.0"
  } 
}
```

According to this, we also accept everything having ``2`` as the major version. Hence, we can use the version ``2.0.0`` provided by the shell for the microfrontend:

![Accepting incompatible versions by defining a version range](singleton-requiredVersion.png)

## Using Different Scopes With shareScope

Every shared library is put into a scope. In most cases, we only use one scope called ``default``. However, there might be edge cases where setting up several scopes with their own libraries and versions might be beneficial. 

In this case, Module Federation will decide for a shared version as well as for possible fallbacks per scope.

For this, let's assume we have two more microfrontends, called MFE2 and MFE3:

Lib          | Shell   | MFE1   | MFE2   | MFE 3
-------------|---------|--------|--------|-------
useless-lib  | ^1.0.0  | ^1.0.1 | ^2.0.0 | ^2.0.1

Let's also assume, MFE2 and MFE3 are provided by another division and that for this reason, we decided to use a different scope for them:

```javascript
// MFE2 + MFE3
shared: { 
  "rxjs": {},
  "useless-lib": {
    shareScope: "other"
  } 
}
```

On the other side, the shell and MFE1 are using a simple array for specifying the shared libraries:

```javascript
// Shell + MFE1
shared: ["rxjs", "useless-lib"]
```

As we don't specify a scope for them, Module Federation is using the ``default`` here.

Now, when starting the application, we see that Module Federation decides for one version within the ``default`` scope and for an other version within the ``other`` scope. In both cases, the usual rules described in this article are applied. Hence, we get version ``1.0.1`` for the shell and MFE1:

![Version used for shell and MFE1 in default scope](scope-1.png)

On the other side, we get version ``2.0.1`` for MFE2 and MFE3:

![Version used for MFE2 in other scope](scope-2.png)
![Version used for MFE3 in other scope](scope-3.png)

## Conclusion

Module Federation brings several options for dealing with different versions and version mismatches. Most of the time, you don't need to do anything, as it uses semantic versioning to decide for the highest compatible version. If a remote needs an incompatible version, it falls back to such one by default. 

In cases where you need to prevent loading several versions of the same package, you can define a shared package as a singleton. In this case, the highest version known during the initialization phase is used, even though it's not compatible to all needed versions. If you want to prevent this, you can make Module Federation to throw an exception by using the ``strictVersion`` option.

Also, you can ease the requirements for a specific version by defining a version range using ``requestedVersion``. For advanced scenarios, you can even define several scopes where each of them can get its own version.