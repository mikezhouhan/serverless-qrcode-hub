# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a serverless QR code generator for WeChat groups, built on Cloudflare Workers with D1 database storage. The application allows creating permanent short links and QR codes that can be managed through an admin interface.

## Development Commands

### Local Development
```bash
# Start development server with cron testing
pnpm run dev
```
This starts Wrangler dev server on 0.0.0.0:8787 with scheduled event testing enabled. The cron endpoint is available at `http://localhost:8787/__scheduled?cron=*+*+*+*+*`.

### Deployment
```bash
# Deploy to Cloudflare Workers
pnpm run deploy
```

## Project Architecture

### Core Components

**Main Entry Point (`index.js`)**
- Single-file Worker that handles all HTTP requests and scheduled events
- Contains database operations, API endpoints, and redirect logic
- Uses Cloudflare D1 for data persistence

**Database Schema (`mappings` table)**
- `path` (TEXT PRIMARY KEY): Short link identifier
- `target` (TEXT): Destination URL
- `name` (TEXT): Display name
- `expiry` (TEXT): Expiration date (ISO string)
- `enabled` (INTEGER): Active status (0/1)
- `isWechat` (INTEGER): WeChat QR code flag (0/1)
- `qrCodeData` (TEXT): Base64 image data for WeChat QR codes
- `created_at` (TEXT): Creation timestamp

**Frontend Assets (`/dist/`)**
- `admin.html`: Main management interface with DaisyUI styling
- `login.html`: Authentication page
- Static assets: CSS (DaisyUI), JS (Tailwind, QR code libraries)

### Key Features

**URL Shortening**
- Creates short links that redirect to target URLs
- Supports expiration dates and enable/disable toggle
- Reserved paths in `banPath` array cannot be used as short links

**WeChat QR Code Management** 
- Special handling for WeChat group QR codes (`isWechat: true`)
- Stores original QR code image as base64 data
- Displays custom WeChat page instead of redirecting
- Provides mobile-optimized interface for QR code scanning

**Admin Interface**
- Cookie-based authentication using PASSWORD environment variable
- CRUD operations for managing mappings
- Pagination support for large datasets
- Responsive design with dark/light theme support

### Configuration Files

**`wrangler.toml`**
- Defines D1 database bindings for production and development
- Sets up scheduled events (cron jobs) for cleanup tasks
- Maps static assets from `/dist/` directory
- **Critical**: `database_id` must be updated with actual D1 database ID

**Environment Variables**
- `PASSWORD`: Admin authentication password (set via Cloudflare dashboard)
- `DB`: D1 database binding (configured in wrangler.toml)

### API Endpoints

**Authentication**
- `POST /api/login`: Authenticate with password
- `POST /api/logout`: Clear authentication cookie

**Mapping Management** 
- `GET /api/mappings`: List all mappings with pagination
- `GET /api/mapping?path=<path>`: Get specific mapping
- `POST /api/mapping`: Create new mapping
- `PUT /api/mapping`: Update existing mapping  
- `DELETE /api/mapping`: Delete mapping

**Monitoring**
- `GET /api/expiring-mappings`: Get expired and soon-to-expire links

### Scheduled Tasks

**Daily Cleanup Cron (`0 2 */1 * *`)**
- Runs `getExpiringMappings()` to identify expired links
- Logs expired and soon-to-expire mappings for monitoring
- Development environment runs every 10 seconds for testing

### Migration Support

**KV to D1 Migration**
- `migrateFromKV()` function handles data migration from legacy KV storage
- Preserves all mapping data including WeChat QR codes
- Manual migration guidance available in `MIGRATE.md`

## Development Guidelines

### Database Operations
- All database operations use prepared statements for security
- Batch operations implemented for performance (cleanup, migration)
- Automatic schema migration on startup adds missing columns

### Error Handling
- Input validation prevents SQL injection and invalid data
- Reserved path checking prevents system conflicts
- Graceful degradation for missing or corrupted data

### Performance Considerations
- Efficient pagination with single-query count/data retrieval
- Database indexes on expiry and created_at columns for performance
- Static asset serving through Cloudflare's edge network

### Security
- HttpOnly cookies with SameSite protection
- Password-based authentication with configurable strength
- Input sanitization for all user data
- Path traversal protection via banPath validation