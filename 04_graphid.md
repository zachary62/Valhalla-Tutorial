# Chapter 4: GraphId - The Universal Address

In [Chapter 3: DirectedEdge & NodeInfo](03_directededge___nodeinfo.md), we saw how Valhalla represents roads (`DirectedEdge`) and intersections (`NodeInfo`) as the building blocks of its map graph. We learned that a `DirectedEdge` needs to know which `NodeInfo` it connects *to*. But how does it specify that destination node? How can Valhalla uniquely identify *any* intersection or road segment anywhere in the world, even across different map files ([Chapter 2: GraphTile & GraphReader](02_graphtile___graphreader.md))?

Imagine trying to send a letter. You can't just write "John Smith"; you need a full address: street number, street name, city, state, zip code. Similarly, Valhalla needs a universal addressing system for its graph elements. This is where `GraphId` comes in.

In this chapter, we'll explore:

*   What a `GraphId` is: A unique "address" for any node or edge.
*   What information it contains: Tile ID, Level, and local ID.
*   How Valhalla uses `GraphId`s to connect the graph.

Let's learn how Valhalla keeps track of everything!

## The Problem: Finding Anything, Anywhere

Valhalla's map data is split into many `GraphTile` files, like chapters in a giant map book. Within each chapter, there are lists of intersections (`NodeInfo`) and road segments (`DirectedEdge`).

Now, imagine a road segment (`DirectedEdge`) that starts in one tile (map chapter) and ends in another. How does that edge store a reference to its endpoint node if that node is listed in a *different* chapter? Just storing the node's list number (index) from the other chapter isn't enough – we also need to know *which* chapter to look in!

We need an identifier that is:

1.  **Unique:** No two intersections or edges should have the same ID.
2.  **Global:** The ID should work across the entire world map, not just within one tile.
3.  **Informative:** The ID should ideally tell us where to find the corresponding data (i.e., which tile).
4.  **Compact:** It should be small and efficient to store and use.

## What is a `GraphId`? The Map's Address System

A `GraphId` is Valhalla's solution to this addressing problem. Think of it as a structured, compact address for every single `NodeInfo` (intersection) and potentially other graph elements within Valhalla's entire dataset.

It cleverly packs three key pieces of information into a single, small package:

1.  **Tile ID (`tileid_`)**: Which specific `GraphTile` (map chapter) does this element belong to? This tells Valhalla which file or data block contains the details.
2.  **Level (`level_`)**: Which level in the map hierarchy ([Chapter 7: TileHierarchy](07_tilehierarchy.md)) does this element belong to? This is important because Valhalla stores map data at different levels of detail (like zoom levels).
3.  **ID (`id_`)**: What is the specific index (position in the list) of this element *within* its `GraphTile` at that specific level? This points to the exact `NodeInfo` (or other element) in the tile's internal lists.

```
      GraphId = [ Level | Tile ID | ID (Index within Tile) ]
```

So, a `GraphId` doesn't just say "element number 5"; it says "element number 5, in map chapter 123, at detail level 0". This combination makes it globally unique and tells Valhalla exactly where to look for the data.

## How Valhalla Uses `GraphId`

`GraphId`s are used everywhere in Valhalla to link parts of the graph together.

**1. Connecting Edges to Nodes:**
Remember the `DirectedEdge` structure from [Chapter 3: DirectedEdge & NodeInfo](03_directededge___nodeinfo.md)? It needs to know where it ends. It stores this using a `GraphId` in its `endnode_` member.

```cpp
// Simplified view of DirectedEdge
// File: baldr/directededge.h

struct DirectedEdge {
  // ... other members like length, speed ...

  uint64_t endnode_; // <--- Stores the GraphId value of the destination NodeInfo

  // ... other members ...

  // Helper function to get the GraphId object
  GraphId endnode() const {
    return GraphId(endnode_);
  }
};
```

When a routing algorithm follows an edge, it can get the `endnode_` value, reconstruct the `GraphId` object, and use the `tileid()`, `level()`, and `id()` information to find and load the correct destination `NodeInfo`, even if it's in a different `GraphTile`.

**2. Identifying Snapped Points:**
In [Chapter 1: Location & PathLocation](01_location___pathlocation.md), we saw that a `PathLocation` contains `PathEdge`s, representing potential points on the road network. Each `PathEdge` needs to identify the specific road segment it lies on. It uses a `GraphId` for this.

```cpp
// Simplified view of PathEdge
// File: baldr/pathlocation.h

struct PathEdge {
  GraphId id;               // <--- Unique ID of the road segment (DirectedEdge)
  double percent_along;     // How far along the edge is this point?
  // ... other info ...
};
```

This `GraphId` in `PathEdge` usually refers to a `DirectedEdge` (though internally, edges don't typically have their own `GraphId`s distinct from their start/end nodes; the context tells us it refers to the edge starting at a node with a certain ID). It allows the routing algorithm ([Chapter 6: PathAlgorithm (Dijkstra/A*)](06_pathalgorithm__dijkstra_a__.md)) to know exactly which edge in the graph corresponds to the user's start/end point.

**3. General Referencing:**
Any time Valhalla needs a stable, unique way to refer to a node across different parts of the code or different levels of the graph hierarchy, it uses a `GraphId`.

## Creating and Using a `GraphId`

A `GraphId` can be created directly by providing its three components, or by using its raw 64-bit integer value.

```cpp
#include "baldr/graphid.h"
#include <iostream>

int main() {
  // Create a GraphId directly
  uint32_t tile_id = 12345; // Which map tile?
  uint32_t level = 0;      // Which hierarchy level?
  uint32_t id_in_tile = 50;  // Which node index within that tile?

  valhalla::baldr::GraphId node_address(tile_id, level, id_in_tile);

  // Access the components
  std::cout << "Level:   " << node_address.level() << std::endl;
  std::cout << "Tile ID: " << node_address.tileid() << std::endl;
  std::cout << "ID:      " << node_address.id() << std::endl;

  // Get the underlying 64-bit value (useful for storage, like in DirectedEdge)
  uint64_t packed_value = node_address.value;
  std::cout << "Packed Value: " << packed_value << std::endl;

  // Reconstruct from the packed value
  valhalla::baldr::GraphId reconstructed_address(packed_value);
  std::cout << "Reconstructed Level: " << reconstructed_address.level() << std::endl;

  // Check validity
  if (node_address.Is_Valid()) {
    std::cout << "Address is valid." << std::endl;
  }

  // Get a GraphId representing just the Tile and Level (useful for fetching tiles)
  valhalla::baldr::GraphId tile_address = node_address.Tile_Base();
  std::cout << "Tile Base ID: " << tile_address.id() << " (ID part is 0)" << std::endl;

  return 0;
}
```

**Example Output:**

```
Level:   0
Tile ID: 12345
ID:      50
Packed Value: 1234500000000050  // Actual value depends on bit packing
Reconstructed Level: 0
Address is valid.
Tile Base ID: 0 (ID part is 0)
```

This example shows how you can create a `GraphId`, extract its parts, get its compact representation, and rebuild it. The `Tile_Base()` method is handy when you just need to identify the tile itself, for example, when asking the `GraphReader` ([Chapter 2: GraphTile & GraphReader](02_graphtile___graphreader.md)) to load a tile.

## Under the Hood: Packing Bits

How does Valhalla store three numbers (level, tile ID, ID) in a single 64-bit integer (`uint64_t`)? It uses **bit manipulation**. Each component is assigned a specific number of bits within the 64 available bits.

The layout looks something like this (simplified, exact bit counts might vary slightly):

```
   64-bit unsigned integer (uint64_t): value
   +----------------------------------------------------------------+
   | Level (4 bits) | Tile ID (32 bits)        | ID (28 bits)       |
   +----------------------------------------------------------------+
   Bit 63         Bit 60                     Bit 28               Bit 0
```

*   **Level:** Occupies the highest 4 bits. This allows for 2^4 = 16 different hierarchy levels (0-15).
*   **Tile ID:** Occupies the next 32 bits. This allows for 2^32 = over 4 billion unique tiles per level, enough to cover the world.
*   **ID:** Occupies the lowest 28 bits. This allows for 2^28 = over 268 million unique node/element IDs *within* a single tile.

When you create a `GraphId(tile_id, level, id_in_tile)`, the constructor uses bit-shifting and bitwise OR operations to place each component into its designated slot within the 64-bit `value`.

```cpp
// Simplified conceptual constructor
// File: baldr/graphid.h

GraphId::GraphId(uint32_t tileid, uint32_t level, uint32_t id) {
  // Check if values fit within their allocated bits
  // ... (error checking omitted) ...

  // Shift level to the highest bits
  uint64_t level_part = static_cast<uint64_t>(level) << kLevelShift; // kLevelShift = 60

  // Shift tileid to the middle bits
  uint64_t tileid_part = static_cast<uint64_t>(tileid) << kTileIdShift; // kTileIdShift = 28

  // ID part stays in the lowest bits (no shift needed if kIdShift = 0)
  uint64_t id_part = static_cast<uint64_t>(id);

  // Combine the parts using bitwise OR
  value = level_part | tileid_part | id_part;
}
```

Similarly, the accessor methods like `level()`, `tileid()`, and `id()` use bit-shifting and bitwise AND operations (with masks) to extract the specific bits corresponding to each component.

```cpp
// Simplified conceptual accessor
// File: baldr/graphid.h

uint32_t GraphId::level() const {
  // Shift the level bits down to the lowest position
  // and mask out any higher bits (though none exist here)
  return (value >> kLevelShift);
}

uint32_t GraphId::tileid() const {
  // Shift the value down to align tileid bits, then mask out the level bits
  return (value >> kTileIdShift) & kTileIdMask; // kTileIdMask = 0xFFFFFFFF (32 bits)
}

uint32_t GraphId::id() const {
  // Just mask out the higher level and tileid bits
  return value & kIdMask; // kIdMask = 0x0FFFFFFF (28 bits)
}
```

This bit-packing technique is extremely efficient:
*   **Storage:** Only 8 bytes are needed to store the complete address.
*   **Comparison:** Checking if two `GraphId`s are the same is just a fast integer comparison.
*   **Passing:** Passing an 8-byte value around in the code is very cheap.

## Conclusion

The `GraphId` is a fundamental concept in Valhalla, providing a compact, unique, and globally valid "address" for elements within the routing graph, particularly `NodeInfo`s.

We learned that:

*   `GraphId` encodes the **Tile ID**, **Level**, and element **ID** (index within the tile).
*   It uses efficient **bit-packing** to store this information in a single 64-bit integer.
*   It's crucial for linking graph elements, such as connecting a `DirectedEdge` to its `endnode_`, especially across different `GraphTile`s.

With this universal addressing system, Valhalla can confidently navigate its vast, distributed map data. Now that we know how to identify specific road segments and intersections, how does Valhalla figure out the *cost* (like time or distance) of traveling along them? That's the topic of our next chapter!

**Next Up:** [Chapter 5: DynamicCost (Costing)](05_dynamiccost__costing_.md)

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)