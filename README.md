# WearCast — Backend API

WearCast is a multi-vendor clothing e-commerce platform with two ways to shop:

- **Ready-made products** listed by sellers, with colours, sizes and stock.
- **Custom designs**: customers pick a blank product from a factory, upload their own images or use the design-asset library, and order a one-off printed item.

Payments go through Stripe. Each paid order is split between the platform's commission and the vendor's payout, and is delivered by a shipping company whose drivers update the delivery status live. Customers also get personalised product recommendations from a separate ML service.

This repository is the backend: an **ASP.NET Core Web API (.NET 10)** with about **170 REST endpoints**, organised by feature (Vertical Slice architecture) on **SQL Server**.

> Graduation project, South Valley University (2026).

---

## Contents

- [Who uses it](#who-uses-it)
- [Features](#features)
- [How an order flows](#how-an-order-flows)
- [Architecture](#architecture)
- [Tech stack](#tech-stack)
- [Getting started](#getting-started)

---

## Who uses it

| Actor | What they do |
|---|---|
| **Customer** | Browses products, creates designs, manages a cart, pays, tracks shipments |
| **Seller** (seller manager) | Applies to join, lists ready-made products, manages stock, sees sales stats and a wallet |
| **Factory** (factory manager) | Offers blank products for custom designs, with colours, sizes and per-side images |
| **Shipping company** (manager) | Receives ready shipments, manages drivers, assigns deliveries, follows a dashboard |
| **Driver** | Picks up and delivers shipments, sees their own dashboard and orders |
| **Admins** | Separate roles: Super Admin, Catalog Admin, Vendor Admin, Customer Service Admin and Operations Admin |

Endpoints are protected with role-based authorization for these roles.

## Features

**Accounts and security**
- Registration, login, JWT access tokens with refresh tokens and revocation.
- Email confirmation, forgot and reset password, change password (MailKit).
- Profiles and profile images for every actor type.
- User activity tracking, with logs that admins can view.

**Catalogue: ready-made products**
- Products with colours, images, per-size stock adjustment, categories and reviews.
- Favourites.
- Seller dashboard statistics; admin views across all sellers' products.

**Catalogue: custom designs**
- Factories publish blank products with colours, sizes and an image for each view side.
- A design-asset library, organised by category and managed by admins.
- Customers upload images, create and edit designs per side, and review designed products.

**Recommendations**
- Personalised designed-product recommendations from an external Python ML service, using each customer's activity.
- A training service runs the Python training script to rebuild the model.

**Vendor onboarding**
- Seller applications with email confirmation, then approval or rejection by an admin.
- Admin listings with filtering and pagination.

**Cart, checkout and payments**
- Cart for both product types, with quantity updates.
- Stripe Checkout sessions and a Stripe webhook that confirms payment.
- Platform commission, configurable by admins, taken from each order; the rest goes to the vendor.
- Wallets and wallet transactions for sellers, factories, shipping companies and customers.
- Platform dashboard for admins.

**Shipping and delivery**
- Shipping companies, their managers and drivers.
- A shipment is opened when payment succeeds, with a 6-digit delivery code that only the customer can see.
- Shipping managers assign ready shipments to available drivers; a driver can hand an assignment back.
- An enforced status lifecycle from payment to delivery (see below).
- Deactivating or deleting a driver is blocked during an active delivery; otherwise the driver's assigned shipments go back to the pool and the company is notified.
- Dashboards for shipping companies (orders and shipments by stage, active drivers, average delivery time) and for drivers.
- Separate shipment views for admins and managers, drivers and customers, with filtering, sorting and pagination.

**Notifications**
- Real-time, per-user notifications over SignalR, stored in the database.
- 11 event types across orders, products, vendor onboarding and shipping.
- List, read, read all, delete, and an undelivered counter.

## How an order flows

```
Customer ──► Cart ──► Checkout (Stripe session)
                           │
                           ▼
              Stripe webhook: payment succeeded
                ├─ orders marked Paid
                ├─ commission and vendor payout calculated, wallets credited
                └─ shipment opened ─────────────────────────────► Pending
                           │
       vendors mark their orders Ready; when the last one is ready
                           ▼
                       Unassigned ── shipping managers notified
                           │
          manager assigns an available driver ── driver notified
                           ▼
                       Assigned ◄──── driver hands it back ──── (→ Unassigned)
                           │
           first order picked up (by the driver or the manager)
                           ▼
                       Picking Up
                           │
                    every order picked up
                           ▼
                    Out for Delivery
                           │
               customer gives the delivery code
                           ▼
                       Delivered
```

Every transition is checked in its handler:

- Only the assigned driver, or an admin, can move a shipment forward.
- **Out for Delivery** requires every order in the shipment to be `PickedUp`.
- **Delivered** requires the customer's delivery code.
- Each step records its time (`ReadyForPickupAt`, `TripStartedAt`, `OutForDeliveryAt`, `DeliveredAt`) and notifies the people involved.

**Concurrency.** Status changes are written as conditional updates: the `UPDATE` only matches the row if the shipment is still in the expected state, and the affected-row count is checked. If two managers assign the same shipment at once, exactly one succeeds and the other gets an "already assigned" error, without locks or lost updates.

```csharp
var rowsAffected = await _context.Shipments
    .Where(s => s.Id == request.ShipmentId && s.ShipmentStatus == ShipmentStatus.Unassigned)
    .ExecuteUpdateAsync(set => set
        .SetProperty(s => s.DriverId, request.DriverId)
        .SetProperty(s => s.ShipmentStatus, ShipmentStatus.Assigned), cancellationToken);

if (rowsAffected == 0)
    return Result.Failure(ShipmentErrors.AlreadyAssigned);
```

## Architecture

### Vertical slices

Code is grouped by feature rather than by technical layer. Each use case is a small folder holding everything it needs:

```
Features/Shipments/AdminAndManager/AssignShipment/
├── AssignShipmentEndPoint.cs          # HTTP endpoint: route, roles, maps the result to a response
├── DTOs/AssignShipmentRequestDTO.cs   # MediatR request + FluentValidation rules
├── Handlers/AssignShipmentHandler.cs  # business logic
└── AssignShipmentEvent.cs             # domain event raised on success
```

- **CQRS with MediatR**: each endpoint sends one command or query to one handler.
- **Result pattern**: handlers return `Result` / `Result<T>` with typed errors instead of throwing, and endpoints turn them into HTTP status codes.
- **FluentValidation** for request validation and **AutoMapper** for mapping.
- **Domain events**: handlers publish MediatR notifications that other features react to.

### Real-time notifications

```
Any feature handler ──publish──► domain event (INotificationEvent)
                                        │
                                        ▼
                          NotificationEventHandler<TEvent>   (generic, one for all events)
                            ├─ saves a notification per recipient
                            ├─ increments each user's undelivered count
                            └─ enqueues a Hangfire background job
                                        │
                                        ▼
                     SignalR hub ──► Clients.User(id).ReceiveNotification(...)
```

Adding a new notification type means declaring an event record. Storage, counting and delivery are handled once. Delivery runs as a background job, so the API request that raised the event never waits on SignalR.

## Tech stack

| Area | Technology |
|---|---|
| Framework | ASP.NET Core Web API (.NET 10) |
| Data | Entity Framework Core 10, SQL Server, LINQ, code-first migrations |
| Architecture | Vertical Slice, CQRS (MediatR), Result pattern, domain events |
| Validation and mapping | FluentValidation, AutoMapper |
| Auth | ASP.NET Core Identity, JWT bearer + refresh tokens, role-based authorization |
| Real-time | SignalR |
| Background jobs | Hangfire (SQL Server storage, protected dashboard) |
| Payments | Stripe Checkout + webhooks |
| Recommendations | External Python ML service over HTTP |
| Email | MailKit / MimeKit |
| API docs | OpenAPI + Swagger UI |
| CI/CD | GitHub Actions: build, publish and deploy on push to `Production` |

## Getting started

### Branches

| Branch | Purpose |
|---|---|
| `Development` | Integration branch; feature branches are merged here |
| `Production` | Deployed automatically by GitHub Actions |

### Requirements

- .NET 10 SDK
- SQL Server (LocalDB works)
- EF Core CLI: `dotnet tool install --global dotnet-ef`
- A Stripe test account, for checkout (optional for everything else)
- The recommendation service, for recommendations only (optional)

### 1. Clone

```bash
git clone https://github.com/AL70SSAIN/WearCast.git
cd WearCast/WearCast.Api
```

### 2. Configure

Secrets are left empty in `appsettings.json`. Set them with user secrets (the project already has a `UserSecretsId`) so they stay out of Git:

```bash
dotnet user-secrets set "ConnectionStrings:DefaultConnection"  "Server=(localdb)\mssqllocaldb;Database=WearCast;Trusted_Connection=True;TrustServerCertificate=True"
dotnet user-secrets set "ConnectionStrings:HangfireConnection" "Server=(localdb)\mssqllocaldb;Database=WearCast;Trusted_Connection=True;TrustServerCertificate=True"
dotnet user-secrets set "Jwt:Key"                    "<random string, at least 32 characters>"
dotnet user-secrets set "HangfireSettings:Username"  "admin"
dotnet user-secrets set "HangfireSettings:Password"  "<password>"

# Email (for confirmation and password reset)
dotnet user-secrets set "MailSettings:Mail"      "<sender email>"
dotnet user-secrets set "MailSettings:Password"  "<app password>"

# Stripe (test keys)
dotnet user-secrets set "StripeSettings:SecretKey"      "sk_test_..."
dotnet user-secrets set "StripeSettings:PublishableKey" "pk_test_..."
dotnet user-secrets set "StripeSettings:WebhookSecret"  "whsec_..."

# Recommendation service (optional)
dotnet user-secrets set "RecommendationServiceSettings:BaseUrl" "http://localhost:8000"
```

### 3. Create the database and run

```bash
dotnet ef database update
dotnet run
```

| URL | What |
|---|---|
| `/swagger` | Swagger UI, with every endpoint grouped by feature |
| `/jobs` | Hangfire dashboard (basic auth from `HangfireSettings`) |
| `/notificationHub` | SignalR hub for notifications |

To test payments locally, forward Stripe events with the [Stripe CLI](https://stripe.com/docs/stripe-cli):
`stripe listen --forward-to https://localhost:<port>/api/Webhook/Stripe`

## Project structure

```
WearCast.Api/
├── Features/              # one folder per feature, one sub-folder per use case
│   ├── AuthenticationManagement/
│   ├── FixedProduct/  FixedProductColor/  Category/  Favourites/
│   ├── DesignedProductManagement/
│   ├── CartManagment/  Checkout/  Orders/
│   ├── Sellers/  Factories/  Customers/  Admins/  Platform/
│   ├── ShippingCompanies/  Drivers/  Shipments/
│   ├── NotificationManagement/
│   └── UserTracking/
├── Entities/              # domain entities (BusinessActors, Shipping, Order, Wallet, ...)
├── Persistence/           # ApplicationDbContext, entity configurations, migrations
├── Common/                # result pattern, auth, email, wallet, tracking, external services
├── DependencyInjection.cs # service registration
└── Program.cs             # middleware pipeline
```

