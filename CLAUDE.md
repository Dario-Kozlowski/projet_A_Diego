# 🎮 Tactical Turn-Based Game - Godot 4.5+

## 📋 Contexte du Projet

Jeu tactique au tour par tour en **Godot 4.5+** avec système de grille, gestion des PA (Points d'Action), et combat entre peuples aux pouvoirs uniques.

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
│   └── ability_base.gd      # Classe de base des compétences
├── resources/
│   └── unit_stats.gd        # Resource pour stats (PV, PA, Move)
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

## ✅ Checklist de Validation (4 Tests)

Avant d'ajouter du contenu, vérifie ces mécaniques :

### 1. Test des PA (Points d'Action)
- [ ] Déplacement de 1 case = -1 PA
- [ ] À 0 PA, impossible de bouger
- [ ] Le label affiche correctement "AP: X"

### 2. Test de la Boucle de Tour
- [ ] Fin de tour → passe au joueur suivant
- [ ] PA réinitialisés au début du tour
- [ ] Impossible de contrôler l'unité adverse

### 3. Test du Pathfinding (Murs)
- [ ] L'unité contourne les obstacles
- [ ] Pas de traversée en diagonale si bloqué
- [ ] Distance maximale respectée (stat Move)

### 4. Test des Collisions
- [ ] Impossible de se déplacer sur une case occupée
- [ ] OU déclenche une attaque (si implémenté)
- [ ] Pas de superposition d'unités

---

## 🔧 Ce Qui Reste à Implémenter

### 1. Système de Combat de Base
```gdscript
# Dans unit_base.gd - à ajouter
func attack(target: UnitBase) -> void:
    var damage = stats.attack - target.stats.defense
    target.take_damage(max(1, damage))
    current_ap -= 1
```

### 2. Condition de Victoire
```gdscript
# Dans turn_manager.gd - à compléter
func check_win_condition() -> void:
    var team0_alive = units.filter(func(u): return u.team == 0 and u.is_alive())
    var team1_alive = units.filter(func(u): return u.team == 1 and u.is_alive())
    
    if team0_alive.is_empty():
        emit_signal("game_over", 1)  # Team 1 gagne
    elif team1_alive.is_empty():
        emit_signal("game_over", 0)  # Team 0 gagne
```

### 3. UI de Sélection d'Action
```gdscript
# Afficher les options quand une unité est sélectionnée
# - Déplacer (Move)
# - Attaquer (Attack) 
# - Compétence Spéciale (Ability)
# - Passer (End Turn)
```

---

## 🎭 Peuples et Compétences (À CRÉER)

### Template pour un Peuple
```gdscript
# res://scripts/abilities/ability_[nom].gd
extends AbilityBase
class_name Ability[Nom]

func _init():
    ability_name = "Nom de la compétence"
    ap_cost = 2
    cooldown = 3
    range_min = 1
    range_max = 4

func execute(caster: UnitBase, target) -> void:
    # Logique de la compétence
    emit_signal("execution_finished")
```

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
3. **Respecte l'architecture** - Utilise les fichiers existants
4. **Teste mentalement** - Vérifie les edge cases avant de proposer

### Quand je demande "vérifie que tout marche"
Réponds avec la **checklist des 4 tests** ci-dessus, pas un prompt de 50 lignes.

### Quand je demande d'ajouter un peuple
1. Crée le fichier `ability_[nom].gd`
2. Crée la resource `[nom]_stats.tres`
3. Montre comment l'instancier dans `main.gd`

### Format de Réponse Préféré
```
✅ Fait : [description courte]
📄 Fichier modifié : [chemin]
⚠️ À noter : [si pertinent]
```

---

## 🚀 Commande de Lancement

```bash
# Lancer le projet Godot
godot --path /chemin/vers/projet

# Ou dans l'éditeur : F5
```

**Recherche dans le code :** `## FIX:` pour voir les corrections appliquées.
