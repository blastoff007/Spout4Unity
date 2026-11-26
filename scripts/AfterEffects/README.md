# Mapping Blinker (After Effects — ExtendScript)

Script de video-mapping pour fenetres de batiments : il genere le clignotement
de chaque fenetre a partir des masques d'un calque PNG.

## Installation

Copier `MappingBlinker.jsx` dans :

- `After Effects/Scripts/` (execution via Fichier > Scripts), ou
- `After Effects/Scripts/ScriptUI Panels/` (panneau ancree dans l'espace de travail).

Relancer After Effects.

## Utilisation

1. Dans la composition, importer le PNG du batiment et creer un calque.
2. Tracer un masque rectangulaire sur chaque fenetre (outil rectangle de masque).
3. Dupliquer un calque plein/noir tenu sous l'image de reference (optionnel :
   le script duplique lui-meme le calque selectionne pour animer les masques).
4. Selectionner le calque masque.
5. Dans le panneau Mapping Blinker :
   - regler le **BPM** (champ +//-) ou **taper le tempo** avec Ctrl+Espace (ou le
     bouton Tap) ;
   - choisir le **mode** : Unisson, Aleatoire, Sequentiel, Vague ;
   - choisir le **regroupement** : fenetres separees, lignes, colonnes, symetrie ;
   - choisir le **sens** : Haut-Bas, Bas-Haut, Gauche-Droite, Droite-Gauche ;
   - regler le **ratio allume** (% de la periode ou la fenetre est allumee),
     la **tolerance** (px) utilisee pour le regroupement en lignes / colonnes /
     symetrie, et la **graine aleatoire** pour un motif aleatoire reproductible ;
   - cliquer sur **Generer le clignotement**.

Le script duplique le calque selectionne en `"<calque> [Mapping Blinker]"` et
anime l'opacite de chaque masque en images-cles sur la zone de travail de la
composition, revelant le calque original place dessous (fenetres qui s'eteignent)
ou s'obscurcissant selon l'empilement.

## Details des modes

| Mode        | Comportement                                                             |
|-------------|--------------------------------------------------------------------------|
| Unisson     | Toutes les fenetres clignotent ensemble au BPM.                          |
| Aleatoire   | Decalage de phase et sauts de periode aleatoires par groupe (graine).    |
| Sequentiel  | Les groupes s'allument l'un apres l'autre (balayage cyclique).           |
| Vague       | Decalage progressif le long du sens choisi (onde de clignotement).       |

## Regroupement

| Regroupement | Fenetres considerees ensemble                                            |
|--------------|--------------------------------------------------------------------------|
| Fenetres separees | Chaque masque independamment.                                      |
| Lignes       | Masques dont les centres Y sont a moins de `tolerance` px.               |
| Colonnes     | Masques dont les centres X sont a moins de `tolerance` px.               |
| Symetrie     | Paires miroir par rapport a l'axe vertical du calque (± `tolerance` px). |
