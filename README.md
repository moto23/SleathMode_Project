# StealthMode

### Full-Stack E-Learning Marketplace

StealthMode is a full-stack e-learning marketplace where users can discover courses, create accounts, purchase individual or multiple courses via Razorpay, and manage their enrolled learning content.

The platform includes JWT-based authentication, role-based access control, course management, cart-based multi-course checkout, server-authoritative pricing, payment verification, enrollment management, a responsive frontend, reusable UI components, and an admin course-management system.

**Live app:** [stealthmode-frontend.vercel.app](https://stealthmode-frontend.vercel.app)

---

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Payment System](#payment-system)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [API Reference](#api-reference)
- [Security](#security)
- [Design System](#design-system)
- [Engineering Principles](#engineering-principles)
- [Future Improvements](#future-improvements)
- [Author](#author)

---

## Overview

StealthMode provides an end-to-end course marketplace experience:

- User registration, login, and JWT-based authentication
- Role-based access control (user / admin)
- Course discovery with search, category filtering, and sorting
- Detailed course pages with "Buy Now" and "Add to Cart"
- Multi-course cart checkout
- Razorpay payments with server-side verification
- Course enrollment and a "My Learning" profile
- Admin course management
- Responsive UI with loading, empty, and error states
- Accessibility-focused interactions
- A cohesive espresso / warm-cream / amber / bronze visual design system

---

## Features

### Authentication
- User registration & login
- JWT authentication with protected routes
- Role-based authorization (admin-only functionality)
- Password visibility controls

### Course Marketplace
- Rich course cards (title, description, category, level, instructor, duration, pricing, discounts)
- Client-side search across title, description, instructor, and category
- Category filtering and multiple sort options (recommended, newest, price, alphabetical)
- Match/result counts and "clear filters"

### Course Details
- Dedicated course-detail page with a responsive purchase panel
- Add to Cart / Buy Now / purchased-state detection

### Cart
- Add/remove/clear courses, with owned-course exclusion
- Subtotal, discount, and final total calculation
- Multi-course checkout
- Sends course IDs (not client-calculated totals) to the backend

### Profile / My Learning
- Enrolled courses displayed as cards with a "Continue Learning" CTA
- Loading, empty, and error states
- Graceful handling of deleted/missing course references

### Admin Course Management
- Create, edit, and delete courses
- JWT + admin-role protected APIs
- Explicit delete confirmation workflow (shows exact course title, defaults focus to Cancel, requires an explicit destructive action)

---

## Payment System

StealthMode integrates **Razorpay Standard Checkout** with a fully server-authoritative payment architecture — the frontend never determines the final payable amount.

### Flow

```
Course selection (Buy Now / Cart)
        │
        ▼
Frontend sends course ID(s)
        │
        ▼
Backend validates user & fetches courses from MongoDB
        │
        ▼
Backend calculates authoritative total from DB prices
        │
        ▼
Razorpay Order created (server-side)
        │
        ▼
Order/Payment persisted in MongoDB
        │
        ▼
Razorpay Checkout opens on the frontend
        │
        ▼
Payment completed → callback sent to backend
        │
        ▼
Backend verifies HMAC-SHA256 signature (timing-safe)
        │
        ▼
Payment/Order marked paid → Enrollment(s) created
```

The same server-authoritative approach applies to both single-course ("Buy Now") and multi-course (cart) checkout.

### Payment Security
- Server-side Razorpay order creation
- Server-authoritative pricing (client sends course IDs, not amounts)
- HMAC-SHA256 signature verification with timing-safe comparison
- Idempotent enrollment on `(userId, courseId)`
- Protected payment routes (require authentication; admin routes require admin role)
- No Razorpay secrets or order-creation logic in frontend code

### Webhook Hardening (Implemented, Deferred from Production)

A complete Razorpay webhook reconciliation system has been built and tested locally, but is intentionally **not enabled in production** since the current deployment is serverless without a durable background queue/worker — and Razorpay's guidance recommends acknowledging webhooks quickly and processing them asynchronously via durable infrastructure. It remains a designed and validated reliability layer for when that infrastructure is in place.

Capabilities include:
- Raw-body HMAC-SHA256 verification of `X-Razorpay-Signature`
- Event-level idempotency (`WebhookEvent` + unique `eventId`) and enrollment-level idempotency
- Configurable replay-window protection (default 300s)
- Amount/currency reconciliation against stored order data
- Out-of-order event handling (`payment.captured`, `order.paid`, `payment.failed`) without downgrading an already-paid record

---

## Architecture

### Backend

Modular MVC / service-oriented architecture:

```
Backend
├── Routes
├── Controllers
├── Services       (auth, courses, payments, orders, enrollment, webhooks)
├── Models          (MongoDB)
├── Middleware      (JWT validation, role authorization)
└── Scripts
```

### Frontend

React application using React Router and the Context API for shared state:

```
React Application
├── App / Routes
├── Context
│   ├── UserContext
│   ├── CartContext
│   └── ToastContext
├── Components
├── Pages
├── Services        (Axios API client)
└── CSS
```

**Main routes:** `/` (landing), `/dashboard` (marketplace), `/enroll/:id` (course detail), `/cart`, `/profile` (My Learning), `/admin/courses`, plus login/registration.

**Reusable UI primitives:** `Skeleton`, `EmptyState`, `ErrorState`, `Toast`.

---

## Tech Stack

| Layer | Technologies |
|---|---|
| Frontend | React.js, React Router v6, Context API, Axios, Create React App |
| Backend | Node.js, Express.js, REST APIs, MVC + service-layer architecture |
| Auth | JWT, RBAC |
| Database | MongoDB |
| Payments | Razorpay Standard Checkout, Razorpay Orders API, HMAC-SHA256 |
| Communication | REST, Axios, EmailJS (contact form) |
| Deployment | Vercel (frontend & backend) |

---

## Project Structure

```
StealthMode/
├── frontend/
│   ├── src/
│   │   ├── components/
│   │   │   ├── home/        # Main, Home, Dashboard, CourseCard, Enroll, Cart, Profile
│   │   │   ├── admin/        # AdminCourses
│   │   │   └── ui/           # Skeleton, EmptyState, ErrorState
│   │   ├── context/          # UserContext, CartContext, ToastContext
│   │   ├── services/         # api.js, checkout.js, price.js
│   │   ├── styles/           # tokens.css
│   │   └── App.js
│   └── package.json
├── backend/
│   ├── controllers/
│   ├── middleware/
│   ├── models/
│   ├── routes/
│   ├── services/
│   ├── scripts/
│   ├── server.js
│   └── package.json
└── README.md
```

---

## API Reference

| Endpoint | Description |
|---|---|
| `POST /api/payments/create-order` | Create a Razorpay order for a single course |
| `POST /api/payments/verify` | Verify payment signature for a single-course purchase |
| `POST /api/payments/cart/create-order` | Create a Razorpay order for cart checkout |
| `POST /api/payments/cart/verify` | Verify payment signature for cart checkout |
| `GET /api/payments/owned` | Retrieve courses owned by the authenticated user |
| `GET /api/payments/status/:id` | Check payment/order status |
| `POST /api/payments/webhook` | *(implemented locally, deferred)* Razorpay server-to-server event handler |

All payment endpoints require authentication; unauthenticated requests are rejected. Admin endpoints additionally require an admin role.

---

## Security

- Server-side Razorpay order creation and signature verification — the backend never trusts a client-supplied payment amount
- Secrets (`RAZORPAY_KEY_SECRET`, `JWT_SECRET`, `MONGODB_URI`, `RAZORPAY_WEBHOOK_SECRET`) are kept server-side only and are never committed
- No Razorpay key secrets, order-creation calls, or `new Razorpay(...)` instantiation in frontend code
- JWT-based authentication and role-based authorization on protected and admin routes
- Idempotent enrollment logic to prevent duplicate course access

### Environment Variables (backend)

```env
MONGODB_URI=...
JWT_SECRET=...
RAZORPAY_KEY_ID=...
RAZORPAY_KEY_SECRET=...

# Webhook (implemented locally, not yet enabled in production)
RAZORPAY_WEBHOOK_SECRET=...
RAZORPAY_WEBHOOK_MAX_AGE_SECONDS=300
```

> Never commit `.env`, `.env.local`, or `.env.production` files.

---

## Design System

A centralized design-token system (`styles/tokens.css`) drives a cohesive **espresso, warm cream, amber, bronze, and selective glass** visual language across the landing page, dashboard, course details, cart, profile, and admin views. Glassmorphism is used selectively on major surfaces rather than applied uniformly, and content-heavy areas favor more opaque surfaces for readability.

Accessibility considerations include accessible form labels, `aria-live` toast notifications, `role="alert"` for errors, visible focus states, keyboard-friendly interactions, and reduced-motion/reduced-transparency support.

The landing page intentionally avoids unsupported marketing claims (student counts, ratings, testimonials, or achievements) — only real course data is displayed.

---

## Engineering Principles

- **Server-authoritative pricing** — never trust frontend payment totals
- **Server-side verification** — never grant paid access from browser data alone
- **Separation of concerns** — payment logic isolated from presentation
- **Idempotency** — repeated payment/enrollment events never create duplicate access
- **RBAC** — administrative operations require explicit authorization
- **Progressive enhancement** — frontend improvements ship without altering stable payment contracts
- **Production safety** — payment-critical files (`services/checkout.js`, `services/price.js`, `context/CartContext.js`, `context/UserContext.js`) are isolated from visual/UI redesign work

---

## Future Improvements

- Durable webhook queue/worker and production webhook enablement
- Server-side course search & pagination as the catalog grows
- Persistent wishlist and course progress tracking
- Curriculum modules and instructor profiles
- Reviews and ratings backed by real user data
- Advanced analytics and automated monitoring

---

## Author

**Prasad Nathe**
Software Engineer

StealthMode — Full-Stack E-Learning Marketplace
