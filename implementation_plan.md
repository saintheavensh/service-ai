# Phone Service Dashboard - Frontend UI/UX Plan

Dashboard modern untuk manajemen service ponsel dan inventaris dengan sistem FIFO batch.

---

## 🗄️ Database Schema (ERD)

```mermaid
erDiagram
    CATEGORIES ||--o{ PRODUCTS : "memiliki"
    SUPPLIERS ||--o{ GRADES : "menyediakan"
    PRODUCTS ||--o{ INVENTORY_BATCHES : "stok masuk"
    GRADES ||--o{ INVENTORY_BATCHES : "tipe kualitas"
    INVENTORY_BATCHES ||--o{ SALES_DETAILS : "terjual"
    SALES ||--o{ SALES_DETAILS : "memiliki detail"
    CUSTOMERS ||--o{ SALES : "melakukan"
    CUSTOMERS ||--o{ SERVICE_ORDERS : "memiliki"
    SERVICE_ORDERS ||--o{ SERVICE_PARTS : "menggunakan"
    INVENTORY_BATCHES ||--o{ SERVICE_PARTS : "dipakai service"
    TECHNICIANS ||--o{ SERVICE_ORDERS : "mengerjakan"

    CATEGORIES {
        int id PK
        string nama_kategori
    }
    SUPPLIERS {
        int id PK
        string nama_supplier
    }
    GRADES {
        int id PK
        string nama_grade
        int supplier_id FK
    }
    PRODUCTS {
        string universal_kode PK
        string nama_produk
        int category_id FK
    }
    INVENTORY_BATCHES {
        int id PK
        string sku
        string universal_kode FK
        int grade_id FK
        decimal harga_beli
        decimal harga_jual_default
        int stok_awal
        int stok_sekarang
        datetime tgl_masuk
    }
    SALES {
        int id PK
        int customer_id FK
        datetime tgl_transaksi
        decimal total_bayar
        string metode_bayar
    }
    SALES_DETAILS {
        int id PK
        int sales_id FK
        int batch_id FK
        int qty
        decimal harga_jual_aktual
    }
    CUSTOMERS {
        int id PK
        string nama
        string telepon
        string email
    }
    SERVICE_ORDERS {
        int id PK
        string ticket_no
        int customer_id FK
        int technician_id FK
        string device_model
        string imei
        string keluhan
        string status
        decimal biaya_jasa
        datetime created_at
    }
    SERVICE_PARTS {
        int id PK
        int service_order_id FK
        int batch_id FK
        int qty
        decimal harga_jual_aktual
    }
    TECHNICIANS {
        int id PK
        string nama
        string telepon
    }
```

### Penjelasan Tabel

| Tabel | Fungsi |
|-------|--------|
| `categories` | Kategori produk (Baterai, LCD, dll) |
| `suppliers` | Daftar supplier (Braderparts, Zevan, dll) |
| `grades` | Tingkatan kualitas per supplier (Original, Premium, Silver) |
| `products` | Identitas universal produk (1 kode bisa punya banyak batch) |
| `inventory_batches` | **Jantung sistem** - setiap barang masuk = 1 batch baru, FIFO berdasarkan `tgl_masuk` |
| `sales` + `sales_details` | Transaksi penjualan dengan `harga_jual_aktual` per batch |
| `service_orders` + `service_parts` | Order service dan part yang digunakan |

---

## 🔄 Application Flow

```mermaid
flowchart TB
    subgraph INIT["📦 INITIAL SETUP"]
        A1[Admin Login] --> A2[Input Inventory Data]
        A2 --> A3[Setup Categories]
        A3 --> A4[System Ready]
    end

    subgraph SERVICE["🔧 SERVICE FLOW"]
        S1[Customer Arrives] --> S2[Create Service Ticket]
        S2 --> S3[Input Customer Data]
        S3 --> S4[Input Device Info<br/>Model, IMEI, Photo]
        S4 --> S5[Record Damage/Complaint]
        S5 --> S6[Print Digital Receipt]
        S6 --> S7{Assign Technician}
        
        S7 --> T1[Status: Checking]
        T1 --> T2{Need Parts?}
        T2 -->|Yes| T3[Status: Waiting for Parts]
        T3 --> T4[Select Parts from Inventory]
        T4 --> T5[Stock Auto Decreases]
        T5 --> T6[Status: Repairing]
        T2 -->|No| T6
        T6 --> T7[Status: Ready to Pick Up]
        T7 --> T8[Customer Notified]
        T8 --> T9[Customer Pays]
        T9 --> T10[Print Invoice]
        T10 --> T11[Status: Completed]
    end

    subgraph RETAIL["💰 RETAIL/POS FLOW"]
        R1[Customer Wants to Buy] --> R2[Open POS Module]
        R2 --> R3[Scan/Search Products]
        R3 --> R4[Add to Cart]
        R4 --> R5[Apply Discount if any]
        R5 --> R6[Process Payment]
        R6 --> R7[Stock Auto Decreases]
        R7 --> R8[Print Receipt]
        R8 --> R9[Transaction Complete]
    end

    subgraph INVENTORY["📦 INVENTORY FLOW"]
        I1[Receive New Stock] --> I2[Add to Inventory]
        I2 --> I3[Set Category<br/>Sparepart/Retail]
        I3 --> I4[Set Min Stock Alert]
        I4 --> I5[Stock Available]
        I5 --> I6{Stock Used}
        I6 -->|Service| T4
        I6 -->|Retail| R4
        I5 --> I7{Low Stock?}
        I7 -->|Yes| I8[Dashboard Alert]
        I8 --> I9[Reorder Stock]
        I9 --> I1
    end

    A4 --> SERVICE
    A4 --> RETAIL
    A4 --> INVENTORY
```

---

## 🧭 Navigation Structure

```mermaid
flowchart LR
    subgraph SIDEBAR["Sidebar Menu"]
        direction TB
        M1["📊 Dashboard"]
        M2["📦 Inventaris"]
        M3["🔧 Service Orders"]
        M4["👥 Pelanggan"]
        M5["💰 POS / Kasir"]
        M6["📈 Laporan"]
        M7["⚙️ Pengaturan"]
    end

    M1 --> D1["Overview Stats"]
    M1 --> D2["Revenue Chart"]
    M1 --> D3["Low Stock Alerts"]

    M2 --> I1["All Items"]
    M2 --> I2["Spareparts"]
    M2 --> I3["Retail Products"]
    M2 --> I4["Stock History"]

    M3 --> S1["All Services"]
    M3 --> S2["By Status Filter"]

    M4 --> C1["Customer List"]
    M4 --> C2["Service History"]

    M5 --> P1["New Transaction"]
    M5 --> P2["Transaction History"]

    M6 --> R1["Revenue Report"]
    M6 --> R2["Inventory Report"]
    M6 --> R3["Technician Report"]
    M6 --> R4["Profit Report"]
```

---

## 🔧 Service Status Flow

```mermaid
stateDiagram-v2
    [*] --> Pending: Customer submits device
    Pending --> Checking: Technician assigned
    Checking --> WaitingParts: Parts needed
    Checking --> Repairing: No parts needed
    Checking --> Cancelled: Cannot be fixed
    WaitingParts --> Repairing: Parts available
    Repairing --> ReadyPickUp: Repair complete
    ReadyPickUp --> Completed: Customer picks up & pays
    Completed --> [*]
    Cancelled --> [*]
```

---

## 💰 POS Batch Selection Logic (FIFO)

```mermaid
sequenceDiagram
    participant K as Kasir
    participant POS as POS System
    participant INV as Inventory

    K->>POS: Search "Baterai Oppo A3s"
    POS->>INV: Get all batches
    INV-->>POS: Batch 1 (Jan, 100k), Batch 2 (Feb, 110k)
    POS-->>K: Show options
    
    alt Pilih Harga 100k
        K->>POS: Select 100k
        POS->>INV: Get oldest batch where price=100k
        INV-->>POS: Batch 1
    else Input Manual 105k
        K->>POS: Enter 105k
        POS->>INV: Get oldest batch (ignore price)
        INV-->>POS: Batch 1
    end
    
    POS->>INV: Deduct Batch 1 stock
    POS->>POS: Save harga_jual_aktual to sales_details
```

---

## 🎨 Design System

### Color Palette
| Token | Dark Mode | Light Mode | Usage |
|-------|-----------|------------|-------|
| `--bg-primary` | `#0f0f1a` | `#f8f9fa` | Main background |
| `--bg-secondary` | `#1a1a2e` | `#ffffff` | Cards, sidebar |
| `--bg-tertiary` | `#16213e` | `#e9ecef` | Elevated surfaces |
| `--accent-blue` | `#4361ee` | `#4361ee` | Primary actions |
| `--accent-green` | `#2ecc71` | `#2ecc71` | Success, completed |
| `--accent-orange` | `#f39c12` | `#f39c12` | Warnings, pending |
| `--accent-red` | `#e74c3c` | `#e74c3c` | Errors, low stock |
| `--text-primary` | `#ffffff` | `#212529` | Main text |
| `--text-secondary` | `#a0a0b0` | `#6c757d` | Muted text |

### Typography
- **Font Family:** Inter (Google Fonts)
- **Headings:** 700 weight, tracking tight
- **Body:** 400-500 weight

### Theme Toggle
- Default: **System preference** (`prefers-color-scheme`)
- Toggle saved to **localStorage**
- CSS variables switch on `[data-theme]`

### Effects
- **Glassmorphism:** `backdrop-filter: blur(10px)` + semi-transparent backgrounds
- **Shadows:** Layered soft shadows for depth
- **Animations:** Smooth 300ms transitions

---

## 🖼️ ASCII Wireframes

### 1. Main Layout Structure

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  HEADER                                                                      │
│  ┌──────────┐  ┌─────────────────────────────────────┐  🔔  👤 Admin ▼      │
│  │ 🔧 Logo  │  │ 🔍 Search...                        │                      │
│  └──────────┘  └─────────────────────────────────────┘                      │
├─────────────┬────────────────────────────────────────────────────────────────┤
│  SIDEBAR    │  MAIN CONTENT AREA                                            │
│             │                                                                │
│  📊 Dash    │                                                                │
│  📦 Invent  │                                                                │
│  🔧 Service │                                                                │
│  👥 Custom  │                                                                │
│  💰 POS     │                                                                │
│  📈 Report  │                                                                │
│             │                                                                │
│  ─────────  │                                                                │
│  ⚙️ Setting │                                                                │
│  🌙/☀️ Theme│                                                                │
└─────────────┴────────────────────────────────────────────────────────────────┘
```

### 2. Dashboard Page

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  📊 Dashboard                                                    [Today ▼] │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │ 🔧 Service  │  │ ✅ Selesai  │  │ ⏳ Pending  │  │ 💰 Omzet    │        │
│  │     12      │  │     45      │  │      8      │  │  Rp 5.2jt   │        │
│  │  In Progress│  │    Today    │  │   Waiting   │  │    Today    │        │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘        │
│                                                                             │
│  ┌────────────────────────────────────┐  ┌─────────────────────────────┐   │
│  │ 📈 REVENUE CHART                   │  │ ⚠️ LOW STOCK ALERT          │   │
│  │                                    │  │                             │   │
│  │     ╭──╮                          │  │  LCD iPhone 13      [2] 🔴  │   │
│  │    ╭╯  ╰╮    ╭──╮                 │  │  Baterai S21        [3] 🔴  │   │
│  │   ╭╯    ╰──╮╭╯  ╰╮                │  │  Charger Type-C     [5] 🟡  │   │
│  │  ─╯        ╰╯    ╰──              │  │  [View All →]               │   │
│  │  Sen Sel Rab Kam Jum Sab Min      │  │                             │   │
│  └────────────────────────────────────┘  └─────────────────────────────┘   │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ 🕐 RECENT ACTIVITIES                                                  │  │
│  │  • 14:30 - Service #1234 completed by Andi                           │  │
│  │  • 14:15 - New stock: LCD Samsung A52 (10 pcs)                       │  │
│  │  • 13:45 - Sale: Charger iPhone to Budi - Rp 150.000                 │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3. Inventory Page

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  📦 Inventaris                                           [+ Tambah Stok]   │
├─────────────────────────────────────────────────────────────────────────────┤
│  [All] [Spareparts] [Retail]           🔍 Search product...    [Filter ▼] │
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │ PRODUCT              │ SKU      │ GRADE    │ STOK │ HARGA    │ ACTION │ │
│  ├──────────────────────┼──────────┼──────────┼──────┼──────────┼────────┤ │
│  │ 📱 LCD iPhone 13     │ LCD-IP13 │ Original │  2🔴 │ 850.000  │ [···]  │ │
│  │    └─ Batch #1 (Jan) │          │ Brader   │  2   │ 850.000  │        │ │
│  ├──────────────────────┼──────────┼──────────┼──────┼──────────┼────────┤ │
│  │ 🔋 Baterai Oppo A3s  │ BAT-OPA3 │          │  10  │          │ [···]  │ │
│  │    ├─ Batch #1 (Jan) │          │ Original │  5   │ 100.000  │        │ │
│  │    └─ Batch #2 (Feb) │          │ Premium  │  5   │ 110.000  │        │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │ 📜 STOCK HISTORY                                        [View All →] │ │
│  │  [+] Admin added 10x LCD iPhone 13 @ Rp 850.000           2 hours ago │ │
│  │  [-] Service #1234 used 1x Baterai Samsung S21            3 hours ago │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4. Service Orders Page

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  🔧 Service Orders                                       [+ New Service]   │
├─────────────────────────────────────────────────────────────────────────────┤
│  [All] [Pending] [In Progress] [Ready] [Completed]    🔍 Search...         │
│                                                                             │
│  ┌────────────────────────────────────────┬─────────────────────────────┐  │
│  │ SERVICE LIST                           │ DETAIL PANEL                │  │
│  ├────────────────────────────────────────┤                             │  │
│  │ #SV-2024-001      ⏳ Checking         │ Ticket: #SV-2024-001        │  │
│  │ Budi - iPhone 13 Pro                   │ ─────────────────────────── │  │
│  │ LCD Pecah          📅 29 Dec          │ 👤 Budi Santoso             │  │
│  │ Teknisi: Andi                          │    081234567890             │  │
│  ├────────────────────────────────────────│                             │  │
│  │ #SV-2024-002      🔧 Repairing        │ 📱 iPhone 13 Pro            │  │
│  │ Sari - Samsung S21                     │    IMEI: 35467890123456     │  │
│  │ Baterai Drop       📅 28 Dec          │                             │  │
│  │ Teknisi: Beno                          │ 📝 LCD Pecah, tidak bisa    │  │
│  ├────────────────────────────────────────│    touch sama sekali        │  │
│  │ #SV-2024-003      ✅ Ready            │                             │  │
│  │ Ani - Oppo A15 | Ganti Casing         │ 🔧 Parts Used               │  │
│  │                                        │  LCD iPhone 13 x1 - 850k    │  │
│  │                                        │  [+ Add Part]               │  │
│  │                                        │                             │  │
│  │                                        │ STATUS TIMELINE             │  │
│  │                                        │  ● Pending    29 Dec 09:00  │  │
│  │                                        │  ● Checking   29 Dec 10:30  │  │
│  │                                        │  ○ Repairing  -             │  │
│  │                                        │  ○ Ready      -             │  │
│  │                                        │                             │  │
│  │                                        │ [Update Status ▼] [Print]   │  │
│  └────────────────────────────────────────┴─────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5. POS / Kasir Page

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  💰 Point of Sale                                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────┬───────────────────────────┐   │
│  │ PRODUCT SEARCH                          │ 🛒 CART                   │   │
│  │ 🔍 Search product name...               │ Customer: [Select... ▼]   │   │
│  │                                         │                           │   │
│  │ ┌─────────────────────────────────────┐ │ ┌───────────────────────┐ │   │
│  │ │ Baterai Oppo A3s                    │ │ │ Baterai Oppo A3s  x1  │ │   │
│  │ │                                     │ │ │ Original - 100.000    │ │   │
│  │ │ Available Batches:                  │ │ │ [−] 1 [+]    [🗑️]    │ │   │
│  │ │ ┌─────────────────────────────────┐ │ │ ├───────────────────────┤ │   │
│  │ │ │ ○ Original (Jan)    Rp 100.000  │ │ │ │ Charger Type-C   x2  │ │   │
│  │ │ │   Stok: 5           [FIFO: 1st] │ │ │ │ Silver - 45.000      │ │   │
│  │ │ ├─────────────────────────────────┤ │ │ │ [−] 2 [+]    [🗑️]    │ │   │
│  │ │ │ ○ Premium (Feb)     Rp 110.000  │ │ │ └───────────────────────┘ │   │
│  │ │ │   Stok: 5           [FIFO: 2nd] │ │ │                           │   │
│  │ │ └─────────────────────────────────┘ │ │ ─────────────────────────  │   │
│  │ │                                     │ │ Subtotal:     Rp 190.000  │   │
│  │ │ Or enter manual price:              │ │ Discount:     [         ] │   │
│  │ │ Rp [___________] [Add to Cart]      │ │ TOTAL:        Rp 190.000  │   │
│  │ └─────────────────────────────────────┘ │                           │   │
│  │                                         │ Payment: [Cash      ▼]    │   │
│  │ 📦 Quick Add Categories:                │                           │   │
│  │ [Baterai] [LCD] [Charger] [Aksesoris]   │ ┌───────────────────────┐ │   │
│  │                                         │ │   💳 BAYAR SEKARANG   │ │   │
│  │                                         │ └───────────────────────┘ │   │
│  └─────────────────────────────────────────┴───────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6. Customer Page

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  👥 Pelanggan                                            [+ Tambah Baru]   │
├─────────────────────────────────────────────────────────────────────────────┤
│  🔍 Search by name or phone...                                              │
│                                                                             │
│  ┌───────────────────────────────────────┬──────────────────────────────┐  │
│  │ CUSTOMER LIST                         │ DETAIL                       │  │
│  ├───────────────────────────────────────┤ 👤 Budi Santoso              │  │
│  │ 👤 Budi Santoso                       │    📞 081234567890           │  │
│  │    081234567890     5 services        │    ✉️ budi@email.com         │  │
│  ├───────────────────────────────────────│                              │  │
│  │ 👤 Sari Dewi                          │ 📊 Statistics                │  │
│  │    082345678901     3 services        │    Total Services: 5         │  │
│  ├───────────────────────────────────────│    Total Spent: Rp 2.500.000 │  │
│  │ 👤 Ani Kusuma                         │                              │  │
│  │    083456789012     1 service         │ 🕐 Service History           │  │
│  │                                       │ #SV-2024-001 ✅ Completed    │  │
│  │                                       │ iPhone 13 - LCD Pecah        │  │
│  └───────────────────────────────────────┴──────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7. Reports Page

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  📈 Laporan                                     [Dec 2024 ▼] [Export CSV]  │
├─────────────────────────────────────────────────────────────────────────────┤
│  [Revenue] [Inventory] [Technician] [Profit]                                │
│                                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │ Total       │  │ Service     │  │ Retail      │  │ Profit      │        │
│  │ Rp 25.5jt   │  │ Rp 18.2jt   │  │ Rp 7.3jt    │  │ Rp 8.1jt    │        │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘        │
│                                                                             │
│  ┌────────────────────────────────────┐  ┌───────────────────────────────┐ │
│  │ 📊 DAILY REVENUE CHART             │  │ 👨‍🔧 TECHNICIAN PERFORMANCE     │ │
│  │         ╭──╮                       │  │  Andi     ████████████  25    │ │
│  │        ╭╯  ╰╮    ╭──╮             │  │  Beno     ████████      18    │ │
│  │       ╭╯    ╰──╮╭╯  ╰╮            │  │  Citra    █████         12    │ │
│  │      ─╯        ╰╯    ╰──          │  │  Total: 55 services           │ │
│  └────────────────────────────────────┘  └───────────────────────────────┘ │
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │ 📋 PROFIT PER PRODUCT (harga_jual_aktual - harga_beli)               │ │
│  │  Product            │ Qty Sold │ Revenue    │ Cost      │ Profit     │ │
│  │  LCD iPhone 13      │    8     │ 6.800.000  │ 5.600.000 │ 1.200.000  │ │
│  │  Baterai Samsung    │   15     │ 1.500.000  │ 1.050.000 │   450.000  │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 🏗️ Component Hierarchy

```mermaid
flowchart TB
    subgraph APP["App.jsx"]
        subgraph LAYOUT["MainLayout"]
            SIDEBAR["Sidebar"]
            HEADER["Header"]
            CONTENT["Page Content"]
        end
    end

    subgraph PAGES["Pages"]
        P1["Dashboard"]
        P2["Inventory"]
        P3["ServiceOrders"]
        P4["Customers"]
        P5["POS"]
        P6["Reports"]
    end

    subgraph COMPONENTS["Shared Components"]
        C1["Button"]
        C2["Card"]
        C3["Modal"]
        C4["DataTable"]
        C5["StatusBadge"]
        C6["SearchBar"]
        C7["ThemeToggle"]
    end

    CONTENT --> PAGES
    PAGES --> COMPONENTS
    HEADER --> C6
    HEADER --> C7
```

---

## 📁 File Structure

```
src/
├── components/
│   ├── common/          # Button, Card, Modal, Badge, DataTable
│   ├── layout/          # Sidebar, Header, MainLayout
│   ├── dashboard/       # StatusCards, Charts, Alerts
│   ├── inventory/       # ItemCard, StockHistory, BatchList
│   ├── service/         # ServiceForm, StatusTimeline, PartSelector
│   ├── customers/       # CustomerCard, HistoryList
│   ├── pos/             # Cart, ProductGrid, BatchSelector, PaymentModal
│   └── reports/         # ChartComponents, Tables
├── pages/               # Dashboard, Inventory, Service, Customers, POS, Reports
├── context/             # ThemeContext, AppContext
├── hooks/               # useTheme, useLocalStorage
├── styles/
│   ├── index.css
│   └── variables.css
├── utils/               # Helpers, formatters
├── App.jsx
└── main.jsx
```

---

## 📱 Responsive Behavior

| Breakpoint | Behavior |
|------------|----------|
| **Desktop (> 1200px)** | Full sidebar, all panels visible |
| **Tablet (768-1200px)** | Collapsible sidebar, stacked panels |
| **Mobile (< 768px)** | Bottom navigation bar, single column |

---

## 🖼️ Visual Mockups

````carousel
![Dashboard Overview](C:/Users/Good/.gemini/antigravity/brain/23ea58ac-0e2d-46bd-b5d7-312090ca071c/dashboard_overview_mockup_1767002592488.png)
<!-- slide -->
![Service Tracking](C:/Users/Good/.gemini/antigravity/brain/23ea58ac-0e2d-46bd-b5d7-312090ca071c/service_tracking_mockup_1767002635654.png)
<!-- slide -->
![Inventory Management](C:/Users/Good/.gemini/antigravity/brain/23ea58ac-0e2d-46bd-b5d7-312090ca071c/inventory_management_mockup_1767002659910.png)
````

---

## ⚙️ Tech Stack

| Layer | Technology |
|-------|------------|
| Framework | React 18 + Vite |
| Routing | React Router v6 |
| State | React Context + useReducer |
| Styling | Vanilla CSS + CSS Variables |
| Icons | Lucide React |
| Charts | Recharts |
| Notifications | React Hot Toast |

---

## ✅ User Decisions (Approved)

| Item | Decision |
|------|----------|
| Theme | Dark mode default, with toggle (follows system preference) |
| Sidebar Order | Approved as proposed |
| Barcode Scanner | Deferred to Phase 2 |
| Additional Features | None needed for Phase 1 |
