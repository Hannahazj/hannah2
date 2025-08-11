// 6) Now the Board_<PaneName> sub‐folder
const boardFolder = thisPaneFolder.folder(`Board_${paneName}`);
boardFolder.file("__init__.py", "");

// 7) functions.py + board.py (concat all results)
const functionsCode = Object.entries(dataWF)
  .filter(([k]) => k.endsWith('/functions.py'))
  .map(([k,v]) => v).join("\n\n");
const boardCode = Object.entries(dataWF)
  .filter(([k]) => k.endsWith('/board.py'))
  .map(([k,v]) => v).join("\n\n");

boardFolder.file("functions.py", functionsCode);
boardFolder.file("board.py",      boardCode);


// 6) Create one output folder per returned board package
const boardKeys = Object.keys(dataWF).filter(k => k.startsWith('Board_'));

// group by folder prefix, e.g. "Board_sales" => ["Board_sales/board.py", ...]
const byFolder = {};
for (const key of boardKeys) {
  const slash = key.indexOf('/');
  if (slash === -1) continue;
  const folder = key.slice(0, slash);     // e.g. "Board_sales"
  byFolder[folder] = byFolder[folder] || [];
  byFolder[folder].push(key);
}

// For each board folder, write its files exactly as returned
for (const folder of Object.keys(byFolder)) {
  const out = thisPaneFolder.folder(folder);

  // write standard files if present; ensure __init__.py exists
  out.file("__init__.py", dataWF[`${folder}/__init__.py`] || "");
  if (dataWF[`${folder}/board.py`]) {
    out.file("board.py", dataWF[`${folder}/board.py`]);
  }
  if (dataWF[`${folder}/functions.py`]) {
    out.file("functions.py", dataWF[`${folder}/functions.py`]);
  }

  // write any additional files under that board folder
  for (const key of byFolder[folder]) {
    const name = key.slice(folder.length + 1); // strip "Board_xxx/"
    if (!["board.py", "functions.py", "__init__.py"].includes(name)) {
      out.file(name, dataWF[key]);
    }
  }
}
