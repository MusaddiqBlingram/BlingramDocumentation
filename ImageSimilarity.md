# Bellokey Image Similarity Engine: Systems Architecture & Theory

This document outlines the first-principles design of Bellokey's visual search engine.

Do not implement this as a naive "feature" where an image goes in and similar images come out. If you execute a brute-force K-Nearest Neighbors (KNN) search across our production database, the system will suffer from compute collapse and volume bias.

We are building a Hybrid Search Pipeline that uses Machine Learning for semantic extraction, and Relational Algebra for strict deduplication.

Read the theory before touching the code.

---

## 1. The Physics of Vector Search (First Principles)

When a user uploads a photo, the computer does not "see" a kitchen. It sees a grid of pixels. To search by aesthetic, we must convert visual pixels into mathematical geometry.

We use the SigLIP Vision Transformer. This model compresses an image into a 768-dimensional Latent Space.

Imagine a 3D graph, but with 768 axes. Every image becomes a single coordinate (a Vector) in this space.

- Modern, dark-granite kitchens cluster tightly together in one region.
- Bright, white-tile bathrooms cluster in another.

To find similar images, we measure the distance between the user's uploaded vector and the database vectors.

We use Cosine Distance (`<=>`), which measures the angle between two vectors rather than the straight-line distance, ensuring that lighting and image resolution do not distort the aesthetic match.

---

### The Two Fatal Flaws of Naive Vector Search

If we just dump all these vectors into a flat table and run a search, the system fails:

#### 1. Compute Collapse 𝒪(N)

If a user searches for a kitchen, a naive database will calculate the complex 768-d floating-point angle against every single exterior, bedroom, and bathroom in the database.

At 100,000 listings, the CPU will pin to 100% and latency will spike to seconds.

#### 2. Volume Bias (Duplicate Listings)

If Listing A has 10 photos of its kitchen, and Listing B has 1 photo, Listing A occupies 10x more "surface area" in the latent space.

A request for the "Top 5 Matches" will return Listing A's kitchen from 5 different angles, pushing unique houses off the UI.

We solve Flaw 1 with Indexes.  
We solve Flaw 2 with Window Functions.

---

## 2. The Database Layer (Storage & Indexes)

**File:** `database/models.py`

We maintain a strict 1-to-N relationship (1 Property → Max 5 Images).

We use PostgreSQL with the `pgvector` extension.

---

### The Engine Block: GIN and HNSW Indexes

We do not perform sequential scans. We use specialized data structures to cheat physics.

---

### 1. The GIN Index (The Pre-Filter)

To prevent Compute Collapse, we must tag every image with a `room_types` array (e.g., `['kitchen', 'modern']`). But standard B-Tree indexes cannot sort arrays.

We use a Generalized Inverted Index (GIN).

**Mental Model:** Think of a textbook glossary. Instead of reading every page to find a word (𝒪(N)), you look up the word in the back, and it gives you the exact array of page numbers (𝒪(1)).

GIN maps the string `'kitchen'` directly to its Row IDs. This drops 80% of the database from the vector math equation instantly.

---

### 2. The HNSW Index (The Mathematician)

To search the remaining 20% of vectors, we use Hierarchical Navigable Small World (HNSW).

**Mental Model:** Instead of driving down every local road to find a house, HNSW builds a highway system graph. It makes massive jumps across the latent space at the top layer, dropping down into local neighborhoods only when it gets close.

This drops vector search time complexity from 𝒪(N) to 𝒪(log N).

```python
from pgvector.sqlalchemy import Vector
from sqlalchemy import Column, Index, Integer, String
from sqlalchemy.dialects.postgresql import ARRAY
from sqlalchemy.orm import declarative_base

Base = declarative_base()

class PropertyImage(Base):
    __tablename__ = 'property_images'

    id = Column(Integer, primary_key=True, autoincrement=True)
    property_id = Column(String, nullable=False, index=True)
    image_url = Column(String, nullable=False)

    # ARRAY allows multiple labels per image (e.g., open floor plans).
    # Prevents flat-table bloat and handles overlapping room semantics.
    room_types = Column(ARRAY(String))

    # 768-d vector matches SigLIP base output. L2-normalized on insert
    # to enable ultra-fast cosine distance math during retrieval.
    embedding = Column(Vector(768))

# GIN index solves the array search bottleneck. It maps values to rows,
# reducing sequential scans to O(1) lookups for pre-filtering.
Index(
    'ix_property_images_room_types_gin',
    PropertyImage.room_types,
    postgresql_using='gin'
)

# HNSW graph index drops vector search from O(N) to O(log N).
# m=16: limits memory per node. ef_construction=64: balances build
# time vs search accuracy. Requires vector_cosine_ops for <=> operator.
Index(
    'ix_property_images_embedding_hnsw',
    PropertyImage.embedding,
    postgresql_using='hnsw',
    postgresql_with={'m': 16, 'ef_construction': 64},
    postgresql_ops={'embedding': 'vector_cosine_ops'}
)
```
3. The Machine Learning Layer (Ingestion Worker)

File: ml/vision_engine.py

This code must run on an asynchronous background worker (e.g., Celery, AWS Lambda) and never on the main web API thread.

Loading a PyTorch model on the web server will trigger an Out-Of-Memory (OOM) kernel panic.

Zero-Shot Taxonomy & Math

We do not train a classifier. We use Zero-Shot Classification.

We calculate the Dot Product between the image vector and predefined Text Anchor vectors.

If the mathematical similarity exceeds 0.25, we append the label.

Handling Multiple Uploads (Mean Pooling & Anchoring)

If a user uploads 3 Kitchens and 1 Bathroom, we do not query the DB 4 times.

Anchoring

We look at the first image (Kitchen). This becomes our strict anchor.

Rejection

We ruthlessly drop the Bathroom image. If we average a Kitchen and a Bathroom, the resulting vector points to "Semantic Mud" (empty space in the latent graph).

Mean Pooling

We mathematically average the 3 Kitchen vectors together into a single "Master Vector", and re-normalize it to the unit sphere.

import logging
import torch
from PIL import Image
from transformers import AutoModel, AutoProcessor

logger = logging.getLogger(__name__)

# Load once in memory on worker boot
MODEL_ID = "google/siglip-base-patch16-224"
processor = AutoProcessor.from_pretrained(MODEL_ID)
model = AutoModel.from_pretrained(MODEL_ID)

ROOM_CATEGORIES = [
    "a professional real estate photo of a modern kitchen",
    "a professional real estate photo of a bathroom",
    "a professional real estate photo of a living room",
    "a professional real estate photo of a bedroom",
    "a professional exterior photo of a house",
    "a professional real estate photo of a balcony or terrace"
]
THRESHOLD = 0.25


def process_property_image(image_path: str) -> dict:
    """Extracts 768-d vector and zero-shot labels from an image."""
    image = Image.open(image_path).convert("RGB")
    inputs = processor(
        text=ROOM_CATEGORIES, images=image,
        padding="max_length", return_tensors="pt"
    )

    with torch.no_grad():
        outputs = model(**inputs)

        # Normalize Image Vector to length 1
        img_feat = outputs.image_features
        img_feat = img_feat / img_feat.norm(p=2, dim=-1, keepdim=True)
        final_vector = img_feat.squeeze().tolist()

        # Normalize Text Vectors & Calculate Cosine Similarity
        txt_feat = outputs.text_features
        txt_feat = txt_feat / txt_feat.norm(p=2, dim=-1, keepdim=True)
        cosine_sim = (img_feat @ txt_feat.T).squeeze()

    mapping = [
        'kitchen', 'bathroom', 'living_room',
        'bedroom', 'exterior', 'balcony'
    ]
    detected = [
        mapping[idx] for idx, score in enumerate(cosine_sim.tolist())
        if score > THRESHOLD
    ]

    if not detected:
        detected.append('other')

    return {"embedding": final_vector, "room_types": detected}


def generate_smart_search_vector(image_paths: list[str]) -> dict:
    """Anchors to the first image's category; averages matches."""
    valid_vectors = []
    anchor = None
    dropped = 0

    for i, path in enumerate(image_paths):
        data = process_property_image(path)

        # Set the Anchor on the very first image
        if i == 0:
            anchor = data["room_types"][0]
            valid_vectors.append(data["embedding"])
            continue

        # Reject cross-category noise to prevent semantic mud
        if anchor in data["room_types"]:
            valid_vectors.append(data["embedding"])
        else:
            dropped += 1
            logger.warning(f"Image {i} rejected. Cross-category noise.")

    # Mathematical Mean Pooling (Averaging)
    vector_matrix = torch.tensor(valid_vectors)
    mean_vector = torch.mean(vector_matrix, dim=0)

    # Re-normalize to surface of unit sphere for Cosine Distance
    final_vector = mean_vector / mean_vector.norm(
        p=2, dim=-1, keepdim=True
    )

    return {
        "search_vector": final_vector.tolist(),
        "target_room": anchor,
        "dropped_images": dropped
    }
4. The Retrieval Layer (Relational Algebra)

File: search/retrieval_service.py

This layer takes the math from the ML Engine and executes the database search.

To solve Volume Bias (Listing A showing up 5 times in the Top 5 results), we use a Common Table Expression (CTE) and a SQL Window Function (PARTITION BY).

Mental Model: The Window function acts as a brutal equalizer. It groups all matching photos into buckets by property_id. It ranks them internally by vector distance, throws away everything except Rank #1, and then compares the best photos from each property against each other.

This guarantees the UI displays 5 distinct houses.

from sqlalchemy import select, func
from database.models import PropertyImage


def execute_hybrid_search(
    session,
    query_vector: list[float],
    target_room: str,
    limit: int = 5
) -> list:
    """Executes hybrid search with relational deduplication."""

    # 1. Mathematical distance calculation (<=> operator)
    distance_expr = PropertyImage.embedding.cosine_distance(
        query_vector
    ).label('distance')

    # 2. The Equalizer: Window Function partitioning by property
    row_rank = func.row_number().over(
        partition_by=PropertyImage.property_id,
        order_by=distance_expr
    ).label('rank')

    # 3. Inner Query (CTE) with GIN Pre-Filtering
    base_query = select(
        PropertyImage.property_id,
        PropertyImage.image_url,
        distance_expr,
        row_rank
    ).where(
        PropertyImage.room_types.contains([target_room])
    ).cte('ranked_images')

    # 4. Outer Query for Deduplication
    final_query = select(
        base_query.c.property_id,
        base_query.c.image_url,
        base_query.c.distance
    ).where(
        base_query.c.rank == 1  # Keep only the #1 photo per property
    ).order_by(
        base_query.c.distance.asc()
    ).limit(limit)

    return session.execute(final_query).fetchall()
