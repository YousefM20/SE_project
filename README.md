ArtAuction — Microservices Platform
A full-stack web application for auctioning artworks, built with a modern microservices architecture. Artists can list and auction artworks, buyers can place real-time bids, and administrators can manage the platform end-to-end.

Architecture Overview
artauction/
├── artauction-backend/
│   ├── eureka-server/          # Service discovery (port 8761)
│   ├── api-gateway/            # JWT validation + routing (port 8080)
│   ├── user-service/           # Auth, registration, RBAC (port 8081)
│   ├── auction-service/        # Artworks, auctions, bids (port 8082)
│   ├── notification-service/   # WebSocket notifications (port 8083)
│   └── watchlist-service/      # Buyer watchlists (port 8084)
└── artauction-frontend/        # React 18 + Vite SPA
Each microservice has its own isolated PostgreSQL database and registers with Eureka for dynamic service discovery.

Tech Stack
LayerTechnologyFrontendReact 18, Vite 5, react-router-dom 6, AxiosBackendSpring Boot 3.2, Spring Cloud 2023, Spring SecurityAuthJWT (24h expiry), BCrypt (strength 10)Real-TimeWebSocket (STOMP / SockJS)DatabasesPostgreSQL 15 (one per service)Service DiscoveryEureka ServerAPI GatewaySpring Cloud GatewayInter-ServiceOpenFeign (HTTP)ContainerizationDocker + Docker ComposeCross-CuttingSpring AOP (logging, audit)

Prerequisites

Docker Desktop >= 4.x with Compose v2
Java 17+ (for local development without Docker)
Node.js 18+ (for frontend development)
6 GB RAM recommended
Available ports: 5432, 8080, 8081, 8082, 8083, 8084, 8761


Quick Start
Run the full stack with Docker
bash# Clone the repository
git clone <repo-url>
cd artauction

# Start all services
docker compose up --build
Once running, the application is available at:
ServiceURLFrontendhttp://localhost:3000API Gatewayhttp://localhost:8080Eureka Dashboardhttp://localhost:8761User Service Swaggerhttp://localhost:8081/swagger-ui.htmlAuction Service Swaggerhttp://localhost:8082/swagger-ui.htmlNotification Service Swaggerhttp://localhost:8083/swagger-ui.html
Stop all services
bashdocker compose down

Data persists across restarts via Docker volumes. To wipe all data: docker compose down -v


User Roles
RoleCapabilitiesGUESTBrowse auctions and artworks (no login required)BUYERPlace bids, manage watchlist, view notificationsARTISTUpload artworks, create and manage auctions (requires admin approval)EMPLOYEEApprove artists, moderate content, cancel auctionsADMINFull platform control — manage users, content, and system
Default Test Accounts
After starting the app, register accounts via the UI or use the API directly. Artists require approval before they can log in — an ADMIN or EMPLOYEE must approve them first.

Authentication Flow

Register at POST /api/auth/register with { fullName, email, password, role, bio? }
ARTIST accounts start with PENDING status and require approval
Login at POST /api/auth/login — returns a JWT token
Include the token in subsequent requests: Authorization: Bearer <token>
Tokens expire after 24 hours


Core API Endpoints
Auth
MethodEndpointAuthDescriptionPOST/api/auth/registerNoRegister new userPOST/api/auth/loginNoLogin, returns JWT
Users
MethodEndpointRoleDescriptionGET/api/usersADMIN/EMPLOYEEList all usersGET/api/users/artists/pendingADMIN/EMPLOYEEList pending artistsPUT/api/users/artists/{id}/approveADMIN/EMPLOYEEApprove artistDELETE/api/users/{id}ADMINDelete user
Artworks
MethodEndpointRoleDescriptionGET/api/artworksPublicList all artworksPOST/api/artworksARTISTUpload artworkDELETE/api/artworks/{id}EMPLOYEE/ADMINDelete artwork
Auctions
MethodEndpointRoleDescriptionGET/api/auctionsPublicList all auctionsGET/api/auctions/activePublicList active auctionsPOST/api/auctionsARTISTCreate auctionPUT/api/auctions/{id}/cancelARTIST/EMPLOYEE/ADMINCancel auction
Bids
MethodEndpointRoleDescriptionPOST/api/bids/auction/{id}BUYERPlace a bidGET/api/bids/auction/{id}PublicView bid historyGET/api/bids/myAuthenticatedMy bid history
Notifications
MethodEndpointRoleDescriptionGET/api/notifications/myAuthenticatedMy notificationsPUT/api/notifications/{id}/readAuthenticatedMark as readGET/api/notifications/allADMINAll notifications
Watchlist
MethodEndpointRoleDescriptionPOST/api/watchlistBUYERAdd auction to watchlistDELETE/api/watchlist/{auctionId}BUYERRemove from watchlistGET/api/watchlist/myBUYERGet my watchlist

Database Schema
Each service owns its database — no cross-service DB access.
ServiceDatabaseTablesUser Serviceuser_service_dbusers, rolesAuction Serviceauction_service_dbartworks, auctions, bidsNotification Servicenotification_service_dbnotificationsWatchlist Servicewatchlist_service_dbwatchlist_items
Key relationships (logical, not FK across DBs):

One Artist → many Artworks
One Artwork → one Auction
One Auction → many Bids
One User → many Notifications
One Buyer → many Watchlist Items


Real-Time Features
WebSocket connections (STOMP over SockJS) are established on login and deliver:

New bid placed — broadcast to all viewers of that auction
Outbid alert — sent to the previously highest bidder
Auction won — sent to the winner when auction closes
Artist approved — sent to the artist on approval

Fallback to polling is available if WebSocket is unavailable.

Auction Lifecycle
SCHEDULED ──(startTime reached)──► ACTIVE ──(endTime reached)──► CLOSED
     │                                │
     └──(cancelled)──► CANCELLED      └──(cancelled)──► CANCELLED
The auction scheduler runs every 30 seconds, automatically activating scheduled auctions and closing expired ones. On close, the highest bidder is set as the winner and the artwork status updates to SOLD (or back to AVAILABLE if no bids).

Design Patterns Used

Singleton — JwtTokenProvider, Eureka Server
Factory — User creation by role
Builder — Lombok @Builder on entities
Repository — JpaRepository abstraction
Observer — WebSocket/STOMP for real-time bid events
Strategy — JWT auth filter per endpoint
Facade — AuctionService simplifying complex operations
API Gateway — Single entry point with centralized JWT validation
Database per Service — Full data isolation per microservice


Security

Passwords hashed with BCrypt (strength 10)
JWT tokens validated at the API Gateway before any request is routed
RBAC enforced at both gateway and controller levels
SQL injection prevented via JPA/Hibernate parameterized queries
XSS prevented via React's built-in escaping
CORS restricted to known frontend origins
HTTPS via nginx reverse proxy in production


Frontend Structure
artauction-frontend/src/
├── context/
│   ├── AuthContext.js          # Login/logout/register state
│   └── NotificationContext.js  # WebSocket notification state
├── services/api.js             # Axios API layer
├── hooks/
│   ├── useAuctions.js          # Auction list with filters
│   └── useAuction.js           # Single auction fetch
├── components/
│   ├── layout/Navbar.jsx
│   ├── auction/AuctionCard.jsx
│   ├── auction/BidForm.jsx
│   ├── auction/BidHistory.jsx
│   └── auth/ProtectedRoute.jsx
└── pages/
    ├── Home.jsx                # Hero + featured auctions
    ├── Auctions.jsx            # Browse with filters
    ├── AuctionDetail.jsx       # Bidding page
    ├── Profile.jsx             # Edit profile + stats
    ├── Watchlist.jsx           # Saved auctions
    ├── Notifications.jsx       # Real-time alerts
    └── CreateAuction.jsx       # Sell artwork form

Non-Functional Requirements
RequirementTargetAPI Response Time< 500ms (95th percentile)WebSocket Latency< 100msPage Load Time< 2s on broadbandConcurrent Users100+Uptime99.9% during business hoursAuction SchedulerEvery 30 seconds
