# Hashgraph Specification

## Table of Contents

- [Hashgraph Specification](#hashgraph-specification)
  - [Table of Contents](#table-of-contents)
  - [Introduction](#introduction)
  - [Specification](#specification)
    - [Interface](#interface)
    - [Data Structure](#data-structure)
    - [Construction](#construction)
    - [Methods](#methods)
      - [Core Methods](#core-methods)
      - [Conflict Resolution](#conflict-resolution)
      - [Causal Relationship Methods](#causal-relationship-methods)
      - [Query Methods](#query-methods)
  - [Examples](#examples)

## Introduction

The Hashgraph is a directed acyclic graph (DAG) data structure that maintains a history of operations in a DRPObject. It is designed to support conflict resolution and causal ordering of operations.

## Specification

### Interface

The Hashgraph implementation is defined by the `IHashGraph` interface which includes the following key components:

```typescript
interface IHashGraph {
    peerId: string;
    vertices: Map<Hash, Vertex>;
    frontier: Hash[];
    forwardEdges: Map<Hash, Hash[]>;
    semanticsTypeDRP?: SemanticsType;

    // Core methods
    linearizeVertices(origin?: Hash, subgraph?: Set<string>): Vertex[];
    topologicalSort(updateBitsets?: boolean, origin?: Hash, subgraph?: Set<Hash>): Hash[];
    resolveConflicts(vertices: Vertex[]): ResolveConflictsType;
    createVertex(operation: Operation, dependencies?: Hash[], timestamp?: number): Vertex;
    addVertex(vertex: Vertex): void;
    
    // Conflict resolution
    resolveConflictsACL(vertices: Vertex[]): ResolveConflictsType;
    resolveConflictsDRP(vertices: Vertex[]): ResolveConflictsType;
    
    // Causal relationship methods
    areCausallyRelatedUsingBitsets(hash1: Hash, hash2: Hash): boolean;
    areCausallyRelatedUsingBFS(hash1: Hash, hash2: Hash): boolean;
    swapReachablePredecessors(hash1: Hash, hash2: Hash): void;
    
    // Query methods
    getFrontier(): Hash[];
    getDependencies(vertexHash: Hash): Hash[];
    getVertex(hash: Hash): Vertex | undefined;
    getLCA(dependencies: Hash[]): LowestCommonAncestorResult;
    getAllVertices(): Vertex[];
    getCurrentBitsetSize(): number;
}
```

### Data Structure

The Hashgraph is implemented as a directed acyclic graph with the following key components:

1. **Vertex**: Represents an operation in the system

   ```typescript
   interface Vertex {
       hash: string;           // Unique identifier
       peerId: string;         // ID of the peer that created the vertex
       operation: Operation;   // The operation performed
       dependencies: string[]; // References to parent vertices
       timestamp: number;      // Creation timestamp
       signature: Uint8Array;  // Cryptographic signature
   }
   ```

2. **Operation**: Represents the actual operation performed

   ```typescript
   interface Operation {
       drpType: string;  // Type of DRP operation
       opType: string;   // Type of operation
       value: any;       // Operation value
   }
   ```

3. **Graph Structure**:
   - `vertices`: Map of vertex hashes to Vertex objects
   - `frontier`: Array of hashes representing the current frontier vertices
   - `forwardEdges`: Map of vertex hashes to their outgoing edges
   - `reachablePredecessors`: Map of vertex hashes to their reachable predecessors (using bitsets)
   - `vertexDistances`: Map of vertex hashes to their distances from the root

### Construction

The Hashgraph is constructed with the following parameters:

```typescript
interface HashGraphOptions {
    peerId: string;                    // Required: ID of the peer
    resolveConflictsACL?: ResolveConflictFn;  // Optional: ACL conflict resolution function
    resolveConflictsDRP?: ResolveConflictFn;  // Optional: DRP conflict resolution function
    semanticsTypeDRP?: SemanticsType;  // Optional: DRP semantics type
    logConfig?: LoggerOptions;         // Optional: Logger configuration
}
```

The construction process:

1. Validates the required `peerId` parameter
2. Initializes the root vertex with a predefined hash
3. Sets up the initial frontier with the root vertex
4. Initializes empty maps for vertices, forward edges, and other data structures
5. Configures logging and conflict resolution functions

### Methods

#### Core Methods

1. **createVertex**
   - Creates a new vertex with the given operation and dependencies
   - Parameters:
     - `operation`: The operation to perform
     - `dependencies`: Optional array of parent vertex hashes (defaults to frontier)
     - `timestamp`: Optional creation timestamp (defaults to current time)
   - Returns: Newly created Vertex

2. **addVertex**
   - Adds a vertex to the graph
   - Updates the frontier and forward edges
   - Computes vertex distances
   - Parameters:
     - `vertex`: The vertex to add
   - Returns: void
   - Time Complexity: O(length(dependencies))

3. **topologicalSort**
   - Performs topological sorting of vertices
   - Can update bitsets for efficient causal relationship checking
   - Parameters:
     - `updateBitsets`: Whether to update bitsets
     - `origin`: Starting vertex hash (defaults to root)
     - `subgraph`: Set of vertex hashes to include
   - Returns: Array of topologically sorted vertex hashes
   - It is deterministic, before pushing vertices onto the stack, the neighbors are sorted by hash
   - Time Complexity: O(V log V + E)

4. **linearizeVertices**
   - Linearizes vertices based on the configured semantics type
   - Parameters:
     - `origin`: Starting vertex hash (defaults to root)
     - `subgraph`: Set of vertex hashes to include
   - Returns: Array of linearized vertices

#### Conflict Resolution

1. **resolveConflicts**
   - Resolves conflicts between vertices
   - Delegates to ACL or DRP conflict resolution based on operation type
   - Parameters:
     - `vertices`: Array of vertices to resolve conflicts between
   - Returns: ResolveConflictsType with action and optional vertices
   - Time Complexity: depends on the conflict resolution function

2. **resolveConflictsACL** and **resolveConflictsDRP**
   - Default implementations return no-op actions
   - Can be overridden with custom conflict resolution logic

#### Causal Relationship Methods

1. **areCausallyRelatedUsingBitsets**
   - Efficiently checks if two vertices are causally related using bitsets
   - Parameters:
     - `hash1`: First vertex hash
     - `hash2`: Second vertex hash
   - Returns: boolean indicating causal relationship

2. **areCausallyRelatedUsingBFS**
   - Checks causal relationship using breadth-first search
   - Parameters:
     - `hash1`: First vertex hash
     - `hash2`: Second vertex hash
   - Returns: boolean indicating causal relationship

3. **swapReachablePredecessors**
   - Swaps the reachable predecessors of two vertices
   - Used in conflict resolution
   - Parameters:
     - `hash1`: First vertex hash
     - `hash2`: Second vertex hash
   - Returns: void

#### Query Methods

1. **getFrontier**
   - Returns the current frontier vertices
   - Returns: Array of vertex hashes

2. **getDependencies**
   - Returns dependencies of a vertex
   - Parameters:
     - `vertexHash`: Hash of the vertex
   - Returns: Array of dependency hashes

3. **getVertex**
   - Retrieves a vertex by hash
   - Parameters:
     - `hash`: Vertex hash
   - Returns: Vertex or undefined

4. **getLCA**
   - Finds the lowest common ancestor of dependencies
   - Parameters:
     - `dependencies`: Array of dependency hashes
   - Returns: LowestCommonAncestorResult with LCA and linearized vertices

## Examples
