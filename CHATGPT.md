# Untersuchung der ChatGPT-Antwort

## Ergebnis

Die geprüfte Antwort eignet sich ohne Korrektur nicht als Implementierungsplan.
Ihre Bestandsaufnahme und viele statische Codebeobachtungen sind korrekt.
Ihre Priorisierung ist falsch, weil sie Messinfrastruktur und spekulative Performance-Arbeit vor belegte Bugs stellt.
Die Einstufungen „Außergewöhnlich“ und „Hoch“ sind für die meisten Performance-Befunde nicht belegt.

Die stärksten Befunde sind stale Cosmetic-Caches, case-sensitive HTTP-Header-Lookups und defektes `$replace`-Parsing.
Diese Befunde beschreiben beobachtbare Correctness-Fehler und keine bloßen Optimierungsmöglichkeiten.
Die Benchmark-Defekte und das fehlende Release-Testgate sind ebenfalls belegt, stehen aber hinter Runtime-Correctness.

## Umfang und Nicht-Ziele

Die Untersuchung prüfte alle 15 Befunde gegen den aktuellen Checkout und maßgebliche externe Primärquellen.
Sie umfasste Repository-Metadaten, Git-Historie, Core-Engine, Integrationen, Benchmarks, Workflows und relevante Tests.
Sie trennte beobachtete Codefakten von abgeleiteten Auswirkungen und ungemessenen Performance-Behauptungen.
Sie änderte keinen Code und führte keine Tests, Benchmarks, Installationen oder sonstigen Mutationen aus.

## Repository-Status

- Der geprüfte `master`-Head ist `707d94549af0eef867253239fcd6122f7f8f38dd`.
- Der Head-Commit trägt den Betreff `docs(repo): document fork context and codebase orientation`.
- Der lokale Fork ist einen Commit vor `upstream/master` und keinen Commit zurück.
- Die einzige Fork-spezifische Dateiänderung ist das hinzugefügte `AGENTS.md`.
- Der untersuchte Programmcode entspricht deshalb dem geprüften Upstream-Stand.
- GitHub meldet `MikeeI/adblocker` als öffentlichen Fork mit `master` als Default-Branch.
- Die Repository-Metadaten sind unter https://github.com/MikeeI/adblocker verfügbar.
- Der direkte Fork-Vergleich ist https://github.com/MikeeI/adblocker/compare/ghostery:master...MikeeI:master.
- GitHub meldete `Branch not protected` für den klassischen Branch-Protection-Endpunkt.
- GitHub lieferte eine leere Liste von Repository-Rulesets.
- Für `master` existiert derzeit keine erzwungene Branch-Regel und kein verpflichtender Statuscheck.
- Das Feld `private: true` in `package.json` verhindert Package-Publishing und macht GitHub nicht privat.

## Bewertung der Befunde

### 1. Leeren `$badfilter`-Zustand cachen

- Status: Der Mechanismus ist belegt, aber Priorität und Lösungsskizze sind überzogen.
- `NetworkFilterBucket.isFilterDisabled()` nutzt `badFiltersIds === null` als nicht initialisierten Zustand.
- Ein leerer `badFilters`-Container kehrt zurück, ohne `null` zu ersetzen.
- Jeder spätere passende Kandidat wiederholt `badFilters.getFilters()` und erzeugt ein neues leeres Array.
- `NetworkFilterBucket.update()` invalidiert den Cache bereits nach einem Update.
- Die wiederholte Arbeit liegt im Netzwerk-Matching-Pfad.
- Keine Messung belegt eine außergewöhnliche Produktwirkung dieser begrenzten Allocation.
- Ein gecachtes leeres `Set` ist einfacher und sicherer als `undefined | null | Set<number>`.
- Ein permanenter Test auf die private Anzahl von `getFilters()`-Aufrufen wäre Implementation-Detail-Testtheater.
- Ein Verhaltenstest sollte Matching nach Hinzufügen und Entfernen einer `$badfilter`-Regel schützen.
- Ein Benchmark sollte die Performance-Wirkung mit repräsentativen Requests und Listen messen.

Evidenz: `packages/adblocker/src/engine/bucket/network.ts:38-86,138-163`.

### 2. Unbetroffene Indizes bei Updates nicht neu aufbauen

- Status: Der algorithmische Mechanismus ist teilweise belegt, während der Schweregrad workloadabhängig bleibt.
- `FilterEngine.update()` leitet jede Netzwerkänderung an alle aktivierten Netzwerk-Buckets weiter.
- Buckets ohne neue passende Filter erhalten leere `newFilters`-Arrays.
- `ReverseIndex.update()` leert seinen Cache, bevor eine tatsächliche Änderung feststeht.
- `ReverseIndex.update()` deserialisiert, tokenisiert, serialisiert und baut bei irrelevanten Updates neu auf.
- `FiltersContainer.update()` deserialisiert vorhandene Filter auch ohne wirksame Änderung.
- `FiltersContainer.update()` baut seine kompakte Repräsentation bei einem No-op nicht zwingend neu auf.
- Ein Guard für leere Additions und leere Removals ist sicher und sinnvoll.
- Ein Removal-Guard muss die tatsächliche Existenz einer ID feststellen, bevor er den Cache erhält.
- Die reale Wirkung hängt von Updatefrequenz und Indexgröße ab.
- Interne Rebuild-Zähler wären außerhalb eines Benchmark-Harness Testtheater.
- Verhaltenstests sollten Matching vor und nach gemischten Additions und Removals vergleichen.

Evidenz: `packages/adblocker/src/engine/engine.ts:621-746`.
Evidenz: `packages/adblocker/src/engine/reverse-index.ts:511-691`.
Evidenz: `packages/adblocker/src/engine/bucket/filters.ts:184-251`.

### 3. Benchmark- und Memory-Harness reparieren

- Status: Die Benchmark-Defekte sind vollständig belegt.
- Die aktive Microbenchmark-Liste enthält keinen Benchmark für `engine.match()`.
- Ein TODO nennt einen realen Engine- und Network-Request-Benchmark ausdrücklich als fehlend.
- Der Harness lädt genau ein EasyList-Fixture und ein uBlock-Resources-Fixture.
- Der Memory-Benchmark behält `serialized`, aber keine starke Referenz auf die erzeugte Engine.
- Die nächste Garbage Collection kann daher gerade die zu messende Engine freigeben.
- Der Memory-Benchmark meldet nur `process.memoryUsage().heapTotal`.
- `heapTotal` misst V8-Heap-Kapazität und nicht den retained Engine-Speicher.
- Der Harness ignoriert `heapUsed`, `external`, `arrayBuffers` und `rss`.
- Node dokumentiert diese Felder unter https://nodejs.org/api/process.html#processmemoryusage.
- Der Vergleich bezeichnet rohe Bytewerte als `MB`, ohne sie umzurechnen.
- Der Harness speichert relative Fehlermarge und Sample-Anzahl, ignoriert beide aber beim Vergleich.
- Der Harness überschreibt `.bench.json` nach jedem Lauf.
- Kein GitHub-Actions-Workflow führt den Benchmark-Harness aus.
- Diese Arbeit verbessert spätere Entscheidungen, repariert aber kein Produktionsverhalten.

Evidenz: `bench/run_benchmark.ts:9-23,48-75,93-195,250-316`.
Evidenz: `bench/micro.ts:20-118`.

### 4. Releases an Tests binden und Browserjobs trennen

- Status: Das fehlende Release-Testgate ist belegt, aber die Workflow-Skizze benötigt weitere Planung.
- `.github/workflows/tests.yml` läuft ausschließlich für `pull_request`.
- `.github/workflows/release.yml` läuft bei Pushes auf `master`.
- Der Release-Workflow installiert, baut, lintet und ruft `yarn release` ohne Tests auf.
- Der Release-Workflow besitzt keine Abhängigkeit zum Test-Workflow.
- `master` besitzt derzeit weder klassische Branch Protection noch ein Repository-Ruleset.
- Ein direkter Push kann den Release-Workflow daher ohne erzwungenes Testergebnis erreichen.
- Die Testmatrix läuft mit Node.js 22, 24 und 26 auf `ubuntu-latest`.
- Jeder Matrixjob baut, lintet, installiert Playwright und Puppeteer und ruft `yarn test` auf.
- Playwright-Browser werden gecacht, während der Workflow den Puppeteer-Download nicht cachet.
- Playwright dokumentiert gezielte Browserinstallation unter https://playwright.dev/docs/browsers.
- Yarn dokumentiert `-p` für paralleles `workspaces foreach` unter https://yarnpkg.com/cli/workspaces/foreach.
- Nur sieben Workspaces definieren ein `test`-Script.
- `yarn test` testet deshalb nicht alle 13 Packages als gleichartige Einheiten.
- Extended Selectors kombiniert Mocha-Unit-Tests mit einem echten Playwright-Browsertest.
- Das Puppeteer-Package enthält einen Mocha-Integrationstest, der einen echten Browser startet.
- Eine browserfreie Unit-Matrix kann das aktuelle Root-Script `yarn test` nicht unverändert aufrufen.
- Ein Split benötigt klare Package-Zuständigkeit, Artefakte, Caches und Release-Dependency-Semantik.
- Die Richtung stimmt, aber die angehängte Skizze ist nicht direkt anwendbar.

Evidenz: `.github/workflows/tests.yml:1-59`.
Evidenz: `.github/workflows/release.yml:1-52`.
Evidenz: `package.json:56-67`.
Evidenz: `packages/adblocker-extended-selectors/package.json:46-52`.
Evidenz: `packages/adblocker-puppeteer/test/index.test.ts:65-68`.

### 5. Cosmetic-Stylesheet-Caches nach Updates invalidieren

- Status: Dies ist ein belegter Correctness-Bug und der wichtigste Codebefund.
- `CosmeticFilterBucket` cachet `baseStylesheet` und `extraGenericRules` lazy.
- Der Cache hängt von `genericRules` und `unhideIndex` ab.
- `CosmeticFilterBucket.update()` ändert beide Owner, ohne die Cachewerte zu invalidieren.
- Ein vor dem Update befüllter Cache kann neue Regeln auslassen oder entfernte Regeln behalten.
- Ein Unhide-Update kann Regeln außerdem in der falschen Cachegruppe belassen.
- Der Kommentar bezeichnet die Daten nur zwischen Updates als stabil und impliziert damit Invalidierung.
- Ein relevantes Update muss beide Cachewerte vor dem nächsten Lookup zurücksetzen.
- Konservative Invalidierung bei jedem Removal ist korrekt, weil IDs nicht nach Container vorklassifiziert sind.
- Ein Verhaltenstest muss den Cache befüllen, Regeln ändern und mit einer frischen Engine vergleichen.
- Der Test sollte eine Generic-Hide-Regel und eine gruppenverändernde Unhide-Regel abdecken.

Evidenz: `packages/adblocker/src/engine/bucket/cosmetic.ts:203-305,567-623`.

### 6. HTTP-Response-Header case-insensitive lesen

- Status: Dies ist ein belegter Correctness-Bug mit bedingter Produktreichweite.
- `getHeaderFromDetails()` vergleicht `header.name === headerName`.
- Die Caller übergeben kleingeschriebene Namen für `content-disposition`, `content-length` und `content-type`.
- Mixed-case Response-Header werden deshalb übersehen.
- Der CSP-Helfer in derselben Datei normalisiert Headernamen bereits mit `toLowerCase()`.
- RFC 9110 definiert HTTP-Feldnamen unter https://www.rfc-editor.org/rfc/rfc9110.html#section-5.1 als case-insensitive.
- Ein übersehenes `Content-Disposition` kann ein Attachment in den Response-Filtering-Pfad lassen.
- Ein übersehenes `Content-Length` kann die Größenablehnung bis nach begonnenem Buffering verzögern.
- Ein übersehenes `Content-Type` kann die `$replace`-Eignungsentscheidung verändern.
- Das bestehende 10-MiB-Limit begrenzt den finalen Buffer, verhindert aber keine unnötige Vorarbeit.
- Aktuelle Tests decken Lowercase-Namen, aber keine äquivalenten Mixed-case-Namen ab.
- Der Helfer sollte beide Namen normalisieren oder einen normalisierten Inputvertrag besitzen.
- Tests müssen identische Entscheidungen für Lowercase und Mixed-case aller drei Header beweisen.
- Der Pfad benötigt `enableHtmlFiltering=true` und `browser.webRequest.filterResponseData`.
- `Config` setzt `enableHtmlFiltering` standardmäßig auf `false`.
- Die WebExtension-Integration dokumentiert `filterResponseData` als Firefox-only.
- Der Bug ist real, aber seine Produktschwere hängt von aktiviertem Firefox-HTML-Filtering ab.

Evidenz: `packages/adblocker-webextension/src/index.ts:120-139,179-227,518-532`.
Evidenz: `packages/adblocker/src/config.ts:44-72`.

### 7. `$replace`-Escapes und Delimiter korrekt parsen

- Status: Dies ist ein belegter Parser-Bug, dessen vorgeschlagener Patchumfang unvollständig ist.
- `isHexLiteral()` prüft Großbuchstaben mit `code <= 65 && code <= 70`.
- Der Ausdruck akzeptiert zahlreiche Werte unter `A` und verwirft `B` bis `F`.
- Die Längenberechnung für `\u{...}` nutzt `close - pos + 3`.
- Diese Berechnung verwirft gültige Codepoint-Escapes mit einer bis sechs Stellen.
- Der Code validiert den Inhalt der Codepoint-Klammern nicht vollständig als hexadezimal.
- Der Helfer entfernt außerdem gültige moderne JavaScript-Escapes wie `\p{...}` und `\k<...>`.
- `getFilterReplaceOptionValue()` liefert drei Strings auch bei fehlenden schließenden Slashes.
- Ein fehlerhafter Wert wie `/foo` kann dadurch als regulärer Ausdruck akzeptiert werden.
- AdGuard definiert `/regexp/replacement/modifiers` unter https://adguard.com/kb/general/ad-filtering/create-own-filters/#replace-modifier.
- Nur Großbuchstabenvergleich und Klammerarithmetik zu reparieren lässt weitere Parserfehler bestehen.
- Die Verifikation benötigt gültige Hex-Escapes, Unicode-Codepoints, ungültige Spans und fehlende Delimiter.
- Sie muss escaped Commas und Slashes in RegExp- und Replacement-Segmenten erhalten.
- Full-Filter-Parsing muss fehlerhafte Werte verwerfen, statt sie unbeabsichtigt zu gültigen Modifiers zu machen.

Evidenz: `packages/adblocker/src/filters/network.ts:477-640,1744-1760`.

### 8. Query-Parsing bis zum ersten `$removeparam`-Kandidaten verschieben

- Status: Die unnötige Arbeit ist belegt, aber ihr Nutzen ist ungemessen.
- `FilterEngine.match()` erzeugt `searchParamLiteral` und `URLSearchParams` vor `removeparams.matchAll()`.
- Requests mit Query zahlen diese Parsing-Kosten auch ohne passenden `$removeparam`-Filter.
- Ein vorgeschalteter Reverse-Index-Lookup kann den gesamten Query-Mutationspfad ohne Kandidaten überspringen.
- Die Änderung muss Filterpriorität, Exceptions, Eventreihenfolge und Output-URL erhalten.
- Ein Benchmark benötigt Query-Requests ohne Regeln, ohne Kandidaten, mit Matches, Exceptions und Inversions.
- Verhaltenstests sollten Output und Events schützen, statt `URLSearchParams`-Konstruktionen zu zählen.

Evidenz: `packages/adblocker/src/engine/engine.ts:1443-1540`.

### 9. Temporäre Arrays bei Cosmetic-Hostname-Tokens vermeiden

- Status: Die Allocation ist belegt, aber ihr Schweregrad nicht.
- `getCosmeticsFilters()` spreadet Typed Arrays in JavaScript-Arrays.
- Der Code nutzt außerdem `flatMap()` und `Array.from()` für Ancestor-Tokens.
- `Uint32Array.from()` kopiert das kombinierte JavaScript-Array wieder in ein Typed Array.
- Der Top-Frame-Pfad könnte `hostnameTokens` ohne Ancestors direkt wiederverwenden.
- Der Ancestor-Pfad könnte ein korrekt dimensioniertes Typed Array mit `set()` füllen.
- Die Änderung bleibt nur verhaltensgleich, wenn der Consumer das Tokenarray nicht verändert.
- Die Messung muss Top-Frame-Aufrufe von verschachtelten Frames unterscheiden.
- Tests auf Allocation-Anzahlen wären Implementation-Detail-Testtheater.

Evidenz: `packages/adblocker/src/engine/bucket/cosmetic.ts:331-395`.

### 10. Temporäre Offsetliste bei kalten Reverse-Index-Lookups entfernen

- Status: Die Cold-Path-Allocation ist belegt, aber keine Evidenz macht sie zur Priorität.
- `ReverseIndex.iterBucket()` erzeugt bei einem Cache-Miss `filtersIndices`.
- Der Code scannt den Bucket einmal zum Sammeln und erneut zum Deserialisieren der Offsets.
- Negative exakte Lookups erzeugen trotzdem ein leeres Array.
- Positive Buckets landen nach dem kalten Lookup im Cache.
- Die Kosten hängen daher von einmaligen Tokens, Cachekonfiguration und Bucketgröße ab.
- Direkte Deserialisierung im ersten Scan kann Offsetarray und zweiten Loop entfernen.
- Die Cursorposition muss für jeden serialisierten Filter explizit gesetzt bleiben.
- Kalte und warme Workloads müssen vor einer Änderung getrennt gemessen werden.

Evidenz: `packages/adblocker/src/engine/reverse-index.ts:726-789`.

### 11. Callback- und Event-Allocations in Matching-Pfaden reduzieren

- Status: Einige Allocations und ein Cleanup-Defekt sind belegt, die Gesamtwirkung bleibt unbekannt.
- `FilterEngine` erzeugt wiederholt Callbacks mit `this.isFilterExcluded.bind(this)`.
- Ein stabiles Arrow-Property könnte Engine-Ownership ohne per-call Binding erhalten.
- `EventEmitter.emit()` erhält ein Rest-Argument-Array, bevor Listenerlosigkeit feststeht.
- `unsubscribe()` entfernt einen Callback, lässt aber ein leeres Eventarray in der Map.
- Ein späteres Emit dieses Events kann einen Microtask mit null Listenern einplanen.
- Das Löschen leerer Map-Einträge ist korrekter Lifecycle-Cleanup und benötigt keinen Benchmarkbeleg.
- Event-Payloads sollten nur bei verfügbarer Listenerprüfung vermieden werden, ohne API-Semantik zu ändern.
- Tests sollten `on`, `once`, `unsubscribe`, asynchrone Ausführung und Argumentidentität schützen.
- Tests sollten weder private Callback-Identität noch interne Mapgröße behaupten.

Evidenz: `packages/adblocker/src/events.ts:24-82,90-140`.
Evidenz: `packages/adblocker/src/engine/engine.ts:1384-1475`.

### 12. Binary-Merge-Deduplizierung gegen Hashkollisionen schützen

- Status: Dies ist ein belegtes Correctness-Risiko in einem optionalen und ausdrücklich schwachen Merge-Modus.
- `ReverseIndex.merge()` speichert genau einen serialisierten Filter pro Hash-Key.
- `FiltersContainer.merge()` speichert ebenfalls genau einen serialisierten Filter pro Hash-Key.
- Keine Implementierung vergleicht Länge und Bytes bei gleichen Hashwerten.
- Eine Kollision kann still einen unterschiedlichen Filter ersetzen.
- Die Source-Kommentare delegieren Kollisionsresistenz ausdrücklich an `hashFunc`.
- Der Default-Fallback ist CRC32 und bietet keine kollisionsfreie Deduplizierung.
- Keine Produktintegration in diesem Repository nutzt `FilterEngine.merge()` mit `useBinaryMerge=true`.
- Die Produktpriorität hängt daher von externen Library-Callern ab.
- Ein erzwungener konstanter Hash ist der richtige Verhaltenstest.
- Der Test muss unterschiedliche Filter erhalten und identische serialisierte Filter weiter deduplizieren.
- Eine Collision-Chain mit Bytevergleich könnte Normalpfad-Performance und Exaktheit erhalten.

Evidenz: `packages/adblocker/src/engine/reverse-index.ts:97-165,167-239`.
Evidenz: `packages/adblocker/src/engine/bucket/filters.ts:30-122`.
Evidenz: `packages/adblocker/src/engine/engine.ts:262-289`.

### 13. Source-Domain-Parsing nur mit klarer Lifecycle-Zuständigkeit wiederverwenden

- Status: Die wiederholte Arbeit existiert, aber die vorgeschlagene Cachelösung ist spekulativ.
- `Request.fromRawDetails()` kann `tldts-experimental.parse()` für Ziel und Quelle aufrufen.
- Browserintegrationen liefern gewöhnlich URLs statt vorberechneter Hostname- und Domainwerte.
- Der Konstruktor berechnet Source-Hostname- und Source-Entity-Hashes eager.
- Ziel-Hostname- und Ziel-Entity-Hashes werden lazy berechnet.
- Viele Requests einer Seite können dieselben Source-Host- und Domaininformationen teilen.
- Das Repository enthält keine Messung für Wiederverwendung oder Cache-Hit-Rate.
- Ein globaler unbegrenzter Source-URL-Cache würde Speicher- und Data-Retention-Probleme erzeugen.
- Ein sicherer Cache benötigt einen Owner wie Tab, Frame, Request-Factory oder begrenzten LRU.
- Navigation und Frame-Replacement müssen Lifecycle-gebundene Daten invalidieren.
- Produkt-Traces müssen Wiederverwendung belegen, bevor diese State-Komplexität entsteht.

Evidenz: `packages/adblocker/src/request.ts:280-303,323-379`.
Evidenz: `packages/adblocker-webextension/src/index.ts:90-105`.

### 14. Unveränderliche Extended-Selector-Argumente einmal pro AST kompilieren

- Status: Wiederholte Compilation ist belegt, aber die Cachegrenze benötigt Präzision.
- `matches()` ruft für stabile Selector-Argumente wiederholt `parseRegex()` auf.
- `matchCSSProperty()` ruft wiederholt `parseCSSValue()` auf und kann einen weiteren RegExp erzeugen.
- Diese Arbeit kann für jedes Kandidatenelement erneut entstehen.
- Der aktuelle RegExp-Parser erlaubt nur die nicht zustandsbehafteten Flags `i`, `m` und `u`.
- Gecachte RegExp-Objekte sind deshalb unter dem aktuellen Flagvertrag semantisch sicher.
- DOM-Matches, Computed Styles, Location-Ergebnisse und XPath-Ergebnisse dürfen nicht gecacht werden.
- `isExtendedSelector()` berechnet außerdem wiederholt eine abgeleitete AST-Klassifikation.
- Diese Klassifikation hängt vom Argument `insideHasSelector` ab.
- Ein naiver `WeakMap<AST, boolean>` kann deshalb ein kontextuell falsches Ergebnis liefern.
- Ein Classification-Cache benötigt AST-Identität und Context-Bit als Key.
- Der bestehende XPath-Expression-Cache ist ein separater globaler Stringcache.
- Browser-Profiling muss relevante Compilation-Kosten belegen, bevor weiterer Cache-State entsteht.

Evidenz: `packages/adblocker-extended-selectors/src/eval.ts:12-25,62-106,141-275,448-510`.

### 15. Peak-Allocations beim Startup vor einem Builder-Redesign profilieren

- Status: Die Allocations sind real, aber keine Messung trägt das vorgeschlagene Redesign.
- `parseFilters()` startet mit `list.split('\n')` und hält ein Array aller Zeilen.
- `ReverseIndex.update()` hält während des Aufbaus Filter-Tokenarrays.
- Der Code erzeugt ein Suffixarray für jeden Token-Lookup-Slot.
- Er speichert Token und Filteroffset als JavaScript-Tupel, bevor er die Daten flach ablegt.
- Diese Strukturen können Peak Memory bei Startup und großen Updates erhöhen.
- Ihre absoluten Kosten für das vollständige Produkt-Filterset sind unbekannt.
- Ein cursorbasierter Parser kann CRLF, Continuations, Preprocessors und Debug-Zeilennummern beschädigen.
- Ein Prefix-Sum-Builder für den Reverse Index würde erhebliche Implementierungskomplexität hinzufügen.
- Node.js- und Chromium-Startup müssen vor einem Redesign getrennt gemessen werden.
- Die Messung benötigt `heapUsed`, `external`, `arrayBuffers`, `rss`, Peak-Dauer und finale Größe.

Evidenz: `packages/adblocker/src/lists.ts:153-260`.
Evidenz: `packages/adblocker/src/engine/reverse-index.ts:511-681`.

## Adversariale Synthese

### Stärkste Befunde

1. Befund 5 ist ein direkter stale-cache Correctness-Defekt in einem öffentlichen Updatepfad.
2. Befund 6 verletzt den HTTP-Header-Vertrag und kann Response-Filtering-Guards umgehen.
3. Befund 7 beschädigt gültige `$replace`-RegExps und akzeptiert fehlerhafte Werte.
4. Befund 3 beweist, dass der aktuelle Benchmark mehrere Optimierungen nicht zuverlässig priorisieren kann.
5. Befund 12 ist bei Nutzung des optionalen Binary Merge ein reales Silent-Data-Loss-Risiko.

### Schwächste Befunde

- Befunde 9 und 10 identifizieren begrenzte Allocations ohne gemessene nutzersichtbare Kosten.
- Befund 11 kombiniert sinnvollen Cleanup mit spekulativer Event-Payload-Optimierung.
- Befund 13 schlägt einen Cache ohne Hit-Rate, Owner, Lifetime und Invalidation vor.
- Befund 14 zeigt wiederholte Compilation, aber keine belegte Browserrelevanz.
- Befund 15 schlägt komplexe Startup-Umbauten ohne Peak-Memory-Evidenz vor.

### Harte Einwände

- Die ursprüngliche Empfehlung stellt Benchmarkarbeit und Mikrooptimierungen vor belegte falsche Outputs.
- Die Einstufung „Außergewöhnlich“ für Befund 1 besitzt weder Cost Model noch Benchmarkbeleg.
- Ein Tri-State-Cache für Befund 1 schafft unnötige Zustände, obwohl ein leeres `Set` das Ergebnis darstellt.
- Private Call-Count-Tests für Befunde 1, 2, 8, 9, 10 und 11 wären brüchiges Testtheater.
- Der Workflow-Vorschlag ignoriert gemischte Unit- und Browserverantwortung in bestehenden Package-Scripts.
- Der Parser-Vorschlag repariert nur zwei sichtbare Defekte und lässt fehlerhafte Delimiter weiter zu.
- Große Caches für Befunde 13 und 14 verschieben Kosten in Invalidation und Memory ohne belegten Nettonutzen.

## Risiken und Annahmen

- Befund 5 bleibt dormant, wenn nach dem ersten cacheerzeugenden Lookup kein Cosmetic-Update erfolgt.
- Befunde 6 und 7 haben geringere Produktwirkung, wenn Firefox-HTML-Filtering deaktiviert bleibt.
- Befund 12 besitzt keine aktuelle Produktwirkung, wenn kein externer Caller `useBinaryMerge` aktiviert.
- Befunde 1 und 2 steigen in der Priorität bei häufigen In-place-Listenupdates.
- Befund 8 steigt bei vielen erlaubten Query-Requests und seltenen `$removeparam`-Kandidaten.
- Befunde 9, 10 und 14 steigen nur, wenn Produktprofile dominante Allocation- oder CPU-Kosten zeigen.
- Vollständige Produktlisten können anderes Cache-, Token-, Collision- und Startup-Verhalten als EasyList zeigen.
- Runtime-Reproduktion kann zusätzliche Event-, Parser- oder Serialization-Constraints sichtbar machen.

## Erforderliche Verifikation vor der Implementierung

1. Cosmetic-Caches befüllen, Generic-Hide- und Unhide-Regeln aktualisieren und mit einer frischen Engine vergleichen.
2. Mixed-case-Formen von `Content-Type`, `Content-Length` und `Content-Disposition` prüfen.
3. Gültige und ungültige `$replace`-Escapes über vollständiges `NetworkFilter.parse()` testen.
4. Unvollständige `$replace`-Werte ohne erforderliche Separatoren verwerfen.
5. Hashkollisionen im Binary Merge erzwingen und unterschiedliche Filter erhalten.
6. Match-only- und Request-plus-Match-Benchmarks mit realen Produktlisten und Requestdaten ergänzen.
7. Kalte und warme Engine-Zustände getrennt messen.
8. Additions, Removals, No-ops und irrelevante Removal-IDs getrennt messen.
9. Retained Memory in einem isolierten Prozess mit stark erreichbarer Engine messen.
10. `heapUsed`, `external`, `arrayBuffers`, `rss`, serialisierte Bytes, Samples und Unsicherheit erfassen.
11. Jeden Package-Test vor dem CI-Umbau einem Unit-, Integration- oder Browser-Owner zuordnen.
12. Den Releasejob an exakt den Commit binden, der die verpflichtenden Quality-Jobs bestanden hat.

## Empfohlene Reihenfolge

1. Behavioral Regressions für Befunde 5, 6 und 7 hinzufügen.
2. Befunde 5, 6 und 7 bei ihren aktuellen Ownern reparieren.
3. Produktnutzung von Firefox-HTML-Filtering und optionalem Binary Merge klären.
4. Befund 12 reparieren, wenn Binary Merge zum unterstützten Produkt- oder Public-API-Vertrag gehört.
5. Match-, Update- und Retained-Memory-Benchmark-Harness reparieren.
6. Eine Produktbaseline mit identischen Listen, Requests, Runtime-Versionen und Cachezuständen erzeugen.
7. Befund 1 mit dem einfachsten Empty-Result-Cache umsetzen, wenn der Benchmark relevante Kosten zeigt.
8. Zuerst den eindeutigen No-input-Guard aus Befund 2 umsetzen.
9. Befund 8 umsetzen, wenn Query-Parsing im gemessenen Allow-Pfad relevant ist.
10. Befunde 9 bis 15 nur nach bestandenem Performance-Finding-Gate erwägen.
11. CI-Package-Ownership und Release-Gating unabhängig von Runtime-Optimierungen neu ordnen.

## Ohne neue Evidenz nicht empfohlene Änderungen

- Keinen unbegrenzten Cache für negative Reverse-Index-Lookups hinzufügen.
- Den Reverse Index nicht ohne belegten algorithmischen Engpass ersetzen oder nach WebAssembly verschieben.
- Kompression nicht ohne getrennte Startup-, Memory- und Serialization-Messungen global aktivieren.
- Integrity Checks nicht für Geschwindigkeit deaktivieren.
- Nicht ohne Buildzeit-Evidenz von Yarn und Lerna zu einem anderen Taskrunner migrieren.
- Puppeteer, Playwright und Electron nicht hinter einer spekulativen gemeinsamen Abstraktion vereinheitlichen.
- Das öffentliche `BlockingResponse`-Objekt nicht poolen, weil Caller es behalten können.
- Keine globalen Source-URL- oder Selector-Result-Caches ohne klare Ownership und Invalidation ergänzen.

## Geprüfte Bereiche ohne weiteren belastbaren Befund

- Die öffentliche Core-API ist vor allem eine bewusste Exportoberfläche und kein Runtime-Hotspot.
- Network-Filter-Matching prüft viele Options bereits vor teurem Pattern-Matching.
- Network-Filter kompilieren RegExps lazy.
- Request-Zieltokens und Zielhashes nutzen bereits Lazy Computation und einen wiederverwendbaren Tokenbuffer.
- Serialization nutzt vordimensionierte Typed-Array-Buffer und gemeinsame Encoder- und Decoder-Owner.
- Die Electron-Integration nutzt bereits stabile Callbacks und eine `WeakMap` für Session-Kontexte.
- Der HTML-Streaming-Core besitzt bereits Größenlimit und konservatives Teardown.
- Der belastbare HTML-Filtering-Defekt liegt in der vorgelagerten Response-Header-Erkennung.
- Playwright-Wrapper-Unit-Tests rechtfertigen keine vollständige Entfernung von Playwright aus CI.
- Extended Selectors benötigt weiterhin einen echten Chromium-Browsertest.

## Primärquellen

- GitHub-Repository-Metadaten: https://github.com/MikeeI/adblocker
- Fork-Vergleich: https://github.com/MikeeI/adblocker/compare/ghostery:master...MikeeI:master
- Node.js-Memory-Metriken: https://nodejs.org/api/process.html#processmemoryusage
- Playwright-Browserinstallation: https://playwright.dev/docs/browsers
- Yarn-Workspace-Ausführung: https://yarnpkg.com/cli/workspaces/foreach
- HTTP-Feldnamen: https://www.rfc-editor.org/rfc/rfc9110.html#section-5.1
- AdGuard-`$replace`-Syntax: https://adguard.com/kb/general/ad-filtering/create-own-filters/#replace-modifier
