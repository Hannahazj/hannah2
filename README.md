Create_Python_repr.py "import json
import networkx as nx
import re
import os
from typing import Dict, List, Any


def clean_string(s: str) -> str:
    """Clean strings for Python code generation."""
    if not s:
        return ""
    # Remove special characters and extra spaces
    s = re.sub(r'[^a-zA-Z0-9_ ]', '', s.strip())
    return re.sub(r'\s+', ' ', s)


def to_snake_case(text: str) -> str:
    """Convert text to snake_case for filenames and functions."""
    text = clean_string(text)
    return '_'.join(text.lower().split())


def to_pascal_case(text: str) -> str:
    """Convert text to PascalCase for class names."""
    text = clean_string(text)
    words = text.split()
    return ''.join(word.capitalize() for word in words)


def create_graph_from_json(json_data: Dict) -> nx.DiGraph:
    """Create a directed graph from JSON data."""
    G = nx.DiGraph()
    nodes = {node["id"]: node for node in json_data.get("nodes", [])}
    
    for edge in json_data.get("edges", []):
        from_node = edge.get("fromNode")
        to_node = edge.get("toNode")
        
        if from_node in nodes:
            label = nodes[from_node].get("label", nodes[from_node].get("text", ""))
            G.add_node(from_node, label=clean_string(label))
            
        if to_node in nodes:
            label = nodes[to_node].get("label", nodes[to_node].get("text", ""))
            G.add_node(to_node, label=clean_string(label))
        
        if from_node and to_node:
            edge_label = clean_string(edge.get("label", ""))
            G.add_edge(from_node, to_node, label=edge_label, id=edge.get("id"))

    return G


def generate_function_code(func_name: str, doc_str: str) -> str:
    """Generate a single Python function definition."""

    try:
        func_part = func_name.split("?")[-1].strip()  # Use part after ? as function name
        doc_part = func_name.split("?")[0].strip()    # Use part before ? as docstring
    except:
        func_part = func_name
        doc_part = doc_str or func_name  # Fallback to func_name if no doc_str
    clean_name = to_snake_case(func_part) or "unnamed_function"
    clean_doc = clean_string(doc_part) or clean_name
    
    result = (
        f"def {clean_name}(mr) -> (bool, dict):\n"
        f'    """{clean_doc}"""\n'
        f'    print("{clean_name} called.")\n'
        f"    return True, {{}}\n\n"
    )
    return result

def generate_board_files(board_name: str, functions: Dict, paths: List) -> Dict[str, str]:
    """Generate all files for a single board."""
    snake_name = to_snake_case(board_name)
    pascal_name = to_pascal_case(board_name)
    
    # First pass: create mapping of clean function names to display names
    function_map = {}
    for func_name in functions:
        if "?" in func_name:
            display_name = func_name.split("?")[0].strip()
            clean_func = to_snake_case(func_name.split("?")[-1].strip())
        else:
            display_name = func_name
            clean_func = to_snake_case(func_name)
        
        # Only keep the first occurrence of each clean function name
        if clean_func not in function_map:
            function_map[clean_func] = display_name
    
    # Generate board.py content
    board_content = (
        '"""\nDISCLAIMER:\nThis python file was created automatically.\n"""\n\n'
        f"from ...framework import Board\n"
        f"from .functions import {', '.join(sorted(function_map.keys()))}\n\n"
        f"class {pascal_name}(Board):\n"
        f"    functions = {{\n"
    )
    
    # Add functions dictionary using display names as keys
    func_entries = []
    for clean_func, display_name in function_map.items():
        func_entries.append(f'        "{clean_string(display_name)}": {clean_func}')
    
    board_content += ",\n".join(func_entries) + "\n    }\n\n"
    board_content += f"    paths = {json.dumps(paths, indent=8)}\n"
    
    # Generate functions.py content
    functions_content = (
        '"""\nDISCLAIMER:\nThis python file was created automatically.\n"""\n\n'
    )
    
    # Second pass: generate function code using original mapping
    seen_functions = set()
    for func_name, doc_str in functions.items():
        if "?" in func_name:
            clean_func = to_snake_case(func_name.split("?")[-1].strip())
        else:
            clean_func = to_snake_case(func_name)
            
        if clean_func not in seen_functions:
            functions_content += generate_function_code(func_name, doc_str)
            seen_functions.add(clean_func)
    
    # Generate __init__.py content
    init_content = (
        '"""\nPackage initialization.\n"""\n\n'
        f"from .board import {pascal_name}\n"
        f"__all__ = ['{pascal_name}']\n"
    )
    
    return {
        f"Board_{snake_name}/board.py": board_content,
        f"Board_{snake_name}/functions.py": functions_content,
        f"Board_{snake_name}/__init__.py": init_content
    }



def generate_workflow_code(flow_name: str, boards: List[str]) -> str:
    """Generate workflow.py content."""
    return (
        '"""\nDISCLAIMER:\nThis python file was created automatically.\n"""\n\n'
        "from ..framework import Workflow\n"
        + "\n".join([
            f"from {flow_name}.Board_{to_snake_case(b)} import {to_pascal_case(b)}" 
            for b in boards
        ]) + "\n\n"
        f"class {to_pascal_case(flow_name)}(Workflow):\n"
        f"    boards = [{', '.join([to_pascal_case(b) for b in boards])}]\n"
    )


def generate_all_code(json_string: str, flow_name: str) -> Dict[str, str]:
    """Generate all Python code files from graph JSON."""
    try:
        canvas_data = json.loads(json_string)
        graph = create_graph_from_json(canvas_data)
        paths_dict = get_paths(canvas_data, graph)
        
        generated_content = {}
        boards = []

        # Generate code for each board
        for elem_id, elem_info in paths_dict.items():
            if elem_info["type"] == "Board":
                board_name = elem_info["label"] or "UntitledBoard"
                board_files = generate_board_files(
                    board_name,
                    elem_info["functions"],
                    elem_info["paths"]
                )
                generated_content.update(board_files)
                boards.append(board_name)

        # Generate workflow.py if we have boards
        if boards:
            generated_content["workflow.py"] = generate_workflow_code(flow_name, boards)
            generated_content["__init__.py"] = '"""\nPackage initialization.\n"""\n'

        return generated_content

    except Exception as e:
        return {"error": f"Code generation failed: {str(e)}"}


def get_paths(canvas_data: Dict, graph: nx.DiGraph) -> Dict:
    """Analyze paths through the graph."""
    nodes = canvas_data.get("nodes", [])
    elements = get_elements_from_canvas(nodes)
    roots = {}
    
    # Identify root nodes
    for node in nodes:
        node_id = node.get("id")
        if graph.in_degree(node_id) == 0:
            closest_element = find_closest_node(node, elements)
            if closest_element:
                roots[node_id] = {
                    "label": clean_string(closest_element.get("label", "")),
                    "type": closest_element.get("elem_type", "Unknown"),
                    "paths": [],
                    "functions": {}
                }
            else:
                roots[node_id] = {
                    "label": clean_string(node.get("label", "")),
                    "type": "Unknown",
                    "paths": [],
                    "functions": {}
                }

    # Analyze paths for each root
    for root_id, root_info in roots.items():
        leaf_nodes = [n for n in graph.nodes() if graph.out_degree(n) == 0]
        
        for leaf in leaf_nodes:
            for path in nx.all_simple_paths(graph, source=root_id, target=leaf):
                path_functions = []
                triggers = []
                
                for i, node_id in enumerate(path):
                    node_label = graph.nodes[node_id].get("label", "")
                    clean_label = clean_string(node_label)
                    path_functions.append(to_snake_case(clean_label))
                    
                    if i < len(path) - 1:
                        edge_data = graph.get_edge_data(node_id, path[i+1])
                        triggers.append(clean_string(edge_data.get("label", "")))

                if path_functions:
                    root_info["paths"].append({
                        "trigger": dict(zip(path_functions, triggers)),
                        "action": path_functions[-1]
                    })
                    
                    for func in path_functions[:-1]:
                        if func not in root_info["functions"]:
                            root_info["functions"][func] = func.replace('_', ' ').title()

    return roots


def get_elements_from_canvas(nodes: List[Dict]) -> List[Dict]:
    """Identify special elements (boards, composite functions) from nodes."""
    elems = []
    for node in nodes:
        if node.get("type") == "group":
            border_color = node.get("color", {}).get("border", "").lower()
            if border_color == "#2b7ce9":
                node["elem_type"] = "Board"
                elems.append(node)
            elif border_color == "#08b94e":
                node["elem_type"] = "CompositeFunction"
                elems.append(node)
    return elems


def find_closest_node(node: Dict, list_nodes: List[Dict]) -> Dict:
    """Find the closest containing group node."""
    if not list_nodes:
        return None
    
    closest_node = None
    min_area = float('inf')
    
    for n in list_nodes:
        node_area = n.get("width", 0) * n.get("height", 0)
        if (n.get("x", 0) < node.get("x", 0)) and \
           (n.get("x", 0) + n.get("width", 0) > node.get("x", 0)) and \
           (n.get("y", 0) < node.get("y", 0)) and \
           (n.get("y", 0) + n.get("height", 0) > node.get("y", 0)) and \
           (node_area < min_area):
            closest_node = n
            min_area = node_area
    
    return closest_node
" and scritp.js"var networkPanes = [];
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
    // add in the framework.py file
    const respF = await fetch ('/Python_Files/framework.py');
    const frameworkCode = await respF.text();
    pyFolder.file("framework.py", frameworkCode);

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
