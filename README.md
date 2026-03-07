# EdisonKart

A full-stack e-commerce platform built with React, Node.js, and MongoDB. Features admin panel, vendor portal, delivery management, payments (Razorpay & Cashfree), AI-powered product import, image search, and more.

**Live:** [edisonkart.com](https://edisonkart.com)

---

## Features

- **User**
  - Auth (email/password, OTP, Google OAuth)
  - Browse products, search, filters
  - Cart, wishlist, compare products
  - Checkout (Razorpay, Cashfree)
  - Order history & tracking
  - Profile, addresses, loyalty/referrals

- **Admin**
  - Dashboard & analytics
  - Products, categories, banners, coupons
  - Orders, returns, users
  - Settings, contact submissions

- **Vendor**
  - Product management
  - Order management

- **Delivery**
  - Assigned orders
  - Status updates, completed orders

- **AI**
  - Image-based product search (OpenAI Vision)
  - Product import & description rewrite (OpenAI)
  - Gemini API integration

- **Other**
  - Live chat
  - Q&A, reviews, notifications
  - Pincode serviceability
  - SEO routes

---

## Tech Stack

| Layer      | Tech                                      |
|-----------|--------------------------------------------|
| Frontend  | React 18, Vite, Tailwind CSS, Zustand, TanStack Query |
| Backend   | Node.js, Express, Mongoose                 |
| Mobile    | React Native, Expo Router                  |
| Database  | MongoDB (GridFS for media)                 |
| Payments  | Razorpay, Cashfree                         |
| AI        | OpenAI (GPT-4o-mini), Google Gemini       |

---

## Project Structure

```
Edisonkart/
├── Edisonkart-backend/     # Node.js API
├── Edisonkart-frontend/    # React web app
├── Edisonkart-mobile/      # Expo mobile app
├── deploy-razorpay-on-live-server.sh
└── nginx-api-proxy.conf
```

---

## Prerequisites

- Node.js 18+
- MongoDB (local or Atlas)
- npm or yarn

---

## Quick Start

### 1. Clone

```bash
git clone https://github.com/kowshik538/edisonkart.git
cd edisonkart
```

### 2. Backend

```bash
cd Edisonkart-backend
cp .env.example .env
# Edit .env with your values
npm install
npm run dev
```

Runs at `http://localhost:5000`.

### 3. Frontend

```bash
cd Edisonkart-frontend
cp .env.example .env
# Set VITE_API_URL=http://localhost:5000/api
npm install
npm run dev
```

Runs at `http://localhost:5173`.

### 4. Mobile (optional)

```bash
cd Edisonkart-mobile
npm install
npx expo start
```

---

## Environment Variables

### Backend (`Edisonkart-backend/.env`)

| Variable          | Description                    |
|-------------------|--------------------------------|
| PORT              | Server port (default: 5000)    |
| MONGODB_URI       | MongoDB connection string      |
| JWT_SECRET        | JWT signing secret             |
| EMAIL_*           | SMTP (Gmail) for OTP           |
| RAZORPAY_KEY_*    | Razorpay keys                  |
| OPENAI_API_KEY    | OpenAI API key (image search, import) |
| GEMINI_API_KEY    | Google Gemini API key          |
| GOOGLE_CLIENT_ID  | Google OAuth client ID         |
| FRONTEND_URL      | Allowed CORS origins           |

See `.env.example` for full list.

### Frontend (`Edisonkart-frontend/.env`)

| Variable             | Description              |
|----------------------|--------------------------|
| VITE_API_URL         | Backend API base URL     |
| VITE_GOOGLE_CLIENT_ID| Google OAuth client ID   |

---

## Scripts

### Backend

```bash
npm start       # Production
npm run dev     # Development (nodemon)
npm run seed    # Seed DB (admin, categories, etc.)
```

### Frontend

```bash
npm run dev     # Dev server
npm run build   # Production build
npm run preview # Preview build
```

---

## API Overview

| Base Path   | Purpose                      |
|-------------|------------------------------|
| /api/auth   | Auth, OTP, OAuth             |
| /api/products | Products, import, image search |
| /api/cart   | Cart                         |
| /api/orders | Orders                       |
| /api/payments | Payments                  |
| /api/admin  | Admin endpoints              |
| /api/delivery | Delivery management       |
| /api/reviews | Reviews                    |
| /api/wishlist | Wishlist                  |
| /api/chat   | Chat                         |
| ...         | See `src/routes.js`          |

---

## License

MIT
