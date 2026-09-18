# Scallop 🐚

## Technical Specification v0.1

**Status:** Prototype specification
**Scope:** EU + UK + Switzerland + Scandinavia + USA + Canada
**Primary purpose:** Multimodal door-to-door journey planning
**Booking:** Out of scope
**Prototype target:** Web application / PWA

---

## 1. Product definition

Scallop is a **multimodal journey-planning engine**.

The user provides:

* an origin
* a destination
* a date/time constraint
* optional return journey
* optional preferences

Scallop returns complete, comparable journeys across:

* air
* rail
* bus/coach
* ferry
* local public transport
* walking/cycling where useful
* private car
* EV

The critical distinction is that Scallop operates on **complete journeys**, not merely major transport legs.

For example:

> Dachau address → Toronto hotel

is a single Scallop journey containing potentially:

> walk → S-Bahn → airport rail → flight → airport transit → local transit → walking

The system should calculate whether each connection is actually feasible and expose the complete door-to-door journey.

---

# 2. Product principles

### P1 — Complexity is hidden

The engine can evaluate a very large number of possible combinations. The user should normally see only a small number of meaningful alternatives.

### P2 — Origin and destination are real-world locations

The user should not be forced to start at an airport or station.

### P3 — Time is a first-class constraint

"Arrive before 22:00" is more important than merely finding a flight departing at a particular time.

### P4 — Car is a genuine alternative

ICE and EV journeys must be evaluated on the same basis as public transport and flights.

### P5 — Cost means total journey cost

The displayed cost should include the relevant components of the journey, not just the ticket for the largest leg.

### P6 — Explainability

Whenever Scallop excludes or prefers a journey, it should be possible to explain why.

### P7 — Provider independence

External routing/search providers are replaceable components.

### P8 — No booking in v0.1

Scallop discovers, calculates and compares. It does not purchase tickets.

---

# 3. Geographic scope

## Phase 1 coverage

### Europe

* Austria
* Belgium
* Bulgaria
* Croatia
* Cyprus
* Czechia
* Denmark
* Estonia
* Finland
* France
* Germany
* Greece
* Hungary
* Ireland
* Italy
* Latvia
* Lithuania
* Luxembourg
* Malta
* Netherlands
* Poland
* Portugal
* Romania
* Slovakia
* Slovenia
* Spain
* Sweden

Plus:

* United Kingdom
* Switzerland
* Norway
* Iceland

### North America

* United States
* Canada

The architecture must remain geographically extensible.

Country coverage should therefore be represented in configuration rather than hard-coded throughout the routing engine.

---

# 4. Technology stack

## Frontend

**TypeScript**

**React + Next.js**

Responsibilities:

* search interface
* journey results
* journey detail
* maps
* user preferences
* journey history later

Recommended supporting technologies:

* Tailwind CSS
* MapLibre
* TanStack Query
* Zod
* Playwright

## Backend

**TypeScript + Node.js**

Architecture:

* REST API
* modular service architecture
* asynchronous provider calls
* caching
* PostgreSQL persistence

Recommended:

* Fastify or NestJS
* Zod for runtime API validation
* Pino for structured logging

I would start with a **modular monolith**, not microservices.

The module boundaries should be clean enough to extract services later if necessary.

## Database

**PostgreSQL + PostGIS**

PostGIS is required for geographic operations such as:

* nearby airport search
* nearby railway station search
* location clustering
* geographic containment
* route geometry
* country detection
* transfer-node discovery

## Transit routing

**OpenTripPlanner 2**

OTP 2 currently provides GTFS and Transmodel GraphQL APIs; its previous REST API was removed in 2025. GTFS is therefore the preferred initial data vocabulary for a GTFS-heavy implementation.

## Road routing

**HERE Routing API v8** for the prototype.

The current HERE API supports car routing, time-aware routing, toll information, route geometry and EV-specific routing.

Its EV routing can model battery state, consumption and charging curves and can automatically add charging stops while optimizing travel plus charging time.

## Flight data

Provider abstraction from day one.

Initial prototype candidate:

**Duffel**

Its current Offer Requests API accepts one or more slices, searches a range of airlines and returns flight offers with one or more segments.

Scallop must treat this as a replaceable source and should not design its domain model around Duffel's schema.

---

# 5. Repository structure

```text
scallop/
│
├── apps/
│   ├── web/
│   └── api/
│
├── packages/
│   ├── domain/
│   ├── journey-engine/
│   ├── optimizer/
│   ├── routing/
│   ├── flights/
│   ├── transit/
│   ├── roads/
│   ├── locations/
│   ├── costs/
│   ├── emissions/
│   ├── vehicles/
│   ├── providers/
│   └── ui/
│
├── infra/
│   ├── docker/
│   ├── postgres/
│   └── otp/
│
├── scripts/
│
├── tests/
│   ├── integration/
│   ├── routing/
│   └── fixtures/
│
└── docs/
```

---

# 6. Domain model

The most important design decision is to establish a provider-independent domain model.

## Location

```ts
type LocationType =
  | "address"
  | "city"
  | "airport"
  | "rail_station"
  | "bus_station"
  | "ferry_terminal"
  | "poi"
  | "transit_stop";

interface Location {
  id: string;
  name: string;
  type: LocationType;

  latitude: number;
  longitude: number;

  address?: string;
  countryCode?: string;
  timezone: string;

  providerRefs: ProviderReference[];
}
```

A location may represent an arbitrary address.

Example:

```text
Marienplatz 1, München
```

must be resolvable into a physical coordinate and timezone before routing starts.

---

# 7. Transport modes

```ts
type TransportMode =
  | "walk"
  | "bike"
  | "car"
  | "ev"
  | "bus"
  | "tram"
  | "metro"
  | "subway"
  | "rail"
  | "flight"
  | "ferry";
```

The engine should not assume that every mode is available everywhere.

---

# 8. Journey Leg

Every journey consists of ordered legs.

```ts
interface JourneyLeg {
  id: string;

  mode: TransportMode;

  origin: Location;
  destination: Location;

  departureTime: string;
  arrivalTime: string;

  durationSeconds: number;
  distanceMeters?: number;

  operator?: Operator;

  cost?: Cost;
  emissions?: Emissions;

  source: ProviderReference[];

  geometry?: GeoJSON;

  metadata?: Record<string, unknown>;
}
```

Examples:

```text
Dachau Bahnhof
    ↓
S2
    ↓
München Hbf
```

or:

```text
MUC
    ↓
Air Canada AC...
    ↓
YYZ
```

---

# 9. Transfer

Transfers are first-class objects.

```ts
interface Transfer {
  fromLegId: string;
  toLegId: string;

  location: Location;

  transferDurationSeconds: number;
  requiredDurationSeconds: number;

  transferType:
    | "same_platform"
    | "same_station"
    | "station_change"
    | "airport_transfer"
    | "security"
    | "immigration"
    | "baggage"
    | "customs"
    | "self_transfer";

  riskLevel: "low" | "medium" | "high";

  notes?: string[];
}
```

This lets the system distinguish:

> Frankfurt Airport → same-terminal connection

from:

> Heathrow Terminal 5 → Heathrow Terminal 2

and eventually:

> self-transfer requiring baggage collection and re-check.

---

# 10. Journey

```ts
interface Journey {
  id: string;

  origin: Location;
  destination: Location;

  departureTime: string;
  arrivalTime: string;

  durationSeconds: number;

  legs: JourneyLeg[];
  transfers: Transfer[];

  totalCost: CostSummary;
  emissions: Emissions;

  distanceMeters: number;

  changes: number;

  attributes: JourneyAttributes;

  scoreVector?: ScoreVector;

  explanation?: JourneyExplanation;
}
```

---

# 11. Cost model

Cost must be decomposable.

```ts
interface Cost {
  amount: number;
  currency: string;
  type: CostType;
  estimated: boolean;
  source?: ProviderReference;
}
```

```ts
type CostType =
  | "airfare"
  | "rail"
  | "bus"
  | "transit"
  | "ferry"
  | "fuel"
  | "electricity"
  | "charging"
  | "tolls"
  | "parking"
  | "rental"
  | "other";
```

A journey produces:

```ts
interface CostSummary {
  total: Money;

  components: Cost[];

  estimatedComponents: Cost[];

  methodologyVersion: string;
}
```

All money calculations should use decimal-safe representations.

---

# 12. ICE vehicle model

```ts
interface CombustionVehicle {
  type: "ice";

  fuelType: "petrol" | "diesel";

  consumptionLitresPer100Km: number;

  tankCapacityLitres?: number;

  startingFuelLitres?: number;

  passengers: number;
}
```

Calculation:

```text
fuelNeeded
=
distanceKm
× consumptionLPer100Km
÷ 100
```

Then:

```text
fuelCost
=
fuelNeeded × estimatedFuelPrice
```

Future versions can calculate actual recommended refueling locations rather than using one blended fuel price.

---

# 13. EV model

```ts
interface ElectricVehicle {
  type: "ev";

  batteryCapacityKWh: number;

  initialChargeKWh: number;

  consumptionKWhPer100Km?: number;

  connectorTypes: string[];

  maxChargingPowerKW?: number;

  chargingCurve?: ChargingCurve;

  minimumArrivalChargeKWh?: number;
}
```

For v0.1, Scallop may delegate the detailed physical routing problem to HERE while maintaining its own normalized output model.

The current HERE EV routing API supports:

* initial charge
* maximum charge
* charging curve
* minimum charge constraints
* charging setup duration
* automatic charging-stop placement
* preferred charging brands/operators.

---

# 14. Fuel pricing

Version 0.1:

Use country-level or regional estimates.

Store:

```ts
interface EnergyPrice {
  countryCode: string;
  energyType: "petrol" | "diesel" | "electricity";

  pricePerUnit: number;
  currency: string;

  validFrom: string;
  validTo?: string;

  source: ProviderReference;
}
```

For Europe, the EU Oil Bulletin provides weekly national consumer fuel-price data and is therefore a useful source for a European baseline.

Later versions can move to station-specific prices.

---

# 15. Charging model

Charging is part of the route.

```ts
interface ChargingStop {
  location: Location;

  arrivalTime: string;
  departureTime: string;

  arrivalChargeKWh: number;
  departureChargeKWh: number;

  energyAddedKWh: number;

  chargingDurationSeconds: number;

  estimatedCost?: Money;

  operator?: string;
  connectorType?: string;
}
```

This allows Scallop to show:

```text
⚡ 3 charging stops
58 min charging
€47 estimated charging
```

rather than simply saying:

> EV: 12h 30m

---

# 16. Flight abstraction

Scallop must not depend on a flight supplier's object model.

```ts
interface FlightProvider {
  search(request: FlightSearchRequest): Promise<FlightOffer[]>;
}
```

```ts
interface FlightSearchRequest {
  originAirportCodes: string[];
  destinationAirportCodes: string[];

  departureEarliest: string;
  departureLatest?: string;

  arrivalEarliest?: string;
  arrivalLatest?: string;

  passengers: Passenger[];
  cabinClass: "economy" | "premium_economy" | "business" | "first";

  maxConnections?: number;
}
```

A flight offer is normalized:

```ts
interface FlightSegment {
  carrier: Carrier;
  flightNumber: string;

  origin: Airport;
  destination: Airport;

  departureTime: string;
  arrivalTime: string;

  aircraft?: string;
}
```

Duffel currently supports multi-segment offers and maximum-connection constraints, which makes it a reasonable prototype provider.

---

# 17. Transit abstraction

```ts
interface TransitRouter {
  route(request: TransitRouteRequest): Promise<TransitJourney[]>;
}
```

```ts
interface TransitRouteRequest {
  origin: Location;
  destination: Location;

  departureTime?: string;
  arrivalTime?: string;

  modes: TransportMode[];

  maxWalkingDistanceMeters?: number;
}
```

OTP handles the detailed transit network.

The resulting transit itinerary is then normalized into Scallop legs.

GTFS currently defines schedule data for routes, stops, trips, transfers and related station information; GTFS-Realtime extends that ecosystem with dynamic information such as trip updates and service alerts.

---

# 18. Road abstraction

```ts
interface RoadRouter {
  routeCar(request: RoadRouteRequest): Promise<RoadJourney>;

  routeEV(request: EVRouteRequest): Promise<EVJourney>;
}
```

The router should return:

* distance
* duration
* route geometry
* tolls
* traffic-aware ETA
* ferry segments
* energy consumption
* charging stops where relevant

HERE's current routing API supports time-aware routing, including traffic and time-dependent road conditions.

---

# 19. Provider architecture

Every external data source is behind an adapter.

```text
Provider
   │
   ├── HERE
   ├── OTP
   ├── GTFS feeds
   ├── Duffel
   └── Future providers
```

Never allow provider-specific types to leak into the domain layer.

Example:

```text
DuffelOffer
     ↓
DuffelAdapter
     ↓
Scallop FlightOffer
     ↓
JourneyEngine
```

This is one of the highest-priority architectural rules.

---

# 20. Search pipeline

A Scallop request follows this sequence:

```text
1. Parse request
       ↓
2. Geocode origin/destination
       ↓
3. Identify nearby transport nodes
       ↓
4. Generate candidate major transport corridors
       ↓
5. Query relevant providers
       ↓
6. Generate local access/egress journeys
       ↓
7. Build temporal journey graph
       ↓
8. Validate transfer feasibility
       ↓
9. Calculate cost
       ↓
10. Calculate emissions
       ↓
11. Remove dominated journeys
       ↓
12. Rank by requested objective
       ↓
13. Return a small set of alternatives
```

---

# 21. Nearby node discovery

For a destination such as Toronto, Scallop should identify:

```text
YYZ
YTZ
Union Station
major regional rail
major bus terminals
local transit nodes
```

For Dachau:

```text
Dachau Bahnhof
Munich Hbf
MUC
possibly FRA
```

This is where PostGIS becomes important.

Example conceptual query:

```sql
SELECT *
FROM transport_nodes
WHERE ST_DWithin(
  geography(location),
  geography(:originPoint),
  :radiusMeters
)
ORDER BY ST_Distance(
  geography(location),
  geography(:originPoint)
);
```

---

# 22. Candidate generation

Do not initially generate every possible combination.

Use a funnel.

### Stage A — Major corridors

Find candidate:

* direct flights
* one-stop flights
* major rail corridors
* major road routes
* major ferry crossings

### Stage B — Access

For each major node, calculate:

> origin → node

### Stage C — Egress

Calculate:

> node → destination

### Stage D — Connect

Combine candidates.

This dramatically reduces the search space.

---

# 23. Temporal graph

The graph is time-dependent.

A simplified edge:

```ts
interface TemporalEdge {
  originNode: string;
  destinationNode: string;

  departureTime: string;
  arrivalTime: string;

  minimumTransferTime?: number;

  leg: JourneyLeg;
}
```

An edge can only follow another edge when:

```text
next.departure
>=
previous.arrival + requiredTransfer
```

This is fundamental to Scallop.

---

# 24. Airport connection rules

Different transfer types require different buffers.

Version 0.1 should maintain configurable rules such as:

```ts
interface ConnectionRule {
  context:
    | "domestic_to_domestic"
    | "domestic_to_international"
    | "international_to_domestic"
    | "international_to_international"
    | "rail_to_flight"
    | "flight_to_rail"
    | "self_transfer";

  minimumSeconds: number;
}
```

Do not hard-code a universal "2 hours".

The required buffer depends on:

* airport
* terminal
* international status
* baggage
* immigration
* self-transfer
* protected itinerary status

The rule engine should therefore remain configurable.

---

# 25. Protected vs unprotected connections

Scallop should distinguish:

### Protected

Example:

> airline itinerary with connecting flight

### Unprotected

Example:

> independently purchased train + flight

The UI can display:

```text
🟢 Protected connection
```

versus:

```text
🟡 Separate tickets
```

without pretending the latter is guaranteed.

---

# 26. Journey scoring

Do not create one universal "best route".

Instead calculate dimensions:

```ts
interface ScoreVector {
  durationSeconds: number;
  totalCost: number;
  changes: number;
  walkingSeconds: number;
  transferRisk: number;
  emissionsKgCO2e: number;
  drivingSeconds: number;
}
```

The engine then generates a **Pareto frontier**.

A journey is dominated if another journey is:

* no slower
* no more expensive
* no more complex
* no higher-emission

with at least one dimension strictly better.

This eliminates huge numbers of redundant alternatives.

---

# 27. Result categories

The frontend can expose friendly interpretations of the Pareto frontier.

Examples:

**⚡ Fastest**

Minimum door-to-door duration.

**💶 Cheapest**

Minimum total estimated cost.

**🧘 Simplest**

Fewest meaningful transfers subject to acceptable duration.

**🚗 Car**

Best private-car alternative.

**🌿 Lower emissions**

Low-emission alternative subject to a reasonable travel-time constraint.

These are optimization views, not absolute judgments.

---

# 28. "Destination time" metric

This should become a signature Scallop feature.

For a round trip:

```text
destinationTime
=
time physically present at destination
```

Example:

```text
Option A
Arrive Toronto Friday 15:20
Leave Toronto Saturday 19:00

Destination time:
27h 40m
```

versus:

```text
Option B
Arrive Friday 21:45
Leave Saturday 18:00

Destination time:
20h 15m
```

Scallop can therefore compare not simply:

> "Flight duration"

but:

> **How much useful time does this itinerary actually give me?**

For a short trip this can be more useful than raw transport duration.

---

# 29. Round-trip optimization

The outbound and return journeys must not be optimized independently.

Example:

```text
Outbound
Friday:
Dachau → Toronto

Return constraint:
Back in Dachau by Sunday 22:00
```

The engine must derive:

> Toronto departure must occur sufficiently early on Saturday to allow arrival in Europe + onward transport Sunday.

Therefore the return search should work backwards from the destination deadline.

This is a major feature.

---

# 30. Backward planning

For an arrival deadline:

```text
Destination:
Dachau
Sunday 22:00
```

Scallop should calculate backwards:

```text
Dachau
≤ 22:00
   ↑
Munich
   ↑
MUC
   ↑
flight arrival
   ↑
flight departure
   ↑
YYZ access
```

This identifies the **latest feasible departure** rather than merely presenting arbitrary schedules.

---

# 31. Example: Dachau → Toronto prototype

Input:

```json
{
  "origin": "Dachau, Germany",
  "destination": "Toronto, Canada",
  "outbound": {
    "date": "2026-10-09",
    "arriveBy": "22:00"
  },
  "return": {
    "date": "2026-10-11",
    "arriveBackBy": "22:00"
  }
}
```

Scallop might discover:

```text
OUTBOUND

Dachau
 ↓ S-Bahn
München Hbf
 ↓ S-Bahn
MUC
 ↓ flight
Toronto
 ↓ local transit
Toronto destination
```

and:

```text
CAR

Dachau
 ↓ motorway
Munich
 ↓
ferry/road if applicable
 ↓
Toronto
```

The result should also make the impossibility of an overly-late Sunday Toronto departure evident.

---

# 32. API

## POST `/api/search`

Request:

```json
{
  "origin": {
    "query": "Dachau, Germany"
  },
  "destination": {
    "query": "Toronto, Canada"
  },
  "outbound": {
    "date": "2026-10-09",
    "earliestDeparture": "05:00",
    "latestArrival": "22:00"
  },
  "return": {
    "date": "2026-10-11",
    "latestArrival": "22:00"
  },
  "passengers": {
    "adults": 1
  },
  "preferences": {
    "priority": "balanced",
    "maxChanges": 4,
    "allowOvernight": true
  }
}
```

Response:

```json
{
  "searchId": "srch_123",
  "status": "complete",
  "journeys": [
    {
      "id": "journey_1",
      "category": "fastest",
      "durationSeconds": 34200,
      "totalCost": {
        "amount": 684,
        "currency": "EUR"
      }
    }
  ]
}
```

---

# 33. GET `/api/search/:id`

Returns the full normalized journeys.

Useful for:

* progressive loading
* caching
* client-side navigation
* eventual async provider searches

---

# 34. GET `/api/journeys/:id`

Returns one detailed itinerary.

Example:

```json
{
  "id": "journey_1",
  "origin": {},
  "destination": {},
  "legs": [],
  "transfers": [],
  "cost": {},
  "emissions": {},
  "explanation": {}
}
```

---

# 35. GET `/api/locations/search`

```text
GET /api/locations/search?q=Dachau
```

Returns:

* addresses
* cities
* airports
* stations
* POIs

with provider-neutral normalized locations.

---

# 36. Vehicle API

## POST `/api/vehicles`

```json
{
  "name": "My Tesla",
  "type": "ev",
  "batteryCapacityKWh": 75,
  "initialChargePercent": 90,
  "connectorTypes": ["CCS"]
}
```

---

# 37. Explanation engine

Every journey should have machine-generated reasons.

```ts
interface JourneyExplanation {
  positives: ExplanationItem[];
  warnings: ExplanationItem[];
  constraintsSatisfied: string[];
  constraintsViolated: string[];
}
```

Example:

```text
Why this route?

✓ Arrives 4h 31m before your deadline
✓ Only 1 major transfer
✓ Protected flight itinerary
✓ €62 cheaper than the fastest alternative

Note:
⚠ 54 minute airport rail transfer
```

This is critical to building trust.

---

# 38. Data freshness

Every result needs a freshness indicator.

```ts
interface DataFreshness {
  provider: string;
  retrievedAt: string;
  validUntil?: string;
}
```

Flight prices are inherently ephemeral. The Duffel API notes that offers can expire and prices may change, so Scallop should explicitly distinguish **estimated/search-time information** from guaranteed booking availability.

Since Scallop does not book, this is acceptable.

The UI should say:

> **Estimated price**

rather than imply a guaranteed fare.

---

# 39. Caching

Use Redis later, but PostgreSQL-based caching is acceptable for the first prototype.

Cache:

* geocoding
* airport/station metadata
* transit queries
* road routes
* flight search results
* energy prices

Never cache dynamic data indefinitely.

---

# 40. First database schema

Initial tables:

```text
users
locations
location_provider_refs

transport_nodes
operators
carriers

journeys
journey_legs
journey_transfers

flight_segments
transit_legs
road_legs
charging_stops

vehicles
energy_prices

emission_factors

provider_requests
provider_responses

journey_history
```

The provider-specific tables can be expanded later.

---

# 41. Emissions

Create a dedicated versioned emissions engine.

```ts
interface Emissions {
  kgCO2e: number;

  methodologyVersion: string;

  confidence: "high" | "medium" | "low";

  baseline?: EmissionsBaseline;
}
```

Scallop must never claim:

> "You saved exactly 48.2 kg CO₂."

without defining:

* baseline
* occupancy
* fuel assumptions
* electricity assumptions
* flight-emission methodology

Instead:

> **Estimated: 48 kg CO₂e**

with methodology available on demand.

---

# 42. Future gamification data foundation

Do not surface this heavily in v0.1, but store the primitives.

```ts
interface JourneyHistoryEntry {
  journeyId: string;
  completedAt: string;

  originCountry?: string;
  destinationCountry?: string;

  countriesVisited: string[];

  distanceMeters: number;

  emissionsKgCO2e: number;

  baselineEmissionsKgCO2e?: number;

  modeBreakdown: Record<TransportMode, number>;
}
```

This allows future:

* visited countries
* travelled distance
* mode achievements
* lower-emission milestones
* Shells 🐚
* journey history

without changing the core journey engine.

---

# 43. Gamification principle

Gamification is a **retention layer**, not a navigation layer.

Do not put:

> "Earn 50 shells!"

on the search screen.

Instead, later provide:

```text
🐚 Your travels

17 countries
24,820 km
312 kg CO₂e avoided
```

The interface remains calm.

---

# 44. UX specification

## Visual language

Direction:

* natural
* calm
* slightly cartoon-like
* rounded
* organic
* clean
* muted colours
* obvious symbols
* generous whitespace

Suggested palette direction:

```text
shell / warm off-white
sand
sage
muted ocean
soft stone
warm charcoal
```

No neon travel-app gradients.

The 🐚 shell should inspire the visual language rather than become an intrusive mascot.

## UX rule

**One question per screen.**

Search:

> Where and when?

Results:

> Which journey?

Journey detail:

> How does it work?

Preferences:

> What matters to me?

---

# 45. Primary search UX

Initial screen:

```text
              🐚
           Scallop

From
[Dachau, Germany          ]

To
[Toronto, Canada          ]

[ 9 Oct ] → [ 11 Oct ]

          Find my way
```

Advanced constraints remain hidden until requested.

Example:

```text
+ Add time limits
```

opens:

```text
Leave after
Arrive before
Back by
```

---

# 46. Result cards

Only a handful should initially be presented.

Example:

```text
⚡ FASTEST

Dachau → Toronto

9h 18m
€684

🚆 → ✈️ → 🚆

Arrives 15:29

View journey →
```

Then:

```text
💶 LOWEST COST
🧘 SIMPLEST
🚗 CAR
🌿 LOWER EMISSIONS
```

More alternatives are available behind:

> See more journeys

---

# 47. Detailed itinerary

Primary view:

```text
🏠 Dachau
06:18

   │
   🚆
   │
München Hbf
06:58

   │
   🚆
   │
MUC
07:42

   │
   ✈️
   │
YYZ
14:35

   │
   🚆
   │
Toronto
15:29
```

Each leg can expand to show detail.

---

# 48. Architecture diagram

```text
                         SCALLOP 🐚
                              │
                    ┌─────────┴─────────┐
                    │     Web App       │
                    │ React / Next.js   │
                    └─────────┬─────────┘
                              │
                         REST API
                              │
                ┌─────────────┴─────────────┐
                │     Journey Orchestrator  │
                └─────────────┬─────────────┘
                              │
             ┌────────────────┼─────────────────┐
             │                │                 │
        Location          Transport         Optimization
          Layer             Layer               Layer
             │                │                 │
        PostGIS        ┌──────┼───────┐      Cost
                       │      │       │      CO₂
                      OTP   HERE   Flights
                       │      │       │
                     GTFS   Roads   Duffel
```

---

# 49. Development milestones

## Milestone 1 — Skeleton

Deliver:

* GitHub repository
* Next.js application
* Node API
* PostgreSQL/PostGIS
* Docker
* CI
* linting
* tests
* environment configuration

Success criterion:

> Application starts locally with one command.

---

## Milestone 2 — Location engine

Deliver:

* geocoding
* normalized Location model
* nearby airport/station lookup
* timezone resolution

Success criterion:

> "Dachau" can become a usable geographic origin and produce nearby transport nodes.

---

## Milestone 3 — Car

Deliver:

* normal car routing
* distance
* time
* tolls
* fuel cost

Success criterion:

> Dachau → Paris returns a complete car journey with estimated total cost.

---

## Milestone 4 — EV

Deliver:

* vehicle profile
* battery state
* charging stops
* charging time
* charging cost

Success criterion:

> Dachau → Paris can produce ICE and EV alternatives.

---

## Milestone 5 — Transit

Deliver:

* OTP
* GTFS
* local public transit
* station access

Success criterion:

> Dachau → MUC produces actual local transport legs.

---

## Milestone 6 — Flights

Deliver:

* flight provider adapter
* airport search
* schedules/offers
* normalized flight legs

Success criterion:

> MUC/YUL/YYZ combinations can enter the Scallop journey graph.

---

## Milestone 7 — Journey engine

Deliver:

* temporal graph
* transfer rules
* valid journey composition
* protected/unprotected connection logic

Success criterion:

> Scallop can produce complete Dachau → Toronto itineraries.

---

## Milestone 8 — Optimization

Deliver:

* fastest
* cheapest
* simplest
* car
* lower emissions

Success criterion:

> Scallop returns materially different but valid journey alternatives rather than duplicates.

---

## Milestone 9 — Round trips

Deliver:

* outbound + return coupling
* latest feasible departure
* arrival deadline
* destination-time metric

Success criterion:

> The October Dachau/Brussels/Toronto-type problem can be solved from a single query.

---

## Milestone 10 — UX

Deliver:

* calm Scallop visual identity
* search
* result cards
* detailed timeline
* map
* explanations

Success criterion:

> A non-technical user can understand the complete journey without needing to understand the underlying routing engine.

---

# 50. Initial test suite

Use test scenarios specifically designed to break the system.

### Scenario A

```text
Dachau → Munich Airport
```

Tests local access.

### Scenario B

```text
Brussels → Paris
```

Tests rail.

### Scenario C

```text
Brussels → London
```

Tests international rail + border considerations.

### Scenario D

```text
Munich → Toronto
```

Tests flights.

### Scenario E

```text
Dachau → Toronto
```

Tests true multimodal orchestration.

### Scenario F

```text
Munich → Milan
```

Tests ICE + EV + rail.

### Scenario G

```text
New York → Toronto
```

Tests US/Canada.

### Scenario H

```text
Toronto → Brussels
```

Tests overnight return and timezone logic.

### Scenario I

```text
Toronto → Dachau,
arrive by Sunday 22:00
```

Tests backward planning.

---

# 51. Prototype success definition

Scallop v0.1 is successful when this query works:

> **Dachau → Toronto**
>
> Friday 9 October
>
> Arrive Toronto before 22:00
>
> Return 11 October
>
> Be physically back in Dachau by 22:00

And Scallop can return several complete itineraries such as:

```text
FAST
local transit → flight → local transit

SIMPLE
car → flight → local transit

CHEAP
rail → flight → local transit

CAR
ICE

EV
EV + charging
```

with:

* complete door-to-door timing
* total estimated cost
* individual cost components
* transfer times
* connection warnings
* approximate emissions
* explanation of why the journey qualifies

---

# 52. What is explicitly out of scope for v0.1

Do not build:

* ticket booking
* payments
* loyalty-program integration
* airline accounts
* hotel booking
* rental-car booking
* social network
* messaging
* sophisticated gamification
* native iOS/Android
* global geographic coverage
* perfect live disruption prediction

These can come later.

---

# 53. First GitHub issues

The repository should immediately contain these epics:

```text
EPIC-001 Foundation
EPIC-002 Location Intelligence
EPIC-003 Road Routing
EPIC-004 EV Routing
EPIC-005 Transit
EPIC-006 Flights
EPIC-007 Journey Graph
EPIC-008 Cost Engine
EPIC-009 Emissions
EPIC-010 Optimization
EPIC-011 Round Trip
EPIC-012 UX/UI
EPIC-013 Journey History
```

The first actual implementation issues:

```text
SC-001 Initialize monorepo
SC-002 Docker PostgreSQL/PostGIS
SC-003 Create domain types
SC-004 Create Location model
SC-005 Create geocoder adapter
SC-006 Create road-router adapter
SC-007 Implement car cost calculation
SC-008 Implement EV vehicle model
SC-009 Add OTP container
SC-010 Create transit adapter
SC-011 Create flight provider interface
SC-012 Implement first flight provider
SC-013 Create temporal graph
SC-014 Implement journey composition
SC-015 Implement transfer validation
SC-016 Implement journey cost aggregation
SC-017 Implement Pareto filtering
SC-018 Build /search endpoint
SC-019 Build results UI
SC-020 Build itinerary UI
SC-021 Dachau → Toronto integration test
```

---

# 54. Architectural north star

The most important sentence in the project documentation should be:

> **Scallop does not search for a flight, train or car route. Scallop searches for a journey.**

Everything else follows from that.

The external systems provide transportation intelligence.

Scallop provides the layer that:

1. understands the user's actual origin and destination;
2. understands temporal constraints;
3. finds compatible connections;
4. joins transport modes;
5. calculates complete cost;
6. calculates emissions;
7. evaluates alternatives;
8. explains the result simply.

That orchestration layer is the product.

---

# 55. Longer-term architecture

Once the prototype works, the system can evolve toward:

```text
                 Scallop Journey Graph
                          │
       ┌──────────────────┼───────────────────┐
       │                  │                   │
    Real-time          Prediction          Personal
       │                  │                   │
 delays/cancellations   reliability        history
 traffic                confidence          preferences
 weather                disruption          gamification
       │                  │                   │
       └──────────────────┼───────────────────┘
                          │
                     Scallop 🐚
```

The user experience should remain simple even as the engine becomes extremely sophisticated.

---

## Initial technology decision summary

**Use TypeScript as the primary language.**

**Use PostgreSQL/PostGIS as the core datastore.**

**Use OpenTripPlanner 2 for public-transport routing.**

**Use HERE for road + EV routing.**

**Use a flight-provider abstraction, initially implemented against Duffel.**

**Use a modular monolith rather than microservices for the prototype.**

**Build a temporal multimodal journey graph as Scallop's central intellectual component.**

**Implement cost, emissions and optimization as independent engines.**

**Build the web UI after the journey engine can solve real end-to-end examples.**

The external APIs chosen above are current as of September 2026: OTP 2.10 is the current documented latest branch, HERE's Routing API v8 is actively maintained, and the current GTFS specification was revised April 27, 2026.
