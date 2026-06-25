# 📈 Stock Trading & 🏨 Hotel Reservation — Design Deep Dive

> Money apps and booking apps. Both involve "don't mess this up" levels of data accuracy.

---

## 📈 PART 1: STOCK TRADING APP (Think Robinhood / eToro / Trade Republic)

---

### Q: "Design a stock trading app." What makes this special?

**A:** Two words: **real-time** and **money**.

Unlike a social feed where a 3-second delay is fine, in trading:
- Stock prices change **multiple times per second**
- A 2-second delay could mean buying at the wrong price
- Incorrect data = financial loss = lawsuits = very bad day

So the core challenge is: *How do you stream constantly-changing data to a phone without melting the battery or crashing the app?*

Scope it:
- ✅ View real-time stock prices
- ✅ View portfolio (what stocks you own)
- ✅ Place buy/sell orders
- ✅ See order history
- ✅ Historical price charts
- ✅ Price alerts (push notifications)
- ❌ Out of scope: options trading, crypto, social features

---

### Q: How does the API work when you need both REST AND real-time data?

**A:** **Hybrid approach** — just like the chat app, but for different reasons:

| What | Protocol | Why |
|------|----------|-----|
| Portfolio, order history, placing orders | **REST** | Standard CRUD stuff — doesn't change every millisecond |
| Live stock prices, order status updates | **WebSocket** | Data streams in constantly — polling would be insane |

The WebSocket connection looks like:

```
Connect to: wss://api.stocktrading.com/ws

// Client subscribes to specific stocks:
{
  "action": "subscribe",
  "channels": [
    { "name": "stock-prices", "symbols": ["AAPL", "GOOGL", "TSLA"] }
  ]
}

// Server streams back price updates:
{ "type": "stock-prices", "symbol": "AAPL", "price": 187.42, "change": +1.23 }
{ "type": "stock-prices", "symbol": "TSLA", "price": 245.67, "change": -3.45 }
```

You only subscribe to stocks the user is actually looking at. No point streaming prices for 5000 stocks if they only own 3.

---

### Q: How do you display prices that change multiple times per second without the UI going haywire?

**A:** Great question — this is the **#1 deep dive** for trading apps.

If you update the UI every time a price arrives, you'll get:
- 🔥 UI janking (too many redraws per second)
- 🔋 Battery drain from hell
- 📱 The app feeling like a strobe light

The solution: **Buffered updates**.

Think of it like a mailroom: instead of running to your desk every time a letter arrives, the mailroom collects letters for 1-2 seconds, then delivers a batch.

```
WebSocket → prices arrive constantly
    ↓
Buffer (collects for 1-2 seconds)
    ↓
UI update (one smooth update with all new prices)
```

| Strategy | Pros | Cons |
|----------|------|------|
| **Direct updates** (every price tick) | Most current data | UI jank, battery killer |
| **Buffered updates** (batch every 1-2s) ✅ | Smooth UI, battery friendly | Tiny delay (acceptable) |
| **Priority-based** (important stuff updates faster) | Smart resource use | Complex to implement |

Buffered is the sweet spot for most trading apps.

**Implementing buffered updates in React Native / JavaScript:**

```typescript
import { useRef, useCallback, useEffect } from 'react';

function useBufferedPriceUpdates(
  onFlush: (prices: Record<string, number>) => void,
  intervalMs = 1000
) {
  const buffer = useRef<Record<string, number>>({});

  useEffect(() => {
    const timer = setInterval(() => {
      if (Object.keys(buffer.current).length > 0) {
        onFlush({ ...buffer.current });
        buffer.current = {}; // Clear after flush
      }
    }, intervalMs);
    return () => clearInterval(timer);
  }, [onFlush, intervalMs]);

  return useCallback((symbol: string, price: number) => {
    buffer.current[symbol] = price; // Latest price overwrites — no duplicates
  }, []);
}

// Usage
const addPriceUpdate = useBufferedPriceUpdates((prices) => {
  setPrices(prev => ({ ...prev, ...prices })); // Merge into state
});

socketService.onPriceUpdate = (symbol, price) => {
  addPriceUpdate(symbol, price); // Cheap — just writes to a ref, no re-render
};
```

Key insight worth mentioning: **throttle vs debounce**. For stock prices you want `throttle` (execute once every N ms, take latest value) not `debounce` (wait for silence). Debounce would never fire during a fast-moving market.

```typescript
// One-liner with lodash (mention this as an alternative)
import throttle from 'lodash/throttle';
const updatePrices = throttle((prices) => setPrices(prices), 1000);
```

---

### Q: What about those fancy stock price charts?

**A:** Charts are surprisingly complex. There are two types:

**Historical charts** (1 day, 1 week, 1 year views):
- Data comes from REST API
- Can have 1000+ data points for a yearly view
- Rendered using charting libraries

**Real-time intraday chart** (the one that moves as you watch):
- Data starts from REST API (today's prices so far)
- Then WebSocket fills in new data points as they come
- Also uses buffered updates to avoid redrawing every millisecond

**How to render charts on mobile?** Three options:

| Approach | Pros | Cons |
|----------|------|------|
| Native canvas drawing | Best performance | Lots of code, platform-specific |
| Third-party libraries (MPAndroidChart, ChartsOrg) | Quick to implement | Some performance overhead |
| **WebView-based** (TradingView, D3.js) ✅ | Works on both platforms, rich features | WebView overhead, less "native" feel |

Many real apps use **WebView-based** rendering because the same chart code works on iOS, Android, AND web. eToro does this with TradingView. Robinhood went native for maximum performance with their Spark library.

**Charting libraries in React Native specifically:**

| Library | Technology | Best for |
|---------|-----------|---------|
| **Victory Native XL** | React Native Skia (GPU-rendered) | Production trading apps — smooth real-time updates |
| **react-native-gifted-charts** | SVG-based | Simple charts with a clean API, good for most use cases |
| **TradingView widget (WebView)** | HTML/Canvas | Copying the industry-standard trading UI, works on all platforms |
| **react-native-chart-kit** | SVG | Quick prototyping — performance issues at scale |

For a trading app, **Victory Native XL** with React Native Skia is the right answer. Skia is the same rendering engine Flutter uses — it draws directly to the GPU, making real-time price chart updates smooth at 60fps without JS thread involvement:

```tsx
import { CartesianChart, Line } from 'victory-native';

function StockChart({ data }: { data: PricePoint[] }) {
  return (
    <CartesianChart
      data={data}
      xKey="timestamp"
      yKeys={["price"]}
      domainPadding={{ top: 20 }}
    >
      {({ points }) => (
        <Line
          points={points.price}
          color="#00C851"
          strokeWidth={2}
          animate={{ type: 'timing', duration: 300 }}
        />
      )}
    </CartesianChart>
  );
}
```

Mention in interview: *"Since the chart updates every second, I'd use Victory Native XL which renders on the UI thread via Skia — the JS thread being busy with WebSocket messages won't cause chart jank."*

The data flow for real-time charts:
```
WebSocket → Stocks Repository (buffers data) → ViewModel → JavaScript Bridge → WebView Chart
```

That **JavaScript Bridge** is key — it's how native code talks to the WebView. Native handles the data; WebView handles the pretty pictures.

---

### Q: What about the architecture?

**A:**

```
┌──────────── UI LAYER ─────────────────┐
│  Home Screen     Stock Detail Screen   │
│  (Portfolio +    (Chart + Price +       │
│   Recent Orders)  Place Order Modal)   │
├──────────── DATA LAYER ───────────────┤
│  Portfolio Repository                  │
│  Stocks Repository (WebSocket + REST) │
│  Orders Repository                     │
│  Push Notifications Client            │
├──────────── EXTERNAL ─────────────────┤
│  Backend │ CDN │ API Gateway          │
│  Push Provider                        │
└────────────────────────────────────────┘
```

**Data storage rule of thumb for trading apps:**

| Data type | Storage | Why |
|-----------|---------|-----|
| Real-time prices | **In-memory only** | Stale prices are dangerous — never persist old prices to disk |
| Order history | Database (Room/CoreData) | Structured data, rarely changes, needs offline access |
| Historical chart data | Disk cache with TTL | Big datasets, cacheable, but needs expiration |
| User preferences | Key-value store | Simple settings |

---

### Q: How does the API Gateway matter here?

**A:** In a *money app*, the API Gateway isn't optional — it's your **security guard**:

- 🔐 **Authentication**: Verify every request (JWT tokens)
- 🚦 **Rate limiting**: Prevent abuse / DDoS
- 🕵️ **Suspicious activity detection**: Flagging weird patterns
- ⚖️ **Load balancing**: Especially during market open/close when traffic spikes
- 📊 **Monitoring**: Track every request for compliance audits

Trading apps are under regulatory scrutiny. Every trade needs an audit trail.

---

## 🏨 PART 2: HOTEL RESERVATION APP (Think Booking.com / Expedia / Airbnb)

---

### Q: "Design a hotel reservation app." What's the core challenge?

**A:** The tricky part isn't showing hotels — it's the **booking flow**. Specifically:

*What happens when 50 people try to book the last room at the same time?*

That's a **concurrency** problem, and it makes this interview way more interesting than just "show a list of hotels."

Scope it:
- ✅ Search hotels by location/dates
- ✅ Autocomplete suggestions while typing
- ✅ View hotel details and room types
- ✅ Book rooms with a timed reservation hold
- ✅ Pay via credit card or third-party (PayPal, Google Pay)
- ✅ View confirmed reservations offline
- ❌ Out of scope: Cancellations, check-in, reviews, push notifications

---

### Q: How does the API look? Pretty standard REST?

**A:** Yep, no real-time needed here. Pure REST:

```
GET  /v1/hotels/search?lat={x}&lng={y}&check_in=2025-03-15&guests=2
     → Search hotels by location and dates

GET  /v1/hotels/{id}
     → Hotel details (rooms, amenities, photos)

POST /v1/reservations
     → Start a reservation (holds the room!)

PUT  /v1/reservations/{id}
     → Add payment details, finalize booking

GET  /v1/reservations/{id}
     → Check reservation status
```

**Notice the POST for reservations includes a `requestId` (idempotency key):**

```json
{
  "requestId": "uuid-unique-123",
  "hotelId": "hotel-456",
  "roomIds": ["room-789"],
  "checkInDate": "2025-03-15",
  "checkOutDate": "2025-03-18",
  "guests": 2
}
```

Why? Because if the network glitches and the app retries, you don't want to accidentally book TWO rooms. The server sees the same `requestId` and says *"Already processed this — here's the same response."*

---

### Q: What's a "reservation hold" and why does it matter?

**A:** This is the **best deep dive topic** for a hotel app. Here's the scenario:

1. You find a room you love 😍
2. You tap "Book Now"
3. The server **holds** that room for 15 minutes — nobody else can book it
4. You enter payment details
5. If you finish in time → room is yours! 🎉
6. If 15 minutes pass → hold released, room goes back to inventory 💨

**The hard part: how do you show an accurate countdown timer on the phone?**

You can't trust the device clock — users can change it. You can't rely on network round-trips — they add latency.

The clever solution: **server timestamp + device uptime**

1. Server responds with `expiresAt: "2025-03-15T14:30:00Z"` (UTC timestamp)
2. Client records the current server time and the device's **uptime** (time since boot — immune to clock changes)
3. Client calculates: `timeRemaining = expiresAt - serverTime`
4. Client uses **device uptime** to count down (not wall clock)

This works even if:
- User changes their phone clock ✅
- App goes to background and comes back ✅
- Network is slow ✅

On Android: `SystemClock.elapsedRealtime()`
On iOS: `ProcessInfo.processInfo.systemUptime`

**In React Native — implement with server time offset:**

```typescript
function useReservationCountdown(serverExpiresAt: string, serverResponseTimestamp: number) {
  // serverResponseTimestamp = Date.now() captured right when you received the API response
  // This gives you the offset between your local clock and the "server's clock" as echoed back
  const clockOffset = useRef(serverResponseTimestamp - Date.now());
  const [secondsLeft, setSecondsLeft] = useState(0);

  useEffect(() => {
    const expiresMs = new Date(serverExpiresAt).getTime();

    const interval = setInterval(() => {
      const adjustedNow = Date.now() + clockOffset.current;
      const remaining = Math.max(0, expiresMs - adjustedNow);
      setSecondsLeft(Math.floor(remaining / 1000));
      if (remaining <= 0) clearInterval(interval);
    }, 1000);

    return () => clearInterval(interval);
  }, [serverExpiresAt]);

  const minutes = Math.floor(secondsLeft / 60);
  const seconds = String(secondsLeft % 60).padStart(2, '0');
  return `${minutes}:${seconds}`;
}

// Usage in your component
function BookingTimer({ reservation }: { reservation: ReservationResponse }) {
  const countdown = useReservationCountdown(
    reservation.expiresAt,
    reservation._receivedAt // Set this when you get the API response: Date.now()
  );
  return <Text style={styles.timer}>{countdown}</Text>;
}
```

The `clockOffset` compensates for the user's device clock being wrong. If their phone is set 5 minutes into the future, a naive `Date.now()` countdown would expire 5 minutes early. The offset anchors the countdown to your server's time reference.

---

### Q: Autocomplete for hotel search — how does that work?

**A:** Nobody wants to type "Marriott Grand Hyatt Resort & Spa, Downtown Manhattan" in full. Autocomplete makes search feel magical.

The challenge on mobile: you can't query the server on every keystroke (too slow, too expensive).

**The solution: Hybrid approach**

1. **On app launch**: Download a "popular destinations" dataset (top cities, popular hotels) and store locally
2. **As user types**: Search the LOCAL dataset first (instant results!) with a 200-300ms debounce (wait for them to stop typing)
3. **When user selects a suggestion**: Fetch additional detailed data from the server and cache it locally for next time

**Storage:** Use **Full-Text Search (FTS)** with SQLite. It's designed for exactly this — fast prefix matching, typo tolerance, relevance ranking.

Think of it as having a mini Google in your phone, but just for hotel names and cities.

```
User types: "Par"
FTS returns: "Paris, France" | "Parker Hotel, NYC" | "Paradise Resort, Bali"
```

---

### Q: How does payment work in the app?

**A:** Two options for integrating payments:

| Approach | How | Pros | Cons |
|----------|-----|------|------|
| **Backend integration** | App sends card info to YOUR server → server talks to Stripe | Centralized, easier to maintain | Extra latency, you handle sensitive data |
| **Client-side integration** ✅ | App uses Stripe/PayPal SDK directly → gets a payment token → sends token to your server | Native UX, biometric auth, no card data on your server | More client complexity |

**Client-side wins** because:
- Users get Apple Pay / Google Pay with biometric auth (Face ID, fingerprint)
- Your server never sees raw card numbers (huge for security/compliance)
- The PSP (Payment Service Provider) handles PCI compliance so you don't have to

The payment flow:
```
1. User taps "Pay" → PSP SDK collects card/biometric
2. PSP returns a payment token
3. App sends token to your backend: PUT /v1/reservations/{id}
4. Your backend calls PSP to verify & capture payment
5. Backend confirms reservation → sends response to app
```

**Critical**: The backend does **authorization** first (reserves the money) and **capture** later (actually takes it). If the booking fails between those two steps, the money stays in the user's account.

**Stripe React Native SDK — what the actual implementation looks like:**

```tsx
import { useStripe } from '@stripe/stripe-react-native';

function PaymentScreen({ reservationId }: { reservationId: string }) {
  const { initPaymentSheet, presentPaymentSheet } = useStripe();

  async function handlePay() {
    // 1. Backend creates a PaymentIntent and returns the clientSecret
    const { clientSecret } = await api.createPaymentIntent(reservationId);

    // 2. Initialize with Apple Pay / Google Pay enabled
    await initPaymentSheet({
      merchantDisplayName: 'HotelApp',
      paymentIntentClientSecret: clientSecret,
      applePay: { merchantCountryCode: 'US' },
      googlePay: { merchantCountryCode: 'US', testEnv: __DEV__ },
      defaultBillingDetails: { name: user.fullName },
    });

    // 3. Show the native Stripe payment UI
    const { error } = await presentPaymentSheet();

    if (!error) {
      // 4. Backend confirms via Stripe webhook — no extra API call from client
      navigation.navigate('BookingConfirmed', { reservationId });
    } else if (error.code !== 'Canceled') {
      showToast('Payment failed: ' + error.message);
    }
  }

  return <Button onPress={handlePay} title="Confirm & Pay" />;
}
```

`presentPaymentSheet()` renders a native UI that handles Apple Pay / Google Pay (with Face ID / Touch ID), saved cards, and new card entry. Your app never touches raw card numbers — Stripe tokenizes everything on-device.

Mention in your interview: *"The client gets a clientSecret from our backend, not card data. Our backend calls Stripe to verify and capture. Even if someone intercepted the clientSecret, they can only charge this one transaction — not reuse it."*

---

### Q: Any common pitfalls to mention for the hotel app?

**A:** Three smart things to say in the interview:

1. **"What if the price changed while the user was filling in payment?"**
   - The backend should validate the price hasn't changed before processing. If it has → return a `PriceMismatch` error and show the user the updated price.

2. **"What about the payment screen crashing mid-flow?"**
   - The reservation hold is still active (it's server-side). When the user reopens the app, check `GET /v1/reservations/{id}` to see if the hold is still valid, and resume from where they left off.

3. **"How do you show reservations offline?"**
   - Cache confirmed `ReservationResponse` objects in the local database. Users can view their upcoming trips even without internet — critical when you're at the airport with no Wi-Fi.

---

*Next up: [04 — Google Drive, YouTube & Pagination Library →](./04_Google_Drive_YouTube_and_Pagination.md)*
