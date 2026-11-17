# AGENTS.md

## Layout
- Three independent coursework parts: `apprentissage/` (Pokémon ML), `analyse_image/` (Fashion-MNIST/Keras), and `agent/` (console C++17 library manager). No shared app, package manifest, or root build.
- Notebook prose and user-facing C++ output are French; preserve that convention.

## Verification
- No test, lint, typecheck, formatter, pre-commit, or CI config exists; don't invent project-wide commands.
- Build the C++ app with `cmake -S agent -B agent/build && cmake --build agent/build`.
- `agent/CMakeLists.txt` uses `file(GLOB "*.cpp")`: rerun CMake configure after adding/removing `.cpp` files. It targets C++17 with `-Wall -Wextra -Wpedantic`.
- `agent/build/` is not committed and is ignored by `agent/.gitignore` (`build/*`).
- VS Code CMake source is hard-coded to `/workspaces/420-5G4-ML/agent` in `.vscode/settings.json`.

## C++ `agent/`
- Run sample data with `./agent/build/bibliotheque -d agent/data` (`--data-dir` also works). Without `-d`, file paths are empty, so no data is loaded and saves fail.
- `data/books.txt` is `title|author|isbn|1-or-0|borrowerId`; `data/users.txt` is `name|userId|isbn1,isbn2,...`. Preserve these formats.
- Menu option `0` auto-saves before exit; option `13` writes `.backup` files beside the data files. Use a copy of `agent/data` for manual testing.

## Notebooks
- No `requirements.txt`/`pyproject.toml`; `.venv/` exists locally with dependencies installed but is gitignored. Notebook metadata expects a `.venv` kernel (Python 3.12.3).
- Run notebooks from their own directory: all data/model/image paths are relative (`data/...`, `modele/...`, `images/...`), not repo-root-relative.
- `apprentissage/`: `partie1_2.ipynb` writes tab-separated `data/dataset.csv`; `partie3_entrainement.ipynb` reads it with `delimiter='\t'` and writes `modele/modele_pokemon.mod`; `partie3_predictions.ipynb` loads that committed model. Rerunning the first two overwrites committed artifacts.
- `analyse_image/classification.ipynb` requires `data/analyse_images/fashion-mnist_{train,test}.csv`; they are present locally but gitignored and untracked. It deliberately disables GPU (`tf.config.set_visible_devices([], 'GPU')`) and writes `modele/modele.json` + `modele/modele.weights.h5`.
- `analyse_image/test_model.ipynb` loads those two model files and uses hard-coded `images/image.png`. Its `modele_4_convolutions` option cannot work as shipped: that optional model is trained but never saved.
