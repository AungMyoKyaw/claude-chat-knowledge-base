# SparkBoard Documentation

A real-time kanban board built with SvelteKit 5, Firebase, and shadcn-svelte.

## Table of Contents

1. [Overview](#overview)
2. [Quick Start](#quick-start)
3. [Architecture](#architecture)
4. [Data Model](#data-model)
5. [Development](#development)
6. [API Reference](#api-reference)
7. [Deployment](#deployment)

---

## Overview

SparkBoard follows a **client-centric, serverless architecture**:

```
┌─────────────────────────────────────────────────────────────────┐
│                         Client Layer                            │
├─────────────────────────────────────────────────────────────────┤
│  SvelteKit (Svelte 5)  │  shadcn-svelte  │  Tailwind CSS       │
├─────────────────────────────────────────────────────────────────┤
│                   State Management Layer                        │
├─────────────────────────────────────────────────────────────────┤
│  Svelte Stores  │  svelte-query  │  Firebase SDK (v11+)        │
├─────────────────────────────────────────────────────────────────┤
│                        Firebase Layer                           │
├─────────────────────────────────────────────────────────────────┤
│  Auth (Magic Links) │  Firestore │  Storage │  Cloud Functions  │
└─────────────────────────────────────────────────────────────────┘
```

**Key Principles:**

- **Serverless First** - No backend server to maintain
- **Real-time by Default** - All data changes stream instantly
- **Security Rules First** - Authorization at the database level
- **Offline Capable** - Local cache with sync reconnection

### Multi-Tenancy Model

SparkBoard uses **URL-based multi-tenancy**. Each workspace has a unique external ID in the URL path:

```
/{workspace_external_id}/boards/{board_id}
```

Example: `/1234567/boards/abc-def-123`

---

## Quick Start

### Prerequisites

- Node.js 20+ (LTS recommended)
- npm, pnpm, or bun
- Firebase CLI: `npm install -g firebase-tools`
- Git

### Initial Setup

```bash
# Clone and install
git clone https://github.com/YOUR_USERNAME/sparkboard.git
cd sparkboard
npm install

# Configure environment
cp .env.example .env.local
```

Edit `.env.local` with your Firebase configuration:

```env
# Firebase Configuration
VITE_PUBLIC_FIREBASE_API_KEY=your_api_key
VITE_PUBLIC_FIREBASE_AUTH_DOMAIN=your_project.firebaseapp.com
VITE_PUBLIC_FIREBASE_PROJECT_ID=your_project_id
VITE_PUBLIC_FIREBASE_STORAGE_BUCKET=your_project.appspot.com
VITE_PUBLIC_FIREBASE_MESSAGING_SENDER_ID=your_sender_id
VITE_PUBLIC_FIREBASE_APP_ID=your_app_id

# Algolia Search (optional)
VITE_PUBLIC_ALGOLIA_APP_ID=your_algolia_app_id
VITE_PUBLIC_ALGOLIA_SEARCH_KEY=your_algolia_search_key
```

### Start Development

```bash
# Terminal 1: Firebase emulators
firebase emulators:start

# Terminal 2: SvelteKit dev server
npm run dev
```

Visit `http://localhost:5173`

---

## Architecture

### Project Structure

```
sparkboard/
├── src/
│   ├── lib/
│   │   ├── components/      # Reusable Svelte components
│   │   │   ├── ui/          # shadcn-svelte components
│   │   │   ├── board/       # Board-related components
│   │   │   ├── card/        # Card-related components
│   │   │   └── auth/        # Auth-related components
│   │   ├── firebase/        # Firebase utilities
│   │   │   ├── client.ts    # Firebase app initialization
│   │   │   ├── auth.ts      # Auth utilities
│   │   │   └── firestore.ts # Firestore helpers
│   │   ├── stores/          # Svelte stores
│   │   ├── types/           # TypeScript types
│   │   └── utils/           # Helper functions
│   ├── routes/              # SvelteKit file-based routing
│   │   ├── (app)/           # App layout (authenticated)
│   │   ├── auth/            # Auth routes
│   │   └── api/             # API endpoints
│   └── hooks.server.ts      # Server-side middleware
├── firebase/
│   ├── firestore.rules      # Security rules
│   ├── firestore.indexes.json
│   └── functions/           # Cloud Functions
└── tests/
```

### Authentication Flow

SparkBoard uses **passwordless magic link authentication**:

```
┌─────────┐         ┌──────────────┐         ┌──────────────┐
│  User   │         │ SvelteKit    │         │   Firebase   │
└────┬────┘         │    App       │         │   Functions  │
     │              └──────┬───────┘         └──────┬───────┘
     │  1. Enter email     │                        │
     ├────────────────────>│  2. Call Cloud Fn      │
     │                     ├───────────────────────>│
     │                     │                        │  3. Generate token
     │  4. Send email      │<───────────────────────┤
     │<────────────────────┤                        │
     │  5. Click link      │                        │
     ├─────────────────────────────────────────────>│
     │  6. Create session  │<───────────────────────┤
     │<────────────────────┤                        │
```

### Real-time Sync

All data changes synchronize via Firestore listeners:

```typescript
export function createCardStore(boardId: string) {
  const { subscribe, set } = writable<Card[]>([]);

  async function listen() {
    const q = query(
      collection(db, "cards"),
      where("boardId", "==", boardId),
      orderBy("position")
    );

    onSnapshot(q, (snapshot) => {
      const cards = snapshot.docs.map((doc) => ({
        id: doc.id,
        ...doc.data()
      }));
      set(cards);
    });
  }

  return { subscribe, listen };
}
```

### Entropy System

Automatically postpones stale cards to prevent endless todo lists. Configuration cascades:

1. **Workspace-level**: `workspace.entropyDays` (default: 30)
2. **Board-level**: `board.entropyDays` (optional override)
3. **Card tracking**: `card.entropyPostponedAt`

A scheduled Cloud Function runs hourly to move stale cards to "not now" status.

### Search Architecture

Uses **Algolia** for full-text search (Firestore has limited text search capabilities):

```typescript
// Cloud Function: Update Algolia on card change
export const indexCard = functions.firestore
  .document("cards/{cardId}")
  .onWrite(async (change, context) => {
    const card = change.after.data();
    if (!card) {
      await algolia.deleteObject(context.params.cardId);
      return;
    }

    await algolia.saveObject({
      objectID: context.params.cardId,
      workspaceId: card.workspaceId,
      title: card.title,
      description: stripHtml(card.description),
      tags: card.tagIds,
      status: card.status
    });
  });
```

---

## Data Model

### Entity Relationships

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Workspace  │────<│     User     │>────│   Identity   │
│              │     │              │     │              │
├──────────────┤     ├──────────────┤     ├──────────────┤
│ id           │     │ id           │     │ id           │
│ externalId   │     │ workspaceId  │     │ email        │
│ name         │     │ identityId   │     │ createdAt    │
│ ownerId      │     │ role         │     └──────────────┘
│ entropyDays  │     │ createdAt    │
│ createdAt    │     └──────────────┘
└──────────────┘            │
         │         ┌─────────┴─────────┐
         │         │                   │
         ▼         ▼                   ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│     Board    │ │   Access     │ │     Tag      │
├──────────────┤ ├──────────────┤ ├──────────────┤
│ id           │ │ boardId      │ │ id           │
│ workspaceId  │ │ userId       │ │ workspaceId  │
│ name         │ │ workspaceId  │ │ name         │
│ columnIds[]  │ └──────────────┘ │ color        │
│ allAccess    │                  └──────────────┘
│ position     │
└──────────────┘
         │
         ▼
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│    Column    │>────│     Card     │<────│    Comment   │
├──────────────┤     ├──────────────┤     ├──────────────┤
│ id           │     │ id           │     │ id           │
│ boardId      │     │ workspaceId  │     │ cardId       │
│ name         │     │ boardId      │     │ authorId     │
│ position     │     │ columnId     │     │ content      │
│ color        │     │ sequenceNum  │     │ createdAt    │
└──────────────┘     │ title        │     └──────────────┘
                     │ description  │
                     │ assigneeId   │
                     │ tagIds[]     │
                     │ status       │
                     │ position     │
                     └──────────────┘
```

### Collection Schemas

#### `workspaces/{workspaceId}`

```typescript
{
  id: string; // Auto-generated UUID
  externalId: string; // 7+ digit number, URL-safe
  name: string; // Human-readable name
  ownerId: string; // Reference to User who created it
  entropyDays: number; // Default days before auto-postpone
  createdAt: Timestamp;
  updatedAt: Timestamp;
}
```

**Indexes:** `externalId` (single-field, unique)

#### `identities/{identityId}`

```typescript
{
  id: string;              // Firebase Auth UID
  email: string;           // Verified email address
  createdAt: Timestamp;
  lastLoginAt?: Timestamp;
}
```

#### `users/{userId}`

```typescript
{
  id: string; // Auto-generated UUID
  workspaceId: string; // Parent workspace
  identityId: string; // Firebase Auth UID
  email: string; // For queries/display
  role: "owner" | "admin" | "member";
  createdAt: Timestamp;
}
```

**Indexes:** `workspaceId`, `identityId`, `[workspaceId, identityId]` (composite, unique)

#### `boards/{boardId}`

```typescript
{
  id: string;              // Auto-generated UUID
  workspaceId: string;     // Parent workspace
  name: string;
  description?: string;
  columnIds: string[];     // Ordered column IDs
  allAccess: boolean;      // True = all members can access
  position: number;        // Order in workspace
  createdAt: Timestamp;
  updatedAt: Timestamp;
}
```

**Indexes:** `workspaceId`, `[workspaceId, position]`

#### `columns/{columnId}`

```typescript
{
  id: string;              // Auto-generated UUID
  boardId: string;         // Parent board
  workspaceId: string;     // Denormalized for queries
  name: string;            // e.g., "To Do", "In Progress"
  position: number;        // Order on board
  color?: string;          // Hex color for visual distinction
}
```

**Indexes:** `boardId`, `[boardId, position]`

#### `cards/{cardId}`

```typescript
{
  id: string;                  // Auto-generated UUID
  workspaceId: string;         // Denormalized for security
  boardId: string;
  columnId: string;
  sequenceNumber: number;      // Per-workspace incrementing number
  title: string;
  description?: string;        // Rich text JSON (Tiptap format)
  assigneeId?: string;         // User ID
  tagIds: string[];
  attachments: Attachment[];
  status: 'triage' | 'active' | 'closed' | 'not_now';
  dueDate?: Timestamp;
  entropyPostponedAt?: Timestamp;
  position: number;            // Order within column
  createdAt: Timestamp;
  updatedAt: Timestamp;
}
```

**Indexes:** `boardId`, `columnId`, `[boardId, position]`, `[workspaceId, status]`

#### `comments/{commentId}`

```typescript
{
  id: string;
  workspaceId: string; // Denormalized for security
  cardId: string;
  authorId: string; // User ID
  content: string; // Rich text JSON
  createdAt: Timestamp;
}
```

**Indexes:** `cardId`

#### `events/{eventId}` - Activity Log

```typescript
{
  id: string;
  workspaceId: string;
  type: string;            // 'card_created', 'card_moved', etc.
  entityType: string;      // 'card', 'board', 'column'
  entityId: string;
  actorId: string;         // User ID or 'system'
  particulars: {           // Event-specific data
    from?: string;
    to?: string;
    changes?: Record<string, any>;
  };
  createdAt: Timestamp;
}
```

**Indexes:** `entityId`, `[workspaceId, createdAt]` (composite, desc)

---

## Development

### Code Style

**TypeScript:**

- Use strict mode
- Prefer explicit return types for public APIs
- Use Svelte 5 runes (`$state`, `$derived`, `$props`)

**Svelte 5 Syntax:**

```svelte
<script lang="ts">
  interface Props {
    title: string;
    count?: number;
  }

  let { title, count = 0 }: Props = $props();
  let doubled = $derived(count * 2);
</script>
```

**Naming Conventions:**

| Type             | Convention           | Example                          |
| ---------------- | -------------------- | -------------------------------- |
| Components       | PascalCase           | `Card.svelte`, `BoardHeader`     |
| Stores           | camelCase with `use` | `useCards()`, `useAuth()`        |
| Functions        | camelCase            | `createCard()`, `getWorkspace()` |
| Constants        | UPPER_SNAKE_CASE     | `DEFAULT_ENTROPY_DAYS`           |
| Types/Interfaces | PascalCase           | `Card`, `Workspace`              |

### Testing

```bash
# Unit tests (Vitest)
npm test
npm run test:watch
npm run test:coverage

# E2E tests (Playwright)
npx playwright install
npm run test:e2e
npx playwright test --ui

# Firestore rules tests
cd firebase/functions && npm test
```

### Adding Components

```bash
# Using shadcn-svelte CLI
npx shadcn-svelte@latest add button
npx shadcn-svelte@latest add dialog
```

### Firebase Emulator

For local development:

```bash
# Start emulators
firebase emulators:start --ui
```

Access the emulator UI at `http://localhost:4000`:

- Authentication: `http://localhost:9099`
- Firestore: `http://localhost:8080`
- Functions: `http://localhost:5001`

Connect to emulators in code:

```typescript
// src/lib/firebase/client.ts
if (import.meta.env.DEV) {
  connectFirestoreEmulator(db, "localhost", 8080);
  connectFunctionsEmulator(functions, "localhost", 5001);
}
```

---

## API Reference

### Workspace API

```typescript
// Get workspace by external ID
export async function getWorkspaceByExternalId(
  externalId: string
): Promise<Workspace | null> {
  const q = query(
    collection(db, "workspaces"),
    where("externalId", "==", externalId)
  );
  const snap = await getDocs(q);
  if (snap.empty) return null;
  return { id: snap.docs[0].id, ...snap.docs[0].data() };
}

// Get workspace members
export async function getWorkspaceMembers(
  workspaceId: string
): Promise<User[]> {
  const q = query(
    collection(db, "users"),
    where("workspaceId", "==", workspaceId)
  );
  const snap = await getDocs(q);
  return snap.docs.map((doc) => ({ id: doc.id, ...doc.data() }));
}
```

### Board API

```typescript
export async function createBoard(data: CreateBoardDto): Promise<string> {
  const ref = await addDoc(collection(db, "boards"), {
    ...data,
    createdAt: serverTimestamp(),
    updatedAt: serverTimestamp()
  });
  return ref.id;
}

export function listenToBoard(
  boardId: string,
  callback: (board: Board | null) => void
) {
  return onSnapshot(doc(db, "boards", boardId), (doc) => {
    callback(doc.exists() ? { id: doc.id, ...doc.data() } : null);
  });
}
```

### Card API

```typescript
export async function createCard(data: CreateCardDto): Promise<string> {
  const sequenceNumber = await getNextSequenceNumber(data.workspaceId);

  const ref = await addDoc(collection(db, "cards"), {
    ...data,
    sequenceNumber,
    status: "triage",
    position: 0,
    createdAt: serverTimestamp(),
    updatedAt: serverTimestamp()
  });

  await logEvent({
    workspaceId: data.workspaceId,
    type: "card_created",
    entityType: "card",
    entityId: ref.id,
    actorId: getCurrentUserId()
  });

  return ref.id;
}

export async function moveCard(
  cardId: string,
  newColumnId: string,
  newPosition: number,
  oldColumnId: string
): Promise<void> {
  const batch = writeBatch(db);

  batch.update(doc(db, "cards", cardId), {
    columnId: newColumnId,
    position: newPosition,
    updatedAt: serverTimestamp()
  });

  await reorderCardsInBatch(batch, oldColumnId);
  await reorderCardsInBatch(batch, newColumnId);
  await batch.commit();

  await logEvent({
    workspaceId: getCurrentWorkspaceId(),
    type: "card_moved",
    entityType: "card",
    entityId: cardId,
    actorId: getCurrentUserId(),
    particulars: { from: oldColumnId, to: newColumnId }
  });
}
```

### Cloud Functions

#### `generateMagicLink`

Generates a magic link for passwordless authentication.

#### `createWorkspace`

Creates a new workspace with the current user as owner.

#### `entropyPostpone`

Runs hourly to postpone stale cards based on entropy settings.

---

## Deployment

### Environment Setup

```bash
# Login to Firebase
firebase login

# Create project
firebase projects:create sparkboard-production

# Initialize Firebase
firebase init
```

### Frontend Deployment

**Option 1: Vercel (Recommended)**

```bash
npm install -g vercel
vercel link
```

Environment variables in Vercel:

```env
VITE_PUBLIC_FIREBASE_API_KEY=xxx
VITE_PUBLIC_FIREBASE_AUTH_DOMAIN=xxx
VITE_PUBLIC_FIREBASE_PROJECT_ID=xxx
VITE_PUBLIC_FIREBASE_STORAGE_BUCKET=xxx
VITE_PUBLIC_FIREBASE_MESSAGING_SENDER_ID=xxx
VITE_PUBLIC_FIREBASE_APP_ID=xxx
```

**Option 2: Firebase Hosting**

```bash
firebase deploy --only hosting
```

### Cloud Functions Deployment

```bash
# Deploy all functions
firebase deploy --only functions

# Deploy specific function
firebase deploy --only functions:generateMagicLink

# Deploy with region
firebase deploy --only functions --region=us-central1
```

### Security Rules

```bash
# Deploy Firestore rules
firebase deploy --only firestore:rules

# Deploy indexes
firebase deploy --only firestore:indexes
```

### Post-Deployment Checklist

**Security:**

- [ ] Firestore rules in production mode
- [ ] API keys restricted by domain/referrer
- [ ] App Check enabled for Cloud Functions

**Performance:**

- [ ] Firestore indexes created
- [ ] CDN caching enabled
- [ ] Bundle size optimized

**Monitoring:**

- [ ] Error tracking enabled
- [ ] Analytics configured
- [ ] Alert thresholds set

---

## Cost Estimates (Monthly)

### Free Tier Capacity

| Service   | Free Tier Limits                        | Est. Active Users\* |
| --------- | --------------------------------------- | ------------------- |
| Firestore | 1 GB storage, 50K reads, 20K writes/day | ~500-1,000 users    |
| Auth      | Email link, email/password (unlimited)  | Unlimited           |
| Functions | 2M invocations/month (Blaze plan)       | ~1,000+ users       |
| Hosting   | 10 GB storage                           | ~10+ apps           |
| Storage   | 5 GB storage                            | ~500-1000 files     |
| Algolia   | 10K searches/month, 1M records          | ~100-300 users\*\*  |

\*Based on typical kanban usage: ~50-100 reads, ~5-10 writes per user per day
\*\*Search-heavy usage: ~30-100 searches per user per month

### Paid Tier Estimates (Blaze Plan)

| Service   | Pricing                                              | Est. Cost for 1K Users |
| --------- | ---------------------------------------------------- | ---------------------- |
| Firestore | $0.18/GB stored, $0.06/100K reads, $0.18/100K writes | ~$25-100/month         |
| Auth      | Phone: $0.01/verification                            | ~$10/month             |
| Functions | $0.40/M invocations, $10/GB-sec                      | ~$10-50/month          |
| Hosting   | $0.026/GB overage                                    | ~$0-5/month            |
| Storage   | $0.026/GB stored                                     | ~$0-5/month            |
| Algolia   | $0.50/1K searches over 10K                           | ~$15-50/month          |

**Total for 1K active users:** ~$60-210/month

---

## Resources

- [SvelteKit Docs](https://kit.svelte.dev)
- [Firebase Docs](https://firebase.google.com/docs)
- [shadcn-svelte](https://shadcn-svelte.com)
- [Svelte 5 Runes](https://svelte.dev/docs/runes)
