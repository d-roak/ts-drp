# DRPObject Specification

## Table of Contents

- [Introduction](#introduction)
- [Construction](#construction)
- [Interface](#interface)
- [Constructor](#constructor)
- [Methods](#methods)
- [Update Pipelines](#update-pipelines)
- [State Management](#state-management)


## Introduction

A **DRPObject** represents a shared, real-time synchronized state within the Distributed Real-time Programs (DRP) network. It allows nodes (participants) to collaboratively modify state, ensuring eventual consistency, automatic conflict resolution, and decentralized finality. 

### Characteristics

- **Decentralized:** No central coordinator; each node independently synchronizes state.
- **Eventual Consistency:** All nodes eventually agree on the same state.
- **Conflict Resolution:** Deterministic and automated resolution using causal ordering.
- **Access Control:** Fine-grained permissions via ACL.
- **Finality:** Achieved through threshold signatures (BLS signatures).

### Core Components

- `HashGraph`: Manages causal history of operations.
- `ACL`: Controls nodes permissions.
- `VertexApplier`: Applies operations and manages internal state.
- `FinalityStore`: Tracks the finality status of operations.

Think of `DRPObject` as:
- A function call interceptor.
- A deterministic state transition machine.
- A local CRDT + ACL + finality state snapshotter.

## Specification

### Interface

Defines the core properties and methods required to implement a **DRPObject**.

```typescript
export interface IDRPObject<T extends IDRP> extends DRPObjectBase {
  /** Unique identifier for the DRPObject instance */
  readonly id: string;

  /** Access Control List (ACL) managing permissions for the object */
  acl: IACL;

  /** Application-specific DRP logic object */
  drp?: T;

  /** Array of vertices (operations) applied to this object */
  vertices: Vertex[];

  /** FinalityStore tracking the finalization status of vertices */
  finalityStore: IFinalityStore;

  /**
   * Retrieves DRP and ACL state associated with a specific vertex.
   * @param vertexHash - Hash identifying the vertex.
   * @returns Tuple containing `[DRPState, ACLState]`, undefined if state missing.
   */
  getStates(vertexHash: string): [DRPState | undefined, DRPState | undefined];

  /**
   * Sets the ACL state for a given vertex.
   * @param vertexHash - Hash identifying the vertex.
   * @param aclState - The ACL state to associate with this vertex.
   */
  setACLState(vertexHash: string, aclState: DRPState): void;

  /**
   * Sets the DRP state for a given vertex.
   * @param vertexHash - Hash identifying the vertex.
   * @param drpState - The DRP state to associate with this vertex.
   */
  setDRPState(vertexHash: string, drpState: DRPState): void;

  /**
   * Subscribes a callback to be notified when the object state changes.
   * @param callback - Function called upon state updates.
   */
  subscribe(callback: DRPObjectCallback<T>): void;

  /**
   * Applies a list of vertices (operations) to the local object state.
   * Typically used for synchronizing remote state.
   * @param vertices - Vertices to apply.
   * @returns Result detailing applied/missing vertices.
   */
  applyVertices(vertices: Vertex[]): Promise<ApplyResult>;

  /**
   * @deprecated Use applyVertices instead.
   * Merges a list of vertices into the local state.
   * @param vertices - Vertices to merge.
   * @returns Result detailing applied/missing vertices.
   */
  merge(vertices: Vertex[]): Promise<MergeResult>;
}
```

### Constructor

The `DRPObject` is constructed using the following options:

```typescript
interface DRPObjectOptions<T extends IDRP> {
  peerId: string;                    // Required: Unique ID of the creating node
  id?: string;                       // Optional: Explicit ID for the DRPObject (default derived from peerId)
  acl?: IACL;                        // Optional: ACL instance managing access permissions (default permissionless)
  drp: T;                            // Required: Application-specific DRP logic instance
  config?: {                         // Optional: Configuration object
    log_config?: LoggerOptions;      // Optional: Logger configuration
    finality_config?: FinalityConfig;// Optional: Finality-related configuration
  };
}
```

The construction process:
1. **Validation**: Confirms the required parameters (peerId, drp) are provided.

2. **Object ID Initialization**: Generates a unique id from the provided peerId if an explicit id is not given.

3. **ACL Initialization**: Creates an ACL instance (defaults to a permissionless ACL if not provided).

4. **HashGraph Setup**:

  -  Initializes the HashGraph with the provided peerId.

  - Configures conflict resolution strategies using ACL and DRP methods.

  - Sets the semantics type (semanticsType) from the provided DRP logic.

5. **FinalityStore Setup**: Initializes tracking for the finalization status of vertices (operations).

6. **VertexApplier Setup**: Creates the internal vertex applier to validate and apply state updates, link states, manage conflicts, and handle notifications.

7. **Logging**: Initializes logging based on provided configurations.

### Methods

#### Core Methods

1. **`get drp()`**
   - Retrieves the DRP logic object associated with this DRPObject.
   - **Returns:** `T | undefined` – The DRP instance or undefined if not set.

2. **`get acl()`**
   - Retrieves the Access Control List (ACL) instance managing permissions.
   - **Returns:** `IACL` – The current ACL instance.

3. **`get vertices()`**
   - Retrieves all vertices (operations) applied to the DRPObject.
   - **Returns:** `Vertex[]` – Array of vertices representing applied operations.

4. **`get finalityStore()`**
   - Retrieves the finality tracking store for the DRPObject.
   - **Returns:** `IFinalityStore` – Finality store tracking operation finalization.

#### State Management Methods

5. **`getStates`**
   - Retrieves the ACL and DRP state snapshots for a given vertex hash.
   - **Parameters:**
     - `vertexHash: string` – Hash identifying the target vertex.
   - **Returns:** `[DRPState | undefined, DRPState | undefined]` – Tuple of ACL and DRP states.

6. **`setACLState`**
   - Sets the ACL state snapshot associated with a given vertex hash.
   - **Parameters:**
     - `vertexHash: string` – Hash identifying the target vertex.
     - `aclState: DRPState` – ACL state to store.
   - **Returns:** `void`

7. **`setDRPState`**
   - Sets the DRP state snapshot associated with a given vertex hash.
   - **Parameters:**
     - `vertexHash: string` – Hash identifying the target vertex.
     - `drpState: DRPState` – DRP state to store.
   - **Returns:** `void`

#### Synchronization Methods

8. **`applyVertices`**
   - Applies an array of vertices (remote operations) to synchronize the DRPObject.
   - **Parameters:**
     - `vertices: Vertex[]` – Vertices to apply.
   - **Returns:** `Promise<ApplyResult>` – Result detailing applied and missing vertices.

9. **`merge`** *(deprecated)*
   - Deprecated method; use `applyVertices` instead.
   - Merges vertices into the DRPObject.
   - **Parameters:**
     - `vertices: Vertex[]` – Vertices to merge.
   - **Returns:** `Promise<MergeResult>` – Result detailing merged vertices.

#### Event Handling Methods

10. **`subscribe`**
    - Subscribes a callback function to be invoked on object state changes.
    - **Parameters:**
      - `callback: DRPObjectCallback<T>` – Function invoked upon state updates.
    - **Returns:** `void`

#### Internal Methods *(private)*

11. **`_notify`** *(internal use only)*
    - Internally triggers subscribed callbacks after state changes.
    - **Parameters:**
      - `origin: string` – Origin of the notification event.
      - `vertices: Vertex[]` – Vertices triggering the notification.
    - **Returns:** `void`


### DRPObject Update Pipelines

Distributed Real-time Programs (DRP) support collaborative, real-time state updates across a decentralized network. DRPObjects handle these updates through specialized pipelines that enforce consistency, access control, and deterministic state transitions. The DRPObject component manages these updates via two key pipelines, ensuring deterministic, secure, and synchronized state across nodes.

#### 1. Local Updates: `callFnPipeline`

Local updates are initiated by calling methods on a DRP or ACL object (e.g., granting permission, modifying state). These go through the `callFnPipeline`, which includes strict validation, permission enforcement, and notification.

**Steps in `callFnPipeline`:**

1. **Vertex Creation**  
   A new vertex is created based on the DRP method, operation type, and arguments.

2. **Validation**  
   The vertex is validated to ensure it meets structural and dependency constraints.

3. **LCA Computation**  
   The Lowest Common Ancestor (LCA) is computed to determine the merge point.

4. **Dependency Replay**  
   DRP and ACL operations from the LCA to the frontier are replayed to reconstruct the current state.

5. **Permission Checking**  
   The pipeline checks whether the calling peer has `Writer` permission (ACL operations are exempt).

6. **Operation Execution**  
   The specified method is executed on the DRP or ACL logic using a proxy bound to the caller's context.

7. **Equality Check**  
   If the state has not changed (deep equal), the operation is skipped as a no-op.

8. **State Assignment**  
   The result of the operation is applied to the in-memory DRP/ACL instance.

9. **State Snapshot**  
   Snapshots of the updated state are stored under the vertex hash for deterministic reconstruction.

10. **Vertex Insertion**  
    The new vertex is inserted into the HashGraph.

11. **Finality Initialization**  
    The finality store is initialized using the list of current finality signers.

12. **Notification**  
    Subscribed callbacks are triggered via `notify("callFn", [vertex])`.

---

#### Remote Updates: `applyVertexPipeline`

Remote updates arrive as a set of vertices from peers. These are validated and merged through the `applyVertexPipeline`.

**Steps in `applyVertexPipeline`:**

1. **Deduplication**  
   Already known vertices are ignored.

2. **Validation**  
   Each incoming vertex is validated structurally and causally.

3. **LCA and Dependency Replay**  
   The LCA is used to replay operations and reconstruct the correct state before applying the new vertex.

4. **Permission Checking**  
   Writer permissions are enforced for DRP operations based on ACL state.

5. **Operation Execution**  
   The method is executed and the resulting state is applied to the internal proxy.

6. **State Snapshot and Vertex Insertion**  
   Updated state snapshots are stored, and the vertex is added to the HashGraph.

7. **Finality Initialization**  
   Finality tracking begins for newly applied vertices.

8. **No Notification**  
   Since the vertex originated remotely, subscribers are not notified.

### Summary of Differences

#### `callFnPipeline` (Local Updates)
- Performs deep equality checks to detect no-op changes
- Triggers local subscriber notifications after successful updates

#### `applyVertices` (Remote Updates)
- Skips already known vertices (deduplication)
- Applies updates without notifying local subscribers

#### Shared Logic in Both Pipelines
- Validate operation structure and access permissions
- Compute LCA and replay dependencies for consistent state
- Execute and assign DRP/ACL state updates
- Snapshot updated state
- Insert vertex into the HashGraph
- Initialize finality tracking for consensus

## State Management

The `DRPObjectStateManager` tracks the **state snapshots associated with each vertex** in the HashGraph. These snapshots represent the state of the DRP and ACL objects **after** applying all operations up to a given vertex.

### Purpose

- Stores the DRP and ACL state **per vertex hash**.
- Enables **deterministic reconstruction** of object state during LCA-based computation.

### Behavior

- When a vertex is applied, its resulting DRP and ACL states are serialized and stored.
- These states are later used to recreate the exact DRP/ACL instances at that point in the graph.

### Usage

```typescript
// Save state after applying a vertex
stateManager.setDRPState(vertexHash, stateFromDRP(drp));
stateManager.setACLState(vertexHash, stateFromDRP(acl));

// Reconstruct state at a specific vertex
const [drp, acl] = stateManager.fromHash(vertexHash);
```