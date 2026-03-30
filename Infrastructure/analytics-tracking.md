# User Tracking & Analytics Architecture

## Summary

Found user tracking and analytics implementations in 4 of 6 projects. All use client-side UUID generation + anonymous tracking with optional authenticated user_id on backends.

---

## Projects with Analytics

### 1. seobi-chat (Next.js)
**Status:** Active Analytics

#### Key Files:
- `/Users/js/Documents/Work/Projects/seobi-chat/lib/analytics.ts` - Core tracking library
- `/Users/js/Documents/Work/Projects/seobi-chat/components/PageTracker.tsx` - Page view tracker
- `/Users/js/Documents/Work/Projects/seobi-chat/app/api/chat/route.ts` - Server-side event tracking

#### Implementation Details:

**Client-side (lib/analytics.ts):**
- Uses browser `localStorage` with key `_aid`
- Generates UUID if not found: `crypto.randomUUID()`
- Sends events to: `NEXT_PUBLIC_INGESTOR_URL/v1/events`
- Each event includes:
  - `user_id`: Anonymous UUID from localStorage
  - `service_id`: "seobi-chat"
  - `event_type`: Customizable (e.g., "page_view")
  - `metadata`: Includes path, referrer, utm_source
  
**Server-side (route.ts):**
- `trackChatEvent(ip, unanswered)` function
- Tracks chat messages with IP address (not UUID)
- Sends: `event_type`, `service_id`, `metadata.unanswered`, `ip_address`
- Environment: `NEXT_PUBLIC_INGESTOR_URL` (optional - silently fails if missing)

---

### 2. boldgobynd (Next.js)
**Status:** Active Analytics

#### Key Files:
- `/Users/js/Documents/Work/Projects/boldgobynd/lib/analytics.ts` - Core tracking library
- `/Users/js/Documents/Work/Projects/boldgobynd/components/PageTracker.tsx` - Page view tracker
- `/Users/js/Documents/Work/Projects/boldgobynd/pages/_app.tsx` - Integration point

#### Implementation Details:

**Client-side (lib/analytics.ts):**
- Uses browser `localStorage` with key `_aid`
- Generates UUID if not found: `crypto.randomUUID()`
- Sends events to: `NEXT_PUBLIC_INGESTOR_URL` (default: `https://ingestor.nuclearbomb6518.com`)
- Each event includes:
  - `user_id`: Anonymous UUID from localStorage
  - `service_id`: "boldgobynd"
  - `event_type`: Customizable
  - `metadata`: Includes path, referrer
  - **Note:** Has hardcoded fallback ingestor URL

**Page Tracking:**
- `PageTracker` component fires on route changes
- Uses Next.js Pages Router (old pattern)

---

### 3. techfeed (React Native + NestJS Backend)
**Status:** Active Analytics (Dual-layer architecture)

#### Key Files:
- `/Users/js/Documents/Work/Projects/techfeed/apps/mobile/src/api/contents.ts` - Event tracking API
- `/Users/js/Documents/Work/Projects/techfeed/apps/mobile/app/content/[id].tsx` - Usage example
- `/Users/js/Documents/Work/Projects/techfeed/apps/api/src/events/events.service.ts` - Backend storage
- `/Users/js/Documents/Work/Projects/techfeed/apps/api/src/events/events.controller.ts` - Backend API

#### Implementation Details:

**Frontend (Mobile - React Native):**
- Uses `apiClient.post('/events', events)`
- Sends event array with:
  - `event_type`: 'read' | 'click' | 'bookmark' | 'share'
  - `content_id`: String ID
  - `duration_ms`: Optional (for read time tracking)
  - `tag`: Optional
  - `metadata`: Optional custom data
- **Note:** No explicit user_id on client — relies on JWT token if present
- Silently fails: "event tracking must not disrupt user experience"

**Backend API (NestJS):**
- Endpoint: `POST /events` with optional JWT guard
- Accepts array of up to 100 events
- Extracts `userId` from JWT if present: `req.user?.userId`
- Stores in TimescaleDB `user_events` hypertable with columns:
  - `time` (TIMESTAMPTZ, auto NOW())
  - `user_id` (UUID, nullable)
  - `event_type` (VARCHAR)
  - `content_id` (VARCHAR)
  - `tag` (VARCHAR)
  - `duration_ms` (INTEGER)
  - `metadata` (JSONB)

**Analytics Metrics Available:**
- `getTagTrends()`: Top tags (7-day window)
- `getHourlyReads()`: Read patterns (24-hour window)
- `getTopContents()`: Most clicked content (7-day window)
- `getActiveUsersCount()`: Distinct users with `user_id NOT NULL` (1-hour window)

---

## Projects WITHOUT Analytics Found

### 1. profile
- No active analytics tracking
- `PageTracker` component exists but not wired to ingestor
- **Status:** No external tracking

### 2. lotto-oracle
- No analytics tracking detected
- Playwright-based test project
- **Status:** No tracking code

### 3. my-ui-lib
- No analytics tracking detected
- Component library
- **Status:** No tracking code

---

## Security & Privacy Considerations

### Anonymous Tracking (seobi-chat, boldgobynd)
- UUID stored in browser `localStorage` key `_aid`
- Persists across browser sessions
- No authentication required
- Cannot be tied to user identity without additional data linking

### Authenticated Tracking (techfeed)
- JWT token optional
- `user_id` saved only if authenticated
- Unauthenticated requests store `user_id: null`
- All events linked by `user_id` or anonymous session

### IP-based Tracking (seobi-chat server)
- Server-side IP extraction from headers
- Used for rate limiting + event attribution
- Not tied to browser UUID
- Separate tracking layer from client-side analytics

### Ingestor Configuration
| Project | Ingestor | Default | Env Variable |
|---------|----------|---------|--------------|
| seobi-chat | External | None | `NEXT_PUBLIC_INGESTOR_URL` |
| boldgobynd | External | `ingestor.nuclearbomb6518.com` | `NEXT_PUBLIC_INGESTOR_URL` |
| techfeed | Embedded NestJS | N/A | N/A |

### Data Collection
| Project | Path | Referrer | UTM | Content | Duration | IP | JWT |
|---------|------|----------|-----|---------|----------|----|----|
| seobi-chat | Yes | Yes | Yes | No | No | Server | No |
| boldgobynd | Yes | Yes | No | No | No | No | No |
| techfeed | No | No | No | Yes | Yes | No | Optional |

---

## Implementation Patterns

### Pattern 1: Client-side Anonymous UUID (seobi-chat, boldgobynd)
```typescript
// Generate or retrieve anonymous ID from localStorage
function getAnonymousId(): string {
  const key = "_aid";
  let id = localStorage.getItem(key);
  if (!id) {
    id = crypto.randomUUID();
    localStorage.setItem(key, id);
  }
  return id;
}

// Send to external ingestor
fetch(`${INGESTOR_URL}/v1/events`, {
  method: "POST",
  body: JSON.stringify({
    event_type: eventType,
    service_id: "service-name",
    user_id: getAnonymousId(),
    metadata: { path, referrer, ... }
  })
});
```

### Pattern 2: Backend-authenticated Events (techfeed)
```typescript
// Client sends minimal data, backend extracts user from JWT
apiClient.post('/events', [
  { event_type: 'read', content_id: 'xyz', duration_ms: 15000 }
]);

// Backend controller extracts userId from JWT and stores
const userId = req.user?.userId; // from JWT guard
await eventsService.saveEvents(events, userId);

// TimescaleDB hypertable for time-series queries
CREATE TABLE user_events (
  time TIMESTAMPTZ NOT NULL,
  user_id UUID,
  event_type VARCHAR(50),
  ...
);
```

### Pattern 3: Server-side IP Tracking (seobi-chat)
```typescript
// Extract client IP from headers
function extractIp(req: NextRequest): string {
  const cfIp = req.headers.get('cf-connecting-ip');
  if (cfIp) return cfIp;
  const forwarded = req.headers.get('x-forwarded-for')?.split(',')[0];
  if (forwarded) return forwarded;
  return '127.0.0.1';
}

// Track separately from client-side UUID tracking
trackChatEvent(extractIp(req), isUnknown);
```

---

## Recommendations

1. **Consolidate analytics** - Consider standardizing on either anonymous UUID or JWT-based tracking
2. **Ingestor endpoint** - Avoid hardcoded fallback URLs in production code (boldgobynd issue)
3. **Privacy policy** - Ensure compliance with GDPR/CCPA for localStorage UUID persistence
4. **Data retention** - Define TTL for time-series data in TimescaleDB
5. **Error handling** - seobi-chat/boldgobynd silently fail on tracking errors (good), but monitor ingestor availability
