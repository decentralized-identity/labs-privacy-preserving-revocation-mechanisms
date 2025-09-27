
# [WIP] An Analysis of Revocation Schemes for Verifiable Credentials

## Executive Summary

This report provides a detailed technical analysis of prominent revocation schemes for Verifiable Credentials (VCs). We examine each scheme’s specifications, implementations, and privacy properties. The primary objective is to delineate each scheme’s pros and cons, and identify core use cases and recommend the most suitable revocation scheme for each.

---
## 0. Linkablity
One of the major privacy risks of verifiable credentials (VCs) is known as linkability. Linkability refers to the possibility of tracking individuals by collating data from their use of digital credentials. This risk arises when traceable digital signatures or identifiers are reused, allowing different parties to correlate multiple interactions with the same individual, thereby compromising privacy. Such linkability can create surveillance potential across society, whether by private companies, state actors, or even foreign adversaries.

Linkability can occur at different layers. For example, at the credential level, the chosen revocation method can contribute to this risk.

![Linkability in Verifiable Credentials](images/linkabillity.png)
Typically, we can distinguish two main types of linkability:
1. RP–RP linkability: Verifiers share or match data to link different interactions back to the same individual.
2. AP–RP linkability: An issuer and a verifier collude to link different interactions back to the same individual.

To achieve true unlinkability, both the signature scheme and the revocation method must support it. 

## 1. The Need for Revocation in Verifiable Credentials

The lifecycle of a VC does not end at issuance; its validity may need to be nullified before its expiration. This process, known as revocation, is critical for maintaining the integrity and trustworthiness of a digital identity ecosystem.

### 1.1 Stakeholders in the Revocation Process

The revocation process involves several key stakeholders:

* **Issuer:** The entity that issues the credential and, in most cases, manages its revocation status.
* **Holder:** The individual or entity that possesses the credential and presents it for verification.
* **Verifier (Relying Party):** The entity that checks the validity of a presented credential.
* **Revocation Authority:** The entity responsible for maintaining and publishing revocation information. This role may be fulfilled by the Issuer.

### 1.2 Scenarios Requiring Revocation

Revocation becomes necessary in various scenarios, including:

1. **Credential Information Update:** An existing credential with outdated information is revoked and reissued with updated details.

   * **Initiator:** Holder or Issuer.
   * **Scope:** Ecosystem-wide (all verifiers), affects a single credential.

2. **Holder's Private Key Compromise:** If a Holder's private key is lost or stolen, all credentials associated with that key must be revoked to prevent misuse.

   * **Scope:** Ecosystem-wide, affects all credentials verifiable with the Holder's public key.

3. **Issuer's Private Key Compromise:** If an Issuer's private key is compromised, all credentials signed with that key must be invalidated to protect all stakeholders.

   * **Scope:** Ecosystem-wide, affects all credentials verifiable with the Issuer's public key.

4. **Revocation Authority's Private Key Compromise:** Similar to an issuer compromise, this necessitates revoking the entire revocation system's validity.

5. **Blocking Malicious Users[^not-strict]:** A specific Verifier may choose to no longer accept a credential from a user who has violated its policies.

   * **Initiator:** Verifier.
   * **Scope:** Verifier-local, affects a single credential.

6. **Revocation for the Right to be Forgotten[^not-strict]:** A Holder may voluntarily revoke a credential with respect to a specific Verifier to exercise their privacy rights.

   * **Initiator:** Holder.
   * **Scope:** Verifier-local, affects a single credential.

[^not-strict]: These two items are policy-based blocks rather than classical credential revocation; they don’t remove the credential’s cryptographic validity, only its acceptability by certain parties.

---

## 2. Revocation Methods by a Revocation Authority

Several technical methods exist to manage universal revocation statuses for scenarios 1-4.

### 2.1 Bitstring-based (StatusList2021)

This method maps each credential to a single bit in a fixed-length bitstring. When a credential is revoked, its corresponding bit is flipped (e.g., from 0 to 1). This approach is standardized by the W3C as **StatusList2021**.

### 2.2 Accumulator-based

Cryptographic accumulators provide a way to represent a set of elements with a single, constant-size value. A credential's status can be proven by checking its membership (or non-membership) in the accumulator.

* **RSA-based:** Schemes like the CL-Accumulator are based on the strong RSA assumption.
* **Pairing-based:** More recent schemes, such as the VB20 and KB21 accumulators, leverage elliptic curve pairings for greater efficiency.

### 2.3 Range Proof-based

This technique uses zero-knowledge proofs, such as Bulletproofs, to prove that a credential's non-revoked ID lies outside of a specific range of revoked IDs. It offers a balance between privacy and efficiency.

### 2.4 Merkle Tree-based

This method uses a Merkle Tree to maintain the list of revoked credentials. A proof of revocation (or non-revocation) is provided via a Merkle Path, offering an efficient and verifiable data structure. The Incremental Merkle Tree (IMT) is commonly used for membership proofs. This data structure is used by projects such as [Semaphore](https://semaphore.pse.dev/) and [World](https://world.org/). There is an optimized implementation of an IMT called [LeanIMT](https://zkkit.org/leanimt-paper.pdf). For non-membership proofs, where users prove that their credential is not included in a list of revoked credentials, the [Sparse Merkle Tree (SMT)](https://docs.iden3.io/publications/pdfs/Merkle-Tree.pdf) is commonly used. [Privado.iD](https://www.privado.id/) is an example of a project using this data structure.



[TODO] Create a table of methods and their categories.

---

## 3. Comparative Analysis of Accumulator-based Schemes

This section focuses on a comparison between two prominent pairing-based accumulators, as RSA-based methods are generally considered too slow for many modern applications.

### 3.1 Comparison Criteria

* **Computational Cost:** The complexity of generating and verifying proofs.
* **Data Size:** The size of the witness and the public revocation list.

### 3.2 Detailed Comparison: VB20 vs. KB21 Accumulators

We compare the dynamic universal accumulator from Vitto–Biryukov 2020 (**VB20**) and the pairing-based accumulator from Karantaidou–Baldimtsi 2021 (**KB21**).

**Key Differences:**

1. **Behavior on Additions:**

   * **VB20:** The accumulator value \$V\$ is updated for every element addition.
   * **KB21:** Employs "static updates," where the accumulator value is *not* updated on additions (assuming non-membership proofs are not required).

2. **Witness Update Mechanism:**

   * **VB20:** The server distributes an update polynomial (with public coefficients) derived from the batch of additions/deletions. Each user evaluates this polynomial locally to update their witness.
   * **KB21:** No witness updates are needed for additions. Updates are only required for deletions.

3. **Asymptotic Complexity (for a batch of *m* updates):**

   * **VB20:** Communication: \$O(m)\$, User computation: \$O(m)\$, Server computation: \$O(m^2)\$.
   * **KB21:** Communication: \$O(m)\$, User computation: \$O(m)\$, Server computation: \$O(1)\$.

   Based on [ALLOSAUR](https://eprint.iacr.org/2022/1362)

### 3.3 Benchmark Results

The following results are based on empirical measurements, comparing batch update performance.

The graph illustrates that the VB accumulator's execution time for batch updates scales significantly better than the KB accumulator's as the number of elements grows.

**Performance Metrics:**

| Operation                   | VB Accumulator | KB Accumulator  |
| :-------------------------- | :------------- | :-------------- |
| **Witness Generation Time** | 1.7 ms         | \~170 ms (Est.) |
| **Witness Size**            | 144 bytes      | 288 bytes       |

**Batch Update Execution Time (Add + Delete operations):**

![VB vs KB Execution Time](images/vb_kb_execution_time.png)

| Number of Elements | VB Accumulator | KB Accumulator |
| :----------------- | :------------- | :------------- |
| **1,000**          | 0.33 ms        | 30.98 ms       |
| **10,000**         | 2.24 ms        | 296.53 ms      |
| **100,000**        | 21.08 ms       | 2,855.11 ms    |
| **1,000,000**      | 286.96 ms      | 30,053.34 ms   |

### 3.4 Overall Assessment

* **VB Accumulator:** Empirical measurements show that the VB accumulator is overwhelmingly faster in practice, making it highly suitable for large-scale systems requiring frequent batch updates.
* **KB Accumulator:** While it boasts a superior theoretical server computation complexity of \$O(1)\$, its performance in practice is substantially slower.

In conclusion, the **VB accumulator demonstrates superior practical performance**. However, the **KB accumulator's** feature of not requiring accumulator updates for additions remains an attractive property that could be beneficial in specific use cases with low revocation rates.

## 4. Comparative Analysis of Merkle Tree-based Schemes

This section focuses on a comparison between two prominent Merkle tree-based schemes - LeanIMT and SMT.
Both approaches are usually implemented with Circom + SnarkJS, using Groth16 as the proving system where these proofs are also compatible with other proving systems such as Plonk, Fflonk, and UltraHonk, which ensures fast client-side generation and low-cost smart contract verification. Since Groth16 is commonly used, these methods require a trusted setup per circuit. 

### 4.1 Comparison Criteria

* **Computational Performance:** The latency and throughput of tree operations including insert, update, proof generation, and proof verification.
* **Data Size:** The size of cryptographic proofs and storage requirements for the tree structure.
* **Constraint Efficiency:** The number of non-linear constraints required in zero-knowledge circuits, affecting proving time and verification costs.

#### Operations Overview

* **Insert:** Adding a new credential to the merkle tree structure.
* **Update:** Modifying the tree by adding, removing, or changing credential entries.
* **Generate Proof:** Creating merkle proofs off-chain that credential holders can use to demonstrate membership (LeanIMT) or non-membership (SMT) in revocation lists.
* **Verify Proof:** Validating merkle proofs, performed either on-chain by smart contracts or off-chain by other verifying parties.

### 4.2 Benchmark Environment

**System Specifications:**
- Computer: MacBook Pro
- Chip: Apple M2 Pro
- Memory (RAM): 16 GB
- Operating System: macOS Sequoia version 15.6.1

**Software Environment:**
- Node.js version: 23.10.0
- Circom compiler version: 2.2.2
- Snarkjs version: 0.7.5

Benchmark code is available at: https://github.com/vplasencia/vc-revocation-benchmarks

### 4.3 Theoretical Evaluation

| Category | LeanIMT | SMT |
| :----------------- | :------------- | :------------- |
| **Time Complexitiy**  | O(log n)[^operation]        | TBW       |
| **Space Complexitiy** | O(n)        | TBW      |

In this table, n is the number of leaves in the Merkle tree.

[^operation] The time complexity of Insert, Update, Generate Merkle Proof and Verify Merkle Proof.

### 4.4 Benchmark Results: LeanIMT v.s. SMT

#### Performance Comparison Between LeanIMT and SMT

| Operation | LeanIMT Latency (ns) | SMT Latency (ns) | LeanIMT Throughput (ops/s) | SMT Throughput (ops/s) | Performance Advantage |
|-----------|---------------------|------------------|---------------------------|------------------------|----------------------|
| **Insert** | 270,917 ± 1.59% | 2,297,386 ± 11.42% | 3,710 ± 1.27% | 475 ± 3.31% | LeanIMT 8.5x faster |
| **Update** | 547,897 ± 1.72% | 2,075,322 ± 5.41% | 1,836 ± 1.38% | 498 ± 2.57% | LeanIMT 3.8x faster |
| **Generate Proof** | 159.57 ± 33.66% | 52,023 ± 16.39% | 7,583,898 ± 2.84% | 21,040 ± 2.86% | LeanIMT 326x faster |
| **Verify Proof** | 553,753 ± 1.64% | 996,201 ± 5.94% | 1,817 ± 1.52% | 1,041 ± 2.55% | LeanIMT 1.8x faster |

#### Data Size Comparison Between LeanIMT and SMT
| Tree Implementation | Proof Size (bytes) | Proof Size (KB on disk) | Storage Efficiency |
|---------------------|-------------------|------------------------|-------------------|
| **LeanIMT** | 807 | 4 | Optimized |
| **SMT** | 804 | 4 | Standard |

**Note**: Data based on tree depth 10. Both implementations show nearly identical proof sizes with minimal difference.

#### Number of Non-linear Constraints
Constraints are mathematical equations or rules that must be satisfied in cryptographic computations, with non-linear constraints being more complex than simple linear equations. Fewer constraints are preferable because they result in faster proving and verification times, reduced computational overhead, and lower memory requirements. In zero-knowledge proof systems, minimizing constraints leads to smaller proof sizes, cheaper transaction costs, and improved scalability. Additionally, systems with fewer constraints are easier to debug, optimize, and maintain, while also reducing potential security vulnerabilities.

![# of Constraints Across Tree Depth](images/merkle_tree/IMT_SMT_num_of_constraints.png)

![Relative Efficiency](images/merkle_tree/IMT_SMT_relative_efficiency.png)


### 4.5 Overall Assessment

* From the benchmarks, membership proofs (LeanIMT) are faster to compute than non-membership proofs (SMT). 

   * Insert: LeanIMT is 8.5 times faster than SMT
   * Update: LeanIMT is 3.8 times faster than SMT
   * Generate Proof: LeanIMT is 326 times faster than SMT
   * Verify Proof: LeanIMT is 1.8 times faster than SMT

* When revocations are rare, it is often more efficient to use an SMT over revoked credentials. When revocations are frequent, and the number of valid credentials is equal to or smaller than the number of revoked credentials, it can be more efficient to use a LeanIMT over the valid credentials.

* At every tree depth, SMT has between ~1,560 and ~1,710 more constraints than LeanIMT. While the absolute difference grows slowly with depth, the relative ratio decreases: for small depths, SMT can have over 4x more constraints, but by depth 32 it is only about 1.22x more. This shows that LeanIMT provides a large relative improvement for small trees, while still maintaining an absolute advantage for larger trees.

* The optimal choice between LeanIMT and SMT depends on the expected revocation frequency and the relative sizes of valid versus revoked credential sets.