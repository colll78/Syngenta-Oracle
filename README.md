# Design Document and Project Plan for Cardano-Based Agricultural Oracle

## 1. Overview

This document outlines the design and project plan for a proof-of-concept (PoC) oracle system built on the Cardano blockchain to empower smallholder farmers in India. The system integrates earth observation, farm, and market data to facilitate trusted data exchange among farmers, Agri-Entrepreneurs (AEs), buyers, and government agencies. The focus is on the data structure, blockchain interaction, component architecture, technical specifications for the oracle contract, integration strategy, unit testing plan, and scalability measurement criteria.

---

## 2. Data Structure and Blockchain Interaction

### 2.1 Oracle Datum Structure

The oracle datum is a critical component stored on-chain to represent farm-related data. It is designed to be lightweight, extensible, and interoperable with Cardano's Plutus smart contracts.

**Datum Fields:**

1. **Farm Land Area** (`Integer`):
   - Represents the area of the farm in square yards.
   - Stored as a 64-bit integer to accommodate large farm sizes while maintaining efficiency.
   - Example: `5000` (for 5000 square yards).
2. **IPFS Hash for Farm Borders** (`ByteString`):
   - A 46-byte IPFS hash linking to a JSON file containing geospatial coordinates (latitude/longitude points) defining the farm's boundaries.
   - Stored as a Cardano `ByteString` for compatibility with Plutus.
   - Example: `QmXyZ123...` (IPFS CIDv0 hash).
3. **Arbitrary Data** (`BuiltinData`):
   - A flexible field for additional metadata (e.g., soil health metrics, crop type, or yield predictions).
   - Uses Cardano's `BuiltinData` type to support arbitrary Plutus-compatible data structures.
   - Example: `{ "cropType": "rice", "soilPH": 6.5 }`.

**Plutus Datum Schema:**

```haskell
data OracleDatum = OracleDatum
  { farmArea :: Integer
  , ipfsHash :: BuiltinByteString
  , arbitraryData :: BuiltinData
  }
```

The oracle datum is a critical component stored on-chain to represent farm-related data. It is designed to be lightweight, extensible, and interoperable with Cardano's Plutus smart contracts.

**Datum Fields:**

1. **Farm Land Area** (`Integer`):
   - Represents the area of the farm in square yards, with one square yard encoded as 1,000,000 units to ensure precision, and to facilitate standard smart contract interfaces for decimal precision operations.
   - Example: A farm of 500 square yards is represented as 500,000,000 (500 * 1,000,000).
2. **IPFS Hash for Farm Borders** (`ByteString`):
   - A 46-byte IPFS hash linking to a JSON file containing geospatial coordinates (latitude/longitude points) defining the farm's boundaries.
   - Stored as a Cardano `ByteString` for compatibility with Plutus.
   - Example: `QmXyZ123...` (IPFS CIDv0 hash).
3. **Arbitrary Data** (`BuiltinData`):
   - A flexible field for additional arbitrary metadata (e.g., soil health metrics, crop type, or yield predictions).
   - Uses Cardano's `BuiltinData` type to support arbitrary Plutus-compatible data structures.
   - Example: `{ "cropType": "rice", "soilPH": 6.5 }` would be encoded as the `BuiltinData` representation of the type:

```haskell
-- | Example arbitrary data that can be included in the oracle datum.
data CropInfo = 
   { cropType :: BuiltinByteString
   , soilPH :: Rational
   }
```

### 2.2 Blockchain Interaction

To ensure maximum usability, the system supports two oracle architectures on Cardano: **Reference UTxO Oracle Architecture** and **Signed Message Oracle Architecture**. Each has distinct trade-offs, balancing data availability, permissionless access, and operational efficiency.

**Reference UTxO Oracle Architecture**

In the reference UTxO oracle architecture, the oracle data is submitted to the chain directly as UTxOs that can be consumed as reference inputs by transactions that seek to interact with the oracle data. For our use-case, we model oracles as [CIP-68 NFTs](https://github.com/cardano-foundation/CIPs/blob/master/CIP-0068/README.md) where the CIP-68 user-token is the farm parcel NFT, and the corresponding CIP-68 reference-token is the UTxO with the oracle data provided for the associated farm parcel.  

There are a few key parameters for the system:
```haskell
-- | The public key hash of the party authorized to issue farm parcel DID NFTs.
issuanceOperator :: PubKeyHash 
issuanceOperator = ...

-- | A list of public key hashes of parties authorized to update the oracle data associated with the farm parcels. 
farmParcelOracleProviders :: [PubKeyHash]
farmParcelOracleProviders = [...]
```

**Oracle Management Spending Validator:** 

- **On-Chain**:
  - The `oracleManagementValidator` is a Plutus spending validator that manages oracle data updates and access control.
    - The lifecycle of all the CIP-68 reference-tokens are managed by this script, each oracle UTxO lives at the `oracleManagementScript` and contains a CIP-68 reference-token corresponding to the relevant farm parcel.
  - The validation logic of this script ensures that only relevant parties (`farmParcelOracleProviders`) are able to spend the Oracle UTxOs at this script, and enforces that they must produce a continuing output which preserves the reference-tokens and produces a valid oracle datum. 
- **Off-Chain**:
  - The oracle backend server fetches satellite data (via Gamma Earth APIs) and farmer inputs (via AE dashboards).
  - At regular intervals, the backend server submits transactions to update the oracle datum with newly fetched data.

**DID NFT Minting Policy** 

The DID NFT minting policy is a Plutus minting policy which manages the issuance of DIDs for individual farm parcels. 

- **On-Chain**:
  - The `farmParcelNFTMintingPolicy` validation logic enforces:
    - Exactly one user-token and reference-token pair is minted per farm parcel, ensuring DID uniqueness.
    - The reference-token is included in an output to the oracle management validator's script address. 
      - The initial oracle datum stored in this output is structurally valid.
    - Issuance of the DID is only possible with the authorization of the `issuanceOperator` (their signature must be present in any minting transaction).   
- **Off-Chain**:
  - Upon successful registration / onboarding of a new farm parcel, the backend will submit a transaction (signed by the `issuanceOperator` public key) that mints the user-token & reference-token pair corresponding to the farm parcel.
    - The transaction includes an output UTxO with the user-token sent to the farm parcel owner's wallet.
    - Another output UTxO with the reference-token and a valid oracle datum is sent to the oracle management validator's script address.
- **DID NFTs**:
  - Each CIP-68 user-token acts as a DID for an individual farm parcel, granting smallholders both ownership and control over their data.
  - The DIDs assign verifiable identities to farm parcels.

This architecture guarantees the data-availability of oracle data and ensures that the data can be consumed by dApps in a permissionless manner. 

**Signed Message Oracle Architecture**

The Signed Message Oracle Architecture is an alternative approach where data is not directly stored on-chain but is instead signed by a trusted oracle operator and passed into transactions that need to verify the data's authenticity. This pattern optimizes for low-cost, high-frequency data updates by minimizing on-chain footprint.

**Key Components:**

- **Off-Chain**:
  - The oracle operator backend periodically collects farm data, uses that data to construct the oracle datum message, and signs the message with their private key. 
  - Each signed message includes:
    - The oracle data (same schema as the on-chain `OracleDatum`) encoded as cborHex.
      - Concretely the oracle data in this message is the result of `serialiseData oracleDatum` where `oracleDatum` is some value of the type `OracleDatum`. 
    - Two timestamps to prevent replay attacks:
      - `validityStart` - the start of the validity range of this oracle data. 
      - `validityEnd` - the end of the validity range of this oracle data. 
      
Smart contract protocols can consume these messages as follows:
- A Plutus smart contract (e.g., for dApps consuming the oracle data) is provided with:
  - The oracle data and validity range timestamps. 
  - The signature of the oracle data message. 
- The validator verifies that:
  - The signature is valid over the provided data.
  - The signing key matches the expected oracle operator's public key.
  - The data timestamp or nonce is within an acceptable range (to prevent stale inputs).

**Advantages:**

- **Lower Oracle Operational Costs**: Since no state is stored or mutated on-chain, the oracle provider does not incur the costs of transaction fees required to publish the data to the chain periodically. 
- **High Frequency**: Oracle data can be can be updated frequently without the constraints of onchain throughput / bandwidth.
- **Dynamic Input**: Multiple dApps can consume the same oracle data simultaneously without regard for the potential of UTxO contention. In the reference input architecture, if an oracle datum is updated in the same block where a dApp is attempting to consume the datum, there is a potential that the dApp's transaction will fail as the oracle UTxO may already have been spent by the update transaction. 

**Trade-offs:**

- **No Native Data Availability**: Oracle data must be provided by the transaction creator; there is no guaranteed on-chain storage.
- **Higher Off-Chain Responsibility**: Applications must fetch and store the oracle data themselves.

This architecture complements the Reference UTxO Oracle architecture by serving use cases that demand real-time updates or when cost sensitivity outweighs on-chain persistence.

---

## 3. Unit Testing Plan

To ensure the correctness, reliability, and upgradability of the oracle system, a comprehensive unit testing strategy is defined across all smart contract components, datum structures, and integration workflows. The testing plan is divided into three primary layers: Datum validation, Script validation, and Integration tests.

### 3.1 Datum Integrity Tests

Unit tests will focus on serialization, schema integrity, and forward compatibility of the `OracleDatum` and any embedded custom data types (e.g., `CropInfo`).

**Test Cases:**

- **Signed Message Structural Integrity:**
  - Validate oracle datum message: `serialiseData oracleDatum <> integerToByteString validityStart <> integerToByteString validityEnd ≡ oracleDatumMessage` for valid `oracleDatum :: OracleDatum`.

- **Precision Integrity:**
  - Verify that farm areas represented with decimal-like scaling (e.g., `500 * 1_000_000`) maintain expected interpretation on-chain over common arithmetic operations.

- **Arbitrary Data Robustness:**
  - Fuzz test `arbitraryData` field with:
    - Valid known schema (e.g., correct `CropInfo`)
    - Unexpected but Plutus-compatible structures
    - Edge cases (empty maps, deeply nested structures, large byte strings)

- **Boundary Validation:**
  - Maximum field lengths for `ipfsHash`
  - Minimum and maximum land area edge cases (0, 1, 2^63-1)

### 3.2 Script Unit Tests

All Plutus validator and minting policy scripts will undergo isolated scenario-based testing.

**Oracle Management Validator:**

- **Positive Tests:**
  - Authorized oracle provider updates a datum with valid structure and preserves reference-token.
  - Continuity of UTxO validated (same asset class, correct script address).

- **Negative Tests:**
  - Unauthorized key attempts to spend an oracle UTxO.
  - Output datum is malformed or missing required fields.
  - Reference-token is removed, burned, or incorrectly transferred.

**DID NFT Minting Policy:**

- **Positive Tests:**
  - Correct issuance by `issuanceOperator` with matching user and reference-token minted.
  - Oracle UTxO correctly initialized with reference-token and valid datum.

- **Negative Tests:**
  - Attempt to mint without operator signature.
  - Minting multiple user-tokens or reference-tokens.
  - Oracle output not created, contains structurally invalid oracle datum, or does not contain the minted reference-token.

**Signed Message Validation (for consuming contracts):**

- **Positive Tests:**
  - Valid signature over CBOR-encoded oracle data with correct public key and valid timestamp window.

- **Negative Tests:**
  - Invalid signature.
  - Timestamps outside acceptable validity window.
  - Message tampering (mismatched data and signature).
