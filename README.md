# DrawStudio — Application de dessin Android native

Base solide pour un logiciel de dessin professionnel : Kotlin + Jetpack Compose,
architecture MVVM, rendu Canvas optimisé par rasterisation de calques.

## Ouvrir le projet

1. Ouvrir le dossier `DrawStudio/` avec Android Studio (Koala ou plus récent).
2. Laisser Gradle synchroniser (JDK 17 requis).
3. Lancer sur un appareil/émulateur API 24+.

Aucune clé API ni configuration externe n'est nécessaire : tout est local
(Room pour la persistance des projets).

## Architecture

```
app/src/main/java/com/drawstudio/app/
├── models/        # Données pures et immuables (Layer, DrawPath, Tool, BrushSettings, CanvasTransform...)
├── drawing/        # Moteur de rendu : lissage des traits, rasterisation des calques, undo/redo
├── layers/         # Logique métier des calques (LayerManager) — fonctions pures, testables isolément
├── repository/     # Persistance (Room) + sérialisation JSON + ProjectRepository
├── viewmodel/      # DrawingViewModel : orchestre l'état via StateFlow, seul point d'entrée pour l'UI
├── ui/
│   ├── screens/    # DrawingScreen : composition de l'écran unique
│   ├── components/ # DrawingCanvas, FloatingToolbar, LayerPanel, ColorPickerDialog
│   └── theme/      # Thème sombre Material 3
└── utils/          # Extensions génériques, sérialisation JSON
```

Chaque couche ne dépend que de celle en dessous (`ui` → `viewmodel` → `layers`/`repository` →
`models`), ce qui permet d'ajouter des fonctionnalités (formes, texte, sélection, filtres,
modes de fusion, import/export PSD...) sans toucher au reste.

## Pourquoi c'est fluide

Le point sensible d'un éditeur de dessin est le nombre de traits à redessiner à
chaque frame. Ici :

- **Rasterisation par calque** (`LayerRasterizer`) : chaque calque figé est converti
  en bitmap et mis en cache. Le cache n'est invalidé que lorsque le contenu du
  calque change (nouveau trait ajouté). À chaque frame, on empile des bitmaps —
  opération très bon marché — au lieu de rejouer tous les traits vectoriels.
- **Seul le trait actif est vectoriel** : pendant que l'utilisateur dessine, un
  unique `Path` (le trait en cours) est redessiné par-dessus le bitmap du calque
  actif. Dès que le doigt est levé, ce trait est rasterisé et le cache mis à jour.
- **Lissage + réduction de points** (`PathSmoother`) : filtre les micro-mouvements
  du doigt pour réduire le nombre de points stockés sans perdre en précision
  perçue, ce qui allège aussi la sérialisation.
- **Distinction stricte des gestes** : un seul doigt dessine, deux doigts ou plus
  pilotent uniquement la caméra (pan/zoom/rotation). Impossible de dessiner
  accidentellement pendant un geste de navigation.

## Prochaines étapes suggérées

- Ajout d'un moteur de formes vectorielles (rectangle, ellipse, ligne) réutilisant
  `DrawPath` avec un type `ShapeType` supplémentaire.
- Outil texte : nouveau type de contenu de calque, toujours rendu au-dessus du
  bitmap rasterisé jusqu'à validation, puis fusionné dans le cache.
- Modes de fusion (multiply, screen, overlay...) : à ajouter dans `LayerRasterizer`
  via `BlendMode` de Compose au moment de `drawImage`.
- Pinceaux personnalisés : remplacer le `Paint` simple de `LayerRasterizer` par une
  stratégie de rendu par texture (stamping) piloté par un nouveau modèle `BrushPreset`.
- Export PNG/PSD : ajouter une fonction dans `ProjectRepository` qui compose tous
  les bitmaps de calques en un seul et l'écrit via `MediaStore`.

## Limites connues de cette base

- La rotation de la toile est appliquée mais l'algorithme d'inversion des
  coordonnées écran → toile (`screenToCanvas`) suppose une rotation autour du
  centre logique de la toile ; à valider/affiner avec des tests d'intégration
  UI une fois le rendu visuel en main.
- Le undo/redo fonctionne par snapshot complet de la liste des calques (simple et
  fiable) ; pour des dessins très volumineux, une approche par delta/commande
  sera plus économe en mémoire.
- Aucune icône bitmap n'est fournie (uniquement un vecteur simple) : à remplacer
  par une identité visuelle définitive.
