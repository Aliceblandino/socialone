# socialone
Progetto del corso di social computing 2025

elisa: 
1) lista autori giusta:
df["authors_list"] = (
    df["Authors"]
    .str.split(";")                          # separa gli autori
    .apply(lambda x: [
        a.strip().replace(",", "")           # rimuove la virgola cognome,nome
        for a in x
    ] if isinstance(x, list) else x)
)
df["authors_list"]

2) grafo statico 2o grafo:
import networkx as nx
import plotly.graph_objects as go
from plotly.subplots import make_subplots

# =========================
# 1) Nodo centrale per anno
# =========================
central_node_per_anno = {}

for anno, G in grafi_per_anno.items():
    if len(G) == 0:
        continue

    central_node_per_anno[anno] = max(G.degree, key=lambda x: x[1])[0]

# =========================
# 2) Vicini equicentrali del nodo centrale
# =========================
highlight_nodes_per_anno = {}

for anno, G in grafi_per_anno.items():
    if anno not in central_node_per_anno:
        continue

    central = central_node_per_anno[anno]
    central_degree = G.degree(central)

    equicentral_neighbors = [
        n for n in G.neighbors(central)
        if G.degree(n) == central_degree
    ]

    highlight_nodes_per_anno[anno] = [central] + equicentral_neighbors

# =========================
# 3) Sottografo: centrale + vicini
# =========================
subgrafi_per_anno = {}
pos_per_anno = {}

for anno, G in grafi_per_anno.items():
    if anno not in central_node_per_anno:
        continue

    central = central_node_per_anno[anno]
    neighbors = list(G.neighbors(central))

    nodes = [central] + neighbors
    H = G.subgraph(nodes).copy()
    subgrafi_per_anno[anno] = H

    initial_pos = {central: (0, 0)}
    pos = nx.spring_layout(
        H,
        seed=42,
        pos=initial_pos,
        fixed=[central]
    )
    pos_per_anno[anno] = pos

# =========================
# 4) Funzione Plotly
# =========================
def plot_graph_plotly(G, pos, highlight_nodes):
    edge_x, edge_y = [], []
    for u, v in G.edges():
        x0, y0 = pos[u]
        x1, y1 = pos[v]
        edge_x += [x0, x1, None]
        edge_y += [y0, y1, None]

    edge_trace = go.Scatter(
        x=edge_x,
        y=edge_y,
        mode="lines",
        line=dict(width=1, color="gray"),
        hoverinfo="none",
        showlegend=False
    )

    node_x, node_y, texts, colors, sizes = [], [], [], [], []

    for node in G.nodes():
        x, y = pos[node]
        node_x.append(x)
        node_y.append(y)
        texts.append(f"{node}<br>Grado: {G.degree(node)}")

        if node in highlight_nodes:
            colors.append("crimson")
            sizes.append(11)
        else:
            colors.append("royalblue")
            sizes.append(8)

    node_trace = go.Scatter(
        x=node_x,
        y=node_y,
        mode="markers",
        text=texts,
        hoverinfo="text",
        marker=dict(
            size=sizes,
            color=colors,
            line=dict(width=0.5, color="black")
        ),
        showlegend=False
    )

    return edge_trace, node_trace

# =========================
# 5) Genera titoli formattati
# =========================
authors_per_anno = {}

for anno, G in grafi_per_anno.items():
    if anno not in central_node_per_anno:
        continue

    central = central_node_per_anno[anno]
    central_degree = G.degree(central)

    # Nodo centrale + vicini con grado uguale
    equicentral_neighbors = [
        n for n in G.neighbors(central)
        if G.degree(n) == central_degree
    ]

    all_authors = [central] + equicentral_neighbors

    if len(all_authors) > 5:
        # Prendi primi 4 + quinto con "..." affiancato
        fifth_author = all_authors[4]
        authors_per_anno[anno] = all_authors[:4] + [f"{fifth_author} ..."]
    else:
        authors_per_anno[anno] = all_authors
   
titoli = [format_title(anno, authors_per_anno[anno]) for anno in anni]

# =========================
# 6) Crea figura 2x5
# =========================
fig = make_subplots(
    rows=2,
    cols=5,
    subplot_titles=titoli,
    horizontal_spacing=0.05,
    vertical_spacing=0.14
)

for i, anno in enumerate(anni):
    G = subgrafi_per_anno[anno]
    pos = pos_per_anno[anno]
    highlight_nodes = highlight_nodes_per_anno[anno]

    edge, node = plot_graph_plotly(G, pos, highlight_nodes)

    row = 1 if i < 5 else 2
    col = i + 1 if i < 5 else i - 4

    fig.add_trace(edge, row=row, col=col)
    fig.add_trace(node, row=row, col=col)

# =========================
# 7) Layout finale
# =========================
fig.update_layout(
    height=700,
    width=1500,
    paper_bgcolor="white",
    plot_bgcolor="white",
    title_x=0.5
)

for r in [1, 2]:
    for c in range(1, 6):
        fig.update_xaxes(visible=False, row=r, col=c)
        fig.update_yaxes(visible=False, row=r, col=c)

fig.show()

