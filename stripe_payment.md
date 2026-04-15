# Drovi Driver Wallet — Flutter Integration Guide

> Complete implementation guide for the Drovi Driver (Rider) mobile app wallet system.
> Based on the actual Drovi backend source code and matching the architecture of the existing React Native apps.

---

## Architecture Overview

```
┌──────────────────────────────────────┐
│  FLUTTER APP                         │
│                                      │
│  1. GET /stripe/config → pk_...      │
│  2. Stripe.publishableKey = pk_...   │
│  3. CardField → user enters card     │
│  4. createPaymentMethod() → pm_xxx   │
│  5. POST /riders/wallet/deposit      │
│     { amount, stripePaymentMethodId } │
└──────────────┬───────────────────────┘
               │
               ▼
┌──────────────────────────────────────┐
│  DROVI BACKEND                       │
│                                      │
│  6. Creates PENDING deposit record   │
│  7. stripe.paymentIntents.create()   │
│     with pm_xxx (uses sk_ secretly)  │
│  8. Returns { success, status }      │
│                                      │
│  9. Stripe webhook arrives           │
│     → payment_intent.succeeded       │
│  10. completeDeposit()               │
│      → wallet.balance += amount      │
└──────────────────────────────────────┘
```

> [!IMPORTANT]
> The Flutter app NEVER needs the Stripe secret key (`sk_...`). It only uses the publishable key (`pk_...`) fetched from `GET /stripe/config`. This key can only tokenize cards — it cannot charge money.

---

## All Wallet API Endpoints

All endpoints require `Authorization: Bearer <token>` header (rider JWT).
Base URL: `https://api.drovi.com/api/v1`

### Stripe Config (Public)

| Method | Endpoint | Auth | Response |
|:---|:---|:---|:---|
| `GET` | `/stripe/config` | ❌ None | `{ publishableKey: "pk_...", configured: true }` |

### Wallet Balance & Deposits

| Method | Endpoint | Auth | Payload / Response |
|:---|:---|:---|:---|
| `GET` | `/riders/wallet` | ✅ Rider | Returns `{ success, data: { id, balance, isFrozen } }` |
| `POST` | `/riders/wallet/deposit` | ✅ Rider | Send `{ amount: 25.0, stripePaymentMethodId: "pm_xxx" }` |
| `GET` | `/riders/wallet/deposits` | ✅ Rider | Returns deposit history array |

### Transactions

| Method | Endpoint | Auth | Query Params |
|:---|:---|:---|:---|
| `GET` | `/riders/wallet/transactions` | ✅ Rider | `?page=1&limit=20&type=CREDIT` or `DEBIT` |

### Withdrawals

| Method | Endpoint | Auth | Payload |
|:---|:---|:---|:---|
| `POST` | `/riders/wallet/withdraw` | ✅ Rider | `{ amount: 50.0 }` (uses saved bank details) |
| `GET` | `/riders/wallet/withdrawals` | ✅ Rider | `?page=1&limit=10` |

### Direct Deposit (Bank Account)

| Method | Endpoint | Auth | Payload |
|:---|:---|:---|:---|
| `PUT` | `/riders/wallet/direct-deposit` | ✅ Rider | `{ bankName, routingNumber, accountNumber, accountType, accountHolderName }` |
| `GET` | `/riders/wallet/direct-deposit` | ✅ Rider | Returns saved bank details |

---

## Deposit Payload — Exact DTO

```json
{
  "amount": 25.00,
  "stripePaymentMethodId": "pm_1Pxxxxxxxxxxxxxxx"
}
```

> [!WARNING]
> - `amount` must be ≥ 1 (validated server-side)
> - `stripePaymentMethodId` must start with `pm_` — this is the token from `createPaymentMethod()`
> - The backend also accepts `squarePaymentToken` (legacy) but **use Stripe only**

---

## Deposit Status Flow

```
PENDING → COMPLETED    (webhook: payment_intent.succeeded)
PENDING → FAILED       (webhook: payment_intent.payment_failed)
```

The wallet balance is **only** updated when the backend receives the Stripe webhook (`payment_intent.succeeded`). The POST response returns `PENDING` — not `COMPLETED`.

---

## Flutter Implementation

### 1. Dependencies

```yaml
# pubspec.yaml
dependencies:
  flutter_stripe: ^11.3.0
  http: ^1.2.0
```

### 2. Initialize Stripe (`main.dart`)

```dart
import 'package:flutter_stripe/flutter_stripe.dart';
import 'dart:convert';
import 'package:http/http.dart' as http;

const String baseUrl = 'https://api.drovi.com/api/v1';

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();

  // Fetch publishable key from backend — NO secret key needed
  final res = await http.get(Uri.parse('$baseUrl/stripe/config'));
  final config = jsonDecode(res.body);

  if (config['configured'] == true) {
    Stripe.publishableKey = config['publishableKey']; // "pk_..."
    await Stripe.instance.applySettings();
  }

  runApp(const MyApp());
}
```

### 3. Top-Up Flow (Core Payment Logic)

```dart
Future<void> topUpWallet(double amount, String authToken) async {
  // STEP 1: Tokenize card details → pm_xxx
  // Card numbers go directly to Stripe servers, never to your backend
  final paymentMethod = await Stripe.instance.createPaymentMethod(
    params: const PaymentMethodParams.card(
      paymentMethodData: PaymentMethodData(),
    ),
  );
  // paymentMethod.id = "pm_1Pxxxxxxxxxxxxxxx"

  // STEP 2: Send token to Drovi backend
  final res = await http.post(
    Uri.parse('$baseUrl/riders/wallet/deposit'),
    headers: {
      'Content-Type': 'application/json',
      'Authorization': 'Bearer $authToken',
    },
    body: jsonEncode({
      'amount': amount,
      'stripePaymentMethodId': paymentMethod.id,
    }),
  );

  final result = jsonDecode(res.body);
  // result = { success: true, message: "Deposit processed", data: { deposit: { status: "PENDING" } } }

  // STEP 3: Wait for webhook to process, then refresh balance
  await Future.delayed(const Duration(seconds: 3));
  // Re-fetch wallet balance from GET /riders/wallet
}
```

### 4. Complete Wallet Screen

```dart
class WalletScreen extends StatefulWidget {
  final String authToken;
  const WalletScreen({super.key, required this.authToken});
  @override State<WalletScreen> createState() => _WalletScreenState();
}

class _WalletScreenState extends State<WalletScreen> {
  final _amountCtrl = TextEditingController();
  double _balance = 0;
  bool _cardComplete = false;
  bool _submitting = false;
  List<dynamic> _deposits = [];

  Map<String, String> get _headers => {
    'Content-Type': 'application/json',
    'Authorization': 'Bearer ${widget.authToken}',
  };

  @override
  void initState() {
    super.initState();
    _loadWallet();
    _loadDeposits();
  }

  Future<void> _loadWallet() async {
    final res = await http.get(
      Uri.parse('$baseUrl/riders/wallet'),
      headers: _headers,
    );
    final data = jsonDecode(res.body)['data'];
    setState(() => _balance = (data['balance'] ?? 0).toDouble());
  }

  Future<void> _loadDeposits() async {
    final res = await http.get(
      Uri.parse('$baseUrl/riders/wallet/deposits'),
      headers: _headers,
    );
    setState(() => _deposits = jsonDecode(res.body)['data'] ?? []);
  }

  Future<void> _handleTopUp() async {
    final amount = double.tryParse(_amountCtrl.text);
    if (amount == null || amount < 1) return;

    // Confirmation dialog
    final ok = await showDialog<bool>(
      context: context,
      builder: (ctx) => AlertDialog(
        title: const Text('Confirm Payment'),
        content: Text('You will be charged \$${amount.toStringAsFixed(2)}.'),
        actions: [
          TextButton(onPressed: () => Navigator.pop(ctx, false), child: const Text('Cancel')),
          ElevatedButton(onPressed: () => Navigator.pop(ctx, true), child: const Text('Charge')),
        ],
      ),
    );
    if (ok != true) return;

    setState(() => _submitting = true);
    try {
      // Tokenize
      final pm = await Stripe.instance.createPaymentMethod(
        params: const PaymentMethodParams.card(paymentMethodData: PaymentMethodData()),
      );

      // Submit
      await http.post(
        Uri.parse('$baseUrl/riders/wallet/deposit'),
        headers: _headers,
        body: jsonEncode({ 'amount': amount, 'stripePaymentMethodId': pm.id }),
      );

      // Wait for webhook, then refresh
      await Future.delayed(const Duration(seconds: 3));
      await _loadWallet();
      await _loadDeposits();
      _amountCtrl.clear();
    } catch (e) {
      ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text('Error: $e')));
    } finally {
      setState(() => _submitting = false);
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Wallet')),
      body: ListView(
        padding: const EdgeInsets.all(20),
        children: [
          // ── Balance Card ──
          Container(
            padding: const EdgeInsets.all(24),
            decoration: BoxDecoration(
              color: const Color(0xFFFFBF1F),
              borderRadius: BorderRadius.circular(16),
            ),
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                const Text('Available Balance', style: TextStyle(fontSize: 14)),
                const SizedBox(height: 8),
                Text('\$${_balance.toStringAsFixed(2)}',
                    style: const TextStyle(fontSize: 32, fontWeight: FontWeight.bold)),
              ],
            ),
          ),
          const SizedBox(height: 24),

          // ── Amount ──
          TextField(
            controller: _amountCtrl,
            keyboardType: const TextInputType.numberWithOptions(decimal: true),
            decoration: const InputDecoration(
              labelText: 'Amount (\$)',
              prefixText: '\$ ',
              border: OutlineInputBorder(),
            ),
          ),
          const SizedBox(height: 16),

          // ── Stripe Card Field ──
          const Text('Card Details', style: TextStyle(fontWeight: FontWeight.w600)),
          const SizedBox(height: 8),
          CardField(
            enablePostalCode: false,
            onCardChanged: (card) {
              setState(() => _cardComplete = card?.complete ?? false);
            },
          ),
          const SizedBox(height: 8),
          Text('Balance updates after payment confirmation.',
              style: TextStyle(fontSize: 12, color: Colors.grey[500])),
          const SizedBox(height: 20),

          // ── Submit Button ──
          SizedBox(
            width: double.infinity,
            height: 50,
            child: ElevatedButton(
              onPressed: (_submitting || !_cardComplete) ? null : _handleTopUp,
              style: ElevatedButton.styleFrom(
                backgroundColor: const Color(0xFFFFBF1F),
                foregroundColor: Colors.black,
              ),
              child: _submitting
                  ? const CircularProgressIndicator()
                  : const Text('Add Money', style: TextStyle(fontSize: 16)),
            ),
          ),
          const SizedBox(height: 32),

          // ── Deposit History ──
          const Text('Recent Deposits', style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold)),
          const SizedBox(height: 12),
          ..._deposits.take(10).map((d) => ListTile(
                title: Text('\$${d['amount']}'),
                subtitle: Text(d['status']),
                trailing: Text(d['requestedAt']?.toString().substring(0, 10) ?? ''),
              )),
        ],
      ),
    );
  }
}
```

---

## Android Setup Requirements

1. Use `FlutterFragmentActivity` in `MainActivity.kt`:
```kotlin
import io.flutter.embedding.android.FlutterFragmentActivity

class MainActivity: FlutterFragmentActivity()
```

2. Use `Theme.AppCompat` in `styles.xml`

3. Add to `proguard-rules.pro`:
```
-dontwarn com.stripe.android.pushProvisioning.**
-keep class com.stripe.** { *; }
```

4. Min SDK 21+

## iOS Setup Requirements

1. Set `platform :ios, '13.0'` in Podfile
2. Min deployment target iOS 13.0

---

## Testing

Use Stripe test cards (with any CVC, future expiry, any postal code):

| Card Number | Scenario |
|:---|:---|
| `4242 4242 4242 4242` | ✅ Successful payment |
| `4000 0000 0000 9995` | ❌ Declined (insufficient funds) |
| `4000 0000 0000 3220` | ⚠️ Requires 3D Secure |

---

## Key Rules

> [!CAUTION]
> 1. **NEVER** embed `sk_...` in the Flutter app
> 2. **ALWAYS** show a confirmation dialog before charging
> 3. **ALWAYS** wait 2-3 seconds before refreshing balance (webhook delay)
> 4. **NEVER** assume payment succeeded from the POST response — it returns `PENDING`
> 5. The `stripePaymentMethodId` field is the ONLY payment field you need — ignore `squarePaymentToken`
