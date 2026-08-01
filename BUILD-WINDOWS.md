# Building and running ridgerunner on Windows (MinGW / MSYS2)

This guide targets **MinGW64 via MSYS2** so you can tighten KnotPlot centerlines
from [SST-Workbench](../SST-Workbench) on Windows. Visual Studio/MSVC is not
supported in this setup.

## One-time MSYS2 packages

In a **MinGW64** shell (`C:\msys64\mingw64.exe`):

```sh
pacman -S --needed \
  mingw-w64-x86_64-toolchain \
  mingw-w64-x86_64-cmake \
  mingw-w64-x86_64-ninja \
  mingw-w64-x86_64-pkgconf \
  mingw-w64-x86_64-openblas \
  mingw-w64-x86_64-gsl \
  mingw-w64-x86_64-argtable
```

Sibling clones expected next to this repo:

- `../ridge_tsnnls`
- `../ridge_plcurve`
- `../SST-Workbench` (read-only reference / sample data)

## Build

From the MinGW64 shell:

```sh
cd /c/workspace/projects/ridgerunner
./scripts/build-mingw.sh
```

This installs into `../ridge-prefix` by default (`PREFIX` overrides).

Or manually:

```sh
export PREFIX=/c/workspace/projects/ridge-prefix
export PATH="/mingw64/bin:$PATH"

cmake -S ../ridge_tsnnls -B ../ridge_tsnnls/build-mingw -G Ninja \
  -DCMAKE_INSTALL_PREFIX=$PREFIX -DCMAKE_BUILD_TYPE=Release
cmake --build ../ridge_tsnnls/build-mingw --target install

cmake -S ../ridge_plcurve -B ../ridge_plcurve/build-mingw -G Ninja \
  -DCMAKE_INSTALL_PREFIX=$PREFIX -DCMAKE_BUILD_TYPE=Release
cmake --build ../ridge_plcurve/build-mingw --target install

cmake -S . -B build-mingw -G Ninja \
  -DCMAKE_PREFIX_PATH=$PREFIX -DCMAKE_INSTALL_PREFIX=$PREFIX \
  -DCMAKE_BUILD_TYPE=Release
cmake --build build-mingw --target install
```

## PATH for running

### Recommended: one-time User PATH for the `.txt` wrapper

```powershell
powershell -ExecutionPolicy Bypass -File .\scripts\windows\install-user-path.ps1
```

That adds `scripts\windows` to your **User** PATH. New terminals can then run:

```bat
ridgerunner -a -s 1000 C:\pad\naar\knotplot.txt
```

The wrapper (`ridgerunner.cmd`) sets `ridge-prefix\bin` and `C:\msys64\mingw64\bin`
internally, converts XYZ `.txt` → VECT, runs `ridgerunner.exe`, and writes
`knotplot_ridgerunned.txt` next to the input. Intermediate `knotplot.vect` and
`knotplot.rr\` stay beside the input as well.

Requires Python 3 on PATH for `.txt` runs. Plain VECT files are forwarded to
`ridgerunner.exe` unchanged.

### Manual PATH (MinGW shell / VECT-only)

```sh
export PATH="/c/workspace/projects/ridge-prefix/bin:/mingw64/bin:$PATH"
```

OpenBLAS DLLs live under `/mingw64/bin`; without that directory on `PATH`,
`ridgerunner.exe` will fail to start.

## KnotPlot workflow (SST-Workbench)

### Easy path: `.txt` wrapper (recommended)

```bat
ridgerunner -a -s 1000 C:\workspace\projects\SST-Workbench\KnotPlot\knots\knot_3.1\T_2_3_trial_005k.txt
```

Output: `T_2_3_trial_005k_ridgerunned.txt` in the same folder as the input.

Or call Python directly:

```bat
python tools\run_knotplot_txt.py -a -s 20 --NoOutputFiles path\to\knot.txt
```

### Manual path: convert then run VECT

SST-Workbench stores KnotPlot exports as plain XYZ `.txt` files. Ridgerunner
reads Geomview **VECT**. Convert with the existing helper (**do not modify
SST-Workbench**):

```powershell
python .\SST-Workbench\KnotPlot\knotplot_txt_to_vect.py `
  .\SST-Workbench\KnotPlot\knots\knot_3.1\T_2_3_trial_005k.txt
```

Docs: `SST-Workbench/KnotPlot/KNOTPLOT_TXT_TO_VECT_README.md`.

Then tighten (recommended: autoscale + step limit):

```sh
ridgerunner -a -s 1000 T_2_3_trial_005k.vect
```

Output directory: `T_2_3_trial_005k.rr/` with `T_2_3_trial_005k.final.vect`.

### Ready-made VECT sample

```sh
cp /c/workspace/projects/SST-Workbench/KnotPlot/T_2_3_trial_005k.vect .
ridgerunner -a -s 20 --NoOutputFiles T_2_3_trial_005k.vect
```

### Multi-component link (optional)

```sh
python /c/workspace/projects/SST-Workbench/KnotPlot/knotplot_txt_to_vect.py \
  /c/workspace/projects/SST-Workbench/KnotPlot/knots/Tlink_6_9/Tlink_6_9_D1_040k.txt
ridgerunner -a -s 20 --NoOutputFiles Tlink_6_9_D1_040k.vect
```

VortexLab’s `knotplot_knots_data.js` Fourier catalog is a **separate** pipeline
and is not an input to ridgerunner.

Ideal-knot campaign scripts and run outputs live in SST-Workbench
`KnotPlot/ridgerunner` (`run_ideal_knot.cmd`), not in this compile repo.

## Notes

- Curses display stays off on Windows (stdout progress).
- Autotools (`./configure && make`) remains the Linux/macOS path; CMake is the
  Windows/MinGW path.
