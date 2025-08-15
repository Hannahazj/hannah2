def get_paths(canvas_data: Dict, graph: nx.DiGraph) -> Dict:
    """
    Analyze paths through the graph and collect per-board functions/paths.
    IMPORTANT: We do NOT clean/strip the node labels before storing them in
    root_info["functions"]; we keep the raw label (which may include '?') so
    later code can split docstring vs. function name properly.
    """
    nodes = canvas_data.get("nodes", [])
    elements = get_elements_from_canvas(nodes)
    roots: Dict[str, Dict[str, Any]] = {}

    # Identify root nodes (in-degree == 0), and associate them to the closest element (Board/Composite)
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
                    "functions": {}  # { callable_key: raw_label_with_possible_question_mark }
                }
            else:
                roots[node_id] = {
                    "label": clean_string(node.get("label", "")),
                    "type": "Unknown",
                    "paths": [],
                    "functions": {}
                }

    # For each root, collect unique paths (source→leaf) with dedupe
    for root_id, root_info in roots.items():
        if root_id not in graph:
            continue

        leaf_nodes = [n for n in graph.nodes() if graph.out_degree(n) == 0]
        seen_path_keys = set()  # dedupe per root based on (node_id sequence, triggers)

        for leaf in leaf_nodes:
            if leaf not in graph:
                continue

            for path in nx.all_simple_paths(graph, source=root_id, target=leaf):
                callable_keys: List[str] = []
                triggers: List[str] = []
                path_node_ids: List[str] = []

                for i, node_id in enumerate(path):
                    # --- keep RAW first (may contain '?') ---
                    raw_label = graph.nodes[node_id].get("raw_label") or graph.nodes[node_id].get("label", "")

                    # Build the callable key ONLY for wiring/triggers (safe identifier)
                    # Fall back to a stable name if the raw cleans to empty.
                    cleaned_for_key = clean_string(raw_label)
                    callable_key = to_snake_case(cleaned_for_key) or f"node_{str(node_id).replace('-', '_')}"
                    callable_keys.append(callable_key)
                    path_node_ids.append(node_id)

                    # Edge trigger label between this node and the next
                    if i < len(path) - 1:
                        edge_data = graph.get_edge_data(node_id, path[i + 1]) or {}
                        triggers.append(clean_string(edge_data.get("label", "")))

                if not callable_keys:
                    continue

                # Dedupe using exact node-id sequence + trigger sequence
                path_key = (tuple(path_node_ids), tuple(triggers))
                if path_key in seen_path_keys:
                    continue
                seen_path_keys.add(path_key)

                # Record the unique path:
                # - trigger map is { callable_key_i : trigger_label_i }
                # - action is the final callable key
                root_info["paths"].append({
                    "trigger": dict(zip(callable_keys, triggers)),
                    "action": callable_keys[-1]
                })

                # Register intermediate functions with their RAW labels (keep '?')
                for idx, func_key in enumerate(callable_keys[:-1]):
                    nid = path_node_ids[idx]
                    raw_for_func = graph.nodes[nid].get("raw_label") or graph.nodes[nid].get("label", "")
                    if func_key not in root_info["functions"]:
                        root_info["functions"][func_key] = raw_for_func

    return roots
