# Kiến Trúc Hệ Thống & Cloud Services - Astrology App MVP

## TỔNG QUAN KIẾN TRÚC

### Architecture Pattern: **3-Tier Architecture với Microservices elements**

```
┌─────────────────────────────────────────────────────────────┐
│                      CLIENT LAYER                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   iOS App    │  │ Android App  │  │  Web Admin   │      │
│  │   (Swift)    │  │  (Kotlin)    │  │  (React)     │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
                            │
                     HTTPS / REST API
                            │
┌─────────────────────────────────────────────────────────────┐
│                   API GATEWAY LAYER                          │
│  ┌──────────────────────────────────────────────────────┐   │
│  │   API Gateway (Kong / AWS API Gateway / Nginx)       │   │
│  │   - Rate Limiting  - Authentication  - Load Balance  │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                            │
┌─────────────────────────────────────────────────────────────┐
│                    APPLICATION LAYER                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │ Auth Service │  │ User Service │  │ Chart Service│      │
│  │              │  │              │  │              │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │Horoscope Svc │  │Social Service│  │Payment Svc   │      │
│  │ (AI Engine)  │  │              │  │ (Stripe)     │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐                        │
│  │ Notification │  │ Content Mgmt │                        │
│  │   Service    │  │   Service    │                        │
│  └──────────────┘  └──────────────┘                        │
└─────────────────────────────────────────────────────────────┘
                            │
┌─────────────────────────────────────────────────────────────┐
│                      DATA LAYER                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  PostgreSQL  │  │    Redis     │  │   S3 Store   │      │
│  │  (Primary)   │  │   (Cache)    │  │  (Assets)    │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
```

---

## 1. TECH STACK CHI TIẾT

### 1.1 Frontend (Mobile Apps)

**Option A: Native Development (Recommended cho performance)**

| Platform    | Language    | UI Framework    | Key Libraries                  |
| ----------- | ----------- | --------------- | ------------------------------ |
| **iOS**     | Swift 5.9+  | SwiftUI + UIKit | Alamofire, Kingfisher, Charts  |
| **Android** | Kotlin 1.9+ | Jetpack Compose | Retrofit, Coil, MPAndroidChart |

**Option B: Cross-platform (Recommended cho budget)**

| Framework        | Language              | Advantages                                           | Disadvantages                             |
| ---------------- | --------------------- | ---------------------------------------------------- | ----------------------------------------- |
| **React Native** | JavaScript/TypeScript | Large community, Expo, Fast development              | Performance issues, Native modules needed |
| **Flutter**      | Dart                  | Excellent performance, Beautiful UI, Single codebase | Larger app size, Learning curve           |

**Khuyến nghị:** React Native cho MVP nếu budget hạn chế, Native nếu muốn performance tốt nhất.

---

### 1.2 Backend

**Core Stack:**
- **Runtime:** Go 1.22+ (Golang)
- **Framework:** **Echo Framework** (High performance, minimalist)
- **API Style:** RESTful API (gRPC cho internal services nếu cần)
- **Authentication:** JWT (Golang-jwt) + OAuth2
- **ORM/Query Builder:** SQLC (Type-safe) hoặc GORM (Feature-rich)

**Key Backend Modules:**

```go
// Example project structure với Echo Framework (Clean Architecture)
internal/
├── auth/                 // Middleware & Auth logic
├── users/                // User domain handlers
├── charts/               // Birth chart calculations
│   ├── astronomical/     // Swiss Ephemeris wrapper (CGO)
│   └── service/          // Zodiac business logic
├── horoscopes/           // Daily horoscope generation
│   └── ai/               // OpenAI client
├── social/               // Friends & compatibility
├── notifications/        // Push notifications (FCM)
├── payments/             // Stripe SDK
└── content/              // CMS & content management
pkg/
├── db/                   // Postgres driver (pgx/sqlc)
├── logger/               // Uber-zap / Zerolog
└── server/               // Echo server configuration & routes
cmd/
└── api/                  // Application entry point
```

---

### 1.3 Databases

**Primary Database: PostgreSQL 15+**
- **Why:** ACID compliance, JSON support, mature ecosystem
- **Schema Design:**

```sql
-- Core tables
users (id, email, name, birth_date, birth_time, birth_location, timezone, created_at)
birth_charts (id, user_id, chart_data_json, calculated_at)
friendships (id, user_id, friend_id, status, created_at)
compatibility_cache (id, user1_id, user2_id, score, analysis_json, cached_at)
horoscopes (id, user_id, date, content, type, created_at)
subscriptions (id, user_id, plan_type, status, stripe_id, expires_at)
```

**Cache Layer: Redis 7+**
- User sessions
- API rate limiting
- Frequently accessed data (horoscopes, chart interpretations)
- Real-time features (notifications queue)

**File Storage: S3-Compatible**
- User avatars
- Chart exports (PNG/PDF)
- Static assets
- Backup storage

---

### 1.4 Third-party Integrations

| Service                | Purpose                  | Provider Options                             | Cost                         |
| ---------------------- | ------------------------ | -------------------------------------------- | ---------------------------- |
| **Astronomical Data**  | Birth chart calculations | Swiss Ephemeris (self-hosted), NASA Horizons | Free                         |
| **AI Content**         | Horoscope generation     | OpenAI GPT-4, Anthropic Claude               | $0.03/1K tokens              |
| **Push Notifications** | Daily alerts             | Firebase Cloud Messaging, OneSignal          | Free tier: 10K/month         |
| **SMS/Email**          | OTP, newsletters         | Twilio, SendGrid                             | Free tier: 100 emails/day    |
| **Payment**            | Subscriptions            | Stripe, PayPal                               | 2.9% + $0.30/transaction     |
| **Analytics**          | User behavior            | Mixpanel, Amplitude                          | Free tier: 100K events/month |
| **Crash Reporting**    | Error tracking           | Sentry, Firebase Crashlytics                 | Free tier: 5K errors/month   |
| **Social Login**       | OAuth providers          | Google, Facebook, Apple                      | Free                         |

---

## 2. DỊCH VỤ CLOUD MIỄN PHÍ CHO MVP

### 2.1 Hosting & Compute - FREE TIER OPTIONS

#### **Option 1: Appwrite Cloud (Recommended - All-in-one)**
- **What:** Open-source Backend-as-a-Service
- **Free Tier:**
  - 75K monthly active users
  - 300GB bandwidth
  - 150GB storage
  - Authentication included
  - Database included
  - Functions (serverless) included
- **Pros:** 
  - Full-stack platform
  - Auth, DB, Storage all integrated
  - Generous free tier
  - Easy deployment
- **Cons:** 
  - Less flexibility vs custom backend
- **Best for:** Rapid MVP development
- **URL:** https://appwrite.io

---

#### **Option 2: Supabase (Recommended - Firebase alternative)**
- **What:** Open-source Firebase alternative (Postgres-based)
- **Free Tier:**
  - 500MB database storage
  - 1GB file storage
  - 2GB bandwidth
  - 50K monthly active users
  - Authentication built-in
  - Real-time subscriptions
  - Edge functions
- **Pros:**
  - PostgreSQL (powerful queries)
  - Row-level security
  - Auto-generated REST API
  - Great for rapid development
- **Cons:**
  - Free tier limits strict
- **Best for:** MVP với standard features
- **URL:** https://supabase.com

---

#### **Option 3: Railway (Recommended - Full backend)**
- **What:** Platform-as-a-Service for deploying apps
- **Free Tier:**
  - $5 free credits/month
  - Deploy Node.js apps + Postgres
  - Custom domains
  - Auto-scaling
- **Pros:**
  - Full control over backend code
  - Support multiple services
  - Easy deployment from GitHub
- **Cons:**
  - Credits run out quickly with heavy usage
- **Best for:** Custom backend deployment
- **URL:** https://railway.app

---

#### **Option 4: Render (Alternative)**
- **What:** Cloud platform for apps
- **Free Tier:**
  - 750 hours/month free
  - PostgreSQL database (expires 90 days)
  - Static site hosting
  - Auto-deploy from Git
- **Pros:**
  - Zero config deployment
  - Good for Node.js apps
- **Cons:**
  - Free tier spins down after inactivity (slow cold starts)
- **URL:** https://render.com

---

#### **Option 5: Vercel (For Web Admin only)**
- **Free Tier:**
  - 100GB bandwidth
  - Serverless functions
  - Edge network
  - Perfect for Next.js apps
- **Best for:** Admin dashboard web app
- **URL:** https://vercel.com

---

### 2.2 Database - FREE OPTIONS

#### **Option 1: Neon (Recommended - Serverless Postgres)**
- **Free Tier:**
  - 0.5GB storage
  - Unlimited databases
  - Auto-pause when inactive (saves costs)
  - Branching feature (for staging)
- **Pros:**
  - Serverless Postgres (no maintenance)
  - Fast cold starts
  - Great developer experience
- **URL:** https://neon.tech

#### **Option 2: PlanetScale (MySQL alternative)**
- **Free Tier:**
  - 5GB storage
  - 1 billion row reads/month
  - Branching workflows
- **Note:** MySQL-based (not Postgres)
- **URL:** https://planetscale.com

#### **Option 3: Supabase Database**
- Included in Supabase free tier (see above)

---

### 2.3 Storage - FREE OPTIONS

#### **Option 1: Cloudflare R2 (S3-compatible)**
- **Free Tier:**
  - 10GB storage
  - 10 million Class A operations/month
  - Zero egress fees (huge advantage!)
- **Best for:** User avatars, chart exports
- **URL:** https://www.cloudflare.com/products/r2/

#### **Option 2: Supabase Storage**
- **Free Tier:**
  - 1GB storage
  - Integrated with Supabase
- **URL:** https://supabase.com/storage

#### **Option 3: Backblaze B2**
- **Free Tier:**
  - 10GB storage
  - 1GB daily download
- **URL:** https://www.backblaze.com/b2/

---

### 2.4 AI Services - FREE/LOW COST

#### **Option 1: OpenAI GPT-4 API**
- **Pricing:** Pay-as-you-go
  - GPT-4o: $2.50 per 1M input tokens, $10 per 1M output tokens
  - GPT-4o-mini: $0.15 per 1M input tokens, $0.60 per 1M output tokens
- **Estimated cost:** ~$50-200/month cho MVP (1000 daily horoscopes)
- **Best for:** High-quality content generation
- **URL:** https://platform.openai.com

#### **Option 2: Anthropic Claude API**
- **Pricing:** Similar to OpenAI
- **Pros:** Better at nuanced writing, longer context
- **URL:** https://www.anthropic.com

#### **Option 3: Open-source LLMs (Self-hosted)**
- **Options:** Llama 3, Mistral
- **Pros:** No per-request costs
- **Cons:** Need compute resources, lower quality
- **Best for:** Extreme budget constraints

---

### 2.5 Authentication - FREE OPTIONS

#### **Option 1: Supabase Auth**
- Included free
- Email/password + OAuth (Google, Facebook, Apple)
- JWT tokens

#### **Option 2: Auth0 (Okta)**
- **Free Tier:**
  - 7,500 monthly active users
  - Social + email login
- **URL:** https://auth0.com

#### **Option 3: Firebase Auth**
- **Free Tier:**
  - Unlimited users
  - Email + social login
- **URL:** https://firebase.google.com/products/auth

---

### 2.6 Notifications - FREE OPTIONS

#### **Option 1: Firebase Cloud Messaging (FCM)**
- **Free Tier:** Unlimited messages
- **Pros:** Best for mobile push notifications
- **Cons:** Requires Firebase SDK
- **URL:** https://firebase.google.com/products/cloud-messaging

#### **Option 2: OneSignal**
- **Free Tier:**
  - 10,000 subscribers
  - Unlimited notifications
- **Pros:** Multi-platform support, easy integration
- **URL:** https://onesignal.com

---

### 2.7 Email/SMS - FREE OPTIONS

#### **Option 1: SendGrid**
- **Free Tier:** 100 emails/day (3,000/month)
- **URL:** https://sendgrid.com

#### **Option 2: Brevo (formerly Sendinblue)**
- **Free Tier:** 300 emails/day
- **URL:** https://www.brevo.com

#### **Option 3: Twilio (SMS)**
- **Free Trial:** $15 credit
- **URL:** https://www.twilio.com

---

### 2.8 Analytics - FREE OPTIONS

#### **Option 1: PostHog**
- **Free Tier:**
  - 1M events/month
  - Product analytics + session recording
  - Feature flags
- **URL:** https://posthog.com

#### **Option 2: Mixpanel**
- **Free Tier:**
  - 100K monthly tracked users
- **URL:** https://mixpanel.com

#### **Option 3: Google Analytics 4**
- **Free:** Unlimited
- **URL:** https://analytics.google.com

---

### 2.9 Monitoring & Error Tracking - FREE OPTIONS

#### **Option 1: Sentry**
- **Free Tier:**
  - 5K errors/month
  - 10K performance units
- **URL:** https://sentry.io

#### **Option 2: LogRocket**
- **Free Tier:**
  - 1K sessions/month
- **URL:** https://logrocket.com

---

## 3. KIẾN TRÚC ĐÁNH GIÁ & KHUYẾN NGHỊ

### 3.1 Kiến trúc Recommended cho MVP

**Stack A: Budget-Friendly (All-Free Tier)**

```
Frontend: React Native (Cross-platform)
Backend: Supabase (BaaS)
Database: Supabase Postgres (500MB)
Storage: Cloudflare R2 (10GB)
Auth: Supabase Auth
AI: OpenAI GPT-4o-mini ($50-100/month)
Push: Firebase Cloud Messaging (Free)
Email: Brevo (300/day free)
Analytics: PostHog (1M events)
Monitoring: Sentry (5K errors)
Hosting: Vercel (admin web)

Total monthly cost: ~$50-150 (mainly AI)
```

**Pros:**
- Minimal infrastructure management
- Fast development (BaaS reduces backend work)
- Proven free tiers
- Easy to scale

**Cons:**
- Less flexibility với Supabase
- Vendor lock-in
- Free tier limitations

---

**Stack B: Custom Go (Echo) Backend**

```
Frontend: React Native or Flutter
Backend: Golang (Echo Framework) on Railway ($5/month free credit)
Database: Neon Postgres (0.5GB)
Storage: Cloudflare R2 (10GB)
Cache: Upstash Redis (10K commands/day free)
Auth: Custom JWT + OAuth (Golang-jwt)
AI: OpenAI GPT-4o-mini
Push: OneSignal (10K subscribers)
Email: SendGrid (100/day)
Analytics: PostHog
Monitoring: Sentry (Go SDK)

Total monthly cost: ~$50-200
```

**Pros:**
- Echo is minimalist, very fast and widely supported
- Easy to build concurrent & scalable systems
- Self-contained binary (easy deployment)

**Cons:**
- Longer development time vs BaaS (Supabase)
- Steep learning curve for junior devs
- Railway credits run out with high traffic
- More maintenance needed v.s Supabase
```

---

**Stack C: Native Apps & Go Microservices (Premium Quality)**

```
Frontend: Swift (iOS) + Kotlin (Android)
Backend: Golang Microservices on Google Cloud Run
Database: Supabase or Neon Postgres
Storage: Cloudflare R2
Auth: Firebase Auth
AI: OpenAI GPT-4
Push: Firebase Cloud Messaging
Payment: Stripe
Analytics: Mixpanel
Monitoring: Sentry + Firebase Crashlytics

Total monthly cost: ~$150-400
```

**Pros:**
- Best performance & user experience
- Full native features
- Professional feel

**Cons:**
- Requires 2 separate dev teams
- Longer development time
- Higher maintenance

---

### 3.2 Khuyến nghị cuối cùng cho MVP

**🎯 RECOMMENDED STACK (Balanced):**

| Component              | Service                      | Why                                                              |
| ---------------------- | ---------------------------- | ---------------------------------------------------------------- |
| **Mobile App**         | **React Native + Expo**      | Cross-platform, faster MVP, one codebase                         |
| **Backend**            | **Supabase**                 | BaaS giảm development time 40%, auth built-in, Postgres powerful |
| **Database**           | **Supabase Postgres**        | Included, row-level security, auto REST API                      |
| **Storage**            | **Cloudflare R2**            | 10GB free, zero egress fees                                      |
| **AI Engine**          | **OpenAI GPT-4o-mini**       | Best price/performance ($0.15 per 1M tokens)                     |
| **Push Notifications** | **Firebase Cloud Messaging** | Unlimited free, reliable                                         |
| **Email**              | **Brevo**                    | 300/day free (enough for MVP)                                    |
| **Analytics**          | **PostHog**                  | 1M events free, includes session replay                          |
| **Monitoring**         | **Sentry**                   | 5K errors free, great DX                                         |
| **Payment**            | **Stripe**                   | Industry standard, great API                                     |
| **Admin Dashboard**    | **React + Vercel**           | 100GB bandwidth free                                             |

**Total Monthly Cost:**
- Development phase: ~$50-100 (mainly AI)
- With 1,000 active users: ~$100-200
- Infrastructure: FREE trong 6 tháng đầu

**Development Time:** 8-10 tuần với 5-7 người

---

## 4. SECURITY & COMPLIANCE

### 4.1 Security Measures

```
┌─────────────────────────────────────────┐
│         Security Layers                  │
├─────────────────────────────────────────┤
│ 1. HTTPS/TLS Everywhere (Cloudflare)   │
│ 2. JWT Authentication (Short-lived)     │
│ 3. Rate Limiting (API Gateway)          │
│ 4. Input Validation (Backend)           │
│ 5. SQL Injection Prevention (ORM)       │
│ 6. XSS Protection (Content Security)    │
│ 7. CORS Configuration                    │
│ 8. Data Encryption at Rest (DB)         │
│ 9. Secure Payment (Stripe PCI)          │
│ 10. DDoS Protection (Cloudflare)        │
└─────────────────────────────────────────┘
```

### 4.2 Compliance

**GDPR (EU users):**
- User consent for data collection
- Right to data export
- Right to be forgotten (delete account)
- Privacy policy clear & accessible

**App Store Guidelines:**
- Privacy manifest (iOS 17+)
- Data usage declarations
- Age rating appropriate
- No misleading claims

**Payment Security:**
- Never store credit card details
- Use Stripe Elements (PCI compliant)
- Secure webhook handling

---

## 5. SCALABILITY PLAN

### Phase 1: MVP (0-1K users)
- Free tier services đủ dùng
- Single backend instance
- No caching needed
- Cost: ~$50-100/month

### Phase 2: Growth (1K-10K users)
- Upgrade Supabase to Pro ($25/month)
- Add Redis caching (Upstash Pro $10/month)
- CDN for static assets (Cloudflare free)
- Cost: ~$200-400/month

### Phase 3: Scale (10K-100K users)
- Migrate to dedicated servers (AWS/GCP)
- Database read replicas
- Load balancing
- Microservices separation
- Cost: ~$1,000-3,000/month

---

## 6. DISASTER RECOVERY & BACKUP

**Backup Strategy:**
1. **Database:** 
   - Supabase: Auto daily backups (free tier: 7 days retention)
   - Manual weekly exports to S3
   
2. **File Storage:**
   - Cloudflare R2: Versioning enabled
   - Monthly full backups

3. **Code:**
   - GitHub repository (private)
   - Tag releases

4. **Secrets:**
   - 1Password Teams / Vault
   - Never commit secrets

**Recovery Plan:**
- RTO (Recovery Time Objective): < 4 hours
- RPO (Recovery Point Objective): < 24 hours
- Runbook documented in Notion/Confluence

---

## TÓM TẮT & NEXT STEPS

### ✅ Kiến trúc MVP đã chọn:
- **Frontend:** React Native (cross-platform)
- **Backend:** Supabase (BaaS)
- **AI:** OpenAI GPT-4o-mini
- **Storage:** Cloudflare R2
- **Notifications:** Firebase Cloud Messaging

### 💰 Chi phí infrastructure MVP:
- Tháng 1-3: **$50-150/month** (chủ yếu AI)
- Tháng 4-6: **$100-300/month** (khi có users)
- Total 6 tháng: **~$600-1,200**

### 📋 Setup checklist:
1. ☐ Tạo Supabase project
2. ☐ Thiết kế database schema
3. ☐ Setup Cloudflare R2 bucket
4. ☐ Tạo OpenAI API key
5. ☐ Setup Firebase project cho FCM
6. ☐ Cấu hình Stripe account
7. ☐ Tạo Sentry project
8. ☐ Setup PostHog analytics
9. ☐ Configure CI/CD (GitHub Actions)
10. ☐ Domain & SSL setup (Cloudflare)

### 🚀 Development workflow:
```
Local Dev → Git Push → GitHub Actions → 
Deploy to Staging (Vercel preview) → 
QA Testing → Deploy to Production
```
