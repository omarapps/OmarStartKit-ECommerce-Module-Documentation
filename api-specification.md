# API Specification for E-Commerce Module

## Overview
This document defines the comprehensive API endpoints for the multi-vendor e-commerce module, following RESTful principles and integrating seamlessly with existing OmarStartKit architecture.

## API Design Principles
- **RESTful Architecture**: Standard HTTP methods and status codes
- **Multi-tenant Support**: Business context isolation
- **Consistent Response Format**: Standardized JSON responses
- **Comprehensive Error Handling**: Detailed error messages and codes
- **Rate Limiting**: Protection against abuse
- **Authentication**: Sanctum token-based authentication
- **Authorization**: Role-based access control integration

## Base URL Structure
```
https://your-domain.com/api/v1/ecommerce/
```

## Authentication
All API endpoints require authentication using Laravel Sanctum tokens:
```http
Authorization: Bearer {access_token}
```

## Standard Response Format

### Success Response
```json
{
    "success": true,
    "data": {
        // Response data
    },
    "meta": {
        "total": 100,
        "page": 1,
        "per_page": 15,
        "last_page": 7
    }
}
```

### Error Response
```json
{
    "success": false,
    "error": {
        "code": "VALIDATION_ERROR",
        "message": "The given data was invalid.",
        "details": {
            "field_name": ["Field is required"]
        }
    }
}
```

## Product Management APIs

### 1. List Products
```http
GET /api/v1/ecommerce/products
```

**Query Parameters:**
- `page` (integer): Page number for pagination
- `per_page` (integer): Items per page (max 100)
- `category_id` (integer): Filter by category
- `vendor_id` (integer): Filter by vendor
- `status` (string): Filter by status (draft|active|inactive|out_of_stock)
- `featured` (boolean): Filter featured products
- `search` (string): Search in name and description
- `sort` (string): Sort by (name|price|created_at|rating)
- `order` (string): Sort order (asc|desc)
- `min_price` (decimal): Minimum price filter
- `max_price` (decimal): Maximum price filter

**Response:**
```json
{
    "success": true,
    "data": [
        {
            "id": 1,
            "business_id": 1,
            "vendor_id": 1,
            "sku": "PROD-001",
            "name_en": "Product Name",
            "name_ar": "اسم المنتج",
            "slug": "product-name",
            "price": 99.99,
            "compare_price": 129.99,
            "status": "active",
            "featured": true,
            "stock_quantity": 50,
            "rating": 4.5,
            "reviews_count": 25,
            "primary_image": "https://example.com/image.jpg",
            "vendor": {
                "id": 1,
                "display_name": "Vendor Name",
                "rating": 4.8
            },
            "category": {
                "id": 1,
                "name_en": "Category Name",
                "name_ar": "اسم الفئة"
            },
            "created_at": "2024-01-15T10:30:00Z"
        }
    ],
    "meta": {
        "total": 150,
        "page": 1,
        "per_page": 15,
        "last_page": 10
    }
}
```

### 2. Get Product Details
```http
GET /api/v1/ecommerce/products/{id}
```

**Response:**
```json
{
    "success": true,
    "data": {
        "id": 1,
        "business_id": 1,
        "vendor_id": 1,
        "sku": "PROD-001",
        "name_en": "Product Name",
        "name_ar": "اسم المنتج",
        "slug": "product-name",
        "description_en": "Product description",
        "description_ar": "وصف المنتج",
        "price": 99.99,
        "compare_price": 129.99,
        "cost_price": 60.00,
        "weight": 0.5,
        "dimensions": {
            "length": 10,
            "width": 15,
            "height": 5
        },
        "status": "active",
        "visibility": "public",
        "featured": true,
        "product_type": "physical",
        "requires_shipping": true,
        "track_inventory": true,
        "stock_quantity": 50,
        "stock_threshold": 5,
        "allow_backorders": false,
        "rating": 4.5,
        "reviews_count": 25,
        "tags": ["electronics", "gadgets"],
        "vendor": {
            "id": 1,
            "display_name": "Vendor Name",
            "rating": 4.8,
            "total_orders": 150
        },
        "category": {
            "id": 1,
            "name_en": "Category Name",
            "name_ar": "اسم الفئة",
            "slug": "category-slug"
        },
        "images": [
            {
                "id": 1,
                "file_url": "https://example.com/image1.jpg",
                "alt_text_en": "Product image",
                "is_primary": true,
                "sort_order": 0
            }
        ],
        "attributes": [
            {
                "id": 1,
                "name_en": "Color",
                "name_ar": "اللون",
                "type": "select",
                "value": "Red"
            }
        ],
        "related_products": [
            {
                "id": 2,
                "name_en": "Related Product",
                "price": 79.99,
                "primary_image": "https://example.com/related.jpg"
            }
        ],
        "created_at": "2024-01-15T10:30:00Z",
        "updated_at": "2024-01-20T14:45:00Z"
    }
}
```

### 3. Create Product
```http
POST /api/v1/ecommerce/products
```

**Request Body:**
```json
{
    "sku": "PROD-002",
    "name_en": "New Product",
    "name_ar": "منتج جديد",
    "description_en": "Product description",
    "description_ar": "وصف المنتج",
    "category_id": 1,
    "price": 99.99,
    "compare_price": 129.99,
    "cost_price": 60.00,
    "weight": 0.5,
    "dimensions": {
        "length": 10,
        "width": 15,
        "height": 5
    },
    "status": "active",
    "visibility": "public",
    "featured": false,
    "product_type": "physical",
    "requires_shipping": true,
    "track_inventory": true,
    "stock_quantity": 100,
    "stock_threshold": 10,
    "allow_backorders": false,
    "tags": ["new", "electronics"],
    "images": [
        {
            "file_url": "https://example.com/image.jpg",
            "alt_text_en": "Product image",
            "is_primary": true,
            "sort_order": 0
        }
    ],
    "attributes": [
        {
            "attribute_id": 1,
            "value": "Blue"
        }
    ]
}
```

### 4. Update Product
```http
PUT /api/v1/ecommerce/products/{id}
```

### 5. Delete Product
```http
DELETE /api/v1/ecommerce/products/{id}
```

## Category Management APIs

### 1. List Categories
```http
GET /api/v1/ecommerce/categories
```

**Query Parameters:**
- `parent_id` (integer): Filter by parent category
- `level` (integer): Filter by hierarchy level
- `featured` (boolean): Filter featured categories
- `active` (boolean): Filter active categories

**Response:**
```json
{
    "success": true,
    "data": [
        {
            "id": 1,
            "parent_id": null,
            "name_en": "Electronics",
            "name_ar": "الإلكترونيات",
            "slug": "electronics",
            "description_en": "Electronic devices and gadgets",
            "image_url": "https://example.com/category.jpg",
            "icon": "mdi-laptop",
            "color": "blue",
            "is_active": true,
            "is_featured": true,
            "products_count": 150,
            "children": [
                {
                    "id": 2,
                    "name_en": "Smartphones",
                    "name_ar": "الهواتف الذكية",
                    "products_count": 45
                }
            ]
        }
    ]
}
```

### 2. Get Category Details
```http
GET /api/v1/ecommerce/categories/{id}
```

### 3. Create Category
```http
POST /api/v1/ecommerce/categories
```

### 4. Update Category
```http
PUT /api/v1/ecommerce/categories/{id}
```

### 5. Delete Category
```http
DELETE /api/v1/ecommerce/categories/{id}
```

## Shopping Cart APIs

### 1. Get Cart
```http
GET /api/v1/ecommerce/cart
```

**Response:**
```json
{
    "success": true,
    "data": {
        "id": 1,
        "user_id": 1,
        "subtotal": 199.98,
        "tax_amount": 20.00,
        "discount_amount": 10.00,
        "shipping_amount": 15.00,
        "total_amount": 224.98,
        "currency_code": "EGP",
        "coupon_code": "SAVE10",
        "items": [
            {
                "id": 1,
                "product_id": 1,
                "vendor_id": 1,
                "product_name": "Product Name",
                "product_sku": "PROD-001",
                "product_image": "https://example.com/image.jpg",
                "quantity": 2,
                "unit_price": 99.99,
                "total_price": 199.98,
                "vendor": {
                    "id": 1,
                    "display_name": "Vendor Name"
                }
            }
        ],
        "shipping_methods": [
            {
                "id": 1,
                "name_en": "Standard Shipping",
                "cost": 15.00,
                "delivery_days": "3-5 days"
            }
        ]
    }
}
```

### 2. Add Item to Cart
```http
POST /api/v1/ecommerce/cart/items
```

**Request Body:**
```json
{
    "product_id": 1,
    "quantity": 2,
    "attributes": {
        "color": "Red",
        "size": "M"
    }
}
```

### 3. Update Cart Item
```http
PUT /api/v1/ecommerce/cart/items/{item_id}
```

**Request Body:**
```json
{
    "quantity": 3
}
```

### 4. Remove Item from Cart
```http
DELETE /api/v1/ecommerce/cart/items/{item_id}
```

### 5. Apply Coupon
```http
POST /api/v1/ecommerce/cart/coupon
```

**Request Body:**
```json
{
    "coupon_code": "SAVE10"
}
```

### 6. Remove Coupon
```http
DELETE /api/v1/ecommerce/cart/coupon
```

### 7. Clear Cart
```http
DELETE /api/v1/ecommerce/cart
```

## Order Management APIs

### 1. List Orders
```http
GET /api/v1/ecommerce/orders
```

**Query Parameters:**
- `status` (string): Filter by order status
- `payment_status` (string): Filter by payment status
- `fulfillment_status` (string): Filter by fulfillment status
- `date_from` (date): Filter orders from date
- `date_to` (date): Filter orders to date
- `vendor_id` (integer): Filter by vendor (for vendor dashboard)

**Response:**
```json
{
    "success": true,
    "data": [
        {
            "id": 1,
            "order_number": "ORD-2024-001",
            "status": "confirmed",
            "payment_status": "paid",
            "fulfillment_status": "processing",
            "total_amount": 224.98,
            "currency_code": "EGP",
            "customer": {
                "id": 1,
                "name": "John Doe",
                "email": "john@example.com"
            },
            "items_count": 2,
            "placed_at": "2024-01-15T10:30:00Z"
        }
    ]
}
```

### 2. Get Order Details
```http
GET /api/v1/ecommerce/orders/{id}
```

**Response:**
```json
{
    "success": true,
    "data": {
        "id": 1,
        "order_number": "ORD-2024-001",
        "status": "confirmed",
        "payment_status": "paid",
        "fulfillment_status": "processing",
        "subtotal": 199.98,
        "tax_amount": 20.00,
        "discount_amount": 10.00,
        "shipping_amount": 15.00,
        "total_amount": 224.98,
        "currency_code": "EGP",
        "coupon_code": "SAVE10",
        "customer": {
            "id": 1,
            "name": "John Doe",
            "email": "john@example.com",
            "phone": "+20123456789"
        },
        "billing_address": {
            "name": "John Doe",
            "email": "john@example.com",
            "phone": "+20123456789",
            "address_line_1": "123 Main St",
            "address_line_2": "Apt 4B",
            "city": "Cairo",
            "state": "Cairo Governorate",
            "postal_code": "12345",
            "country": "Egypt"
        },
        "shipping_address": {
            "name": "John Doe",
            "address_line_1": "123 Main St",
            "city": "Cairo",
            "country": "Egypt"
        },
        "items": [
            {
                "id": 1,
                "product_id": 1,
                "vendor_id": 1,
                "product_name": "Product Name",
                "product_sku": "PROD-001",
                "product_image": "https://example.com/image.jpg",
                "quantity": 2,
                "unit_price": 99.99,
                "total_price": 199.98,
                "fulfillment_status": "processing",
                "tracking_number": null,
                "vendor": {
                    "id": 1,
                    "display_name": "Vendor Name"
                }
            }
        ],
        "payment_transactions": [
            {
                "id": 1,
                "transaction_id": "TXN-123456",
                "gateway_transaction_id": "stripe_pi_123",
                "type": "payment",
                "status": "completed",
                "amount": 224.98,
                "payment_method": {
                    "name_en": "Credit Card",
                    "type": "card"
                },
                "processed_at": "2024-01-15T10:35:00Z"
            }
        ],
        "notes": "Please deliver to the main entrance",
        "placed_at": "2024-01-15T10:30:00Z",
        "confirmed_at": "2024-01-15T11:00:00Z"
    }
}
```

### 3. Create Order (Checkout)
```http
POST /api/v1/ecommerce/orders
```

**Request Body:**
```json
{
    "billing_address": {
        "name": "John Doe",
        "email": "john@example.com",
        "phone": "+20123456789",
        "address_line_1": "123 Main St",
        "address_line_2": "Apt 4B",
        "city": "Cairo",
        "state": "Cairo Governorate",
        "postal_code": "12345",
        "country": "Egypt"
    },
    "shipping_address": {
        "name": "John Doe",
        "address_line_1": "123 Main St",
        "city": "Cairo",
        "country": "Egypt"
    },
    "shipping_method_id": 1,
    "payment_method_id": 1,
    "notes": "Please deliver to the main entrance",
    "use_saved_address": false
}
```

### 4. Update Order Status
```http
PUT /api/v1/ecommerce/orders/{id}/status
```

**Request Body:**
```json
{
    "status": "shipped",
    "fulfillment_status": "shipped",
    "tracking_number": "TRACK123456",
    "notes": "Order shipped via DHL"
}
```

### 5. Cancel Order
```http
POST /api/v1/ecommerce/orders/{id}/cancel
```

**Request Body:**
```json
{
    "reason": "Customer requested cancellation",
    "refund_amount": 224.98
}
```

## Vendor Management APIs

### 1. List Vendors
```http
GET /api/v1/ecommerce/vendors
```

**Response:**
```json
{
    "success": true,
    "data": [
        {
            "id": 1,
            "business_id": 1,
            "vendor_code": "VND001",
            "display_name": "Tech Store",
            "status": "active",
            "verification_status": "verified",
            "rating": 4.8,
            "total_sales": 15000.00,
            "total_orders": 150,
            "commission_rate": 10.00,
            "business": {
                "name_en": "Tech Business",
                "city": "Cairo",
                "country": "Egypt"
            },
            "created_at": "2024-01-01T00:00:00Z"
        }
    ]
}
```

### 2. Get Vendor Details
```http
GET /api/v1/ecommerce/vendors/{id}
```

### 3. Register as Vendor
```http
POST /api/v1/ecommerce/vendors/register
```

**Request Body:**
```json
{
    "business_id": 1,
    "display_name": "My Store",
    "bank_name": "Bank of Egypt",
    "bank_account_number": "123456789",
    "payment_method": "bank_transfer",
    "store_description": "We sell quality products",
    "return_policy": "30-day return policy"
}
```

### 4. Update Vendor Profile
```http
PUT /api/v1/ecommerce/vendors/{id}
```

### 5. Get Vendor Dashboard Stats
```http
GET /api/v1/ecommerce/vendors/{id}/dashboard
```

**Response:**
```json
{
    "success": true,
    "data": {
        "total_products": 50,
        "active_products": 45,
        "total_orders": 150,
        "pending_orders": 5,
        "total_revenue": 15000.00,
        "pending_commissions": 500.00,
        "rating": 4.8,
        "recent_orders": [
            {
                "id": 1,
                "order_number": "ORD-2024-001",
                "total_amount": 99.99,
                "status": "confirmed",
                "placed_at": "2024-01-15T10:30:00Z"
            }
        ],
        "low_stock_products": [
            {
                "id": 1,
                "name_en": "Product Name",
                "stock_quantity": 3,
                "stock_threshold": 5
            }
        ]
    }
}
```

## Payment APIs

### 1. List Payment Methods
```http
GET /api/v1/ecommerce/payment-methods
```

**Response:**
```json
{
    "success": true,
    "data": [
        {
            "id": 1,
            "name_en": "Credit Card",
            "name_ar": "بطاقة ائتمان",
            "code": "stripe_card",
            "type": "card",
            "gateway": "stripe",
            "is_active": true,
            "settings": {
                "accepts_visa": true,
                "accepts_mastercard": true,
                "processing_fee": 2.9
            }
        }
    ]
}
```

### 2. Process Payment
```http
POST /api/v1/ecommerce/payments/process
```

**Request Body:**
```json
{
    "order_id": 1,
    "payment_method_id": 1,
    "payment_details": {
        "card_token": "stripe_token_123",
        "save_card": false
    }
}
```

### 3. Get Payment Status
```http
GET /api/v1/ecommerce/payments/{transaction_id}/status
```

## Shipping APIs

### 1. List Shipping Methods
```http
GET /api/v1/ecommerce/shipping-methods
```

**Query Parameters:**
- `vendor_id` (integer): Filter by vendor
- `city_id` (integer): Filter by delivery city
- `order_total` (decimal): Calculate rates for order total

**Response:**
```json
{
    "success": true,
    "data": [
        {
            "id": 1,
            "name_en": "Standard Shipping",
            "name_ar": "شحن عادي",
            "type": "flat_rate",
            "cost": 15.00,
            "min_delivery_days": 3,
            "max_delivery_days": 5,
            "description": "3-5 business days"
        }
    ]
}
```

### 2. Calculate Shipping Rate
```http
POST /api/v1/ecommerce/shipping/calculate
```

**Request Body:**
```json
{
    "items": [
        {
            "product_id": 1,
            "quantity": 2,
            "weight": 1.0
        }
    ],
    "shipping_address": {
        "city_id": 1,
        "postal_code": "12345"
    },
    "shipping_method_id": 1
}
```

## Coupon/Discount APIs

### 1. Validate Coupon
```http
POST /api/v1/ecommerce/coupons/validate
```

**Request Body:**
```json
{
    "coupon_code": "SAVE10",
    "cart_total": 100.00,
    "customer_id": 1,
    "products": [1, 2, 3]
}
```

**Response:**
```json
{
    "success": true,
    "data": {
        "valid": true,
        "discount_amount": 10.00,
        "coupon": {
            "id": 1,
            "code": "SAVE10",
            "type": "percentage",
            "value": 10.00,
            "description": "10% off your order"
        }
    }
}
```

### 2. List Available Coupons
```http
GET /api/v1/ecommerce/coupons
```

## Search APIs

### 1. Product Search
```http
GET /api/v1/ecommerce/search/products
```

**Query Parameters:**
- `q` (string): Search query
- `category_id` (integer): Filter by category
- `min_price` (decimal): Minimum price
- `max_price` (decimal): Maximum price
- `vendor_id` (integer): Filter by vendor
- `sort` (string): Sort by (relevance|price|rating|newest)
- `filters` (array): Additional filters

**Response:**
```json
{
    "success": true,
    "data": {
        "products": [
            {
                "id": 1,
                "name_en": "Product Name",
                "price": 99.99,
                "rating": 4.5,
                "primary_image": "https://example.com/image.jpg",
                "vendor": {
                    "display_name": "Vendor Name"
                }
            }
        ],
        "filters": {
            "categories": [
                {"id": 1, "name_en": "Electronics", "count": 25}
            ],
            "price_ranges": [
                {"min": 0, "max": 50, "count": 10},
                {"min": 50, "max": 100, "count": 15}
            ],
            "vendors": [
                {"id": 1, "name": "Vendor Name", "count": 8}
            ]
        },
        "facets": {
            "brands": ["Apple", "Samsung", "LG"],
            "attributes": {
                "color": ["Red", "Blue", "Green"],
                "size": ["S", "M", "L"]
            }
        }
    },
    "meta": {
        "total": 50,
        "page": 1,
        "per_page": 15
    }
}
```

### 2. Vendor Search
```http
GET /api/v1/ecommerce/search/vendors
```

### 3. Search Suggestions
```http
GET /api/v1/ecommerce/search/suggestions
```

**Query Parameters:**
- `q` (string): Partial search query

**Response:**
```json
{
    "success": true,
    "data": {
        "suggestions": [
            "iPhone 15",
            "iPhone 14",
            "iPhone accessories"
        ],
        "popular_searches": [
            "Samsung Galaxy",
            "Laptop",
            "Headphones"
        ]
    }
}
```

## Analytics APIs (Admin/Vendor)

### 1. Sales Analytics
```http
GET /api/v1/ecommerce/analytics/sales
```

**Query Parameters:**
- `period` (string): Time period (today|week|month|quarter|year)
- `date_from` (date): Start date
- `date_to` (date): End date
- `vendor_id` (integer): Filter by vendor
- `group_by` (string): Group results (day|week|month)

**Response:**
```json
{
    "success": true,
    "data": {
        "total_revenue": 50000.00,
        "total_orders": 500,
        "average_order_value": 100.00,
        "commission_earned": 5000.00,
        "chart_data": [
            {"date": "2024-01-15", "revenue": 1500.00, "orders": 15},
            {"date": "2024-01-16", "revenue": 2000.00, "orders": 20}
        ],
        "top_products": [
            {
                "id": 1,
                "name_en": "Best Seller",
                "revenue": 5000.00,
                "quantity_sold": 50
            }
        ],
        "top_vendors": [
            {
                "id": 1,
                "display_name": "Top Vendor",
                "revenue": 10000.00,
                "orders": 100
            }
        ]
    }
}
```

### 2. Product Performance
```http
GET /api/v1/ecommerce/analytics/products
```

### 3. Customer Analytics
```http
GET /api/v1/ecommerce/analytics/customers
```

## Webhook APIs

### 1. Payment Webhooks
```http
POST /api/v1/ecommerce/webhooks/payment/{gateway}
```

### 2. Shipping Webhooks
```http
POST /api/v1/ecommerce/webhooks/shipping/{provider}
```

## Admin APIs

### 1. Platform Statistics
```http
GET /api/v1/ecommerce/admin/stats
```

### 2. Manage Vendors
```http
GET /api/v1/ecommerce/admin/vendors
PUT /api/v1/ecommerce/admin/vendors/{id}/approve
PUT /api/v1/ecommerce/admin/vendors/{id}/suspend
```

### 3. Commission Management
```http
GET /api/v1/ecommerce/admin/commissions
POST /api/v1/ecommerce/admin/commissions/{id}/payout
```

### 4. Platform Settings
```http
GET /api/v1/ecommerce/admin/settings
PUT /api/v1/ecommerce/admin/settings
```

## Rate Limiting

### Standard Limits
- **Authenticated Users**: 1000 requests per hour
- **Guest Users**: 100 requests per hour
- **Search Endpoints**: 60 requests per minute
- **Admin Endpoints**: 500 requests per hour

### Headers
```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1640995200
```

## Error Codes

### Standard HTTP Status Codes
- `200`: Success
- `201`: Created
- `400`: Bad Request
- `401`: Unauthorized
- `403`: Forbidden
- `404`: Not Found
- `422`: Validation Error
- `429`: Too Many Requests
- `500`: Internal Server Error

### Custom Error Codes
- `INVALID_COUPON`: Coupon code is invalid or expired
- `INSUFFICIENT_STOCK`: Product is out of stock
- `VENDOR_SUSPENDED`: Vendor account is suspended
- `PAYMENT_FAILED`: Payment processing failed
- `SHIPPING_UNAVAILABLE`: Shipping not available to destination
- `ORDER_CANNOT_BE_CANCELLED`: Order is past cancellation window

## Integration Examples

### Frontend Integration (React/TypeScript)
```typescript
// API Client
class ECommerceAPI {
    private baseURL = '/api/v1/ecommerce';
    private token: string;

    async getProducts(params: ProductFilters): Promise<ProductResponse> {
        const response = await fetch(`${this.baseURL}/products?${new URLSearchParams(params)}`, {
            headers: {
                'Authorization': `Bearer ${this.token}`,
                'Accept': 'application/json'
            }
        });
        return response.json();
    }

    async addToCart(productId: number, quantity: number): Promise<CartResponse> {
        const response = await fetch(`${this.baseURL}/cart/items`, {
            method: 'POST',
            headers: {
                'Authorization': `Bearer ${this.token}`,
                'Content-Type': 'application/json'
            },
            body: JSON.stringify({ product_id: productId, quantity })
        });
        return response.json();
    }
}
```

### Backend Integration (Laravel)
```php
// Controller Example
class ProductController extends Controller
{
    public function index(ProductIndexRequest $request): JsonResponse
    {
        $products = Product::query()
            ->with(['vendor', 'category', 'primaryImage'])
            ->filter($request->validated())
            ->paginate($request->get('per_page', 15));

        return response()->json([
            'success' => true,
            'data' => ProductResource::collection($products->items()),
            'meta' => [
                'total' => $products->total(),
                'page' => $products->currentPage(),
                'per_page' => $products->perPage(),
                'last_page' => $products->lastPage()
            ]
        ]);
    }
}
```

This comprehensive API specification provides all necessary endpoints for building a complete multi-vendor e-commerce system integrated with the OmarStartKit architecture. The APIs follow RESTful principles, include proper authentication and authorization, and provide detailed error handling and response formats.