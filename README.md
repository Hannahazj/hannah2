def derive_callable_name(raw_label: str, fallback: str) -> str:
    """
    Canonical callable name:
      - if raw has '?', use right side
      - else use fallback (or raw)
    Then snake_case it.
    """
    raw = (raw_label or "").strip()
    if "?" in raw:
        _, right = raw.split("?", 1)
        base = right.strip()
    else:
        base = fallback or raw
    return to_snake_case(base) or "unnamed_function"


def derive_display_name(raw_label: str, fallback: str) -> str:
    """
    Human display name for Board.functions:
      - if raw has '?', use cleaned left side (fallback to right)
      - else use cleaned raw
    """
    raw = (raw_label or "").strip()
    if "?" in raw:
        left, right = raw.split("?", 1)
        disp = clean_string(left.strip()) or clean_string(right.strip()) or fallback
    else:
        disp = clean_string(raw) or fallback
    return disp

def get_paths(canvas_data: Dict, graph: nx.DiGraph) -> Dict:
    """
    Build per-root (board) paths and function registry.
    KEYS in 'functions' and in 'paths.trigger'/'paths.action' are the
    FINAL callable names (match 'from .functions import <name>').
    VALUES in 'functions' keep the RAW label (may include '?') for docs.
    """
    nodes = canvas_data.get("nodes", [])
    elements = get_elements_from_canvas(nodes)
    roots: Dict[str, Dict[str, Any]] = {}

    # Identify root nodes (in-degree 0) and associate to nearest element (Board/Composite)
    for node in nodes:
      node_id = node.get("id")
      if node_id not in graph:
          continue
      if graph.in_degree(node_id) == 0:
          closest_element = find_closest_node(node, elements)
          if closest_element:
              roots[node_id] = {
                  "label": clean_string(closest_element.get("label", "")),
                  "type": closest_element.get("elem_type", "Unknown"),
                  "paths": [],
                  "functions": {}  # { callable_name: raw_label }
              }
          else:
              roots[node_id] = {
                  "label": clean_string(node.get("label", "")),
                  "type": "Unknown",
                  "paths": [],
                  "functions": {}
              }

    # For each root, enumerate unique paths to leaves
    for root_id, root_info in roots.items():
        if root_id not in graph:
            continue

        leaf_nodes = [n for n in graph.nodes() if graph.out_degree(n) == 0]
        seen_path_keys = set()

        for leaf in leaf_nodes:
            if leaf not in graph:
                continue

            for path in nx.all_simple_paths(graph, source=root_id, target=leaf):
                callable_seq: List[str] = []
                triggers: List[str] = []
                node_ids: List[str] = []

                for i, node_id in enumerate(path):
                    # RAW label first (may include '?')
                    raw_label = graph.nodes[node_id].get("raw_label") or graph.nodes[node_id].get("label", "")
                    # Fallback name source if needed
                    fallback = graph.nodes[node_id].get("label", "") or f"node_{node_id}"
                    # 🔑 Canonical callable for this node
                    callable_name = derive_callable_name(raw_label, fallback)
                    callable_seq.append(callable_name)
                    node_ids.append(node_id)

                    if i < len(path) - 1:
                        edge_data = graph.get_edge_data(node_id, path[i + 1]) or {}
                        triggers.append(clean_string(edge_data.get("label", "")))

                if not callable_seq:
                    continue

                # Dedupe by exact node-id sequence + triggers
                key = (tuple(node_ids), tuple(triggers))
                if key in seen_path_keys:
                    continue
                seen_path_keys.add(key)

                # Record the path (keys are callable names)
                root_info["paths"].append({
                    "trigger": dict(zip(callable_seq, triggers)),
                    "action": callable_seq[-1]
                })

                # Register intermediate functions under their callable names with RAW labels
                for idx, cname in enumerate(callable_seq[:-1]):
                    nid = node_ids[idx]
                    raw_for_func = graph.nodes[nid].get("raw_label") or graph.nodes[nid].get("label", "")
                    # Dict ensures de-dupe by callable name; first one wins
                    root_info["functions"].setdefault(cname, raw_for_func)

    return roots
