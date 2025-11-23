# SNOW.md - New API Project Documentation

## Project Name
**New API** - 新一代大模型网关与AI资产管理系统 (Next-Generation LLM Gateway & AI Asset Management System)

## Overview

New API is an open-source, next-generation AI gateway and asset management system built on top of [One API](https://github.com/songquanpeng/one-api). It serves as a comprehensive platform for managing multiple AI model providers and providing unified API interfaces. The project supports a wide range of AI models including OpenAI (GPT series), Claude, Google Gemini, AWS Bedrock, and many other LLM providers.

This system is designed to be a centralized hub for AI operations, offering features like channel management, billing, user authentication, request routing, token counting, and real-time monitoring. It includes both a powerful backend API and a modern web-based dashboard for administration and user management.

## Technology Stack

### Language & Runtime
- **Go** 1.25.1 - Backend server implementation
- **JavaScript/React** - Frontend UI with Vite bundler
- **Node.js (Bun)** - Frontend package management and build tool

### Key Backend Frameworks & Libraries
- **Gin** - HTTP web framework for REST API routing and middleware
- **GORM** - ORM for database abstraction (supports SQLite, MySQL, PostgreSQL)
- **JWT** (`golang-jwt`) - Token-based authentication
- **Redis** (`go-redis`) - Caching and distributed data storage
- **WebAuthn** (`go-webauthn`) - Passwordless authentication support

### Database Support
- **SQLite** - Default local database (embedded in container)
- **MySQL** ≥ 5.7.8 - Remote database option
- **PostgreSQL** ≥ 9.6 - Alternative remote database

### Frontend Stack
- **React** - UI component framework
- **Vite** - Fast build tool and dev server
- **Tailwind CSS** - Utility-first CSS framework
- **i18next** - Internationalization (Chinese, English, French, Japanese)

### Key Dependencies (Selected)
- **Authentication**: JWT, WebAuthn, OIDC support
- **Payment Integration**: Stripe, Custom payment gateways (易支付)
- **Audio Processing**: MP3, FLAC, OGG Vorbis, WAV support
- **Video Processing**: MP4 codec support
- **Media**: Image processing, audio/video transcoding
- **Cloud**: AWS SDK for Bedrock integration
- **Utilities**: UUID generation, encryption, compression

## Project Structure

```
new-api/
├── main.go                          # Application entry point
├── go.mod / go.sum                  # Go module dependencies
├── Dockerfile / docker-compose.yml  # Container configuration
├── makefile                         # Build automation
├── VERSION                          # Version file
├── LICENSE                          # AGPLv3 + Commercial dual licensing
│
├── web/                             # Frontend React application
│   ├── src/
│   │   ├── components/              # React components
│   │   ├── pages/                   # Page components
│   │   ├── utils/                   # Utility functions
│   │   └── i18n/                    # Internationalization
│   ├── public/                      # Static assets
│   ├── package.json                 # Frontend dependencies
│   ├── vite.config.js               # Vite configuration
│   ├── tailwind.config.js           # Tailwind CSS config
│   └── i18next.config.js            # i18n configuration
│
├── controller/                      # HTTP request handlers
│   ├── relay.go                     # API relay/proxy logic
│   ├── channel.go                   # Channel management
│   ├── user.go                      # User management
│   ├── token.go                     # Token operations
│   ├── billing.go                   # Billing/quota management
│   ├── model.go                     # Model configuration
│   ├── image.go                     # Image processing
│   ├── audio.go                     # Audio processing
│   └── [40+ controller files]       # Additional features (payments, auth, etc.)
│
├── service/                         # Business logic layer
│   ├── channel.go                   # Channel service logic
│   ├── http.go                      # HTTP utilities
│   ├── quota.go                     # Quota/billing calculations
│   ├── convert.go                   # Format conversion
│   ├── token_counter.go             # Token counting
│   ├── error.go                     # Error handling
│   └── [20+ service files]          # Additional services
│
├── model/                           # Data models & database
│   ├── main.go                      # Database initialization
│   ├── user.go                      # User model
│   ├── channel.go                   # Channel model
│   ├── token.go                     # Token model
│   ├── ability.go                   # Model abilities
│   ├── pricing.go                   # Pricing configuration
│   ├── log.go                       # Logging model
│   └── [20+ model files]            # Additional entities
│
├── router/                          # Route definitions
│   ├── main.go                      # Router setup
│   ├── api-router.go                # API routes
│   ├── relay-router.go              # Relay/proxy routes
│   ├── web-router.go                # Web UI routes
│   ├── dashboard.go                 # Dashboard routes
│   └── video-router.go              # Video proxy routes
│
├── middleware/                      # HTTP middleware
│   ├── request-id.go                # Request tracking
│   ├── logging.go                   # Request/response logging
│   └── [other middleware]
│
├── common/                          # Shared utilities
│   ├── env.go                       # Environment variable loading
│   ├── logging.go                   # Logging utilities
│   ├── config.go                    # Configuration management
│   └── [utilities]
│
├── constant/                        # Application constants
├── dto/                             # Data transfer objects
├── types/                           # Type definitions
├── logger/                          # Logging setup
├── setting/                         # Configuration settings
├── relay/                           # Relay logic (proxy)
├── docs/                            # Documentation
├── bin/                             # Binary utilities & migrations
├── electron/                        # Electron app (desktop)
│
├── .env.example                     # Environment variables template
├── README.md / README.*.md          # Multi-language documentation
└── .snow/                           # Snow CLI metadata
```

## Key Features

### Core Gateway Features
- ✅ **Unified API Gateway** - Proxy requests to multiple AI providers through a single endpoint
- ✅ **Channel Management** - Configure and manage multiple API keys and service channels
- ✅ **Smart Routing** - Weighted random distribution, automatic failover, retry logic
- ✅ **Rate Limiting** - User-level and model-level request throttling
- ✅ **Request/Response Logging** - Detailed audit trails for all API calls

### Model Support
- ✅ **OpenAI** - GPT-4, GPT-3.5, and all variants with reasoning effort support
- ✅ **Claude** - Anthropic Claude messages API with thinking mode
- ✅ **Google Gemini** - Gemini Chat API with thinking capabilities
- ✅ **Azure OpenAI** - Azure-hosted OpenAI models
- ✅ **AWS Bedrock** - AWS foundation models
- ✅ **Other Providers** - DeepSeek, Qwen, Cohere, and 50+ more models
- ✅ **Midjourney** - Image generation via Midjourney Proxy
- ✅ **Suno** - Music generation

### API Format Support
- ✅ **OpenAI Chat Completions** - Compatible with OpenAI API
- ✅ **OpenAI Realtime** - Real-time audio/video communication
- ✅ **Claude Messages** - Anthropic API format
- ✅ **Google Gemini** - Gemini API format
- ✅ **Format Conversion** - Automatic conversion between OpenAI ↔ Claude ↔ Gemini

### Billing & Quota Management
- ✅ **Token-based Billing** - Flexible pricing per model
- ✅ **Cache-aware Billing** - Separate pricing for cache hits
- ✅ **Usage Tracking** - Detailed quota tracking and consumption history
- ✅ **Quota Enforcement** - Hard limits and soft limits per user
- ✅ **Online Recharge** - Support for Stripe, 易支付, custom payment gateways
- ✅ **Redemption Codes** - Promotional and gift codes

### User Management & Authentication
- ✅ **OAuth Integration** - Discord, Telegram, LinuxDO, GitHub, WeChat
- ✅ **OIDC Support** - OpenID Connect for enterprise SSO
- ✅ **Passkeys** - Passwordless authentication (WebAuthn)
- ✅ **Two-Factor Authentication** - TOTP/SMS support
- ✅ **Role-Based Access** - Admin, user, channel manager roles

### Administrative Features
- ✅ **Visual Dashboard** - Real-time statistics and monitoring
- ✅ **Channel Management** - Add, edit, test channels
- ✅ **Model Configuration** - Manage model abilities and pricing
- ✅ **User Management** - Create, edit, delete users
- ✅ **System Settings** - Configure rates, limits, notification settings
- ✅ **Batch Operations** - Bulk updates and migrations
- ✅ **Automated Updates** - Channel health checks and model sync

### Developer Features
- ✅ **API Documentation** - Interactive API documentation
- ✅ **Webhook Support** - Event notifications for quota changes
- ✅ **Token Counting** - Accurate token calculation
- ✅ **Debug Tools** - pprof performance profiling
- ✅ **Request Tracking** - Request ID correlation

## Getting Started

### Prerequisites
- **Docker** & **Docker Compose** (recommended) or standalone installation
- **Go** 1.25.1+ (for development)
- **Node.js/Bun** (for frontend development)
- **MySQL 5.7.8+** or **PostgreSQL 9.6+** (optional, for production)
- **Redis** (optional, recommended for production)

### Installation

#### Option 1: Docker Compose (Recommended)

```bash
# Clone the repository
git clone https://github.com/Calcium-Ion/new-api.git
cd new-api

# Edit configuration (optional)
nano docker-compose.yml

# Start services
docker-compose up -d

# Access the application
# Dashboard: http://localhost:3000
```

#### Option 2: Docker Command

**Using SQLite (default):**
```bash
docker run --name new-api -d --restart always \
  -p 3000:3000 \
  -e TZ=Asia/Shanghai \
  -v ./data:/data \
  calciumion/new-api:latest
```

**Using MySQL:**
```bash
docker run --name new-api -d --restart always \
  -p 3000:3000 \
  -e SQL_DSN="root:password@tcp(mysql-host:3306)/new_api?parseTime=true" \
  -e TZ=Asia/Shanghai \
  -v ./data:/data \
  calciumion/new-api:latest
```

#### Option 3: Local Development

```bash
# Install dependencies
cd web && bun install && cd ..

# Build frontend
cd web && DISABLE_ESLINT_PLUGIN='true' bun run build && cd ..

# Run backend
go run main.go
```

### Usage

#### Initial Setup
1. Access `http://localhost:3000`
2. Create admin account
3. Configure channels (API keys)
4. Create users and tokens
5. Set up billing/pricing

#### Configuration Examples

**Environment Variables (.env):**
```bash
# Core
PORT=3000
DEBUG=false

# Database
SQL_DSN=root:password@tcp(localhost:3306)/new_api?parseTime=true
# OR for SQLite
SQLITE_PATH=/data/new_api.db

# Cache & Session
REDIS_CONN_STRING=redis://localhost:6379/0
SESSION_SECRET=your-secure-random-string
CRYPTO_SECRET=your-encryption-key

# Timeouts
STREAMING_TIMEOUT=300
RELAY_TIMEOUT=0

# Features
MEMORY_CACHE_ENABLED=true
BATCH_UPDATE_ENABLED=true
UPDATE_TASK=true
CHANNEL_UPDATE_FREQUENCY=30

# Analytics (optional)
UMAMI_WEBSITE_ID=your-umami-id
GOOGLE_ANALYTICS_ID=your-ga-id
```

## Development

### Available Scripts

**Frontend:**
```bash
cd web
bun run dev       # Development server with hot reload
bun run build     # Production build
bun run preview   # Preview production build
```

**Backend:**
```bash
# Development
go run main.go

# Build binary
go build -o new-api

# Run with specific settings
DEBUG=true GIN_MODE=debug go run main.go

# Database migration (from /bin directory)
mysql -u root -p new_api < bin/migration_v0.3-v0.4.sql
```

**Makefile:**
```bash
make build-frontend   # Build frontend only
make start-backend    # Start backend dev server
make all              # Build frontend and start backend
```

### Development Workflow

1. **Frontend Changes**
   - Modify files in `web/src/`
   - Dev server auto-reloads
   - Build with `bun run build` before committing

2. **Backend Changes**
   - Modify Go files in root, `service/`, `controller/`, etc.
   - Test locally with `go run main.go`
   - Ensure no lint issues

3. **Database Schema Changes**
   - Update model files in `model/`
   - Create migration SQL if needed
   - Test with both SQLite and MySQL

4. **Testing & Verification**
   - Test with multiple database backends
   - Verify Docker build: `docker build -t new-api:test .`
   - Check with different environment configurations

### Code Organization

- **Controllers** (`controller/`): HTTP request handlers, business logic entry points
- **Services** (`service/`): Reusable business logic, external API calls
- **Models** (`model/`): Database entities, schema definitions, GORM models
- **Router** (`router/`): API route definitions, middleware setup
- **Middleware**: Cross-cutting concerns (logging, auth, request tracking)

## Configuration

### Database Configuration
- **SQLite**: Default, stores data in `/data/new_api.db` (Docker) or local path
- **MySQL**: `SQL_DSN=user:pass@tcp(host:3306)/dbname?parseTime=true`
- **PostgreSQL**: `SQL_DSN=postgresql://user:pass@host:5432/dbname`

### Cache Configuration
- **Redis** (recommended): `REDIS_CONN_STRING=redis://user:pass@host:6379/0`
- **Memory Cache**: `MEMORY_CACHE_ENABLED=true` with `SYNC_FREQUENCY=60`

### Session Management
- `SESSION_SECRET`: Required for session encryption (generate with: `openssl rand -base64 32`)
- `CRYPTO_SECRET`: Required when using Redis (for token encryption)

### API Timeouts
- `RELAY_TIMEOUT`: Maximum time for API relay (0 = unlimited)
- `STREAMING_TIMEOUT`: Maximum time for streaming responses (default 300s)

### Optional Features
- **Analytics**: Umami or Google Analytics integration
- **Payments**: Stripe or 易支付 integration
- **OAuth**: Configure via settings panel
- **OIDC**: For enterprise SSO

See [Environment Variables Documentation](https://docs.newapi.pro/installation/environment-variables) for complete configuration reference.

## Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Client Applications                   │
├─────────────────────────────────────────────────────────┤
│
├──→ HTTP/REST API ──→ Gin Router ──┐
│                                    │
│    WebSocket ──────────────────────→ API Routes
│                                    │
│    Web Dashboard ─────────────────→ Web Routes
│
├─────────────────────────────────────────────────────────┤
│                    Middleware Layer                      │
│  (Auth, CORS, Rate Limiting, Request Tracking)          │
├─────────────────────────────────────────────────────────┤
│                 Business Logic (Services)                │
│  (Token Counting, Quota Mgmt, Format Conversion, etc)   │
├─────────────────────────────────────────────────────────┤
│              Data Access & Controllers                   │
│  (User, Channel, Token, Pricing Management)            │
├─────────────────────────────────────────────────────────┤
│                   Data Persistence                       │
│  ┌────────────┐  ┌────────────┐  ┌──────────┐          │
│  │  SQLite    │  │   MySQL    │  │ PostgreSQL│          │
│  └────────────┘  └────────────┘  └──────────┘          │
├─────────────────────────────────────────────────────────┤
│                    Cache Layer                           │
│  ┌────────────┐  ┌──────────────────┐                  │
│  │   Redis    │  │  Memory Cache    │                  │
│  └────────────┘  └──────────────────┘                  │
├─────────────────────────────────────────────────────────┤
│              External AI Providers                       │
│  (OpenAI, Claude, Gemini, AWS, Azure, etc)             │
└─────────────────────────────────────────────────────────┘
```

### Key Components

1. **API Gateway**: Routes requests to appropriate provider based on model/channel
2. **Channel Manager**: Manages API keys and load balancing across providers
3. **Quota System**: Tracks usage, enforces limits, handles billing
4. **Token Counter**: Accurate token calculation for pricing
5. **Format Converter**: Converts between different API formats
6. **Authentication**: Multi-method auth (OAuth, JWT, WebAuthn)
7. **Dashboard**: React-based admin and user interface
8. **Relay System**: Smart routing with failover and retry

## Contributing

We welcome contributions! Please:

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Submit a Pull Request
5. All contributions are licensed under AGPLv3

See [Contributing Guidelines](https://docs.newapi.pro/support) for more details.

## License

**Dual Licensing Model:**

### AGPLv3 (Default Open Source License)
- Free to use for personal/non-commercial projects
- Required to open-source modifications and derivative works
- Cannot remove branding, logos, or copyright notices
- See [LICENSE](./LICENSE) for complete terms

### Commercial License
Required for:
- Removing branding/logos
- Closed-source SaaS deployment
- OEM integration in proprietary products
- No public source code requirement

**Contact for Commercial License**: `support@quantumnous.com`

## Additional Resources

- 📖 **Official Documentation**: https://docs.newapi.pro/
- 🐛 **Issue Tracker**: https://github.com/Calcium-Ion/new-api/issues
- 💬 **Community Chat**: https://docs.newapi.pro/support/community-interaction
- 🚀 **Latest Release**: https://github.com/Calcium-Ion/new-api/releases
- 📺 **Video Guides**: Tutorials available in documentation

## Project Metadata

- **Repository**: https://github.com/Calcium-Ion/new-api
- **Author/Maintainer**: QuantumNous (Calcium-Ion)
- **Go Version**: 1.25.1+
- **License**: AGPLv3 + Commercial
- **Last Updated**: 2024-2025
- **Status**: Active Development
- **Docker Hub**: https://hub.docker.com/r/CalciumIon/new-api
- **GHCR**: https://github.com/users/Calcium-Ion/packages/container/package/new-api

---

**Built with ❤️ by QuantumNous**
