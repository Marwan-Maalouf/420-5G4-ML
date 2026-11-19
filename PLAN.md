# Plan — Bibliothèque C++ / pybind11 / CustomTkinter

## Objectif

Remplacer l’interface CLI de `agent/main.cpp` par une application graphique CustomTkinter, en conservant le cœur C++ existant et en l’exposant à Python avec pybind11.

## Constats de départ

- [ ] Le cœur métier est dans `book.*`, `user.*`, `library.*` et `filemanager.*`.
- [ ] `main.cpp` contient uniquement l’interface CLI et l’orchestration.
- [ ] `agent/CMakeLists.txt` utilise `file(GLOB "*.cpp")` : il faut séparer le cœur, le CLI et `bindings.cpp`.
- [ ] `Library` retourne des `Book*`/`User*` : les bindings doivent retourner des copies pour éviter les pointeurs pendants.
- [ ] `Library::removeBook` laisse l’ISBN dans les emprunts des utilisateurs : à corriger.
- [ ] `searchBooksByTitle/Author` utilisent `::tolower` sur des `char` signés avec de l’UTF-8 : à sécuriser.
- [ ] `FileManager::createBackup` échoue si les fichiers `.backup` existent déjà.
- [ ] `tkinter`, `customtkinter` et `pybind11` sont absents de l’environnement actuel.
- [ ] `.venv/bin/python` est en réalité Python 3.13.5 malgré un `pyvenv.cfg` indiquant 3.12.3.
- [ ] Le devcontainer `agent/.devcontainer` installe `pybind11` et `python3-tk`, mais pas `customtkinter`.
- [ ] Le devcontainer racine ne monte pas X11 ; utiliser `agent/.devcontainer` pour la GUI.

## Architecture cible

```text
agent/
├── book.* / user.* / library.* / filemanager.*   # cœur C++
├── bindings.cpp                                  # module pybind11
├── main.cpp                                      # CLI conservé
├── gui/
│   ├── main.py                                   # point d’entrée GUI
│   ├── app.py                                    # fenêtre principale
│   ├── service.py                                # wrapper Library + FileManager
│   ├── requirements.txt                          # customtkinter, pybind11
│   └── views/                                    # écrans
├── tests/test_bindings.py                        # tests Python
└── build/python/bibliotheque_py*.so              # module compilé
```

## API C++ à exposer

### `Book` — lecture seule

- [ ] `Book(title, author, isbn)`
- [ ] `getTitle` → `title`
- [ ] `getAuthor` → `author`
- [ ] `getISBN` → `isbn`
- [ ] `getAvailability` → `is_available`
- [ ] `getBorrowerId` → `borrower_id`
- [ ] `toString` → `to_string`
- [ ] `toFileFormat` optionnel, pour les tests uniquement
- [ ] Ne pas exposer les setters ni `checkOut`/`returnBook`

### `User` — lecture seule

- [ ] `User(name, userId)`
- [ ] `getName` → `name`
- [ ] `getUserId` → `user_id`
- [ ] `getBorrowedBooks` → `borrowed_books`
- [ ] `getNumberOfBorrowedBooks` → `borrowed_count`
- [ ] `toString` → `to_string`
- [ ] Ne pas exposer `borrowBook`/`returnBook`

### `Library`

- [ ] `Library()`
- [ ] `addBook` → `add_book(book)`
- [ ] `removeBook` → `remove_book(isbn) -> bool`
- [ ] `findBookByISBN` → `find_book_by_isbn(isbn) -> Book | None`
- [ ] `searchBooksByTitle` → `search_books_by_title(query) -> list[Book]`
- [ ] `searchBooksByAuthor` → `search_books_by_author(query) -> list[Book]`
- [ ] `getAvailableBooks` → `get_available_books() -> list[Book]`
- [ ] `getAllBooks` → `get_all_books() -> list[Book]`
- [ ] `addUser` → `add_user(user)`
- [ ] `findUserById` → `find_user_by_id(id) -> User | None`
- [ ] `getAllUsers` → `get_all_users() -> list[User]`
- [ ] `checkOutBook` → `check_out_book(isbn, user_id) -> bool`
- [ ] `returnBook` → `return_book(isbn) -> bool`
- [ ] `getTotalBooks` → `total_books()`
- [ ] `getAvailableBookCount` → `available_book_count()`
- [ ] `getCheckedOutBookCount` → `checked_out_book_count()`
- [ ] Ne pas exposer `displayAllBooks`, `displayAvailableBooks`, `displayAllUsers`

### `FileManager`

- [ ] `FileManager(books_file, users_file)`
- [ ] `loadLibraryData` → `load_library_data(library) -> bool`
- [ ] `saveLibraryData` → `save_library_data(library) -> bool`
- [ ] `createBackup` → `create_backup()`
- [ ] `fileExists` → `file_exists(path) -> bool`
- [ ] Les opérations individuelles `saveBooksToFile`/`loadUsersFromFile` restent internes

### Détails pybind11

- [ ] Inclure `pybind11/stl.h` pour `vector` et `optional`.
- [ ] Retourner des copies : `std::optional<Book>`, `std::vector<Book>`, etc.
- [ ] Ne pas exposer les `unique_ptr` internes de `Library`.
- [ ] Nommer le module `bibliotheque_py`.
- [ ] Conserver `Library` et `FileManager` ensemble dans le service Python.

## Écrans nécessaires

- [ ] **Fenêtre principale** : navigation latérale, zone de contenu, barre de statut.
- [ ] **Tableau de bord** : cartes total livres, disponibles, empruntés, utilisateurs ; actions rapides.
- [ ] **Livres** : tableau ISBN/titre/auteur/disponibilité/emprunteur ; recherche titre/auteur ; filtre tous/disponibles/empruntés ; ajouter/supprimer.
- [ ] **Dialog ajout de livre** : titre, auteur, ISBN ; validation des champs vides et des doublons.
- [ ] **Utilisateurs** : tableau nom/ID/nombre d’emprunts/ISBN empruntés ; ajouter.
- [ ] **Dialog ajout d’utilisateur** : nom, ID ; validation des doublons.
- [ ] **Emprunts** : formulaire d’emprunt avec livres disponibles et utilisateurs ; formulaire de retour avec livres empruntés.
- [ ] **Statistiques** : total, disponibles, empruntés, utilisateurs.
- [ ] **Données / sauvegarde** : dossier utilisé, recharger, sauvegarder, créer un backup.
- [ ] **Dialogs transverses** : confirmation de suppression, détail livre/utilisateur, échec de sauvegarde à la fermeture.

## Plan par étapes

### Étape 0 — Préparer l’environnement

- [x] Installer `python3-tk`.
- [x] Installer `pybind11` et `customtkinter` dans `.venv`.
- [x] Ajouter `agent/gui/requirements.txt` avec les versions de `customtkinter` et `pybind11`.
- [x] Mettre à jour `agent/.devcontainer/Dockerfile` pour inclure `customtkinter`.
- [x] Ajouter les exclusions Python au `.gitignore` (`__pycache__`, `*.pyc`).
- [x] Utiliser explicitement `.venv/bin/python` pour le build et les tests.
- [x] Vérifier `DISPLAY` et l’accès X11 via `agent/.devcontainer`.

**Test** : imports réussis de `tkinter`, `customtkinter`, `pybind11`, puis build CLI actuel.

### Étape 1 — Établir le comportement de référence

- [ ] Copier `agent/data` dans un dossier temporaire.
- [ ] Tester le CLI sur cette copie : ajout, suppression, recherche, emprunt, retour, sauvegarde, backup, sortie.
- [ ] Conserver les `books.txt`/`users.txt` obtenus comme référence de non-régression.

**Test** : le scénario CLI complet produit des fichiers de données conformes.

### Étape 2 — Séparer le cœur, le CLI et le module Python dans CMake

- [ ] Créer une cible `bibliotheque_core` avec `book.cpp`, `user.cpp`, `library.cpp`, `filemanager.cpp`.
- [x] Garder l’exécutable `bibliotheque` avec seulement `main.cpp`.
- [x] Supprimer la logique de glob pour le CLI et le module Python.
- [x] Appliquer C++17 et `-Wall -Wextra -Wpedantic` à la cible cœur.
- [x] Ajouter une option CMake `BUILD_PYTHON_BINDINGS`, désactivée par défaut.
- [x] Éviter que `main.cpp` soit compilé dans le module et `bindings.cpp` dans le CLI.

**Test** : `cmake -S agent -B agent/build && cmake --build agent/build`, puis exécution du CLI sur la copie de données.

### Étape 3 — Durcir le cœur avant exposition

- [ ] Corriger `Library::removeBook` pour retirer l’ISBN des emprunts de tous les utilisateurs.
- [ ] Remplacer `::tolower` non sûr par un traitement ASCII sûr, ou filtrer avec `casefold` côté Python.
- [ ] Corriger `FileManager::createBackup` : écraser, horodater, ou retourner un statut.
- [ ] Intercepter les erreurs `std::filesystem` du backup.
- [ ] Définir le comportement si `books.txt` ou `users.txt` est absent.
- [ ] Définir le comportement si une ligne est mal formée.
- [ ] Définir le comportement si le dossier n’est pas accessible en écriture.

**Test** : emprunter un livre, le supprimer, sauvegarder/recharger, vérifier que l’utilisateur ne contient plus l’ISBN ; lancer deux backups successifs.

### Étape 4 — Ajouter `bindings.cpp` et la cible pybind11

- [ ] Créer `agent/bindings.cpp` avec `PYBIND11_MODULE`.
- [ ] Exposer `Book`, `User`, `Library` et `FileManager` selon la liste ci-dessus.
- [ ] Ajouter `find_package(Python COMPONENTS Interpreter Development REQUIRED)`.
- [ ] Ajouter `find_package(pybind11 CONFIG REQUIRED)`.
- [ ] Lier le module à `bibliotheque_core`, pas à `main.cpp`.
- [ ] Définir le répertoire de sortie `agent/build/python`.
- [ ] Documenter `Python_EXECUTABLE` et `pybind11_DIR` issus du `.venv`.

**Test** : configurer avec `BUILD_PYTHON_BINDINGS=ON` et vérifier la présence du `.so`.

### Étape 5 — Tester les bindings en Python

- [ ] Ajouter `agent/tests/test_bindings.py`.
- [ ] Tester ajout/recherche/suppression de livre.
- [ ] Tester le refus des ISBN en doublon.
- [ ] Tester ajout d’utilisateur et refus des IDs en doublon.
- [ ] Tester emprunt, retour et statistiques.
- [ ] Tester sauvegarde et rechargement.
- [ ] Tester la stabilité des objets `Book`/`User` après suppression.
- [ ] Tester la recherche avec accents.

**Test** : `PYTHONPATH=agent/build/python .venv/bin/python -m unittest discover -s agent/tests -v`.

### Étape 6 — Créer la couche de service Python

- [ ] Ajouter `agent/gui/service.py` avec `LibraryService`.
- [ ] Faire posséder `Library` et `FileManager` par le service.
- [ ] Résoudre `books.txt` et `users.txt` à partir du dossier de données.
- [ ] Gérer le chargement initial, la sauvegarde et les erreurs.
- [ ] Valider les champs vides, les doublons et les ISBN.
- [ ] Convertir les booléens C++ en messages ou exceptions françaises.
- [ ] Ne pas dépendre des messages `stdout` du C++.
- [ ] Recalculer les statistiques après chaque mutation.
- [ ] Prévoir la recréation complète du service lors d’un changement de dossier.

**Test** : tests unitaires du service sur un dossier temporaire, sans interface graphique.

### Étape 7 — Créer le shell CustomTkinter

- [ ] Ajouter `agent/gui/main.py` et `agent/gui/app.py`.
- [ ] Ajouter `__init__.py` aux packages nécessaires.
- [ ] Ajouter la navigation latérale et la zone de contenu.
- [ ] Ajouter une barre de statut pour les succès et erreurs.
- [ ] Accepter `--data-dir`.
- [ ] Charger les données au démarrage.
- [ ] Intercepter la fermeture de fenêtre pour sauvegarder.
- [ ] Afficher un état « modifications non sauvegardées ».
- [ ] Décider si les mutations sont auto-sauvegardées ou seulement à la fermeture.

**Test** : lancer la fenêtre, naviguer entre les pages, fermer et vérifier la sauvegarde.

### Étape 8 — Écran Livres

- [ ] Afficher le tableau des livres avec `ttk.Treeview` dans un conteneur CTk.
- [ ] Ajouter la recherche par titre/auteur.
- [ ] Ajouter le filtre tous/disponibles/empruntés.
- [ ] Ajouter le dialog d’ajout de livre.
- [ ] Ajouter la confirmation de suppression.
- [ ] Avertir avant de supprimer un livre emprunté.
- [ ] Rafraîchir le tableau et les listes après chaque opération.
- [ ] Afficher un état vide si la bibliothèque est vide.

**Test** : reproduire les anciennes options 1 à 6 du menu CLI.

### Étape 9 — Écran Utilisateurs

- [ ] Afficher le tableau des utilisateurs et leurs emprunts.
- [ ] Ajouter le dialog d’ajout d’utilisateur.
- [ ] Contrôler les doublons d’ID.
- [ ] Rafraîchir après ajout.
- [ ] Afficher un état vide si aucun utilisateur.

**Test** : reproduire les anciennes options 7 et 8.

### Étape 10 — Écran Emprunts

- [ ] Ajouter le formulaire d’emprunt avec livres disponibles et utilisateurs.
- [ ] Ajouter le formulaire de retour avec livres empruntés.
- [ ] Désactiver les actions sans sélection.
- [ ] Rafraîchir les listes après emprunt/retour.
- [ ] Afficher les erreurs : livre indisponible, utilisateur inconnu, livre non emprunté.

**Test** : reproduire les anciennes options 9 et 10, puis vérifier les fichiers après sauvegarde.

### Étape 11 — Statistiques, données et sortie

- [ ] Alimenter le tableau de bord et l’écran statistiques avec les compteurs C++.
- [ ] Afficher le dossier de données utilisé.
- [ ] Ajouter les boutons recharger, sauvegarder et créer un backup.
- [ ] Gérer le changement de dossier de données en recréant le service.
- [ ] Rafraîchir toutes les vues après un rechargement.
- [ ] Gérer l’échec de sauvegarde à la fermeture.
- [ ] Rendre le backup idempotent (écrasement ou horodatage).
- [ ] Afficher un message clair en cas de backup impossible.

**Test** : reproduire les options 11, 12, 13 et 0 ; vérifier deux backups successifs.

### Étape 12 — Validation finale et documentation

- [ ] Rejouer le scénario de référence de l’étape 1 uniquement via la GUI.
- [ ] Comparer les fichiers sauvegardés avec ceux du CLI.
- [ ] Vérifier que le CLI reste fonctionnel.
- [ ] Tester le lancement sans terminal et sans messages `stdout` visibles.
- [ ] Documenter les commandes de build, de test et de lancement dans `agent/README.md`.
- [ ] Documenter l’emplacement du module compilé et le `PYTHONPATH`.
- [ ] Mettre à jour `AGENTS.md` si de nouvelles commandes ou dépendances sont ajoutées.

**Test** : checklist GUI couvrant les 14 anciennes options et scénario complet sauvegarde/rechargement.

## Périmètre explicite

- [ ] Pas d’édition de livre : le cœur C++ ne le permet pas.
- [ ] Pas de suppression d’utilisateur : le cœur C++ ne le permet pas.
- [ ] Pas de gestion de plusieurs bibliothèques simultanées : un seul dossier de données à la fois.
- [ ] Pas de dépendance Unicode externe : la recherche accent-insensible sera faite côté Python si nécessaire.
- [ ] Le CLI reste disponible comme fallback et outil de test.
