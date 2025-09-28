# Development Workflow & Implementation Guide

## Overview
This document provides a step-by-step guide for implementing the multi-vendor e-commerce module within the OmarStartKit ecosystem, including development workflows, testing strategies, and deployment procedures.

## Prerequisites

### System Requirements
- PHP 8.2+
- Laravel 11+
- Node.js 18+
- MySQL 8.0+ or PostgreSQL 13+
- Redis 6.0+
- Composer 2.0+
- NPM/Yarn

### Existing Module Dependencies
- **Shared Module**: Core entities, value objects, and utilities
- **BusinessDirectory Module**: Business profiles and vendor management
- **Warehouses Module**: Inventory management integration
- **Accounting Module**: Financial transactions and reporting
- **HR Module**: User management and permissions

## Phase 1: Foundation Setup (Week 1-2)

### 1.1 Module Structure Creation

```bash
# Create module directory structure
mkdir -p app/Modules/ECommerce/{Application,Domain,Infrastructure,Presentation}
mkdir -p app/Modules/ECommerce/Application/{Commands,DTOs,Events,Listeners,Queries,Services}
mkdir -p app/Modules/ECommerce/Domain/{Contracts,Entities,ValueObjects,Enums}
mkdir -p app/Modules/ECommerce/Infrastructure/{Models,Repositories,Providers,Jobs,Services}
mkdir -p app/Modules/ECommerce/Presentation/{Controllers,Requests,Resources,Middleware}
```

### 1.2 Service Provider Registration

Create the main module service provider:

```php
<?php
// app/Modules/ECommerce/Infrastructure/Providers/ECommerceServiceProvider.php

namespace App\Modules\ECommerce\Infrastructure\Providers;

use App\Modules\Shared\Infrastructure\Providers\BaseModuleServiceProvider;

class ECommerceServiceProvider extends BaseModuleServiceProvider
{
    protected string $moduleSlug = 'ecommerce';
    
    protected array $repositories = [
        \App\Modules\ECommerce\Domain\Contracts\ProductRepositoryInterface::class => 
            \App\Modules\ECommerce\Infrastructure\Repositories\ProductRepository::class,
        \App\Modules\ECommerce\Domain\Contracts\OrderRepositoryInterface::class => 
            \App\Modules\ECommerce\Infrastructure\Repositories\OrderRepository::class,
        \App\Modules\ECommerce\Domain\Contracts\VendorRepositoryInterface::class => 
            \App\Modules\ECommerce\Infrastructure\Repositories\VendorRepository::class,
        \App\Modules\ECommerce\Domain\Contracts\CartRepositoryInterface::class => 
            \App\Modules\ECommerce\Infrastructure\Repositories\CartRepository::class,
    ];
    
    protected array $services = [
        \App\Modules\ECommerce\Application\Services\ProductService::class,
        \App\Modules\ECommerce\Application\Services\OrderService::class,
        \App\Modules\ECommerce\Application\Services\VendorService::class,
        \App\Modules\ECommerce\Application\Services\CartService::class,
        \App\Modules\ECommerce\Application\Services\PaymentService::class,
        \App\Modules\ECommerce\Application\Services\ShippingService::class,
    ];
    
    public function register(): void
    {
        parent::register();
        
        // Register command handlers
        $this->app->bind(
            \App\Modules\ECommerce\Application\Commands\CreateProductCommandHandler::class
        );
        
        $this->app->bind(
            \App\Modules\ECommerce\Application\Commands\CreateOrderCommandHandler::class
        );
    }
    
    public function boot(): void
    {
        parent::boot();
        
        // Load routes
        $this->loadRoutesFrom(__DIR__ . '/../../routes/api.php');
        $this->loadRoutesFrom(__DIR__ . '/../../routes/web.php');
        
        // Load migrations
        $this->loadMigrationsFrom(__DIR__ . '/../../database/migrations');
        
        // Load views
        $this->loadViewsFrom(__DIR__ . '/../../resources/views', 'ecommerce');
        
        // Publish assets
        $this->publishes([
            __DIR__ . '/../../resources/assets' => public_path('vendor/ecommerce'),
        ], 'ecommerce-assets');
        
        // Register event listeners
        $this->registerEventListeners();
    }
    
    private function registerEventListeners(): void
    {
        // Product events
        Event::listen(
            \App\Modules\ECommerce\Application\Events\ProductCreated::class,
            \App\Modules\ECommerce\Application\Listeners\CreateProductInventoryItem::class
        );
        
        Event::listen(
            \App\Modules\ECommerce\Application\Events\ProductPriceUpdated::class,
            \App\Modules\ECommerce\Application\Listeners\UpdateProductPricing::class
        );
        
        // Order events
        Event::listen(
            \App\Modules\ECommerce\Application\Events\OrderConfirmed::class,
            \App\Modules\ECommerce\Application\Listeners\CreateAccountingEntries::class
        );
        
        Event::listen(
            \App\Modules\ECommerce\Application\Events\OrderShipped::class,
            \App\Modules\ECommerce\Application\Listeners\UpdateInventoryLevels::class
        );
        
        // Vendor events
        Event::listen(
            \App\Modules\ECommerce\Application\Events\VendorApproved::class,
            \App\Modules\ECommerce\Application\Listeners\SetupVendorAccounting::class
        );
    }
}
```

### 1.3 Configuration Setup

Create module configuration:

```php
<?php
// config/ecommerce.php

return [
    'enabled' => env('ECOMMERCE_MODULE_ENABLED', true),
    
    'currency' => [
        'default' => env('ECOMMERCE_DEFAULT_CURRENCY', 'EGP'),
        'supported' => ['EGP', 'USD', 'EUR', 'SAR'],
    ],
    
    'vendor' => [
        'auto_approve_products' => env('ECOMMERCE_AUTO_APPROVE_PRODUCTS', false),
        'default_commission_rate' => env('ECOMMERCE_DEFAULT_COMMISSION_RATE', 10.0),
        'min_payout_amount' => env('ECOMMERCE_MIN_PAYOUT_AMOUNT', 100.0),
        'payout_schedule' => env('ECOMMERCE_PAYOUT_SCHEDULE', 'monthly'),
    ],
    
    'orders' => [
        'order_number_prefix' => env('ECOMMERCE_ORDER_PREFIX', 'ORD'),
        'auto_confirm_orders' => env('ECOMMERCE_AUTO_CONFIRM_ORDERS', false),
        'cancellation_window_hours' => env('ECOMMERCE_CANCELLATION_WINDOW', 24),
    ],
    
    'inventory' => [
        'track_inventory_by_default' => env('ECOMMERCE_TRACK_INVENTORY', true),
        'default_stock_threshold' => env('ECOMMERCE_DEFAULT_STOCK_THRESHOLD', 5),
        'allow_backorders_by_default' => env('ECOMMERCE_ALLOW_BACKORDERS', false),
    ],
    
    'cart' => [
        'session_lifetime_minutes' => env('ECOMMERCE_CART_SESSION_LIFETIME', 10080), // 1 week
        'guest_checkout_enabled' => env('ECOMMERCE_GUEST_CHECKOUT', true),
    ],
    
    'payment' => [
        'gateways' => [
            'stripe' => [
                'enabled' => env('STRIPE_ENABLED', false),
                'public_key' => env('STRIPE_KEY'),
                'secret_key' => env('STRIPE_SECRET'),
            ],
            'paypal' => [
                'enabled' => env('PAYPAL_ENABLED', false),
                'client_id' => env('PAYPAL_CLIENT_ID'),
                'client_secret' => env('PAYPAL_CLIENT_SECRET'),
                'sandbox' => env('PAYPAL_SANDBOX', true),
            ],
            'fawry' => [
                'enabled' => env('FAWRY_ENABLED', false),
                'merchant_code' => env('FAWRY_MERCHANT_CODE'),
                'security_key' => env('FAWRY_SECURITY_KEY'),
            ],
        ],
    ],
    
    'shipping' => [
        'providers' => [
            'aramex' => [
                'enabled' => env('ARAMEX_ENABLED', false),
                'username' => env('ARAMEX_USERNAME'),
                'password' => env('ARAMEX_PASSWORD'),
                'account_number' => env('ARAMEX_ACCOUNT_NUMBER'),
            ],
            'dhl' => [
                'enabled' => env('DHL_ENABLED', false),
                'site_id' => env('DHL_SITE_ID'),
                'password' => env('DHL_PASSWORD'),
            ],
        ],
        'default_weight_unit' => 'kg',
        'default_dimension_unit' => 'cm',
    ],
    
    'seo' => [
        'product_url_structure' => env('ECOMMERCE_PRODUCT_URL_STRUCTURE', '/product/{slug}'),
        'category_url_structure' => env('ECOMMERCE_CATEGORY_URL_STRUCTURE', '/category/{slug}'),
        'vendor_store_url_structure' => env('ECOMMERCE_VENDOR_URL_STRUCTURE', '/store/{slug}'),
    ],
    
    'cache' => [
        'product_cache_ttl' => env('ECOMMERCE_PRODUCT_CACHE_TTL', 3600), // 1 hour
        'category_cache_ttl' => env('ECOMMERCE_CATEGORY_CACHE_TTL', 7200), // 2 hours
        'cart_cache_ttl' => env('ECOMMERCE_CART_CACHE_TTL', 1800), // 30 minutes
    ],
];
```

### 1.4 Database Migration Setup

Create initial migration files:

```bash
# Generate migration files
php artisan make:migration create_ecommerce_products_table
php artisan make:migration create_ecommerce_product_categories_table
php artisan make:migration create_ecommerce_vendors_table
php artisan make:migration create_ecommerce_orders_table
php artisan make:migration create_ecommerce_carts_table
```

## Phase 2: Core Domain Implementation (Week 3-4)

### 2.1 Value Objects Implementation

```php
<?php
// app/Modules/ECommerce/Domain/ValueObjects/Money.php

namespace App\Modules\ECommerce\Domain\ValueObjects;

class Money
{
    private float $amount;
    private string $currency;
    
    public function __construct(float $amount, string $currency = 'EGP')
    {
        if ($amount < 0) {
            throw new InvalidArgumentException('Amount cannot be negative');
        }
        
        $this->amount = round($amount, 4);
        $this->currency = strtoupper($currency);
    }
    
    // Implementation methods as shown in architecture document
}
```

### 2.2 Domain Entities Implementation

Implement core entities following the architecture patterns:
- Product Entity
- Order Entity  
- Vendor Entity
- Cart Entity
- OrderItem Entity

### 2.3 Domain Contracts

Create repository interfaces:

```php
<?php
// app/Modules/ECommerce/Domain/Contracts/ProductRepositoryInterface.php

namespace App\Modules\ECommerce\Domain\Contracts;

use App\Modules\ECommerce\Domain\Entities\Product;
use App\Modules\ECommerce\Domain\ValueObjects\SKU;

interface ProductRepositoryInterface
{
    public function findById(int $id): ?Product;
    public function findBySku(SKU $sku): ?Product;
    public function existsBySku(SKU $sku): bool;
    public function save(Product $product): Product;
    public function delete(Product $product): void;
    public function findByVendor(int $vendorId, array $filters = []): LengthAwarePaginator;
}
```

## Phase 3: Application Layer Implementation (Week 5-6)

### 3.1 Command/Query Handlers

Implement CQRS pattern:

```php
<?php
// app/Modules/ECommerce/Application/Commands/CreateProductCommand.php

namespace App\Modules\ECommerce\Application\Commands;

class CreateProductCommand
{
    public function __construct(
        public readonly int $businessId,
        public readonly int $vendorId,
        public readonly string $sku,
        public readonly string $nameEn,
        public readonly float $price,
        // ... other properties
    ) {}
}
```

### 3.2 Application Services

Implement business logic services:
- ProductService
- OrderService
- CartService
- VendorService
- PaymentService
- ShippingService

### 3.3 Event System

Create domain events and listeners:

```php
<?php
// app/Modules/ECommerce/Application/Events/ProductCreated.php

namespace App\Modules\ECommerce\Application\Events;

use App\Modules\ECommerce\Domain\Entities\Product;

class ProductCreated
{
    public function __construct(
        public readonly Product $product
    ) {}
}
```

## Phase 4: Infrastructure Layer Implementation (Week 7-8)

### 4.1 Eloquent Models

Create Eloquent models with proper relationships:

```php
<?php
// app/Modules/ECommerce/Infrastructure/Models/Product.php

namespace App\Modules\ECommerce\Infrastructure\Models;

use App\Modules\Shared\Infrastructure\Models\BaseModel;

class Product extends BaseModel
{
    protected $table = 'ecommerce_products';
    
    protected $fillable = [
        'business_id',
        'vendor_id',
        'sku',
        'name_en',
        'name_ar',
        'price',
        // ... other fields
    ];
    
    protected $casts = [
        'price' => 'decimal:4',
        'dimensions' => 'array',
        'tags' => 'array',
        'featured' => 'boolean',
    ];
    
    // Relationships
    public function business()
    {
        return $this->belongsTo(\App\Modules\BusinessDirectory\Infrastructure\Models\Business::class);
    }
    
    public function vendor()
    {
        return $this->belongsTo(Vendor::class);
    }
    
    // Scopes
    public function scopeActive($query)
    {
        return $query->where('status', 'active');
    }
}
```

### 4.2 Repository Implementation

Implement repository pattern:

```php
<?php
// app/Modules/ECommerce/Infrastructure/Repositories/ProductRepository.php

namespace App\Modules\ECommerce\Infrastructure\Repositories;

use App\Modules\ECommerce\Domain\Contracts\ProductRepositoryInterface;
use App\Modules\ECommerce\Infrastructure\Models\Product as ProductModel;

class ProductRepository implements ProductRepositoryInterface
{
    public function findById(int $id): ?Product
    {
        $model = ProductModel::with(['vendor', 'category'])->find($id);
        return $model ? $this->toDomainEntity($model) : null;
    }
    
    // Implementation methods
}
```

### 4.3 Jobs and Queues

Create background jobs for heavy operations:

```php
<?php
// app/Modules/ECommerce/Infrastructure/Jobs/ProcessOrderPaymentJob.php

namespace App\Modules\ECommerce\Infrastructure\Jobs;

use Illuminate\Bus\Queueable;
use Illuminate\Queue\SerializesModels;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Contracts\Queue\ShouldQueue;

class ProcessOrderPaymentJob implements ShouldQueue
{
    use InteractsWithQueue, Queueable, SerializesModels;
    
    public function __construct(
        private int $orderId,
        private array $paymentDetails
    ) {}
    
    public function handle(): void
    {
        // Payment processing logic
    }
}
```

## Phase 5: Presentation Layer Implementation (Week 9-10)

### 5.1 API Controllers

Create RESTful API controllers:

```php
<?php
// app/Modules/ECommerce/Presentation/Controllers/ProductController.php

namespace App\Modules\ECommerce\Presentation\Controllers;

use App\Http\Controllers\Controller;
use App\Modules\ECommerce\Application\Services\ProductService;
use App\Modules\ECommerce\Presentation\Requests\CreateProductRequest;
use App\Modules\ECommerce\Presentation\Resources\ProductResource;

class ProductController extends Controller
{
    public function __construct(
        private ProductService $productService
    ) {}
    
    public function index(Request $request)
    {
        $products = $this->productService->getProducts($request->validated());
        return ProductResource::collection($products);
    }
    
    public function store(CreateProductRequest $request)
    {
        $product = $this->productService->createProduct($request->validated());
        return new ProductResource($product);
    }
}
```

### 5.2 Form Requests

Create validation requests:

```php
<?php
// app/Modules/ECommerce/Presentation/Requests/CreateProductRequest.php

namespace App\Modules\ECommerce\Presentation\Requests;

use Illuminate\Foundation\Http\FormRequest;

class CreateProductRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'sku' => 'required|string|max:100|unique:ecommerce_products,sku',
            'name_en' => 'required|string|max:255',
            'name_ar' => 'nullable|string|max:255',
            'price' => 'required|numeric|min:0',
            'vendor_id' => 'required|exists:ecommerce_vendors,id',
            // ... other validation rules
        ];
    }
    
    public function authorize(): bool
    {
        return $this->user()->can('create', Product::class);
    }
}
```

### 5.3 API Resources

Create API response transformers:

```php
<?php
// app/Modules/ECommerce/Presentation/Resources/ProductResource.php

namespace App\Modules\ECommerce\Presentation\Resources;

use Illuminate\Http\Resources\Json\JsonResource;

class ProductResource extends JsonResource
{
    public function toArray($request): array
    {
        return [
            'id' => $this->id,
            'sku' => $this->sku,
            'name_en' => $this->name_en,
            'name_ar' => $this->name_ar,
            'price' => $this->price,
            'formatted_price' => $this->formatted_price,
            'status' => $this->status,
            'featured' => $this->featured,
            'stock_quantity' => $this->stock_quantity,
            'vendor' => new VendorResource($this->whenLoaded('vendor')),
            'category' => new CategoryResource($this->whenLoaded('category')),
            'images' => ProductImageResource::collection($this->whenLoaded('images')),
            'created_at' => $this->created_at,
        ];
    }
}
```

## Phase 6: Frontend Implementation (Week 11-12)

### 6.1 React Component Structure

```typescript
// resources/js/pages/ecommerce/products/ProductList.tsx

import React, { useState, useEffect } from 'react';
import { useTranslation } from 'react-i18next';
import { usePage } from '@inertiajs/react';
import ProductCard from '@/components/ecommerce/ProductCard';
import ProductFilters from '@/components/ecommerce/ProductFilters';
import Pagination from '@/components/ui/Pagination';

interface ProductListProps {
    products: PaginatedResponse<Product>;
    categories: Category[];
    vendors: Vendor[];
    filters: ProductFilters;
}

export default function ProductList({ 
    products, 
    categories, 
    vendors, 
    filters 
}: ProductListProps) {
    const { t } = useTranslation('ecommerce');
    const [activeFilters, setActiveFilters] = useState(filters);
    
    return (
        <div className="container mx-auto px-4 py-8">
            <div className="flex flex-col lg:flex-row gap-8">
                {/* Filters Sidebar */}
                <div className="lg:w-1/4">
                    <ProductFilters
                        categories={categories}
                        vendors={vendors}
                        filters={activeFilters}
                        onFiltersChange={setActiveFilters}
                    />
                </div>
                
                {/* Products Grid */}
                <div className="lg:w-3/4">
                    <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4 gap-6">
                        {products.data.map((product) => (
                            <ProductCard
                                key={product.id}
                                product={product}
                                onAddToCart={handleAddToCart}
                            />
                        ))}
                    </div>
                    
                    {products.meta.last_page > 1 && (
                        <div className="mt-8">
                            <Pagination
                                current={products.meta.current_page}
                                total={products.meta.last_page}
                                onPageChange={handlePageChange}
                            />
                        </div>
                    )}
                </div>
            </div>
        </div>
    );
}
```

### 6.2 State Management

```typescript
// resources/js/stores/ecommerceStore.ts

import { create } from 'zustand';
import { persist } from 'zustand/middleware';

interface Cart {
    id: string;
    items: CartItem[];
    total: number;
    currency: string;
}

interface ECommerceStore {
    cart: Cart | null;
    wishlist: Product[];
    
    // Cart actions
    addToCart: (product: Product, quantity: number) => Promise<void>;
    removeFromCart: (itemId: string) => Promise<void>;
    updateCartItemQuantity: (itemId: string, quantity: number) => Promise<void>;
    clearCart: () => Promise<void>;
    
    // Wishlist actions
    addToWishlist: (product: Product) => void;
    removeFromWishlist: (productId: string) => void;
    
    // API actions
    fetchCart: () => Promise<void>;
    checkout: (orderData: CheckoutData) => Promise<Order>;
}

export const useECommerceStore = create<ECommerceStore>()(
    persist(
        (set, get) => ({
            cart: null,
            wishlist: [],
            
            addToCart: async (product, quantity) => {
                try {
                    const response = await fetch('/api/v1/ecommerce/cart/items', {
                        method: 'POST',
                        headers: {
                            'Content-Type': 'application/json',
                            'Authorization': `Bearer ${getAuthToken()}`,
                        },
                        body: JSON.stringify({
                            product_id: product.id,
                            quantity,
                        }),
                    });
                    
                    const result = await response.json();
                    if (result.success) {
                        set({ cart: result.data });
                    }
                } catch (error) {
                    console.error('Failed to add to cart:', error);
                }
            },
            
            // Other implementations...
        }),
        {
            name: 'ecommerce-store',
            partialize: (state) => ({ wishlist: state.wishlist }),
        }
    )
);
```

## Phase 7: Integration Implementation (Week 13-14)

### 7.1 Warehouses Integration

```php
<?php
// app/Modules/ECommerce/Application/Services/InventoryIntegrationService.php

namespace App\Modules\ECommerce\Application\Services;

use App\Modules\Warehouses\Domain\Contracts\InventoryRepositoryInterface;
use App\Modules\ECommerce\Domain\Entities\Product;

class InventoryIntegrationService
{
    public function __construct(
        private InventoryRepositoryInterface $inventoryRepository
    ) {}
    
    public function syncProductInventory(Product $product): void
    {
        $inventoryItem = $this->inventoryRepository->findByProductId($product->getId());
        
        if ($inventoryItem) {
            // Update product stock based on inventory
            $product->adjustStock($inventoryItem->getAvailableQuantity() - $product->getStockQuantity());
        }
    }
    
    public function createInventoryItem(Product $product): void
    {
        $inventoryItem = new InventoryItem(
            productId: $product->getId(),
            warehouseId: $product->getWarehouseId(),
            quantity: $product->getStockQuantity(),
            unitCost: $product->getCostPrice()
        );
        
        $this->inventoryRepository->save($inventoryItem);
    }
}
```

### 7.2 Accounting Integration

```php
<?php
// app/Modules/ECommerce/Application/Services/AccountingIntegrationService.php

namespace App\Modules\ECommerce\Application\Services;

use App\Modules\Accounting\Domain\Contracts\JournalEntryRepositoryInterface;
use App\Modules\ECommerce\Domain\Entities\Order;

class AccountingIntegrationService
{
    public function __construct(
        private JournalEntryRepositoryInterface $journalEntryRepository
    ) {}
    
    public function createOrderJournalEntries(Order $order): void
    {
        // Create sales journal entry
        $salesEntry = new JournalEntry(
            businessId: $order->getBusinessId(),
            description: "Sale - Order {$order->getOrderNumber()}",
            date: $order->getPlacedAt()
        );
        
        // Debit: Accounts Receivable
        $salesEntry->addLineItem(
            accountId: $this->getAccountsReceivableAccountId(),
            debitAmount: $order->getTotalAmount(),
            description: "Customer payment due"
        );
        
        // Credit: Sales Revenue
        $salesEntry->addLineItem(
            accountId: $this->getSalesRevenueAccountId(),
            creditAmount: $order->getSubtotal(),
            description: "Product sales revenue"
        );
        
        // Credit: Sales Tax Payable (if applicable)
        if (!$order->getTaxAmount()->isZero()) {
            $salesEntry->addLineItem(
                accountId: $this->getSalesTaxPayableAccountId(),
                creditAmount: $order->getTaxAmount(),
                description: "Sales tax collected"
            );
        }
        
        $this->journalEntryRepository->save($salesEntry);
    }
    
    public function createVendorCommissionEntries(VendorCommission $commission): void
    {
        // Create commission expense entry
        $commissionEntry = new JournalEntry(
            businessId: $commission->getVendor()->getBusinessId(),
            description: "Vendor commission - Order {$commission->getOrder()->getOrderNumber()}",
            date: now()
        );
        
        // Debit: Commission Expense
        $commissionEntry->addLineItem(
            accountId: $this->getCommissionExpenseAccountId(),
            debitAmount: $commission->getCommissionAmount(),
            description: "Vendor commission expense"
        );
        
        // Credit: Commission Payable
        $commissionEntry->addLineItem(
            accountId: $this->getCommissionPayableAccountId(),
            creditAmount: $commission->getCommissionAmount(),
            description: "Commission due to vendor"
        );
        
        $this->journalEntryRepository->save($commissionEntry);
    }
}
```

### 7.3 BusinessDirectory Integration

```php
<?php
// app/Modules/ECommerce/Application/Services/BusinessDirectoryIntegrationService.php

namespace App\Modules\ECommerce\Application\Services;

use App\Modules\BusinessDirectory\Domain\Contracts\BusinessRepositoryInterface;
use App\Modules\ECommerce\Domain\Entities\Vendor;

class BusinessDirectoryIntegrationService
{
    public function __construct(
        private BusinessRepositoryInterface $businessRepository
    ) {}
    
    public function setupVendorStorefront(Vendor $vendor): void
    {
        $business = $this->businessRepository->findById($vendor->getBusinessId());
        
        // Update business profile for e-commerce
        $business->enableECommerce();
        $business->setStorefrontUrl("/store/{$vendor->getVendorCode()}");
        
        // Create business categories if needed
        if (!$business->hasCategories()) {
            $this->createDefaultBusinessCategories($business);
        }
        
        $this->businessRepository->save($business);
    }
    
    public function syncVendorRating(Vendor $vendor): void
    {
        $business = $this->businessRepository->findById($vendor->getBusinessId());
        
        // Update business rating based on vendor performance
        $business->updateRating($vendor->getRating());
        $business->setReviewsCount($vendor->getTotalOrders());
        
        $this->businessRepository->save($business);
    }
}
```

## Phase 8: Testing Implementation (Week 15-16)

### 8.1 Unit Tests

```php
<?php
// tests/Unit/Modules/ECommerce/Domain/Entities/ProductTest.php

namespace Tests\Unit\Modules\ECommerce\Domain\Entities;

use Tests\TestCase;
use App\Modules\ECommerce\Domain\Entities\Product;
use App\Modules\ECommerce\Domain\ValueObjects\SKU;
use App\Modules\ECommerce\Domain\ValueObjects\Money;

class ProductTest extends TestCase
{
    public function test_can_create_product(): void
    {
        $product = new Product(
            businessId: 1,
            vendorId: 1,
            sku: new SKU('TEST-001'),
            nameEn: 'Test Product',
            price: new Money(99.99)
        );
        
        $this->assertEquals(1, $product->getBusinessId());
        $this->assertEquals(1, $product->getVendorId());
        $this->assertEquals('TEST-001', $product->getSku()->getValue());
        $this->assertEquals('Test Product', $product->getNameEn());
        $this->assertEquals(99.99, $product->getPrice()->getAmount());
    }
    
    public function test_can_adjust_stock(): void
    {
        $product = new Product(
            businessId: 1,
            vendorId: 1,
            sku: new SKU('TEST-001'),
            nameEn: 'Test Product',
            price: new Money(99.99)
        );
        
        $product->setStockQuantity(10);
        $product->adjustStock(5);
        
        $this->assertEquals(15, $product->getStockQuantity());
        
        $product->adjustStock(-3);
        $this->assertEquals(12, $product->getStockQuantity());
    }
    
    public function test_cannot_adjust_stock_below_zero(): void
    {
        $this->expectException(InsufficientStockException::class);
        
        $product = new Product(
            businessId: 1,
            vendorId: 1,
            sku: new SKU('TEST-001'),
            nameEn: 'Test Product',
            price: new Money(99.99)
        );
        
        $product->setStockQuantity(5);
        $product->adjustStock(-10);
    }
}
```

### 8.2 Integration Tests

```php
<?php
// tests/Feature/Modules/ECommerce/ProductManagementTest.php

namespace Tests\Feature\Modules\ECommerce;

use Tests\TestCase;
use App\Modules\ECommerce\Infrastructure\Models\Product;
use App\Modules\ECommerce\Infrastructure\Models\Vendor;
use App\Modules\BusinessDirectory\Infrastructure\Models\Business;

class ProductManagementTest extends TestCase
{
    use RefreshDatabase;
    
    public function test_vendor_can_create_product(): void
    {
        $user = User::factory()->create();
        $business = Business::factory()->create();
        $vendor = Vendor::factory()->create([
            'business_id' => $business->id,
            'user_id' => $user->id,
            'status' => 'active'
        ]);
        
        $response = $this->actingAs($user)
            ->postJson('/api/v1/ecommerce/products', [
                'sku' => 'TEST-001',
                'name_en' => 'Test Product',
                'price' => 99.99,
                'stock_quantity' => 10,
                'status' => 'active'
            ]);
        
        $response->assertStatus(201)
            ->assertJson([
                'success' => true,
                'data' => [
                    'sku' => 'TEST-001',
                    'name_en' => 'Test Product',
                    'price' => 99.99
                ]
            ]);
        
        $this->assertDatabaseHas('ecommerce_products', [
            'sku' => 'TEST-001',
            'vendor_id' => $vendor->id
        ]);
    }
    
    public function test_customer_can_add_product_to_cart(): void
    {
        $customer = User::factory()->create();
        $product = Product::factory()->create([
            'status' => 'active',
            'stock_quantity' => 10
        ]);
        
        $response = $this->actingAs($customer)
            ->postJson('/api/v1/ecommerce/cart/items', [
                'product_id' => $product->id,
                'quantity' => 2
            ]);
        
        $response->assertStatus(200)
            ->assertJson([
                'success' => true,
                'data' => [
                    'items' => [
                        [
                            'product_id' => $product->id,
                            'quantity' => 2
                        ]
                    ]
                ]
            ]);
        
        $this->assertDatabaseHas('ecommerce_cart_items', [
            'product_id' => $product->id,
            'quantity' => 2
        ]);
    }
}
```

### 8.3 Frontend Tests

```typescript
// resources/js/__tests__/components/ecommerce/ProductCard.test.tsx

import React from 'react';
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { vi } from 'vitest';
import ProductCard from '@/components/ecommerce/ProductCard';
import { useECommerceStore } from '@/stores/ecommerceStore';

// Mock the store
vi.mock('@/stores/ecommerceStore');

const mockProduct = {
    id: 1,
    sku: 'TEST-001',
    name_en: 'Test Product',
    price: 99.99,
    formatted_price: '99.99 EGP',
    stock_quantity: 10,
    primary_image: 'https://example.com/image.jpg',
    vendor: {
        id: 1,
        display_name: 'Test Vendor'
    }
};

describe('ProductCard', () => {
    const mockAddToCart = vi.fn();
    
    beforeEach(() => {
        (useECommerceStore as any).mockReturnValue({
            addToCart: mockAddToCart
        });
    });
    
    afterEach(() => {
        vi.clearAllMocks();
    });
    
    test('renders product information correctly', () => {
        render(<ProductCard product={mockProduct} />);
        
        expect(screen.getByText('Test Product')).toBeInTheDocument();
        expect(screen.getByText('99.99 EGP')).toBeInTheDocument();
        expect(screen.getByText('Test Vendor')).toBeInTheDocument();
        expect(screen.getByAltText('Test Product')).toHaveAttribute('src', 'https://example.com/image.jpg');
    });
    
    test('calls addToCart when add to cart button is clicked', async () => {
        render(<ProductCard product={mockProduct} />);
        
        const addToCartButton = screen.getByText('Add to Cart');
        fireEvent.click(addToCartButton);
        
        await waitFor(() => {
            expect(mockAddToCart).toHaveBeenCalledWith(mockProduct, 1);
        });
    });
    
    test('shows out of stock when stock quantity is zero', () => {
        const outOfStockProduct = { ...mockProduct, stock_quantity: 0 };
        render(<ProductCard product={outOfStockProduct} />);
        
        expect(screen.getByText('Out of Stock')).toBeInTheDocument();
        expect(screen.getByRole('button', { name: /add to cart/i })).toBeDisabled();
    });
});
```

## Phase 9: Performance & Security (Week 17-18)

### 9.1 Performance Optimization

```php
<?php
// app/Modules/ECommerce/Infrastructure/Services/ProductCacheService.php

namespace App\Modules\ECommerce\Infrastructure\Services;

use Illuminate\Support\Facades\Cache;
use App\Modules\ECommerce\Domain\Entities\Product;

class ProductCacheService
{
    private const CACHE_PREFIX = 'ecommerce:product:';
    private const CACHE_TTL = 3600; // 1 hour
    
    public function getProduct(int $productId): ?Product
    {
        $cacheKey = self::CACHE_PREFIX . $productId;
        
        return Cache::remember($cacheKey, self::CACHE_TTL, function () use ($productId) {
            return app(ProductRepositoryInterface::class)->findById($productId);
        });
    }
    
    public function invalidateProduct(int $productId): void
    {
        Cache::forget(self::CACHE_PREFIX . $productId);
        
        // Also invalidate related caches
        Cache::forget('ecommerce:featured_products');
        Cache::forget('ecommerce:vendor_products:' . $productId);
    }
    
    public function getFeaturedProducts(): Collection
    {
        return Cache::remember('ecommerce:featured_products', self::CACHE_TTL, function () {
            return app(ProductRepositoryInterface::class)->getFeaturedProducts(12);
        });
    }
}
```

### 9.2 Security Implementation

```php
<?php
// app/Modules/ECommerce/Presentation/Middleware/VendorAccessMiddleware.php

namespace App\Modules\ECommerce\Presentation\Middleware;

use Closure;
use Illuminate\Http\Request;

class VendorAccessMiddleware
{
    public function handle(Request $request, Closure $next)
    {
        $user = $request->user();
        
        if (!$user) {
            return response()->json(['error' => 'Unauthorized'], 401);
        }
        
        // Check if user is a vendor
        $vendor = $user->vendor;
        if (!$vendor || $vendor->status !== 'active') {
            return response()->json(['error' => 'Vendor access required'], 403);
        }
        
        // Add vendor to request for easy access
        $request->merge(['vendor' => $vendor]);
        
        return $next($request);
    }
}
```

## Development Workflow Best Practices

### 1. Git Workflow

```bash
# Feature branch workflow
git checkout develop
git pull origin develop
git checkout -b feature/ecommerce-product-catalog
# Make changes
git add .
git commit -m "feat: implement product catalog with filtering"
git push origin feature/ecommerce-product-catalog
# Create pull request
```

### 2. Code Quality

```bash
# Run code analysis
composer run phpstan
composer run phpcs

# Run tests
php artisan test
npm run test

# Check test coverage
php artisan test --coverage
npm run test:coverage
```

### 3. Database Management

```bash
# Create and run migrations
php artisan make:migration create_ecommerce_products_table
php artisan migrate

# Seed data for testing
php artisan db:seed --class=ECommerceSeeder

# Fresh install with sample data
php artisan migrate:fresh --seed
```

### 4. Asset Building

```bash
# Development
npm run dev

# Production
npm run build

# Watch for changes
npm run dev --watch
```

This comprehensive development workflow and implementation guide provides a structured approach to building the multi-vendor e-commerce module while maintaining code quality, performance, and integration with existing OmarStartKit modules.