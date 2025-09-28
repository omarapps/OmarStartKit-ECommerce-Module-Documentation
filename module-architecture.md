# E-Commerce Module Architecture

## Overview
This document outlines the detailed architecture for the multi-vendor e-commerce module within the OmarStartKit ecosystem, following Domain-Driven Design (DDD) principles and integrating seamlessly with existing modules.

## Module Structure

### Directory Organization
```
app/Modules/ECommerce/
├── Application/
│   ├── Commands/           # Command handlers for business operations
│   ├── DTOs/              # Data transfer objects
│   ├── Events/            # Domain events
│   ├── Listeners/         # Event listeners
│   ├── Queries/           # Query handlers (CQRS pattern)
│   └── Services/          # Application services
├── Domain/
│   ├── Contracts/         # Repository interfaces and domain contracts
│   ├── Entities/          # Domain entities
│   ├── ValueObjects/      # Value objects
│   └── Enums/            # Domain enums
├── Infrastructure/
│   ├── Models/           # Eloquent models
│   ├── Repositories/     # Repository implementations
│   ├── Providers/        # Service providers
│   ├── Jobs/             # Background jobs
│   └── Services/         # Infrastructure services
└── Presentation/
    ├── Controllers/      # HTTP controllers
    ├── Requests/         # Form requests
    ├── Resources/        # API resources
    └── Middleware/       # Module-specific middleware
```

## Domain Layer

### Core Entities

#### Product Entity
```php
<?php

namespace App\Modules\ECommerce\Domain\Entities;

use App\Modules\ECommerce\Domain\ValueObjects\Money;
use App\Modules\ECommerce\Domain\ValueObjects\Dimensions;
use App\Modules\ECommerce\Domain\ValueObjects\Weight;
use App\Modules\ECommerce\Domain\ValueObjects\ProductStatus;
use App\Modules\ECommerce\Domain\ValueObjects\SKU;
use Illuminate\Support\Collection;

class Product
{
    private int $id;
    private int $businessId;
    private int $vendorId;
    private SKU $sku;
    private string $nameEn;
    private ?string $nameAr;
    private string $slug;
    private ?string $descriptionEn;
    private ?string $descriptionAr;
    private Money $price;
    private ?Money $comparePrice;
    private ?Money $costPrice;
    private ?Weight $weight;
    private ?Dimensions $dimensions;
    private ProductStatus $status;
    private bool $featured;
    private int $stockQuantity;
    private int $stockThreshold;
    private bool $trackInventory;
    private bool $allowBackorders;
    private Collection $images;
    private Collection $attributes;
    private Collection $categories;
    
    public function __construct(
        int $businessId,
        int $vendorId,
        SKU $sku,
        string $nameEn,
        Money $price
    ) {
        $this->businessId = $businessId;
        $this->vendorId = $vendorId;
        $this->sku = $sku;
        $this->nameEn = $nameEn;
        $this->price = $price;
        $this->status = ProductStatus::DRAFT;
        $this->featured = false;
        $this->stockQuantity = 0;
        $this->stockThreshold = 5;
        $this->trackInventory = true;
        $this->allowBackorders = false;
        $this->images = collect();
        $this->attributes = collect();
        $this->categories = collect();
    }
    
    public function updatePrice(Money $price): void
    {
        $this->price = $price;
        // Domain event: ProductPriceUpdated
    }
    
    public function adjustStock(int $quantity, string $reason = ''): void
    {
        if (!$this->trackInventory) {
            return;
        }
        
        $oldQuantity = $this->stockQuantity;
        $this->stockQuantity += $quantity;
        
        if ($this->stockQuantity < 0) {
            throw new InsufficientStockException();
        }
        
        // Domain event: StockAdjusted
    }
    
    public function reserveStock(int $quantity): void
    {
        if (!$this->isAvailable($quantity)) {
            throw new InsufficientStockException();
        }
        
        $this->stockQuantity -= $quantity;
        // Domain event: StockReserved
    }
    
    public function isAvailable(int $quantity = 1): bool
    {
        if (!$this->trackInventory) {
            return true;
        }
        
        return $this->stockQuantity >= $quantity || $this->allowBackorders;
    }
    
    public function isLowStock(): bool
    {
        return $this->trackInventory && $this->stockQuantity <= $this->stockThreshold;
    }
    
    public function activate(): void
    {
        $this->status = ProductStatus::ACTIVE;
        // Domain event: ProductActivated
    }
    
    public function deactivate(): void
    {
        $this->status = ProductStatus::INACTIVE;
        // Domain event: ProductDeactivated
    }
    
    // Getters
    public function getId(): ?int { return $this->id; }
    public function getBusinessId(): int { return $this->businessId; }
    public function getVendorId(): int { return $this->vendorId; }
    public function getSku(): SKU { return $this->sku; }
    public function getNameEn(): string { return $this->nameEn; }
    public function getPrice(): Money { return $this->price; }
    public function getStatus(): ProductStatus { return $this->status; }
    public function getStockQuantity(): int { return $this->stockQuantity; }
}
```

#### Order Entity
```php
<?php

namespace App\Modules\ECommerce\Domain\Entities;

use App\Modules\ECommerce\Domain\ValueObjects\OrderNumber;
use App\Modules\ECommerce\Domain\ValueObjects\Money;
use App\Modules\ECommerce\Domain\ValueObjects\Address;
use App\Modules\ECommerce\Domain\ValueObjects\OrderStatus;
use App\Modules\ECommerce\Domain\ValueObjects\PaymentStatus;
use App\Modules\ECommerce\Domain\ValueObjects\FulfillmentStatus;
use Illuminate\Support\Collection;
use Carbon\Carbon;

class Order
{
    private int $id;
    private ?int $businessId;
    private int $customerId;
    private OrderNumber $orderNumber;
    private OrderStatus $status;
    private PaymentStatus $paymentStatus;
    private FulfillmentStatus $fulfillmentStatus;
    private Money $subtotal;
    private Money $taxAmount;
    private Money $shippingAmount;
    private Money $discountAmount;
    private Money $totalAmount;
    private string $currencyCode;
    private Address $billingAddress;
    private Address $shippingAddress;
    private Collection $items;
    private Collection $transactions;
    private ?string $couponCode;
    private ?string $notes;
    private Carbon $placedAt;
    private ?Carbon $confirmedAt;
    private ?Carbon $shippedAt;
    private ?Carbon $deliveredAt;
    
    public function __construct(
        ?int $businessId,
        int $customerId,
        Address $billingAddress,
        Address $shippingAddress,
        string $currencyCode = 'EGP'
    ) {
        $this->businessId = $businessId;
        $this->customerId = $customerId;
        $this->orderNumber = OrderNumber::generate();
        $this->status = OrderStatus::PENDING;
        $this->paymentStatus = PaymentStatus::PENDING;
        $this->fulfillmentStatus = FulfillmentStatus::PENDING;
        $this->billingAddress = $billingAddress;
        $this->shippingAddress = $shippingAddress;
        $this->currencyCode = $currencyCode;
        $this->items = collect();
        $this->transactions = collect();
        $this->subtotal = new Money(0, $currencyCode);
        $this->taxAmount = new Money(0, $currencyCode);
        $this->shippingAmount = new Money(0, $currencyCode);
        $this->discountAmount = new Money(0, $currencyCode);
        $this->totalAmount = new Money(0, $currencyCode);
        $this->placedAt = now();
    }
    
    public function addItem(OrderItem $item): void
    {
        $this->items->push($item);
        $this->recalculateTotal();
        // Domain event: OrderItemAdded
    }
    
    public function removeItem(int $itemId): void
    {
        $this->items = $this->items->reject(fn($item) => $item->getId() === $itemId);
        $this->recalculateTotal();
        // Domain event: OrderItemRemoved
    }
    
    public function applyCoupon(string $couponCode, Money $discountAmount): void
    {
        $this->couponCode = $couponCode;
        $this->discountAmount = $discountAmount;
        $this->recalculateTotal();
        // Domain event: CouponApplied
    }
    
    public function confirm(): void
    {
        if (!$this->status->isPending()) {
            throw new InvalidOrderStatusException('Order cannot be confirmed');
        }
        
        $this->status = OrderStatus::CONFIRMED;
        $this->confirmedAt = now();
        // Domain event: OrderConfirmed
    }
    
    public function ship(string $trackingNumber = null): void
    {
        if (!$this->canBeShipped()) {
            throw new InvalidOrderStatusException('Order cannot be shipped');
        }
        
        $this->fulfillmentStatus = FulfillmentStatus::SHIPPED;
        $this->shippedAt = now();
        
        // Update individual items
        foreach ($this->items as $item) {
            $item->ship($trackingNumber);
        }
        
        // Domain event: OrderShipped
    }
    
    public function deliver(): void
    {
        if (!$this->fulfillmentStatus->isShipped()) {
            throw new InvalidOrderStatusException('Order must be shipped before delivery');
        }
        
        $this->fulfillmentStatus = FulfillmentStatus::DELIVERED;
        $this->deliveredAt = now();
        // Domain event: OrderDelivered
    }
    
    public function cancel(string $reason = ''): void
    {
        if (!$this->canBeCancelled()) {
            throw new InvalidOrderStatusException('Order cannot be cancelled');
        }
        
        $this->status = OrderStatus::CANCELLED;
        
        // Release reserved stock
        foreach ($this->items as $item) {
            $item->getProduct()->adjustStock($item->getQuantity(), 'Order cancelled');
        }
        
        // Domain event: OrderCancelled
    }
    
    private function recalculateTotal(): void
    {
        $this->subtotal = new Money(
            $this->items->sum(fn($item) => $item->getTotalPrice()->getAmount()),
            $this->currencyCode
        );
        
        $this->totalAmount = $this->subtotal
            ->add($this->taxAmount)
            ->add($this->shippingAmount)
            ->subtract($this->discountAmount);
    }
    
    private function canBeShipped(): bool
    {
        return $this->status->isConfirmed() && 
               $this->paymentStatus->isPaid() && 
               $this->fulfillmentStatus->isPending();
    }
    
    private function canBeCancelled(): bool
    {
        return in_array($this->status, [OrderStatus::PENDING, OrderStatus::CONFIRMED]) &&
               !$this->fulfillmentStatus->isShipped();
    }
    
    // Getters
    public function getId(): ?int { return $this->id; }
    public function getOrderNumber(): OrderNumber { return $this->orderNumber; }
    public function getStatus(): OrderStatus { return $this->status; }
    public function getPaymentStatus(): PaymentStatus { return $this->paymentStatus; }
    public function getTotalAmount(): Money { return $this->totalAmount; }
    public function getItems(): Collection { return $this->items; }
    public function getCustomerId(): int { return $this->customerId; }
}
```

#### Vendor Entity
```php
<?php

namespace App\Modules\ECommerce\Domain\Entities;

use App\Modules\ECommerce\Domain\ValueObjects\VendorCode;
use App\Modules\ECommerce\Domain\ValueObjects\CommissionRate;
use App\Modules\ECommerce\Domain\ValueObjects\VendorStatus;
use App\Modules\ECommerce\Domain\ValueObjects\VerificationStatus;
use App\Modules\ECommerce\Domain\ValueObjects\Money;

class Vendor
{
    private int $id;
    private int $businessId;
    private int $userId;
    private VendorCode $vendorCode;
    private string $displayName;
    private VendorStatus $status;
    private VerificationStatus $verificationStatus;
    private CommissionRate $commissionRate;
    private Money $totalSales;
    private int $totalOrders;
    private float $rating;
    private ?string $bankAccountNumber;
    private ?string $paymentMethod;
    
    public function __construct(
        int $businessId,
        int $userId,
        string $displayName,
        CommissionRate $commissionRate
    ) {
        $this->businessId = $businessId;
        $this->userId = $userId;
        $this->displayName = $displayName;
        $this->vendorCode = VendorCode::generate();
        $this->status = VendorStatus::PENDING;
        $this->verificationStatus = VerificationStatus::UNVERIFIED;
        $this->commissionRate = $commissionRate;
        $this->totalSales = new Money(0);
        $this->totalOrders = 0;
        $this->rating = 0.0;
    }
    
    public function approve(): void
    {
        if (!$this->status->isPending()) {
            throw new InvalidVendorStatusException('Vendor must be pending to approve');
        }
        
        $this->status = VendorStatus::ACTIVE;
        // Domain event: VendorApproved
    }
    
    public function suspend(string $reason = ''): void
    {
        if (!$this->status->isActive()) {
            throw new InvalidVendorStatusException('Only active vendors can be suspended');
        }
        
        $this->status = VendorStatus::SUSPENDED;
        // Domain event: VendorSuspended
    }
    
    public function verify(): void
    {
        $this->verificationStatus = VerificationStatus::VERIFIED;
        // Domain event: VendorVerified
    }
    
    public function updateCommissionRate(CommissionRate $rate): void
    {
        $oldRate = $this->commissionRate;
        $this->commissionRate = $rate;
        // Domain event: CommissionRateUpdated
    }
    
    public function recordSale(Money $amount): void
    {
        $this->totalSales = $this->totalSales->add($amount);
        $this->totalOrders++;
        // Domain event: VendorSaleRecorded
    }
    
    public function updateRating(float $rating): void
    {
        if ($rating < 0 || $rating > 5) {
            throw new InvalidRatingException('Rating must be between 0 and 5');
        }
        
        $this->rating = $rating;
        // Domain event: VendorRatingUpdated
    }
    
    // Getters
    public function getId(): ?int { return $this->id; }
    public function getBusinessId(): int { return $this->businessId; }
    public function getUserId(): int { return $this->userId; }
    public function getVendorCode(): VendorCode { return $this->vendorCode; }
    public function getDisplayName(): string { return $this->displayName; }
    public function getStatus(): VendorStatus { return $this->status; }
    public function getCommissionRate(): CommissionRate { return $this->commissionRate; }
    public function getTotalSales(): Money { return $this->totalSales; }
    public function getRating(): float { return $this->rating; }
}
```

### Value Objects

#### Money Value Object
```php
<?php

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
    
    public function add(Money $other): Money
    {
        $this->ensureSameCurrency($other);
        return new Money($this->amount + $other->amount, $this->currency);
    }
    
    public function subtract(Money $other): Money
    {
        $this->ensureSameCurrency($other);
        $result = $this->amount - $other->amount;
        
        if ($result < 0) {
            throw new InvalidArgumentException('Subtraction would result in negative amount');
        }
        
        return new Money($result, $this->currency);
    }
    
    public function multiply(float $factor): Money
    {
        if ($factor < 0) {
            throw new InvalidArgumentException('Factor cannot be negative');
        }
        
        return new Money($this->amount * $factor, $this->currency);
    }
    
    public function percentage(float $percent): Money
    {
        return $this->multiply($percent / 100);
    }
    
    public function equals(Money $other): bool
    {
        return $this->amount === $other->amount && $this->currency === $other->currency;
    }
    
    public function isGreaterThan(Money $other): bool
    {
        $this->ensureSameCurrency($other);
        return $this->amount > $other->amount;
    }
    
    public function isZero(): bool
    {
        return $this->amount === 0.0;
    }
    
    private function ensureSameCurrency(Money $other): void
    {
        if ($this->currency !== $other->currency) {
            throw new InvalidArgumentException('Currency mismatch');
        }
    }
    
    public function getAmount(): float
    {
        return $this->amount;
    }
    
    public function getCurrency(): string
    {
        return $this->currency;
    }
    
    public function format(): string
    {
        return number_format($this->amount, 2) . ' ' . $this->currency;
    }
    
    public function __toString(): string
    {
        return $this->format();
    }
}
```

#### SKU Value Object
```php
<?php

namespace App\Modules\ECommerce\Domain\ValueObjects;

class SKU
{
    private string $value;
    
    public function __construct(string $value)
    {
        if (empty($value) || strlen($value) > 100) {
            throw new InvalidArgumentException('SKU must be between 1 and 100 characters');
        }
        
        if (!preg_match('/^[A-Z0-9\-_]+$/', $value)) {
            throw new InvalidArgumentException('SKU can only contain uppercase letters, numbers, hyphens, and underscores');
        }
        
        $this->value = $value;
    }
    
    public static function generate(string $prefix = 'PROD'): self
    {
        return new self($prefix . '-' . strtoupper(uniqid()));
    }
    
    public function getValue(): string
    {
        return $this->value;
    }
    
    public function equals(SKU $other): bool
    {
        return $this->value === $other->value;
    }
    
    public function __toString(): string
    {
        return $this->value;
    }
}
```

#### OrderNumber Value Object
```php
<?php

namespace App\Modules\ECommerce\Domain\ValueObjects;

class OrderNumber
{
    private string $value;
    
    public function __construct(string $value)
    {
        if (empty($value) || strlen($value) > 100) {
            throw new InvalidArgumentException('Order number must be between 1 and 100 characters');
        }
        
        $this->value = $value;
    }
    
    public static function generate(string $prefix = 'ORD'): self
    {
        $year = date('Y');
        $sequence = str_pad(random_int(1, 999999), 6, '0', STR_PAD_LEFT);
        return new self("{$prefix}-{$year}-{$sequence}");
    }
    
    public function getValue(): string
    {
        return $this->value;
    }
    
    public function equals(OrderNumber $other): bool
    {
        return $this->value === $other->value;
    }
    
    public function __toString(): string
    {
        return $this->value;
    }
}
```

### Domain Enums

#### Product Status Enum
```php
<?php

namespace App\Modules\ECommerce\Domain\ValueObjects;

enum ProductStatus: string
{
    case DRAFT = 'draft';
    case ACTIVE = 'active';
    case INACTIVE = 'inactive';
    case OUT_OF_STOCK = 'out_of_stock';
    case DISCONTINUED = 'discontinued';
    
    public function isDraft(): bool
    {
        return $this === self::DRAFT;
    }
    
    public function isActive(): bool
    {
        return $this === self::ACTIVE;
    }
    
    public function isInactive(): bool
    {
        return $this === self::INACTIVE;
    }
    
    public function isOutOfStock(): bool
    {
        return $this === self::OUT_OF_STOCK;
    }
    
    public function isDiscontinued(): bool
    {
        return $this === self::DISCONTINUED;
    }
    
    public function isVisible(): bool
    {
        return in_array($this, [self::ACTIVE, self::OUT_OF_STOCK]);
    }
    
    public function label(): string
    {
        return match($this) {
            self::DRAFT => 'Draft',
            self::ACTIVE => 'Active',
            self::INACTIVE => 'Inactive',
            self::OUT_OF_STOCK => 'Out of Stock',
            self::DISCONTINUED => 'Discontinued'
        };
    }
}
```

#### Order Status Enum
```php
<?php

namespace App\Modules\ECommerce\Domain\ValueObjects;

enum OrderStatus: string
{
    case PENDING = 'pending';
    case CONFIRMED = 'confirmed';
    case PROCESSING = 'processing';
    case SHIPPED = 'shipped';
    case DELIVERED = 'delivered';
    case CANCELLED = 'cancelled';
    case REFUNDED = 'refunded';
    
    public function isPending(): bool
    {
        return $this === self::PENDING;
    }
    
    public function isConfirmed(): bool
    {
        return $this === self::CONFIRMED;
    }
    
    public function isProcessing(): bool
    {
        return $this === self::PROCESSING;
    }
    
    public function isShipped(): bool
    {
        return $this === self::SHIPPED;
    }
    
    public function isDelivered(): bool
    {
        return $this === self::DELIVERED;
    }
    
    public function isCancelled(): bool
    {
        return $this === self::CANCELLED;
    }
    
    public function isRefunded(): bool
    {
        return $this === self::REFUNDED;
    }
    
    public function canBeModified(): bool
    {
        return in_array($this, [self::PENDING, self::CONFIRMED]);
    }
    
    public function canBeCancelled(): bool
    {
        return in_array($this, [self::PENDING, self::CONFIRMED, self::PROCESSING]);
    }
    
    public function label(): string
    {
        return match($this) {
            self::PENDING => 'Pending',
            self::CONFIRMED => 'Confirmed',
            self::PROCESSING => 'Processing',
            self::SHIPPED => 'Shipped',
            self::DELIVERED => 'Delivered',
            self::CANCELLED => 'Cancelled',
            self::REFUNDED => 'Refunded'
        };
    }
}
```

## Application Layer

### Commands and Command Handlers

#### Create Product Command
```php
<?php

namespace App\Modules\ECommerce\Application\Commands;

class CreateProductCommand
{
    public function __construct(
        public readonly int $businessId,
        public readonly int $vendorId,
        public readonly string $sku,
        public readonly string $nameEn,
        public readonly ?string $nameAr,
        public readonly string $descriptionEn,
        public readonly ?string $descriptionAr,
        public readonly int $categoryId,
        public readonly float $price,
        public readonly ?float $comparePrice,
        public readonly ?float $costPrice,
        public readonly ?float $weight,
        public readonly ?array $dimensions,
        public readonly string $status,
        public readonly bool $featured,
        public readonly int $stockQuantity,
        public readonly array $images = [],
        public readonly array $attributes = []
    ) {}
}
```

#### Create Product Command Handler
```php
<?php

namespace App\Modules\ECommerce\Application\Commands;

use App\Modules\ECommerce\Domain\Contracts\ProductRepositoryInterface;
use App\Modules\ECommerce\Domain\Entities\Product;
use App\Modules\ECommerce\Domain\ValueObjects\SKU;
use App\Modules\ECommerce\Domain\ValueObjects\Money;
use App\Modules\ECommerce\Domain\ValueObjects\ProductStatus;

class CreateProductCommandHandler
{
    public function __construct(
        private ProductRepositoryInterface $productRepository
    ) {}
    
    public function handle(CreateProductCommand $command): Product
    {
        // Validate SKU uniqueness
        if ($this->productRepository->existsBySku(new SKU($command->sku))) {
            throw new DuplicateSKUException('SKU already exists');
        }
        
        // Create product entity
        $product = new Product(
            businessId: $command->businessId,
            vendorId: $command->vendorId,
            sku: new SKU($command->sku),
            nameEn: $command->nameEn,
            price: new Money($command->price)
        );
        
        // Set additional properties
        if ($command->nameAr) {
            $product->setNameAr($command->nameAr);
        }
        
        if ($command->comparePrice) {
            $product->setComparePrice(new Money($command->comparePrice));
        }
        
        if ($command->costPrice) {
            $product->setCostPrice(new Money($command->costPrice));
        }
        
        $product->setStatus(ProductStatus::from($command->status));
        $product->setFeatured($command->featured);
        $product->setStockQuantity($command->stockQuantity);
        
        // Handle images and attributes
        foreach ($command->images as $imageData) {
            $product->addImage($imageData);
        }
        
        foreach ($command->attributes as $attributeData) {
            $product->addAttribute($attributeData);
        }
        
        // Persist the product
        return $this->productRepository->save($product);
    }
}
```

### Services

#### Cart Service
```php
<?php

namespace App\Modules\ECommerce\Application\Services;

use App\Modules\ECommerce\Domain\Contracts\CartRepositoryInterface;
use App\Modules\ECommerce\Domain\Contracts\ProductRepositoryInterface;
use App\Modules\ECommerce\Domain\Entities\Cart;
use App\Modules\ECommerce\Domain\Entities\CartItem;

class CartService
{
    public function __construct(
        private CartRepositoryInterface $cartRepository,
        private ProductRepositoryInterface $productRepository
    ) {}
    
    public function getOrCreateCart(int $userId, ?string $sessionId = null): Cart
    {
        $cart = $this->cartRepository->findByUserOrSession($userId, $sessionId);
        
        if (!$cart) {
            $cart = new Cart($userId, $sessionId);
            $cart = $this->cartRepository->save($cart);
        }
        
        return $cart;
    }
    
    public function addItem(Cart $cart, int $productId, int $quantity, array $attributes = []): Cart
    {
        $product = $this->productRepository->findById($productId);
        
        if (!$product) {
            throw new ProductNotFoundException('Product not found');
        }
        
        if (!$product->isAvailable($quantity)) {
            throw new InsufficientStockException('Product is not available in requested quantity');
        }
        
        $existingItem = $cart->findItemByProduct($productId);
        
        if ($existingItem) {
            $existingItem->updateQuantity($existingItem->getQuantity() + $quantity);
        } else {
            $item = new CartItem(
                productId: $productId,
                vendorId: $product->getVendorId(),
                quantity: $quantity,
                unitPrice: $product->getPrice(),
                productName: $product->getNameEn(),
                productSku: $product->getSku()->getValue(),
                attributes: $attributes
            );
            
            $cart->addItem($item);
        }
        
        return $this->cartRepository->save($cart);
    }
    
    public function updateItemQuantity(Cart $cart, int $itemId, int $quantity): Cart
    {
        $item = $cart->getItem($itemId);
        
        if (!$item) {
            throw new CartItemNotFoundException('Cart item not found');
        }
        
        $product = $this->productRepository->findById($item->getProductId());
        
        if (!$product->isAvailable($quantity)) {
            throw new InsufficientStockException('Product is not available in requested quantity');
        }
        
        if ($quantity <= 0) {
            $cart->removeItem($itemId);
        } else {
            $item->updateQuantity($quantity);
        }
        
        return $this->cartRepository->save($cart);
    }
    
    public function applyCoupon(Cart $cart, string $couponCode): Cart
    {
        // Validate coupon through coupon service
        $couponService = app(CouponService::class);
        $discount = $couponService->validateAndCalculateDiscount($couponCode, $cart);
        
        $cart->applyCoupon($couponCode, $discount);
        
        return $this->cartRepository->save($cart);
    }
    
    public function calculateTotals(Cart $cart): array
    {
        $subtotal = $cart->getSubtotal();
        $taxAmount = $this->calculateTax($cart);
        $shippingAmount = $this->calculateShipping($cart);
        $discountAmount = $cart->getDiscountAmount();
        
        $total = $subtotal
            ->add($taxAmount)
            ->add($shippingAmount)
            ->subtract($discountAmount);
        
        return [
            'subtotal' => $subtotal,
            'tax_amount' => $taxAmount,
            'shipping_amount' => $shippingAmount,
            'discount_amount' => $discountAmount,
            'total' => $total
        ];
    }
    
    private function calculateTax(Cart $cart): Money
    {
        // Tax calculation logic based on business rules
        // This might integrate with a tax service or use business-specific rates
        return new Money(0);
    }
    
    private function calculateShipping(Cart $cart): Money
    {
        // Shipping calculation logic
        // This would integrate with shipping providers or use configured rates
        return new Money(0);
    }
}
```

#### Order Service
```php
<?php

namespace App\Modules\ECommerce\Application\Services;

use App\Modules\ECommerce\Domain\Contracts\OrderRepositoryInterface;
use App\Modules\ECommerce\Domain\Contracts\ProductRepositoryInterface;
use App\Modules\ECommerce\Domain\Entities\Order;
use App\Modules\ECommerce\Domain\Entities\OrderItem;
use App\Modules\ECommerce\Domain\Entities\Cart;
use App\Modules\ECommerce\Domain\ValueObjects\Address;

class OrderService
{
    public function __construct(
        private OrderRepositoryInterface $orderRepository,
        private ProductRepositoryInterface $productRepository,
        private PaymentService $paymentService,
        private InventoryService $inventoryService
    ) {}
    
    public function createFromCart(
        Cart $cart,
        Address $billingAddress,
        Address $shippingAddress,
        int $paymentMethodId,
        int $shippingMethodId,
        ?string $notes = null
    ): Order {
        // Create order
        $order = new Order(
            businessId: $cart->getBusinessId(),
            customerId: $cart->getUserId(),
            billingAddress: $billingAddress,
            shippingAddress: $shippingAddress,
            currencyCode: $cart->getCurrencyCode()
        );
        
        // Add items from cart
        foreach ($cart->getItems() as $cartItem) {
            $product = $this->productRepository->findById($cartItem->getProductId());
            
            // Verify product availability
            if (!$product->isAvailable($cartItem->getQuantity())) {
                throw new InsufficientStockException(
                    "Product {$product->getNameEn()} is not available in requested quantity"
                );
            }
            
            $orderItem = new OrderItem(
                productId: $cartItem->getProductId(),
                vendorId: $cartItem->getVendorId(),
                productName: $cartItem->getProductName(),
                productSku: $cartItem->getProductSku(),
                quantity: $cartItem->getQuantity(),
                unitPrice: $cartItem->getUnitPrice()
            );
            
            $order->addItem($orderItem);
        }
        
        // Apply coupon if present
        if ($cart->getCouponCode()) {
            $order->applyCoupon($cart->getCouponCode(), $cart->getDiscountAmount());
        }
        
        // Set shipping and notes
        if ($notes) {
            $order->setNotes($notes);
        }
        
        // Calculate totals
        $this->calculateOrderTotals($order, $shippingMethodId);
        
        // Save order
        $order = $this->orderRepository->save($order);
        
        // Reserve inventory
        $this->reserveInventory($order);
        
        return $order;
    }
    
    public function processPayment(Order $order, int $paymentMethodId, array $paymentDetails): bool
    {
        try {
            $result = $this->paymentService->processPayment(
                order: $order,
                paymentMethodId: $paymentMethodId,
                paymentDetails: $paymentDetails
            );
            
            if ($result->isSuccessful()) {
                $order->markAsPaid($result->getTransactionId());
                $order->confirm();
                $this->orderRepository->save($order);
                
                // Update inventory
                $this->inventoryService->confirmReservation($order);
                
                return true;
            }
            
            return false;
        } catch (PaymentException $e) {
            // Handle payment failure
            $order->markPaymentAsFailed($e->getMessage());
            $this->orderRepository->save($order);
            
            // Release reserved inventory
            $this->inventoryService->releaseReservation($order);
            
            return false;
        }
    }
    
    public function cancelOrder(Order $order, string $reason = ''): void
    {
        if (!$order->canBeCancelled()) {
            throw new InvalidOrderStatusException('Order cannot be cancelled');
        }
        
        $order->cancel($reason);
        $this->orderRepository->save($order);
        
        // Release inventory
        $this->inventoryService->releaseReservation($order);
        
        // Process refund if payment was made
        if ($order->getPaymentStatus()->isPaid()) {
            $this->paymentService->processRefund($order);
        }
    }
    
    private function reserveInventory(Order $order): void
    {
        foreach ($order->getItems() as $item) {
            $product = $this->productRepository->findById($item->getProductId());
            $product->reserveStock($item->getQuantity());
            $this->productRepository->save($product);
        }
    }
    
    private function calculateOrderTotals(Order $order, int $shippingMethodId): void
    {
        // Tax calculation
        $taxAmount = $this->calculateOrderTax($order);
        $order->setTaxAmount($taxAmount);
        
        // Shipping calculation
        $shippingAmount = $this->calculateOrderShipping($order, $shippingMethodId);
        $order->setShippingAmount($shippingAmount);
        
        // Recalculate total
        $order->recalculateTotal();
    }
    
    private function calculateOrderTax(Order $order): Money
    {
        // Tax calculation logic
        return new Money(0);
    }
    
    private function calculateOrderShipping(Order $order, int $shippingMethodId): Money
    {
        // Shipping calculation logic
        return new Money(0);
    }
}
```

## Infrastructure Layer

### Repository Implementations

#### Product Repository
```php
<?php

namespace App\Modules\ECommerce\Infrastructure\Repositories;

use App\Modules\ECommerce\Domain\Contracts\ProductRepositoryInterface;
use App\Modules\ECommerce\Domain\Entities\Product;
use App\Modules\ECommerce\Domain\ValueObjects\SKU;
use App\Modules\ECommerce\Infrastructure\Models\Product as ProductModel;
use Illuminate\Contracts\Pagination\LengthAwarePaginator;

class ProductRepository implements ProductRepositoryInterface
{
    public function findById(int $id): ?Product
    {
        $model = ProductModel::with(['vendor', 'category', 'images', 'attributes'])->find($id);
        
        return $model ? $this->toDomainEntity($model) : null;
    }
    
    public function findBySku(SKU $sku): ?Product
    {
        $model = ProductModel::where('sku', $sku->getValue())->first();
        
        return $model ? $this->toDomainEntity($model) : null;
    }
    
    public function existsBySku(SKU $sku): bool
    {
        return ProductModel::where('sku', $sku->getValue())->exists();
    }
    
    public function save(Product $product): Product
    {
        $model = $product->getId() ? 
            ProductModel::find($product->getId()) : 
            new ProductModel();
        
        $this->mapToModel($product, $model);
        $model->save();
        
        // Handle images and attributes
        $this->syncImages($product, $model);
        $this->syncAttributes($product, $model);
        
        // Return updated domain entity
        return $this->toDomainEntity($model->fresh(['vendor', 'category', 'images', 'attributes']));
    }
    
    public function delete(Product $product): void
    {
        if ($product->getId()) {
            ProductModel::destroy($product->getId());
        }
    }
    
    public function findByVendor(int $vendorId, array $filters = []): LengthAwarePaginator
    {
        $query = ProductModel::where('vendor_id', $vendorId)
            ->with(['category', 'images', 'attributes']);
        
        // Apply filters
        if (isset($filters['status'])) {
            $query->where('status', $filters['status']);
        }
        
        if (isset($filters['category_id'])) {
            $query->where('product_category_id', $filters['category_id']);
        }
        
        if (isset($filters['search'])) {
            $query->where(function ($q) use ($filters) {
                $q->where('name_en', 'like', '%' . $filters['search'] . '%')
                  ->orWhere('name_ar', 'like', '%' . $filters['search'] . '%')
                  ->orWhere('sku', 'like', '%' . $filters['search'] . '%');
            });
        }
        
        // Sorting
        $sortBy = $filters['sort'] ?? 'created_at';
        $sortOrder = $filters['order'] ?? 'desc';
        $query->orderBy($sortBy, $sortOrder);
        
        return $query->paginate($filters['per_page'] ?? 15);
    }
    
    private function toDomainEntity(ProductModel $model): Product
    {
        $product = new Product(
            businessId: $model->business_id,
            vendorId: $model->vendor_id,
            sku: new SKU($model->sku),
            nameEn: $model->name_en,
            price: new Money($model->price)
        );
        
        // Set ID and other properties
        $product->setId($model->id);
        
        if ($model->name_ar) {
            $product->setNameAr($model->name_ar);
        }
        
        if ($model->compare_price) {
            $product->setComparePrice(new Money($model->compare_price));
        }
        
        if ($model->cost_price) {
            $product->setCostPrice(new Money($model->cost_price));
        }
        
        $product->setStatus(ProductStatus::from($model->status));
        $product->setFeatured($model->featured);
        $product->setStockQuantity($model->stock_quantity);
        $product->setStockThreshold($model->stock_threshold);
        $product->setTrackInventory($model->track_inventory);
        $product->setAllowBackorders($model->allow_backorders);
        
        return $product;
    }
    
    private function mapToModel(Product $product, ProductModel $model): void
    {
        $model->business_id = $product->getBusinessId();
        $model->vendor_id = $product->getVendorId();
        $model->sku = $product->getSku()->getValue();
        $model->name_en = $product->getNameEn();
        $model->name_ar = $product->getNameAr();
        $model->slug = $product->getSlug();
        $model->price = $product->getPrice()->getAmount();
        $model->compare_price = $product->getComparePrice()?->getAmount();
        $model->cost_price = $product->getCostPrice()?->getAmount();
        $model->status = $product->getStatus()->value;
        $model->featured = $product->isFeatured();
        $model->stock_quantity = $product->getStockQuantity();
        $model->stock_threshold = $product->getStockThreshold();
        $model->track_inventory = $product->shouldTrackInventory();
        $model->allow_backorders = $product->allowsBackorders();
    }
    
    private function syncImages(Product $product, ProductModel $model): void
    {
        // Sync product images
        $imageData = $product->getImages()->map(function ($image) {
            return [
                'file_url' => $image->getFileUrl(),
                'alt_text_en' => $image->getAltTextEn(),
                'alt_text_ar' => $image->getAltTextAr(),
                'is_primary' => $image->isPrimary(),
                'sort_order' => $image->getSortOrder()
            ];
        })->toArray();
        
        $model->images()->delete();
        $model->images()->createMany($imageData);
    }
    
    private function syncAttributes(Product $product, ProductModel $model): void
    {
        // Sync product attributes
        $attributeData = $product->getAttributes()->mapWithKeys(function ($value, $attributeId) {
            return [$attributeId => ['value' => $value]];
        })->toArray();
        
        $model->attributes()->sync($attributeData);
    }
}
```

### Eloquent Models

#### Product Model
```php
<?php

namespace App\Modules\ECommerce\Infrastructure\Models;

use App\Modules\Shared\Infrastructure\Models\BaseModel;
use App\Modules\BusinessDirectory\Infrastructure\Models\Business;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;
use Illuminate\Database\Eloquent\Relations\BelongsToMany;
use Illuminate\Database\Eloquent\SoftDeletes;

class Product extends BaseModel
{
    use SoftDeletes;
    
    protected $table = 'ecommerce_products';
    
    protected $fillable = [
        'business_id',
        'vendor_id',
        'warehouse_id',
        'product_category_id',
        'parent_id',
        'sku',
        'name_en',
        'name_ar',
        'slug',
        'description_en',
        'description_ar',
        'short_description_en',
        'short_description_ar',
        'price',
        'compare_price',
        'cost_price',
        'tax_class',
        'weight',
        'dimensions',
        'status',
        'visibility',
        'featured',
        'product_type',
        'digital_file_url',
        'requires_shipping',
        'track_inventory',
        'stock_quantity',
        'stock_threshold',
        'allow_backorders',
        'meta_title',
        'meta_description',
        'seo_keywords',
        'tags',
        'published_at'
    ];
    
    protected $casts = [
        'dimensions' => 'array',
        'tags' => 'array',
        'price' => 'decimal:4',
        'compare_price' => 'decimal:4',
        'cost_price' => 'decimal:4',
        'weight' => 'decimal:3',
        'featured' => 'boolean',
        'requires_shipping' => 'boolean',
        'track_inventory' => 'boolean',
        'allow_backorders' => 'boolean',
        'published_at' => 'datetime'
    ];
    
    protected $dates = [
        'published_at',
        'deleted_at'
    ];
    
    // Relationships
    public function business(): BelongsTo
    {
        return $this->belongsTo(Business::class);
    }
    
    public function vendor(): BelongsTo
    {
        return $this->belongsTo(Vendor::class);
    }
    
    public function category(): BelongsTo
    {
        return $this->belongsTo(ProductCategory::class, 'product_category_id');
    }
    
    public function warehouse(): BelongsTo
    {
        return $this->belongsTo(\App\Modules\Warehouses\Infrastructure\Models\Warehouse::class);
    }
    
    public function parent(): BelongsTo
    {
        return $this->belongsTo(Product::class, 'parent_id');
    }
    
    public function variants(): HasMany
    {
        return $this->hasMany(Product::class, 'parent_id');
    }
    
    public function images(): HasMany
    {
        return $this->hasMany(ProductMedia::class)->where('type', 'image');
    }
    
    public function media(): HasMany
    {
        return $this->hasMany(ProductMedia::class);
    }
    
    public function attributes(): BelongsToMany
    {
        return $this->belongsToMany(ProductAttribute::class, 'ecommerce_product_attribute_values')
            ->withPivot(['value_text', 'value_number', 'value_boolean', 'value_date'])
            ->withTimestamps();
    }
    
    public function cartItems(): HasMany
    {
        return $this->hasMany(CartItem::class);
    }
    
    public function orderItems(): HasMany
    {
        return $this->hasMany(OrderItem::class);
    }
    
    // Accessors & Mutators
    public function getPrimaryImageAttribute()
    {
        return $this->images()->where('is_primary', true)->first();
    }
    
    public function getFormattedPriceAttribute(): string
    {
        return number_format($this->price, 2) . ' EGP';
    }
    
    public function getDiscountPercentageAttribute(): ?float
    {
        if (!$this->compare_price || $this->compare_price <= $this->price) {
            return null;
        }
        
        return round((($this->compare_price - $this->price) / $this->compare_price) * 100, 2);
    }
    
    // Scopes
    public function scopeActive($query)
    {
        return $query->where('status', 'active');
    }
    
    public function scopeVisible($query)
    {
        return $query->whereIn('status', ['active', 'out_of_stock'])
                     ->where('visibility', 'public');
    }
    
    public function scopeFeatured($query)
    {
        return $query->where('featured', true);
    }
    
    public function scopeInStock($query)
    {
        return $query->where(function ($q) {
            $q->where('track_inventory', false)
              ->orWhere(function ($sq) {
                  $sq->where('track_inventory', true)
                     ->where(function ($ssq) {
                         $ssq->where('stock_quantity', '>', 0)
                            ->orWhere('allow_backorders', true);
                     });
              });
        });
    }
    
    public function scopeByCategory($query, $categoryId)
    {
        return $query->where('product_category_id', $categoryId);
    }
    
    public function scopeByVendor($query, $vendorId)
    {
        return $query->where('vendor_id', $vendorId);
    }
    
    public function scopeSearch($query, $search)
    {
        return $query->where(function ($q) use ($search) {
            $q->where('name_en', 'like', "%{$search}%")
              ->orWhere('name_ar', 'like', "%{$search}%")
              ->orWhere('description_en', 'like', "%{$search}%")
              ->orWhere('description_ar', 'like', "%{$search}%")
              ->orWhere('sku', 'like', "%{$search}%");
        });
    }
    
    public function scopePriceRange($query, $minPrice = null, $maxPrice = null)
    {
        if ($minPrice !== null) {
            $query->where('price', '>=', $minPrice);
        }
        
        if ($maxPrice !== null) {
            $query->where('price', '<=', $maxPrice);
        }
        
        return $query;
    }
}
```

This comprehensive architecture document provides the foundation for implementing the e-commerce module using Domain-Driven Design principles. The structure ensures maintainability, testability, and seamless integration with the existing OmarStartKit modules while following Laravel best practices.