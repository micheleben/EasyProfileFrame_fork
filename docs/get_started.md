# Easy Profile Frame - Getting Started

## What the tool expects (the intended workflow)

The "Create Profile" command is **not a standalone script** — it's deeply integrated with FreeCAD's runtime object model. It reads geometry live from the active document via FreeCAD's Python API. Here's the step-by-step intended use:

### Step 1: You prepare the "path" (edges/wires)

Before the command activates, you need to **select edges** in the 3D view. These edges define *where* the profile frames will be placed — think of them as the skeleton of your frame.

These edges can come from:
- **A Sketcher sketch** — e.g. a rectangle sketch where each side becomes a frame member
- **Draft lines** (`Part::Part2DObjectPython`) — lines drawn with the Draft workbench

The command's `IsActive()` guard (`create_profiles.py:419-425`) checks this — the button stays greyed out until you've selected valid edges:

```python
def IsActive(self):
    selected_objects = Gui.Selection.getSelectionEx()
    if not IsAllWires(selected_objects):
        return False
    return selected_objects != []
```

### Step 2: You click the command

`Activated()` in `create_profiles.py:439` fires. It:
1. Creates a `CreateProfilesBySketchPanel` which opens a Qt side-panel via `Gui.Control.showDialog()`
2. Immediately reads your current selection (`self.form.add_wires()` in the constructor)
3. Creates an `App::Part` container in the active document to hold all frame bodies

### Step 3: You pick a cross-section profile

The panel gives you two options:

- **Library profile**: Opens `AluminumProfiles.FCStd` as a hidden background document, lists all `Sketcher::SketchObject` objects inside it, and lets you pick one. These are 2D cross-section shapes (e.g. a 20x20 aluminum extrusion outline).

- **Custom sketch**: You select any `Sketcher::SketchObject` from your current document — this is your own 2D profile cross-section.

### Step 4: The geometry gets built

When `_draw()` fires (`create_profiles.py:209`), the magic happens. For each selected edge:

1. **`CreateProfileFrameBody()`** (`ProfileFrameObject.py:475-496`) creates a `PartDesign::FeatureAdditivePython` — this is FreeCAD's mechanism for custom parametric objects backed by Python. The `ProfileFrameObject` class becomes its `Proxy`.

2. **The profile sketch is copied** into the body (via `CopyObj` in `utils.py:77-85`), then the copy is attached to the edge using FreeCAD's attachment system:
   ```python
   obj.AttachmentSupport = [(edge_sketch, subedge)]
   obj.MapMode = "NormalToEdge"
   ```
   This orients the 2D profile perpendicular to the edge at its start point.

3. **A `PartDesign::Pad`** is created (`ProfileFrameObject.py:250-276`) that extrudes the profile sketch along the edge's length — this is the actual 3D frame member.

4. **Corner joints** are applied depending on your radio button choice:
   - **NoProcessing**: Just pads, no cuts
   - **MiterCut** (`create_profiles.py:286-339`): Finds pairs of edges that share a vertex (intersection), calculates the angle between them via `calculate_edges_angle()`, then creates chamfer cuts (triangular `PartDesign::Pocket` operations) at each end
   - **AutoAlignA/B**: For 90-degree corners, extends/shortens frame members so they overlap/butt correctly

## The interface between Sketcher and the Python script

There is no file-based interface or parsing. The script talks to FreeCAD objects **live in memory** through the FreeCAD Python API:

| What the script needs | How it gets it | API call |
|---|---|---|
| Selected edges | FreeCAD selection service | `Gui.Selection.getSelectionEx()` |
| Edge names (e.g. `Sketch:Edge1`) | Selection subelement names | `obj.SubElementNames` |
| Actual edge geometry | Document object tree | `doc.getObject("Sketch").getSubObject("Edge1")` |
| Edge length | Part.Edge property | `edge.Length` |
| Shared vertices between edges | Vertex point comparison | `edge.Vertexes[i].Point == edge2.Vertexes[j].Point` |
| Angle between edges | Tangent vectors + dot product | `edge.tangentAt()` → `math.acos()` |
| Profile 2D shape | Sketcher object from library or document | `doc.getObjectsByLabel("profileName")` |
| 3D extrusion | PartDesign Pad feature | `PartDesign::Pad` with `.Profile` and `.Length` |
| Chamfer cuts | PartDesign Pocket + generated sketch | `PartDesign::Pocket` with a triangular cutting sketch |

## How to use it

1. Open FreeCAD, switch to the **Easy Profile Frame** workbench
2. Create a new document
3. Create a **Sketch** (using Sketcher workbench) — draw lines/a rectangle representing where you want frame members
4. Go back to the Easy Profile Frame workbench
5. In the 3D view, **select the edges** of your sketch (click individual edges, or select the whole sketch)
6. The "Create Profile" button should now be active — click it
7. Pick a profile from the library (e.g. an aluminum extrusion) and choose a joint method
8. Frame members appear as 3D solids along your selected edges

## Manual Installation (bypassing the Addon Manager)

FreeCAD loads workbenches from its `Mod/` directories. The user-level location depends on your OS.

---

### Windows

The user-level `Mod/` directory is:

```
%APPDATA%\FreeCAD\Mod\
```

Which typically resolves to:

```
C:\Users\<username>\AppData\Roaming\FreeCAD\Mod\
```

**1. Find/create the Mod directory**

```cmd
mkdir "%APPDATA%\FreeCAD\Mod" 2>nul
```

**2. Create a symbolic link** (preferred — changes in the repo are reflected instantly):

Open a **Command Prompt as Administrator** and run:

```cmd
mklink /D "%APPDATA%\FreeCAD\Mod\EasyProfileFrame" "path\to\your\repo\clone"
```

**Alternative** — copy the entire repo folder:

```cmd
xcopy "path\to\your\repo\clone" "%APPDATA%\FreeCAD\Mod\EasyProfileFrame\" /E /I
```

**3. Restart FreeCAD**

---

### Linux (Ubuntu) — installed via apt

When FreeCAD is installed with `apt`, the user-level `Mod/` directory is:

```
~/.local/share/FreeCAD/Mod/
```

**1. Install FreeCAD**

```bash
sudo apt update
sudo apt install freecad
```

**2. Create the Mod directory**

```bash
mkdir -p ~/.local/share/FreeCAD/Mod
```

**3. Create a symbolic link** (preferred):

```bash
ln -s /path/to/your/repo/clone ~/.local/share/FreeCAD/Mod/EasyProfileFrame
```

**Alternative** — copy the repo folder:

```bash
cp -r /path/to/your/repo/clone ~/.local/share/FreeCAD/Mod/EasyProfileFrame
```

**4. Restart FreeCAD**

---

### Linux (Ubuntu) — installed via AppImage

When FreeCAD is run as an AppImage, FreeCAD still uses the same user-level `Mod/` directory:

```
~/.local/share/FreeCAD/Mod/
```

**1. Download the AppImage**

Get the latest AppImage from the [FreeCAD releases page](https://github.com/FreeCAD/FreeCAD/releases).

**2. Make it executable and run it**

```bash
chmod +x FreeCAD_*.AppImage
./FreeCAD_*.AppImage
```

**3. Create the Mod directory**

```bash
mkdir -p ~/.local/share/FreeCAD/Mod
```

**4. Create a symbolic link** (preferred):

```bash
ln -s /path/to/your/repo/clone ~/.local/share/FreeCAD/Mod/EasyProfileFrame
```

**Alternative** — copy the repo folder:

```bash
cp -r /path/to/your/repo/clone ~/.local/share/FreeCAD/Mod/EasyProfileFrame
```

**5. Restart FreeCAD**

---

After restarting, FreeCAD will (on all platforms):
1. Scan `Mod/EasyProfileFrame/`
2. Add it to `sys.path`
3. Discover the `freecad.easy_profile_frame` namespace package
4. Execute `init_gui.py` which calls `Gui.addWorkbench()`
5. The workbench **"Easy profile frame"** should appear in the workbench dropdown

### Why the Addon Manager fails

This add-on has **no `package.xml`** file, which newer FreeCAD versions expect for the Addon Manager. The manual installation method above bypasses this requirement entirely.

### Requirements

- **FreeCAD 0.19+** (ideally 0.21+) — required for the namespace package pattern (`freecad.easy_profile_frame` with `init_gui.py`)
- No external Python dependencies — only FreeCAD built-in modules (`Part`, `PartDesign`, `Sketcher`, `Spreadsheet`, `PySide`)
