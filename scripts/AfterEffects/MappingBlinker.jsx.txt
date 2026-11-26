/*
    Mapping Blinker
    Script Adobe After Effects (ExtendScript) pour le video-mapping de fenetres.

    Principe : les fenetres d'un batiment sont definies par les masques d'un calque
    PNG (calque "mapping"). La duplication de ce calque recoit des images-cles
    d'opacite de masque : chaque fenetre s'allume et s'eteint au rythme desire,
    revelant par transparence le calque original place dessous.

    Fonctions :
    - Tap tempo avec Ctrl+Espace (ou bouton Tap), BPM ajustable
    - Fenetres = masques d'un calque PNG selectionne
    - Modes de clignotement : unisson / aleatoire / sequentiel / vague
    - Regroupement des fenetres : fenetres / lignes / colonnes / symetrie
    - Sens de propagation : haut-bas, bas-haut, gauche-droite, droite-gauche

    Utilisation :
    1. Placer ce fichier dans le dossier Scripts de After Effects (ou Scripts/ScriptUI Panels
       pour un panneau ancree), puis relancer After Effects.
    2. Dans la composition, creer un calque PNG du batiment et tracer un masque
       rectangulaire par fenetre.
    3. Selectionner ce calque, regler BPM / mode / regroupement / sens, puis cliquer
       sur "Generer le clignotement".
*/

(function (thisObj) {

    var SCRIPT_NAME = "Mapping Blinker";
    var DIR_TOP_TO_BOTTOM = 0;
    var DIR_BOTTOM_TO_TOP = 1;
    var DIR_LEFT_TO_RIGHT = 2;
    var DIR_RIGHT_TO_LEFT = 3;

    var tapTimes = [];
    var rndState = 1;

    function seededRandom() {
        rndState = (rndState * 1664525 + 1013904223) % 4294967296;
        return rndState / 4294967296;
    }

    function isCompActive() {
        var item = app.project ? app.project.activeItem : null;
        return (item !== null && item instanceof CompItem);
    }

    function getMappingLayer(comp) {
        if (comp.selectedLayers.length === 0) {
            return null;
        }
        var layer = comp.selectedLayers[0];
        if (!(layer.property("Masks")) || layer.property("Masks").numProperties === 0) {
            return null;
        }
        return layer;
    }

    function maskBounds(mask) {
        var path = mask.property("ADBE Mask Shape").value;
        var v = path.vertices;
        var minX = v[0][0], maxX = v[0][0], minY = v[0][1], maxY = v[0][1];
        var i;
        for (i = 1; i < v.length; i++) {
            if (v[i][0] < minX) { minX = v[i][0]; }
            if (v[i][0] > maxX) { maxX = v[i][0]; }
            if (v[i][1] < minY) { minY = v[i][1]; }
            if (v[i][1] > maxY) { maxY = v[i][1]; }
        }
        return {
            left: minX,
            top: minY,
            right: maxX,
            bottom: maxY,
            cx: (minX + maxX) / 2,
            cy: (minY + maxY) / 2
        };
    }

    function collectWindows(layer) {
        var masks = layer.property("Masks");
        var windows = [];
        var i;
        for (i = 0; i < masks.numProperties; i++) {
            var mask = masks.property(i + 1);
            if (mask.maskMode !== MaskMode.NONE) {
                var b = maskBounds(mask);
                b.mask = mask;
                b.index = i;
                windows.push(b);
            }
        }
        return windows;
    }

    function groupWindows(windows, mode, tolerance, layerWidth) {
        var groups = [];
        var i, j, g;

        function makeGroups(axisKey) {
            var sorted = windows.slice();
            sorted.sort(function (a, b) { return a[axisKey] - b[axisKey]; });
            var out = [];
            var current = [sorted[0]];
            var ref = sorted[0][axisKey];
            for (i = 1; i < sorted.length; i++) {
                if (Math.abs(sorted[i][axisKey] - ref) <= tolerance) {
                    current.push(sorted[i]);
                } else {
                    out.push(current);
                    current = [sorted[i]];
                    ref = sorted[i][axisKey];
                }
            }
            out.push(current);
            return out;
        }

        if (mode === 1) {
            groups = makeGroups("cy");
        } else if (mode === 2) {
            groups = makeGroups("cx");
        } else if (mode === 3) {
            var used = [];
            for (i = 0; i < windows.length; i++) {
                if (used[i]) { continue; }
                var pair = [windows[i]];
                used[i] = true;
                for (j = i + 1; j < windows.length; j++) {
                    if (used[j]) { continue; }
                    if (Math.abs(windows[i].cy - windows[j].cy) <= tolerance &&
                        Math.abs(windows[i].cx + windows[j].cx - layerWidth) <= tolerance) {
                        pair.push(windows[j]);
                        used[j] = true;
                    }
                }
                groups.push(pair);
            }
        } else {
            for (i = 0; i < windows.length; i++) {
                groups.push([windows[i]]);
            }
        }

        for (i = 0; i < groups.length; i++) {
            g = groups[i];
            var cx = 0, cy = 0;
            for (j = 0; j < g.length; j++) {
                cx += g[j].cx;
                cy += g[j].cy;
            }
            g.centerX = cx / g.length;
            g.centerY = cy / g.length;
        }
        return groups;
    }

    function sortGroups(groups, direction) {
        var key;
        if (direction === DIR_TOP_TO_BOTTOM) { key = "centerY"; }
        else if (direction === DIR_BOTTOM_TO_TOP) { key = "centerY"; }
        else if (direction === DIR_LEFT_TO_RIGHT) { key = "centerX"; }
        else { key = "centerX"; }

        var sorted = groups.slice();
        sorted.sort(function (a, b) { return a[key] - b[key]; });
        if (direction === DIR_BOTTOM_TO_TOP || direction === DIR_RIGHT_TO_LEFT) {
            sorted.reverse();
        }
        return sorted;
    }

    function dirNormalized(groups, direction) {
        var axisKey = (direction <= DIR_BOTTOM_TO_TOP) ? "centerY" : "centerX";
        var values = [];
        var i;
        for (i = 0; i < groups.length; i++) { values.push(groups[i][axisKey]); }
        var min = values[0], max = values[0];
        for (i = 1; i < values.length; i++) {
            if (values[i] < min) { min = values[i]; }
            if (values[i] > max) { max = values[i]; }
        }
        var n = [];
        for (i = 0; i < values.length; i++) {
            n[i] = (max === min) ? 0 : (values[i] - min) / (max - min);
        }
        if (direction === DIR_BOTTOM_TO_TOP || direction === DIR_RIGHT_TO_LEFT) {
            for (i = 0; i < n.length; i++) { n[i] = 1 - n[i]; }
        }
        for (i = 0; i < groups.length; i++) {
            groups[i].norm = n[i];
        }
        return n;
    }

    function snap(t, frameDuration) {
        return Math.round(t / frameDuration) * frameDuration;
    }

    function clearAndSetKeys(prop, times, values) {
        while (prop.numKeys > 0) {
            prop.removeKey(prop.numKeys);
        }
        var i;
        for (i = 0; i < times.length; i++) {
            prop.setValueAtTime(times[i], values[i]);
        }
    }

    function blinkOnKeys(start, end, offset, onDuration, frameDuration) {
        return {
            t0: snap(start + offset, frameDuration),
            t1: snap(start + offset + onDuration, frameDuration)
        };
    }

    function generate(comp, layer, opts) {
        var windows = collectWindows(layer);
        if (windows.length === 0) {
            alert("Aucun masque actif trouve sur le calque selectionne.");
            return;
        }

        var effGroups = groupWindows(windows, opts.grouping, opts.tolerance, layer.width);
        var ordered = sortGroups(effGroups, opts.direction);
        dirNormalized(effGroups, opts.direction);

        var frameDuration = comp.frameDuration;
        var start = comp.workAreaStart;
        var end = start + comp.workAreaDuration;
        if (comp.duration < end) { end = comp.duration; }
        var T = 60 / opts.bpm;
        var onDuration = T * opts.duty / 100;

        var blinkLayer = layer.duplicate();
        blinkLayer.name = layer.name + " [Mapping Blinker]";
        blinkLayer.moveBefore(layer);
        blinkLayer.moveToBeginning();
        blinkLayer.selected = true;

        app.beginUndoGroup(SCRIPT_NAME);

        var masks = blinkLayer.property("Masks");
        var gi, ki, times, values, offset, skipped;

        for (gi = 0; gi < ordered.length; gi++) {
            var group = ordered[gi];
            for (ki = 0; ki < group.length; ki++) {
                var mask = group[ki].mask;
                if (mask === null) { continue; }
                var opProp = masks.property(group[ki].index + 1).property("ADBE Mask Opacity");
                if (opProp === null) { continue; }

                times = [];
                values = [];

                if (opts.mode === 0) {
                    for (var p = 0; start + p * T < end; p++) {
                        var k = blinkOnKeys(start, end, 0, onDuration, frameDuration);
                        times.push(k.t0); values.push(100);
                        times.push(k.t1); values.push(0);
                    }

                } else if (opts.mode === 1) {
                    var phase = seededRandom() * (T - onDuration);
                    skipped = seededRandom() < 0.15;
                    for (var q = 0; start + q * T < end; q++) {
                        times.push(snap(start + q * T, frameDuration)); values.push(0);
                        if (!skipped) {
                            var on0 = snap(start + q * T + phase, frameDuration);
                            var on1 = snap(start + q * T + phase + onDuration, frameDuration);
                            times.push(on0); values.push(100);
                            times.push(on1); values.push(0);
                        }
                    }

                } else if (opts.mode === 2) {
                    var slotStep = T;
                    var count = ordered.length;
                    times.push(snap(start, frameDuration)); values.push(0);
                    for (var s = 0; start + s * slotStep < end; s++) {
                        var activeIndex = s % count;
                        times.push(snap(start + s * slotStep, frameDuration)); values.push((gi === activeIndex) ? 100 : 0);
                        times.push(snap(start + (s + 1) * slotStep, frameDuration)); values.push(0);
                    }

                } else {
                    offset = (typeof group.norm === "number") ? group.norm * (T - onDuration) : 0;
                    for (var r = 0; start + r * T - offset < end + T; r++) {
                        var w0 = snap(start + r * T + offset, frameDuration);
                        var w1 = snap(start + r * T + offset + onDuration, frameDuration);
                        times.push(w0); values.push(100);
                        times.push(w1); values.push(0);
                    }
                }

                clearAndSetKeys(opProp, times, values);
            }
        }

        app.endUndoGroup();

        alert(SCRIPT_NAME + " :\n" + effGroups.length + " groupe(s) provenant de " + windows.length + " fenetre(s).\nBPM : " + opts.bpm.toFixed(1));
    }

    function parseNumber(text, fallback, min, max) {
        var v = parseFloat(text);
        if (isNaN(v)) { return fallback; }
        if (v < min) { v = min; }
        if (v > max) { v = max; }
        return v;
    }

    function onTapTempo(win, bpmField, status) {
        var now = new Date().getTime();
        if (tapTimes.length > 0 && (now - tapTimes[tapTimes.length - 1]) > 2500) {
            tapTimes = [];
        }
        tapTimes.push(now);
        if (tapTimes.length > 8) { tapTimes.shift(); }
        if (tapTimes.length >= 2) {
            var sum = 0;
            var i;
            for (i = 1; i < tapTimes.length; i++) {
                sum += (tapTimes[i] - tapTimes[i - 1]);
            }
            var avg = sum / (tapTimes.length - 1);
            if (avg > 60) {
                var bpm = 60000 / avg;
                bpmField.text = bpm.toFixed(1);
                status.text = "Tap : " + tapTimes.length + " tempo(s)";
            }
        } else {
            status.text = "Tap : continuez le rythme...";
        }
    }

    function buildUI(thisObj) {
        var win = (thisObj instanceof Panel) ? thisObj
            : new Window("palette", SCRIPT_NAME, undefined, { resizeable: true });

        win.orientation = "column";
        win.alignChildren = ["fill", "top"];
        win.spacing = 6;
        win.margins = 10;

        var bpmRow = win.add("group");
        bpmRow.orientation = "row";
        bpmRow.alignChildren = ["left", "center"];
        bpmRow.add("statictext", undefined, "BPM :");
        var bpmField = bpmRow.add("edittext", undefined, "120");
        bpmField.characters = 8;
        var bpmDown = bpmRow.add("button", undefined, "-");
        bpmDown.maximumSize = [24, 24];
        var bpmUp = bpmRow.add("button", undefined, "+");
        bpmUp.maximumSize = [24, 24];
        var tapBtn = bpmRow.add("button", undefined, "Tap");

        var status = win.add("statictext", undefined, "Tap tempo : Ctrl+Espace");
        status.characters = 34;

        win.add("statictext", undefined, "Mode de clignotement :");
        var modeDrop = win.add("dropdownlist", undefined,
            ["Unisson", "Aleatoire", "Sequentiel", "Vague"]);
        modeDrop.selection = 0;

        win.add("statictext", undefined, "Regroupement :");
        var groupDrop = win.add("dropdownlist", undefined,
            ["Fenetres separees", "Lignes", "Colonnes", "Symetrie"]);
        groupDrop.selection = 0;

        win.add("statictext", undefined, "Sens de propagation :");
        var dirDrop = win.add("dropdownlist", undefined,
            ["Haut - Bas", "Bas - Haut", "Gauche - Droite", "Droite - Gauche"]);
        dirDrop.selection = 0;

        var optRow = win.add("group");
        optRow.orientation = "row";
        optRow.alignChildren = ["left", "center"];
        optRow.add("statictext", undefined, "Ratio allume :");
        var dutyField = optRow.add("edittext", undefined, "50");
        dutyField.characters = 5;
        optRow.add("statictext", undefined, "%");

        var tolRow = win.add("group");
        tolRow.orientation = "row";
        tolRow.alignChildren = ["left", "center"];
        tolRow.add("statictext", undefined, "Tolerance :");
        var tolField = tolRow.add("edittext", undefined, "10");
        tolField.characters = 5;
        tolRow.add("statictext", undefined, "px");

        var seedRow = win.add("group");
        seedRow.orientation = "row";
        seedRow.alignChildren = ["left", "center"];
        seedRow.add("statictext", undefined, "Graine aleatoire :");
        var seedField = seedRow.add("edittext", undefined, "1");
        seedField.characters = 6;

        var info = win.add("statictext", undefined,
            "Selectionner le calque PNG masque, puis generer.", { truncate: true });
        info.characters = 34;

        var goBtn = win.add("button", undefined, "Generer le clignotement");
        goBtn.preferredSize = [-1, 32];

        win.addEventListener("keydown", function (kd) {
            if (kd.keyName === "Space" && (kd.ctrlKey || kd.metaKey)) {
                onTapTempo(win, bpmField, status);
                kd.preventDefault();
            }
        });
        tapBtn.onClick = function () {
            onTapTempo(win, bpmField, status);
        };

        bpmDown.onClick = function () {
            bpmField.text = (parseNumber(bpmField.text, 120, 10, 600) - 1).toFixed(1);
        };
        bpmUp.onClick = function () {
            bpmField.text = (parseNumber(bpmField.text, 120, 10, 600) + 1).toFixed(1);
        };

        goBtn.onClick = function () {
            if (!app.project || !isCompActive()) {
                alert("Ouvrez d'abord une composition.");
                return;
            }
            var comp = app.project.activeItem;
            var layer = getMappingLayer(comp);
            if (layer === null) {
                alert("Selectionnez un calque comportant au moins un masque de fenetre.");
                return;
            }

            rndState = Math.abs(parseInt(seedField.text, 10)) % 4294967296;
            if (isNaN(rndState) || rndState === 0) { rndState = 1; }

            var opts = {
                bpm: parseNumber(bpmField.text, 120, 10, 600),
                duty: parseNumber(dutyField.text, 50, 5, 95),
                mode: modeDrop.selection ? modeDrop.selection.index : 0,
                grouping: groupDrop.selection ? groupDrop.selection.index : 0,
                direction: dirDrop.selection ? dirDrop.selection.index : 0,
                tolerance: parseNumber(tolField.text, 10, 1, 1000)
            };

            generate(comp, layer, opts);
        };

        win.layout.layout(true);
        win.layout.resize();
        win.onResizing = win.onResize = function () {
            this.layout.resize();
        };

        return win;
    }

    var ui = buildUI(thisObj);
    if (ui instanceof Window) {
        ui.center();
        ui.show();
    }

})(this);
