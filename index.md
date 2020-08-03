# Fragen zu Versioning in Webpack Module Federation

## requiredVersion vs. version

> Ich habe das Gefühl, dass es in sehr vielen Szenarien egal ist, ob man ``requiredVersion`` oder ``version`` verwendet. Warum hat man sich entschieden, hier zwei Properties einzuführen? 

Dazu hilft es vielleicht erst zu erklären wie Shared Modules intern funktioniert. Sie bestehen nämlich aus zwei Aspekten.

Wenn `shared` im `ModuleFederationPlugin` verwendet wird passiert das Folgende: Das `ModuleFederationPlugin` ist nur eine Fasade und leitet `shared` direkt an das `SharePlugin` weiter. Das `SharePlugin` verwendet dann selbst wieder das `ConsumeSharedPlugin` und das `ProvideSharedPlugin`. Dabei geht `version` and das `ProvideSharedPlugin` und `requiredVersion` an.

Das führt uns zu den zwei Aspekten von Shared Modules: Consuming and Providing.

Nehmen wir folgendes Beispiel: `shared: { "some-lib": { import: "some-lib-import" } }`, das ist zwar eher ungewönlich aber hilft zum Erkären.
(Üblich wäre zum Beispiel `shared: ["a-package"]` das entspricht `shared: { "a-package": { import: "a-package" } }`)

Auf der Consuming Seite passiert das Folgende: Immer wenn ein `Module` für eine `Dependency` (eine `Dependency` is zum Beispiel: `import "some-lib"` oder `require("some-lib")`) ermittelt werden soll, wird vorher geprüft ob es nicht ein `ConsumeSharedModule` werden soll. Das passiert wenn der `request` (hier: `some-lib`) dem Wert aus der `shared` Konfiguration entspricht. Falls `requiredVersion` nicht gesetzt ist, wird sie nun auch ermittelt. Dazu wird die die nächste `package.json` ausgehend von **dem Modul in dem die `Dependency` steht** nommen und der `request` in `dependencies`, `peerDependencies`, etc. gesucht. Das heist `import "some-lib"` führt nicht immer zu dem gleichen `ConsumeSharedModule`, sondern es können mehrere existieren abhängig von der ermittelten `requiredVersion`.

Auserdem hat ein `ConsumeSharedModule` auch ein `fallback` Module. Das wird durch den Wert der `import` Option erstellt. Hier würde also ein normales Modul für `some-lilb-import` erstellt.

Auf der Providing Seite passiert Ähnliches: Immer wenn ein `Module` für eine `Dependency` erstellt wurde, bei der `request` dem Wert der `import` Option entspricht, wird zusätzlich auch ein `ProvideSharedModule` erstellt, welches allerdings global für alle Entrypoints automatisch geladen wird. Diese `ProvideSharedModule` registriert das erstellte `Module` als `SharedModule`. Dabei wird `version` verwendet um das `Module` zu registrieren. Wurde keine `version` angegeben wird die `version` aus der `package.json` **des erstellen `Module`s** verwendet (Hinweis: Das ist nicht die gleiche `package.json` wie auf der Consuming Seite).

Zur Laufzeit werden also zunächst alle Shared Module von allen Builds via `ProvideSharedModule` registriert und dann, wenn die entsprechnenden Shared Module benutzt werden, wieder via `ConsumeSharedModule` geladen. Beim Laden der Module wird `requiredVersion` mit der `version` von allen registrieren Modulen verglichen. Dabei passiert in Abhängigkeit von verscheidenen anderen Optionen (`singleton`, `strictVersion`) eins von den Folgenden:

* Die höchste `version`, die noch mit `requiredVersion` kompatible ist wird benutzt. Gibt es keine wird das `fallback` Modul verwendet. (Standard)
* Die höchste `version`, die noch mit `requiredVersion` kompatible ist wird benutzt. Gibt es keine wird höchste `version` verwendet und eine Warning ausgegeben. (`strictVersion: false`)
* Eine bereits geladene oder die höchste `version` wird benutzt. Wenn sie nicht mit `requiredVersion` kompatible ist wird eine Warning ausgegeben. (`singleton: true`)
* Wenn die bereits geladene bzw. die höchste `version` mit `requiredVersion` kompatible ist wird sie benutzt. Sonst führt das Laden zu einem Fehler. (`singleton: true, strictVersion: true`)
* Es gibt noch mehr Fällen für `import: false`, `requiredVersion: false` oder `version: false`, aber die sind weniger relevant.

Zusammengefasst `version` und `requiredVersion` machen sehr verschiedene Dinge. Beide verwenden sogar verscheidene Syntax. `version` erwartet eine Versionsnummer, z. B. `1.2.3`. `requiredVersion` erwartet eine Version Range, z. B. `^1.2.3`, `1.2.x - 3.4` oder `^1.2.3 || >=2.1.4 <3`

> Wann würde man ``requiredVersion`` nutzen?

Für `npm` packages, am besten nie. Sie wird ja automatisch aus der `package.json` ermittelt.

Für packages in monorepos auch nicht. Da stehen die benötigten Versionen ja auch in der `package.json`.

Es kann relevant werden wenn `resolve.modules` in webpack verwendet wird und so andere (virtuelle?) "packages" verwendet werden.
Hier stehene der Versionen nicht in der `package.json` und man muss sie angeben mit `requiredVersion`.

Webpack wirft eine Fehlermeldung wenn die `requiredVersion` nicht automatisch ermittelt werden kann, dann muss sie angegeben werden.

Es gibt auserdem noch einen Sonderfall in dem `requiredVersion` angegeben werden muss:
Man kann auch relative `request`s als Shared Module verwenden: `shared: { "./src/module": { shareKey: "special-module", requiredVersion: "^1.2" }`.
Dann kann keine `package.json` verwendet werden.

> Wann würde man ``version`` nutzen?

Vergleichbar wie `requiredVersion`.

Nicht verwenden wenn sie in der `package.json` steht.

Wenn webpack eine Fehlermeldung wirft, dass `version` nicht automatisch ermittelt werden kann, dann muss man sie angeben.

Für relative `request`s also Shared Modul kann es auch Sinn machen eine `version` anzugeben. Automatisch wird die `version` von der Applikation hier sonst verwendet.

## Singleton

> Sehe ich das richtig: Wenn ich in einer Anwendung einen Singleton definiere, sollte diese shared Library auch in allen Anderen Anwendungen ein Singleton sein.

Ja. `singleton` beeinflusst nur die Consuming Seite. In den aller meisten Fällen muss `singleton: true` überall angegeben werden.

Man könnte auf die Idee kommen das webpack doch automatisch ein Modul überall als Singleton konsumieren könnte wenn dies in einer Anwendung angegeben ist.
Allerdings ist ein Ziel von Module Federation, dass jeder Build/Anwendung möglichst isoliert ist und möglichst wenig von anderen Builds beeinflusst wird.
Ein falsches `singleton: true` würde so alle anderen Anwendungen zerstören, während mit der aktuellen Funktionsweise nur die Anwendung mit dem falschen `singleton: true` zerstört wird.

## Eager und statische Imports

### Konfiguration

> Shell nutzt folgende Konfiguration:

> ```
> shared: { 
>     "rxjs": "rxjs", 
>     "useless-lib": {
>         eager: true
>     }
> }
> ```

> Shell importiert useless-lib über einen statischen Import:

> ```
> import * as useless_static from 'useless-lib';
> 
> console.debug('useless_static', useless_static.version);
> ```

### Ergebnis

> ```
> main.js:950 Uncaught Error: Shared module is not available for eager consumption: webpack/sharing/consume/default/useless-lib/useless-lib
>     at Object.__webpack_modules__.<computed> (main.js:950)
>     at __webpack_require__ (main.js:542)
>     at eval (main.ts:2)
>     at Module../src/main.ts (main.js:195)
>     at __webpack_require__ (main.js:542)
>     at main.js:1087
>     at main.js:1091
> ```

### Fragen

> Ist das ein Bug und/oder mache ich hier was falsch?

Es sind vielleicht falsche Erwartungen an `eager: true`.

Normalerweise werden Shared Modules asynchron konsumiert. Das heist Shared Module werden asynchron vom Container geladen und auch der Container kann asynchron geladen werden.
Das macht Sinn, da asynchrones Laden es ermöglicht die Module in separate Dateien auszulagern die dann nur bei Bedarf geladen werden.

Mit Eager Shared Modules werden sie synchron konsumiert. Das heist Shared Module werden synchron vom Container geladen und auch der Container muss synchron geladen werden.
Shared Module müssen also schon vorab geladen werden und nicht erst bei Bedarf. Im Web macht das aufgrund des zusätzlichen unnötigen Downloads natürlich keinen Sinn.

Auch wenn es kaum Sinn macht, so kann es verwendet werden:

* Das Shared Module muss in allen Builds mit `eager: true` markiert sein (Ähnlich wie `singleton: true`).
* Container müssen synchron geladen werden. z. B. mit `remoteType: "var"` anstatt `remoteType: "script"` (Standard)
  Bei `remoteType: "var"` mussen die Script-Tags alle remotes bereits vor der Applikation geladen sein.

In Node.js kann `eager` allerdings verwendet werden. Dort wird eh `remoteType: "commonjs-module"` verwendet, das ist synchron und auch schadet es kaum zuviel Code zu laden.

## 2 Container + Singletons

> Derzeit scheint es, als ob Singletons shareScope-übergreifend gelten. 

> Ist das so gewollt oder ein Bug?

Das ist nicht so gewollt.

Allerdings muss man dazu auch erklären das shareScopes lokal pro Build existieren und **nicht** global sind.
Für jedes `remote` kann man konfigurieren welche der lokalen shareScopes die `default` shareScope des remotes werden soll.
Das heist `shareScope` nur ein `shared` zu konfigurieren macht selten Sinn.

> Nachfolgend eine Beschreibung meines Experiments dazu.

### Konfiguration

> - Shell
>   - useless-lib: ^1.0.0*
>   - singleton: true
>   - shareScope: default*

> - Mfe1
>   - useless-lib: ^1.0.1*
>   - singleton: true
>   - shareScope: default*

> - Mfe2
>   - useless-lib: ^2.0.0*
>   - singleton: true
>   - shareScope: other

> - Mfe3
>   - useless-lib: ^2.0.1*
>   - singleton: true
>   - shareScope: other

> \* Nicht in ``webpack.config.json`` konfiguriert, sondern Standardwert bzw. Version in ``package.json`` hinterlegt.

Ohne die `version` von `useless-lib` kann man schlecht was sagen. Diese wird in dieser Konfiguration ja automatisch aus der `package.json` ermittelt.
Ich nehme einfach mal an, dass die `version` = `requiredVersion` ist ohne das `^`.

Das führt also zu folgenden ShareScopes:

```
A (Shell.default, Mfe1.default, Mfe2.default, Mfe3.default)
  - useless-lib 1.0.0 (from: Shell)
  - useless-lib 1.0.1 (from: Mfe1)
B (Mfe2.other)
  - useless-lib 2.0.0 (from: Mfe2)
C (Mfe3.other)
  - useless-lib 2.0.1 (from: Mfe3)
```

Und das würde zu zu folgenden konsumierten Versionen führen:

```
Shell: useless-lib 1.0.1 from A (^1.0.0: ok)
Mfe1: useless-lib 1.0.1 from A (^1.0.1: ok)
Mfe2: useless-lib 2.0.0 from B (^2.0.0: ok)
Mfe3: useless-lib 2.0.1 from C (^2.0.1: ok)
```

Scheint aber nicht so zu sein...

=> Da ist ein Bug in `SharePlugin` und es gibt den `shareScope` nicht weiter an `ConsumeSharedPlugin` und `ProvideSharedPlugin`.

### Ergebnis

> - Alle nutzen Version 2.0.1
> - Es kommt für Shell und mfe1 eine Singleton-Warnung

### Fragen

> - Ist das so gewollt?
> - Sollten Singletons pro shareScope gelten oder wirklich über alle shareScopes hinweg?

### Variation

> Wenn man überall ``singleton:true`` weglässt, nutzt der eine Container Version ``1.0.1`` und der andere Version ``2.0.1``.

## Strict Version ohne Singleton

> Macht ``strictVersion:true`` ohne ``singleton:true`` Sinn? 

> Ich sehe derzeit keinen Effekt.

Liegt vermutlich daran, dass in dem Fall `strictVersion: true` der Standard ist.

`strictVersion` macht dann einen Unterschied wenn keine passende Version gefunden wird. Also eigentlich nur wenn man selbst keine bereitstellt (`import: false`), oder selbst eine unpassende bereitstellt (unwahrscheinlich). Dann wird entweder ein Fehler geworfen (`strictVersion: true`) oder die höchste Version verwendet und eine Warning ausgegeben (`strictVersion: false`).

Der Standard für `strictVersion` wird automatisch so gewählt, dass die Anwendung möglich keinen Fehler wirft, also es werden in-kompatible Versionen versucht zu verwenden.
Es macht Sinn `strictVersion: true` zu setzen wenn man eine gute Fehlerbehandlung für Fehler in z. B. `import()` hat.


