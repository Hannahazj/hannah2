async function exportJson() {
  const zip = new JSZip();

  // 1) Top-level Python folder
  const pyFolder = zip.folder("Python");

  // 1a) Include framework.py once at the root of Python/
  try {
    const respF = await fetch('/Python_Files/framework.py');
    const frameworkCode = await respF.text();
    pyFolder.file("framework.py", frameworkCode);
  } catch (e) {
    console.warn("framework.py fetch failed; continuing without it.", e);
    pyFolder.file("framework.py", ""); // keep structure predictable
  }

  // 2) Iterate each visual pane
  for (let idx = 0; idx < networkPanes.length; idx++) {
    const pane = networkPanes[idx];

    // 2a) Pane name from tab text → sanitized
    const tab = document.querySelector(`.tab[data-index="${idx}"]`);
    let paneName = (tab && tab.textContent.trim()) || `pane${idx + 1}`;
    paneName = paneName.replace(/\W+/g, '_').replace(/^(\d)/, '_$1');

    // 2b) Create pane folder
    const thisPaneFolder = pyFolder.folder(paneName);

    // 2c) Dump graph JSON
    const graph = getGraphJson(pane);
    const jsonStr = JSON.stringify(graph, null, 2);
    thisPaneFolder.file("json.canvas", jsonStr);

    // 2d) Pane-level __init__.py
    thisPaneFolder.file("__init__.py", "");

    // 2e) Ask backend to generate code for this pane
    let dataWF = {};
    try {
      const respWF = await fetch('/generate', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ json: graph, flowName: paneName })
      });
      dataWF = await respWF.json();
    } catch (e) {
      console.error("Code generation failed for pane:", paneName, e);
      dataWF = {};
    }

    // 2f) Write workflow.py if present
    const workflowCode = dataWF['workflow.py'] || "";
    thisPaneFolder.file("workflow.py", workflowCode);

    // 2g) MULTI-BOARD OUTPUT: create one folder per returned Board_*
    const boardKeys = Object.keys(dataWF).filter(k => k.startsWith('Board_'));

    // group by folder prefix, e.g. "Board_sales" => ["Board_sales/board.py", ...]
    const byFolder = {};
    for (const key of boardKeys) {
      const slash = key.indexOf('/');
      if (slash === -1) continue; // safety
      const folder = key.slice(0, slash); // e.g. "Board_sales"
      (byFolder[folder] ||= []).push(key);
    }

    // write each board's files into its own subfolder
    for (const folder of Object.keys(byFolder)) {
      const out = thisPaneFolder.folder(folder);

      // standard files; ensure __init__.py exists
      out.file("__init__.py", dataWF[`${folder}/__init__.py`] || "");
      if (dataWF[`${folder}/board.py`]) {
        out.file("board.py", dataWF[`${folder}/board.py`]);
      }
      if (dataWF[`${folder}/functions.py`]) {
        out.file("functions.py", dataWF[`${folder}/functions.py`]);
      }

      // future-proof: write any extras returned under that board folder
      for (const key of byFolder[folder]) {
        const name = key.slice(folder.length + 1); // strip "Board_xxx/"
        if (!["board.py", "functions.py", "__init__.py"].includes(name)) {
          out.file(name, dataWF[key]);
        }
      }
    }
  }

  // 3) Zip it up and download
  const blob = await zip.generateAsync({ type: "blob" });
  const filename = (projectName || "exported_graphs_and_code") + ".zip";
  saveAs(blob, filename);
}
