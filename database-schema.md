# Database Schema Design for E-Commerce Module

## Overview
This document outlines the comprehensive database schema for the multi-vendor e-commerce module, designed to integrate seamlessly with existing OmarStartKit modules.

## Schema Principles
- **Modular Integration**: Leverages existing tables from BusinessDirectory, Warehouses, and Accounting modules
- **Multi-tenant Architecture**: Business-level data isolation
- **Scalable Design**: Optimized for high-volume e-commerce operations
- **Flexible Structure**: Supports various product types and business models

## Core E-Commerce Tables

### Products and Catalog

```sql
-- Core product information
CREATE TABLE ecommerce_products (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    business_id BIGINT UNSIGNED NOT NULL,                -- Links to BusinessDirectory
    warehouse_id BIGINT UNSIGNED NULL,                   -- Links to Warehouses module
    product_category_id BIGINT UNSIGNED NULL,
    parent_id BIGINT UNSIGNED NULL,                      -- For product variants
    
    -- Basic Information
    sku VARCHAR(100) UNIQUE NOT NULL,
    name_en VARCHAR(255) NOT NULL,
    name_ar VARCHAR(255) NULL,
    slug VARCHAR(255) UNIQUE NOT NULL,
    barcode VARCHAR(100) NULL,
    
    -- Description and Content
    description_en LONGTEXT NULL,
    description_ar LONGTEXT NULL,
    short_description_en TEXT NULL,
    short_description_ar TEXT NULL,
    
    -- Pricing
    price DECIMAL(15,4) NOT NULL,
    compare_price DECIMAL(15,4) NULL,                    -- Original price for discount display
    cost_price DECIMAL(15,4) NULL,                       -- For profit calculations
    tax_class VARCHAR(50) DEFAULT 'standard',
    
    -- Physical Properties
    weight DECIMAL(8,3) NULL,                            -- In kg
    dimensions JSON NULL,                                -- {length, width, height} in cm
    
    -- Status and Visibility
    status ENUM('draft', 'active', 'inactive', 'out_of_stock', 'discontinued') DEFAULT 'draft',
    visibility ENUM('public', 'private', 'hidden') DEFAULT 'public',
    featured BOOLEAN DEFAULT FALSE,
    
    -- Product Type
    product_type ENUM('physical', 'digital', 'service', 'subscription') DEFAULT 'physical',
    digital_file_url VARCHAR(500) NULL,                 -- For digital products
    requires_shipping BOOLEAN DEFAULT TRUE,
    
    -- Inventory Management
    track_inventory BOOLEAN DEFAULT TRUE,
    stock_quantity INT DEFAULT 0,
    stock_threshold INT DEFAULT 5,                       -- Low stock alert threshold
    allow_backorders BOOLEAN DEFAULT FALSE,
    
    -- SEO and Marketing
    meta_title VARCHAR(255) NULL,
    meta_description TEXT NULL,
    seo_keywords TEXT NULL,
    tags JSON NULL,                                      -- Array of tags
    
    -- Timestamps
    published_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP NULL,
    
    -- Indexes
    INDEX idx_business_id (business_id),
    INDEX idx_warehouse_id (warehouse_id),
    INDEX idx_category_id (product_category_id),
    INDEX idx_sku (sku),
    INDEX idx_slug (slug),
    INDEX idx_status_visibility (status, visibility),
    INDEX idx_featured (featured),
    
    -- Foreign Key Constraints
    FOREIGN KEY (business_id) REFERENCES businesses(id) ON DELETE CASCADE,
    FOREIGN KEY (warehouse_id) REFERENCES warehouses(id) ON DELETE SET NULL,
    FOREIGN KEY (product_category_id) REFERENCES ecommerce_product_categories(id) ON DELETE SET NULL,
    FOREIGN KEY (parent_id) REFERENCES ecommerce_products(id) ON DELETE CASCADE
);

-- Product categories hierarchy
CREATE TABLE ecommerce_product_categories (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    parent_id BIGINT UNSIGNED NULL,
    business_id BIGINT UNSIGNED NULL,                    -- NULL for global categories
    
    name_en VARCHAR(255) NOT NULL,
    name_ar VARCHAR(255) NULL,
    slug VARCHAR(255) UNIQUE NOT NULL,
    description_en TEXT NULL,
    description_ar TEXT NULL,
    
    image_url VARCHAR(500) NULL,
    icon VARCHAR(100) NULL,                              -- Icon class or file
    color VARCHAR(7) NULL,                               -- Hex color code
    
    sort_order INT DEFAULT 0,
    is_active BOOLEAN DEFAULT TRUE,
    is_featured BOOLEAN DEFAULT FALSE,
    
    -- SEO
    meta_title VARCHAR(255) NULL,
    meta_description TEXT NULL,
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX idx_parent_id (parent_id),
    INDEX idx_business_id (business_id),
    INDEX idx_slug (slug),
    INDEX idx_active_featured (is_active, is_featured),
    
    FOREIGN KEY (parent_id) REFERENCES ecommerce_product_categories(id) ON DELETE CASCADE,
    FOREIGN KEY (business_id) REFERENCES businesses(id) ON DELETE CASCADE
);

-- Product attributes (size, color, material, etc.)
CREATE TABLE ecommerce_product_attributes (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    business_id BIGINT UNSIGNED NULL,                    -- NULL for global attributes
    
    name_en VARCHAR(100) NOT NULL,
    name_ar VARCHAR(100) NULL,
    slug VARCHAR(100) NOT NULL,
    type ENUM('text', 'number', 'select', 'multiselect', 'boolean', 'date', 'color') NOT NULL,
    
    is_required BOOLEAN DEFAULT FALSE,
    is_filterable BOOLEAN DEFAULT TRUE,
    is_comparable BOOLEAN DEFAULT TRUE,
    sort_order INT DEFAULT 0,
    
    -- For select/multiselect types
    options JSON NULL,                                   -- Array of options
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX idx_business_id (business_id),
    INDEX idx_slug (slug),
    INDEX idx_type (type),
    
    FOREIGN KEY (business_id) REFERENCES businesses(id) ON DELETE CASCADE
);

-- Product attribute values
CREATE TABLE ecommerce_product_attribute_values (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    product_id BIGINT UNSIGNED NOT NULL,
    attribute_id BIGINT UNSIGNED NOT NULL,
    
    value_text TEXT NULL,
    value_number DECIMAL(15,4) NULL,
    value_boolean BOOLEAN NULL,
    value_date DATE NULL,
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    UNIQUE KEY unique_product_attribute (product_id, attribute_id),
    INDEX idx_attribute_id (attribute_id),
    
    FOREIGN KEY (product_id) REFERENCES ecommerce_products(id) ON DELETE CASCADE,
    FOREIGN KEY (attribute_id) REFERENCES ecommerce_product_attributes(id) ON DELETE CASCADE
);

-- Product images and media
CREATE TABLE ecommerce_product_media (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    product_id BIGINT UNSIGNED NOT NULL,
    
    file_url VARCHAR(500) NOT NULL,
    file_name VARCHAR(255) NOT NULL,
    file_size INT NULL,                                  -- In bytes
    mime_type VARCHAR(100) NULL,
    
    type ENUM('image', 'video', 'document') DEFAULT 'image',
    alt_text_en VARCHAR(255) NULL,
    alt_text_ar VARCHAR(255) NULL,
    
    is_primary BOOLEAN DEFAULT FALSE,
    sort_order INT DEFAULT 0,
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_product_id (product_id),
    INDEX idx_type (type),
    INDEX idx_primary (is_primary),
    
    FOREIGN KEY (product_id) REFERENCES ecommerce_products(id) ON DELETE CASCADE
);
```

### Vendor Management

```sql
-- Vendor profiles (extends businesses)
CREATE TABLE ecommerce_vendors (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    business_id BIGINT UNSIGNED UNIQUE NOT NULL,        -- One-to-one with businesses table
    user_id BIGINT UNSIGNED NOT NULL,                   -- Primary contact user
    
    -- Vendor Information
    vendor_code VARCHAR(50) UNIQUE NOT NULL,
    display_name VARCHAR(255) NOT NULL,
    
    -- Commission and Payouts
    commission_rate DECIMAL(5,2) DEFAULT 10.00,         -- Percentage
    commission_type ENUM('percentage', 'fixed') DEFAULT 'percentage',
    min_payout_amount DECIMAL(15,4) DEFAULT 100.00,
    payout_schedule ENUM('daily', 'weekly', 'monthly') DEFAULT 'monthly',
    
    -- Banking Information
    bank_name VARCHAR(255) NULL,
    bank_account_number VARCHAR(100) NULL,
    bank_routing_number VARCHAR(100) NULL,
    payment_method ENUM('bank_transfer', 'paypal', 'stripe') DEFAULT 'bank_transfer',
    
    -- Status and Metrics
    status ENUM('pending', 'active', 'suspended', 'banned') DEFAULT 'pending',
    verification_status ENUM('unverified', 'pending', 'verified', 'rejected') DEFAULT 'unverified',
    rating DECIMAL(3,2) DEFAULT 0.00,
    total_sales DECIMAL(15,4) DEFAULT 0.00,
    total_orders INT DEFAULT 0,
    
    -- Settings
    auto_approve_products BOOLEAN DEFAULT FALSE,
    allow_cod BOOLEAN DEFAULT TRUE,
    shipping_policy TEXT NULL,
    return_policy TEXT NULL,
    
    -- Timestamps
    onboarded_at TIMESTAMP NULL,
    last_login_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX idx_business_id (business_id),
    INDEX idx_user_id (user_id),
    INDEX idx_vendor_code (vendor_code),
    INDEX idx_status (status),
    INDEX idx_verification_status (verification_status),
    
    FOREIGN KEY (business_id) REFERENCES businesses(id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

-- Vendor commission history
CREATE TABLE ecommerce_vendor_commissions (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    vendor_id BIGINT UNSIGNED NOT NULL,
    order_id BIGINT UNSIGNED NOT NULL,
    
    order_amount DECIMAL(15,4) NOT NULL,
    commission_rate DECIMAL(5,2) NOT NULL,
    commission_amount DECIMAL(15,4) NOT NULL,
    platform_fee DECIMAL(15,4) DEFAULT 0.00,
    
    status ENUM('pending', 'approved', 'paid', 'cancelled') DEFAULT 'pending',
    
    paid_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_vendor_id (vendor_id),
    INDEX idx_order_id (order_id),
    INDEX idx_status (status),
    
    FOREIGN KEY (vendor_id) REFERENCES ecommerce_vendors(id) ON DELETE CASCADE,
    FOREIGN KEY (order_id) REFERENCES ecommerce_orders(id) ON DELETE CASCADE
);
```

### Orders and Shopping

```sql
-- Shopping cart
CREATE TABLE ecommerce_carts (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT UNSIGNED NULL,                       -- NULL for guest carts
    session_id VARCHAR(255) NULL,                       -- For guest carts
    business_id BIGINT UNSIGNED NULL,                   -- Customer's business context
    
    currency_code VARCHAR(3) DEFAULT 'EGP',
    
    -- Cart totals (calculated values)
    subtotal DECIMAL(15,4) DEFAULT 0.00,
    tax_amount DECIMAL(15,4) DEFAULT 0.00,
    discount_amount DECIMAL(15,4) DEFAULT 0.00,
    shipping_amount DECIMAL(15,4) DEFAULT 0.00,
    total_amount DECIMAL(15,4) DEFAULT 0.00,
    
    -- Applied discounts
    coupon_code VARCHAR(100) NULL,
    discount_id BIGINT UNSIGNED NULL,
    
    -- Status
    status ENUM('active', 'abandoned', 'converted') DEFAULT 'active',
    
    expires_at TIMESTAMP NULL,                          -- For automatic cleanup
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX idx_user_id (user_id),
    INDEX idx_session_id (session_id),
    INDEX idx_business_id (business_id),
    INDEX idx_status (status),
    INDEX idx_expires_at (expires_at),
    
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (business_id) REFERENCES businesses(id) ON DELETE CASCADE
);

-- Cart items
CREATE TABLE ecommerce_cart_items (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    cart_id BIGINT UNSIGNED NOT NULL,
    product_id BIGINT UNSIGNED NOT NULL,
    vendor_id BIGINT UNSIGNED NOT NULL,
    
    quantity INT NOT NULL DEFAULT 1,
    unit_price DECIMAL(15,4) NOT NULL,
    total_price DECIMAL(15,4) NOT NULL,
    
    -- Product snapshot (in case product changes)
    product_name VARCHAR(255) NOT NULL,
    product_sku VARCHAR(100) NOT NULL,
    product_image VARCHAR(500) NULL,
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    UNIQUE KEY unique_cart_product (cart_id, product_id),
    INDEX idx_product_id (product_id),
    INDEX idx_vendor_id (vendor_id),
    
    FOREIGN KEY (cart_id) REFERENCES ecommerce_carts(id) ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES ecommerce_products(id) ON DELETE CASCADE,
    FOREIGN KEY (vendor_id) REFERENCES ecommerce_vendors(id) ON DELETE CASCADE
);

-- Orders
CREATE TABLE ecommerce_orders (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    business_id BIGINT UNSIGNED NULL,                   -- Customer's business context
    customer_id BIGINT UNSIGNED NOT NULL,
    
    -- Order identification
    order_number VARCHAR(100) UNIQUE NOT NULL,
    invoice_number VARCHAR(100) NULL,
    
    -- Order status
    status ENUM('pending', 'confirmed', 'processing', 'shipped', 'delivered', 'cancelled', 'refunded') DEFAULT 'pending',
    payment_status ENUM('pending', 'processing', 'paid', 'failed', 'cancelled', 'refunded') DEFAULT 'pending',
    fulfillment_status ENUM('pending', 'processing', 'shipped', 'delivered', 'cancelled') DEFAULT 'pending',
    
    -- Financial information
    currency_code VARCHAR(3) DEFAULT 'EGP',
    subtotal DECIMAL(15,4) NOT NULL,
    tax_amount DECIMAL(15,4) DEFAULT 0.00,
    shipping_amount DECIMAL(15,4) DEFAULT 0.00,
    discount_amount DECIMAL(15,4) DEFAULT 0.00,
    total_amount DECIMAL(15,4) NOT NULL,
    
    -- Applied discounts
    coupon_code VARCHAR(100) NULL,
    discount_description VARCHAR(255) NULL,
    
    -- Addresses (JSON for flexibility)
    billing_address JSON NOT NULL,
    shipping_address JSON NOT NULL,
    
    -- Order details
    notes TEXT NULL,
    admin_notes TEXT NULL,
    
    -- Important timestamps
    placed_at TIMESTAMP NOT NULL,
    confirmed_at TIMESTAMP NULL,
    shipped_at TIMESTAMP NULL,
    delivered_at TIMESTAMP NULL,
    cancelled_at TIMESTAMP NULL,
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX idx_business_id (business_id),
    INDEX idx_customer_id (customer_id),
    INDEX idx_order_number (order_number),
    INDEX idx_status (status),
    INDEX idx_payment_status (payment_status),
    INDEX idx_fulfillment_status (fulfillment_status),
    INDEX idx_placed_at (placed_at),
    
    FOREIGN KEY (business_id) REFERENCES businesses(id) ON DELETE SET NULL,
    FOREIGN KEY (customer_id) REFERENCES users(id) ON DELETE CASCADE
);

-- Order items (supports multi-vendor orders)
CREATE TABLE ecommerce_order_items (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    order_id BIGINT UNSIGNED NOT NULL,
    product_id BIGINT UNSIGNED NOT NULL,
    vendor_id BIGINT UNSIGNED NOT NULL,
    
    -- Product information at time of order
    product_name VARCHAR(255) NOT NULL,
    product_sku VARCHAR(100) NOT NULL,
    product_image VARCHAR(500) NULL,
    
    -- Quantity and pricing
    quantity INT NOT NULL,
    unit_price DECIMAL(15,4) NOT NULL,
    total_price DECIMAL(15,4) NOT NULL,
    
    -- Item-specific status
    fulfillment_status ENUM('pending', 'processing', 'shipped', 'delivered', 'cancelled') DEFAULT 'pending',
    
    -- Tracking
    tracking_number VARCHAR(255) NULL,
    tracking_url VARCHAR(500) NULL,
    
    shipped_at TIMESTAMP NULL,
    delivered_at TIMESTAMP NULL,
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX idx_order_id (order_id),
    INDEX idx_product_id (product_id),
    INDEX idx_vendor_id (vendor_id),
    INDEX idx_fulfillment_status (fulfillment_status),
    
    FOREIGN KEY (order_id) REFERENCES ecommerce_orders(id) ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES ecommerce_products(id) ON DELETE CASCADE,
    FOREIGN KEY (vendor_id) REFERENCES ecommerce_vendors(id) ON DELETE CASCADE
);
```

### Payment and Shipping

```sql
-- Payment methods
CREATE TABLE ecommerce_payment_methods (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    
    name_en VARCHAR(100) NOT NULL,
    name_ar VARCHAR(100) NULL,
    code VARCHAR(50) UNIQUE NOT NULL,
    
    gateway VARCHAR(100) NOT NULL,                      -- stripe, paypal, fawry, etc.
    type ENUM('card', 'wallet', 'bank_transfer', 'cash_on_delivery') NOT NULL,
    
    is_active BOOLEAN DEFAULT TRUE,
    sort_order INT DEFAULT 0,
    
    -- Configuration (JSON)
    settings JSON NULL,
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX idx_code (code),
    INDEX idx_gateway (gateway),
    INDEX idx_active (is_active)
);

-- Payment transactions
CREATE TABLE ecommerce_payment_transactions (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    order_id BIGINT UNSIGNED NOT NULL,
    payment_method_id BIGINT UNSIGNED NOT NULL,
    
    -- Transaction details
    transaction_id VARCHAR(255) UNIQUE NOT NULL,
    gateway_transaction_id VARCHAR(255) NULL,           -- Gateway's transaction ID
    
    type ENUM('payment', 'refund', 'partial_refund') DEFAULT 'payment',
    status ENUM('pending', 'processing', 'completed', 'failed', 'cancelled') DEFAULT 'pending',
    
    -- Amounts
    amount DECIMAL(15,4) NOT NULL,
    currency_code VARCHAR(3) DEFAULT 'EGP',
    gateway_fee DECIMAL(15,4) DEFAULT 0.00,
    
    -- Gateway response data
    gateway_response JSON NULL,
    
    -- Timestamps
    processed_at TIMESTAMP NULL,
    failed_at TIMESTAMP NULL,
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX idx_order_id (order_id),
    INDEX idx_payment_method_id (payment_method_id),
    INDEX idx_transaction_id (transaction_id),
    INDEX idx_gateway_transaction_id (gateway_transaction_id),
    INDEX idx_status (status),
    
    FOREIGN KEY (order_id) REFERENCES ecommerce_orders(id) ON DELETE CASCADE,
    FOREIGN KEY (payment_method_id) REFERENCES ecommerce_payment_methods(id) ON DELETE CASCADE
);

-- Shipping methods
CREATE TABLE ecommerce_shipping_methods (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    vendor_id BIGINT UNSIGNED NULL,                     -- NULL for global shipping methods
    
    name_en VARCHAR(100) NOT NULL,
    name_ar VARCHAR(100) NULL,
    code VARCHAR(50) NOT NULL,
    
    type ENUM('flat_rate', 'free_shipping', 'local_pickup', 'calculated') DEFAULT 'flat_rate',
    
    -- Pricing
    base_cost DECIMAL(15,4) DEFAULT 0.00,
    per_item_cost DECIMAL(15,4) DEFAULT 0.00,
    per_kg_cost DECIMAL(15,4) DEFAULT 0.00,
    
    -- Conditions
    min_order_amount DECIMAL(15,4) DEFAULT 0.00,
    max_order_amount DECIMAL(15,4) NULL,
    
    -- Delivery time
    min_delivery_days INT DEFAULT 1,
    max_delivery_days INT DEFAULT 7,
    
    -- Settings
    is_active BOOLEAN DEFAULT TRUE,
    sort_order INT DEFAULT 0,
    
    -- Configuration
    settings JSON NULL,
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX idx_vendor_id (vendor_id),
    INDEX idx_code (code),
    INDEX idx_active (is_active),
    
    FOREIGN KEY (vendor_id) REFERENCES ecommerce_vendors(id) ON DELETE CASCADE
);

-- Shipping zones
CREATE TABLE ecommerce_shipping_zones (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    vendor_id BIGINT UNSIGNED NULL,                     -- NULL for global zones
    
    name VARCHAR(100) NOT NULL,
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX idx_vendor_id (vendor_id),
    
    FOREIGN KEY (vendor_id) REFERENCES ecommerce_vendors(id) ON DELETE CASCADE
);

-- Shipping zone locations (cities/regions)
CREATE TABLE ecommerce_shipping_zone_locations (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    zone_id BIGINT UNSIGNED NOT NULL,
    city_id BIGINT UNSIGNED NULL,                       -- Links to cities table
    region_id BIGINT UNSIGNED NULL,                     -- Links to regions table
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_zone_id (zone_id),
    INDEX idx_city_id (city_id),
    INDEX idx_region_id (region_id),
    
    FOREIGN KEY (zone_id) REFERENCES ecommerce_shipping_zones(id) ON DELETE CASCADE,
    FOREIGN KEY (city_id) REFERENCES cities(id) ON DELETE CASCADE,
    FOREIGN KEY (region_id) REFERENCES regions(id) ON DELETE CASCADE
);

-- Shipping method rates per zone
CREATE TABLE ecommerce_shipping_method_zones (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    shipping_method_id BIGINT UNSIGNED NOT NULL,
    zone_id BIGINT UNSIGNED NOT NULL,
    
    cost DECIMAL(15,4) NOT NULL,
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    UNIQUE KEY unique_method_zone (shipping_method_id, zone_id),
    
    FOREIGN KEY (shipping_method_id) REFERENCES ecommerce_shipping_methods(id) ON DELETE CASCADE,
    FOREIGN KEY (zone_id) REFERENCES ecommerce_shipping_zones(id) ON DELETE CASCADE
);
```

### Marketing and Promotions

```sql
-- Coupons and discounts
CREATE TABLE ecommerce_coupons (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    vendor_id BIGINT UNSIGNED NULL,                     -- NULL for platform-wide coupons
    
    code VARCHAR(100) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    description TEXT NULL,
    
    type ENUM('fixed_amount', 'percentage', 'free_shipping') NOT NULL,
    value DECIMAL(15,4) NOT NULL,
    
    -- Usage limits
    usage_limit INT NULL,                               -- NULL = unlimited
    usage_limit_per_customer INT DEFAULT 1,
    used_count INT DEFAULT 0,
    
    -- Conditions
    minimum_amount DECIMAL(15,4) DEFAULT 0.00,
    maximum_discount DECIMAL(15,4) NULL,               -- For percentage discounts
    
    -- Product/category restrictions
    applicable_products JSON NULL,                      -- Array of product IDs
    applicable_categories JSON NULL,                    -- Array of category IDs
    excluded_products JSON NULL,                        -- Array of product IDs
    excluded_categories JSON NULL,                      -- Array of category IDs
    
    -- Customer restrictions
    applicable_customers JSON NULL,                     -- Array of customer IDs
    first_purchase_only BOOLEAN DEFAULT FALSE,
    
    -- Date restrictions
    starts_at TIMESTAMP NULL,
    expires_at TIMESTAMP NULL,
    
    -- Status
    is_active BOOLEAN DEFAULT TRUE,
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX idx_vendor_id (vendor_id),
    INDEX idx_code (code),
    INDEX idx_active (is_active),
    INDEX idx_dates (starts_at, expires_at),
    
    FOREIGN KEY (vendor_id) REFERENCES ecommerce_vendors(id) ON DELETE CASCADE
);

-- Coupon usage history
CREATE TABLE ecommerce_coupon_usage (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    coupon_id BIGINT UNSIGNED NOT NULL,
    order_id BIGINT UNSIGNED NOT NULL,
    customer_id BIGINT UNSIGNED NOT NULL,
    
    discount_amount DECIMAL(15,4) NOT NULL,
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_coupon_id (coupon_id),
    INDEX idx_order_id (order_id),
    INDEX idx_customer_id (customer_id),
    
    FOREIGN KEY (coupon_id) REFERENCES ecommerce_coupons(id) ON DELETE CASCADE,
    FOREIGN KEY (order_id) REFERENCES ecommerce_orders(id) ON DELETE CASCADE,
    FOREIGN KEY (customer_id) REFERENCES users(id) ON DELETE CASCADE
);
```

## Integration Tables

### BusinessDirectory Integration

```sql
-- Links vendors to their business profiles
-- (This is handled by the vendor_id -> business_id relationship in ecommerce_vendors table)

-- Vendor store settings (extends business information)
CREATE TABLE ecommerce_vendor_store_settings (
    vendor_id BIGINT UNSIGNED PRIMARY KEY,
    
    -- Store customization
    store_name VARCHAR(255) NOT NULL,
    store_slug VARCHAR(255) UNIQUE NOT NULL,
    store_description TEXT NULL,
    store_logo VARCHAR(500) NULL,
    store_banner VARCHAR(500) NULL,
    
    -- Store policies
    return_policy TEXT NULL,
    shipping_policy TEXT NULL,
    privacy_policy TEXT NULL,
    terms_of_service TEXT NULL,
    
    -- Social media
    facebook_url VARCHAR(500) NULL,
    instagram_url VARCHAR(500) NULL,
    twitter_url VARCHAR(500) NULL,
    website_url VARCHAR(500) NULL,
    
    -- SEO
    meta_title VARCHAR(255) NULL,
    meta_description TEXT NULL,
    meta_keywords TEXT NULL,
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    FOREIGN KEY (vendor_id) REFERENCES ecommerce_vendors(id) ON DELETE CASCADE
);
```

### Warehouses Integration

```sql
-- Links products to warehouse inventory items
CREATE TABLE ecommerce_product_inventory_sync (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    product_id BIGINT UNSIGNED NOT NULL,
    inventory_item_id BIGINT UNSIGNED NOT NULL,         -- From warehouses module
    
    sync_enabled BOOLEAN DEFAULT TRUE,
    sync_direction ENUM('product_to_inventory', 'inventory_to_product', 'bidirectional') DEFAULT 'bidirectional',
    
    last_synced_at TIMESTAMP NULL,
    sync_status ENUM('synced', 'pending', 'error') DEFAULT 'synced',
    sync_error_message TEXT NULL,
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    UNIQUE KEY unique_product_inventory (product_id, inventory_item_id),
    INDEX idx_inventory_item_id (inventory_item_id),
    INDEX idx_sync_enabled (sync_enabled),
    INDEX idx_sync_status (sync_status),
    
    FOREIGN KEY (product_id) REFERENCES ecommerce_products(id) ON DELETE CASCADE,
    FOREIGN KEY (inventory_item_id) REFERENCES inventory_items(id) ON DELETE CASCADE
);

-- Stock movement tracking for e-commerce transactions
CREATE TABLE ecommerce_stock_movements (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    product_id BIGINT UNSIGNED NOT NULL,
    order_id BIGINT UNSIGNED NULL,                      -- NULL for manual adjustments
    stock_movement_id BIGINT UNSIGNED NULL,             -- Links to warehouses stock_movements
    
    movement_type ENUM('sale', 'return', 'adjustment', 'reservation', 'release') NOT NULL,
    quantity_change INT NOT NULL,                       -- Positive or negative
    
    reason VARCHAR(255) NULL,
    notes TEXT NULL,
    
    created_by BIGINT UNSIGNED NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_product_id (product_id),
    INDEX idx_order_id (order_id),
    INDEX idx_stock_movement_id (stock_movement_id),
    INDEX idx_movement_type (movement_type),
    
    FOREIGN KEY (product_id) REFERENCES ecommerce_products(id) ON DELETE CASCADE,
    FOREIGN KEY (order_id) REFERENCES ecommerce_orders(id) ON DELETE CASCADE,
    FOREIGN KEY (stock_movement_id) REFERENCES stock_movements(id) ON DELETE SET NULL,
    FOREIGN KEY (created_by) REFERENCES users(id) ON DELETE SET NULL
);
```

### Accounting Integration

```sql
-- Links products to chart of accounts
CREATE TABLE ecommerce_product_accounts (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    product_id BIGINT UNSIGNED NOT NULL,
    
    revenue_account_id BIGINT UNSIGNED NOT NULL,        -- From accounting module
    cogs_account_id BIGINT UNSIGNED NULL,               -- Cost of goods sold
    tax_account_id BIGINT UNSIGNED NULL,                -- Tax account
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    UNIQUE KEY unique_product_accounts (product_id),
    INDEX idx_revenue_account_id (revenue_account_id),
    INDEX idx_cogs_account_id (cogs_account_id),
    
    FOREIGN KEY (product_id) REFERENCES ecommerce_products(id) ON DELETE CASCADE,
    FOREIGN KEY (revenue_account_id) REFERENCES chart_of_accounts(id) ON DELETE CASCADE,
    FOREIGN KEY (cogs_account_id) REFERENCES chart_of_accounts(id) ON DELETE SET NULL,
    FOREIGN KEY (tax_account_id) REFERENCES chart_of_accounts(id) ON DELETE SET NULL
);

-- Links orders to journal entries
CREATE TABLE ecommerce_order_journal_entries (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    order_id BIGINT UNSIGNED NOT NULL,
    journal_entry_id BIGINT UNSIGNED NOT NULL,          -- From accounting module
    
    entry_type ENUM('sale', 'payment', 'refund', 'commission', 'tax', 'shipping') NOT NULL,
    amount DECIMAL(15,4) NOT NULL,
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_order_id (order_id),
    INDEX idx_journal_entry_id (journal_entry_id),
    INDEX idx_entry_type (entry_type),
    
    FOREIGN KEY (order_id) REFERENCES ecommerce_orders(id) ON DELETE CASCADE,
    FOREIGN KEY (journal_entry_id) REFERENCES journal_entries(id) ON DELETE CASCADE
);

-- Vendor commission journal entries
CREATE TABLE ecommerce_vendor_commission_entries (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    commission_id BIGINT UNSIGNED NOT NULL,
    journal_entry_id BIGINT UNSIGNED NOT NULL,          -- From accounting module
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_commission_id (commission_id),
    INDEX idx_journal_entry_id (journal_entry_id),
    
    FOREIGN KEY (commission_id) REFERENCES ecommerce_vendor_commissions(id) ON DELETE CASCADE,
    FOREIGN KEY (journal_entry_id) REFERENCES journal_entries(id) ON DELETE CASCADE
);
```

## Indexes and Performance Optimization

### Primary Indexes
```sql
-- Product search and filtering
CREATE INDEX idx_products_search ON ecommerce_products (status, visibility, business_id, product_category_id);
CREATE INDEX idx_products_featured ON ecommerce_products (featured, status, visibility);
CREATE INDEX idx_products_price ON ecommerce_products (price, status, visibility);

-- Order management
CREATE INDEX idx_orders_vendor_status ON ecommerce_order_items (vendor_id, fulfillment_status);
CREATE INDEX idx_orders_customer_date ON ecommerce_orders (customer_id, placed_at);
CREATE INDEX idx_orders_business_date ON ecommerce_orders (business_id, placed_at);

-- Vendor performance
CREATE INDEX idx_vendor_performance ON ecommerce_vendors (status, rating, total_sales);
CREATE INDEX idx_commission_payout ON ecommerce_vendor_commissions (vendor_id, status, created_at);

-- Analytics queries
CREATE INDEX idx_analytics_sales ON ecommerce_orders (placed_at, status, total_amount);
CREATE INDEX idx_analytics_products ON ecommerce_order_items (product_id, created_at);
```

### Composite Indexes for Common Queries
```sql
-- Product catalog queries
CREATE INDEX idx_catalog_active ON ecommerce_products (status, visibility, featured, created_at);
CREATE INDEX idx_catalog_category ON ecommerce_products (product_category_id, status, visibility, price);

-- Cart and order processing
CREATE INDEX idx_cart_user_session ON ecommerce_carts (user_id, session_id, status);
CREATE INDEX idx_order_processing ON ecommerce_orders (status, payment_status, placed_at);
```

## Data Relationships Summary

### Primary Relationships
1. **Products ↔ Businesses**: Many-to-one (via business_id)
2. **Vendors ↔ Businesses**: One-to-one (via business_id)
3. **Orders ↔ Users**: Many-to-one (customer relationship)
4. **Order Items ↔ Vendors**: Many-to-one (multi-vendor support)

### Integration Relationships
1. **Products ↔ Inventory Items**: Many-to-many (warehouse integration)
2. **Orders ↔ Journal Entries**: One-to-many (accounting integration)
3. **Vendors ↔ Commission Entries**: One-to-many (accounting integration)
4. **Products ↔ Chart of Accounts**: Many-to-many (accounting integration)

### Cross-Module Dependencies
- **Cities/Regions**: Used for shipping zones and vendor locations
- **Users**: Customers, vendors, and admin users
- **Businesses**: Core entity for multi-tenant architecture
- **Permissions**: Role-based access control integration

This schema design ensures scalability, performance, and seamless integration with existing OmarStartKit modules while maintaining data integrity and supporting complex e-commerce workflows.