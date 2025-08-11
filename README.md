def get_paths(canvas_data: Dict, graph: nx.DiGraph) -> Dict:
    """Analyze paths through the graph and collect board functions/paths (deduped)."""
    nodes = canvas_data.get("nodes", [])
    elements = get_elements_from_canvas(nodes)
    roots: Dict[str, Dict[str, Any]] = {}

    # Identify root nodes
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
                    "functions": {}  # { clean_func_name: raw_label_for_docstring }
                }
            else:
                roots[node_id] = {
                    "label": clean_string(node.get("label", "")),
                    "type": "Unknown",
                    "paths": [],
                    "functions": {}
                }

    # For each root, collect unique paths
    for root_id, root_info in roots.items():
        if root_id not in graph:
            continue

        leaf_nodes = [n for n in graph.nodes() if graph.out_degree(n) == 0]
        seen_path_keys = set()  # <-- dedupe per root

        for leaf in leaf_nodes:
            if leaf not in graph:
                continue

            for path in nx.all_simple_paths(graph, source=root_id, target=leaf):
                path_functions: List[str] = []
                triggers: List[str] = []
                path_node_ids: List[str] = []

                for i, node_id in enumerate(path):
                    node_label = graph.nodes[node_id].get("label", "")
                    clean_label = clean_string(node_label)
                    path_functions.append(to_snake_case(clean_label))
                    path_node_ids.append(node_id)

                    if i < len(path) - 1:
                        edge_data = graph.get_edge_data(node_id, path[i + 1]) or {}
                        triggers.append(clean_string(edge_data.get("label", "")))

                if not path_functions:
                    continue

                # Build a stable key using exact node-id sequence + triggers.
                # This prevents the same logical path from being added multiple times.
                path_key = (tuple(path_node_ids), tuple(triggers))
                if path_key in seen_path_keys:
                    continue
                seen_path_keys.add(path_key)

                # Append unique path
                root_info["paths"].append({
                    "trigger": dict(zip(path_functions, triggers)),  # mapping func->edge label
                    "action": path_functions[-1]
                })

                # Register functions (all but final action) with their raw labels
                for idx, func in enumerate(path_functions[:-1]):
                    node_id = path_node_ids[idx]
                    raw_label = graph.nodes[node_id].get("raw_label") or graph.nodes[node_id].get("label", "")
                    if func not in root_info["functions"]:
                        root_info["functions"][func] = raw_label

    return roots
