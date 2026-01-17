# 🔬 RAPPORT D'AUDIT "RAYON X" - CORRECTIFS À APPLIQUER

> **Date :** Janvier 2026  
> **Statut :** Bugs critiques identifiés  
> **Action requise :** Appliquer les correctifs ci-dessous

---

## 📋 Résumé de l'Audit

| Mécanique | Trace Littérale | Verdict |
|-----------|-----------------|---------|
| Mort Unité / HealthBar | `unit_base._die()` → `queue_free()`. HealthBar dans `_process()` appelle `is_instance_valid(unit)` → si false, `queue_free()` sur elle-même | ✅ OK |
| Trigger Game Over | `unit_base._die()` → `EventBus.unit_died.emit()` → ⚠️ **NON CONNECTÉ** dans TurnManager actuel. `_check_win_condition()` existe mais appelé où ? | ❌ BUG |
| Blocage Input UI | `PlayerController._unhandled_input()` utilise `_unhandled_input` (pas `_input`). Les clics passent à travers l'UI si le panel ne les consomme pas. Pas de `get_viewport().set_input_as_handled()` | ⚠️ RISQUE |
| Nettoyage Overlay | `PlayerController._cancel_action()` → `EventBus.cells_cleared.emit()` → `GridOverlay._on_cells_cleared()` → `clear()` | ✅ OK |

---

## ❌ BUG CRITIQUE #1 : Chaîne de Victoire Cassée

### Diagnostic

**Trace détaillée :**
```
1. unit_base.gd:231 → émet EventBus.unit_died.emit(self)
2. ??? → PERSONNE n'écoute ce signal dans TurnManager !
3. _check_win_condition() existe mais n'est appelé que dans _advance_to_next_team()
4. Résultat: Le Leader peut mourir SANS déclencher Game Over
```

**Conséquence :** Partie bloquée après mort du Leader. Le jeu continue alors qu'il devrait être terminé.

### Correctif à Appliquer

**Fichier :** `res://autoload/turn_manager.gd`

**Étape 1 : Connecter le signal `unit_died`**

Localiser la fonction `_ready()` ou `start_game()` et ajouter :

```gdscript
func _ready():
    # ... code existant ...
    
    # ═══════════════════════════════════════════════════════════════
    # FIX: Connecter le signal de mort d'unité pour trigger Game Over
    # ═══════════════════════════════════════════════════════════════
    if not EventBus.unit_died.is_connected(_on_unit_died):
        EventBus.unit_died.connect(_on_unit_died)
```

**Étape 2 : Créer la fonction de callback**

Ajouter cette fonction dans `turn_manager.gd` :

```gdscript
## Appelé quand une unité meurt - vérifie les conditions de victoire
## @param unit: L'unité qui vient de mourir
func _on_unit_died(unit: UnitBase) -> void:
    print("💀 TurnManager: Unité morte détectée - %s (Team %d, Leader: %s)" % [
        unit.name, unit.team_index, unit.is_leader
    ])
    
    # ═══════════════════════════════════════════════════════════════
    # Retirer l'unité de sa team (si tracking par liste)
    # ═══════════════════════════════════════════════════════════════
    if _teams.size() > unit.team_index:
        var team_units = _teams[unit.team_index]
        if unit in team_units:
            team_units.erase(unit)
            print("   → Retirée de Team %d (%d unités restantes)" % [
                unit.team_index, team_units.size()
            ])
    
    # ═══════════════════════════════════════════════════════════════
    # Vérification IMMÉDIATE des conditions de victoire
    # ═══════════════════════════════════════════════════════════════
    var winner: int = _check_win_condition()
    
    if winner != -1:
        print("🏆 VICTOIRE détectée! Équipe %d gagne!" % winner)
        _trigger_game_over(winner)
```

**Étape 3 : Créer/Modifier la fonction `_trigger_game_over()`**

```gdscript
## Déclenche la fin de partie
## @param winning_team: L'index de l'équipe gagnante
func _trigger_game_over(winning_team: int) -> void:
    if is_game_over:
        return  # Éviter double trigger
    
    is_game_over = true
    current_phase = Phase.GAME_OVER  # Si tu utilises un enum Phase
    
    print("═══════════════════════════════════════")
    print("       🎮 GAME OVER - Équipe %d gagne!" % winning_team)
    print("═══════════════════════════════════════")
    
    # Émettre le signal pour l'UI
    EventBus.game_ended.emit(winning_team)
```

**Étape 4 : Vérifier/Corriger `_check_win_condition()`**

```gdscript
## Vérifie si une équipe a gagné
## @return: Index de l'équipe gagnante, ou -1 si partie continue
func _check_win_condition() -> int:
    # ═══════════════════════════════════════════════════════════════
    # Méthode 1: Vérifier les Leaders (PRIORITAIRE)
    # ═══════════════════════════════════════════════════════════════
    var leaders_alive: Array[bool] = [false, false]
    
    for unit in get_tree().get_nodes_in_group("units"):
        if unit is UnitBase and unit.is_leader and unit.current_hp > 0:
            if unit.team_index < leaders_alive.size():
                leaders_alive[unit.team_index] = true
    
    # Si Leader Team 0 mort → Team 1 gagne
    if not leaders_alive[0] and leaders_alive[1]:
        return 1
    
    # Si Leader Team 1 mort → Team 0 gagne
    if not leaders_alive[1] and leaders_alive[0]:
        return 0
    
    # ═══════════════════════════════════════════════════════════════
    # Méthode 2: Team Wipe (toutes les unités mortes)
    # ═══════════════════════════════════════════════════════════════
    var units_alive: Array[int] = [0, 0]
    
    for unit in get_tree().get_nodes_in_group("units"):
        if unit is UnitBase and unit.current_hp > 0:
            if unit.team_index < units_alive.size():
                units_alive[unit.team_index] += 1
    
    # Team 0 éliminée → Team 1 gagne
    if units_alive[0] == 0 and units_alive[1] > 0:
        return 1
    
    # Team 1 éliminée → Team 0 gagne
    if units_alive[1] == 0 and units_alive[0] > 0:
        return 0
    
    # Partie continue
    return -1
```

---

## ⚠️ RISQUE #2 : Input Traverse l'UI

### Diagnostic

**Problème :** `_unhandled_input()` reçoit les clics même quand ils sont sur le panel `ActionUI`. Le joueur peut accidentellement cliquer sur une case en voulant cliquer sur un bouton.

**Trace :**
```
1. Joueur clique sur bouton "Attack" dans ActionUI
2. Le bouton reçoit le clic ET _unhandled_input aussi
3. Si la souris est au-dessus d'une case, ça peut déclencher une action non voulue
```

### Correctif à Appliquer

**Fichier :** `res://scripts/controllers/player_controller.gd`

**Option A : Guard Clause basé sur la souris au-dessus de l'UI**

```gdscript
func _unhandled_input(event: InputEvent) -> void:
    # ═══════════════════════════════════════════════════════════════
    # FIX: Ignorer les inputs si la souris est sur un élément UI
    # ═══════════════════════════════════════════════════════════════
    if event is InputEventMouseButton:
        # Vérifier si un Control a le focus ou est sous la souris
        var focused = get_viewport().gui_get_focus_owner()
        if focused != null:
            return  # Un élément UI a le focus
        
        # Alternative: vérifier si on hover un Control
        var mouse_pos = get_viewport().get_mouse_position()
        var hovered = _get_control_at_position(mouse_pos)
        if hovered != null:
            return  # Souris sur un élément UI
    
    # ... reste du code ...

## Helper pour trouver un Control sous la souris
func _get_control_at_position(pos: Vector2) -> Control:
    # Parcourir les CanvasLayers/UI pour trouver un Control
    var ui_root = get_node_or_null("/root/Main/UI")  # Adapter le chemin
    if ui_root:
        for child in ui_root.get_children():
            if child is Control and child.visible:
                if child.get_global_rect().has_point(pos):
                    return child
    return null
```

**Option B : Utiliser un flag géré par l'UI**

```gdscript
# Dans PlayerController
var _is_ui_blocking: bool = false

func _ready():
    # ... code existant ...
    
    # Connecter les signaux d'UI
    EventBus.ui_opened.connect(func(): _is_ui_blocking = true)
    EventBus.ui_closed.connect(func(): _is_ui_blocking = false)

func _unhandled_input(event: InputEvent) -> void:
    # ═══════════════════════════════════════════════════════════════
    # FIX: Ignorer les inputs si l'UI est ouverte
    # ═══════════════════════════════════════════════════════════════
    if _is_ui_blocking:
        return
    
    # ... reste du code ...
```

**Option C : Dans ActionUI, consommer l'input**

```gdscript
# Dans action_ui.gd
func _ready():
    # Assurer que le panel capture les clics
    mouse_filter = Control.MOUSE_FILTER_STOP
    
    # Propager à tous les enfants
    for child in get_children():
        if child is Control:
            child.mouse_filter = Control.MOUSE_FILTER_STOP
```

**Option D : set_input_as_handled() après action UI**

```gdscript
# Dans action_ui.gd - sur chaque bouton
func _on_move_button_pressed():
    EventBus.action_selected.emit("move")
    get_viewport().set_input_as_handled()  # Empêche propagation

func _on_attack_button_pressed():
    EventBus.action_selected.emit("attack")
    get_viewport().set_input_as_handled()
```

---

## 📁 Résumé des Fichiers à Modifier

| Fichier | Modification | Priorité |
|---------|--------------|----------|
| `turn_manager.gd` | + Connecter `unit_died`, + `_on_unit_died()`, + Fix `_check_win_condition()` | 🔴 CRITIQUE |
| `player_controller.gd` | + Guard clause UI blocking | 🟡 IMPORTANT |
| `action_ui.gd` | + `mouse_filter = STOP` ou `set_input_as_handled()` | 🟡 IMPORTANT |
| `event_bus.gd` | + Signaux `ui_opened`, `ui_closed` (si Option B) | 🟢 OPTIONNEL |

---

## ✅ Checklist de Validation Post-Fix

### Test Bug #1 (Game Over)
```
[ ] Lancer une partie
[ ] Attaquer le Leader ennemi jusqu'à 0 HP
[ ] Vérifier que _on_unit_died() est appelé (voir console)
[ ] Vérifier que _check_win_condition() retourne le bon winner
[ ] Vérifier que EventBus.game_ended est émis
[ ] Vérifier que l'UI Game Over s'affiche
[ ] Vérifier que les inputs sont bloqués après Game Over
```

### Test Bug #2 (UI Input)
```
[ ] Sélectionner une unité (ActionUI s'affiche)
[ ] Cliquer sur le bouton "Move" 
[ ] Vérifier que SEUL le bouton réagit (pas de mouvement accidentel)
[ ] Cliquer en dehors de l'UI
[ ] Vérifier que ça désélectionne OU ignore selon le contexte
```

---

## 🔧 Commandes Debug Utiles

Ajouter temporairement pour tracer le flow :

```gdscript
# Dans unit_base.gd - _die()
func _die():
    print("💀 [UNIT] %s._die() appelé" % name)
    print("   → Émission EventBus.unit_died")
    EventBus.unit_died.emit(self)
    queue_free()

# Dans turn_manager.gd - _on_unit_died()
func _on_unit_died(unit):
    print("📡 [TURN_MANAGER] _on_unit_died reçu pour %s" % unit.name)
    # ...

# Dans turn_manager.gd - _check_win_condition()
func _check_win_condition() -> int:
    print("🔍 [TURN_MANAGER] _check_win_condition() appelé")
    # ... logic ...
    print("   → Résultat: %d" % result)
    return result
```

---

## 📝 Instructions pour Claude Code

### Comportement Attendu
1. **Appliquer les correctifs dans l'ordre** : Bug #1 d'abord (critique), puis Bug #2
2. **Ne pas casser le code existant** : Ajouter, ne pas remplacer
3. **Garder les prints de debug** : Utiles pour valider
4. **Tester après chaque modification** : F5 dans Godot

### Format de Réponse
```
✅ Fix appliqué : [description]
📄 Fichier modifié : [chemin]
🧪 Test suggéré : [action à faire pour valider]
```

---

**Document créé pour guider les corrections du jeu tactique Godot 4.5+**
