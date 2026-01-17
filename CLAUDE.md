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

## 📝 Instructions pour Claude Code

### Comportement Attendu
1. **Sois concis** - Réponds directement sans sur-expliquer
2. **Code d'abord** - Génère le code, explique après si demandé
3. **Respecte l'architecture** - UI → EventBus → Controller → Unit
4. **Jamais de spaghetti** - L'UI n'appelle jamais les unités directement

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
