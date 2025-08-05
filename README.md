import json
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
    clean_name = to_snake_case(func_name) or "unnamed_function"
    return (
        f"def {clean_name}(mr) -> (bool, dict):\n"
        f'    """{clean_string(doc_str) or clean_name}"""\n'
        f'    print("{clean_name} called.")\n'
        f"    return True, {{}}\n\n"
    )


def generate_board_files(board_name: str, functions: Dict, paths: List) -> Dict[str, str]:
    """Generate all files for a single board."""
    snake_name = to_snake_case(board_name)
    pascal_name = to_pascal_case(board_name)
    
    # Generate board.py content
    board_content = (
        '"""\nDISCLAIMER:\nThis python file was created automatically.\n"""\n\n'
        f"from ...framework import Board\n"
        f"from .functions import {', '.join([to_snake_case(f) for f in functions])}\n\n"
        f"class {pascal_name}(Board):\n"
        f"    functions = {{\n"
    )
    
    # Add functions dictionary
    func_entries = []
    for func_name in functions:
        clean_func = to_snake_case(func_name)
        func_entries.append(f'        "{clean_string(func_name)}": {clean_func}')
    
    board_content += ",\n".join(func_entries) + "\n    }\n\n"
    board_content += f"    paths = {json.dumps(paths, indent=8)}\n"
    
    # Generate functions.py content
    functions_content = (
        '"""\nDISCLAIMER:\nThis python file was created automatically.\n"""\n\n'
    )
    for func_name, doc_str in functions.items():
        functions_content += generate_function_code(func_name, doc_str)
    
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
