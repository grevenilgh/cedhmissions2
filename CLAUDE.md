# CLAUDE.md

Este archivo da contexto a Claude Code (claude.ai/code) para trabajar en este repositorio.

## Qué es

**Puntos & Misiones** (paquete `cedhmissions`): app Flutter para llevar la puntuación por misiones en partidas multijugador de cEDH/Commander, o de cualquier juego por rondas.

Flujo principal:
1. **Jugadores favoritos**: lista de jugadores habituales que se guarda en el móvil.
2. **Planes de misión**: conjuntos de misiones con nombre, descripción opcional, puntos (pueden ser negativos) y tipo. Se pueden reordenar y también exportar o importar como `.json`.
3. **Partida**: se añaden jugadores (desde favoritos o nuevos), se crean rondas eligiendo un plan y en cada ronda se marca qué jugador ha completado cada misión. La clasificación se ordena por puntos.

El texto de la interfaz está en español y el tema es siempre oscuro (`ThemeMode.dark`, tonos blueGrey).

## Comandos

```bash
flutter pub get
flutter run
flutter analyze                 # flutter_lints 6
flutter test                    # ver "Problemas conocidos"
flutter build apk --release
dart run flutter_launcher_icons # icono desde assets/icon/icon.png
```

SDK de Dart: `^3.8.1`. Flutter no está en el PATH de esta máquina, así que hay que instalarlo o añadirlo antes de ejecutar estos comandos.

## Arquitectura

`provider` + `ChangeNotifier`, con persistencia en `shared_preferences` en formato JSON.

```
lib/
  main.dart               MultiProvider (Game, MissionPlan, FavoritePlayers) + tema oscuro
  models/                 Game, Round, Mission(+MissionType), MissionPlan, Player — todos con toJson/fromJson
  providers/              lógica de negocio; cada mutación => _save*() + notifyListeners()
  services/storage_service.dart   única capa de persistencia (claves: missionPlans, players, gameState)
  screens/                home → game / favorite_players / mission_plans → mission_detail; game → round_detail
  widgets/player_score_card.dart
  data/mission_plans.dart planes de ejemplo hardcodeados — NO se usan en ningún sitio
```

**Patrón de los providers:** el constructor llama a `_load*()` de forma asíncrona y cada método público modifica la lista o el objeto, guarda **sin `await`** y llama a `notifyListeners()`. Cualquier método nuevo debe seguir el mismo patrón. Los IDs son UUID v4 (`uuid`).

### Modelo de puntuación (`GameProvider`)

- `Round.completedMissionsByPlayer`: `missionId → [playerIds]` (quién completó cada misión).
- `Game.playerPointsPerRound`: `roundId → {playerId → puntos}`. Es un valor **derivado** que calcula `_recalculateRoundPoints` a partir de las misiones completadas.
- `Player.totalPoints`: también derivado. Lo calcula `_recalculateAllPlayerTotalPoints` sumando todas las rondas.
- `MissionType.unique`: solo puede completarla un jugador por ronda. Al marcarla se desmarca a los demás (`toggleMissionCompletion`).
- `MissionType.multiple`: varios jugadores pueden completarla y todos reciben los puntos.
- Cada ronda guarda una **copia** de las misiones del plan (`List.from(missionPlan.missions)`). Ojo: es una copia superficial, así que los objetos `Mission` son los mismos que los del plan. Si se edita una misión del plan, cambia también en las rondas que ya existen hasta que se reinicie la app. Hay que tenerlo en cuenta si se toca la edición de misiones.

Si cambias cómo se puntúa, recalcula siempre con esas dos funciones y no modifiques los totales a mano.

### Importar y exportar

- Exportar (`mission_detail_screen.dart`): `FilePicker.platform.saveFile` con los bytes JSON de `MissionPlan.toJson()`.
- Importar (`mission_plans_screen.dart`): `pickFiles(['json'])` → `File.readAsString` → `MissionPlan.fromJson`. `importMissionPlan` asigna un **id nuevo** al plan, pero los ids de las misiones se mantienen.
- Formato: `{id, name, missions:[{id,name,description,points,type:"unique"|"multiple"}]}`.

### Persistencia

Si cambias un modelo, cambia también su `toJson`/`fromJson` y ten en cuenta que los datos ya guardados no tienen el campo nuevo (usa `?? valorPorDefecto`). No hay versionado ni migraciones.

## Problemas conocidos

- `test/widget_test.dart` es la plantilla del contador de Flutter y **falla**. Además, `MyApp` necesita el `MultiProvider` y `SharedPreferences.setMockInitialValues({})` para poder probarse.
- `game_screen.dart` ordena `players.sort(...)` dentro de `build`, y esa lista es la misma que tiene el provider, así que se modifica el estado mientras se dibuja la pantalla. Es mejor ordenar una copia.
- `GameProvider.updatePlayerName` comprueba el nombre recortado (`trimmedNewName`) pero guarda `newName` sin recortar.
- `StorageService.loadGameState` nunca devuelve `null`: si no hay nada guardado crea un `Game` vacío.
- `permission_handler` y los permisos de almacenamiento del `AndroidManifest` no se usan en el código, porque `file_picker` usa SAF.
- `mission_plan_provider.dart` usa `print` en los `catch` (aviso `avoid_print`).
- El identificador de Android es `com.example.cedhmissions`. Hay que cambiarlo antes de publicar en tiendas.

## Convenciones

- Comentarios y textos de la interfaz en español, y cada archivo empieza con `// lib/ruta/archivo.dart`.
- Los diálogos son `AlertDialog` con botones "Cancelar" / acción. Antes de borrar algo se pide confirmación.
- Para leer providers se usa `Consumer<T>` en el árbol de widgets y `Provider.of<T>(context, listen: false)` en los callbacks.
