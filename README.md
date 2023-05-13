# Generative Art Platform

## Case Study

In this case study, we'll be creating a Generative Art platform on the Ethereum blockchain. This platform will allow artists to publish their projects, and users can mint unique iterations of these projects as NFTs. We'll be focusing on the implementation of core smart contracts and providing a high-level overview of the system architecture.

### Deliverables:

1. Smart Contracts in Solidity
2. Schema of the system architecture
3. Explanations of various system components
4. Rough projection of the implementation schedule

### System Architecture

The system architecture consists of the following components:

1. Frontend: A web application built using React and TypeScript, allowing users to browse, mint, and trade NFTs.
2. Backend: A Node.js server built using TypeScript and Express, handling API requests and interacting with the Ethereum blockchain.
3. Database: A PostgreSQL database for storing data created by the smart contracts and front-end app.
4. Rendering Pipeline: A module for generating images from the generative art code.
5. Decentralized Media Cluster: A module for storing and serving files, such as project details and NFT metadata.
6. Marketplace Smart Contract: A smart contract for trading NFTs.
7. Backend Wallet Manager: A module for securely managing on-chain operations in the backend.

### Smart Contracts

We will be implementing two core smart contracts for this application:

1. GenerativeArtProject: This contract will handle the creation and management of generative art projects. It will store project details, minting rules, and splits for funds distribution.
2. GenerativeArtNFT: This contract will handle the minting, revealing, and trading of NFTs associated with the generative art projects.

#### GenerativeArtProject.sol:

```solidity
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC721/IERC721.sol";
import "./GenerativeArtNFT.sol";

contract GenerativeArtProject {
    struct Project {
        string name;
        uint256 editions;
        uint256 price;
        uint256 openingTime;
        string codePointer;
        string detailsPointer;
        address[] splits;
        uint256[] percentages;
        uint256 royalties;
    }

    mapping(uint256 => Project) public projects;
    uint256 public projectCount;

    event ProjectCreated(uint256 indexed projectId, address indexed creator);

    function createProject(
        string memory _name,
        uint256 _editions,
        uint256 _price,
        uint256 _openingTime,
        string memory _codePointer,
        string memory _detailsPointer,
        address[] memory _splits,
        uint256[] memory _percentages,
        uint256 _royalties
    ) external {
        require(_editions > 0, "Invalid number of editions");
        require(_price > 0, "Invalid price");
        require(_splits.length == _percentages.length, "Splits and percentages length mismatch");

        uint256 projectId = projectCount++;
        projects[projectId] = Project({
            name: _name,
            editions: _editions,
            price: _price,
            openingTime: _openingTime,
            codePointer: _codePointer,
            detailsPointer: _detailsPointer,
            splits: _splits,
            percentages: _percentages,
            royalties: _royalties
        });

        emit ProjectCreated(projectId, msg.sender);
    }
}
```

#### GenerativeArtNFT.sol:

```solidity
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC721/extensions/ERC721Enumerable.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "./GenerativeArtProject.sol";

contract GenerativeArtNFT is ERC721Enumerable, Ownable {
    using Strings for uint256;

    GenerativeArtProject public artProject;
    uint256 public nftCount;
    mapping(uint256 => uint256) public tokenProject;
    mapping(uint256 => bytes32) public tokenSeed;
    mapping(uint256 => string) private _tokenURIs;

    event NFTMinted(uint256 indexed tokenId, address indexed minter);

    constructor(address _artProject) ERC721("GenerativeArtNFT", "GANFT") {
        artProject = GenerativeArtProject(_artProject);
    }

    function mint(uint256 _projectId) external payable {
        GenerativeArtProject.Project memory project = artProject.projects(_projectId);

        require(project.editions > 0, "Project sold out");
        require(msg.value >= project.price, "Insufficient payment");
        require(block.timestamp >= project.openingTime, "Project not open yet");

        uint256 tokenId = nftCount++;
        _safeMint(msg.sender, tokenId);

        tokenProject[tokenId] = _projectId;
        tokenSeed[tokenId] = keccak256(abi.encodePacked(block.timestamp, msg.sender, tokenId));

        project.editions--;

        emit NFTMinted(tokenId, msg.sender);
    }

    function setTokenURI(uint256 tokenId, string memory _tokenURI) external onlyOwner {
        require(_exists(tokenId), "Token does not exist");
        _tokenURIs[tokenId] = _tokenURI;
    }

    function tokenURI(uint256 tokenId) public view virtual override returns (string memory) {
        require(_exists(tokenId), "Token does not exist");

        string memory _tokenURI = _tokenURIs[tokenId];
        return bytes(_tokenURI).length > 0 ? _tokenURI : "";
    }

    function _beforeTokenTransfer(address from, address to, uint256 tokenId) internal virtual override(ERC721Enumerable) {
        super._beforeTokenTransfer(from, to, tokenId);
    }

    function supportsInterface(bytes4 interfaceId) public view virtual override(ERC721Enumerable) returns (bool) {
        return super.supportsInterface(interfaceId);
    }
}
```

### Listener/Indexer Service

The Listener/Indexer service is a separate component of the Generative Art platform that listens to events emitted by the smart contracts and updates the database with the relevant information. This service is crucial for providing market statistics and ensuring the platform's data is up-to-date.

The Listener/Indexer service can be implemented using a Node.js server that connects to the Ethereum network, listens to contract events, and updates the PostgreSQL database accordingly.

#### Key Features:

1. Subscribe to events emitted by the `GenerativeArtProject` and `GenerativeArtNFT` smart contracts.
2. Parse the event data and update the database tables with the relevant information.
3. Handle reorgs and chain reorganizations to ensure data consistency.
4. Provide an API for querying the collected data and generating market statistics.

### Database Tables for Listener/Indexer Service

To store data collected by the Listener/Indexer service, we will need the following additional database tables:

### Implementation Schedule

1. Week 1-2: Design and implement the GenerativeArtProject and GenerativeArtNFT smart contracts.
2. Week 3-4: Develop the backend server and API, including database schema and integration with smart contracts.
3. Week 5-6: Implement the frontend application for browsing, minting, and trading NFTs.
4. Week 7-8: Integrate the rendering pipeline and decentralized media cluster with the backend server.
5. Week 9-10: Test the entire system, including smart contracts, frontend, and backend components.
6. Week 11-12: Deploy the platform to a test network and perform user testing and bug fixing.

### Traffic

To handle a large volume of requests (2000 req/s) in a public API that exposes the internal database, we need to ensure that the architecture is designed to be scalable, efficient, and robust. Here are some strategies and best practices to achieve this:

1. Use a load balancer: Deploy a load balancer, such as AWS Elastic Load Balancer or NGINX, to distribute incoming traffic across multiple instances of the API server. This helps to ensure that no single server is overwhelmed with traffic and helps to maintain high availability.

2. Implement horizontal scaling: Design the API server to be stateless, so that it can be easily scaled horizontally by adding more instances as needed. This can be achieved using containerization technologies like Docker and orchestration tools like Kubernetes.

3. Use a caching mechanism: Implement caching at various levels (in-memory, server-side, or client-side) to reduce the load on the database and improve response times. Popular caching solutions include Redis, Memcached, or even a CDN for caching static assets.

4. Rate limiting: Implement rate limiting to prevent abuse of the API and to ensure fair usage among clients. Rate limiting can be applied at various levels, such as IP address, user account, or API key.

### APIs

To fulfill the requirements of the users and artists on the platform, we would need the following API endpoints:

1. User Authentication and Authorization

```bash
   - POST /auth/register: Register a new user account
   - POST /auth/login: Authenticate and log in a user
   - GET /auth/user: Get the currently authenticated user's details
```

2. Generative Art Projects

```bash
   - POST /projects: Create a new generative art project
   - GET /projects: List all generative art projects
   - GET /projects/:id: Get a specific generative art project by ID
   - PUT /projects/:id: Update a specific generative art project by ID
   - DELETE /projects/:id: Delete a specific generative art project by ID
```

3. NFT Minting

```bash
   - POST /nfts/mint: Mint a new NFT for a specific project
   - GET /nfts/:id: Get a specific NFT by ID
   - PUT /nfts/:id/reveal: Reveal a specific NFT by ID, updating its metadata
```

4. NFT Trading

```bash
   - POST /nfts/:id/sell: List an NFT for sale
   - POST /nfts/:id/cancel: Cancel an NFT sale listing
   - POST /nfts/:id/buy: Buy an NFT listed for sale
   - POST /nfts/:id/offer: Make an offer on an NFT
   - POST /nfts/:id/accept: Accept an offer on an NFT
   - POST /nfts/:id/reject: Reject an offer on an NFT
```

5. Market Statistics

```bash
   - GET /stats/overall: Get overall platform statistics
   - GET /stats/projects/:id: Get statistics for a specific generative art project
```

6. User Collections

```bash
   - GET /users/:id/collection: Get the NFT collection for a specific user
```

7. Decentralized Media Cluster

```bash
   - POST /media: Upload a file to the decentralized media cluster
   - GET /media/:id: Get a file from the decentralized media cluster by ID
```

8. Rendering Pipeline

```bash
   - POST /render: Generate an image from generative art code and unique seed
```

### Database Schema

To store data for the Generative Art platform, we will need the following database tables:

1. Users

```sql
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  username VARCHAR(255) UNIQUE NOT NULL,
  email VARCHAR(255) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  created_at TIMESTAMP NOT NULL,
  updated_at TIMESTAMP NOT NULL
);
```

2. Projects

```sql
CREATE TABLE projects (
  id SERIAL PRIMARY KEY,
  creator_id INTEGER NOT NULL REFERENCES users(id),
  name VARCHAR(255) NOT NULL,
  editions INTEGER NOT NULL,
  price NUMERIC(18, 6) NOT NULL,
  opening_time TIMESTAMP NOT NULL,
  code_pointer VARCHAR(255) NOT NULL,
  details_pointer VARCHAR(255) NOT NULL,
  royalties INTEGER NOT NULL,
  created_at TIMESTAMP NOT NULL,
  updated_at TIMESTAMP NOT NULL
);
```

3. Splits

```sql
CREATE TABLE splits (
  id SERIAL PRIMARY KEY,
  project_id INTEGER NOT NULL REFERENCES projects(id),
  beneficiary_address VARCHAR(255) NOT NULL,
  percentage INTEGER NOT NULL,
  created_at TIMESTAMP NOT NULL,
  updated_at TIMESTAMP NOT NULL
);
```

4. NFTs

```sql
CREATE TABLE nfts (
  id SERIAL PRIMARY KEY,
  project_id INTEGER NOT NULL REFERENCES projects(id),
  token_id INTEGER NOT NULL,
  owner_id INTEGER NOT NULL REFERENCES users(id),
  seed VARCHAR(255) NOT NULL,
  token_uri VARCHAR(255),
  created_at TIMESTAMP NOT NULL,
  updated_at TIMESTAMP NOT NULL
);
```

5. Sales

```sql
CREATE TABLE sales (
  id SERIAL PRIMARY KEY,
  nft_id INTEGER NOT NULL REFERENCES nfts(id),
  seller_id INTEGER NOT NULL REFERENCES users(id),
  price NUMERIC(18, 6) NOT NULL,
  status VARCHAR(255) NOT NULL,
  created_at TIMESTAMP NOT NULL,
  updated_at TIMESTAMP NOT NULL
);
```

6. Offers

```sql
CREATE TABLE offers (
  id SERIAL PRIMARY KEY,
  nft_id INTEGER NOT NULL REFERENCES nfts(id),
  buyer_id INTEGER NOT NULL REFERENCES users(id),
  price NUMERIC(18, 6) NOT NULL,
  status VARCHAR(255) NOT NULL,
  created_at TIMESTAMP NOT NULL,
  updated_at TIMESTAMP NOT NULL
);
```

7. Media

```sql
CREATE TABLE media (
  id SERIAL PRIMARY KEY,
  file_hash VARCHAR(255) NOT NULL,
  file_url VARCHAR(255) NOT NULL,
  created_at TIMESTAMP NOT NULL,
  updated_at TIMESTAMP NOT NULL
);
```

8. Rendered Images

```sql
CREATE TABLE rendered_images (
  id SERIAL PRIMARY KEY,
  nft_id INTEGER NOT NULL REFERENCES nfts(id),
  image_url VARCHAR(255) NOT NULL,
  created_at TIMESTAMP NOT NULL,
  updated_at TIMESTAMP NOT NULL
);
```

9. Contract Events

```sql
CREATE TABLE contract_events (
  id SERIAL PRIMARY KEY,
  event_name VARCHAR(255) NOT NULL,
  contract_address VARCHAR(255) NOT NULL,
  block_number INTEGER NOT NULL,
  transaction_hash VARCHAR(255) NOT NULL,
  event_data JSONB NOT NULL,
  created_at TIMESTAMP NOT NULL
);
```

10. Market Statistics

```sql
CREATE TABLE market_statistics (
  id SERIAL PRIMARY KEY,
  project_id INTEGER REFERENCES projects(id),
  daily_platform_sales_primary NUMERIC(18, 6),
  daily_platform_sales_secondary NUMERIC(18, 6),
  daily_users INTEGER,
  total_sales_primary NUMERIC(18, 6),
  total_sales_secondary NUMERIC(18, 6),
  floor_price NUMERIC(18, 6),
  median_price NUMERIC(18, 6),
  sales_last_24h_primary NUMERIC(18, 6),
  sales_last_24h_secondary NUMERIC(18, 6),
  created_at TIMESTAMP NOT NULL,
  updated_at TIMESTAMP NOT NULL
);
```

11. Market Statistics Charts

```sql
CREATE TABLE market_statistics_charts (
  id SERIAL PRIMARY KEY,
  project_id INTEGER REFERENCES projects(id),
  chart_type VARCHAR(255) NOT NULL,
  data JSONB NOT NULL,
  created_at TIMESTAMP NOT NULL,
  updated_at TIMESTAMP NOT NULL
);
```

```mermaid
graph TD
  subgraph Frontend
    A[React Web App]
  end
  subgraph Backend
    B[Node.js Server]
    C[Database Postgresql]
    D[Backend Wallet Manager]
  end
  subgraph Ethereum
    E[GenerativeArtProject Smart Contract]
    F[GenerativeArtNFT Smart Contract]
    G[Marketplace Smart Contract]
  end
  subgraph External Services
    H[Rendering Pipeline]
    I[Decentralized Media Cluster]
    J[Content Delivery Network CDN]
  end
  subgraph Load Balancer
    K[Load Balancer e.g. NGINX, AWS ELB]
  end
  subgraph Caching
    L[Cache e.g. Redis, Memcached]
  end
  A --> K
  K --> B
  B --> C
  B --> D
  B --> E
  B --> F
  B --> G
  B --> H
  B --> I
  B --> L
  E --> F
  F --> G
  H --> I
  I --> J
  L --> C
```

### Conclusion

In this case study, we have outlined the requirements, system architecture, and implementation schedule for a Generative Art platform on Ethereum. By implementing core smart contracts and designing a scalable system architecture, we can create a platform that allows artists to showcase their work and users to mint and trade unique NFTs. The following requirements are fulfilled:

1. Artists can publish their Generative Art project on the platform:

   - The `GenerativeArtProject` smart contract allows artists to create and manage projects with the required options.
   - The API endpoints for projects enable artists to interact with the platform and upload their code and project details to the decentralized media network.

2. Users can browse the different projects on a website:

   - The frontend web application built using React and TypeScript allows users to browse and interact with the projects.

3. Users can mint unique iterations of the projects:

   - The `GenerativeArtNFT` smart contract handles the minting of NFTs with the required conditions and generates a unique seed for each iteration.
   - The frontend application allows users to mint NFTs by interacting with the smart contract.

4. NFTs should be revealed with an off-chain system:

   - The rendering pipeline and decentralized media cluster handle the generation of images and metadata for the NFTs, which are then updated in the smart contract.

5. Users can put their NFTs for sale on the platform and accept sales:

   - The API endpoints for NFT trading enable users to list their NFTs for sale, accept offers, and interact with the `GenerativeArtNFT` smart contract.

6. Users can make offers on other NFTs or project-wide offers and accept such offers:

   - The API endpoints for offers allow users to make and manage offers on NFTs and interact with the smart contract.

7. Users have access to market statistics:
   - The API endpoints for market statistics provide the required data for overall and project-specific statistics, which can be displayed on the frontend application.

### Improvement Proposals

1. Implement different pricing strategies for minting NFTs, such as Step Dutch Auction and Linear Dutch Auction.
2. Provide a feature for users to stake their NFTs and earn rewards based on the popularity of the project.
3. Implement a governance system for the platform, allowing users to vote on upgrades and new features.
