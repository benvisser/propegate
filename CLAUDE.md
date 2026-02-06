# Propagate — MVP Build Plan

## MVP Scope — 7 Core Features Only

1. Add plants you can propagate (photo + type)
2. Add plants you want (wish list)
3. Match with users who have what you want
4. Swipe through available plants & initiate match or propose swap
5. Approve/decline swap requests
6. Exchange addresses for shipping
7. Rate the person during & after the swap

---

## Tech Stack

| Layer | Choice |
|---|---|
| Platform | iOS 17+ (Swift / SwiftUI) |
| Architecture | MVVM + Repository pattern |
| Backend | Firebase Firestore |
| Auth | Firebase Phone Auth |
| Photo Storage | Firebase Cloud Storage |
| Push Notifications | Firebase Cloud Messaging |
| Payments | None for MVP (shipping is handled between users) |

---

## Design Direction

**Style:** Clean, modern, open, white — inspired by Airbnb and AllTrails.
**Swipe UX:** Inspired by dating apps (Tinder/Hinge) for the plant discovery and matching flow.

### Design Tokens

```
Background:        #FFFFFF (pure white)
Surface:           #F9FAFB (light gray, cards/sections)
Primary:           #2D6A4F (deep sage green)
Primary Light:     #52B788 (fresh green, accents/CTAs)
Text Primary:      #111827 (near black)
Text Secondary:    #6B7280 (gray)
Text Tertiary:     #9CA3AF (light gray, hints)
Border:            #E5E7EB (subtle dividers)
Success:           #10B981
Warning:           #F59E0B
Error:             #EF4444
Star Rating:       #F59E0B (amber)

Font:              SF Pro (system default)
Corner Radius:     12pt (cards), 24pt (buttons), 999pt (pills/avatars)
Shadow:            0 1px 3px rgba(0,0,0,0.08) — very subtle, Airbnb-style
Spacing unit:      8pt grid
```

---

## Data Model (Firestore)

```
users/{userId}
  ├── displayName: string
  ├── profilePhotoURL: string
  ├── zipCode: string
  ├── state: string
  ├── bio: string (optional, short)
  ├── ratingAvg: number (0-5)
  ├── ratingCount: number
  ├── createdAt: timestamp
  │
  ├── myPlants/{plantId}          ← plants I can propagate
  │     ├── name: string
  │     ├── type: string          (e.g. "Monstera", "Pothos", "Succulent")
  │     ├── photoURL: string
  │     ├── notes: string         (optional — "healthy, easy to prop")
  │     ├── isAvailable: bool
  │     └── createdAt: timestamp
  │
  └── wantList/{plantId}          ← plants I'm looking for
        ├── name: string
        ├── type: string
        └── addedAt: timestamp

plantFeed/{plantId}               ← denormalized feed for swiping
  ├── name: string
  ├── type: string
  ├── photoURL: string
  ├── ownerId: string
  ├── ownerName: string
  ├── ownerPhotoURL: string
  ├── ownerZip: string
  ├── ownerRatingAvg: number
  ├── isAvailable: bool
  └── createdAt: timestamp

swaps/{swapId}
  ├── status: string              "proposed" → "accepted" → "addresses_shared"
  │                                → "shipped" → "received" → "completed"
  │                                or "declined" / "cancelled"
  ├── type: string                "request" or "swap"
  │
  ├── initiator                   ← person who swiped / proposed
  │     ├── userId: string
  │     ├── displayName: string
  │     ├── plantId: string
  │     ├── plantName: string
  │     ├── plantPhotoURL: string
  │     ├── shippingAddress: string (added after accept)
  │     ├── hasShipped: bool
  │     └── rating: map { score: number, comment: string, createdAt: timestamp }
  │
  ├── receiver                    ← person who owns the requested plant
  │     ├── userId: string
  │     ├── displayName: string
  │     ├── plantId: string       (null if type="request", populated if "swap")
  │     ├── plantName: string
  │     ├── plantPhotoURL: string
  │     ├── shippingAddress: string (added after accept)
  │     ├── hasShipped: bool
  │     └── rating: map { score: number, comment: string, createdAt: timestamp }
  │
  ├── createdAt: timestamp
  └── updatedAt: timestamp

swipedPlants/{userId}/seen/{plantId}   ← track what user has already swiped on
  └── action: string ("liked" or "passed")
  └── timestamp: timestamp
```

**Why a separate `plantFeed` collection?**
Firestore can't query across subcollections easily. Denormalizing into a flat feed collection lets you query all available plants, filter, and paginate efficiently for the swipe screen. When a user adds/removes a plant in `myPlants`, a simple write also updates `plantFeed`.

---

## Screen Map

```
┌─────────────────────────────────────────────────┐
│                   APP FLOW                       │
│                                                  │
│  Splash → Phone Auth → OTP → Profile Setup       │
│                          ↓                       │
│              ┌── Tab Bar ──────────────┐         │
│              │                         │         │
│         ┌────┴────┐  ┌──────┐  ┌──────┴───┐     │
│         │ Explore  │  │  My  │  │  Swaps   │     │
│         │ (Swipe)  │  │Garden│  │ (Inbox)  │     │
│         └────┬─────┘  └──┬───┘  └────┬─────┘     │
│              │           │           │           │
│         Plant Card   Add Plant   Swap Detail     │
│         Stack        Add Want    → Accept/Decline│
│         ↓            Plant List  → Share Address  │
│         Swipe Right              → Mark Shipped   │
│         → Propose                → Confirm Receipt│
│           Match/Swap             → Rate & Review  │
│                                                  │
│              ┌──────────┐                        │
│              │ Profile   │                        │
│              │ (4th tab) │                        │
│              └──────────┘                        │
└─────────────────────────────────────────────────┘
```

### 4 Tabs

| Tab | Icon | Purpose |
|---|---|---|
| **Explore** | leaf | Swipe through plants, propose match or swap |
| **My Garden** | plant | Manage your plants (have) and wish list (want) |
| **Swaps** | arrows | Inbox of proposed, active, and completed swaps |
| **Profile** | person | Your info, ratings, settings |

---

## Screen-by-Screen Detail

### 1. Auth Flow

**Welcome Screen**
- App logo + tagline: "Share the love of plants"
- Big "Get Started" CTA button
- Clean white background, centered layout

**Phone Entry**
- "Enter your phone number" with country code picker
- Large, friendly number input
- "Continue" button (disabled until valid number)

**OTP Verification**
- 6-digit code input (auto-advancing boxes)
- "Resend code" timer link
- Auto-verifies on completion

**Profile Setup**
- Profile photo (camera or library — circular crop)
- Display name
- Zip code (for general proximity, not exact location)
- Optional short bio
- "Start Growing" CTA

---

### 2. My Garden (Tab 2)

**Two sections, segmented control at top:**

**"I Have" Section**
- Grid of plant cards (2 columns, like AllTrails saved lists)
- Each card: photo, name, type, availability toggle
- "+" FAB or top-right button to add new plant
- Tap card → edit or remove

**"I Want" Section**
- Simpler list (no photos needed, but allow optional photo)
- Each row: plant name, type, "x" to remove
- "+" to add new want
- These feed the matching algorithm

**Add Plant Sheet (bottom sheet)**
- Photo picker (camera or library)
- Plant name (text field with autocomplete from common plant list)
- Plant type / category (dropdown: Tropical, Succulent, Herb, Vine, Fern, Flowering, Other)
- Optional notes ("Easy to propagate", "Variegated", etc.)
- "Add to My Garden" button

**Add Want Sheet**
- Plant name (text field with autocomplete)
- Plant type (dropdown)
- "Add to Wish List" button

---

### 3. Explore / Swipe (Tab 1) — The Core Loop

**Layout:** Full-screen card stack (dating app style)

**Each card shows:**
- Large plant photo (fills most of the card, rounded corners)
- Plant name + type overlaid at bottom (white text on gradient)
- Owner's name, profile photo (small), general location ("Charlotte, NC")
- Owner's average rating (stars, if they have ratings)

**Swipe gestures:**
- **Swipe left** → Pass (not interested)
- **Swipe right** → "I want this!" → triggers match proposal
- **Tap card** → Expand to full detail view

**Full Detail View (modal/push):**
- Larger photo(s)
- Plant name, type, owner notes
- Owner mini-profile (photo, name, rating, number of successful swaps)
- Two CTAs:
  - **"Request Clipping"** — one-way request (you want theirs)
  - **"Propose Swap"** — you pick one of YOUR plants to offer in return

**Propose Swap Flow:**
- Bottom sheet showing your available plants
- Select which plant you're offering
- Confirm: "Propose swap: Your [Plant A] <-> Their [Plant B]"
- Sends to their Swaps inbox

**Matching Logic:**
- Feed excludes: your own plants, plants you've already swiped on
- Feed sorts by: relevance to your want list first, then recency
- If you swipe right on a plant that matches something on your want list, the match feels intentional

---

### 4. Swaps Inbox (Tab 3)

**Three sections (segmented control or collapsible):**

**Pending**
- Incoming requests/swaps awaiting your response
- Each row: plant photo, plant name, requester name + photo, "Request" or "Swap" badge
- Tap → Swap Detail

**Active**
- Accepted swaps in progress
- Status badge: "Share Addresses" → "Ship It" → "In Transit" → "Received"
- Tap → Swap Detail

**Completed**
- Past swaps
- Rating badge (rated / unrated)
- Tap → view details or leave rating

---

### 5. Swap Detail Screen — The Full Lifecycle

This is the most important screen. It guides both users through the entire swap process step-by-step.

**State: Proposed (receiver's view)**
```
┌─────────────────────────────┐
│  [Requester photo + name]    │
│  wants your [Plant Name]     │
│  ┌───────────────────────┐   │
│  │   [Plant Photo]       │   │
│  └───────────────────────┘   │
│                              │
│  ── If swap ──               │
│  In exchange for their:      │
│  ┌───────────────────────┐   │
│  │   [Their Plant Photo] │   │
│  └───────────────────────┘   │
│                              │
│  [Accept]     [Decline]      │
└─────────────────────────────┘
```

**State: Accepted → Share Addresses**
- Both users see: "Swap accepted! Share your shipping address to continue."
- Secure address input form (street, city, state, zip)
- Address only visible to the other person in this swap
- Once both addresses are shared → status moves to "Ready to Ship"

**State: Ready to Ship**
- Both users see each other's address
- Shipping instructions card:
  - Take a healthy cutting (4-6 inches, include a node)
  - Wrap stem end in damp paper towel
  - Wrap in plastic wrap or small zip-lock
  - Place in small rigid box with cushioning
  - Label "LIVE PLANT — HANDLE WITH CARE"
  - Ship Monday–Wednesday (avoid weekend sitting)
  - Use USPS Priority Mail (2-3 days) for best results
- "Mark as Shipped" button
- Optional: add tracking number

**State: Shipped / In Transit**
- Shows who has shipped and who hasn't
- "Mark as Received" button (appears after reasonable time)

**State: Received → Rate**
- "How was your experience with [Name]?"
- Star rating (1-5)
- Optional comment
- "Submit Rating"
- Once both rate (or after 7 days) → swap moves to "Completed"

**State: Completed**
- Summary view of the swap
- Both ratings visible
- "Done" — archived

---

### 6. Profile (Tab 4)

- Profile photo, name, bio
- Stats row: "12 plants" / "8 swaps" / "4.8 stars"
- Edit profile button
- Settings: notifications, privacy, help, log out

---

## Xcode Project Structure

```
Propagate/
├── App/
│   ├── PropagateApp.swift           ← entry point
│   ├── ContentView.swift            ← tab bar root
│   └── AppState.swift               ← global auth state
│
├── Models/
│   ├── User.swift
│   ├── Plant.swift
│   ├── WantedPlant.swift
│   ├── Swap.swift
│   └── Rating.swift
│
├── ViewModels/
│   ├── AuthViewModel.swift
│   ├── GardenViewModel.swift        ← my plants + want list
│   ├── ExploreViewModel.swift       ← swipe feed logic
│   ├── SwapsViewModel.swift         ← inbox + swap lifecycle
│   └── ProfileViewModel.swift
│
├── Views/
│   ├── Auth/
│   │   ├── WelcomeView.swift
│   │   ├── PhoneEntryView.swift
│   │   ├── OTPVerificationView.swift
│   │   └── ProfileSetupView.swift
│   │
│   ├── Explore/
│   │   ├── ExploreView.swift        ← card stack
│   │   ├── PlantCardView.swift      ← individual swipe card
│   │   ├── PlantDetailView.swift    ← expanded detail
│   │   └── ProposeSwapSheet.swift   ← pick your plant to offer
│   │
│   ├── Garden/
│   │   ├── GardenView.swift         ← segmented: Have / Want
│   │   ├── PlantGridItem.swift      ← card in grid
│   │   ├── AddPlantSheet.swift
│   │   └── AddWantSheet.swift
│   │
│   ├── Swaps/
│   │   ├── SwapsListView.swift      ← inbox (pending/active/completed)
│   │   ├── SwapRowView.swift        ← list row
│   │   ├── SwapDetailView.swift     ← THE lifecycle screen
│   │   ├── AddressInputView.swift
│   │   ├── ShippingInstructionsView.swift
│   │   └── RatingView.swift
│   │
│   ├── Profile/
│   │   ├── ProfileView.swift
│   │   └── EditProfileView.swift
│   │
│   └── Shared/
│       ├── DesignSystem.swift       ← colors, fonts, spacing
│       ├── AvatarView.swift
│       ├── RatingStarsView.swift
│       ├── PillBadge.swift
│       ├── LoadingView.swift
│       └── EmptyStateView.swift
│
├── Services/
│   ├── AuthService.swift            ← Firebase phone auth
│   ├── UserService.swift            ← user CRUD
│   ├── PlantService.swift           ← plant + feed CRUD
│   ├── SwapService.swift            ← swap lifecycle
│   ├── ImageService.swift           ← photo upload to Storage
│   └── NotificationService.swift    ← FCM setup
│
├── Extensions/
│   ├── Color+Theme.swift
│   ├── View+Extensions.swift
│   └── Timestamp+Formatting.swift
│
└── Resources/
    ├── Assets.xcassets
    ├── PlantTypes.json              ← static list of common plants
    └── LaunchScreen.storyboard
```

---

## Build Order (Sprint Plan)

### Sprint 1 — Foundation (Week 1-2)
**Goal:** User can sign up, set up profile, and see the app shell.

- [ ] Create Xcode project with SwiftUI
- [ ] Set up Firebase project (Auth, Firestore, Storage)
- [ ] Implement design system (Color+Theme, typography, shared components)
- [ ] Build phone auth flow (Welcome → Phone → OTP → Profile Setup)
- [ ] Build tab bar navigation shell (4 empty tabs)
- [ ] Create Firestore user document on sign-up
- [ ] Profile photo upload to Firebase Storage

**Deliverable:** User can sign in with phone number and see main tab interface.

---

### Sprint 2 — My Garden (Week 3-4)
**Goal:** User can add plants they have and plants they want.

- [ ] Build GardenView with segmented control (Have / Want)
- [ ] Build AddPlantSheet (photo, name, type, notes)
- [ ] Photo picker integration (camera + library)
- [ ] Image upload to Firebase Storage, save URL to Firestore
- [ ] Build plant grid (2-column, with availability toggle)
- [ ] Build AddWantSheet (name, type)
- [ ] Want list UI (simple list with delete)
- [ ] Write to both `myPlants` subcollection AND `plantFeed` collection on add
- [ ] Edit and delete plants
- [ ] Create static PlantTypes.json with ~100 common houseplants for autocomplete

**Deliverable:** User can build their plant collection and wish list.

---

### Sprint 3 — Explore & Swipe (Week 5-6)
**Goal:** User can swipe through plants and propose matches/swaps.

- [ ] Build ExploreView with card stack
- [ ] Swipe gesture handling (left = pass, right = like)
- [ ] Card animations (rotation, opacity, snap-back)
- [ ] Query `plantFeed` excluding own plants and already-swiped
- [ ] Track swiped plants in `swipedPlants` subcollection
- [ ] PlantDetailView (tap to expand)
- [ ] "Request Clipping" action → creates swap doc (type: "request")
- [ ] "Propose Swap" action → ProposeSwapSheet → pick your plant → creates swap doc (type: "swap")
- [ ] Push notification to plant owner on new proposal

**Deliverable:** User can discover plants and send match/swap proposals.

---

### Sprint 4 — Swap Lifecycle (Week 7-8)
**Goal:** Full swap flow from proposal to completion.

- [ ] Build SwapsListView (Pending / Active / Completed segments)
- [ ] Build SwapDetailView with state machine UI
- [ ] Proposed state: Accept / Decline buttons
- [ ] Accepted state: Address input form
- [ ] Address exchange: only reveal after both submit
- [ ] Ready to Ship state: show addresses + shipping instructions
- [ ] "Mark as Shipped" button + optional tracking number
- [ ] "Mark as Received" button
- [ ] Push notifications at each state transition
- [ ] Decline flow + cancellation

**Deliverable:** Two users can complete a full swap end-to-end.

---

### Sprint 5 — Ratings & Polish (Week 9-10)
**Goal:** Rating system works, app feels polished and ready.

- [ ] Build RatingView (1-5 stars + optional comment)
- [ ] Save ratings to swap doc AND update user's ratingAvg/ratingCount
- [ ] Display ratings on profile and in explore cards
- [ ] Empty states for all screens (no plants yet, no swaps, etc.)
- [ ] Loading states and error handling
- [ ] Pull-to-refresh where appropriate
- [ ] Haptic feedback on swipe, match, and rating
- [ ] Transition animations (card stack, sheet presentations)
- [ ] App icon and launch screen
- [ ] TestFlight build

**Deliverable:** Complete, polished MVP ready for beta testing.

---

## Timeline Summary

| Sprint | Weeks | Focus | Milestone |
|---|---|---|---|
| 1 | 1-2 | Auth + shell | Users can sign in |
| 2 | 3-4 | My Garden | Users can add plants |
| 3 | 5-6 | Explore + swipe | Users can discover & propose |
| 4 | 7-8 | Swap lifecycle | Full swap flow works |
| 5 | 9-10 | Ratings + polish | Beta-ready MVP |

**Total: ~10 weeks to MVP**

---

## What's NOT in MVP (Intentionally)

- In-app payments / Stripe (users handle shipping cost themselves for now)
- In-app messaging / chat (users share enough info through the swap flow)
- AI plant identification
- Rarity tiers / cred points
- Search / filter on explore
- Nursery partnerships
- Android
- Admin dashboard

These all come after you validate the core loop: **add plants → discover → match → swap → rate.**
