script.js "var networkPanes = [];
var currentActiveIndex = 0;
var paneCount = 1; // starting with one pane at index 0

// ------------------------------
// Global Helper Variables & Functions
// ------------------------------

const transparentColor = {
  border: '#2B7CE9',
  background: 'rgba(0, 0, 0, 0)',
  highlight: { border: '#2B7CE9', background: 'rgba(0,0,0,0)' },
  hover: { border: '#2B7CE9', background: 'rgba(0,0,0,0)' },
  selected: { border: '#2B7CE9', background: 'rgba(0,0,0,0)' }
};



const obsidianColorMap = {
  "1": "#e93147", // Red
  "2": "#08b94e", // Green
  "3": "#086ddd", // Blue
  "4": "#e0ac00", // Yellow
  "5": "#7852ee", // Purple
  "6": "#00bfbc"  // Cyan
};

/**
 * Normalize incoming color values so importJson can always
 * pick a valid border color.
 */
function getConvertedColor(raw) {
  // 1) If it’s already a Vis.js color‐object
  if (typeof raw === 'object' && raw.border) {
    return raw.border;
  }
  // 2) If it’s one of your obsidian map keys
  if (typeof raw === 'string' && obsidianColorMap[raw]) {
    return obsidianColorMap[raw];
  }
  // 3) If it’s any other CSS color string
  if (typeof raw === 'string') {
    return raw;
  }
  // 4) Default
  return '#2B7CE9';
}



const MIN_ALPHA = 0.05;   // absolute minimum opacity
const MAX_ALPHA = 0.20;   // absolute maximum opacity

/** @returns {r,g,b} from a “#RRGGBB” string */
function hexToRgb(hex) {
  const clean = hex.replace(/^#/, '');
  const num   = parseInt(clean, 16);
  return {
    r: (num >> 16) & 0xFF,
    g: (num >>  8) & 0xFF,
    b: num        & 0xFF
  };
}

/**
 * Given r,g,b 0–255, return a brightness 0–1
 * (simple average; you can swap in a perceptual formula if you like)
 */
function getBrightness({r, g, b}) {
  return (r + g + b) / (3 * 255);
}

/**
 * Build a Vis.js color object whose fill alpha scales
 * inversely with brightness so it’s always legible.
 */
function normalizeColor(colorStr) {
  const ctx = document.createElement("canvas").getContext("2d");
  ctx.fillStyle = colorStr;
  return ctx.fillStyle;   // e.g. "#00ff00"
}

/**
 * Build a Vis.js color object whose fill alpha scales inversely with brightness
 * so it’s always legible, and *normalize* any CSS names first.
 */
function makeGroupColor(rawColor) {
  // 1) normalize any CSS name/rgb()/hex → "#rrggbb"
  const borderHex = normalizeColor(rawColor);

  // 2) parse it to RGB
  const clean = borderHex.replace(/^#/, '');
  const num   = parseInt(clean, 16);
  const rgb   = {
    r: (num >> 16) & 0xFF,
    g: (num >>  8) & 0xFF,
    b: num        & 0xFF
  };

  // 3) compute brightness & alpha
  const brightness = (rgb.r + rgb.g + rgb.b) / (3 * 255);
  const alpha      = MIN_ALPHA + (1 - brightness) * (MAX_ALPHA - MIN_ALPHA);
  const bg         = `rgba(${rgb.r},${rgb.g},${rgb.b},${alpha.toFixed(3)})`;

  return {
    border:    borderHex,
    background: bg,
    highlight: { border: borderHex, background: bg },
    hover:     { border: borderHex, background: bg },
    selected:  { border: borderHex, background: bg }
  };
}

// A helper to prompt for an edge label via a dropdown.
function promptEdgeLabel(callback) {
  var modal = document.createElement("div");
  modal.style.position = "fixed";
  modal.style.top = "0";
  modal.style.left = "0";
  modal.style.width = "100%";
  modal.style.height = "100%";
  modal.style.backgroundColor = "rgba(0,0,0,0.5)";
  modal.style.display = "flex";
  modal.style.alignItems = "center";
  modal.style.justifyContent = "center";
  modal.style.zIndex = "1000";

  var containerDiv = document.createElement("div");
  containerDiv.style.backgroundColor = "#fff";
  containerDiv.style.padding = "20px";
  containerDiv.style.borderRadius = "5px";
  containerDiv.style.boxShadow = "0 2px 10px rgba(0,0,0,0.3)";
  containerDiv.style.textAlign = "center";

  var label = document.createElement("label");
  label.textContent = "Select edge label:";
  containerDiv.appendChild(label);

  var select = document.createElement("select");
  select.style.marginLeft = "10px";
  var optionTrue = document.createElement("option");
  optionTrue.value = "true";
  optionTrue.textContent = "true";
  select.appendChild(optionTrue);
  var optionFalse = document.createElement("option");
  optionFalse.value = "false";
  optionFalse.textContent = "false";
  select.appendChild(optionFalse);
  containerDiv.appendChild(select);

  var button = document.createElement("button");
  button.textContent = "OK";
  button.style.marginLeft = "10px";
  button.onclick = function() {
    var value = select.value;
    document.body.removeChild(modal);
    callback(value);
  };
  containerDiv.appendChild(button);

  modal.appendChild(containerDiv);
  document.body.appendChild(modal);
}

// code added for modal
let projectName = '';

window.addEventListener('DOMContentLoaded', () => {
  const modal = document.getElementById('projectModal');
  const input = document.getElementById('projectNameInput');
  const btn   = document.getElementById('createProjectBtn');

  // show modal
  modal.style.display = 'flex';

  btn.addEventListener('click', () => {
    projectName = input.value.trim() || 'MyProject';
    modal.style.display = 'none';
    document.getElementById('projectDisplay').textContent = projectName;
  });
});


// ------------------------------
// Functions automatic creation of groupnodes
// --

function autoGroupNodes(pane) {
  // gather the “real” (non-group, non-temp) nodes
  const real = pane.nodes.get().filter(n => n.type !== 'group' && !n.id.startsWith('temp_'));

  // find any existing auto-group(s)
  const existing = pane.nodes.get().filter(n =>
    n.type === 'group' && n.label === 'Temp group node'
  );

  // find a "hub" node with 2+ outgoing edges among real nodes
  const outgoingCounts = {};
  pane.edges.get().forEach(e => {
    if (real.some(n => n.id === e.from) && real.some(n => n.id === e.to)) {
      outgoingCounts[e.from] = (outgoingCounts[e.from] || 0) + 1;
    }
  });

  // find a node with 2+ outgoing edges
  const hubNodeId = Object.keys(outgoingCounts).find(id => outgoingCounts[id] >= 2);

  // Only create the group if:
  // - at least 3 real nodes exist
  // - a hub node with 2+ outgoing edges exists
  if (real.length >= 3 && hubNodeId) {
    // only create if one doesn’t already exist
    if (existing.length) return;

    // —— compute bounding box of real[] as you already do ——
    let minX = Infinity, minY = Infinity, maxX = -Infinity, maxY = -Infinity;
    real.forEach(n => {
      const bb = pane.network.getBoundingBox(n.id)
                || { left: n.x, top: n.y, right: n.x, bottom: n.y };
      minX = Math.min(minX, bb.left);
      minY = Math.min(minY, bb.top);
      maxX = Math.max(maxX, bb.right);
      maxY = Math.max(maxY, bb.bottom);
    });
    const P = 50;
    minX -= P; minY -= P; maxX += P; maxY += P;
    const width  = maxX - minX;
    const height = maxY - minY;
    const centerX = (minX + maxX)/2;
    const centerY = (minY + maxY)/2;

    // add exactly one
    const groupId = 'temp_' + Date.now();
    pane.nodes.add({
      id:       groupId,
      type:     'group',
      label:    'Temp group node',
      x:        centerX,
      y:        centerY,
      shape:    'box',
      widthConstraint:  { minimum: width,  maximum: width  },
      heightConstraint: { minimum: height, maximum: height },
      color:    makeGroupColor('#2B7CE9'),
      font:     { size: 40, align: 'left', vadjust: -height/2 + 20 },
      margin:   { left: 20 }
    });

    real.forEach(n => pane.nodes.update({ id: n.id, group: groupId }));
    updateGroupNodeSize(groupId, pane);
  }
  else {
    // if we’ve dropped below 3 real nodes OR hub disappeared, clean up any leftover Temp group
    existing.forEach(g => pane.nodes.remove(g.id));
  }
}

// ------------------------------
// Functions for network pane operations
// ------------------------------

function isGroupNode(nodeId, pane) {
  const node = pane.nodes.get(nodeId);
  // Here we assume group nodes have type 'group'
  return node && node.type === 'group';
}

function selectNodesFromRectangle(rect, pane) {
  var scale = pane.network.getScale();
  var offset = pane.network.getViewPosition();
  var selectedNodes = pane.nodes.get().filter(function (node) {
    var nodePosition = pane.network.getPositions([node.id])[node.id];
    var nodeScreenPosition = {
      x: (nodePosition.x - offset.x) * scale + pane.container.clientWidth / 2,
      y: (nodePosition.y - offset.y) * scale + pane.container.clientHeight / 2
    };
    return (
      nodeScreenPosition.x >= rect.x &&
      nodeScreenPosition.x <= rect.x + rect.width &&
      nodeScreenPosition.y >= rect.y &&
      nodeScreenPosition.y <= rect.y + rect.height
    );
  });
  pane.network.selectNodes(selectedNodes.map(node => node.id));
}

function updateGroupNodeSize(groupNodeId, pane) {
  const groupNode = pane.nodes.get(groupNodeId);
  if (!groupNode) return;
  const childNodes = pane.nodes.get().filter(n => n.group === groupNodeId);
  if (childNodes.length === 0) return;
  let minX = Infinity, minY = Infinity, maxX = -Infinity, maxY = -Infinity;
  childNodes.forEach(child => {
    let childBB = pane.network.getBoundingBox(child.id);
    if (!childBB) {
      const pos = pane.network.getPositions([child.id])[child.id];
      childBB = { left: pos.x, top: pos.y, right: pos.x, bottom: pos.y };
    }
    minX = Math.min(minX, childBB.left);
    minY = Math.min(minY, childBB.top);
    maxX = Math.max(maxX, childBB.right);
    maxY = Math.max(maxY, childBB.bottom);
  });
  const padding = 20;
  minX -= padding;
  minY -= padding;
  maxX += padding;
  maxY += padding;
  const textPadding = 20;
  const estimatedCharWidth = 8;
  let textWidth = groupNode.label ? groupNode.label.length * estimatedCharWidth + textPadding : 0;
  let currentWidth = maxX - minX;
  if (textWidth > currentWidth) {
    const diff = textWidth - currentWidth;
    minX -= diff / 2;
    maxX += diff / 2;
  }
  const width = maxX - minX;
  const height = maxY - minY;
  const centerX = (minX + maxX) / 2;
  const centerY = (minY + maxY) / 2;
  groupNode.x = centerX;
  groupNode.y = centerY;
  groupNode.widthConstraint = { minimum: width, maximum: width };
  groupNode.heightConstraint = { minimum: height, maximum: height };
  pane.nodes.update(groupNode);
}

function keepNodeWithinGroup(childNodeId, groupNodeId, pane) {
  const groupNode = pane.nodes.get(groupNodeId);
  if (!groupNode) return;
  const groupPos = pane.network.getPositions([groupNodeId])[groupNodeId];
  const childPos = pane.network.getPositions([childNodeId])[childNodeId];
  if (!childPos || !groupPos) return;

  if (isGroupNode(childNodeId, pane)) {
    const GROUP_OFFSET_X = 60;
    const GROUP_OFFSET_Y = 109;
    const desiredX = groupPos.x + GROUP_OFFSET_X;
    const desiredY = groupPos.y + GROUP_OFFSET_Y;
    const dx = childPos.x - desiredX;
    const dy = childPos.y - desiredY;
    const threshold = 50;
    if (Math.abs(dx) > threshold || Math.abs(dy) > threshold) {
      pane.network.moveNode(childNodeId, desiredX, desiredY);
    }
  } else {
    const groupBB = pane.network.getBoundingBox(groupNodeId);
    const groupWidth = groupBB.right - groupBB.left;
    const groupHeight = groupBB.bottom - groupBB.top;
    const halfW = groupWidth / 2;
    const halfH = groupHeight / 2;
    const left = groupPos.x - halfW;
    const right = groupPos.x + halfW;
    const top = groupPos.y - halfH;
    const bottom = groupPos.y + halfH;
    if (childPos.x < left) childPos.x = left;
    if (childPos.x > right) childPos.x = right;
    if (childPos.y < top) childPos.y = top;
    if (childPos.y > bottom) childPos.y = bottom;
    pane.network.moveNode(childNodeId, childPos.x, childPos.y);
  }
}



// function to dynamically update sizes of node
function updateAllNodeFontSizes(pane) {
  const minFont = 18, maxFont = 48, minW = 600, maxW = 2000;
  const w = window.innerWidth;
  const fontSize = Math.round(minFont + ((Math.min(Math.max(w, minW), maxW) - minW) / (maxW - minW)) * (maxFont - minFont));
  pane.nodes.get().forEach(node => {
    if (node.label) {
      pane.nodes.update({ id: node.id, font: { ...node.font, size: fontSize } });
    }
  });
}


// ------------------------------
// Template Graph Data (fixed to valid JSON)
// ------------------------------

var templateGraph = {
  "nodes": [
    {
      "id": "3e7b8acd417adc40",
      "type": "group",
      "x": -582,
      "y": -258,
      "width": 640,
      "height": 461,
      "label": "Flow"
    },
    {
      "id": "04f402c4e2af7f56",
      "type": "group",
      "x": -562,
      "y": -98,
      "width": 595,
      "height": 281,
      "color": "2",
      "label": "Board_Name"
    },
    {
      "id": "6d1ef69553b877d9",
      "type": "text",
      "label": "Actions_True",
      "x": -542,
      "y": 103,
      "width": 250,
      "height": 60,
      "color": "#f6f4f4"
    },
    {
      "id": "214991ab8cafdfbc",
      "type": "text",
      "label": "Actions_False",
      "x": -237,
      "y": 103,
      "width": 250,
      "height": 60,
      "color": "#f4f0f0"
    },
    {
      "id": "01c0c38aeb5e5e13",
      "type": "text",
      "label": "Comments",
      "x": -262,
      "y": -238,
      "width": 300,
      "height": 100,
      "color": "4"
    },
    {
      "id": "6babf3007067a1e7",
      "type": "text",
      "label": "Function_Name",
      "x": -482,
      "y": -78,
      "width": 380,
      "height": 80,
      "color": "3"
    }
  ],
  "edges": [
    {
      "id": "37f808b16516c77e",
      "fromNode": "6babf3007067a1e7",
      "fromSide": "bottom",
      "toNode": "6d1ef69553b877d9",
      "toSide": "top",
      "color": "4",
      "label": "True"
    },
    {
      "id": "b5e7f2a787970ec8",
      "fromNode": "6babf3007067a1e7",
      "fromSide": "bottom",
      "toNode": "214991ab8cafdfbc",
      "toSide": "top",
      "color": "1",
      "label": "False"
    }
  ]
};

function createTemplate() {
  const pane = networkPanes[currentActiveIndex];
  pane.nodes.clear();
  pane.edges.clear();

  const fontSize = 40;
  const extraPad = 10;

  // 1) Precompute parent‐group bounds
  const groupsJSON = templateGraph.nodes
    .filter(n => n.type === 'group')
    .map(n => ({
      id:      n.id,
      left:    n.x,
      top:     n.y,
      right:   n.x + n.width,
      bottom:  n.y + n.height
    }));

  // 2) Add all nodes (groups first in your JSON order)
  templateGraph.nodes.forEach(node => {
    const isGroup = node.type === 'group';
    const w = node.width  || 0;
    const h = node.height || 0;
    const x = node.x + w/2;
    const y = node.y + h/2;

    // pick a border color
    let borderColor = typeof node.color === 'string'
      ? (obsidianColorMap[node.color] || node.color)
      : '#2B7CE9';

    // build the color object
    const colorObj = isGroup
      ? makeGroupColor(borderColor)
      : {
          border:     borderColor,
          background: 'rgba(0,0,0,0)',
          highlight:  { border: borderColor, background: 'rgba(0,0,0,0)' },
          hover:      { border: borderColor, background: 'rgba(0,0,0,0)' },
          selected:   { border: borderColor, background: 'rgba(0,0,0,0)' }
        };

    // assemble base opts
    const opts = {
      id:      node.id,
      label:   node.label || node.text || '',
      x, y,
      shape:   'box',
      type:    node.type,
      color:   colorObj,
      widthConstraint:  node.width  ? { minimum: w, maximum: w } : undefined,
      heightConstraint: node.height ? { minimum: h, maximum: h } : undefined
    };

    // if it’s a group, style the font
    if (isGroup) {
      const vadjust = -h/2 + fontSize/2 + extraPad;
      opts.font   = { size: fontSize, align: 'left', vadjust };
      opts.margin = { left: extraPad };
    }

    // 3) **Nesting**: anyone sitting fully inside another group
    for (const parent of groupsJSON) {
      if (parent.id !== node.id &&
          node.x       >= parent.left   &&
          node.x + w   <= parent.right  &&
          node.y       >= parent.top    &&
          node.y + h   <= parent.bottom) {
        opts.group = parent.id;
        break;
      }
    }

    pane.nodes.add(opts);
  });

  // 4) Add edges unchanged
  templateGraph.edges.forEach(edge => {
    const edgeColor = obsidianColorMap[edge.color] || edge.color || 'black';
    pane.edges.add({
      id:     edge.id,
      from:   edge.fromNode,
      to:     edge.toNode,
      label:  edge.label,
      arrows: 'to',
      color:  edgeColor
    });
  });
}

// ------------------------------
// New Function: renamePane()
// ------------------------------
/**
 * Rename a pane by updating the text content on its associated tab.
 * If the pane index is not provided, it renames the current active pane.
 *
 * @param {number} [index] - The pane index to rename.
 * @param {string} newName - The new name for the pane.
 */
function renamePane(index, newName) {
  if (index === undefined || index === null) {
    index = currentActiveIndex;
  }
  var tab = document.querySelector('.tab[data-index="' + index + '"]');
  if (tab) {
    tab.textContent = newName;
  } else {
    console.error("Tab for pane index " + index + " not found.");
  }
}

// ------------------------------
// Export/Import Functions (unchanged)
// ------------------------------

async function exportJson() {
  const zip = new JSZip();

  // 1) Single top‐level Python folder
  const pyFolder = zip.folder("Python");

  for (let idx = 0; idx < networkPanes.length; idx++) {
    const pane = networkPanes[idx];

    // Determine the pane‐name from the tab text, fallback to "pane<idx>"
    const tab = document.querySelector(`.tab[data-index="${idx}"]`);
    let paneName = tab && tab.textContent.trim() || `pane${idx+1}`;

    // Sanitize: non‐word → underscore, leading digit → underscore+digit
    paneName = paneName
      .replace(/\W+/g, '_')
      .replace(/^(\d)/, '_$1');

    // 2) Create the folder for this pane
    const thisPaneFolder = pyFolder.folder(paneName);

    // 3) Dump the JSON into json.canvas
    const graph = getGraphJson(pane);
    const jsonStr = JSON.stringify(graph, null, 2);
    thisPaneFolder.file("json.canvas", jsonStr);

    // 4) Blank __init__.py
    thisPaneFolder.file("__init__.py", "");

    // 5) workflow.py
    //    (you may want to reuse whatever string you already generate in showRepresentation)
    //    Here we'll fetch it on the fly:
    const respWF = await fetch('/generate', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ json: graph, flowName: paneName })
    });
    const dataWF = await respWF.json();
    const workflowCode = dataWF['workflow.py'] || "";
    thisPaneFolder.file("workflow.py", workflowCode);

    // 6) Now the Board_<PaneName> sub‐folder
    const boardFolder = thisPaneFolder.folder(`Board_${paneName}`);
    boardFolder.file("__init__.py", "");

    // 7) functions.py + board.py
    //    We fetch the same generation and split out the two parts:
    //    (or you could call your existing slicing routine directly)
    const functionsCode = Object.entries(dataWF)
      .filter(([k]) => k.endsWith('/functions.py'))
      .map(([k,v]) => v).join("\n\n");
    const boardCode = Object.entries(dataWF)
      .filter(([k]) => k.endsWith('/board.py'))
      .map(([k,v]) => v).join("\n\n");

    boardFolder.file("functions.py", functionsCode);
    boardFolder.file("board.py",      boardCode);
  }

  // 8) Generate and save
  const blob = await zip.generateAsync({ type: "blob" });
  const filename = (projectName || "exported_graphs_and_code") + ".zip";
  saveAs(blob, filename);
}


function importJson() {
  const pane = networkPanes[currentActiveIndex];
  const input = document.getElementById("importFile");
  if (!input.files.length) {
    return alert("Please select a JSON file to import.");
  }

  const reader = new FileReader();
  reader.onload = function(e) {
    const json = JSON.parse(e.target.result);
    const importedNodes = json.nodes || [];
    const importedEdges = json.edges || [];

    // 1) Clear everything out
    pane.nodes.clear();
    pane.edges.clear();

    // 2) Re-add all nodes
    pane.nodes.add(importedNodes.map(node => {
      const isGroup = node.type === 'group';
      const borderColor = getConvertedColor(node.color) || '#2B7CE9';
      const colorObj = isGroup ? makeGroupColor(borderColor) : {
        border: borderColor,
        background: 'rgba(0,0,0,0)',
        highlight: { border: borderColor, background: 'rgba(0,0,0,0)' },
        hover:     { border: borderColor, background: 'rgba(0,0,0,0)' },
        selected:  { border: borderColor, background: 'rgba(0,0,0,0)' }
      };

      const opts = {
        id:    node.id,
        label: node.label || node.text || "",
        x:     node.x,
        y:     node.y,
        shape: 'box',
        type:  node.type,
        color: colorObj
      };

      if (isGroup) {
        opts.widthConstraint  = { minimum: node.width,  maximum: node.width };
        opts.heightConstraint = { minimum: node.height, maximum: node.height };
        opts.font   = { size: 40, align: 'left', vadjust: -node.height/2 + 20 };
        opts.margin = { left: 20 };
      }

      return opts;
    }));

    // 3) Wire up nesting: give each non-group its `group` prop
    const groups = importedNodes.filter(n => n.type === 'group');
    importedNodes
      .filter(n => n.type !== 'group')
      .forEach(n => {
        let best = null, bestArea = Infinity;
        groups.forEach(g => {
          const left   = g.x - g.width/2;
          const right  = g.x + g.width/2;
          const top    = g.y - g.height/2;
          const bottom = g.y + g.height/2;
          if (n.x >= left && n.x <= right && n.y >= top && n.y <= bottom) {
            const area = g.width * g.height;
            if (area < bestArea) {
              bestArea = area;
              best = g.id;
            }
          }
        });
        if (best) pane.nodes.update({ id: n.id, group: best });
      });

    // 4) Filter edges by your four rules
    const validEdges = importedEdges.filter(edge => {
      const from = pane.nodes.get(edge.fromNode);
      const to   = pane.nodes.get(edge.toNode);

      // a) must stay in the same group
      const fg = from.group||null, tg = to.group||null;
      if (fg !== tg) return false;

      // b) no more than 2 outgoing
      if (pane.edges.get().filter(e=>e.from===edge.fromNode).length >= 2) return false;

      // c) no more than 1 incoming
      if (pane.edges.get().filter(e=>e.to  ===edge.toNode).length >= 1) return false;

      // d) no reverse (cycle)
      if (
        importedEdges.some(e=>e.fromNode===edge.toNode && e.toNode===edge.fromNode) ||
        pane.edges.get().some(e=>e.from===edge.toNode && e.to===edge.fromNode)
      ) return false;

      return true;
    });

    // 5) Add only the valid edges
    pane.edges.add(validEdges.map(edge=>({
      id:     edge.id,
      from:   edge.fromNode,
      to:     edge.toNode,
      label:  edge.label||"",
      arrows: 'to',
      color:  getConvertedColor(edge.color)||'black'
    })));

    // 6) Recompute sizes on every group (and its ancestors)
    importedNodes
      .filter(n=>n.type==='group')
      .forEach(n=>{
        updateGroupNodeSize(n.id, pane);
        let p = pane.nodes.get(n.id).group;
        while(p){
          updateGroupNodeSize(p, pane);
          p = pane.nodes.get(p).group;
        }
      });

    // 7) Refresh the side-panel
    showRepresentation();
  };

  reader.readAsText(input.files[0]);
}


// ------------------------------
// Factory function: Create a new network pane instance.
// ------------------------------

function createNetworkPane(index) {
  // Create new DataSets for this pane.
  var nodes = new vis.DataSet();
  var edges = new vis.DataSet();
  var data = { nodes: nodes, edges: edges };

  // Get the container for this pane based on its unique ID.
  var container = document.getElementById('mynetwork-' + index);
  // Make the container focusable so key events always work.
  container.tabIndex = 0;
  container.style.outline = "none";

  // Focus the container when clicked.
  container.addEventListener("click", function() {
    container.focus();
  });

  // Reference to the global selectionBox element (shared across panes).
  var selectionBox = document.getElementById('selectionBox');
  var startX, startY, dragging = false;
  var resizingNode = null, resizeStartX = 0, resizeStartY = 0, initialWidth = 0, initialHeight = 0;

  var options = {
    manipulation: {
      enabled: true,
      addNode: function (data, callback) {
        data.label = prompt('Enter node label:', 'New Node');
        var nodeColor = '#2B7CE9';
        data.color = {
          border: nodeColor,
          background: 'rgba(0,0,0,0)',
          highlight: { border: nodeColor, background: 'rgba(0,0,0,0)' },
          hover: { border: nodeColor, background: 'rgba(0,0,0,0)' },
          selected: { border: nodeColor, background: 'rgba(0,0,0,0)' },
        };
        callback(data);
        updateAllNodeFontSizes({network, nodes, edges, container});
      },
      editNode: function (data, callback) { callback(null); },
      editEdge: function (data, callback) { callback(null); },
      deleteNode: function (data, callback) { callback(null); },
      deleteEdge: function (data, callback) { callback(null); },
      addEdge: function(data, callback) {
      var fromNode = nodes.get(data.from);
      var toNode   = nodes.get(data.to);

      // 1) No self-loops
      if (data.from === data.to) {
        alert("Self-loops are not allowed.");
        return callback(null);
      }

      // 2) If either node is in a group, enforce same-group
      if (fromNode.group || toNode.group) {
        if (!fromNode.group || !toNode.group || fromNode.group !== toNode.group) {
          alert("Edges must stay within the same group.");
          return callback(null);
        }
      }

      // 3) Outgoing edges cap at 2
      const outgoing = edges.get().filter(e => e.from === data.from);
      if (outgoing.length >= 2) {
        alert("This node already has two outgoing edges.");
        return callback(null);
      }
      if (outgoing.length === 1) {
        // toggle the second edge’s label/color
        const existing = outgoing[0].label;
        data.label = (existing === "true") ? "false" : "true";
        data.color = (data.label === "true") ? "green" : "red";
        return callback(data);
      }

      // 4) **Incoming must be ≤1** (fixed)
      const incoming = edges.get().filter(e => e.to === data.to);
      if (incoming.length >= 1) {
        alert("A node cannot have more than one incoming edge.");
        return callback(null);
      }

      // 5) No direct reverse (cycle)
      const reverseExists = edges.get().some(e => e.from === data.to && e.to === data.from);
      if (reverseExists) {
        alert("That would create a cycle.");
        return callback(null);
      }

      // 6) All good → prompt label
      promptEdgeLabel(value => {
        data.label = value;
        data.color = (value === "true") ? "green" : "red";

        const pane = networkPanes[currentActiveIndex];

        // 1) keep the original A→B edge
        callback(data);

        // 2) create the TempNode further down
        const tempId = 'temp_' + Date.now();
        const fromNode = pane.nodes.get(data.from);
        pane.nodes.add({
          id:    tempId,
          label: 'TempNode',
          shape: 'box',
          x:     fromNode.x,               // same horizontal position
          y:     fromNode.y + 150,         // 150px further down
          color: transparentColor
        });

        // 3) label/color of A→Temp should be the opposite of data.label
        const tempLabel = data.label === "true" ? "false" : "true";
        const tempColor = tempLabel === "true" ? "green" : "red";
        pane.edges.add({
          from:   data.from,
          to:     tempId,
          label:  tempLabel,
          arrows: 'to',
          color:  tempColor
        });

        // 4) auto-group if you now have ≥3 real nodes
        autoGroupNodes(pane);
      });
    }
    },
    nodes: {
      shape: 'box',
      size: 20,
      font: { size: 40 },
      color: transparentColor,
    borderWidth: 4,            // ← normal thickness
    borderWidthSelected: 6     // ← when you click/select it
    },
    edges: {
              font: {
        size: 40,
              },
      smooth: {
        enabled: true,
        type: 'continuous',
        roundness: 0.5
      },
      arrowStrikethrough: false,
      arrows: { to: { enabled: true, type: 'arrow' } }
    },
    interaction: {
      multiselect: true,
      navigationButtons: true,
      dragNodes: true,
      dragView: true,
      zoomView: true,
      selectable: true,
      selectConnectedEdges: true,
      hoverConnectedEdges: true
    },
    physics: { enabled: false }
  };

  var network = new vis.Network(container, data, options);

  // update side panel on the right
  nodes.on('add',    () => showRepresentation());
  nodes.on('update', () => showRepresentation());
  nodes.on('remove', () => showRepresentation());
  edges.on('add',    () => showRepresentation());
  edges.on('update', () => showRepresentation());
  edges.on('remove', () => showRepresentation());



      // ——— ADD THIS ———
  const pane = { network, nodes, edges, container };

    // whenever an edge *or* node is removed, re-run auto-group
    pane.edges.on('remove', () => autoGroupNodes(pane));
    pane.nodes.on('remove', () => autoGroupNodes(pane));

     // whenever an edge *or* node is add, re-run auto-group
    pane.edges.on('add',    () => autoGroupNodes(pane));
    pane.nodes.on('add',    () => autoGroupNodes(pane));

    // functionality for highlighting on the right
    network.on('selectNode', params => {
  highlightTree(params.nodes[0]);
});

// call update font size function
updateAllNodeFontSizes({network, nodes, edges, container});


network.on('deselectNode', () => {
  highlightTree(null);
});


  // also update after dragging (so positions refresh)
  network.on('dragEnd', () => showRepresentation());

  // Helper: Show custom resize menu.
  function showResizeMenu(nodeId, clientX, clientY) {
    let menu = document.createElement("div");
    menu.id = "customResizeMenu";
    menu.style.position = "fixed"; // fixed relative to viewport
    menu.style.left = clientX + "px";
    menu.style.top = clientY + "px";
    menu.style.backgroundColor = "#fff";
    menu.style.border = "1px solid #ccc";
    menu.style.padding = "5px";
    menu.style.zIndex = 10000;
    menu.style.cursor = "pointer";
    menu.textContent = "Resize Node";
    menu.addEventListener("click", function(e) {
      if (document.body.contains(menu)) {
        document.body.removeChild(menu);
      }
      resizingNode = nodeId;
      resizeStartX = e.clientX;
      resizeStartY = e.clientY;
      const bbox = network.getBoundingBox(nodeId);
      if (bbox) {
        initialWidth = bbox.right - bbox.left;
        initialHeight = bbox.bottom - bbox.top;
      } else {
        initialWidth = 100;
        initialHeight = 50;
      }
    });
    document.body.appendChild(menu);
    document.addEventListener("click", function onDocClick(ev) {
      if(ev.target !== menu) {
        if(document.body.contains(menu)) {
          document.body.removeChild(menu);
        }
        document.removeEventListener("click", onDocClick);
      }
    });
  }

  // Contextmenu handler to show the resize menu.
  container.addEventListener("contextmenu", function(event) {
    event.preventDefault();
    const rect = container.getBoundingClientRect();
    const pointer = {
      x: event.clientX - rect.left,
      y: event.clientY - rect.top
    };
    const nodeId = network.getNodeAt(pointer);
    if (nodeId) {
      showResizeMenu(nodeId, event.clientX, event.clientY);
    }
  });

    container.addEventListener('mousedown', function (event) {
      if (event.button !== 0 || resizingNode) return;
      const rect = container.getBoundingClientRect();
      const pointer = {
        x: event.clientX - rect.left,
        y: event.clientY - rect.top
      };
      const clickedNodeId = network.getNodeAt(pointer);
      const clickedEdgeId = (typeof network.getEdgeAt === 'function') ? network.getEdgeAt(pointer) : null;
      if (!clickedNodeId && !clickedEdgeId) {
        startX = pointer.x;
        startY = pointer.y;
        selectionBox.style.left = startX + 'px';
        selectionBox.style.top = startY + 'px';
        selectionBox.style.width = '0';
        selectionBox.style.height = '0';
        selectionBox.style.display = 'block';
        dragging = true;
        network.setOptions({ interaction: { dragView: false } });
      }
    });

    container.addEventListener('mousemove', function (event) {
          if (resizingNode) {
        // how far have we dragged?
        const dx = event.clientX - resizeStartX;
        const dy = event.clientY - resizeStartY;
        // compute new dims, with a sensible minimum
        const newWidth  = Math.max(50, initialWidth  + dx);
        const newHeight = Math.max(30, initialHeight + dy);
        // update the node’s constraints
        const n = nodes.get(resizingNode);
        n.widthConstraint  = { minimum: newWidth,  maximum: newWidth  };
        n.heightConstraint = { minimum: newHeight, maximum: newHeight };
        nodes.update(n);
        return;
      }
      if (dragging) {
        const rect = container.getBoundingClientRect();
        const x = event.clientX - rect.left;
        const y = event.clientY - rect.top;
        selectionBox.style.width = Math.abs(x - startX) + 'px';
        selectionBox.style.height = Math.abs(y - startY) + 'px';
        selectionBox.style.left = (x > startX ? startX : x) + 'px';
        selectionBox.style.top = (y > startY ? startY : y) + 'px';
        return;
      }
      container.style.cursor = 'default';
    });

    container.addEventListener('mouseup', function (event) {
      if (resizingNode) {
        resizingNode = null;
        container.style.cursor = 'default';
        return;
      }
      if (!dragging) return;
      dragging = false;
      selectionBox.style.display = 'none';
      network.setOptions({ interaction: { dragView: true } });
      const domRect = container.getBoundingClientRect();
      const curX = event.clientX - domRect.left;
      const curY = event.clientY - domRect.top;
      const rect = {
        x:      Math.min(startX, curX),
        y:      Math.min(startY, curY),
        width:  Math.abs(curX - startX),
        height: Math.abs(curY - startY)
      };
      selectNodesFromRectangle(rect, { network: network, nodes: nodes, container: container });
    });

network.on("doubleClick", function(params) {
  // 1) Node label & color edit
  if (params.nodes.length === 1) {
    const nodeId   = params.nodes[0];
    const nodeData = nodes.get(nodeId);

    // — Edit label —
    const newLabel = prompt("Edit node label:", nodeData.label);
    if (newLabel !== null) {
      nodeData.label = newLabel;
    }

    // — Edit border color —
    // pull out whatever border you’ve got now (or default blue)
    const currentBorder = nodeData.color?.border || "#2B7CE9";
    const newColor     = prompt("Edit node border color:", currentBorder);
    if (newColor !== null) {
      if (nodeData.type === "group") {
        // for groups, repaint border + translucent fill
        nodeData.color = makeGroupColor(newColor);
      } else {
        // for regular nodes, keep fully-transparent background
        nodeData.color = {
          border:     newColor,
          background: "rgba(0,0,0,0)",
          highlight:  { border: newColor, background: "rgba(0,0,0,0)" },
          hover:      { border: newColor, background: "rgba(0,0,0,0)" },
          selected:   { border: newColor, background: "rgba(0,0,0,0)" }
        };
      }
    }

    // commit both label and color changes at once
    nodes.update(nodeData);
    updateAllNodeFontSizes({network, nodes, edges, container});

  // 2) Edge‐only branch stays the same
  } else if (params.edges.length === 1) {
    const edgeId   = params.edges[0];
    const edgeData = edges.get(edgeId);
    const newLabel = prompt("Edit edge label:", edgeData.label);
    if (newLabel !== null) {
      edgeData.label = newLabel;
      edgeData.color = (newLabel === "true") ? "green" : "red";
      edges.update(edgeData);
    }
  }
});

    network.on("click", function(params) {
      if (params.nodes.length > 0) {
        // force-select exactly that topmost node:
        network.setSelection({ nodes: params.nodes, edges: [] });
      }
      else if (params.edges.length > 0) {
        // your existing edge-only branch
        network.setSelection({ nodes: [], edges: params.edges });
      }
         showRepresentation();
    });

  container.addEventListener("keydown", function (event) {
    if (event.key === "Delete" || event.key === "Backspace") {
      event.preventDefault();
      let sel = network.getSelection();
      if (sel.nodes.length > 0 && confirm("Delete selected node(s)?")) {
        nodes.remove(sel.nodes);
      }
      if (sel.edges.length > 0 && confirm("Delete selected edge(s)?")) {
        edges.remove(sel.edges);
      }
    }
  });

  network.on("dragStart", function (params) {
    if (params.nodes.length > 0) {
      const nodeId = params.nodes[0];
      if (isGroupNode(nodeId, { network: network, nodes: nodes, container: container })) {
        const groupPos = network.getPositions([nodeId])[nodeId];
        const memberNodes = nodes.get().filter(node => node.group === nodeId);
        memberNodes.forEach(node => {
          const pos = network.getPositions([node.id])[node.id];
          node.relativePos = { x: pos.x - groupPos.x, y: pos.y - groupPos.y };
        });
      }
    }
  });

  network.on("dragging", function (params) {
    if (params.nodes.length > 0) {
      const nodeId = params.nodes[0];
      if (isGroupNode(nodeId, { network: network, nodes: nodes, container: container })) {
        const groupPos = network.getPositions([nodeId])[nodeId];
        const memberNodes = nodes.get().filter(node => node.group === nodeId);
        memberNodes.forEach(node => {
          const newX = groupPos.x + node.relativePos.x;
          const newY = groupPos.y + node.relativePos.y;
          network.moveNode(node.id, newX, newY);
        });
      }
    }
  });

  network.on("dragEnd", function (params) {
    if (params.nodes.length > 0) {
      const nodeId = params.nodes[0];
      const node = nodes.get(nodeId);
      if (!node) return;
      let currentGroupId = node.group;
      while (currentGroupId) {
        keepNodeWithinGroup(nodeId, currentGroupId, { network: network, nodes: nodes, container: container });
        updateGroupNodeSize(currentGroupId, { network: network, nodes: nodes, container: container });
        const parentGroup = nodes.get(currentGroupId);
        if (parentGroup && parentGroup.group) {
          currentGroupId = parentGroup.group;
        } else {
          break;
        }
      }
    }
  });

  return pane;
}

// ------------------------------
// Multi-pane Functions
// ------------------------------

function addNewPane() {
  // 1) Decide on the new index
  const newIndex = paneCount;
  paneCount++;

  // 2) Prompt for the pane name
  let name = prompt("Enter name for this new pane:", "Visual " + (newIndex + 1));
  if (!name || !name.trim()) {
    name = "Visual " + (newIndex + 1);
  }
  name = name.trim();

  // 3) Create the tab element
  const tabBar = document.getElementById('tabBar');
  const newTab = document.createElement('div');
  newTab.className = 'tab';
  newTab.textContent = name;
  newTab.setAttribute('data-index', newIndex);

  // 4) Single‐click activates the pane
  newTab.onclick = function() {
    switchToPane(newIndex);
  };

  // 5) Double‐click to rename
  newTab.ondblclick = function(e) {
    e.stopPropagation();
    let newName = prompt("Rename this pane:", newTab.textContent);
    if (newName && newName.trim()) {
      renamePane(newIndex, newName.trim());
    }
  };

  // 6) Insert it just before the “+” button
  const addTabButton = document.getElementById('addTabButton');
  tabBar.insertBefore(newTab, addTabButton);

  // 7) Create the corresponding visual pane container
  const visualContainer = document.getElementById('visualContainer');
  const newPane = document.createElement('div');
  newPane.className = 'visualPane';
  newPane.id = 'visual-' + newIndex;
  const networkDiv = document.createElement('div');
  networkDiv.id = 'mynetwork-' + newIndex;
  newPane.appendChild(networkDiv);
  visualContainer.appendChild(newPane);

  // 8) Initialize the Vis pane and store it
  const pane = createNetworkPane(newIndex);
  networkPanes[newIndex] = pane;

  // 9) Activate it
  switchToPane(newIndex);
}


function switchToPane(index) {
  var panes = document.getElementsByClassName('visualPane');
  for (var i = 0; i < panes.length; i++) {
    panes[i].classList.remove('active');
  }
  var tabs = document.getElementsByClassName('tab');
  for (var i = 0; i < tabs.length; i++) {
    tabs[i].classList.remove('active');
  }
  document.getElementById('visual-' + index).classList.add('active');
  var tabElements = document.querySelectorAll('.tab');
  tabElements.forEach(function(tab){
    if (tab.getAttribute('data-index') == index) {
      tab.classList.add('active');
    }
  });
  currentActiveIndex = index;
  showRepresentation();
}

function createGroupNode() {
  const pane = networkPanes[currentActiveIndex];

  // 1) pull exactly the set of selected IDs
  const sel = pane.network.getSelection().nodes.slice();
  if (sel.length < 1) {
    return alert("Please select one or more nodes or groups to group.");
  }

  // 2) figure out the dominant border color
  const childColors = sel.map(id => {
    const node = pane.nodes.get(id);
    return node.color?.border || (typeof node.color === 'string' ? node.color : '#2B7CE9');
  });
  const freq = {};
  childColors.forEach(c => freq[c] = (freq[c]||0) + 1);
  let groupBorderColor = childColors[0];
  childColors.forEach(c => {
    if (freq[c] > freq[groupBorderColor]) groupBorderColor = c;
  });

  // 3) compute a bounding box that encloses *all* the selected items
  const allBBs = sel.map(id => pane.network.getBoundingBox(id) || (() => {
    const p = pane.network.getPositions([id])[id];
    return { left: p.x, top: p.y, right: p.x, bottom: p.y };
  })());
  let minX = Math.min(...allBBs.map(b => b.left));
  let minY = Math.min(...allBBs.map(b => b.top));
  let maxX = Math.max(...allBBs.map(b => b.right));
  let maxY = Math.max(...allBBs.map(b => b.bottom));
  const PADDING = 20;
  minX -= PADDING; minY -= PADDING;
  maxX += PADDING; maxY += PADDING;
  const width  = maxX - minX;
  const height = maxY - minY;
  const centerX = (minX + maxX)/2;
  const centerY = (minY + maxY)/2;

  // 4) create the new group node
  const groupId = 'group_'+Date.now();
  const fontSize = 40, extraPad = 10;
  pane.nodes.add({
    id:       groupId,
    type:     'group',
    label:    prompt("Enter label for this group:","Group") || "",
    x:        centerX,
    y:        centerY,
    widthConstraint:  { minimum: width,  maximum: width  },
    heightConstraint: { minimum: height, maximum: height },
    shape:    'box',
    color:    makeGroupColor(groupBorderColor),
    font:     { size: fontSize, align: 'left', vadjust: -height/2 + fontSize/2 + extraPad },
    margin:   { left: extraPad }
  });

  // 5) re-assign each selected item into the new group
  sel.forEach(id => {
    const node = pane.nodes.get(id);
    node.group = groupId;
    pane.nodes.update(node);
  });

  // 6) now re-compute sizes of that group *and* any ancestor groups
  function resizeRecursively(gid) {
    updateGroupNodeSize(gid, pane);
    const parent = pane.nodes.get(gid).group;
    if (parent) resizeRecursively(parent);
  }
  resizeRecursively(groupId);


  // 7) Finally resize to fit text
  updateGroupNodeSize(groupId, pane);
}


// Add dblclick listener to the initial tab (assumed to be present in HTML).
document.querySelector('.tab[data-index="0"]').ondblclick = function(e) {
  e.stopPropagation();
  var tab = document.querySelector('.tab[data-index="0"]');
  var newName = prompt("Enter new name for this pane:", tab.textContent);
  if(newName) {
    renamePane(0, newName);
  }
};

document.getElementById('addTabButton').addEventListener('click', addNewPane);
networkPanes[0] = createNetworkPane(0);
document.querySelector('.tab[data-index="0"]').addEventListener('click', function() {
  switchToPane(0);
});

window.addEventListener('resize', function() {
  // Update font sizes on all panes (if multipane), else just the current one
  networkPanes.forEach(pane => {
    if (pane) updateAllNodeFontSizes(pane);
  });
});


const firstTab = document.querySelector('.tab[data-index="0"]');
firstTab.onclick = () => switchToPane(0);
firstTab.ondblclick = function(e) {
  e.stopPropagation();
  const curr = firstTab.textContent;
  const name = prompt("New name for this pane:", curr);
  if (name && name.trim()) {
    renamePane(0, name.trim());
  }
};


/**
 * Rename a pane by updating the text content on its associated tab.
 * If the pane index is not provided, it renames the current active pane.
 *
 * @param {number} [index] - The pane index to rename.
 * @param {string} newName - The new name for the pane.
 */
function renamePane(index, newName) {
  if (index === undefined || index === null) {
    index = currentActiveIndex;
  }
  var tab = document.querySelector('.tab[data-index="' + index + '"]');
  if (tab) {
    tab.textContent = newName;
  } else {
    console.error("Tab for pane index " + index + " not found.");
  }
}

// hook up the dropdown
document.getElementById('viewSelect').addEventListener('change', showRepresentation);

/**
 * Build a JSON-serializable graph object
 */
function getGraphJson(pane) {
  const nodesArray = pane.nodes.get();
  const edgesArray = pane.edges.get();

  const nodesJson = nodesArray.map(node => {
    // mirror exportJson’s mapping but return the object
    const bbox = pane.network.getBoundingBox(node.id) || { right:0, left:0, top:0, bottom:0 };
    const w = bbox.right - bbox.left;
    const h = bbox.bottom - bbox.top;
    const x = node.x - w/2;
    const y = node.y - h/2;
    return {
      id:    node.id,
      type:  node.type === 'group' ? 'group' : 'text',
      label: node.label,
      x, y,
      width:  w,
      height: h,
      color: node.color
    };
  });

  const edgesJson = edgesArray.map(edge => ({
    id:       edge.id,
    fromNode: edge.from,
    toNode:   edge.to,
    label:    edge.label,
    color:    edge.color
  }));

  return { nodes: nodesJson, edges: edgesJson };
}

// 1) After your existing hookup for viewSelect:
const viewSelect = document.getElementById('viewSelect');
const pySelect   = document.getElementById('pythonModeSelect');

viewSelect.addEventListener('change', () => {
  // show/hide the python sub-selector
  if (viewSelect.value === 'python') {
    pySelect.style.display = 'block';
  } else {
    pySelect.style.display = 'none';
  }
  showRepresentation();
});

// also re-run whenever python sub-mode changes:
pySelect.addEventListener('change', showRepresentation);


// 2) Extend showRepresentation():
async function showRepresentation() {
  const pane = networkPanes[currentActiveIndex];
  const mode = viewSelect.value;
  const tree = document.getElementById('jsonTree');
  const pre  = document.getElementById('outputArea');
  const pySel = pythonModeSelect;

  if (mode === 'json') {
    // ───────── JSON MODE ─────────
    tree.innerHTML = '';
    const graph = getGraphJson(pane);
    tree.appendChild(renderCollapsible(graph, null));

    tree.style.display = 'block';
    pre.style.display  = 'none';
    pySel.style.display = 'none';
    pythonButtons.style.display = 'none';
    return;
  }

  if (mode === 'python') {
    // ───────── PYTHON MODE ─────────
    tree.style.display = 'none';
    pre.style.display  = 'block';
    pySel.style.display = 'block';
    updatePythonButtons();       // show/hide Edit/Save buttons

    // pane name for workflow.py naming convention
    const activeTab = document.querySelector('.tab.active');
    let paneName =activeTab ? activeTab.textContent.trim() : '';
    // default to "example" if still the autogenerated name or empty
    if (!paneName || /^Visual \b/i.test(paneName)) {
      paneName = 'example';
    }
    // sanitize for a valid Python identifier
    const flowName = paneName
      .replace(/\W+/g, '_')      // non-alphanumeric → underscore
      .replace(/^(\d)/, '_$1');  // leading digit → underscore+digit

    // If there are no group nodes, just show function names
    const groupExists = pane.nodes.get().some(n => n.type === 'group');
    if (!groupExists && pySel.value === "functions") {
        const funcs = pane.nodes.get().filter(
          n => n.type !== 'group' && !n.id.startsWith('temp_')
      );
      pre.textContent = funcs.map((n, i) => {
        const funcName = n.label && n.label.trim() ? n.label.replace(/\W+/g, "_") : `func${i + 1}`;
        return `
    def ${funcName}(mr) -> (bool, dict):
        """
        ${funcName}
        """
        print("${funcName} called.")
        return True, {}
    `.trim();
      }).join('\n\n');
      return;
    }

    pre.textContent = 'Loading…';

    // fetch & slice
    const fullJson = getGraphJson(pane);
    const resp     = await fetch('/generate', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ json: fullJson, flowName: flowName })
    });
    const data = await resp.json();

    let result = '';
    if (pySel.value === 'board') {
      Object.entries(data)
            .filter(([k]) => k.endsWith('/board.py'))
            .forEach(([k,v]) => result += `// ${k}\n${v}\n\n`);
    }
    else if (pySel.value === 'functions') {
      Object.entries(data)
            .filter(([k]) => k.endsWith('/functions.py'))
            .forEach(([k,v]) => result += `// ${k}\n${v}\n\n`);
    }
    else { // workflow
      result = data['workflow.py'] || '';
    }

    let processed = (result || '')
    // 1) Remove question marks everywhere
    .replace(/\?/g, '')
    // 2) Only tweak the def lines:
    .split('\n')
    .map(line => {
      // if this is a function signature...
      if (/^\s*def\s+/.test(line)) {
        return line.replace(
          // capture "def ", the name, and the "("
          /^(\s*def\s+)([^(]+)(\()/,
          (_, pre, name, paren) => {
            // replace spaces inside the name → underscores
            const safeName = name.trim().replace(/ +/g, '_');
            return pre + safeName + paren;
          }
        );
      }
      // otherwise leave the line exactly as-is
      return line;
    })
    .join('\n');

    pre.textContent = processed || "// no Python code generated";
    return;
  }

  // (if you ever add more modes, handle them here)
}




// 3) New: call your Flask endpoint and slice the result
async function generatePythonRepresentation(subMode, pane) {
  const full = getGraphJson(pane);
  const resp = await fetch('/generate', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ json: full, flowName: flowName })
  });
  const data = await resp.json();

  let result = '';
  if (subMode === 'board') {
    // gather all the board.py snippets
    for (let key in data) {
      if (key.endsWith('/board.py')) {
        result += `// ${key}\n` + data[key] + '\n\n';
      }
    }
  }
  else if (subMode === 'functions') {
    for (let key in data) {
      if (key.endsWith('/functions.py')) {
        result += `// ${key}\n` + data[key] + '\n\n';
      }
    }
  }
  else if (subMode === 'workflow') {
    result = data['workflow.py'] || '';
  }

  document.getElementById('outputArea').textContent = result;
}

// store user edits per-pane
const pythonEdits = {};  // pythonEdits[paneIndex] = { board: "...", functions: "...", workflow: "..." }

const editBtn = document.getElementById('editBtn');
const saveBtn = document.getElementById('saveBtn');
const outputPre = document.getElementById('outputArea');
const editArea  = document.getElementById('editArea');
const pythonButtons = document.getElementById('pythonButtons');

editBtn.addEventListener('click', () => {
  const idx = currentActiveIndex;
  // load existing edited text if any
  const mode = pySelect.value;
  editArea.value = pythonEdits[idx]?.[mode] ?? outputPre.textContent;
  outputPre.style.display = 'none';
  editArea.style.display = 'block';
  editBtn.style.display = 'none';
  saveBtn.style.display = 'inline-block';
  editArea.focus();
});

saveBtn.addEventListener('click', () => {
  const idx = currentActiveIndex;
  const mode = pySelect.value;
  // persist
  pythonEdits[idx] = pythonEdits[idx] || {};
  pythonEdits[idx][mode] = editArea.value;
  // restore view
  outputPre.textContent = editArea.value;
  editArea.style.display = 'none';
  outputPre.style.display = 'block';
  saveBtn.style.display = 'none';
  editBtn.style.display = 'inline-block';
});

// Ensure buttons only show in Python mode
function updatePythonButtons() {
  if (viewSelect.value === 'python') {
    pythonButtons.style.display = 'block';
    editBtn.style.display = 'inline-block';
    saveBtn.style.display = 'none';
  } else {
    pythonButtons.style.display = 'none';
  }
}
// call in showRepresentation after rendering






// ---------- splitter drag logic ----------
const splitter = document.getElementById('splitter');
const sidePanel = document.getElementById('sidePanel');

splitter.addEventListener('mousedown', (e) => {
  e.preventDefault();
  document.addEventListener('mousemove', onDrag);
  document.addEventListener('mouseup',  stopDrag);
});

function onDrag(e) {
  // calculate new width so that #sidePanel hugs the right edge:
  const newWidth = window.innerWidth - e.clientX;
  // enforce a minimum so it never collapses completely:
  sidePanel.style.width = Math.max(newWidth, 100) + 'px';
}

function stopDrag() {
  document.removeEventListener('mousemove', onDrag);
  document.removeEventListener('mouseup',  stopDrag);
}


/**
 * Recursively build a collapsible DOM tree from JS value.
 */
function renderCollapsible(value, key) {
  const node = document.createElement(key == null ? 'div' : 'details');
  if (key != null) {
    const summary = document.createElement('summary');
    summary.textContent = key + (typeof value !== 'object' ? `: ${value}` : '');
    node.appendChild(summary);
  }

  if (value && typeof value === 'object') {
    const isArray = Array.isArray(value);
    const entries = isArray ? value : Object.entries(value);

    entries.forEach((entry, idx) => {
      let childKey, childVal;
      if (isArray) {
        childVal = entry;
        childKey = (entry && entry.label != null)
          ? entry.label
          : `[${idx}]`;
      } else {
        [childKey, childVal] = entry;
      }

      // recurse to build the child <details> (or <div>)
      const childNode = renderCollapsible(childVal, childKey);

      // **If this entry is one of your graph nodes, tag the childNode**
      if (entry && entry.id != null && (entry.type === 'text' || entry.type === 'group')) {
        childNode.dataset.nodeId = entry.id;
      }

      node.appendChild(childNode);
    });

    if (key != null) node.open = false;  // start collapsed
  }

  return node;
}


// function for highlighting on the right
function highlightTree(nodeId) {
  const tree = document.getElementById('jsonTree');

  // 1️⃣ Clear any old highlights
  tree.querySelectorAll('details.highlighted').forEach(d => {
    d.classList.remove('highlighted');
    d.open = false;                  // also collapse it again
  });

  if (!nodeId) return;

  // 2️⃣ Find the <details> for this nodeId
  const selector = `details[data-node-id="${nodeId}"]`;
  const target   = tree.querySelector(selector);
  if (!target) return;

  // 3️⃣ Highlight it
  target.classList.add('highlighted');

  // 4️⃣ Ensure it's open, and open all ancestors
  target.open = true;
  let parent = target.parentElement;
  while (parent && parent !== tree) {
    if (parent.tagName === 'DETAILS') {
      parent.open = true;
    }
    parent = parent.parentElement;
  }
}




// initial render
showRepresentation();
"

styles.css ":root {
  --ui-font-size: 16px;
  --button-font-size: 15px;
  --sidepanel-font-size: 16px;
  --sidepanel-width: 270px;
}

body {
  font-size: var(--ui-font-size);
}

#mynetwork,
[id^="mynetwork-"] {
    width: 100%;
    height: 100%;
    border: 1px solid lightgray;
    box-sizing: border-box;
}

#controls {
    padding: 10px;
}

#selectionBox {
    position: absolute;
    border: 1px dashed blue;
    background-color: rgba(0, 0, 255, 0.1);
    pointer-events: none;
    z-index: 1002;
    display: none;
}

.vis-network .vis-node.vis-box {
    border: 2px dashed #2B7CE9;
    background-color: rgba(0, 0, 0, 0) !important;
}

/* layout for the two-column view */
#mainContent {
    display: flex;
    height: 100vh;
    margin: 0;
}

#leftPane {
    flex: 1;
    display: flex;
    flex-direction: column;
    position: relative;  /* positioning context */
    min-height: 0;
}

/* ----------- Top Bar Buttons ----------- */
button[onclick="createGroupNode()"] {
  font-size: var(--button-font-size);
  padding: 4px 14px;
  margin-right: 8px;
  margin-top: 4px;
  margin-bottom: 4px;
  z-index: 1000;
}

/* Style the Create Template button - DON'T CREATE IT HERE */
button[onclick="createTemplate()"] {
  position: absolute;
  top: 10px;
  right: 10px;
  left: auto;
  font-size: 20px;
  z-index: 1000;
}

/* ----------- Bottom Buttons ----------- */
#bottomButtons {
    display: flex;
    gap: 5px;
    margin-top: auto;   /* pushes this to bottom of #leftPane */
    padding: 8px;
}

#bottomButtons .small-btn,
#bottomButtons button,
#bottomButtons input[type="file"] {
  font-size: var(--button-font-size);
  padding: 3px 10px;
  cursor: pointer;
}

#bottomButtons input[type="file"] {
  flex: 1 1 auto;
  min-width: 0;
}

/* ----------- Side Panel ----------- */
#sidePanel {
    width: var(--sidepanel-width);
    border-left: 1px solid #ccc;
    padding: 8px;
    box-sizing: border-box;
    overflow: auto;
    background: #fafafa;
    font-size: var(--sidepanel-font-size);
}

#sidePanel label {
    font-weight: bold;
    margin-right: 5px;
}

#viewSelect {
    margin-bottom: 10px;
    width: 100%;
    font-size: var(--button-font-size);
}

#pythonModeSelect {
  margin-bottom: 10px;
  width: 100%;
  box-sizing: border-box;
  font-size: var(--button-font-size);
}

#outputArea {
    white-space: pre-wrap;
    background-color: #f5f5f5;
    padding: 7px;
    height: calc(100% - 2em);
    overflow: auto;
    border: 1px solid #ddd;
    box-sizing: border-box;
    font-size: 15px;
}

/* separator line just above the bottom buttons */
#buttonSeparator {
    width: 100%;
    margin: 0;
    border: none;
    border-top: 1px solid #ccc;
}

/* new: draggable vertical splitter */
#splitter {
  width: 5px;
  cursor: col-resize;
  background-color: rgba(0,0,0,0.1);
  flex: none;               /* don’t flex-grow/shrink */
  z-index: 1001;
}

/* Highlight the <details> wrapping a selected node in the JSON tree */
#jsonTree details.highlighted > summary {
  background-color: #ffff99;  /* pale yellow */
}

/* Visual Pane */
#visualContainer {
    flex: 1;
    position: relative;
    height: auto;
    min-height: 0;
}

.visualPane {
    position: absolute;
    top: 0; left: 0; bottom: 0; right: 0;
}

/* Modal styles */
.modal-overlay {
  position: fixed;
  top: 0; left: 0; right: 0; bottom: 0;
  background: rgba(0,0,0,0.5);
  display: flex; align-items: center; justify-content: center;
  z-index: 2000;
}
.modal-content {
  background: white;
  padding: 1.5em;
  border-radius: 8px;
  text-align: center;
  width: 300px;
}
.modal-content input {
  width: 100%;
  padding: 0.5em;
  margin: 0.5em 0;
}
.modal-content button {
  padding: 0.5em 1em;
}

/* ----------- Responsive adjustments ----------- */
@media (max-width: 1100px) {
  :root {
    --ui-font-size: 14px;
    --button-font-size: 13px;
    --sidepanel-font-size: 13.5px;
    --sidepanel-width: 190px;
  }
  #sidePanel {
    padding: 6px;
  }
}

@media (max-width: 800px) {
  :root {
    --ui-font-size: 12px;
    --button-font-size: 11.5px;
    --sidepanel-font-size: 11.5px;
    --sidepanel-width: 110px;
  }
  #sidePanel {
    padding: 4px;
  }
  button[onclick="createGroupNode()"],
  button[onclick="createTemplate()"] {
    padding: 2px 7px;
  }
  #bottomButtons .small-btn,
  #bottomButtons button {
    padding: 2px 6px;
  }
}

@media (min-width: 1400px) {
  :root {
    --ui-font-size: 18px;
    --button-font-size: 17px;
    --sidepanel-font-size: 18px;
    --sidepanel-width: 320px;
  }
  #sidePanel {
    padding: 14px;
  }
}

/* End CSS */
" main.py"from flask import Flask, send_from_directory, render_template_string, request, jsonify
import os
import json as _json
import Create_Python_Repr as repr_mod
import traceback
import json


app = Flask(__name__)

# Read external files
with open("C:\\Users\\hannah\\metaprogramming-\\Assets\\styles.css", "r", encoding='utf-8') as css_file:
    css_content = css_file.read()

with open("C:\\Users\\hannah\\metaprogramming-\\Assets\\script.js", "r", errors='ignore') as js_file:
    js_content = js_file.read()

html_string = f'''
<!doctype html>
<html>
<head>
  <meta charset="utf-8">
  <title>Interactive Network</title>
  <style>
    {css_content}

    /* Additional styles for the tab view */
    #tabBar {{
      display: flex;
      border-bottom: 1px solid #ccc;
      padding: 5px;
      background: #f9f9f9;
    }}
    .tab {{
      padding: 5px 10px;
      margin-right: 5px;
      border: 1px solid #ccc;
      border-bottom: none;
      cursor: pointer;
    }}
    .tab.active {{
      background-color: #fff;
      font-weight: bold;
    }}
    .visualPane {{
      display: none;
      position: absolute;
      top: 0;
      left: 0;
      width: 100%;
      height: 100%;
    }}
    .visualPane.active {{
      display: block;
    }}
  </style>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/jszip/3.7.1/jszip.min.js"></script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/FileSaver.js/2.0.5/FileSaver.min.js"></script>
  <script src="https://unpkg.com/vis-network/standalone/umd/vis-network.min.js"></script>
</head>
<body>
<!-- project creation modal -->
<div id="projectModal" class="modal-overlay">
  <div class="modal-content">
    <h2>Create New Project</h2>
    <input id="projectNameInput" placeholder="Project name…" />
    <button id="createProjectBtn">Create</button>
  </div>
</div>
  <div id="mainContent">
    <!-- LEFT: your existing controls + visual panes -->
    <div id="leftPane">
      <div id="controls">
        <button onclick="createGroupNode()">Create Group from Selection</button>
        <!-- project name placeholder -->
        <span id="projectDisplay" style="margin:0 1em; font-weight:bold;"></span>
        <button onclick="createTemplate()">Create Template</button>
      </div>
      <div id="tabBar">
        <div class="tab active" data-index="0">Visual 1</div>
        <div class="tab" id="addTabButton">+</div>
      </div>
      <div id="visualContainer">
        <div class="visualPane active" id="visual-0">
          <div id="mynetwork-0"></div>
        </div>
        <div id="selectionBox"></div>
      </div>
        <div id="bottomButtons">
          <button class="small-btn" onclick="exportJson()">Export JSON</button>
          <button class="small-btn" onclick="importJson()">Import JSON</button>
          <input type="file" id="importFile" accept=".json">
        </div>
    </div>
    <div id="splitter"></div>    <!-- ← new draggable handle -->
    <!-- RIGHT: the new dropdown + output area -->
    <div id="sidePanel">
      <label for="viewSelect">View:</label>
      <select id="viewSelect">
        <option value="json">JSON</option>
        <option value="python">Python</option>
      </select>
      <!-- new: choose which Python output to show -->
      <select id="pythonModeSelect" style="display: none;">
        <option value="board">Board</option>
        <option value="functions">Functions</option>
        <option value="workflow">Workflow</option>
      </select>
      <div id="jsonTree" style="display:none; overflow:auto; max-height:100%;"></div>
      <pre id="outputArea" style="display:none;"></pre>
      <textarea id="editArea" style="display:none; width:100%; height:60%; box-sizing:border-box; font-family: monospace;"></textarea>
      <div id="pythonButtons" style="display:none; margin-top:5px;">
        <button id="editBtn" class="small-btn">Edit</button>
        <button id="saveBtn" class="small-btn" style="display:none;">Save</button>
      </div>
    </div>
  </div>

  <script type="text/javascript">
    {js_content}
    // ← we’ll inject our new handlers here
  </script>
</body>
</html>
'''

@app.route('/')
def index():
    return render_template_string(html_string)

@app.route('/generate', methods=['POST'])
def generate():
    payload   = request.get_json()
    graph_json= payload['json']
    flow_name = payload.get('flowName','flow')
    # run your generator
    try:
        generated = repr_mod.generate_all_code(json.dumps(graph_json), flow_name)
        return jsonify(generated)
    except Exception as e:
        # ─── Print the full traceback to your terminal so you can see exactly where it failed ───
        traceback.print_exc()
        # ─── Return a JSON error payload (status 400) so the front end can read e.__str__() ───
        return jsonify({"error": str(e)}), 400

if __name__ == '__main__':
    app.run(debug=True)" Create_python_repr "import json
import networkx as nx
import re


def create_graph_from_json(json_data):
    G = nx.DiGraph()
    nodes = {node["id"]: node for node in json_data["nodes"]}
    
    for edge in json_data["edges"]:
        from_node = edge["fromNode"]
        to_node = edge["toNode"]
        
        if from_node in nodes:
            G.add_node(from_node, label=nodes[from_node].get("label", nodes[from_node].get("text", "")))
        if to_node in nodes:
            G.add_node(to_node, label=nodes[to_node].get("label", nodes[to_node].get("text", "")))
        
        G.add_edge(from_node, to_node, label=edge["label"], id=edge["id"])
    
    return G


def get_elements_from_canvas(nodes):
    elems = []
    
    for node in nodes:
        # Example logic to detect certain node attributes
        if "color" in node.keys():
            if node["color"]["border"] == "#2b7ce9" and node["type"] == "group":
                node["elem_type"] = "Board"
                elems.append(node)
            elif node["color"]["border"] == "#2b7ce9" and node["type"] == "group":
                node["elem_type"] = "CompositeFunction"
                elems.append(node)
    
    return elems


def find_closest_node(node, list_nodes):
    if not list_nodes:
        return None
    # Your existing logic to find the "closest" group
    closest_node = list_nodes[0]
    min_area = closest_node["width"] * closest_node["height"]
    
    for n in list_nodes:
        node_area = n["width"] * n["height"]
        
        if (
            (n["x"] < node["x"]) and
            (n["x"] + n["width"] > node["x"]) and
            (n["y"] < node["y"]) and
            (n["y"] + n["height"] > node["y"]) and
            (node_area < min_area)
        ):
            closest_node = n
            min_area = node_area
    
    return closest_node


def merge_unique_key_value_pairs(dictionary_list):
    merged = {}
    
    for d in dictionary_list:
        for key, value in d.items():
            if key not in merged:
                merged[key] = value
    
    return merged


def string_to_bool_or_original(value):
    if value == "True":
        return True
    elif value == "False":
        return False
    else:
        return value


def extract_between_brackets(text):
    pattern = r"[[(.*?)]]"
    matches = re.findall(pattern, text)
    
    if len(matches) == 0:
        return [text]
    else:
        return matches


def to_camel_case(text):
    words = text.strip().split(" ")
    if len(words) == 1:
        return text.capitalize()
    return "".join(w.capitalize() for w in words)


def get_paths(canvas_data, graph):
    """
    Build a structure describing all root -> leaf paths
    and the functions in each step, triggers, actions, etc.
    """
    nodes = canvas_data["nodes"]
    elements = get_elements_from_canvas(nodes)
    roots = {}
    
    # Identify root nodes
    for x in nodes:
        if graph.in_degree(x["id"]) == 0:
            closest_element = find_closest_node(x, elements)
            if closest_element:
                closest_element_label = closest_element.get("label", closest_element.get("text", ""))
                closest_element_type = closest_element["elem_type"]
                roots[x["id"]] = {
                    "label": closest_element_label,
                    "type": closest_element_type,
                    "paths": [],
                }
            else:
                fallback_label = x.get("label", x.get("text", ""))
                roots[x["id"]] = {
                    "label": fallback_label,
                    "type": "Unknown",
                    "paths": [],
                }

    # For each root, gather possible paths
    for root in roots:
        func_map = []
        leaf_nodes = [nid for nid in graph.nodes() if graph.out_degree(nid) == 0]
        paths = []

        for leaf in leaf_nodes:
            possible_paths = list(nx.all_simple_paths(graph, source=root, target=leaf))
            
            for p in possible_paths:
                funcs = []
                doc_strs = []
                
                for i in p:
                    node_label = graph.nodes[i].get("label", graph.nodes[i].get("text", ""))
                    doc_str = node_label.split("[")[0]
                    doc_strs.append(doc_str)
                    bracket_content = extract_between_brackets(node_label)[0].split("|")
                    
                    if len(bracket_content) == 1:
                        func_name = bracket_content[0]
                    else:
                        # e.g. if bracket_content = ["something", "functionName"]
                        func_name = bracket_content[-1]
                    
                    funcs.append(func_name)

                func_map.append(dict(zip(funcs[:-1], doc_strs[:-1])))

                triggers = []
                for s, t in zip(p[:-1], p[1:]):
                    z = graph.get_edge_data(s, t)["label"]
                    triggers.append(z)
                
                # Last function is the "action"
                final_action = string_to_bool_or_original(funcs[-1])
                paths.append({"trigger": dict(zip(funcs, triggers)), "action": final_action})

        updated_map = merge_unique_key_value_pairs(func_map)
        roots[root]["paths"] = paths
        roots[root]["functions"] = updated_map

    return roots


# IN-MEMORY CODE GENERATORS
def generate_function_code(flow_name, elem_label, elem_type, functions_dict, markdown_lookup=None):
    """
    Generates the content that would go into functions.py (as a string).
    Optionally reads from an external “markdown_lookup” dict if needed.
    Returns a single string.
    """
    # If you have no Markdown or don’t need it, simply omit that part
    disclaimer = (
        '"""\n'
        "DISCLAIMER:\n"
        "This python file was created automatically.\n"
        '"""\n\n'
    )
    code_lines = [disclaimer]
    
    # If you had a dictionary of markdown_content keyed by function name:
    # markdown_lookup = { "functionA": "...md content..." }

    composite_funcs = []  # You might gather them separately if needed

    for func_name, doc_str in functions_dict.items():
        # Optionally skip if it's a composite function; depends on your logic
        if func_name in composite_funcs:
            continue

        # If you have actual Python code from markdown_lookup
        # just sample logic below:
        if markdown_lookup and func_name in markdown_lookup:
            py_code = markdown_lookup[func_name]
            # Minimal check or parse
            if len(py_code) > 5:
                code_lines.append(py_code.strip() + "\n\n")
            else:
                # fallback
                code_lines.append(
                    f"def {func_name}(mr) -> (bool, dict):\n"
                    f'    """{doc_str}"""\n'
                    f'    print("Function {func_name} called.")\n'
                    f"    return (True, {{}})\n\n"
                )
        else:
            # fallback
            code_lines.append(
                f"def {func_name}(mr) -> (bool, dict):\n"
                f'    """{doc_str}"""\n'
                f'    print("Function {func_name} called.")\n'
                f"    return (True, {{}})\n\n"
            )

    return "".join(code_lines)


def generate_board_code(flow_name, elem_label, elem_type, functions_dict, paths):
    """
    Generates the content for board.py or compositefunction.py (as a string).
    """
    disclaimer = (
        '"""\n'
        "DISCLAIMER:\n"
        "This python file was created automatically.\n"
        '"""\n\n'
    )
    code_lines = [disclaimer]
    # Possibly an import to your validation framework
    code_lines.append(f"from radv.medical_record.validations.framework import {elem_type}\n")

    # Then, if referencing local functions or composite ones
    # This is where you might import from .functions
    # or from another node’s compositefunction if needed.
    func_names = list(functions_dict.keys())
    imported_func_list = ", ".join(func_names)
    code_lines.append(f"from .functions import {imported_func_list}\n\n")

    # Define the class for the board or composite function
    class_name = to_camel_case(elem_label)
    code_lines.append(f"class {class_name}({elem_type}):\n")

    # Indent assignment lines
    # Convert functions_dict keys to a dictionary { "funcName": funcName, ... }
    dict_items = ", ".join([f"'{k}': {k}" for k in functions_dict])
    code_lines.append(f"    functions = {{{dict_items}}}\n")
    code_lines.append(f"    paths = {paths}\n")

    return "".join(code_lines)


def generate_workflow_code(flow_name, boards):
    """
    Generates the content for workflow.py (as a string).
    """
    disclaimer = (
        '"""\n'
        "DISCLAIMER:\n"
        "This python file was created automatically.\n"
        '"""\n\n'
    )
    code_lines = [disclaimer]
    code_lines.append("from radv.medical_record.validations.framework import Workflow\n\n")
    
    # Import any boards you discovered
    for board_class_name in boards:
        code_lines.append(
            f"from radv.medical_record.validations.framework import.{flow_name}.Board_{board_class_name}.board import {board_class_name}\n"
        )

    code_lines.append("\n")
    code_lines.append(f"class {flow_name}(Workflow):\n")
    # boards = ["BoardOne", "BoardTwo", ...]
    joined_boards = ", ".join(boards)
    code_lines.append(f"    boards = [{joined_boards}]\n")

    return "".join(code_lines)


def generate_all_code(json_string, flow_name):
    """
    Example “master” function that:
      1) Reads the JSON (already in memory as a string)
      2) Builds the graph
      3) Extracts paths, boards, composite functions
      4) Generates in-memory code for each board
      5) Generates in-memory code for each composite function
      6) Generates in-memory “workflow” code
    Returns a data structure with all code as strings
    """
    # 1) Load data
    canvas_data = json.loads(json_string)
    graph = create_graph_from_json(canvas_data)
    paths_dict = get_paths(canvas_data, graph)
    
    # 2) Identify boards, composites, etc.
    boards = []
    generated_content = {}

    for key, info in paths_dict.items():
        elem_label = info["label"]
        elem_type = info["type"]
        func_dict = info["functions"]
        path_data = info["paths"]

        # Convert label for directory/class usage
        u_elem_label = to_camel_case(elem_label)

        # (A) functions.py content
        func_py_content = generate_function_code(
            flow_name, elem_label, elem_type, func_dict
        )

        # (B) board.py or compositefunction.py content
        if elem_type == "Board":
            board_py_content = generate_board_code(
                flow_name, elem_label, elem_type, func_dict, path_data
            )
            boards.append(u_elem_label)

            # Store in memory
            generated_content[f"{elem_type}_{u_elem_label}/board.py"] = board_py_content
            generated_content[f"{elem_type}_{u_elem_label}/functions.py"] = func_py_content

        elif elem_type == "CompositeFunction":
            comp_py_content = generate_board_code(
                flow_name, elem_label, elem_type, func_dict, path_data
            )
            # Store in memory
            generated_content[f"{elem_type}_{u_elem_label}/compositefunction.py"] = comp_py_content
            generated_content[f"{elem_type}_{u_elem_label}/functions.py"] = func_py_content

    # 3) Finally, workflow.py
    workflow_content = generate_workflow_code(flow_name, boards)
    generated_content["workflow.py"] = workflow_content

    return generated_content


# Testing with the provided JSON string
json_str = """
{
  "nodes": [
    {
      "id": "3e7b8acd417adc40",
      "type": "group",
      "x": -595.5,
      "y": -269,
      "width": 667,
      "height": 483,
      "label": "Flow",
      "color": {
        "border": "#2b7ce9",
        "background": "rgba(43,124,233,0.122)",
        "highlight": {
          "border": "#2b7ce9",
          "background": "rgba(43,124,233,0.122)"
        },
        "hover": {
          "border": "#2b7ce9",
          "background": "rgba(43,124,233,0.122)"
        },
        "selected": {
          "border": "#2b7ce9",
          "background": "rgba(43,124,233,0.122)"
        }
      }
    },
    {
      "id": "04f402c4e2af7f56",
      "type": "group",
      "x": -575.5,
      "y": -109,
      "width": 622,
      "height": 303,
      "label": "Replace",
      "color": {
        "border": "#08b94e",
        "background": "rgba(8,185,78,0.147)",
        "highlight": {
          "border": "#08b94e",
          "background": "rgba(8,185,78,0.147)"
        },
        "hover": {
          "border": "#08b94e",
          "background": "rgba(8,185,78,0.147)"
        },
        "selected": {
          "border": "#08b94e",
          "background": "rgba(8,185,78,0.147)"
        }
      }
    },
    {
      "id": "6d1ef69553b877d9",
      "type": "text",
      "text": "Replace with text",
      "x": -553,
      "y": 92,
      "width": 272,
      "height": 82,
      "color": {
        "border": "#f6f4f4",
        "background": "rgba(0,0,0,0)",
        "highlight": {
          "border": "#f6f4f4",
          "background": "rgba(0,0,0,0)"
        },
        "hover": {
          "border": "#f6f4f4",
          "background": "rgba(0,0,0,0)"
        },
        "selected": {
          "border": "#f6f4f4",
          "background": "rgba(0,0,0,0)"
        }
      }
    },
    {
      "id": "214991ab8cafdfbc",
      "type": "text",
      "text": "Replace with text",
      "x": -248,
      "y": 92,
      "width": 272,
      "height": 82,
      "color": {
        "border": "#f4f0f0",
        "background": "rgba(0,0,0,0)",
        "highlight": {
          "border": "#f4f0f0",
          "background": "rgba(0,0,0,0)"
        },
        "hover": {
          "border": "#f4f0f0",
          "background": "rgba(0,0,0,0)"
        },
        "selected": {
          "border": "#f4f0f0",
          "background": "rgba(0,0,0,0)"
        }
      }
    },
    {
      "id": "01c0c38aeb5e5e13",
      "type": "text",
      "text": "Replace with Text",
      "x": -273,
      "y": -249,
      "width": 322,
      "height": 122,
      "color": {
        "border": "#e0ac00",
        "background": "rgba(0,0,0,0)",
        "highlight": {
          "border": "#e0ac00",
          "background": "rgba(0,0,0,0)"
        },
        "hover": {
          "border": "#e0ac00",
          "background": "rgba(0,0,0,0)"
        },
        "selected": {
          "border": "#e0ac00",
          "background": "rgba(0,0,0,0)"
        }
      }
    },
    {
      "id": "6babf3007067a1e7",
      "type": "text",
      "text": "Replace with Text",
      "x": -493,
      "y": -89,
      "width": 402,
      "height": 102,
      "color": {
        "border": "#086ddd",
        "background": "rgba(0,0,0,0)",
        "highlight": {
          "border": "#086ddd",
          "background": "rgba(0,0,0,0)"
        },
        "hover": {
          "border": "#086ddd",
          "background": "rgba(0,0,0,0)"
        },
        "selected": {
          "border": "#086ddd",
          "background": "rgba(0,0,0,0)"
        }
      }
    }
  ],
  "edges": [
    {
      "id": "37f808b16516c77e",
      "fromNode": "6babf3007067a1e7",
      "toNode": "6d1ef69553b877d9",
      "fromSide": "bottom",
      "toSide": "top",
      "label": "True",
      "color": "#e0ac00"
    },
    {
      "id": "b5e7f2a787970ec8",
      "fromNode": "6babf3007067a1e7",
      "toNode": "214991ab8cafdfbc",
      "fromSide": "bottom",
      "toSide": "top",
      "label": "False",
      "color": "#e93147"
    }
  ]
}
"""


#res = generate_all_code(json_str, "example")
#print(json.dumps(res, indent=2))"
