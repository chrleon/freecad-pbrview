# PbrView

En levende render av FreeCAD-dokumentet, i et eget vindu ved siden av
modellen. Principled-aktig BSDF, miljøbelysning, kastskygger og filmisk
tonemapping, tegnet med OpenGL 4.1 inne i FreeCADs egen prosess.

**Det er en render, ikke en viewport.** Den har ingen plukking, så du kan
ikke velge flater, måle eller modellere i den. Det skal FreeCADs egen
visning gjøre. Denne er til å se på, og til å ta bilder av.

Navigasjonen styrer FreeCADs kamera i stedet for et lokalt, så de to
vinduene følger hverandre begge veier. Roterer du i FreeCAD, følger
renderen med. Roterer du i renderen, følger FreeCAD med.

## Installasjon

Lenk makroen og ikonet inn i FreeCADs makromappe. FreeCAD 1.1 bruker en
versjonert mappe:

```bash
ln -sfn "$PWD/PbrView.FCMacro" ~/Library/Application\ Support/FreeCAD/v1-1/Macro/
ln -sfn "$PWD/PbrView.svg" ~/Library/Application\ Support/FreeCAD/v1-1/Macro/
```

Riktig sti får du alltid med `FreeCAD.getUserMacroDir(True)` i konsollen.

Krever FreeCAD 1.0 eller nyere og numpy, som følger med FreeCAD.

## Bruk

Åpne et dokument og kjør makroen. Dra for å rotere, hold Shift eller bruk
midtknappen for å panorere, rull for å zoome.

| Mål på et ratt med rifling | Verdi |
| --- | --- |
| Trekanter | 239 846 |
| Tesselering og opplasting | 2,5 s |
| Bilde | 12,4 ms, altså 80 per sekund |

## Materialer fra MatCap

Velger du et materiale i [MatCap](https://github.com/chrleon/freecad-matcap),
skriver den navnet til egenskapen `MatCapMaterial` på objektet, og den
lagres i FCStd-fila. PbrView leser navnet og slår opp PBR-parametrene.

**Oppslaget er eksakt, ikke et anslag.** Matcap-teksturene ble generert
fra nettopp disse tallene, så teksturen og denne rendringen er to
avbildninger av samme materiale. `metal_steel` er grunnfarge 0,70 / 0,72 /
0,75 med metallverdi 1,0 og ruhet 0,30, begge steder.

To unntak. Blenders to teksturer har ingen oppskrift å hente tall fra, så
de er lest av teksturene og er anslag. De trinnvise materialene har ingen
fysisk ekvivalent, og gjengis som matt maling i grunnfargen.

Uten MatCap brukes objektets egen `ShapeColor` med nøytrale verdier. De to
verktøyene kjenner ikke hverandre; de er bare enige om et navn.

## Hvordan det virker

**Normalene midles per flate, ikke over hele modellen.** Det er nettopp
riktig for CAD: en sylinder er én flate og blir jevn, mens møtet mellom to
flater forblir en skarp kant. Midler man over alt, ser faste kanter
avrundede ut og modellen slutter å se ut som CAD.

**Skyggekartet lagrer avstand til lyset, ikke dybde i klippkoordinater.**
Da blir sammenligningen rett fram, og skjevheten kan oppgis i millimeter
i stedet for i en ikke-lineær dybdeskala.

**Skyggepasset tegner baksidene.** Da havner selvskyggeaknen på flater vi
ikke ser, og skjevheten kan være liten nok til at skyggen holder kontakten
med objektet i stedet for å løsne fra det.

**Himmelen tegnes med samme miljøfunksjon som speilingene.** Det hørtes ut
som pynt, men uten det toner gulvet mot én farge mens bakgrunnen er en
annen, og overgangen er synlig uansett hvor mykt man toner.

**Klippeplanene regnes ut på nytt, ikke hentet fra FreeCAD.** FreeCAD
setter dem tett rundt modellen, og gulvplanet stikker langt utenfor. Da
skjærer det bakre klippeplanet tvers gjennom gulvet, og resultatet ser ut
som en horisont man ikke har bedt om. Den feilen kostet to gale
diagnoser før den ble funnet.

**Kameraet polles på en timer, ikke via en Coin-sensor.** En callback fra
Coins tegneløkke ville kalt Python midt i rendringen, og den veien har
allerede tatt ned FreeCAD én gang under utviklingen.

## Fallgruver i PySide som ikke handler om grafikk

Fire av feilene under utviklingen lå i limet mellom Python og Qt, ingen i
selve rendringen. De er verdt å kjenne hvis du skal skrive noe liknende.

| Symptom | Årsak |
| --- | --- |
| Helt svart vindu, ingen feilmelding | `setUniformValue` med en Python-float binder seg til heltallsvarianten. Eksponeringen blir null. Bruk `setUniformValue1f` |
| `setUniformValue2f` finnes ikke | Bare `1f` og `1i` er typet i PySide6. En vec2 må gå via `QVector2D` |
| FreeCAD krasjer med segfault i shiboken | En Python-feil slapp ut av en Qt-virtual. Pakk inn `initializeGL` og `paintGL`, ellers dør programmet uten at meldingen skrives ut |
| `glDrawElements` avviser både `None` og `0` | Det finnes ingen god oversettelse av en nullpeker. Pakk ut indeksene og bruk `glDrawArrays` |

I tillegg: på en `QOpenGLWidget` må du binde tilbake widgetens eget
rammebuffer etter et FBO-pass. `release()` binder null, og null er ikke
der widgeten tegner.

## Det som mangler

* Ingen plukking, seleksjon eller måling. Det er med vilje.
* Ingen HDRI. Miljøet er analytisk, en himmel med en sol i.
* Ingen antialiasing utover MSAA. Kantene kan flimre når du roterer.
* Geometrien lastes på nytt bare når du kjører makroen igjen. Endrer du
  modellen, må du starte visningen på nytt.
* Ett materiale per objekt, hentet fra objektets `ShapeColor`. Metall og
  ruhet er faste verdier i koden.
