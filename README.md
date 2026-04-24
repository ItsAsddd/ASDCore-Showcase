# ⚔ ASDCore — Sistema de Rankeds

## Documentación Completa

---

## 📋 Índice

1. [Resumen General](#resumen-general)
2. [Estructura de Archivos](#estructura-de-archivos)
3. [Rangos y Puntos](#rangos-y-puntos)
4. [Temporadas](#temporadas)
5. [Minijuegos](#minijuegos)
6. [Comandos](#comandos)
7. [GUIs](#guis)
8. [Herramientas de Configuración](#herramientas-de-configuración)
9. [Sistema de Matchmaking PvP](#sistema-de-matchmaking-pvp)
10. [Listeners y Protecciones](#listeners-y-protecciones)
11. [Placeholders](#placeholders)
12. [Mensajes Configurables](#mensajes-configurables)
13. [Archivo de Configuración](#archivo-de-configuración)
14. [Integración con BBattle](#integración-con-bbattle)
15. [Estructura de Datos de Jugador](#estructura-de-datos-de-jugador)
16. [Flujo Completo de una Ranked](#flujo-completo-de-una-ranked)

---

## Resumen General

El sistema de Rankeds añade un modo competitivo al servidor de Prisión basado en 4 ramas:

- **⛏ Minería** — Minar líneas o minas enteras contra reloj
- **🎣 Pesca** — Pesca a ciegas y pesca de ovejas
- **⚔ PvP** — Combates 1vs1 con sets de hierro u OP
- **💀 PvE** — Arenas de mobs (integración con BBattle)

Cada rama tiene modos **solitarios** y **PvP**. Los jugadores ganan puntos competitivos que determinan su rango, desde Bronce hasta Legendario.

---

## Estructura de Archivos

```
src/main/java/me/asdcore/rankeds/
├── data/                          # Modelos de datos
│   ├── RankedRank.java            # Enum de los 7 rangos con colores HEX y símbolos
│   ├── GameType.java              # Enum de los 12 minijuegos
│   ├── PlayerRankedData.java      # Datos completos por jugador
│   ├── RankedSession.java         # Sesión activa (guarda/restaura inventario)
│   └── TopEntry.java              # Entrada de leaderboard
│
├── managers/
│   └── RankedsManager.java        # Manager central (821 líneas) — todo el sistema
│
├── games/                         # Sistema de minijuegos
│   ├── GameHandler.java           # Interface que implementan todos los handlers
│   ├── BaseGameHandler.java       # Clase base con lógica compartida
│   ├── GameRegistry.java          # Registro de los 12 handlers
│   ├── mining/
│   │   ├── MinaLineaSoloHandler.java
│   │   ├── MinaEnteraSoloHandler.java
│   │   ├── MinaLineaPvpHandler.java
│   │   └── MinaEnteraPvpHandler.java
│   ├── fishing/
│   │   ├── PescaCiegaSoloHandler.java
│   │   ├── PescaOvejasSoloHandler.java
│   │   ├── PescaCiegaPvpHandler.java
│   │   └── PescaOvejasPvpHandler.java
│   ├── pvp/
│   │   ├── PvpHierroHandler.java
│   │   └── PvpOpHandler.java
│   └── pve/
│       ├── PveSoloHandler.java
│       └── PvePvpHandler.java
│
├── gui/                           # Interfaces gráficas
│   ├── RankedsPlayerGUI.java      # GUI del jugador (/rankeds)
│   ├── RankedsConfigGUI.java      # GUI config admin (herramientas)
│   ├── RankedsGeneralGUI.java     # GUI admin general (sesiones, bans, stats)
│   └── RankedsSettingsGUI.java    # GUI editor numérico (bloques, tiempos)
│
├── listeners/
│   ├── RankedsListener.java       # Listener principal (eventos de juego)
│   └── RankedsToolListener.java   # Listener de herramientas de config
│
├── commands/
│   ├── RankedsCommand.java        # /asdcore rankeds (admin)
│   └── RankedsPlayerCommand.java  # /rankeds (jugador)
│
└── placeholders/
    └── RankedsPlaceholders.java   # Integración PlaceholderAPI
```

### Archivos modificados del core:
- `ASDCore.java` — Añadidos campos, inicialización, listeners, shutdown
- `ASDCoreCommand.java` — Añadido subcomando `rankeds` con tab-complete
- `plugin.yml` — Añadido comando `/rankeds` y permiso `asdcore.rankeds`
- `build.gradle.kts` — Añadida dependencia de BBattle

---

## Rangos y Puntos

| Rango | Símbolo | Color HEX | Puntos Requeridos |
|---|---|---|---|
| Bronce | ⬥ | `#CD7F32` (café) | 0 |
| Plata | ⬥ | `#C0C0C0` (plata) | 100 |
| Oro | ⬥ | `#FFD700` (dorado) | 300 |
| Cristal | ✦ | `#9B30FF` (morado) | 800 |
| Maestros | ✦ | `#2C2C2C` (negro) | 1600 |
| Campeones | ♛ | `#DC143C` (rojo) | 2000 |
| Legendario | ⚝ | `#DA70D6` (lila) | 3000 |

- Los umbrales de puntos son **configurables** en `rankeds-config.yml` bajo la sección `rangos`
- El símbolo cambia según el tier del rango (⬥ para los primeros, ✦ para medio, ♛ y ⚝ para los altos)
- Los prefixes se muestran con color HEX completo en chat

### Sistema de puntos

**Modo solitario:**
- Se compara el resultado del jugador (tiempo o puntaje) contra el Top 1
- Si el jugador ES el nuevo Top 1: recibe **30 puntos** (máximo)
- Si no: se calcula proporcionalmente. Ejemplo: Top 1 = 3 min, jugador = 6 min → recibe ~15 puntos
- Mínimo siempre 1 punto

**Modo PvP:**
- Ganador: **+30 puntos**
- Perdedor: **-28 puntos**
- Desconexión durante PvP: **-15 puntos** al que se fue, el otro gana

**Boost de temporada:** Cuando está activo, multiplica los puntos ganados por un factor configurable (default: 1.05x)

---

## Temporadas

- **Duración:** 28 días (configurable)
- **Descanso:** 1 día entre temporadas (configurable)
- El sistema verifica automáticamente cada 60 segundos si la temporada debe terminar o iniciar

### Al terminar una temporada:

1. Se envía un mensaje global a todos
2. Se anuncia el jugador con más puntos
3. Se entregan **rewards por rango** a los jugadores online:
   - Dinero (mediante Vault)
   - Comandos de consola (configurables por rango)
4. Todos los jugadores **bajan 3 rangos** (configurable)
5. Los puntos se establecen al mínimo del rango resultante
6. Jugadores que no estén online reciben sus rewards al reconectarse

### Rewards por defecto (configurables):

| Rango | Dinero | Comandos |
|---|---|---|
| Bronce | $100,000 | Configurables |
| Plata | $300,000 | Configurables |
| Oro | $1,000,000 | Configurables |
| Cristal | $2,500,000 | Configurables |
| Maestros | $5,000,000 | Configurables |
| Campeones | $10,000,000 | Configurables |
| Legendario | $25,000,000 | Configurables |

---

## Minijuegos

### ⛏ Minería

#### Mina Línea (Solitario)
- Región de 3 de ancho × 8 de alto × 50 de largo
- La línea central es la que se debe minar
- Las líneas laterales tienen bloques de hierro (bonus obligatorio)
- **Condiciones para parar el cronómetro:**
  - Haber minado TODOS los bloques de hierro
  - Haber minado toda la línea central
- Bloques: Stone (relleno), Redstone Ore (obstáculo), Iron Block (obligatorio)
- Pico de diamante con Eficiencia 5, Irrompibilidad 3
- Efecto de Prisa Minera 2 infinito
- Los items minados NO van al inventario
- Scoreboard con cronómetro + Top 1, 2, 3
- BossBar con bloques restantes
- ActionBar con tiempo

#### Mina Entera (Solitario)
- Mina cúbica completa (default 16×16×8)
- TODOS los bloques cuentan
- Contiene: Stone, Redstone Ore, Gold Ore, Iron Block, Iron Ore
- Mismo sistema de puntos, avisos y equipo

#### Mina Línea PvP
- 2 jugadores, cada uno con su propia línea
- Regiones separadas: region-mina-1/2, region-linea-1/2, spawn-1/2
- El que la mine más rápido gana (+30/-28)
- Scoreboard muestra bloques minados de cada jugador

#### Mina Entera PvP
- 2 jugadores, cada uno con su mina entera
- Mismo sistema que la solitaria pero competitivo

### 🎣 Pesca

#### Pesca a Ciegas (Solitario)
- Caña con Lure 100, durabilidad infinita
- Efectos: DARKNESS + BLINDNESS (no ves nada)
- Solo puedes pescar por sonido y movimiento de boya
- Cada pesca exitosa = +1 a la racha
- Si fallas una pesca = juego terminado
- Solo se puede pescar bacalao (cod)
- Puntuación = número de racha
- Movimiento bloqueado (solo puedes rotar la cabeza)

#### Pesca de Ovejas (Solitario)
- Usa la caña para atraer ovejas a tu zona de puntos
- Oveja blanca = +1 punto
- Oveja amarilla = +3 puntos
- Duración: 3 minutos
- Las ovejas que caen en la zona de puntos desaparecen con partículas
- Movimiento limitado a la región del jugador

#### Pesca a Ciegas PvP
- 2 jugadores pescan a ciegas simultáneamente
- El primero que falle pierde
- Cada uno tiene su spawn y zona

#### Pesca de Ovejas PvP
- 2 jugadores, ovejas compartidas, zonas de puntos separadas
- Compiten por atraer las mismas ovejas a su zona
- Al terminar los 3 minutos, el que tenga más puntos gana

### ⚔ PvP

#### 1vs1 Set Hierro
- Kit default: armadura de hierro, hacha, 4 manzanas doradas, caña, 32 bloques, cubo de lava, arco + 10 flechas, 2 pociones de curación
- Sin escudo
- Los jugadores NO mueren realmente (cuando la vida llega a ≤1, se declara ganador/perdedor)
- Kit capturable: un admin puede ponerse el equipo que quiera y hacer click en "Capturar Kit" para guardar ese inventario como el kit del minijuego

#### 1vs1 Set OP
- Kit default: diamante protección 4, espada con filo 5 y aspecto ígneo 2, 2 manzanas encantadas, arco, 64 bloques, pociones de curación/velocidad/fuerza
- Mismo sistema de no-muerte y kit capturable

### 💀 PvE

#### Mob Arena Solo
- Rondas infinitas, cada vez más difíciles
- Mobs escalan: +15% vida y +10% daño por ronda
- Rondas 1-5: zombies, esqueletos, arañas
- Rondas 6-10: + brujas, pillagers
- Rondas 11+: + vindicadores, ravagers
- Cuando el jugador "muere" (vida ≤1), termina
- Puntuación = ronda alcanzada
- **Integración BBattle:** Si hay una arena de BBattle configurada, usa sus rondas y mobs en lugar del sistema interno
- Kit capturable

#### Mob Arena PvP
- 10 rondas fijas: 9 fáciles + 1 boss (araña de cueva con 100HP)
- 2 jugadores en arenas separadas
- El primero que complete las 10 rondas gana
- Si uno muere y el otro sobrevive, el sobreviviente gana
- Si ambos mueren, gana el que llegó a más rondas

---

## Comandos

### `/rankeds` (Permiso: `asdcore.rankeds`, default: true)
Comando principal para jugadores.

| Uso | Descripción |
|---|---|
| `/rankeds` | Abre el GUI principal (perfil, jugar, top, espectear) |
| `/rankeds` (estando en cola) | Cancela la búsqueda de rival PvP |
| `/rankeds spectate` | Abre la lista de rankeds activas para espectear |

### `/asdcore rankeds` (Permiso: `asdcore.admin`)
Comando de administración.

| Uso | Descripción |
|---|---|
| `/asdcore rankeds config` | Abre el GUI de configuración (herramientas para regiones, spawns, kits) |
| `/asdcore rankeds general` | Abre el GUI de administración general (sesiones activas, bans, temporada, stats) |
| `/asdcore rankeds puntos add <jugador> <cantidad>` | Añade puntos a un jugador |
| `/asdcore rankeds puntos take <jugador> <cantidad>` | Quita puntos a un jugador |
| `/asdcore rankeds puntos set <jugador> <cantidad>` | Establece los puntos de un jugador |
| `/asdcore rankeds puntos info <jugador>` | Muestra toda la info ranked del jugador |
| `/asdcore rankeds puntos reset <jugador>` | Resetea puntos a 0 |

---

## GUIs

### GUI del Jugador (`/rankeds`)
- **Cabeza del jugador (slot 20):** Muestra perfil con rango, puntos, partidas, victorias, derrotas, winrate, racha actual y mejor racha
- **Estrella del Nether (slot 22):** "Iniciar Ranked" → abre sub-GUI de selección de modo
- **Ojo de Ender (slot 24):** "Espectear" → lista de rankeds activas
- **Casco dorado (slot 31):** Top Global → muestra top 10 en chat
- **Reloj (slot 40):** Info de la temporada actual

### Sub-GUI Selección de Modo
- **Espada de hierro (slot 11):** Modo Solitario (muestra cooldown si aplica)
- **Espada de diamante (slot 15):** Modo PvP (muestra cooldown si aplica)

### GUI Config Admin (`/asdcore rankeds config`)
Muestra los 12 minijuegos organizados por categoría con estado de configuración (✓/✗).
Al hacer click en uno se abre el panel de herramientas específico:

- **Hachas doradas:** Para seleccionar regiones (click izq = pos1, click der = pos2)
- **Azadas doradas:** Para establecer spawns (click der = establecer)
- **Palo:** Capturar Kit (guarda tu inventario actual como kit del minijuego)
- **Ojo de Ender:** Spawn de espectadores
- **Comparador:** Abre el editor de ajustes numéricos

### GUI General Admin (`/asdcore rankeds general`)
- **Rankeds Activas:** Ver y forzar terminación de rankeds en curso
- **Cola de Matchmaking:** Ver jugadores esperando rival PvP
- **Temporada:** Click izq = iniciar nueva, click der = terminar actual
- **Estadísticas:** Jugadores registrados, partidas totales, victorias
- **Top Global:** Top 10 en chat
- **Ban/Desban:** Lista de jugadores online para banear/desbanear de rankeds
- **Terminar TODAS:** Fuerza el fin de todas las rankeds activas
- **Recargar Config:** Recarga rankeds-config.yml

### GUI Settings (Editor Numérico)
Editor interactivo para ajustes por minijuego:
- **Click izq:** +1
- **Shift+Click izq:** +10
- **Click der:** -1
- **Shift+Click der:** -10
- **Click medio:** Escribir valor por chat

Configuraciones disponibles por juego:
- Minas: tiempo máximo, cantidad de Iron Block, Redstone Ore, Iron Ore, Gold Ore
- Pesca ovejas: duración, cantidad de ovejas, radio de spawn, ovejas amarillas
- Globales: cooldowns, máx puntos, penalización desconexión, diferencia de rangos

---

## Herramientas de Configuración

Se obtienen desde el GUI de config (`/asdcore rankeds config` → click en un minijuego).

### Hacha Dorada (Región)
- **Click izquierdo en bloque:** Establece pos1
- **Click derecho en bloque:** Establece pos2 y guarda la región
- Muestra dimensiones y total de bloques al guardar
- Cada hacha tiene codificado el juego y la clave de región (via PersistentDataContainer)

### Azada Dorada (Spawn)
- **Click derecho:** Establece el spawn en tu posición actual (incluyendo yaw/pitch)
- Tipos: spawn del jugador, spawn de ovejas, spawn de espectadores

### Palo (Captura de Kit)
- Al hacer click, toma TODO tu inventario actual (items + armadura) y lo guarda como kit para ese minijuego
- Aplica para PvP (Hierro, OP) y PvE (Mob Arena)

### Regiones por minijuego:

| Minijuego | Regiones necesarias | Spawns necesarios |
|---|---|---|
| Mina Línea Solo | region-mina, region-linea | spawn |
| Mina Entera Solo | region-mina | spawn |
| Mina Línea PvP | region-mina-1/2, region-linea-1/2 | spawn-1/2 |
| Mina Entera PvP | region-mina-1/2 | spawn-1/2 |
| Pesca Ciega Solo | region-juego | spawn |
| Pesca Ovejas Solo | region-juego, zona-puntos, region-jugador | spawn, spawner-ovejas |
| Pesca Ciega PvP | region-juego | spawn-1/2 |
| Pesca Ovejas PvP | region-juego, zona-puntos-1/2 | spawn-1/2, spawner-ovejas |
| PvP Hierro | region-juego | spawn-1/2 |
| PvP OP | region-juego | spawn-1/2 |
| PvE Solo | region-juego | spawn |
| PvE PvP | region-1/2 | spawn-1/2 |

Todos los minijuegos también aceptan `spawn-espectadores` opcional.

---

## Sistema de Matchmaking PvP

1. El jugador abre `/rankeds` → "Iniciar Ranked" → "PvP"
2. Entra en la cola de matchmaking
3. El sistema busca un rival cada 2 segundos
4. **Restricción de rangos:** Los jugadores no pueden tener más de 3 rangos de diferencia (configurable)
5. Si encuentra rival compatible:
   - Ambos ven la animación de "ruleta" con los nombres de minijuegos PvP
   - La ruleta dura 10 segundos, empieza rápido y se desacelera
   - Al final selecciona un minijuego PvP al azar
   - BossBar de 10 segundos de preparación
   - Comienza la ranked
6. Si no encuentra rival en 5 minutos, se cancela automáticamente
7. El jugador puede cancelar con `/rankeds` mientras está en cola

---

## Listeners y Protecciones

### RankedsListener.java
Maneja todos los eventos durante una ranked:

- **BlockBreak:** Redirige al handler del minijuego correspondiente, cancela drops y XP
- **PlayerFish:** Detecta pesca exitosa y fallida para los minijuegos de pesca
- **PlayerDeath:** Previene la muerte real, mantiene inventario y nivel
- **PlayerQuit:** Restaura inventario, aplica penalización de -15 puntos, notifica al rival
- **PlayerJoin:** Muestra warning de desconexiones, mensaje de temporada, entrega rewards pendientes
- **CommandPreprocess:** Bloquea TODOS los comandos durante una ranked (solo permite `/rankeds`)
- **PlayerDropItem:** Bloquea soltar items durante rankeds
- **EntityDamageByEntity:** Solo permite PvP entre rivales del mismo match
- **PlayerMove:** Bloquea movimiento en pesca a ciegas (solo rotación de cabeza)
- **EntityDeath:** Previene drops de mobs de PvE
- **EntityPickupItem:** Previene recoger items (excepto en PvP donde se necesitan flechas/bloques)

### Protecciones de sesión:
- **Re-entrancy guard:** `completeSoloGame` y `completePvpGame` verifican `session.isActive()` antes de ejecutar, evitando doble ejecución
- **Deep inventory copy:** El inventario se clona ítem por ítem para evitar corrupción de referencias
- **Potion cleanup:** Al restaurar estado, se limpian todos los efectos de poción primero
- **Disconnect guard:** Si un jugador se desconecta, se marca en la sesión para no restaurar su inventario dos veces
- **Safety checks pre-inicio:** Antes de crear la sesión (después del countdown de 10s), se verifica que ambos jugadores sigan online y no estén en otra ranked

---

## Placeholders

Requiere **PlaceholderAPI**. Prefijo: `asdcore_rankeds`

### Por jugador:
| Placeholder | Descripción |
|---|---|
| `%asdcore_rankeds_puntos%` | Puntos actuales |
| `%asdcore_rankeds_rango%` | Nombre del rango con color |
| `%asdcore_rankeds_prefix%` | Símbolo del rango con color |
| `%asdcore_rankeds_rango_nombre%` | Nombre del rango sin color |
| `%asdcore_rankeds_victorias%` | Total de victorias |
| `%asdcore_rankeds_derrotas%` | Total de derrotas |
| `%asdcore_rankeds_partidas%` | Total de partidas jugadas |
| `%asdcore_rankeds_winrate%` | Porcentaje de victorias |
| `%asdcore_rankeds_racha%` | Racha de victorias actual |
| `%asdcore_rankeds_mejor_racha%` | Mejor racha histórica |
| `%asdcore_rankeds_baneado%` | "Sí" o "No" |
| `%asdcore_rankeds_desconexiones%` | Veces desconectado en ranked |

### Globales:
| Placeholder | Descripción |
|---|---|
| `%asdcore_rankeds_temporada%` | Número de temporada actual |
| `%asdcore_rankeds_top_1_nombre%` | Nombre del Top 1 global |
| `%asdcore_rankeds_top_1_puntos%` | Puntos del Top 1 |
| `%asdcore_rankeds_top_1_rango%` | Rango del Top 1 (con color) |
| `%asdcore_rankeds_top_1_prefix%` | Prefix del Top 1 |
| `%asdcore_rankeds_top_2_nombre%` | Nombre del Top 2 |
| `%asdcore_rankeds_top_2_puntos%` | Puntos del Top 2 |
| `%asdcore_rankeds_top_3_nombre%` | Nombre del Top 3 |
| `%asdcore_rankeds_top_3_puntos%` | Puntos del Top 3 |

---

## Mensajes Configurables

Todos los mensajes globales se editan en `rankeds-config.yml` bajo `mensajes`. Soportan HEX colors con `&#RRGGBB`.

| Clave | Cuándo se envía | Variables |
|---|---|---|
| `victoria-pvp` | Cuando un jugador gana PvP | `{player1}`, `{player2}`, `{juego}` |
| `top-nuevo` | Cuando alguien entra al top 3 | `{player}`, `{posicion}`, `{juego}` |
| `temporada-fin` | Cuando termina la temporada | — |
| `temporada-inicio` | Cuando empieza nueva temporada | — |
| `max-puntos` | Mayor puntaje de la temporada | `{player}` |
| `subir-rango` | Cuando un jugador sube de rango | `{player}`, `{rango}` |
| `bajar-rango` | Cuando un jugador baja de rango | `{player}`, `{rango}` |
| `racha` | Cada 3 victorias consecutivas | `{player}`, `{racha}` |
| `primera-ranked` | Primera ranked completada | `{player}` |
| `desconexion` | Desconexión en PvP | `{player}`, `{player2}`, `{puntos}` |
| `desconexion-solo` | Desconexión en solo | `{player}`, `{puntos}` |
| `humillacion` | Victoria aplastante | `{player1}`, `{player2}`, `{juego}` |
| `racha-pvp` | Racha PvP diaria | `{player}`, `{victorias}` |
| `top1-global` | Nuevo top 1 global | `{player}` |

---

## Archivo de Configuración

Se genera automáticamente en `plugins/ASDCore/rankeds/rankeds-config.yml`:

```yaml
# ═══ Rangos (puntos mínimos) ═══
rangos:
  BRONCE: 0
  PLATA: 100
  ORO: 300
  CRISTAL: 800
  MAESTROS: 1600
  CAMPEONES: 2000
  LEGENDARIO: 3000

# ═══ Cooldowns ═══
cooldown-solo-minutos: 60        # 1 hora entre rankeds solo
cooldown-pvp-minutos: 30         # 30 min entre rankeds PvP

# ═══ Puntos ═══
max-diferencia-rangos: 3         # Máx diferencia de rangos para matchmaking
max-puntos-solo: 30              # Máx puntos por ranked solo
max-puntos-pvp: 30               # Máx puntos por ganar PvP
puntos-perder-pvp: 28            # Puntos que pierde el que pierde PvP
penalizacion-desconexion: 15     # Puntos restados por desconectarse

# ═══ Temporada ═══
temporada:
  duracion-dias: 28
  descanso-dias: 1
  bajar-rangos: 3                # Cuántos rangos bajan al terminar temporada
  boost-puntos: 1.05             # Multiplicador de puntos (5% extra)
  rewards:
    BRONCE:
      dinero: 100000
      comandos:
        - "say {player} terminó en Bronce"
    # ... (cada rango tiene su sección)

# ═══ Nombres de juegos (editables) ═══
nombres-juegos:
  MINA_LINEA_SOLO: "Mina Línea"
  MINA_ENTERA_SOLO: "Mina Entera"
  # ... etc

# ═══ Mensajes globales ═══
mensajes:
  victoria-pvp: "&#FFD700 ⚔ &f{player1} &#FFD700ha vencido a ..."
  # ... etc

# ═══ Configuración por juego (se genera al usar herramientas) ═══
juegos:
  MINA_LINEA_SOLO:
    mundo: "world"
    spawn: "world,100.5,65.0,200.5,90.0,0.0"
    region-mina:
      pos1: "world,100,60,200"
      pos2: "world,103,67,250"
    region-linea:
      pos1: "world,101,60,200"
      pos2: "world,101,67,250"
    tiempo-maximo: 600
    bloques:
      IRON_BLOCK: 25
      REDSTONE_ORE: 40
  # ... etc
```

### Otros archivos de datos:
- `rankeds/season.yml` — Estado de la temporada (número, inicio, activa)
- `rankeds/tops.yml` — Leaderboards de cada minijuego (top 50)
- `rankeds/players/{uuid}.yml` — Datos individuales por jugador

---

## Integración con BBattle

El modo PvE puede usar arenas del plugin **BBattle (BossBattle)** para spawneo de mobs.

### Cómo configurar:
1. Coloca el JAR compilado de BBattle en `ASDCore/libs/BossBattle.jar`
2. En el GUI de settings del PvE Solo, edita `bbattle-arena` con el nombre de una arena existente
3. Si la arena existe y tiene rondas configuradas, usará los mobs de BBattle
4. Si no, usa el sistema interno de spawneo con escalado por ronda

### Sin BBattle:
El sistema funciona perfectamente sin BBattle. Genera mobs con escalado automático de stats (vida, daño) por ronda.

---

## Estructura de Datos de Jugador

Archivo: `rankeds/players/{uuid}.yml`

```yaml
name: "NombreJugador"
points: 450
games-played: 23
wins: 15
losses: 8
current-streak: 3
best-streak: 7
last-solo: 1714000000000     # timestamp última ranked solo
last-pvp: 1714000000000      # timestamp última ranked PvP
disconnects: 0
banned: false
season-rewards-claimed: 1    # número de temporada en la que ya cobró

# Stats por minijuego
game-wins:
  MINA_LINEA_SOLO: 5
  PVP_HIERRO: 3
game-played:
  MINA_LINEA_SOLO: 8
  PVP_HIERRO: 6

# Mejores marcas personales
best-records:
  MINA_LINEA_SOLO: 185.4     # segundos
  PESCA_CIEGA_SOLO: 42.0     # racha
```

---

## Flujo Completo de una Ranked

### Solo:
```
1. /rankeds → GUI → "Iniciar Ranked" → "Solitario"
2. Verificar: no baneado, temporada activa, sin cooldown
3. Animación de ruleta (10s) — títulos cambiando entre minijuegos
4. Se selecciona minijuego random
5. BossBar countdown de 10 segundos
6. Se guarda inventario/posición/vida/XP/comida del jugador (deep copy)
7. Se limpia al jugador y se equipa con items del minijuego
8. Se teleporta al spawn del minijuego
9. Se rellenan las regiones (minas con bloques, ovejas, etc.)
10. Se muestran instrucciones, BossBar, Scoreboard
11. El jugador juega
12. Al terminar: se calculan puntos, se actualiza top, se envían mensajes
13. Se restaura inventario/posición/vida/XP/comida
14. Se teleporta de vuelta a donde estaba
15. Se limpia scoreboard, bossbars, efectos de poción
```

### PvP:
```
1. /rankeds → GUI → "Iniciar Ranked" → "PvP"
2. Entra en cola de matchmaking
3. Sistema busca rival compatible (máx 3 rangos de diferencia)
4. Cuando encuentra: ambos ven la ruleta PvP
5. Se selecciona minijuego PvP random
6. BossBar countdown de 10 segundos para ambos
7. Se guardan estados de ambos jugadores
8. Se equipan y teleportan a sus spawns
9. Juegan
10. Al terminar: ganador +30, perdedor -28
11. Mensaje global de victoria
12. Se restauran ambos jugadores
```

### Desconexión durante ranked:
```
1. Se restaura inventario del jugador que se fue
2. -15 puntos al desertor
3. Se incrementa contador de desconexiones
4. Mensaje global de desconexión
5. Si era PvP: el rival gana automáticamente (+30)
6. Al reconectarse: warning de sanción
```

---

## Permisos

| Permiso | Descripción | Default |
|---|---|---|
| `asdcore.rankeds` | Acceso a `/rankeds` | `true` |
| `asdcore.admin` | Acceso a `/asdcore rankeds` (config, general, puntos) | `op` |

---

## Dependencias

- **Vault** — Para dar dinero de rewards de temporada
- **PlaceholderAPI** — Para los placeholders (opcional)
- **BBattle/BossBattle** — Para integración PvE (opcional)
- **EssentialsX** — `/spawn` para espectadores que salen (opcional)
