# Carte des scripts

Les 30 scripts Python du dépôt, groupés par rôle et dans l'ordre où ils ont tourné. Document
de navigation : il dit **à quoi sert chaque fichier et s'il est une étape du pipeline ou un
rendu jetable**, ce que la liste alphabétique de la racine ne dit pas.

Écrit en français, comme [`ARCHITECTURE.md`](ARCHITECTURE.md) et [`../data/LECTURE.md`](../data/LECTURE.md),
et comme les docstrings vers lesquelles il renvoie. Le [`README`](../README.md) d'entrée est en anglais.

- Ce que chaque script calcule et **pourquoi ce choix plutôt qu'un autre** : sa docstring.
- Le détail de `egfr_epitope_map.py` : [`ARCHITECTURE.md`](ARCHITECTURE.md)
- Les décisions, runs, coûts et erreurs, datés : [`../NOTES.md`](../NOTES.md)

Colonne **rôle** : `étape` = maillon du pipeline, `mesure` = produit un résultat rapporté,
`calibration` = mesure unique en lecture seule, `infra` = exécution distante, `rendu` = sortie
dérivée, régénérable, hors pipeline.

---

## 1. Cible et carte d'épitope — local, CPU, coût nul

Tout ce qui précède la génération. Aucun modèle n'est appelé.

| script | lignes | rôle | ce qu'il fait |
|---|---|---|---|
| [`egfr_epitope_map.py`](../egfr_epitope_map.py) | 1332 | étape | Énumère les patches de surface du domaine III et les décrit sur les cinq axes du challenge : conservation humain/souris, apolaire exposé, ancres acides conservées, proximité des N-glycanes, distance à l'empreinte du cétuximab. **Ne sélectionne pas** de site : produit des colonnes et des distributions. Sort `data/egfr_residues.csv` et `data/egfr_patches.csv`. Aussi importé comme bibliothèque par cinq autres scripts. |
| [`prepare_target.py`](../prepare_target.py) | 100 | étape | Extrait de 6ARU la chaîne A, polymère seul, tronquée aux bornes du domaine III → `inputs/6ARU_A_309-506.pdb`. Les bornes ne sont pas codées en dur : il interroge le résolveur CATH de `egfr_epitope_map`. |
| [`carboxylate_access.py`](../carboxylate_access.py) | 261 | mesure | SASA du **carboxylate** (OD1/OD2, OE1/OE2) et non du résidu entier, plus l'angle sortant du groupe. Un Asp dont le CB est exposé mais dont les oxygènes rentrent passerait un seuil de résidu tout en étant inatteignable. Valide `D323`. |
| [`hotspot_distances.py`](../hotspot_distances.py) | 67 | mesure | Distances CA entre hotspots candidats, sans PyMOL. Vérifie que chaque résidu existe avant de mesurer : une sélection vide donnerait une distance aberrante en silence. Source de l'étendue de 16,73 Å du jeu retenu. |
| [`footprint_extent.py`](../footprint_extent.py) | 156 | calibration | Étendue réelle de l'empreinte du Fab de cétuximab sur 6ARU, pour situer `PATCH_RADIUS = 11 Å` contre une empreinte mesurée. Lecture seule, ne modifie aucune constante. Source du « 11 Å ne capture que 40 % d'une empreinte réelle ». |
| [`mouse_target.py`](../mouse_target.py) | 170 | étape | Séquence du domaine III de l'EGFR murin (Q01279), par **deux dérivations indépendantes** — alignement, et colonne `aa_mouse` des CSV. Refuse d'écrire le fasta si elles divergent, s'il y a un indel dans la fenêtre, ou si le résidu 409 murin n'est pas une histidine. |

## 2. Génération et prédiction — Modal, GPU

| script | lignes | rôle | ce qu'il fait |
|---|---|---|---|
| [`modal_bindcraft2.py`](../modal_bindcraft2.py) | 589 | infra | Entrypoint de génération : image Modal portée du Dockerfile amont de BindCraft 2.0, entrypoint `selfcheck`, câblage des hotspots et coldspots, conversion d'un budget en dollars vers un timeout. Le build échoue exprès si `libcupti` n'est pas déclaré au loader — sinon jax tombe sur CPU en silence. |
| [`modal_boltz2.py`](../modal_boltz2.py) | 463 | infra | Prédicteur **orthogonal** : architecture et poids indépendants d'AF2, donc répond à « la pose est-elle réelle ou exploite-t-elle les biais d'AF2 ». Entrypoint `diagnose` à lancer avant toute série : sans `--no_kernels`, boltz sort en code 0 sans produire aucune prédiction. |
| [`fetch_msa.py`](../fetch_msa.py) | 176 | étape | Récupère **une fois** la MSA de chaque cible via l'API ColabFold et la met en cache, au lieu d'un appel réseau par complexe depuis le conteneur GPU. Fusionne les `.a3m` de l'archive en un alignement unique valide. MSA vide pour le binder, qui est de novo et n'a aucun homologue naturel. |
| [`modal_esm2.py`](../modal_esm2.py) | 151 | mesure | Pseudo-vraisemblance ESM-2. **Colonne descriptive, hors de tout classement** : elle mesure la ressemblance aux protéines naturelles, or le règlement exige la nouveauté. Classer dessus favoriserait les designs les moins nouveaux. |
| [`modal_proteinmpnn.py`](../modal_proteinmpnn.py) | 195 | mesure | Auto-cohérence de séquence par repliement inverse. **Circularité énoncée** : BindCraft utilise ProteinMPNN pour produire ses séquences, donc ce n'est pas une validation indépendante. |

## 3. Mécanisme pH — le cœur du dossier

| script | lignes | rôle | ce qu'il fait |
|---|---|---|---|
| [`prepare_structures.py`](../prepare_structures.py) | 118 | étape | Convertit les `.cif` de BindCraft en `.pdb` via gemmi, biopython refusant un fichier sans colonne `_atom_site.occupancy`. **Et refuse de convertir** un fichier dont le résidu 409 de la cible n'est pas une histidine : un décalage de numérotation ferait mesurer le mauvais résidu partout en aval sans qu'aucune valeur n'ait l'air anormale. |
| [`his_acid_pairing.py`](../his_acid_pairing.py) | 237 | mesure | Appariement His–acide aux interfaces, critère de 4,0 Å entre atomes lourds chargés (Barlow & Thornton 1983). Mesure sur `ND1`/`NE2` et `OD1/OD2`/`OE1/OE2`, pas sur le CB ni le centroïde. Verdict : route 1 morte, route 2 vivante sur un design, par un double pont salin. |
| [`propka_scan.py`](../propka_scan.py) | 307 | mesure | 141 runs PROPKA sur complexe, cible seule et binder seul. C'est le **décalage** de pKa de H409 entre lié et libre qui crée la sélectivité pH ; un pKa absolu n'en dit rien. Source du ΔpKa de +2,84. |
| [`verify_geometry.py`](../verify_geometry.py) | 350 | mesure | Trois mesures distinctes : réalité des ponts salins WT (distance, angle, enfouissement, contacts anormalement courts), portée des mutations greffées, qualité des rotamères. |
| [`bidentate_rule.py`](../bidentate_rule.py) | 263 | mesure | Teste la règle « il faut **deux** carboxylates distincts, un par azote de l'imidazole ». Trois tests, dont deux invalidés par auto-référence ; le seul correct sépare à 2,20 Å. Rapportée comme **soutenue, non établie**, et n'entre dans aucun critère de classement. |

## 4. Campagne de mutants acides — échouée, diagnostiquée

| script | lignes | rôle | ce qu'il fait |
|---|---|---|---|
| [`acid_mutants.py`](../acid_mutants.py) | 244 | étape | Propose une substitution vers Asp ou Glu par design, pour mettre un carboxylate à portée de H409. Choisit Asp si le CB est déjà proche, Glu s'il faut aller plus loin. **Ne prédit rien** : les métriques affichées décrivent la séquence d'origine, un mutant est une hypothèse à faire scorer. |
| [`thread_mutants.py`](../thread_mutants.py) | 191 | étape | Greffe la substitution sur le `.cif` du parent via le wizard de mutagenèse de PyMOL en headless, et sort plusieurs rotamères. Le threading garde le squelette **rigoureusement identique** entre parent et mutant, donc l'analyse appariée n'a qu'une variable ; plusieurs rotamères permettent de dire si le verdict de pKa est stable ou indéterminé. |

Résultat : sur 12 mutants, 3 ont un rotamère à portée et sans clash, et l'analyse appariée donne
1 amélioration, 4 neutres, 7 dégradations. Aucun mutant n'entre dans la soumission. Le coût
structural mesuré étant négligeable, **l'échec est chimique et non structural** — proxy de
sélection pris sur le CB plutôt que sur le carboxylate, et carboxylates greffés dont le pKa
propre dépasse 5,0, donc neutres dès pH 6,5.

## 5. Analyse, classement, soumission

| script | lignes | rôle | ce qu'il fait |
|---|---|---|---|
| [`analyze_campaign.py`](../analyze_campaign.py) | 290 | mesure | Funnel, distributions, chronométrage et coût d'une campagne, depuis les CSV rapatriés du volume Modal. Source du coût par trajectoire et du taux d'acceptation. |
| [`contact_recovery.py`](../contact_recovery.py) | 265 | mesure | Accord entre la pose AF2 et la pose Boltz-2, en **paires de résidus en contact** et non en RMSD : un RMSD d'interface pardonne une rotation qui détruit l'interface et pénalise une translation sans conséquence. Asymétrique à dessein. |
| [`cross_species.py`](../cross_species.py) | 307 | mesure | Objectif n°2 **mesuré** et non plus approché par un proxy de séquence : pose et mécanisme pH sur la cible murine, même prédicteur des deux côtés. A révélé trois designs qui changent de mode de liaison, que le proxy ne distinguait pas. |
| [`ipsae_scores.py`](../ipsae_scores.py) | 199 | mesure | ipSAE sur les prédictions Boltz-2, via l'implémentation de **référence** de ses auteurs et non une réécriture. Corrige la dilution de l'ipTM sur une petite interface. |
| [`sequence_liabilities.py`](../sequence_liabilities.py) | 238 | mesure | Motifs de liabilité **pondérés par la SASA** du binder seul : un motif enfoui dans le cœur hydrophobe n'est pas une liabilité, l'eau n'y accède pas. Un balayage de motifs nu produirait des faux positifs en masse. |
| [`rank_designs.py`](../rank_designs.py) | 692 | étape | Classement du vivier et analyse appariée WT contre mutant. Tri **lexicographique par groupes** dans l'ordre du règlement — sélectivité pH, puis cross-réactivité, puis affinité — et non score composite pondéré, qui exigerait d'inventer des poids injustifiables et masquerait les arbitrages. |
| [`build_submission.py`](../build_submission.py) | 603 | étape | Construit le CSV de soumission et le classeur de travail, et passe les contrôles. Un design par squelette ; le quota de 20 est traité comme un **plafond** et le lot n'est jamais complété par des frères pour atteindre un chiffre rond. |

## 6. Rendus — hors pipeline, régénérables

Dépendances jetables via `uv run --with`, volontairement non déclarées dans `pyproject.toml`.

| script | lignes | rôle | ce qu'il fait |
|---|---|---|---|
| [`build_metadata_package.py`](../build_metadata_package.py) | 503 | rendu | Paquet de métadonnées de la soumission, **en anglais** parce qu'il est lu par le sélecteur : métriques à clés explicites, dictionnaire de colonnes, provenance, structures. |
| [`build_metrics_markdown.py`](../build_metrics_markdown.py) | 157 | rendu | Les mêmes métriques en Markdown : un CSV dans une archive peut ne pas être ouvert, un tableau Markdown est lu directement. |
| [`build_metrics_workbook.py`](../build_metrics_workbook.py) | 540 | rendu | Classeur public consolidé, lisible sans le reste du dépôt : chaque nombre avec sa source et son statut. Les seuils sont **importés des modules**, jamais recopiés. |
| [`build_workbook.py`](../build_workbook.py) | 294 | rendu | Classeur de travail du choix de site et de hotspots, quatre feuilles. Reprend les CSV sans les modifier et ajoute les colonnes dérivées qui manquaient pour trancher à la main. |
| [`csv_to_workbook.py`](../csv_to_workbook.py) | 254 | rendu | CSV d'une campagne BindCraft vers un classeur par campagne. Exclut par défaut les `*_losses.csv` par trajectoire, qui sont des traces de gradient et ne se lisent pas à 90 feuilles. |

---

## Ordre d'exécution

```
prepare_target ──┐
egfr_epitope_map ┤→ hotspot_distances, carboxylate_access, footprint_extent   choix du site
mouse_target ────┘

modal_bindcraft2 (selfcheck → design)                                         génération
fetch_msa → modal_boltz2                                                      re-scoring

prepare_structures → his_acid_pairing → propka_scan → verify_geometry         mécanisme pH
                                      → bidentate_rule

acid_mutants → thread_mutants → propka_scan                                   mutants (échec)

analyze_campaign, contact_recovery, cross_species, ipsae_scores,
sequence_liabilities → rank_designs → build_submission                        classement
```

`prepare_structures.py`, `mouse_target.py` et `hotspot_distances.py` **refusent de produire une
sortie** quand un invariant casse. C'est volontaire : dans ce pipeline, l'erreur qui coûte cher
est celle qui ne se signale pas.

## Dette technique connue

Contractée sciemment pour tenir la clôture du 5 octobre, et énoncée ici plutôt que découverte
par un relecteur.

- **`egfr_epitope_map.py` est à la fois un CLI et une bibliothèque.** Cinq scripts font
  `import egfr_epitope_map as E` — `prepare_target`, `carboxylate_access`, `footprint_extent`,
  `hotspot_distances`, `build_workbook`. Cela ne fonctionne que parce que tous les fichiers sont
  frères à la racine. Il aurait dû être scindé en un paquet importable et un entrypoint.
- **Trois stratégies de lecture mmCIF coexistent** : `MMCIFParser` de biopython, gemmi, et un
  parseur maison de la boucle `_atom_site`. Le parseur maison et sa dataclass `Atom` sont
  dupliqués presque à l'identique dans `his_acid_pairing.py` et `acid_mutants.py`. Un module
  `cif_io.py` serait le premier refactor.
- **Aucun test automatisé.** Les invariants sont vérifiés à l'exécution par les scripts qui
  refusent de produire une sortie, jamais par une suite de tests.
- **30 scripts à plat à la racine.** Défendable dans la mesure où chacun est un entrypoint CLI
  autonome ; la conséquence est que ce document est nécessaire.

Hors dépôt : `build_cours_pdf.py` existe en local et n'est pas suivi par git. Il génère un
document pédagogique dérivé des CSV et ne fait pas partie du pipeline.
