# Chain Table in 3FS

## Overview

Chain tables are a fundamental component of the 3FS (Fire-Flyer File System) distributed storage system. They implement the Chain Replication with Apportioned Queries (CRAQ) protocol to provide strong consistency while maximizing read throughput in a distributed environment.

A chain table organizes storage targets into chains, where each chain is responsible for storing and replicating a subset of data chunks. This organization enables efficient data placement, replication, and recovery in the event of storage target failures.

## Chain Table Structure

The chain table structure is defined in `src/fbs/mgmtd/ChainTable.h`:

```cpp
struct ChainTable {
  uint64_t id;
  uint64_t version;
  std::string name;
  uint32_t numChains;
  uint32_t replicaFactor;
  uint32_t stripeSize;
  LayoutType layoutType;
  std::vector<ChainInfo> chains;
  std::optional<uint64_t> seed;
};
```

Key components:
- **id**: Unique identifier for the chain table
- **version**: Version number that increments when the chain table is updated
- **name**: Human-readable name for the chain table
- **numChains**: Total number of chains in the table
- **replicaFactor**: Number of replicas for each chunk (typically 3)
- **stripeSize**: Number of consecutive chunks assigned to the same chain
- **layoutType**: Type of layout (Empty, ChainRange, or ChainList)
- **chains**: Vector of ChainInfo structures representing the chains
- **seed**: Optional seed for random chain selection

## Chain Information

Each chain in the chain table is represented by a `ChainInfo` structure defined in `src/fbs/mgmtd/ChainInfo.h`:

```cpp
struct ChainInfo {
  uint64_t id;
  uint64_t version;
  std::vector<ChainTargetInfo> targets;
};
```

Key components:
- **id**: Unique identifier for the chain
- **version**: Version number that increments when the chain is updated
- **targets**: Vector of ChainTargetInfo structures representing the targets in the chain

## Chain Target Information

Each target in a chain is represented by a `ChainTargetInfo` structure defined in `src/fbs/mgmtd/ChainTargetInfo.h`:

```cpp
struct ChainTargetInfo {
  uint64_t id;
  std::string address;
  uint16_t port;
  uint32_t targetId;
  TargetState state;
};
```

Key components:
- **id**: Unique identifier for the target
- **address**: IP address or hostname of the storage node
- **port**: Port number for the storage service
- **targetId**: Identifier for the target within the storage node
- **state**: State of the target (serving, syncing, waiting, lastsrv, or offline)

## Chain References

Chain references are used to map chunk IDs to chains. They are defined in `src/fbs/mgmtd/ChainRef.h`:

```cpp
struct ChainRef {
  uint64_t chainTableId;
  uint64_t chainTableVersion;
  uint64_t chainId;
};
```

Key components:
- **chainTableId**: ID of the chain table containing the chain
- **chainTableVersion**: Version of the chain table
- **chainId**: ID of the chain within the table

## Mapping from Chunk ID to Chain ID

The mapping from chunk ID to chain ID is a three-step process:

1. **File offset to chunk index**: The file offset is divided by the chunk size to get the chunk index.
2. **Chunk index to chain reference**: The chunk index is mapped to a chain reference using the `getChainOfChunk` method in `Schema.cc`.
3. **Chain reference to chain ID**: The chain reference is used to look up the chain ID in the routing information.

### Step 1: File offset to chunk index

```cpp
uint64_t chunkIndex = fileOffset / chunkSize;
```

### Step 2: Chunk index to chain reference

The `getChainOfChunk` method in `Schema.cc` maps a chunk index to a chain reference:

```cpp
ChainRef Schema::getChainOfChunk(uint64_t chunkIndex) const {
  if (layout_.layoutType == LayoutType::Empty) {
    return ChainRef{0, 0, 0};
  }

  uint64_t chainIndex;
  if (layout_.layoutType == LayoutType::ChainRange) {
    chainIndex = (chunkIndex / layout_.stripeSize) % layout_.numChains;
  } else {  // ChainList
    if (layout_.seed) {
      std::mt19937 gen(*layout_.seed);
      std::vector<uint64_t> indices(layout_.numChains);
      std::iota(indices.begin(), indices.end(), 0);
      std::shuffle(indices.begin(), indices.end(), gen);
      chainIndex = indices[chunkIndex % layout_.numChains];
    } else {
      chainIndex = chunkIndex % layout_.numChains;
    }
  }

  return ChainRef{layout_.chainTableId, layout_.chainTableVersion,
                 layout_.chainIds[chainIndex]};
}
```

This method handles different layout types:
- **Empty**: Returns a zero chain reference
- **ChainRange**: Uses a striping approach where consecutive chunks are assigned to the same chain
- **ChainList**: Uses a modulo operation to distribute chunks across chains, with optional shuffling using a random seed

### Step 3: Chain reference to chain ID

The chain reference is used to look up the chain ID in the routing information:

```cpp
uint64_t chainId = routingInfo.getChain(chainRef).id;
```

## Chain Table Management

Chain tables are managed by the management daemon (`mgmtd_main`), which is responsible for:
- Creating and updating chain tables
- Monitoring the health of storage targets
- Handling target failures and recoveries
- Updating chain tables when targets change state

When a storage target fails, the management daemon:
1. Marks the target as offline
2. Moves the target to the end of its chains
3. Increments the chain version
4. Broadcasts the updated chain table to all services and clients

## Load Balancing During Recovery

When a storage target fails, its read traffic must be redirected to other targets that contain replicas of the same data. This redirection can create hotspots and bottlenecks if not carefully managed.

### Balanced Incomplete Block Design

To achieve maximum read throughput during recovery, 3FS uses a mathematical approach called Balanced Incomplete Block Design (BIBD) to optimize the distribution of redirected read traffic.

A Balanced Incomplete Block Design is a concept from combinatorial mathematics that arranges elements into blocks such that:
1. Each block contains the same number of elements (k)
2. Each element appears in the same number of blocks (r)
3. Any pair of distinct elements appears together in exactly λ blocks

In the context of 3FS chain tables:
- The "elements" are the storage targets
- The "blocks" are the chains
- The goal is to ensure that when any target fails, its load is evenly distributed among the remaining targets

### Example: Simple vs. Balanced Chain Tables

Consider two different chain table configurations:

**Simple Chain Table:**
```
| Chain | Version | Target 1 (head) | Target 2 | Target 3 (tail) |
| :---: | :-----: | :-------------: | :------: | :-------------: |
|   1   |    1    |      `A1`       |   `B1`   |      `C1`       |
|   2   |    1    |      `D1`       |   `E1`   |      `F1`       |
|   3   |    1    |      `A2`       |   `B2`   |      `C2`       |
```

In this configuration, if node A fails, all its read traffic would be redirected to nodes B and C, which would immediately become bottlenecks.

**Balanced Chain Table:**
```
| Chain | Version | Target 1 (head) | Target 2 | Target 3 (tail) |
| :---: | :-----: | :-------------: | :------: | :-------------: |
|   1   |    1    |      `B1`       |   `E1`   |      `F1`       |
|   2   |    1    |      `A1`       |   `B2`   |      `D1`       |
|   3   |    1    |      `A2`       |   `D2`   |      `F2`       |
```

In this improved configuration, node A is paired with every other node. When A fails, each of the other nodes receives 1/5 of A's read traffic, distributing the load more evenly.

### Integer Programming Formulation

To find the optimal BIBD for a given set of storage targets, the problem can be formulated as an integer programming problem:

**Variables:**
- Binary variable x_ij = 1 if target i is assigned to chain j, 0 otherwise

**Constraints:**
1. Each chain must have exactly k targets: ∑_i x_ij = k for all j
2. Each target must appear in exactly r chains: ∑_j x_ij = r for all i
3. Any pair of targets i and i' must appear together in exactly λ chains: ∑_j x_ij × x_i'j = λ for all i ≠ i'

**Objective:**
- Minimize the maximum load on any target when any other target fails

This is a complex optimization problem that requires specialized solvers. The design_notes.md mentions using an "integer programming solver" to find the optimal solution.

## Chain Table Implementation

The chain table implementation in 3FS involves several key components:

### Chain Allocator

The `ChainAllocator` class in `src/meta/components/ChainAllocator.h` is responsible for allocating chains for files:

```cpp
class ChainAllocator {
 public:
  ChainAllocator(std::shared_ptr<KVStore> kvStore);

  CoTryTask<ChainRef> allocateChain(Transaction &txn, uint64_t inodeId,
                                   uint64_t version, uint64_t chunkIndex);

  CoTryTask<std::vector<ChainRef>> allocateChains(
      Transaction &txn, uint64_t inodeId, uint64_t version,
      const std::vector<uint64_t> &chunkIndices);

  CoTryTask<void> freeChains(Transaction &txn, uint64_t inodeId,
                            uint64_t version);

 private:
  std::shared_ptr<KVStore> kvStore_;
};
```

### Routing Information

The `RoutingInfo` class in `src/fbs/mgmtd/RoutingInfo.cc` manages the mapping between chain references and chains:

```cpp
const ChainInfo &RoutingInfo::getChain(const ChainRef &chainRef) const {
  auto it = chainTables_.find(chainRef.chainTableId);
  if (it == chainTables_.end()) {
    throw std::runtime_error("Chain table not found");
  }

  const auto &chainTable = it->second;
  if (chainTable.version != chainRef.chainTableVersion) {
    throw std::runtime_error("Chain table version mismatch");
  }

  for (const auto &chain : chainTable.chains) {
    if (chain.id == chainRef.chainId) {
      return chain;
    }
  }

  throw std::runtime_error("Chain not found");
}
```

### Chain Table CLI Commands

The 3FS system provides CLI commands for managing and inspecting chain tables:

- `ListChainTables`: Lists all chain tables in the system
- `DumpChainTable`: Dumps the details of a specific chain table

These commands are implemented in `src/client/cli/admin/ListChainTables.cc` and `src/client/cli/admin/DumpChainTable.cc`.

## Conclusion

Chain tables are a critical component of the 3FS distributed storage system, providing a flexible and efficient mechanism for data placement, replication, and recovery. By using the Chain Replication with Apportioned Queries (CRAQ) protocol and optimizing load balancing through Balanced Incomplete Block Design, 3FS achieves strong consistency while maximizing read throughput, even during recovery scenarios.

The chain table system demonstrates how theoretical concepts from mathematics and distributed systems can be applied to solve practical problems in high-performance storage systems.
