# schattenspiel – Leuchten-Konfigurator

Webseite, auf der Kund:innen eine gefräste Stahl-Leuchte gestalten und den Schattenwurf live in einem 3D-Raum sehen. Später kommen Produktions-Checks und ein Bestellvorgang dazu.

Sprache: **UI komplett Deutsch**. Mit dem Inhaber auf Deutsch kommunizieren.

---

## Das Produkt

- Die Leuchte ist ein **Oktaeder („Raute“)**: oben eine vierseitige Pyramide, unten dieselbe gespiegelt.
- **Maße:**
  - Äquator 420 × 420 mm, Gesamthöhe 600 mm, also 300 mm pro Pyramide.
  - Jede der 8 Flächen ist ein gleichschenkliges Dreieck: Basis 420 mm, Höhe (Schenkelhöhe) ≈ 366 mm.
- Die Flächen sind **gefräste Metallplatten**, Blechstärke laut UI-Text 1,5 mm. Das Frästmuster bestimmt den Schatten an Wänden, Decke und Boden. Genau das ist das Verkaufsargument.
- **Materialien:**
  - Cortenstahl (Standard)
  - Messing gebürstet
  - Schwarz pulverbeschichtet
  - Weiß Klarlack
- Die Flächen sind an den Kanten mit feinen Profilen verbunden. Oben sitzt die Fassung, an der Decke ein Baldachin, dazwischen ein Textilkabel.
- **Kabelfarben (10):** Schwarz, Anthrazit, Grau, Weiß, Sand, Senf, Rostrot, Weinrot, Petrol, Oliv.

---

## Aktueller Stand: V4 (maßgeblich)

- **`index.html`** (HTML + CSS + `<script type="module">`) plus Ordner **`textures/`** mit den Bodentexturen. Beides muss auf den Server.
- **Lokal nur über einen Webserver öffnen**, zum Beispiel mit `python3 -m http.server` oder VS Code Live Server. Per `file://` blockiert Chrome die Bildtexturen, dann bleibt der Boden schwarz.
- Three.js **r180** wird per CDN importiert: `https://cdn.jsdelivr.net/npm/three@0.180.0/build/three.module.js`.
- Kein Build-Schritt, kein Framework, kein Server-Code.
- **Deploy:** Der Inhaber legt die Datei selbst auf einen **Hetzner-Server**. **Kein Serverbudget**, kein Blender oder GPU-Rendering auf dem Server.
- **Zielgeräte:** Desktop. Mobil hat niedrige Priorität, sollte aber nicht kaputtgehen.
- Die Seite füllt genau einen Bildschirm (100vw × 100vh) und scrollt als Ganzes nicht. Nur die rechte Konfigurationsspalte scrollt intern.

### Versionshistorie (alte Versionen bleiben bestehen, neue immer als eigene Version)

| Version | Inhalt | Status |
|---|---|---|
| V1 | Dunkles UI, Three.js r128 per UMD, nur untere Flächen gefräst, harte Schatten | archiviert |
| V2 | Helles Dashboard-UI, **keine Rundungen**, Geist-Schrift; Muster auf „Nur unten / Nur oben / Beide“; weiche Echtzeit-Schatten | archiviert |
| V3 | Three r180 + `three-gpu-pathtracer` (Raytracing im Stillstand) | **verworfen** (siehe unten) |
| V4 | Kein Raytracing, 3D-Technik aus V3; UI im **Apple-Store-Konfigurator-Stil** | **aktuell** |

**Arbeitsweise des Inhabers:** Bei größeren Änderungen will er eine **neue Version**, die alte bleibt unverändert. Wenn er „erstmal debattieren“ sagt, wird nichts gebaut, nur diskutiert.

---

## Architektur (V4, in `index.html`)

### Zustand
Ein globales `state`-Objekt:

```js
state = {
  faces: 'beide' | 'unten' | 'oben',
  patternId, pattern /* Canvas: Lückenmaske */, patternName, fileName,
  repeat: bool, scale /* mm, Breite eines Rapports, Start 300 */, offX, offY /* mm */, border /* Rahmen in mm, Start 20 */,
  material: 'corten' | 'messing' | 'schwarz' | 'weiss',
  bri /* 0..1 */, kelvin, lightMode: 'k' | 'c', lightHex, soft /* 0..1 Schattenweichheit */,
  cable /* Index */, rotate: bool
}
```

`apply*()`-Funktionen übertragen den Zustand in die Szene: `applyLight`, `applySoft`, `applyMaterial`, `applyFaces`, `applyCable` und `updateFace` für das Muster. `updateSpec()` aktualisiert die Zusammenfassung unten rechts.

### Muster-Pipeline (Kernstück)
1. **Vorlage → Lückenmaske:**
   - Hochgeladenes Bild auf einen weißen Grund zeichnen. Pixel mit `min(r,g,b) > 200` gelten als **Lücke**. Weiß und Transparentes werden also gefräst, jede andere Farbe bleibt Stahl.
   - Ergebnis ist ein Canvas, in dem Lücken deckend sind und Stahl transparent ist.
   - Rasterbilder werden auf max. 1400 px verkleinert. SVGs werden auf 1400 px hochgerechnet.
   - Hinweis-Toast, wenn die Vorlage fast gar keine oder fast nur Lücken hat.
2. **Flächen-Textur:**
   - Canvas `FACE_W × FACE_H = 1024 × 893`. Das Seitenverhältnis entspricht 420 × 366 mm, deshalb sind die Pixel in beiden Richtungen maßhaltig (`pxPerMm = W / 420`).
   - Das Dreieck liegt im Canvas mit der **Basis oben** und der **Spitze unten**.
   - `drawSolid()` füllt zuerst alles mit Stahl (weiß).
   - Dann wird auf das **Innendreieck** geclippt. Das Innendreieck entsteht, indem die Ecken zum Inkreismittelpunkt hin verkleinert werden, um den Rahmen in mm.
   - Dort wird das Muster mit `destination-out` ausgestanzt: gekachelt per `createPattern` mit `DOMMatrix` oder als Einzelbild. Außerhalb des Bildes bleibt Stahl.
   - Referenzpunkt für die Platzierung ist die Mitte des umschließenden Rechtecks.
3. Diese Textur ist die **`alphaMap`** (`alphaTest: 0.5`, `DoubleSide`) des gemeinsamen `cutMat`. Alle gefrästen Flächen tragen dasselbe Muster.
4. **UVs:**
   - Untere Flächen: Basis `v=1`, Spitze `v=0`.
   - Obere Flächen haben dieselbe Zuordnung, deshalb erscheint das Muster dort gespiegelt, mit der Spitze oben.
   - Bei „Nur oben“ zeigt die 2D-Vorschau das Dreieck gespiegelt, und das Ziehen in Y ist invertiert.
5. **Vorschau im Panel:** `#preview`. Stahl wird in Materialfarbe eingefärbt (bei Corten mit der Rosttextur), Lücken in Lichtfarbe. Ziehen verschiebt das Muster, Mausrad und Pinch skalieren.

**Beispielmuster** (prozedural, kachelbar, 512 px, mit Standard-Rapport):
- Zellen (Voronoi), 200 mm
- Raute, 90 mm
- Blüte, 150 mm
- Lochblech, 70 mm
- Welle (Schlitze mit Stegen), 130 mm

Keins davon hat Inseln.

### 3D-Szene
- **Raum:** Grundriss 2:1, x −5,4…5,4 m, Höhe 3,3 m, z −1,8…3,6 m. Die Kamera steht im Raum, die Rückwand ist bei z = −2,4. Dazu Sockelleisten.
  - Wände: Reibeputz. Ein Höhenfeld (`plasterH`: weiche Kellen-Wellen + Sandkörner) ergibt Normal-, Farb- und Rauheits-Map.
  - Die UVs der Raumflächen sind in **Metern** (`plane()`), `repeat` heißt also Kacheln pro Meter.
  - Boden: alte abgezogene Kieferndielen. Scan „Wood Floor Worn“ von Poly Haven (CC0), 2K, liegt in `textures/` (Farbe, Normal GL, Rauheit), 1 Kachel = 2 × 2 m.
    - Die Farbtextur ist vorab bearbeitet: 40 % entsättigt und aufgehellt (Gamma 0,78), weil das Original orange lackiert wirkt.
- **Baldachin:** `LatheGeometry` aus einem Profil in Metern: Schale mit Radiuskante und Zugentlastung fürs Kabel.
- **Lens-Flare:** eigene additive Sprites mit `depthTest` aus (`FLARE`, `placeFlare()`), Texturen prozedural.
  - Elemente: Glühen, Beugungsstern (6 Blendenlamellen), anamorphotischer bläulicher Streif, schwacher Regenbogen-Halo und sechseckige Geister mit Farbsaum auf der Achse Birne → Bildmitte.
  - `toneMapped:false`. Geister und Streif haben eine Eigenfarbe (`tint`), der Rest folgt der Lichtfarbe.
  - **Verdeckung:** `bulbVisibility()` schießt pro Frame 32 Strahlen von der Kamera auf die Birnenscheibe. Jeder wird mit den 8 Flächen (`FACE_TRIS`) geschnitten und fragt in der Fräsmaske (`maskData`) ab, ob er auf Stahl oder ein Loch trifft. Der sichtbare Anteil steuert die `opacity` des Flares, er scheint also nur durch die Löcher.
  - Das `Lensflare`-Addon von three ist dafür ungeeignet: Es prüft nur 9 Pixel und ist praktisch immer verdeckt.
  - Die Stärke folgt Helligkeit und Lichtfarbe (in `applyLight`).
  - Wand-, Decken- und Leuchtentexturen werden prozedural auf Canvas erzeugt. **Ausnahme:** der Boden, siehe oben.
- **Leuchte:** Gruppe `lamp` mit Mittelpunkt bei y = 1,75. Das Licht und die Birne sitzen 3 cm tiefer.
- **Geometrie:** Die Dreiecke werden mit N = 24 fein unterteilt, siehe Stolperfallen.
  - Die Kanten sind **Schalenprofile** („rods“, Bogen ±63°), nur nach außen gewölbt, mit eigenem einseitigem `rodMat`. Beim vollen Halbrund bekommen die flachen Flanken noch Birnenlicht ab, das zeigt sich als gestrichelte Linie.
  - **Keine vollen Zylinder mittig auf der Kante.** Deren Innenhälfte wird von der Birne beleuchtet, weil der Schatten-Bias von 4 mm größer ist als der Stab. Sie blitzt dann als heller Strich zwischen den Platten durch und sieht aus wie ein Spalt.
- **Materialien:** `MeshPhysicalMaterial`. Die Parameter setzt `MATS[id].set(material)`:
  - Corten: Rostfarbe, Rauheits-Map und Normal-Map aus dem Rostbild selbst (`rustHeight`, Stärke 1,3), metalness 0,15.
  - Messing: metalness 1, gebürstete Roughness-Map und Normal-Map aus denselben Schleifriefen.
  - Schwarz: Pulver-Normal-Map mit Orangenhaut (0,9). Weiß nutzt dieselbe mit 0,5.
  - Weiß: clearcoat 1.
  - Alle Leuchtenmaterialien bekommen eine PMREM-Umgebung als `envMap`, die grob dem beleuchteten Raum entspricht (helle, warme Wände, dunkler Boden). `envMapIntensity = 1,1 · Helligkeit`. Ohne diese Umgebung bleiben die Außenseiten schwarz, weil die Birne nur von innen strahlt. Dann sieht man weder Material noch Struktur.
  - Auf den Profilen (`rodMat`) gibt es keine Normal-Map, dort sähe sie gestrichelt aus.
  - **Fasen:** Die gefrästen Flächen (`cutMat`) bekommen eine eigene gebackene Normal-Map in Flächen-UV (`bakeFaceNormal()`, `nrmTex`). Sie enthält das Materialrelief aus `MATS[..].bump` und eine Fase von ca. 1 mm an jeder Fräskante: weichgezeichnete Stahlmaske als Höhe, Whiteout-Mischung. Neu gebacken wird nach Muster- oder Materialwechsel, entprellt mit 120 ms.
  - Fassung und Baldachin sind Drehteile mit gebrochenen Kanten.
- **Umgebungslicht** (`state.amb`, 0…1, Start 0,5): Faktor 0…2 auf `HemisphereLight` und `envMapIntensity`.
- **Licht:**
  - Ein `PointLight` mit `castShadow` und physikalischer Einheit (Candela): `intensity = 22·b² + 0,3`.
  - Dazu ein `HemisphereLight` (`0,08 + 0,42·b`) als Ersatz für indirektes Licht.
  - Tone Mapping ACES mit Belichtung 0,82.
  - **Weißabgleich:** Kelvin-Farben werden zu 50 % Richtung Weiß gezogen, sonst ist der Raum orange. Farbiges Licht bleibt unverändert.
- **Weiche Schatten:** Siehe unten. Der Regler „Schattenkante“ stellt den Durchmesser des Leuchtkörpers ein, von ca. 2 mm (Filament) bis 70 mm (Opal): `radius = 0,001 + soft^1,3 · 0,034` m. Der Schattenradius in mrad ist `radius / 0,25 m · 1000`.
- **Schatten-Map:** 1024 pro Würfelseite, bias −0,004, near 0,01, far 12. `shadowMap.autoUpdate = false`, aktualisiert wird nur bei Änderung (`shadowDirty`) oder beim Drehen.
- **Kamera:**
  - Eigene Steuerung ohne OrbitControls, Ziel (0, 1,62, 0).
  - Abstand 2,1–3,4 m, Azimut ±50°, Elevation −4…24°, weich nachgezogen.
  - FOV 55, im Hochformat 64. Pixel Ratio höchstens 1,75.
  - „Drehen“ dreht die Leuchte (0,14 rad/s) und ist standardmäßig **an**.
  - Die Maus steuert **immer** die Kamera, auch im Dreh-Modus. Der Schalter dreht nur die Leuchte automatisch (vom Inhaber so gewünscht).
- **Intro** (`playIntro()`, ca. 3 s, bewusst schnell und clean, **kein Blur**):
  - Wortmarke „schattenspiel“ auf Weiß, die Buchstaben schnellen gestaffelt aus einer Maske (`overflow:hidden`) hoch. Danach fliegt sie per FLIP ins Nav-Logo.
  - Gleichzeitig öffnet sich die Bühne per `clip-path` aus einem Schlitz, und das UI baut sich gestaffelt auf (CSS-Variable `--d`).
  - 3D-Teil: `flyIntro()` fährt die Kamera aus der Raumecke heran und dreht die Leuchte ein. `lightFade` 0→1 zündet das Licht aus dem Dunkel.
  - Der Startzustand hängt an `body.booting`. **Nicht `.pre` nennen**, die Klasse ist schon für die Licht-Presets vergeben.
  - Bei `prefers-reduced-motion` wird das Intro übersprungen.

---

## Technische Stolperfallen (bereits gelöst – nicht wieder einbauen)

1. **Punktlicht-Schatten und Clipping:** Eine Dreiecks-Ecke, die im Würfel-Schattenpass genau in der Blickebene des Lichts liegt (w ≈ 0), führte zu einer falschen Interpolation. Die seitlichen Würfelseiten waren dadurch komplett verdeckt, und die Wände bekamen kein Licht. **Lösung:** die Dreiecke fein unterteilen (N = 24). Diese Unterteilung nicht entfernen.
2. **Alpha im Schatten:** In r180 übernimmt der Schatten-Pass `alphaMap` und `alphaTest` automatisch vom Material. In r128 (V1/V2) war dafür ein `customDistanceMaterial` nötig.
3. **Weiche Echtzeit-Schatten:**
   - `THREE.ShaderChunk.shadowmap_pars_fragment` wird gepatcht: `getPointShadow` wird durch einen Poisson-Filter mit 28 Taps über den **Raumwinkel** ersetzt. `shadow.radius` wird dabei als Winkelradius in Milliradiant interpretiert, pro Pixel zufällig rotiert.
   - Das entspricht physikalisch dem Halbschatten einer Kugelleuchte hinter einem Schirm, der etwa 25 cm entfernt ist.
   - Die Signatur hat in r180 den Parameter `shadowIntensity`. Bei einem Three-Update muss der Patch geprüft werden (String-Replace auf `float getPointShadow(`).
4. **r180 Farbmanagement:** Hex-Farben werden automatisch aus sRGB umgerechnet. Kelvin-RGB wird deshalb mit `setRGB(..., THREE.SRGBColorSpace)` gesetzt, Farbtexturen bekommen `colorSpace = SRGBColorSpace`.
5. **Raytracing (V3) wurde verworfen, nicht wieder vorschlagen, außer der Inhaber fragt danach:**
   - `three-gpu-pathtracer` 0.0.24 lief, war aber zu langsam: Der ganze Rechner wurde träge, und bis zu einem sauberen Bild dauerte es Minuten.
   - `renderScale < 1` erzeugt in dieser Bibliothek ein bleibendes Wurmmuster.
   - Filter auf interne Render-Targets zu setzen, verfälscht die Sobol-Zufallstabelle (`_sobolTarget`).
   - Punktlichter haben dort keinen Radius. Weiche Schatten liefen nur über eine pro Sample verschobene Lichtposition, was viel Rauschen erzeugt.
   - Unity oder Unreal im Browser bringen keine bessere Qualität, sie nutzen dieselbe Grafikschnittstelle (WebGL/WebGPU). Pixel-Streaming kostet Server-GPU-Budget, das es nicht gibt.
6. **Testen in einer Cloud-Sandbox:**
   - Headless-Chromium mit SwiftShader rendert WebGL nur per Software und ist sehr langsam. Screenshots erst nach ca. 30 s machen.
   - Google Fonts und CDNs sind dort oft blockiert. Three.js dann lokal aus `node_modules` einbinden und Font-Anfragen abbrechen.

---

## Design-System V4 (Apple-Store-Konfigurator-Stil)

- **Schrift:** `-apple-system, BlinkMacSystemFont, "SF Pro Text", "SF Pro Display", "Inter", …`. Inter wird als Google Font für Nicht-Apple-Geräte geladen.
- **Typografie:**
  - Headline 40/1,08, weight 600, letter-spacing −0,028em.
  - Abschnittstitel 24 px in **Zweiton**: „Muster. <grau>Wähle eine Vorlage …</grau>“.
  - Fließtext 14 px, Kleingedrucktes 12 px.
- **Farben:**
  - Text `#1d1d1f` / `#6e6e73` / `#86868b`
  - Flächen `#f5f5f7` / `#e8e8ed`
  - Linien `#d2d2d7`
  - Auswahl **Apple-Blau `#0071e3`**
  - Schalter-Grün `#34c759`
  - Akzent Corten `#b4562a`, nur in der Eyebrow-Zeile
- **Radien:** Bühne 22, Karten und Kacheln 14, Editor 18, Segment 9/7, Pills voll rund.
  - Achtung: Bei V2 wollte der Inhaber **keine** Rundungen. Für den Apple-Stil sind sie bewusst wieder drin. Er wurde darauf hingewiesen, dass es sich schnell umstellen lässt.
- **Layout:**
  - Navigationsleiste 52 px, milchglasartig, mit Logo „schattenspiel“ (19 px, weight 600) und Scroll-Spy-Links: Muster, Material, Licht, Kabel.
  - Links die 3D-Bühne als abgerundete Karte, rechts eine Spalte von 440 px mit Intro, Abschnitten und einer festen Zusammenfassung „Deine Leuchte“ unten. Dort ist später der Platz für Preis und „Bestellen“.
- **Elemente:**
  - iOS-Schalter
  - macOS-Schieberegler mit blauer Füllung (CSS-Variable `--p`, gesetzt über `fillRange()`)
  - Umschalter mit gleitendem weißem Knopf (`segSync()`)
  - Materialkarten mit 2 px blauem Rahmen bei Auswahl
  - runde Farbwähler mit blauem Ring
  - in der 3D-Ansicht eine dunkle, milchglasartige Steuerleiste (Drehen, Zoom)
- **Animationen:** dezent, mit Feder-Kurve `cubic-bezier(.32,.72,0,1)`. `prefers-reduced-motion` wird respektiert.
- **Wünsche des Inhabers zum Stil:** elegant, hochwertig, nicht verspielt, gute Typografie, Produkt-Design-Anmutung.

---

## Roadmap (vom Inhaber gewünscht, noch offen)

1. **Produktions-Checks** auf der Flächen-Maske:
   - **Stencil-Check:** Inseln erkennen, also Metallteile ohne Verbindung zum Rahmen, die herausfallen würden. Umsetzbar per Connected-Component-Analyse auf dem Stahl-Anteil innerhalb des Dreiecks.
   - **Mindeststeg-Check:** Stege schmaler als X mm finden, zum Beispiel per Distanztransformation. Rot in der Vorschau markieren.
   - Offene Frage an den Inhaber: **tatsächliche Mindeststegbreite und Blechstärke**.
2. **Projekt-Link** statt Screenshot: Konfiguration teilbar machen. Eigene Uploads bräuchten dafür Speicher. Ohne Server nur Beispielmuster und Parameter in der URL.
3. **Bestellen:** Checkout oder Anfrage, Preis in der Zusammenfassung.
4. Optional: **echte Blechstärke** als Geometrie. Die Lochkanten bekämen dann Tiefe, und dieselben Daten könnten die Grundlage für den **Fräsdatei-Export** (DXF/SVG) sein.
5. Weitere Leuchtenformen sind nicht besprochen.

---

## Konventionen für künftige Sessions

- Neue Versionen als **neue Datei bzw. neues Artefakt**, alte nicht überschreiben, außer der Inhaber will ausdrücklich ein Update.
- **Eine Datei** bevorzugen. Externe Skripte nur von jsdelivr, cdnjs oder unpkg mit fester Version.
- Keine Server-Abhängigkeiten, kein Build-Schritt, solange nicht ausdrücklich gewünscht.
- Code-Kommentare auf Deutsch, Bezeichner auf Englisch oder Deutsch wie bestehend.
- Vor großen Richtungswechseln kurz nachfragen. Kleine Designentscheidungen selbst treffen und im Anschluss erwähnen.
- Ehrlich sagen, was getestet wurde. In der Sandbox kann oft nur die Software-Darstellung geprüft werden.
