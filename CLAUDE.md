# 🎮 Tactical Turn-Based Game - Godot 4.5+

## 📋 Contexte du Projet

Jeu tactique au tour par tour en **Godot 4.5+** avec système de grille, gestion des PA (Points d'Action), et combat entre peuples aux pouvoirs uniques.

**État actuel :** 16 fichiers créés, UI d'action en place, première compétence (Fireball) implémentée.

---

## 🏗️ Architecture du Squelette (COMPLÉTÉ)

### Structure des Fichiers
```
res://
├── scripts/
│   ├── main.gd              # Point d'entrée, orchestration
│   ├── grid_manager.gd      # Grille + AStar pathfinding
│   ├── turn_manager.gd      # Gestion des tours (autoload)
│   ├── player_controller.gd # Input joueur + sélection unités
│   ├── unit_base.gd         # Classe de base des unités
│   ├── ability_base.gd      # Classe de base des compétences
│   ├── event_bus.gd         # Signaux globaux (autoload)
│   └── abilities/
│       └── ability_fireball.gd  # Première compétence
├── resources/
│   └── unit_stats.gd        # Resource pour stats (PV, PA, Move)
├── ui/
│   └── action_ui.gd         # Menu d'actions (Move/Attack/End)
└── scenes/
    ├── main.tscn
    └── unit.tscn
```

### Fichiers Corrigés ✅
| Fichier | Correction Appliquée |
|---------|---------------------|
| `ability_base.gd` | Signal `execution_finished` + `await get_tree().process_frame` |
| `turn_manager.gd` | `await ability.execute()` + logique win condition simplifiée |
| `player_controller.gd` | Supprimé double appel `on_cell_clicked()` |
| `unit_base.gd` | Guard clause `if not stats: return` dans `reset_ap()` |
| `main.gd` | Remplacé `_process()` par signaux pour `queue_redraw()` |

---

## 🚨 PHASE ACTUELLE : Intégration UI ↔ Gameplay

### ⚠️ Piège à Éviter : "Spaghetti UI"
L'UI ne doit **JAMAIS** modifier directement les PV ou PA. Elle envoie des **intentions** au Controller.

### Architecture de Communication
```
[ActionUI] --signal--> [EventBus] --signal--> [PlayerController] --appel--> [Unit]
```

### États du PlayerController
```gdscript
enum State {
    STATE_IDLE,                    # Attente de sélection
    STATE_UNIT_SELECTED,           # Unité sélectionnée, menu affiché
    STATE_SELECTING_MOVE_DESTINATION,  # Clic pour destination
    STATE_SELECTING_ATTACK_TARGET      # Clic pour cible
}
```

---

## ✅ Checklist de Validation UI

### 1. Indépendance de l'UI
- [ ] ActionUI n'appelle **pas** `unit.attack()` directement
- [ ] Les boutons émettent des signaux via EventBus
- [ ] Exemple correct : `EventBus.action_selected.emit("attack")`

### 2. Flux de Sélection
- [ ] Clic sur unité alliée (avec PA) → UI apparaît
- [ ] Clic dans le vide → UI disparaît
- [ ] Clic sur ennemi → Rien (ou info)

### 3. Flux d'Action
- [ ] Bouton "Move" → Mode sélection destination
- [ ] Bouton "Attack" → Mode sélection cible
- [ ] Bouton "End Turn" → `TurnManager.end_turn()`

### 4. Annulation
- [ ] Clic droit / Echap → Annule l'action en cours
- [ ] Retour à l'état `STATE_UNIT_SELECTED`

---

## 🔧 Code d'Intégration Requis

### EventBus (autoload)
```gdscript
# res://scripts/event_bus.gd
extends Node

signal unit_selected(unit: UnitBase)
signal unit_deselected
signal action_selected(action_name: String)
signal action_cancelled
```

### ActionUI - Signaux
```gdscript
# res://scripts/ui/action_ui.gd
func _on_move_pressed():
    EventBus.action_selected.emit("move")

func _on_attack_pressed():
    EventBus.action_selected.emit("attack")

func _on_end_turn_pressed():
    EventBus.action_selected.emit("end_turn")
```

### PlayerController - États
```gdscript
# res://scripts/controllers/player_controller.gd
func _ready():
    EventBus.action_selected.connect(_on_action_selected)

func _on_action_selected(action: String):
    match action:
        "move":
            current_state = State.STATE_SELECTING_MOVE_DESTINATION
            action_ui.visible = false
        "attack":
            current_state = State.STATE_SELECTING_ATTACK_TARGET
            action_ui.visible = false
        "end_turn":
            TurnManager.end_turn()

func _input(event):
    if event.is_action_pressed("ui_cancel"):  # Echap
        _cancel_action()

func _cancel_action():
    current_state = State.STATE_UNIT_SELECTED
    action_ui.visible = true
    EventBus.action_cancelled.emit()
```

---

## 🧪 Test d'Intégration (À faire manuellement)

1. **Lance le jeu** (F5)
2. **Clique sur ton Mage** → Le menu doit s'ouvrir
3. **Clique sur "Move"** → Le menu se ferme
4. **Clique sur une case** → L'unité bouge
5. **Le menu réapparaît** (s'il reste des PA)
6. **Appuie sur Echap** → Annule et réaffiche le menu

✅ Si ça marche = **Tactical RPG fonctionnel !**

---

## 🎯 PHASE SUIVANTE : Finalisation Gameplay

### 3 Piliers Manquants
| Pilier | Description | Fichier Principal |
|--------|-------------|-------------------|
| 👁️ Visuels | Highlight des cases accessibles | `grid_overlay.gd` |
| 👑 Leaders | Condition de victoire (tuer le Leader) | `unit_stats.gd` + `turn_manager.gd` |
| 💥 AOE | Zones d'effet pour les compétences | `ability_base.gd` |

---

## 👁️ ÉTAPE 1 : Visual Feedback (GridOverlay)

### Nouveau Fichier : `grid_overlay.gd`
```gdscript
# res://scripts/grid/grid_overlay.gd
extends TileMapLayer

const HIGHLIGHT_TILE_ID = 0  # ID du tile blanc dans le TileSet

func highlight_cells(cells: Array[Vector2i], color: Color) -> void:
    clear()
    modulate = color
    for cell in cells:
        set_cell(cell, HIGHLIGHT_TILE_ID, Vector2i.ZERO)

func clear() -> void:
    clear()  # Efface toutes les tuiles
```

### Ajout dans `game_board.gd`
```gdscript
func get_reachable_cells(start: Vector2i, max_range: int) -> Array[Vector2i]:
    var reachable: Array[Vector2i] = []
    var visited: Dictionary = {}
    var queue: Array = [[start, 0]]
    
    while queue.size() > 0:
        var current = queue.pop_front()
        var pos: Vector2i = current[0]
        var dist: int = current[1]
        
        if visited.has(pos) or dist > max_range:
            continue
        if not is_walkable(pos):
            continue
            
        visited[pos] = true
        reachable.append(pos)
        
        for neighbor in get_neighbors(pos):
            if not visited.has(neighbor):
                queue.append([neighbor, dist + 1])
    
    return reachable
```

### Mise à jour `player_controller.gd`
```gdscript
@onready var grid_overlay: TileMapLayer = $"../GridOverlay"

func _on_action_selected(action: String):
    match action:
        "move":
            current_state = State.STATE_SELECTING_MOVE_DESTINATION
            action_ui.visible = false
            # Affiche les cases accessibles en BLEU
            var cells = game_board.get_reachable_cells(
                selected_unit.grid_position, 
                selected_unit.stats.move_range
            )
            grid_overlay.highlight_cells(cells, Color.CORNFLOWER_BLUE)
        "attack":
            current_state = State.STATE_SELECTING_ATTACK_TARGET
            action_ui.visible = false

func _cancel_action():
    current_state = State.STATE_UNIT_SELECTED
    action_ui.visible = true
    grid_overlay.clear()  # Efface le highlight
```

### Setup dans l'Éditeur
1. Créer un `TileMapLayer` nommé **"GridOverlay"**
2. Créer un `TileSet` avec un carré blanc (ou `icon.svg`)
3. Assigner le TileSet au TileMapLayer
4. Mettre le TileMapLayer **au-dessus** de la grille principale (Z-Index)

---

## 👑 ÉTAPE 2 : Gestion du Leader & Victoire

### Mise à jour `unit_stats.gd`
```gdscript
# res://resources/unit_stats.gd
extends Resource
class_name UnitStats

@export var max_hp: int = 10
@export var max_ap: int = 3
@export var move_range: int = 4
@export var attack: int = 3
@export var defense: int = 1
@export var is_leader: bool = false  # ← NOUVEAU
```

### Mise à jour `unit_base.gd`
```gdscript
signal unit_died(unit: UnitBase)

func take_damage(amount: int) -> void:
    current_hp -= amount
    current_hp = max(0, current_hp)
    
    if current_hp <= 0:
        _die()

func _die() -> void:
    # Animation de mort (fade out)
    var tween = create_tween()
    tween.tween_property(self, "modulate:a", 0.0, 0.5)
    tween.tween_callback(_on_death_animation_finished)

func _on_death_animation_finished() -> void:
    unit_died.emit(self)
    queue_free()
```

### Mise à jour `turn_manager.gd`
```gdscript
func register_unit(unit: UnitBase) -> void:
    units.append(unit)
    unit.unit_died.connect(_on_unit_died)

func _on_unit_died(unit: UnitBase) -> void:
    units.erase(unit)
    
    if unit.stats.is_leader:
        # Le Leader est mort → GAME OVER
        var winner_team = 1 if unit.team == 0 else 0
        EventBus.game_ended.emit(winner_team)
        print("🏆 GAME OVER - Team %d wins!" % winner_team)
    else:
        # Vérifie si l'équipe a encore des unités
        var team_alive = units.filter(func(u): return u.team == unit.team)
        if team_alive.is_empty():
            var winner_team = 1 if unit.team == 0 else 0
            EventBus.game_ended.emit(winner_team)
```

### Ajout dans `event_bus.gd`
```gdscript
signal game_ended(winner_team: int)
```

---

## 💥 ÉTAPE 3 : Système de Zones (AOE)

### Mise à jour `ability_base.gd`
```gdscript
# res://scripts/abilities/ability_base.gd
extends Resource
class_name AbilityBase

signal execution_finished

@export var ability_name: String = "Ability"
@export var ap_cost: int = 1
@export var cooldown: int = 0
@export var range_min: int = 1
@export var range_max: int = 4

# Retourne les cases affectées (à surcharger)
func get_target_pattern(center: Vector2i) -> Array[Vector2i]:
    return [center]  # Par défaut : une seule case

func execute(caster: UnitBase, target) -> void:
    # À surcharger
    await caster.get_tree().process_frame
    execution_finished.emit()
```

### Exemple : `ability_fireball.gd` (Croix AOE)
```gdscript
# res://scripts/abilities/ability_fireball.gd
extends AbilityBase
class_name AbilityFireball

func _init():
    ability_name = "Fireball"
    ap_cost = 2
    cooldown = 2
    range_max = 5

# Pattern en croix (+)
func get_target_pattern(center: Vector2i) -> Array[Vector2i]:
    return [
        center,
        center + Vector2i.UP,
        center + Vector2i.DOWN,
        center + Vector2i.LEFT,
        center + Vector2i.RIGHT
    ]

func execute(caster: UnitBase, targets: Array[UnitBase]) -> void:
    for target in targets:
        target.take_damage(4)  # Dégâts de feu
    await caster.get_tree().process_frame
    execution_finished.emit()
```

### Prévisualisation AOE dans `player_controller.gd`
```gdscript
func _unhandled_input(event: InputEvent) -> void:
    if current_state == State.STATE_SELECTING_ATTACK_TARGET:
        if event is InputEventMouseMotion:
            var mouse_cell = game_board.get_cell_at_mouse()
            var pattern = current_ability.get_target_pattern(mouse_cell)
            grid_overlay.highlight_cells(pattern, Color.INDIAN_RED)
```

---

## 🧪 Tests de Validation Finale

### Test 1 : Visuels (Move)
- [ ] Sélectionne une unité → Clique "Move"
- [ ] Les cases accessibles s'affichent en **BLEU**
- [ ] Le highlight disparaît après le mouvement

### Test 2 : Visuels (Attack AOE)
- [ ] Sélectionne Fireball → Mode attaque
- [ ] Une **croix ROUGE** suit la souris
- [ ] Le pattern correspond à la compétence

### Test 3 : Leader & Victoire
- [ ] Configure un Leader (`is_leader = true`)
- [ ] Tue le Leader adverse
- [ ] Message "GAME OVER" affiché
- [ ] Le jeu s'arrête ou affiche écran de fin

### Test 4 : Mort d'Unité
- [ ] Tue une unité non-leader
- [ ] Animation de fade-out jouée
- [ ] L'unité disparaît de la liste

---

## 🔬 STRESS TEST : AOE & Victoire

### Scénario de Test Automatisé
```gdscript
# res://scripts/main.gd - Ajout temporaire dans _ready()
func _ready():
    # === STRESS TEST SETUP ===
    EventBus.game_ended.connect(_on_game_ended_test)
    
    # Créer les unités de test
    var mage = _spawn_unit(Vector2i(2, 2), 0, false, preload("res://resources/abilities/fireball.tres"))
    var garde = _spawn_unit(Vector2i(2, 4), 1, false)  # 2 cases sous le mage
    var roi = _spawn_unit(Vector2i(3, 4), 1, true)     # Leader, à droite du garde
    
    # Simuler l'attaque après 1 seconde
    await get_tree().create_timer(1.0).timeout
    _run_aoe_test(mage, Vector2i(2, 4))

func _run_aoe_test(caster: UnitBase, target_cell: Vector2i):
    var ability = caster.abilities[0] as AbilityFireball
    var pattern = ability.get_target_pattern(target_cell)
    
    print("🎯 Pattern AOE: ", pattern)
    # Attendu: [2,4], [1,4], [3,4], [2,3], [2,5]
    
    # Trouver les unités dans la zone
    var targets: Array[UnitBase] = []
    for cell in pattern:
        var unit = game_board.get_unit_at(cell)
        if unit:
            print("  → Touché: %s en %s (HP: %d)" % [unit.name, cell, unit.current_hp])
            targets.append(unit)
    
    # Exécuter l'attaque
    ability.execute(caster, targets)
    
    await ability.execution_finished
    for t in targets:
        if is_instance_valid(t):
            print("  → %s HP après: %d" % [t.name, t.current_hp])

func _on_game_ended_test(winner: int):
    print("🏆 VICTOIRE DÉTECTÉE - Team %d gagne !" % winner)
```

### Résultats Attendus
| Vérification | Résultat Attendu |
|--------------|------------------|
| Pattern affiché | `[(2,4), (1,4), (3,4), (2,3), (2,5)]` |
| Garde touché | Oui (centre de la croix) |
| Roi touché | Oui (droite de la croix) |
| Signal `game_ended` | Émis si Roi meurt (is_leader) |

---

## 🗺️ FEUILLE DE ROUTE (Prochaines Étapes)

### ⚠️ STOP : Ne plus toucher au moteur après validation du Stress Test

---

### 📊 Étape 1 : HUD & Interface (Priorité Haute)

**Objectif :** Remplacer les `print()` par une vraie UI

#### Barre de Vie (HealthBar)
```gdscript
# res://scripts/ui/health_bar.gd
extends TextureProgressBar

@export var unit: UnitBase

func _ready():
    if unit:
        unit.hp_changed.connect(_update_bar)
        _update_bar(unit.current_hp, unit.stats.max_hp)

func _update_bar(current: int, max_val: int):
    max_value = max_val
    value = current
```

#### Panneau Game Over
```gdscript
# res://scripts/ui/game_over_panel.gd
extends CanvasLayer

@onready var label: Label = $Panel/Label
@onready var replay_btn: Button = $Panel/ReplayButton

func _ready():
    visible = false
    EventBus.game_ended.connect(_show_game_over)
    replay_btn.pressed.connect(_on_replay)

func _show_game_over(winner: int):
    label.text = "Team %d Wins!" % winner
    visible = true
    get_tree().paused = true

func _on_replay():
    get_tree().paused = false
    get_tree().reload_current_scene()
```

---

### 🎭 Étape 2 : Création des Peuples (Data)

**Conseil :** Utiliser un tableur pour l'équilibrage, puis générer les `.tres`

#### Template de Champion
```
res://resources/champions/
├── feu/
│   ├── mage_feu_stats.tres
│   └── guerrier_feu_stats.tres
├── temps/
│   ├── oracle_stats.tres
│   └── gardien_temps_stats.tres
└── ...
```

#### Équilibrage Suggéré
| Peuple | HP | PA | Move | Atk | Def | Spécialité |
|--------|----|----|------|-----|-----|------------|
| 🔥 Feu | 8 | 3 | 3 | 5 | 1 | Dégâts AOE |
| ⏰ Temps | 10 | 4 | 4 | 2 | 2 | Contrôle |
| 🌿 Nature | 12 | 3 | 3 | 3 | 3 | Sustain |
| ⚡ Foudre | 6 | 3 | 5 | 4 | 1 | Mobilité |
| 🪨 Terre | 15 | 2 | 2 | 3 | 5 | Tank |

---

### ⚔️ Étape 3 : Compétences Uniques

#### Peuple du Temps - Rembobinage
```gdscript
# res://scripts/abilities/ability_rewind.gd
extends AbilityBase
class_name AbilityRewind

var position_history: Dictionary = {}  # unit_id -> last_position

func execute(caster: UnitBase, target: UnitBase) -> void:
    if position_history.has(target.get_instance_id()):
        var old_pos = position_history[target.get_instance_id()]
        target.teleport_to(old_pos)
        target.current_hp = min(target.current_hp + 3, target.stats.max_hp)
    await caster.get_tree().process_frame
    execution_finished.emit()
```

#### Peuple de la Terre - Mur
```gdscript
# res://scripts/abilities/ability_wall.gd
extends AbilityBase
class_name AbilityWall

func get_target_pattern(center: Vector2i) -> Array[Vector2i]:
    return [center]  # Une seule case

func execute(caster: UnitBase, target_cell: Vector2i) -> void:
    GameBoard.set_cell_walkable(target_cell, false)
    # Spawner un sprite de mur
    var wall = preload("res://scenes/wall.tscn").instantiate()
    wall.position = GameBoard.cell_to_world(target_cell)
    caster.get_tree().root.add_child(wall)
    await caster.get_tree().process_frame
    execution_finished.emit()
```

---

### 🤖 Étape 4 : Intelligence Artificielle

#### AIController Basique
```gdscript
# res://scripts/controllers/ai_controller.gd
extends Node
class_name AIController

func play_turn(units: Array[UnitBase], game_board: GameBoard) -> void:
    for unit in units:
        if unit.current_ap <= 0:
            continue
        
        var action = _decide_action(unit, game_board)
        await _execute_action(unit, action)

func _decide_action(unit: UnitBase, board: GameBoard) -> Dictionary:
    var enemies = _get_enemies(unit)
    var leader = _find_enemy_leader(enemies)
    
    # Priorité 1: Tuer le leader si possible
    if leader and _can_attack(unit, leader):
        return {type = "attack", target = leader}
    
    # Priorité 2: Tuer l'ennemi le plus faible
    var weakest = _find_weakest(enemies)
    if weakest and _can_attack(unit, weakest):
        return {type = "attack", target = weakest}
    
    # Priorité 3: Se rapprocher
    var closest = _find_closest(unit, enemies)
    if closest:
        return {type = "move", target = _get_closest_cell(unit, closest)}
    
    return {type = "end_turn"}
```

---

## 🌐 MULTIJOUEUR EN LIGNE (PvP)

### 🎯 Modes de Jeu Supportés
| Mode | Description | Statut |
|------|-------------|--------|
| **Hotseat** | 2 joueurs, même écran | ✅ Déjà fonctionnel |
| **En Ligne** | 2 joueurs, via Internet | 🔄 À implémenter |

### 🏗️ Architecture Réseau : "Authoritative Server"

```
┌─────────────────────┐         ┌─────────────────────┐
│   JOUEUR 1 (HOST)   │◄───────►│   JOUEUR 2 (CLIENT) │
│   ID = 1            │  ENet   │   ID = auto         │
│   Team 0            │         │   Team 1            │
│   ══════════════    │         │                     │
│   SERVEUR LOGIQUE   │         │   Envoie intentions │
│   (valide tout)     │         │   Reçoit résultats  │
└─────────────────────┘         └─────────────────────┘
```

**Principe :** Le Host (Serveur) valide toutes les actions. Le Client envoie des "intentions", jamais d'exécutions directes.

---

### 📁 Nouveaux Fichiers Requis

```
res://
├── scripts/
│   ├── core/
│   │   └── network_manager.gd    # Autoload - Gestion connexion
│   └── controllers/
│       └── player_controller.gd  # Modifié pour réseau
├── ui/
│   └── lobby_ui.gd               # Menu Host/Join
└── scenes/
    └── lobby.tscn                # Scène de connexion
```

---

### 🔧 Étape 1 : Network Manager (Autoload)

```gdscript
# res://scripts/core/network_manager.gd
extends Node

signal player_connected(id: int)
signal player_disconnected(id: int)
signal connection_failed
signal server_started
signal connected_to_server

const DEFAULT_PORT = 7777
const MAX_PLAYERS = 2

var peer: ENetMultiplayerPeer
var player_info: Dictionary = {}  # {id: {name, team}}

func _ready():
    multiplayer.peer_connected.connect(_on_peer_connected)
    multiplayer.peer_disconnected.connect(_on_peer_disconnected)
    multiplayer.connected_to_server.connect(_on_connected_to_server)
    multiplayer.connection_failed.connect(_on_connection_failed)

# === HOST (Serveur) ===
func host_game(player_name: String = "Host") -> Error:
    peer = ENetMultiplayerPeer.new()
    var error = peer.create_server(DEFAULT_PORT, MAX_PLAYERS)
    if error != OK:
        return error
    
    multiplayer.multiplayer_peer = peer
    player_info[1] = {name = player_name, team = 0}
    server_started.emit()
    print("🖥️ Serveur démarré sur le port %d" % DEFAULT_PORT)
    return OK

# === CLIENT (Rejoindre) ===
func join_game(ip: String, player_name: String = "Client") -> Error:
    peer = ENetMultiplayerPeer.new()
    var error = peer.create_client(ip, DEFAULT_PORT)
    if error != OK:
        return error
    
    multiplayer.multiplayer_peer = peer
    return OK

# === Callbacks ===
func _on_peer_connected(id: int):
    print("👤 Joueur connecté: %d" % id)
    if multiplayer.is_server():
        # Assigner Team 1 au client
        player_info[id] = {name = "Player_%d" % id, team = 1}
        _sync_player_info.rpc()
    player_connected.emit(id)

func _on_peer_disconnected(id: int):
    print("👤 Joueur déconnecté: %d" % id)
    player_info.erase(id)
    player_disconnected.emit(id)

func _on_connected_to_server():
    print("✅ Connecté au serveur!")
    connected_to_server.emit()

func _on_connection_failed():
    print("❌ Échec de connexion")
    connection_failed.emit()

@rpc("authority", "call_local", "reliable")
func _sync_player_info():
    # Synchroniser les infos joueurs sur tous les clients
    pass

func get_my_team() -> int:
    return player_info.get(multiplayer.get_unique_id(), {}).get("team", -1)

func is_my_turn(current_team: int) -> bool:
    return get_my_team() == current_team
```

---

### 🔧 Étape 2 : Lobby UI

```gdscript
# res://scripts/ui/lobby_ui.gd
extends Control

@onready var host_btn: Button = $VBox/HostButton
@onready var join_btn: Button = $VBox/JoinButton
@onready var ip_input: LineEdit = $VBox/IPInput
@onready var status_label: Label = $VBox/StatusLabel

func _ready():
    host_btn.pressed.connect(_on_host_pressed)
    join_btn.pressed.connect(_on_join_pressed)
    
    NetworkManager.server_started.connect(_on_server_started)
    NetworkManager.connected_to_server.connect(_on_connected)
    NetworkManager.player_connected.connect(_on_player_joined)
    NetworkManager.connection_failed.connect(_on_failed)

func _on_host_pressed():
    status_label.text = "Création du serveur..."
    var error = NetworkManager.host_game()
    if error != OK:
        status_label.text = "Erreur: %s" % error

func _on_join_pressed():
    var ip = ip_input.text if ip_input.text else "127.0.0.1"
    status_label.text = "Connexion à %s..." % ip
    var error = NetworkManager.join_game(ip)
    if error != OK:
        status_label.text = "Erreur: %s" % error

func _on_server_started():
    status_label.text = "🖥️ En attente d'un joueur..."

func _on_connected():
    status_label.text = "✅ Connecté! Chargement..."
    _start_game()

func _on_player_joined(id: int):
    if multiplayer.is_server() and NetworkManager.player_info.size() >= 2:
        status_label.text = "✅ Partie complète! Démarrage..."
        _start_game.rpc()

func _on_failed():
    status_label.text = "❌ Échec de connexion"

@rpc("authority", "call_local", "reliable")
func _start_game():
    get_tree().change_scene_to_file("res://scenes/main.tscn")
```

---

### 🔧 Étape 3 : Ownership des Unités

```gdscript
# res://scripts/units/unit_base.gd - AJOUTS
var owner_peer_id: int = 1  # Par défaut: le host

func _ready():
    # ... code existant ...
    
    # Configurer l'autorité réseau
    set_multiplayer_authority(owner_peer_id)

func is_owned_by_local_player() -> bool:
    return multiplayer.get_unique_id() == owner_peer_id
```

---

### 🔧 Étape 4 : Filtrage des Inputs (CRITIQUE)

```gdscript
# res://scripts/controllers/player_controller.gd - MODIFICATIONS

var my_team: int = -1

func _ready():
    # ... code existant ...
    
    # Récupérer ma team depuis le NetworkManager
    if multiplayer.has_multiplayer_peer():
        my_team = NetworkManager.get_my_team()

func _unhandled_input(event: InputEvent) -> void:
    # === VÉRIFICATION RÉSEAU ===
    if multiplayer.has_multiplayer_peer():
        # Vérifier si c'est mon tour
        if not NetworkManager.is_my_turn(TurnManager.current_team):
            return
    
    # ... reste du code input ...
```

---

### 🔧 Étape 5 : Synchronisation RPC

```gdscript
# res://scripts/controllers/player_controller.gd - AJOUTS RPC

# Quand je veux bouger une unité
func _request_move(unit: UnitBase, target_cell: Vector2i):
    if multiplayer.is_server():
        # Je suis le serveur, j'exécute direct
        _execute_move(unit.get_path(), target_cell)
    else:
        # Je suis client, j'envoie au serveur
        _request_move_rpc.rpc_id(1, unit.get_path(), target_cell)

@rpc("any_peer", "reliable")
func _request_move_rpc(unit_path: NodePath, target_cell: Vector2i):
    # Serveur reçoit la demande
    if not multiplayer.is_server():
        return
    
    var unit = get_node_or_null(unit_path) as UnitBase
    if not unit:
        return
    
    # === ANTI-TRICHE ===
    var sender_id = multiplayer.get_remote_sender_id()
    if unit.owner_peer_id != sender_id:
        print("⚠️ Triche détectée: %d essaie de bouger l'unité de %d" % [sender_id, unit.owner_peer_id])
        return
    
    # Valide le mouvement
    if not _is_valid_move(unit, target_cell):
        return
    
    # Exécute sur tous les clients
    _execute_move.rpc(unit_path, target_cell)

@rpc("authority", "call_local", "reliable")
func _execute_move(unit_path: NodePath, target_cell: Vector2i):
    var unit = get_node(unit_path) as UnitBase
    unit.move_to(target_cell)
```

---

### ✅ Checklist Multijoueur

#### Configuration Projet
- [ ] `NetworkManager` ajouté aux Autoloads
- [ ] `lobby.tscn` créée avec les boutons Host/Join
- [ ] `main.tscn` chargée via Lobby (pas en démarrage direct)

#### Tests Connexion
- [ ] Host démarre → "En attente..."
- [ ] Client rejoint (127.0.0.1) → Les deux passent à Main
- [ ] Déconnexion gérée proprement

#### Tests Gameplay
- [ ] Joueur 1 ne peut bouger que Team 0
- [ ] Joueur 2 ne peut bouger que Team 1
- [ ] Les mouvements sont synchronisés
- [ ] Les attaques/dégâts sont synchronisés
- [ ] Game Over affiché sur les deux écrans

#### Anti-Triche
- [ ] Impossible de bouger une unité adverse
- [ ] Impossible de jouer hors de son tour
- [ ] Serveur valide toutes les actions

---

## 🔒 FINALISATION MULTIJOUEUR P2P (SÉCURITÉ & RPC)

> **Contexte :** Le Lobby fonctionne (Host/Join). NetworkManager existe.
> **Objectif :** Sécuriser les inputs (chacun son tour) et synchroniser TOUTES les actions sur tous les écrans.

---

### 📋 Vue d'Ensemble du Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    FLOW RÉSEAU SÉCURISÉ                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  [JOUEUR CLIQUE]                                                        │
│       │                                                                 │
│       ▼                                                                 │
│  ┌─────────────────┐                                                    │
│  │ PlayerController │                                                   │
│  │ Guard Clauses:   │                                                   │
│  │ - is_my_turn()?  │ ──NO──► return (ignore input)                    │
│  │ - is_my_unit()?  │                                                   │
│  └────────┬────────┘                                                    │
│           │ YES                                                         │
│           ▼                                                             │
│  ┌─────────────────┐                                                    │
│  │ RPC Request     │ ─────► rpc_request_move.rpc(...)                  │
│  └────────┬────────┘        (envoi à TOUS via call_local)              │
│           │                                                             │
│           ▼                                                             │
│  ┌─────────────────────────────────────────────────────────────┐       │
│  │                    TOUS LES PEERS                            │       │
│  │  ┌──────────────┐     ┌──────────────┐                      │       │
│  │  │    HOST      │     │   CLIENT     │                      │       │
│  │  │ (Authority)  │     │              │                      │       │
│  │  │              │     │              │                      │       │
│  │  │ 1. Valide    │     │ 1. Reçoit    │                      │       │
│  │  │ 2. Exécute   │     │ 2. Exécute   │                      │       │
│  │  │ 3. Broadcast │     │    (sync)    │                      │       │
│  │  └──────────────┘     └──────────────┘                      │       │
│  └─────────────────────────────────────────────────────────────┘       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### 🎯 Tâche 1 : Identification Joueur (network_manager.gd)

**Fichier :** `res://autoload/network_manager.gd`

**Variables à ajouter :**
```gdscript
## Identification du joueur local
var my_player_id: int = 0  # 0 = Local, 1 = Host, X = Client (peer ID)

## Mapping peer_id -> team_index
## Host (ID 1) = Team 0, Premier Client = Team 1
var player_teams: Dictionary = {}  # { peer_id: team_index }

## Constantes
const TEAM_HOST: int = 0
const TEAM_CLIENT: int = 1
const LOCAL_PLAYER_ID: int = 0
```

**Fonction d'identification :**
```gdscript
## Détermine si c'est le tour du joueur local
## @param active_team_index: L'index de l'équipe dont c'est le tour (depuis TurnManager)
## @return: true si c'est MON tour
func is_my_turn(active_team_index: int) -> bool:
    # Mode Local (pas de connexion réseau) → Toujours mon tour
    if not multiplayer.has_multiplayer_peer():
        return true
    
    # Mode Réseau → Vérifier si ma team correspond
    var my_team = player_teams.get(my_player_id, -1)
    return my_team == active_team_index

## Détermine si une unité m'appartient
## @param unit: L'unité à vérifier
## @return: true si c'est MON unité
func is_my_unit(unit: UnitBase) -> bool:
    # Mode Local → Toutes les unités de l'équipe active
    if not multiplayer.has_multiplayer_peer():
        return unit.team_index == TurnManager.current_team
    
    # Mode Réseau → Vérifier ownership
    var my_team = player_teams.get(my_player_id, -1)
    return unit.team_index == my_team

## Récupérer ma team
func get_my_team() -> int:
    if not multiplayer.has_multiplayer_peer():
        return TurnManager.current_team  # Local: team active
    return player_teams.get(my_player_id, -1)
```

**Initialisation au moment de la connexion :**
```gdscript
func _on_server_created():
    my_player_id = 1  # Host = ID 1
    player_teams[1] = TEAM_HOST  # Host = Team 0
    print("🖥️ Host démarré (ID: 1, Team: 0)")

func _on_connected_to_server():
    my_player_id = multiplayer.get_unique_id()
    print("🎮 Connecté au serveur (ID: %d)" % my_player_id)

func _on_peer_connected(id: int):
    if multiplayer.is_server():
        # Assigner Team 1 au premier client
        player_teams[id] = TEAM_CLIENT
        print("👤 Client %d assigné à Team %d" % [id, TEAM_CLIENT])
        
        # Synchroniser les teams sur tous les clients
        _sync_teams.rpc()

@rpc("authority", "call_local", "reliable")
func _sync_teams():
    if multiplayer.is_server():
        # Envoyer le dictionnaire complet
        _receive_teams.rpc(player_teams)

@rpc("any_peer", "reliable")
func _receive_teams(teams: Dictionary):
    player_teams = teams
    print("📋 Teams synchronisées: %s" % str(player_teams))
```

---

### 🎯 Tâche 2 : Sécuriser l'Input (player_controller.gd)

**Fichier :** `res://scripts/controllers/player_controller.gd`

**Guard Clauses au début de _unhandled_input :**
```gdscript
func _unhandled_input(event: InputEvent) -> void:
    # ═══════════════════════════════════════════════════════════════
    # GUARD CLAUSE 1: Vérifier si c'est mon tour
    # ═══════════════════════════════════════════════════════════════
    if not NetworkManager.is_my_turn(TurnManager.current_team):
        return  # Pas mon tour → Ignorer tout input
    
    # ═══════════════════════════════════════════════════════════════
    # GUARD CLAUSE 2: Ignorer si partie terminée ou UI bloquée
    # ═══════════════════════════════════════════════════════════════
    if TurnManager.is_game_over or _is_ui_blocking:
        return
    
    # ... reste du code existant (traitement du clic) ...
```

**Vérification avant action sur unité :**
```gdscript
func _on_cell_clicked(cell: Vector2i) -> void:
    var unit = GridManager.get_unit_at(cell)
    
    if unit:
        # ═══════════════════════════════════════════════════════════
        # GUARD: Vérifier que c'est MON unité
        # ═══════════════════════════════════════════════════════════
        if not NetworkManager.is_my_unit(unit):
            print("⚠️ Ce n'est pas mon unité!")
            return
        
        _select_unit(unit)
    else:
        # Clic sur case vide...
```

---

### 🎯 Tâche 3 : Synchronisation RPC des Actions

**Fichier :** `res://scripts/controllers/player_controller.gd`

**Principe du "call_local" :**
```
@rpc("call_local") signifie:
- J'exécute la fonction chez MOI immédiatement
- ET j'envoie l'appel à tous les autres peers
- Résultat: TOUT LE MONDE exécute le même code, synchronisé
```

**Déclaration des fonctions RPC :**
```gdscript
# ═══════════════════════════════════════════════════════════════════════════
#                           RPC FUNCTIONS
# ═══════════════════════════════════════════════════════════════════════════

## RPC: Demande de mouvement
## @rpc("call_local"): Exécute ici ET envoie aux autres
## @rpc("reliable"): Garantit la livraison (TCP-like)
## @rpc("any_peer"): N'importe qui peut appeler (pas juste le serveur)
@rpc("call_local", "reliable", "any_peer")
func rpc_request_move(unit_path: NodePath, target_cell_x: int, target_cell_y: int) -> void:
    var unit = get_node_or_null(unit_path) as UnitBase
    if not unit:
        push_error("RPC Move: Unité introuvable: %s" % unit_path)
        return
    
    var target_cell = Vector2i(target_cell_x, target_cell_y)
    
    # ═══════════════════════════════════════════════════════════════
    # VALIDATION SERVEUR (Anti-triche)
    # ═══════════════════════════════════════════════════════════════
    if multiplayer.is_server():
        var sender_id = multiplayer.get_remote_sender_id()
        # sender_id == 0 si c'est nous-mêmes (call_local)
        if sender_id != 0 and not _validate_action(sender_id, unit):
            push_warning("⚠️ Action rejetée: Joueur %d ne peut pas contrôler cette unité" % sender_id)
            return
    
    # Émettre le signal EventBus → La logique existante s'exécute
    EventBus.move_requested.emit(unit, target_cell)


## RPC: Demande d'attaque
@rpc("call_local", "reliable", "any_peer")
func rpc_request_attack(attacker_path: NodePath, target_path: NodePath) -> void:
    var attacker = get_node_or_null(attacker_path) as UnitBase
    var target = get_node_or_null(target_path) as UnitBase
    
    if not attacker or not target:
        push_error("RPC Attack: Unité introuvable")
        return
    
    # Validation serveur
    if multiplayer.is_server():
        var sender_id = multiplayer.get_remote_sender_id()
        if sender_id != 0 and not _validate_action(sender_id, attacker):
            push_warning("⚠️ Attaque rejetée: Joueur %d" % sender_id)
            return
    
    EventBus.attack_requested.emit(attacker, target)


## RPC: Demande d'utilisation de compétence
@rpc("call_local", "reliable", "any_peer")
func rpc_request_ability(caster_path: NodePath, ability_id: String, target_x: int, target_y: int) -> void:
    var caster = get_node_or_null(caster_path) as UnitBase
    if not caster:
        push_error("RPC Ability: Caster introuvable")
        return
    
    var target_cell = Vector2i(target_x, target_y)
    
    # Validation serveur
    if multiplayer.is_server():
        var sender_id = multiplayer.get_remote_sender_id()
        if sender_id != 0 and not _validate_action(sender_id, caster):
            push_warning("⚠️ Compétence rejetée: Joueur %d" % sender_id)
            return
    
    EventBus.ability_requested.emit(caster, ability_id, target_cell)


## RPC: Fin de tour
@rpc("call_local", "reliable", "any_peer")
func rpc_end_turn() -> void:
    # Validation: Seul le joueur actif peut terminer le tour
    if multiplayer.is_server():
        var sender_id = multiplayer.get_remote_sender_id()
        if sender_id != 0:
            var sender_team = NetworkManager.player_teams.get(sender_id, -1)
            if sender_team != TurnManager.current_team:
                push_warning("⚠️ Fin de tour rejetée: Pas le tour de %d" % sender_id)
                return
    
    EventBus.end_turn_requested.emit()


## Validation anti-triche côté serveur
func _validate_action(sender_peer_id: int, unit: UnitBase) -> bool:
    # Vérifier que le joueur contrôle bien cette équipe
    var sender_team = NetworkManager.player_teams.get(sender_peer_id, -1)
    if sender_team != unit.team_index:
        return false
    
    # Vérifier que c'est bien le tour de cette équipe
    if unit.team_index != TurnManager.current_team:
        return false
    
    return true
```

**Modification des appels (remplacer les anciens) :**
```gdscript
# ═══════════════════════════════════════════════════════════════════════════
# AVANT (appel direct - NE FONCTIONNE PAS EN RÉSEAU):
# EventBus.move_requested.emit(selected_unit, target_cell)
#
# APRÈS (appel RPC - SYNCHRONISÉ SUR TOUS LES CLIENTS):
# rpc_request_move.rpc(selected_unit.get_path(), target_cell.x, target_cell.y)
# ═══════════════════════════════════════════════════════════════════════════

func _execute_move_action(target_cell: Vector2i) -> void:
    if not selected_unit:
        return
    
    # Vérification locale (feedback immédiat)
    if not GridManager.is_cell_reachable(selected_unit, target_cell):
        print("❌ Case non atteignable")
        return
    
    # APPEL RPC → Synchronisé sur tous les clients
    rpc_request_move.rpc(selected_unit.get_path(), target_cell.x, target_cell.y)
    
    _deselect_unit()


func _execute_attack_action(target_unit: UnitBase) -> void:
    if not selected_unit:
        return
    
    # Vérification locale
    if not _can_attack(selected_unit, target_unit):
        print("❌ Attaque impossible")
        return
    
    # APPEL RPC → Synchronisé
    rpc_request_attack.rpc(selected_unit.get_path(), target_unit.get_path())
    
    _deselect_unit()


func _execute_ability_action(ability_id: String, target_cell: Vector2i) -> void:
    if not selected_unit:
        return
    
    # APPEL RPC → Synchronisé
    rpc_request_ability.rpc(selected_unit.get_path(), ability_id, target_cell.x, target_cell.y)
    
    _deselect_unit()


func _on_end_turn_pressed() -> void:
    # APPEL RPC → Tout le monde change de tour en même temps
    rpc_end_turn.rpc()
```

---

### 🎯 Tâche 4 : Synchronisation du Tour (turn_manager.gd)

**Fichier :** `res://autoload/turn_manager.gd`

**Le TurnManager doit réagir aux signaux, pas les générer directement :**
```gdscript
# ═══════════════════════════════════════════════════════════════════════════
# IMPORTANT: Le changement de tour est déclenché par RPC depuis PlayerController
# Pas d'appel direct à next_turn() depuis l'UI ou autre
# ═══════════════════════════════════════════════════════════════════════════

func _ready():
    # Écouter le signal de fin de tour (émis par RPC)
    EventBus.end_turn_requested.connect(_on_end_turn_requested)

func _on_end_turn_requested():
    # Cette fonction est appelée sur TOUS les clients via RPC
    _advance_turn()

func _advance_turn():
    # Réinitialiser PA des unités de l'équipe qui vient de jouer
    _reset_team_ap(current_team)
    
    # Passer à l'équipe suivante
    current_team = (current_team + 1) % total_teams
    turn_count += 1
    
    print("🔄 Tour %d - Équipe %d" % [turn_count, current_team])
    
    # Émettre le signal pour l'UI et autres systèmes
    EventBus.turn_changed.emit(current_team, turn_count)
    
    # Vérifier conditions de victoire
    _check_victory_conditions()

func _reset_team_ap(team_index: int):
    for unit in get_tree().get_nodes_in_group("units"):
        if unit.team_index == team_index:
            unit.reset_ap()
```

---

### 🎯 Tâche 5 : Synchronisation des Dégâts/Morts (combat_system.gd)

**Si tu as un système de combat séparé, il doit aussi être synchronisé :**
```gdscript
# res://scripts/systems/combat_system.gd

## Applique les dégâts - DOIT être appelé via RPC pour être synchronisé
func apply_damage(target: UnitBase, amount: int, source: UnitBase = null) -> void:
    target.current_hp -= amount
    
    print("💥 %s subit %d dégâts (HP: %d/%d)" % [
        target.name, amount, target.current_hp, target.stats.max_hp
    ])
    
    EventBus.unit_damaged.emit(target, amount, source)
    
    if target.current_hp <= 0:
        _handle_death(target, source)

func _handle_death(unit: UnitBase, killer: UnitBase = null) -> void:
    print("💀 %s est mort!" % unit.name)
    
    EventBus.unit_died.emit(unit, killer)
    
    # Vérifier si c'était le Leader
    if unit.is_leader:
        var losing_team = unit.team_index
        var winning_team = 1 - losing_team  # Assuming 2 teams
        EventBus.game_over.emit(winning_team, "leader_killed")
```

---

### 🧪 Tests et Debugging

**Configuration Debug (2 instances) :**
1. Dans Godot : `Debug → Run Multiple Instances → 2`
2. Ou lancer 2 exports séparément

**Checklist de test :**
```
[ ] Instance 1: Cliquer "Host"
    → Affiche "En attente..."
    → my_player_id = 1
    → player_teams = {1: 0}

[ ] Instance 2: Entrer 127.0.0.1, cliquer "Join"
    → Les deux passent à la scène de jeu
    → my_player_id = [unique_id]
    → player_teams = {1: 0, [id]: 1}

[ ] Host bouge une unité Team 0
    → L'unité bouge sur LES DEUX écrans

[ ] Client essaie de bouger une unité Team 0
    → RIEN ne se passe (pas son tour)

[ ] Client bouge une unité Team 1 (après End Turn du Host)
    → L'unité bouge sur LES DEUX écrans

[ ] Host essaie de bouger pendant le tour du Client
    → RIEN ne se passe

[ ] Attaque → Dégâts visibles sur les deux écrans
[ ] Mort d'unité → Disparaît sur les deux écrans
[ ] Leader meurt → Game Over sur les deux écrans
```

**Debug prints utiles :**
```gdscript
# Dans NetworkManager
print("🔍 my_player_id: %d" % my_player_id)
print("🔍 player_teams: %s" % str(player_teams))
print("🔍 is_my_turn(%d): %s" % [TurnManager.current_team, is_my_turn(TurnManager.current_team)])

# Dans PlayerController (début de _unhandled_input)
print("🎮 Input reçu - Mon tour: %s" % NetworkManager.is_my_turn(TurnManager.current_team))
```

---

### ⚠️ Pièges Courants

| Piège | Solution |
|-------|----------|
| `rpc()` sans `.rpc()` | Toujours appeler `fonction.rpc()` pas `fonction()` |
| NodePath invalide après changement de scène | Utiliser des IDs uniques ou recalculer les paths |
| Désync des HP | Tous les dégâts doivent passer par RPC |
| Turn_manager appelé 2 fois | Un seul point d'entrée via EventBus |
| Client qui triche | TOUJOURS valider côté serveur (Host) |
| Vector2i en paramètre RPC | Godot ne sérialise pas Vector2i → Utiliser int x, int y |

---

### 📁 Résumé des Fichiers à Modifier

| Fichier | Modifications |
|---------|---------------|
| `network_manager.gd` | + `my_player_id`, `player_teams`, `is_my_turn()`, `is_my_unit()`, `_sync_teams()` |
| `player_controller.gd` | + Guard clauses, + 4 fonctions RPC, modifier les appels d'action |
| `turn_manager.gd` | Écouter `end_turn_requested`, synchroniser `_advance_turn()` |
| `combat_system.gd` | S'assurer que `apply_damage` est appelé via la chaîne RPC |

---

### 🔄 Flux de Données Réseau (Complet)

```
[CLIENT clique "Move"]
       │
       ▼
[Guard: is_my_turn?] ──NO──► return
       │
       │ YES
       ▼
[Guard: is_my_unit?] ──NO──► return
       │
       │ YES
       ▼
[rpc_request_move.rpc(...)]
       │
       ├──────────────────────────────────┐
       │                                  │
       ▼                                  ▼
[CLIENT exécute]                    [HOST reçoit]
(call_local)                              │
       │                                  ▼
       │                          [Validation serveur]
       │                          - sender owns unit?
       │                          - correct turn?
       │                                  │
       │                           ┌──────┴──────┐
       │                           │ VALID       │ INVALID
       │                           ▼             ▼
       │                    [Exécute +         [return]
       │                     log/broadcast]
       │                           │
       ▼                           ▼
[EventBus.move_requested]   [EventBus.move_requested]
       │                           │
       ▼                           ▼
[Unit.move_to()]            [Unit.move_to()]
       │                           │
       ▼                           ▼
[Visuel CLIENT]             [Visuel HOST]
    (synchronisé!)              (synchronisé!)
```

---

## 🎭 Peuples et Compétences (PROCHAINE ÉTAPE)

### Peuples Prévus
| Peuple | Thème | Compétence Passive | Compétence Active |
|--------|-------|-------------------|-------------------|
| 🔥 Feu | Agression | Brûlure (DoT) | Boule de feu (AoE) |
| ⏰ Temps | Contrôle | Ralentissement | Rembobinage (heal) |
| 🌿 Nature | Terrain | Regen en forêt | Mur de ronces |
| ⚡ Foudre | Mobilité | Vitesse +1 | Téléportation |
| 🪨 Terre | Défense | Armure +2 | Mur de pierre |

---

## 🌐 Backend Multijoueur Online (À IMPLÉMENTER)

> **Référence :** Architecture inspirée du backend LIM Truco Game Server (REST + WebSocket).
> **Objectif :** Serveur Python FastAPI autoritaire pour matchs PvP online.

---

### 🏗️ Architecture Cible

```
server/
├── app/
│   ├── __init__.py              # Package init, version
│   ├── main.py                  # FastAPI app, routes, lifespan
│   ├── models.py                # Pydantic models (request/response)
│   ├── game_logic.py            # Champions, abilities, damage calc, victory
│   ├── actions.py               # Action processors (move, attack, ability)
│   └── websocket_manager.py     # Connection pool, broadcast
├── requirements.txt             # fastapi, uvicorn, pydantic, websockets
├── Dockerfile                   # Python 3.11-slim
├── docker-compose.yml           # Service + optional Redis
└── README.md                    # Setup instructions
```

---

### 🔐 Authentification

**Méthode :** API Key dans header `X-API-Key`

```http
# HTTP Requests
X-API-Key: your-api-key-here

# WebSocket
ws://server/ws/matches/{matchId}?slot=0&api_key=your-api-key
```

**Validation :**
```python
API_KEYS = {"dev-key-12345", "godot-client-key"}

def verify_api_key(x_api_key: str = Header(None)):
    if x_api_key not in API_KEYS:
        raise HTTPException(401, {"errorCode": "UNAUTHORIZED"})
```

---

### 📡 API Endpoints Détaillés

#### Health (No Auth)

```http
GET /status/health
Response 200:
{
  "status": "ok",
  "uptimeMs": 123456,
  "activeMatches": 5,
  "connectedPlayers": 10
}
```

#### Debug (Auth Required)

```http
GET /status/debug
Response 200:
{
  "lobbies": {...},
  "matches": {...},
  "totalLobbies": 5,
  "totalMatches": 3
}
```

---

#### Lobby Management

**POST /lobbies** - Créer un lobby
```json
// Request
{
  "displayName": "Alice",
  "team": "fire",
  "champions": ["mage_fire", "warrior_fire", "assassin_fire"]
}

// Response 201
{
  "lobbyId": "uuid-here",
  "playerSlot": 0,
  "state": "waiting",
  "inviteCode": "ABCD1234"
}
```

**GET /lobbies/{lobbyId}** - État du lobby
```json
// Response 200
{
  "lobbyId": "uuid",
  "players": [
    {"slot": 0, "name": "Alice", "team": "fire", "champions": [...], "ready": true},
    {"slot": 1, "name": "Bob", "team": "earth", "champions": [...], "ready": false}
  ],
  "state": "waiting|ready|started",
  "matchId": null,
  "inviteCode": "ABCD1234"
}
```

**POST /lobbies/{lobbyId}/join** - Rejoindre
```json
// Request
{
  "displayName": "Bob",
  "team": "earth",
  "champions": ["tank_earth", "guardian_earth", "mage_earth"]
}
```

**POST /lobbies/{lobbyId}/ready** - Toggle ready
```json
// Request
{"slot": 0, "ready": true}
```

**POST /lobbies/{lobbyId}/start** - Démarrer (tous ready)
```json
// Response 201
{"matchId": "match-uuid"}
```

**DELETE /lobbies/{lobbyId}** - Supprimer lobby
```
Response 204 No Content
```

---

#### Match Management

**POST /matches** - Créer match direct (bypass lobby)
```json
// Request
{
  "playerA": {
    "name": "Alice",
    "team": "fire",
    "champions": ["mage_fire", "warrior_fire"]
  },
  "playerB": {
    "name": "Bob",
    "team": "earth",
    "champions": ["tank_earth", "guardian_earth"]
  },
  "gridSize": {"width": 10, "height": 10},
  "obstacles": [[4, 4], [4, 5], [4, 6]]
}

// Response 201
{
  "matchId": "uuid",
  "initialState": {
    "matchId": "uuid",
    "status": "active",
    "turn": 1,
    "currentTeam": 0,
    "grid": {"width": 10, "height": 10, "obstacles": [...]},
    "units": [
      {
        "id": "unit-abc123",
        "ownerId": 0,
        "championType": "mage_fire",
        "position": [1, 2],
        "currentHp": 8,
        "maxHp": 8,
        "currentAp": 3,
        "maxAp": 3,
        "moveRange": 3,
        "attackRange": 2,
        "attack": 4,
        "defense": 0,
        "isLeader": true,
        "abilities": ["fireball"],
        "status": []
      }
    ],
    "scores": [0, 0]
  }
}
```

**GET /matches/{matchId}** - État actuel
```json
// Response 200
{
  "matchId": "uuid",
  "status": "active|finished|abandoned",
  "turn": 5,
  "currentTeam": 1,
  "grid": {...},
  "units": [...],
  "scores": [2, 1],
  "lastAction": {
    "type": "move",
    "actorSlot": 0,
    "unitId": "unit-1",
    "timestamp": "2026-01-17T10:30:00Z"
  }
}
```

**POST /matches/{matchId}/forfeit** - Abandonner
```json
// Request
{"actorSlot": 1}

// Response 200
{"winnerSlot": 0, "reason": "forfeit"}
```

**GET /matches/{matchId}/log** - Journal des actions
```json
// Response 200
{
  "events": [
    {"timestamp": "...", "type": "matchStarted"},
    {"timestamp": "...", "type": "unitMoved", "payload": {...}},
    {"timestamp": "...", "type": "abilityUsed", "payload": {...}}
  ]
}
```

---

#### Game Actions

**POST /matches/{matchId}/actions**

| Action | Type | Payload | Coût AP |
|--------|------|---------|---------|
| Déplacer | `move` | `{unitId, to: [x,y]}` | 1/case (Manhattan) |
| Attaquer | `attack` | `{unitId, targetId}` | 1 |
| Compétence | `ability` | `{unitId, abilityId, target: [x,y]}` | Variable |
| Fin tour | `endTurn` | `{}` | 0 |
| Passer | `skip` | `{unitId}` | 0 |

**Move Action:**
```json
{
  "actorSlot": 0,
  "type": "move",
  "payload": {"unitId": "unit-1", "to": [3, 4]}
}

// Response 202
{
  "applied": true,
  "newState": {...},
  "events": [
    {"type": "unitMoved", "unitId": "unit-1", "from": [1,1], "to": [3,4], "path": [[2,1],[2,2],[3,3],[3,4]]},
    {"type": "apConsumed", "unitId": "unit-1", "amount": 4, "remaining": 0}
  ]
}
```

**Attack Action:**
```json
{
  "actorSlot": 0,
  "type": "attack",
  "payload": {"unitId": "unit-1", "targetId": "unit-5"}
}

// Response 202
{
  "applied": true,
  "events": [
    {"type": "unitAttacked", "attackerId": "unit-1", "targetId": "unit-5", "baseDamage": 4, "actualDamage": 3, "shieldAbsorbed": 0},
    {"type": "apConsumed", "unitId": "unit-1", "amount": 1},
    {"type": "unitDied", "unitId": "unit-5", "isLeader": true, "killedBy": "unit-1"},
    {"type": "gameOver", "winner": 0, "reason": "leaderKilled", "finalScores": [3, 0]}
  ]
}
```

**Ability Action:**
```json
{
  "actorSlot": 0,
  "type": "ability",
  "payload": {"unitId": "unit-1", "abilityId": "fireball", "target": [5, 5]}
}

// Response 202
{
  "applied": true,
  "events": [
    {
      "type": "abilityUsed",
      "casterId": "unit-1",
      "abilityId": "fireball",
      "targetCells": [[5,5], [4,5], [6,5], [5,4], [5,6]],
      "affectedUnits": [
        {"unitId": "unit-5", "damage": 4, "newHp": 6},
        {"unitId": "unit-6", "damage": 4, "newHp": 0}
      ]
    },
    {"type": "unitDied", "unitId": "unit-6", "isLeader": false}
  ]
}
```

**End Turn Action:**
```json
{
  "actorSlot": 0,
  "type": "endTurn",
  "payload": {}
}

// Response 202
{
  "events": [
    {"type": "turnEnded", "team": 0},
    {"type": "statusDamage", "unitId": "unit-3", "status": "burning", "damage": 2},
    {"type": "statusExpired", "unitId": "unit-4", "status": "slowed"},
    {"type": "turnStarted", "turn": 6, "team": 1, "unitsReset": ["unit-5", "unit-6"]}
  ]
}
```

---

### 🔌 WebSocket Protocol

**Connection:**
```
ws://localhost:8080/ws/matches/{matchId}?slot=0&api_key=your-key
```

**Close Codes:**
| Code | Raison |
|------|--------|
| 4000 | Match invalide |
| 4001 | Slot invalide |
| 4002 | Trop de connexions |
| 4003 | API key invalide |
| 4004 | Match terminé |

**Server → Client Messages:**

```json
// State Update (full state on connect + after each action)
{"kind": "stateUpdate", "payload": {/* full match state */}}

// Unit Event
{"kind": "unitEvent", "payload": {
  "type": "unitMoved|unitAttacked|unitDied|unitTeleported",
  ...
}}

// Ability Event
{"kind": "abilityEvent", "payload": {
  "abilityId": "fireball",
  "casterId": "unit-1",
  "targetCells": [[5,5], ...],
  "affectedUnits": [...],
  "damage": 4
}}

// Turn Event
{"kind": "turnEvent", "payload": {
  "type": "turnStart|turnEnd",
  "turn": 6,
  "activeTeam": 0,
  "unitsReset": ["unit-1", "unit-2"]
}}

// Status Event
{"kind": "statusEvent", "payload": {
  "type": "statusApplied|statusExpired|statusDamage",
  "unitId": "unit-3",
  "status": "burning",
  "damage": 2
}}

// Game Over
{"kind": "gameOver", "payload": {
  "winner": 0,
  "reason": "leaderKilled|teamWipe|forfeit|timeout",
  "finalScores": [5, 2]
}}
```

---

### 🎮 Game Logic

#### Champions (par Team)

**🔥 Fire Team:**
| Champion | HP | AP | Move | Range | ATK | DEF | Abilities |
|----------|----|----|------|-------|-----|-----|-----------|
| mage_fire | 8 | 3 | 3 | 2 | 4 | 0 | fireball |
| warrior_fire | 12 | 3 | 4 | 1 | 3 | 1 | charge |
| assassin_fire | 6 | 4 | 5 | 1 | 5 | 0 | ignite |

**🪨 Earth Team:**
| Champion | HP | AP | Move | Range | ATK | DEF | Abilities |
|----------|----|----|------|-------|-----|-----|-----------|
| tank_earth | 16 | 2 | 2 | 1 | 2 | 3 | wall |
| guardian_earth | 12 | 3 | 3 | 1 | 2 | 2 | shield |
| mage_earth | 8 | 3 | 3 | 3 | 3 | 1 | earthquake |

**⏰ Time Team:**
| Champion | HP | AP | Move | Range | ATK | DEF | Abilities |
|----------|----|----|------|-------|-----|-----|-----------|
| oracle_time | 8 | 4 | 3 | 2 | 2 | 0 | rewind |
| chrono_time | 10 | 3 | 4 | 1 | 3 | 1 | slow |
| sage_time | 6 | 3 | 3 | 3 | 3 | 0 | hasten |

**⚡ Lightning Team:**
| Champion | HP | AP | Move | Range | ATK | DEF | Abilities |
|----------|----|----|------|-------|-----|-----|-----------|
| assassin_lightning | 7 | 4 | 5 | 1 | 4 | 0 | teleport |
| storm_lightning | 9 | 3 | 4 | 2 | 3 | 1 | chain_lightning |
| flash_lightning | 8 | 4 | 6 | 1 | 2 | 0 | dash |

**🌿 Nature Team:**
| Champion | HP | AP | Move | Range | ATK | DEF | Abilities |
|----------|----|----|------|-------|-----|-----|-----------|
| druid_nature | 10 | 3 | 3 | 2 | 2 | 1 | heal |
| treant_nature | 14 | 2 | 2 | 1 | 3 | 2 | root |
| ranger_nature | 8 | 3 | 4 | 4 | 3 | 0 | poison_arrow |

#### Abilities

| Ability | Cost | Range | Pattern | Effect |
|---------|------|-------|---------|--------|
| fireball | 2 | 5 | cross (5 cells) | 4 damage AOE |
| charge | 2 | 3 | line | Dash + 3 damage |
| ignite | 1 | 1 | single | 2 dmg + burning (2/turn) |
| wall | 2 | 4 | line_3 | Create 3 obstacles (3 turns) |
| shield | 1 | 2 | single | Absorb 4 damage |
| earthquake | 3 | 4 | diamond_5 | 3 damage large AOE |
| rewind | 3 | 3 | single | Teleport ally back + heal 3 |
| slow | 1 | 3 | single | -1 Move (2 turns) |
| hasten | 1 | 2 | single | +1 AP this turn |
| teleport | 2 | 6 | point | Instant teleport |
| chain_lightning | 2 | 3 | chain_3 | 2 dmg bounces to 3 enemies |
| dash | 1 | 4 | point | Free teleport move |
| heal | 2 | 3 | cross | Heal allies 3 HP |
| root | 2 | 3 | single | 1 dmg + stun (skip turn) |
| poison_arrow | 1 | 5 | single | 2 dmg + poison (1/turn, 3 turns) |

#### Status Effects

| Status | Effect | Duration |
|--------|--------|----------|
| burning | -2 HP/turn | Until removed |
| poisoned | -1 HP/turn | 3 turns |
| slowed | -1 Move Range | 2 turns |
| stunned | Skip next turn | 1 turn |
| shielded | Absorb X damage | Until depleted |
| hastened | +1 AP | This turn only |

#### Victory Conditions

| Condition | Trigger |
|-----------|---------|
| Leader Kill | Enemy Leader HP ≤ 0 |
| Team Wipe | All enemy units dead |
| Forfeit | Opponent surrenders |
| Timeout | Most total HP wins |

#### Damage Calculation

```python
actual_damage = max(1, attacker.attack - target.defense)

# Shield check first
if target.has_status("shielded"):
    shield = target.get_shield_amount()
    if shield >= actual_damage:
        shield -= actual_damage
        actual_damage = 0
    else:
        actual_damage -= shield
        remove_shield()

target.currentHp -= actual_damage
```

---

### ❌ Error Handling

**Error Response Format:**
```json
{
  "errorCode": "ILLEGAL_MOVE",
  "message": "Unit does not have enough AP",
  "details": {"required": 3, "available": 1}
}
```

**Error Codes:**
| Code | HTTP | Description |
|------|------|-------------|
| UNAUTHORIZED | 401 | API key invalide |
| BAD_REQUEST | 400 | Payload malformé |
| NOT_FOUND | 404 | Ressource introuvable |
| CONFLICT | 409 | État invalide (lobby plein, pas ton tour) |
| GONE | 410 | Ressource expirée |
| ILLEGAL_MOVE | 422 | Règle violée |
| RATE_LIMITED | 429 | Trop de requêtes |
| SERVER_ERROR | 500 | Erreur interne |

**Illegal Move Reasons:**
| Reason | Description |
|--------|-------------|
| NOT_YOUR_TURN | Ce n'est pas ton tour |
| NOT_YOUR_UNIT | Unité adverse |
| INSUFFICIENT_AP | PA insuffisants |
| INVALID_TARGET | Cible invalide |
| OUT_OF_RANGE | Hors de portée |
| PATH_BLOCKED | Chemin bloqué |
| CELL_OCCUPIED | Case occupée |
| ON_COOLDOWN | Compétence en cooldown |
| UNIT_DEAD | Unité morte |
| UNIT_STUNNED | Unité étourdie |

---

### 🎯 Validation Rules (Server-Side)

**Move Validation:**
```
1. match.status == "active"
2. match.currentTeam == actorSlot
3. unit exists && unit.ownerId == actorSlot
4. unit.currentHp > 0
5. unit not stunned
6. distance(from, to) <= unit.moveRange
7. unit.currentAp >= distance
8. path exists (BFS, no obstacles/units blocking)
9. destination not occupied
```

**Attack Validation:**
```
1. All move checks (1-5)
2. unit.currentAp >= 1
3. target exists && target.ownerId != actorSlot
4. target.currentHp > 0
5. distance(unit, target) <= unit.attackRange
```

**Ability Validation:**
```
1. All move checks (1-5)
2. ability in unit.abilities
3. ability not on cooldown
4. unit.currentAp >= ability.cost
5. distance(unit, target) <= ability.range
6. (for teleport) destination not blocked
```

---

### 🖥️ Godot Client Integration

**Fichiers à créer dans Godot:**
```
res://scripts/network/
├── api_client.gd        # HTTP client (lobbies, matches, actions)
├── ws_client.gd         # WebSocket real-time
└── network_manager.gd   # High-level wrapper + signals
```

**Signals NetworkManager:**
```gdscript
signal lobby_created(lobby_id, slot)
signal lobby_joined(lobby_id, slot)
signal lobby_updated(lobby)
signal match_started(match_id, initial_state)
signal match_state_changed(state)
signal action_result(success, events)
signal turn_changed(turn, active_team)
signal unit_moved(unit_id, from, to)
signal unit_attacked(attacker_id, target_id, damage)
signal unit_died(unit_id, is_leader)
signal ability_used(caster_id, ability_id, cells)
signal game_ended(winner, reason)
signal network_error(message)
```

**Flow Integration:**
```
1. UI: "Create Lobby" → NetworkManager.create_lobby()
2. NetworkManager → HTTP POST /lobbies
3. Response → emit lobby_created
4. UI: Show invite code
5. Opponent joins → HTTP POST /lobbies/{id}/join
6. Both ready → HTTP POST /lobbies/{id}/start
7. Start → Connect WebSocket
8. Game loop: PlayerController.on_move() → NetworkManager.move()
9. Server validates → WebSocket broadcasts state
10. All clients sync
```

---

### 📦 Deployment

**Requirements:**
```txt
fastapi>=0.100.0
uvicorn[standard]>=0.22.0
pydantic>=2.0.0
websockets>=11.0
```

**Docker:**
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY app/ ./app/
EXPOSE 8080
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8080"]
```

**Run:**
```bash
# Dev
uvicorn app.main:app --reload --port 8080

# Prod
uvicorn app.main:app --host 0.0.0.0 --port 8080 --workers 4
```

---

## 📝 Instructions pour Claude Code

### Comportement Attendu
1. **Sois concis** - Réponds directement sans sur-expliquer
2. **Code d'abord** - Génère le code, explique après si demandé
3. **Respecte l'architecture** - UI → EventBus → Controller → Unit
4. **Jamais de spaghetti** - L'UI n'appelle jamais les unités directement

### ⛔ ZONE INTERDITE - NE PAS TOUCHER

```
┌─────────────────────────────────────────────────────────────────┐
│  🚫 CONTENU DU JEU = INTERDIT DE MODIFIER                       │
│  ─────────────────────────────────────────────────────────────  │
│                                                                 │
│  ❌ Peuples (races, factions)                                   │
│  ❌ Champions (noms, designs, lore)                             │
│  ❌ Abilities (compétences, effets, descriptions)               │
│  ❌ Stats (HP, ATK, DEF, vitesse, portée)                       │
│  ❌ Équilibrage (dégâts, coûts, cooldowns)                      │
│                                                                 │
│  💡 Le game design est géré par l'humain, pas par l'IA          │
│  💡 Claude Code = technique UNIQUEMENT                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### 🚨 PRIORITÉ IMMÉDIATE - ORDRE OBLIGATOIRE

**Claude Code DOIT suivre cet ordre exact :**

```
┌─────────────────────────────────────────────────────────────────┐
│  PHASE 1 : 🐛 BUGS CRITIQUES (BLOCKERS)                        │
│  ─────────────────────────────────────────────────────────────  │
│  📄 Lire : AUDIT_FIXES_PROMPT.md                                │
│                                                                 │
│  Bug #1 : Game Over chain brisée                                │
│    → unit_died non connecté dans TurnManager                    │
│    → Mort du Leader = rien ne se passe                          │
│                                                                 │
│  Bug #2 : UI input passthrough                                  │
│    → Clics traversent les panels UI                             │
│    → Actions involontaires sur les unités                       │
│                                                                 │
│  ⚠️ SANS CES FIXES, LE JEU EST INJOUABLE                        │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  PHASE 2 : 🔌 MULTIJOUEUR P2P LOCAL (RPC)                       │
│  ─────────────────────────────────────────────────────────────  │
│  📄 Section : Multiplayer P2P RPC Security (ci-dessus)          │
│                                                                 │
│  → ENet avec Godot High-Level Multiplayer                       │
│  → RPCs "call_local" pour synchronisation                       │
│  → Validation serveur-side (host = autorité)                    │
│  → Fonctionne sur réseau local SANS serveur                     │
│                                                                 │
│  ✅ Permet de tester le jeu à 2 rapidement                      │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  PHASE 3 : 🖥️ BACKEND PYTHON (OPTIONNEL - ONLINE)              │
│  ─────────────────────────────────────────────────────────────  │
│  📄 Section : Backend Multiplayer Online (ci-dessus)            │
│                                                                 │
│  → FastAPI + WebSocket                                          │
│  → Serveur autoritaire pour matchmaking online                  │
│  → Anti-cheat renforcé                                          │
│                                                                 │
│  ⏳ À faire APRÈS que P2P fonctionne                            │
└─────────────────────────────────────────────────────────────────┘
```

### Ordre de Développement Global
```
1. ✅ Moteur (FAIT - ne plus toucher)
2. 🚨 BUGS CRITIQUES (AUDIT_FIXES_PROMPT.md) ← FAIRE EN PREMIER
3. 🔌 Multiplayer P2P RPC (réseau local)
4. ⏳ Peuples (Data .tres)
5. ⏳ Compétences uniques
6. ⏳ IA basique (si PvE)
7. ⏳ Backend Python Online (optionnel)
8. ⏳ Intégration Godot ↔ Backend
```

### Format de Réponse Préféré
```
✅ Fait : [description courte]
📄 Fichier modifié : [chemin]
⚠️ À noter : [si pertinent]
```

---

## 🚀 Commande de Lancement

```bash
# Dans l'éditeur Godot : F5
```

**Recherche dans le code :** `## FIX:` pour voir les corrections appliquées.
