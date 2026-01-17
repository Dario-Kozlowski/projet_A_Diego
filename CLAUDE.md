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
