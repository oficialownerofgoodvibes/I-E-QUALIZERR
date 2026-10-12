# EqForge 2.1

Ecualizador de sistema para Android con **TWS Sound** de fábrica, motor de 30 bandas,
compresor multibanda, limitador estilo Spotify y loudness inteligente.

## Cómo compilarlo

**Opción A – GitHub (sin Android Studio):** sube la carpeta a un repo. Con cada push, la
pestaña *Actions → Build APK* compila y te deja el `app-debug.apk` como artefacto.

**Opción B – Android Studio:** *Open* en esta carpeta, esperar el sync y *Run*.
(El zip no trae `gradle-wrapper.jar`; Android Studio lo regenera solo.)

Requisitos: Android 9+ (API 28), JDK 17.

## Cómo usarlo con Spotify

1. Abre EqForge una vez (concede los permisos). Desde ahí arranca solo con el teléfono.
2. En Spotify: **Ajustes → Reproducción → Ecualizador** abre EqForge directamente (Spotify en
   Android no trae EQ propio, usa el del sistema). También basta con dar play: Spotify
   anuncia su sesión de audio y EqForge se engancha.
3. El indicador de la pantalla *Sonido* dice «Conectado a Spotify» cuando está procesando.

## Modo Juego (nuevo en 2.1)

Pensado para Free Fire: oír los pasos del enemigo y aun así tener algo de música.

1. **Escaneo de apps.** Pestaña *Juego → Elegir de las apps instaladas* lista todas las apps con icono
   en el lanzador (sin `QUERY_ALL_PACKAGES`). La primera vez se vigilan solas Free Fire / Free Fire MAX y otros
   juegos conocidos que ya tengas; si instalas uno conocido después, se agrega solo.
2. **Detección.** Con el permiso *Acceso de uso*, el servicio lee qué app está al frente cada segundo
   (`UsageStatsManager`). Al abrir un juego vigilado entra el Modo Juego; al salir (3 s fuera) todo vuelve a la normalidad.
3. **Notificación.** «Free Fire detectado · Spotify al 40 %». Al tocarla sale una **barra flotante** encima del
   juego (permiso *Mostrar sobre otras apps*) que controla solo el volumen de Spotify y se oculta sola a los 6 s.
   Sin ese permiso, la notificación trae botones **− Música / + Música**.
4. **Volumen solo de Spotify.** Android no tiene volumen por app sin root, así que EqForge baja la ganancia de la
   *sesión de audio de Spotify* (la misma a la que ya engancha el EQ) con `DynamicsProcessing`. El juego no se toca.
5. **Hueco para pasos.** Mientras juegas, la música recibe además dos recortes suaves en 1.8 y 3.6 kHz
   (ajustables o apagables) para dejar libre la zona donde suelen caer los pasos.

Notas: Spotify debe estar reproduciendo (o haber reproducido) para que anuncie su sesión; si el teléfono no soporta
`DynamicsProcessing`, el volumen propio queda limitado al rango del Equalizer de hardware (~-15 dB).

## Qué trae

| | |
|---|---|
| **TWS Sound** | Preset de fábrica. Target tipo Harman + 2 dB de graves, lodo limpio en 300 Hz, presencia en 3.1 kHz, muesca de 6.2 kHz anti-aspereza y aire en 10.5 kHz. |
| **16 presets** | Reggaetón, Corridos y Regional, Trap 808, Pop, Rock, EDM, Cine, Noche, Calle, Voz, Bocina del cel, Acústico, Harman, Plano, Menos graves, Anti-fatiga. Cada uno son filtros paramétricos con frecuencia, ganancia y Q exactas. |
| **Editor paramétrico** | Nodos arrastrables sobre el espectro real. Mantén pulsado en vacío = nodo nuevo. Pasos finos: 1/24 de octava, 0.1 dB, Q 0.05. |
| **5 faders de tono** | Sub / Graves / Cuerpo / Voz / Brillo, ±6 dB encima de cualquier preset. |
| **Headroom automático** | Preamp calculado por el pico de la curva; nunca satura por boostear. |
| **Level match (ISO 226)** | Iguala el volumen *percibido* del EQ al del audio sin procesar, para que el A/B sea honesto. «Mantén: sin EQ» en el hero. |
| **Loudness inteligente** | Compensación Fletcher-Munson según el volumen del sistema (graves y aire que el oído pierde al bajar). |
| **Volumen estilo Spotify** | +3 dB (≈ «Alto», −11 LUFS) y +5 dB con limitador a −1 dB, 5 ms / 100 ms, igual que el de Spotify. |
| **Preset por salida** | Cada audífono recuerda su sonido; la bocina del cel recibe «Bocina del cel» sola. |
| **Importador AutoEq** | `ParametricEQ.txt` con PK / LSC / HSC, cada filtro entra tal cual. |
| **Tile de ajustes rápidos** | Prende/apaga todo. |

## Qué se corrigió respecto a v1

- En `DynamicsProcessing` la frecuencia de banda es el **tope superior**, no el centro. v1 las usaba
  como centros y el compresor tenía crossovers `[1, 200, 2000]` que dejaban los agudos sin cubrir.
- Colgar efectos de la sesión global 0 está deprecado por Android; v1 solo usaba eso. Ahora
  se enganchan las sesiones que anuncian Spotify / YT Music (con la global como relleno, sin doble proceso).
- El servicio era `mediaPlayback`, que Android 15 prohíbe arrancar desde `BOOT_COMPLETED`: ahora es `specialUse`.
- Ícono de launcher invisible (vector blanco sobre transparente), espectro lineal (las barras
  graves no existían), importador AutoEq que solo entendía `PK`.

## Estructura

```
app/src/main/java/com/eqforge/
  audio/    Model, Presets, Renderer (headroom + level match), Psycho (ISO 226),
            Curve (biquads RBJ), FxChain (DynamicsProcessing), Engine (sesiones),
            Forge (estado), Store, AutoEq, DeviceWatcher, Spectrum
  game/     Modo Juego: GameWatcher (UsageStats), GameOverlay (barra), GameNotifier, Apps (escaneo), Perms
  service/  DspService (FGS + broadcasts de sesión + volumen), BootReceiver, EqTileService
  ui/       Compose: Theme, Widgets, CurveEditor, Home / Presets / Lab, MainActivity
tools/      Referencia en Python de toda la matemática (curvas, ISO 226, headroom)
docs/       presets.png: las 16 curvas
```

## Verificación

La matemática de audio (respuesta de biquads, tablas ISO 226, headroom, level match, regex de
AutoEq) está validada con los scripts de `tools/`. Error de cuantizar la curva a 1/3 de octava:
0.64 dB máx., 0.15 dB RMS.
