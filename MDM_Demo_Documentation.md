# MDM Demo Documentation

## Overview
This document outlines the Master Data Management (MDM) demonstration architecture, integration patterns, data domains, and data models used across retail and manufacturing use cases.

---

## 1. MDM Integration Patterns

### 1.1 API-Based Integration (Real-Time)
**Pattern**: RESTful API calls to Informatica MDM Business360  
**Use Cases**: 
- Real-time entity lookups
- Create/update operations during business processes
- Relationship and hierarchy management
- Search and match operations

**Implementation**:
- Session-based authentication (30-minute token lifecycle)
- REST endpoints for CRUD operations
- JSON payload structure
- Rate limiting and retry logic

**Advantages**:
- Immediate data availability
- Transactional consistency
- Real-time validation and matching

**Trade-offs**:
- Network latency considerations
- API rate limits
- Requires stable connectivity

### 1.2 Batch File Integration
**Pattern**: Bulk data loads via file-based ETL  
**Use Cases**:
- Initial data migration
- Periodic bulk updates
- Large-scale data synchronization
- Scheduled master data refreshes

**Implementation**:
- CSV/JSON file generation
- BigQuery staging tables
- Batch processing workflows
- Validation and error handling

**Advantages**:
- High throughput for large volumes
- Lower API consumption
- Easier troubleshooting via file inspection

**Trade-offs**:
- Latency between updates
- File management overhead
- Potential for stale data between loads

### 1.3 Event-Driven Integration
**Pattern**: Publish/subscribe messaging for data changes  
**Use Cases**:
- Real-time downstream system synchronization
- Change data capture (CDC)
- Audit trail and compliance tracking

**Implementation**:
- Event streams (Pub/Sub, Kafka)
- Event schemas for create/update/delete operations
- Consumer services for data propagation

### 1.4 Hybrid Integration
**Pattern**: Combination of real-time API and batch processing  
**Use Cases**:
- High-priority transactions via API
- Bulk updates via batch
- Reference data via periodic sync

**Implementation**:
- Tiered data criticality (real-time vs. batch)
- Orchestration layer for pattern routing
- Unified monitoring and error handling

---

## 2. The Data Domains

### 2.1 Product Domain
**Purpose**: Master data for products, items, SKUs, and product hierarchies  
**Key Entities**:
- **Product**: Logical product definition (e.g., "Pikachu Graphic Tee")
- **Item**: Physical SKU variants (size, color) beneath a product
- **Category**: Product classification hierarchy (Clothing > Tops > T-Shirts)
- **Material**: Components or ingredients (manufacturing)

**Attributes**:
- Product Name, Description
- Item SKU, Size, Color, Price
- Category Path (multi-level hierarchy)
- Stock Status, UPC/EAN codes

**Relationships**:
- Product → Item (1:many)
- Category → Product (1:many)
- Material → Product (many:many for bill of materials)

**Use Cases**:
- E-commerce product catalogs
- Inventory management
- Pricing and promotion management
- Supply chain planning

### 2.2 Location Domain
**Purpose**: Master data for physical and logical locations  
**Key Entities**:
- **Location**: Physical store, warehouse, distribution center
- **Organization**: Operating entities (brands, franchise groups, regions)

**Attributes**:
- Location Name, Address, Geo-coordinates
- Store Type, Format (franchise, corporate)
- Operational Status, Hours
- Organization hierarchy levels

**Relationships**:
- Location → Organization (many:1)
- Location hierarchies (Region > Market > Store)
- Organization hierarchies (Brand > Franchise Group > Store)

**Use Cases**:
- Store locators and mapping
- Franchise territory management
- Supply chain routing and fulfillment
- Location-based reporting and analytics

**Example**: Dine Brands 5-level hierarchy:
```
Domestic/International > Brand (Applebee's/IHOP) > Franchise Group > Price Tier > Store
```

### 2.3 Customer Domain
**Purpose**: Master data for customers, accounts, and contacts  
**Key Entities**:
- **Customer**: Individual or business customer
- **Account**: Customer account hierarchy
- **Contact**: Individual contact within an account

**Attributes**:
- Customer ID, Name, Type (B2B, B2C)
- Contact information (email, phone)
- Segmentation and classification
- Account hierarchy and relationships

**Relationships**:
- Customer → Account (1:many)
- Account → Contact (1:many)
- Account hierarchies (parent-child)

**Use Cases**:
- Customer 360-degree view
- Sales territory management
- Marketing segmentation
- Billing and invoicing consolidation

### 2.4 Vendor/Supplier Domain
**Purpose**: Master data for suppliers, manufacturers, and partners  
**Key Entities**:
- **Vendor**: Supplier or manufacturer
- **Vendor Location**: Physical vendor sites
- **Vendor Relationship**: Contracts and agreements

**Attributes**:
- Vendor ID, Name, Type
- Payment terms, tax information
- Quality ratings, certifications
- Preferred status, risk scores

**Relationships**:
- Vendor → Material/Product (sources)
- Vendor → Vendor Location (1:many)

**Use Cases**:
- Procurement and sourcing
- Supplier performance management
- Risk and compliance tracking

---

## 3. The Data Model: Product Paradigm

### 3.1 Three-Tier Product Model
The product paradigm uses a **Category → Product → Item** hierarchy to support flexible merchandise management.

```
┌─────────────────────────────────────────────────────┐
│                    Category                         │
│  (Classification Hierarchy)                         │
│                                                     │
│  Level 1 > Level 2 > Level 3                       │
│  Clothing > Tops > Hoodies                         │
└────────────────┬────────────────────────────────────┘
                 │
                 │ (1:many)
                 │
                 ▼
┌─────────────────────────────────────────────────────┐
│                    Product                          │
│  (Logical Product Definition)                       │
│                                                     │
│  - Product Name                                     │
│  - Product Description                              │
│  - Base Attributes (brand, style)                  │
└────────────────┬────────────────────────────────────┘
                 │
                 │ (1:many)
                 │
                 ▼
┌─────────────────────────────────────────────────────┐
│                     Item                            │
│  (Physical SKU / Sellable Unit)                     │
│                                                     │
│  - Item SKU (unique identifier)                    │
│  - Size, Color, Configuration                      │
│  - Price, UPC/EAN                                   │
│  - Stock Status                                     │
└─────────────────────────────────────────────────────┘
```

### 3.2 Category Entity

**Purpose**: Multi-level classification structure for organizing products

**Schema**:
```
Category
├── category_id (PK)
├── category_name
├── category_level_1  (e.g., "Clothing")
├── category_level_2  (e.g., "Tops")
├── category_level_3  (e.g., "Hoodies")
├── parent_category_id (FK, self-referencing)
├── category_path (full path concatenation)
└── category_attributes (flexible metadata)
```

**Hierarchy Characteristics**:
- **Variable depth**: Support 3-7 levels depending on business needs
- **Self-referencing**: Parent-child relationships enable tree structure
- **Path materialization**: Pre-computed full paths for query performance

**Examples**:
- `Clothing > Tops > Hoodies`
- `Electronics > Audio > Earbuds`
- `Toys > Plush > Large Plush`
- `Collectibles > Trading Cards > Booster Packs`

### 3.3 Product Entity

**Purpose**: Logical product definition representing a style or model

**Schema**:
```
Product
├── product_id (PK)
├── product_name
├── product_description
├── category_id (FK → Category)
├── brand
├── product_family
├── base_price
├── created_date
├── modified_date
└── product_attributes (extensible JSON/key-value)
```

**Characteristics**:
- **Style-level definition**: Represents the "concept" of a product
- **Category assignment**: Links to classification hierarchy
- **Variant-agnostic**: Does not contain size/color specifics
- **Marketing-focused**: Description and imagery at this level

**Examples**:
- Product: "Pokemon x Dolly Noire: Pixel Charizard Hoodie - Adult"
  - Category: Clothing > Tops > Hoodies
  - Brand: Pokemon x Dolly Noire
  - Base Description: "Premium black hoodie featuring pixel art Charizard design..."

- Product: "Pikachu Classic Graphic Tee - Adult"
  - Category: Clothing > Tops > T-Shirts
  - Brand: Pokemon
  - Base Description: "Soft cotton tee with classic Pikachu graphic..."

### 3.4 Item Entity

**Purpose**: Physical, orderable, sellable SKU with specific attributes

**Schema**:
```
Item
├── item_id (PK)
├── item_sku (unique, scannable identifier)
├── product_id (FK → Product)
├── item_name (product + variant description)
├── size
├── color
├── configuration (other variant dimensions)
├── price
├── cost
├── upc_code / ean_code
├── weight
├── dimensions
├── stock_status
└── item_attributes (extensible)
```

**Characteristics**:
- **Fully specified**: All variant dimensions resolved (size, color, etc.)
- **Orderable**: Can be added to cart, invoiced, shipped
- **Inventory-tracked**: Each item has stock levels
- **Price-carrying**: Each SKU has specific pricing

**Examples**:
- Item: "Pokemon Pixel Charizard Hoodie - XS"
  - SKU: PKM-CH-HDI-001-XS
  - Product: "Pokemon x Dolly Noire: Pixel Charizard Hoodie"
  - Size: XS
  - Color: Black
  - Price: $59.99

- Item: "Pikachu Classic Tee - M"
  - SKU: PKM-PK-TEE-002-M
  - Product: "Pikachu Classic Graphic Tee"
  - Size: M
  - Color: Yellow
  - Price: $24.99

### 3.5 Relationship Tables

#### Category_Product Relationship
```
Category_Product
├── category_id (FK → Category)
├── product_id (FK → Product)
├── is_primary (boolean, one primary per product)
├── display_order
└── effective_date / end_date
```

**Purpose**: 
- Supports multi-category assignment (cross-merchandising)
- Primary vs. secondary categorization
- Time-bound category assignments (seasonal shifts)

#### Product_Item Relationship
```
Product_Item (implicit via product_id FK in Item table)
```

**Purpose**:
- One product to many items
- All items under a product share base attributes
- Variant explosion managed at item level

### 3.6 Data Model Benefits

#### For E-Commerce
- **Product Pages**: Display product-level info with item dropdown for size/color
- **Search & Browse**: Navigate by category, filter at product level, select item to purchase
- **Inventory**: Track stock at item (SKU) level while reporting at product level

#### For Merchandising
- **Assortment Planning**: Plan at product level, fulfill with items
- **Pricing Strategy**: Base price at product, promotional pricing at item
- **Catalog Management**: Organize by category, merchandise by product

#### For Supply Chain
- **Procurement**: Order by item SKU
- **Fulfillment**: Pick/pack/ship specific items
- **Forecasting**: Aggregate demand at product level, allocate to items

### 3.7 Example: Pokemon Product Data

**Category Hierarchy**:
```
Clothing
├── Tops
│   ├── Hoodies
│   ├── T-Shirts
│   └── Sweatshirts
├── Bottoms
│   └── Joggers
```

**Product Example**:
- **Product**: Pokemon x Dolly Noire: Pixel Charizard Hoodie - Adult
- **Category Path**: Clothing > Tops > Hoodies
- **Description**: Premium black hoodie featuring pixel art Charizard design. 80% cotton, 20% polyester. Kangaroo pocket, drawstring hood.

**Item Variants** (6 SKUs):
1. PKM-CH-HDI-001-XS (Size: XS)
2. PKM-CH-HDI-001-S (Size: S)
3. PKM-CH-HDI-001-M (Size: M)
4. PKM-CH-HDI-001-L (Size: L)
5. PKM-CH-HDI-001-XL (Size: XL)
6. PKM-CH-HDI-001-2XL (Size: 2XL)

**Data Lineage**:
```
Category: Clothing > Tops > Hoodies
    ↓
Product: Pixel Charizard Hoodie (1 product)
    ↓
Items: 6 SKUs (XS, S, M, L, XL, 2XL)
```

---

## 4. MDM Implementation Architecture

### 4.1 BigQuery as MDM Repository
- **Project**: ehc-alexandra-sommers-4108e6
- **Datasets**:
  - `mdm_Manufacture`: Product, Category, Material, Customer, Vendor
  - `mdm_Location`: Location, Organization hierarchies

### 4.2 Informatica MDM Business360 Integration
- **API Endpoint**: REST-based entity management
- **Authentication**: Session tokens (30-minute lifecycle)
- **Operations**: Search, Create, Update, Relationship management
- **Hierarchy Management**: Support for complex multi-level hierarchies

### 4.3 Data Load Patterns
1. **Synthetic Data Generation**: Generate realistic test data from natural language prompts
2. **Schema Mapping**: Map generated data to MDM entity schemas
3. **Relationship Building**: Create foreign key relationships and hierarchy tables
4. **BigQuery Load**: Bulk insert into staging and master tables
5. **API Sync** (optional): Push to Informatica MDM via REST API

---

## 5. Demo Scenarios

### 5.1 Retail Product Catalog
**Domain**: Product, Category  
**Use Case**: E-commerce product hierarchy management  
**Data**: Pokemon merchandise (40 products, 8 categories)  
**Demo Flow**:
1. Category hierarchy creation (3-level)
2. Product loading with category assignment
3. Item/SKU variant creation
4. Query by category, product lookup, SKU search

### 5.2 Restaurant Location Hierarchy
**Domain**: Location, Organization  
**Use Case**: Franchise territory and store management  
**Data**: Dine Brands (24 locations, 5-level hierarchy)  
**Demo Flow**:
1. Organization hierarchy (Domestic/International > Brand > Franchise Group > Price > Store)
2. Location creation with geo-coordinates
3. Hierarchy traversal queries
4. Territory-based reporting

### 5.3 Manufacturing Supply Chain
**Domain**: Product, Material, Vendor  
**Use Case**: Bill of materials and supplier management  
**Demo Flow**:
1. Material master creation
2. Product BOM relationships
3. Vendor-material sourcing links
4. Supply chain traceability queries

---

## 6. Next Steps

### Documentation Enhancements
- [ ] Add ER diagrams for each domain
- [ ] Include sample SQL queries for common patterns
- [ ] Document API endpoint specifications
- [ ] Create data dictionary with attribute definitions

### Demo Expansions
- [ ] Customer 360 with account hierarchies
- [ ] Cross-domain relationships (Product → Vendor → Location)
- [ ] Data quality and matching rules
- [ ] Golden record creation and survivorship

### Technical Integrations
- [ ] Event-driven CDC patterns
- [ ] Real-time API integration examples
- [ ] Data governance and lineage tracking
- [ ] Performance tuning for large-scale hierarchies

---

## Appendix A: Glossary

**MDM (Master Data Management)**: Discipline of creating and maintaining authoritative, reliable master data across an enterprise

**Golden Record**: Single, trusted version of a master data entity created through matching, merging, and survivorship rules

**Hierarchy**: Parent-child relationship structure (e.g., category trees, organizational charts, location rollups)

**SKU (Stock Keeping Unit)**: Unique identifier for a specific item variant (size, color, configuration)

**UPC/EAN**: Universal Product Code / European Article Number - barcode standards for retail products

**BOM (Bill of Materials)**: List of raw materials, components, and assemblies required to manufacture a product

**CDC (Change Data Capture)**: Pattern for identifying and capturing changes to source data for downstream propagation

---

## Appendix B: References

- Informatica MDM Business360 API Documentation
- BigQuery best practices for hierarchical data
- Pokemon product data: `/Users/alexandra.sommers/pokemon_products.csv`
- Dine Brands location hierarchy: Memory reference `project_dine_brands.md`
