<!--
  CHAPTER: 24
  TITLE: Payments & Money Handling — Stripe, IAP & Revenue
  PART: V — Deployment & Operations
  PREREQS: Chapters 5, 6, 22
  KEY_TOPICS: Stripe, React Native Stripe SDK, Stripe Elements, Payment Intents, Subscriptions, Apple Pay, Google Pay, In-App Purchases, RevenueCat, webhooks, PCI compliance, idempotency, refunds, testing
  DIFFICULTY: Intermediate → Advanced
  UPDATED: 2026-04-07
-->

# Chapter 24: Payments & Money Handling — Stripe, IAP & Revenue

> **Part V — Deployment & Operations** | Prerequisites: Chapters 5, 6, 22 | Difficulty: Intermediate → Advanced

> "Move fast and break things" does not apply to payment code. Move carefully and break nothing.

---

## The Money Mindset

Let me set the tone for this chapter with something that should make you slightly uncomfortable.

Every other chapter in this book deals with code where a bug is annoying. A broken animation? Users grumble. A messed-up navigation stack? They force-quit and reopen. A slow API call? They wait an extra second. These are bad, but they are recoverable. Users forgive them, usually without even contacting support.

Payment bugs are different. Payment bugs cost real money. Not your company's money -- your users' money. And users do not forgive that.

Here are things that have happened to real teams who treated payment code like any other feature:

- **Double-charging**: A retry loop on a flaky network caused the same PaymentIntent to be confirmed twice. 2,400 customers were charged double. The support team worked for three weeks processing refunds. The app's rating dropped from 4.7 to 3.9 on the App Store.
- **Missed charges**: A race condition between the client and webhook handler meant some orders were marked as "paid" before the payment actually succeeded. The company shipped $43,000 worth of product to customers whose payments later failed.
- **Broken refunds**: A refund endpoint was deployed with a bug that returned a 200 status but never actually called the Stripe refund API. Customer support told users their refunds were "processing" for weeks.
- **Leaked API keys**: A developer committed the Stripe secret key to a public GitHub repo. Someone found it within hours and issued $12,000 in fraudulent refunds to their own account.

This chapter treats payment integration with the seriousness it deserves. Every line of code we write here, every architectural decision we make, every warning I give you -- it all comes from the principle that payment code is the most consequential code you will write.

Here is the mindset shift I need you to make:

| Normal Feature Code | Payment Code |
|---------------------|-------------|
| Optimistic updates are fine | Pessimistic updates only -- wait for confirmation |
| Retry aggressively | Retry with idempotency keys or do not retry at all |
| Log everything for debugging | Never log card numbers, CVVs, or full tokens |
| Move fast, fix bugs later | Get it right the first time, test exhaustively |
| Client-driven state | Server-driven state, verified by webhooks |
| "It works on my machine" | "It works with test card 4242, declined cards, 3D Secure, network timeouts, and webhook replays" |

If you are not a little nervous about writing payment code, you are not thinking hard enough about what can go wrong. That nervousness is good. It will make you careful. Let us channel it into building something bulletproof.

### In This Chapter
- Stripe architecture -- PaymentIntents, the client-server split, API keys, webhooks
- React Native + Stripe -- @stripe/stripe-react-native, Payment Sheet, CardField, Apple Pay, Google Pay
- Next.js + Stripe (web) -- Stripe Elements, Checkout, Server Actions, webhook Route Handlers
- Subscriptions -- Stripe Billing, lifecycle management, dunning, grace periods
- In-App Purchases -- Apple IAP, Google Play Billing, RevenueCat, the 30% commission
- Apple Pay and Google Pay -- wallet payments, merchant IDs, tokenization
- Webhook architecture -- signature verification, idempotency, event handling
- PCI compliance -- what Stripe handles, what you handle, what to never do
- Testing payments -- test cards, Stripe CLI, edge cases, CI integration
- Refunds and disputes -- programmatic refunds, chargebacks, evidence submission
- Common mistakes -- and how to avoid every single one of them

### Related Chapters
- [Ch 5: Expo & React Native Setup] -- the mobile app foundation that payment SDKs integrate with
- [Ch 6: Next.js Deep Dive] -- the web framework and API routes that power server-side payment logic
- [Ch 22: Security & Data Protection] -- securing API keys, HTTPS, and sensitive data handling

---

## 1. STRIPE ARCHITECTURE

### 1.1 Why Stripe

There are many payment processors. Stripe is the one you should use. Here is why:

1. **Developer experience**: Stripe's API is the gold standard. The documentation is exceptional. The SDKs are well-maintained. The Dashboard is genuinely useful.
2. **React Native SDK**: Stripe has an official, first-party React Native SDK (`@stripe/stripe-react-native`). Not a community wrapper around a web SDK -- an actual native SDK with native payment sheets, Apple Pay, Google Pay, and 3D Secure built in.
3. **Next.js integration**: Stripe works seamlessly with Next.js API routes, Server Actions, and Route Handlers. Vercel even has official Stripe integration templates.
4. **PCI compliance**: Stripe handles the hard parts of PCI compliance. You never touch raw card data. This alone saves you months of security auditing.
5. **Global coverage**: 135+ currencies, 46+ countries, dozens of payment methods beyond cards.

I am not going to pretend to be balanced here. If you are building a startup or growth-stage product with React Native and Next.js, use Stripe. The alternatives (Braintree, Adyen, Square) are fine companies, but none of them match Stripe's developer experience for our stack.

### 1.2 How Stripe Works: The Big Picture

Here is the fundamental architecture you need to understand:

```
┌─────────────────────────────────────────────────────────┐
│                    YOUR APPLICATION                       │
│                                                          │
│  ┌──────────────┐          ┌──────────────────────────┐  │
│  │   CLIENT     │          │        SERVER             │  │
│  │              │          │                           │  │
│  │  React Native│  ──(1)── │  Next.js API / Server     │  │
│  │  or Web App  │          │  Actions                  │  │
│  │              │ ◄──(2)── │                           │  │
│  │  Confirms    │          │  Creates PaymentIntent    │  │
│  │  payment     │          │  with secret key          │  │
│  │  with SDK    │          │                           │  │
│  │              │          │  ◄──(4)── Stripe Webhook  │  │
│  └──────┬───────┘          └──────────────────────────┘  │
│         │                                                 │
│         │ (3) Sends card data directly to Stripe          │
│         │     (never touches your server)                 │
└─────────┼─────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────┐
│       STRIPE        │
│                     │
│  Processes payment  │
│  Sends webhook      │
│  Returns result     │
└─────────────────────┘
```

The critical insight: **card data never touches your server.** The client sends card data directly to Stripe using Stripe's SDK. Your server only deals with PaymentIntents (which are references to payments, not actual card data).

This is not optional. This is how Stripe works. This is how PCI compliance works. If you are building a system where card numbers pass through your server, you are doing it wrong and you are taking on enormous legal and financial liability.

### 1.3 PaymentIntents: The Core Concept

A PaymentIntent represents a single payment attempt. It is the central object in Stripe's payment flow. Here is its lifecycle:

```
Created (server)
    │
    ▼
requires_payment_method
    │
    ▼ (client provides card / wallet)
requires_confirmation
    │
    ▼ (client confirms)
requires_action (maybe -- 3D Secure, etc.)
    │
    ▼
processing
    │
    ▼
succeeded  ───or───  requires_payment_method (failed, try again)
                         │
                         ▼
                      canceled (gave up)
```

**The flow:**

1. Your **server** creates a PaymentIntent with the amount and currency. This returns a `client_secret`.
2. Your **client** uses the `client_secret` to confirm the payment with the user's card data.
3. **Stripe** processes the payment and sends a webhook to your server.
4. Your **server** handles the webhook and updates your database.

The `client_secret` is the key that lets the client-side SDK interact with a specific PaymentIntent. It is safe to send to the client -- it can only be used to confirm that specific payment, not to create new ones or read sensitive data.

### 1.4 API Keys: The Rules

Stripe gives you two pairs of API keys (test and live):

| Key | Prefix | Where It Lives | Purpose |
|-----|--------|----------------|---------|
| Publishable key | `pk_test_` / `pk_live_` | Client (React Native, browser) | Identifies your account, used by Stripe SDK |
| Secret key | `sk_test_` / `sk_live_` | Server ONLY | Creates charges, refunds, customers -- full API access |

**Rules that are not negotiable:**

1. The **secret key** never, ever, ever goes in client code. Not in React Native. Not in a web bundle. Not in an environment variable that gets bundled. Not "just for testing." Never.
2. The **publishable key** is safe for client code. That is literally what it is for.
3. Store secret keys in environment variables on your server. Not in your codebase. Not in a config file that gets committed.
4. Use test keys (`pk_test_`, `sk_test_`) for development. Use live keys (`pk_live_`, `sk_live_`) for production. Never test with live keys.

```bash
# .env.local (server-side, never committed to git)
STRIPE_SECRET_KEY=sk_test_51ABC123...
STRIPE_PUBLISHABLE_KEY=pk_test_51ABC123...
STRIPE_WEBHOOK_SECRET=whsec_abc123...

# .gitignore -- this should already be there, but double-check
.env.local
.env*.local
```

### 1.5 The Stripe Dashboard

The Stripe Dashboard is not just an admin panel. It is your payment operations center. You should have it bookmarked and check it regularly.

Key things you will use:

- **Payments**: See every payment, its status, timeline of events, metadata
- **Customers**: View customer objects, their payment methods, subscription history
- **Webhooks**: See webhook delivery attempts, failures, and retry schedules
- **Logs**: Every API call your server makes, with request/response bodies
- **Test mode toggle**: Switch between test and live data (top-right corner)
- **Developers > Events**: The firehose of everything happening in your account

**Pro tip**: When debugging a payment issue, the Stripe Dashboard's payment detail page shows you the complete timeline -- when the PaymentIntent was created, when the customer provided their card, when 3D Secure was triggered, when the charge succeeded or failed, and every webhook that was sent. This timeline is often more useful than your own logs.

### 1.6 Test Mode vs Live Mode

Stripe has completely separate test and live environments. They share nothing -- different API keys, different data, different webhooks. This is great for safety but means you need to set up everything twice.

```typescript
// Determine which key to use based on environment
const stripePublishableKey = process.env.NODE_ENV === 'production'
  ? process.env.STRIPE_LIVE_PUBLISHABLE_KEY!
  : process.env.STRIPE_TEST_PUBLISHABLE_KEY!;
```

In test mode:
- No real money moves
- Use test card numbers (we will cover these later)
- Webhooks are sent but marked as test events
- You can trigger any event manually from the Dashboard

---

## 2. REACT NATIVE + STRIPE

### 2.1 SDK Setup with Expo

The official Stripe React Native SDK is `@stripe/stripe-react-native`. It provides native payment UIs, Apple Pay, Google Pay, and handles PCI compliance by ensuring card data never touches your JavaScript code.

```bash
# Install the SDK
npx expo install @stripe/stripe-react-native
```

The Expo config plugin handles native configuration automatically:

```typescript
// app.config.ts
import { ExpoConfig } from 'expo/config';

const config: ExpoConfig = {
  name: 'MyApp',
  slug: 'my-app',
  // ...

  plugins: [
    [
      '@stripe/stripe-react-native',
      {
        // For Apple Pay (required for iOS wallet payments)
        merchantIdentifier: 'merchant.com.mycompany.myapp',
        // Enable Apple Pay in your entitlements
        enableGooglePay: true,
      },
    ],
  ],
};

export default config;
```

**Important**: After adding the plugin, you need to run `npx expo prebuild` to regenerate native code, or if you are using EAS Build, it happens automatically.

Initialize the Stripe provider at the root of your app:

```typescript
// src/providers/StripeProvider.tsx
import { StripeProvider as NativeStripeProvider } from '@stripe/stripe-react-native';
import { ReactNode } from 'react';

const PUBLISHABLE_KEY = process.env.EXPO_PUBLIC_STRIPE_PUBLISHABLE_KEY!;

// Validate at startup -- fail loudly, not silently
if (!PUBLISHABLE_KEY) {
  throw new Error(
    'Missing EXPO_PUBLIC_STRIPE_PUBLISHABLE_KEY. ' +
    'Add it to your .env file. See Chapter 24 for details.'
  );
}

interface Props {
  children: ReactNode;
}

export function StripeProvider({ children }: Props) {
  return (
    <NativeStripeProvider
      publishableKey={PUBLISHABLE_KEY}
      merchantIdentifier="merchant.com.mycompany.myapp" // For Apple Pay
      urlScheme="myapp" // For 3D Secure return URLs
    >
      {children}
    </NativeStripeProvider>
  );
}
```

```typescript
// App.tsx (or your root layout)
import { StripeProvider } from '@/providers/StripeProvider';

export default function App() {
  return (
    <StripeProvider>
      {/* Rest of your app */}
    </StripeProvider>
  );
}
```

### 2.2 Payment Sheet: The Recommended Approach

The Payment Sheet is Stripe's pre-built payment UI. It handles card input, validation, Apple Pay, Google Pay, saved payment methods, and 3D Secure -- all in a native bottom sheet that looks and feels like a first-party iOS/Android payment experience.

**Use the Payment Sheet for 90% of payment flows.** It is faster to implement, harder to mess up, automatically PCI compliant, and looks better than anything you would build yourself.

Here is the complete flow:

```typescript
// src/hooks/usePaymentSheet.ts
import { useState, useCallback } from 'react';
import { useStripe } from '@stripe/stripe-react-native';
import { Alert } from 'react-native';

interface PaymentSheetParams {
  amount: number; // in cents (e.g., 2999 = $29.99)
  currency?: string;
}

export function usePaymentSheet() {
  const { initPaymentSheet, presentPaymentSheet } = useStripe();
  const [loading, setLoading] = useState(false);

  const pay = useCallback(async ({ amount, currency = 'usd' }: PaymentSheetParams) => {
    setLoading(true);

    try {
      // Step 1: Create PaymentIntent on your server
      const response = await fetch('/api/payments/create-intent', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ amount, currency }),
      });

      if (!response.ok) {
        throw new Error('Failed to create payment intent');
      }

      const {
        clientSecret,
        ephemeralKey,
        customerId,
      } = await response.json();

      // Step 2: Initialize the Payment Sheet
      const { error: initError } = await initPaymentSheet({
        paymentIntentClientSecret: clientSecret,
        customerEphemeralKeySecret: ephemeralKey,
        customerId: customerId,
        merchantDisplayName: 'My Company',
        // Apple Pay / Google Pay
        applePay: {
          merchantCountryCode: 'US',
        },
        googlePay: {
          merchantCountryCode: 'US',
          testEnv: __DEV__,
        },
        // Appearance customization
        appearance: {
          colors: {
            primary: '#0066FF',
            background: '#FFFFFF',
            componentBackground: '#F5F5F5',
          },
          shapes: {
            borderRadius: 12,
          },
        },
        // Allow saving payment methods for future use
        allowsDelayedPaymentMethods: false,
        returnURL: 'myapp://payment-return',
      });

      if (initError) {
        throw new Error(initError.message);
      }

      // Step 3: Present the Payment Sheet to the user
      const { error: paymentError } = await presentPaymentSheet();

      if (paymentError) {
        if (paymentError.code === 'Canceled') {
          // User dismissed the sheet -- not an error
          return { success: false, canceled: true };
        }
        throw new Error(paymentError.message);
      }

      // Step 4: Payment succeeded!
      // But DO NOT trust this alone. Wait for the webhook on your server.
      return { success: true, canceled: false };

    } catch (error) {
      Alert.alert(
        'Payment Failed',
        error instanceof Error ? error.message : 'An unexpected error occurred'
      );
      return { success: false, canceled: false };
    } finally {
      setLoading(false);
    }
  }, [initPaymentSheet, presentPaymentSheet]);

  return { pay, loading };
}
```

Using it in a screen:

```typescript
// src/screens/CheckoutScreen.tsx
import { View, Text, Pressable, ActivityIndicator, StyleSheet } from 'react-native';
import { usePaymentSheet } from '@/hooks/usePaymentSheet';
import { useRouter } from 'expo-router';

interface CheckoutScreenProps {
  totalAmount: number; // in cents
}

export function CheckoutScreen({ totalAmount }: CheckoutScreenProps) {
  const { pay, loading } = usePaymentSheet();
  const router = useRouter();

  const handleCheckout = async () => {
    const result = await pay({ amount: totalAmount });

    if (result.success) {
      // Navigate to success screen
      // The server will confirm via webhook and update the order
      router.replace('/order-confirmation');
    }
    // If canceled or failed, stay on this screen
  };

  return (
    <View style={styles.container}>
      <View style={styles.summary}>
        <Text style={styles.label}>Total</Text>
        <Text style={styles.amount}>
          ${(totalAmount / 100).toFixed(2)}
        </Text>
      </View>

      <Pressable
        style={[styles.button, loading && styles.buttonDisabled]}
        onPress={handleCheckout}
        disabled={loading}
      >
        {loading ? (
          <ActivityIndicator color="#FFFFFF" />
        ) : (
          <Text style={styles.buttonText}>Pay Now</Text>
        )}
      </Pressable>
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    padding: 24,
    justifyContent: 'flex-end',
  },
  summary: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    marginBottom: 24,
  },
  label: {
    fontSize: 18,
    color: '#666',
  },
  amount: {
    fontSize: 32,
    fontWeight: '700',
    color: '#000',
  },
  button: {
    backgroundColor: '#0066FF',
    paddingVertical: 16,
    borderRadius: 12,
    alignItems: 'center',
  },
  buttonDisabled: {
    opacity: 0.6,
  },
  buttonText: {
    color: '#FFFFFF',
    fontSize: 18,
    fontWeight: '600',
  },
});
```

### 2.3 Custom Payment Forms with CardField

Sometimes the Payment Sheet does not fit your design requirements. Maybe you need the card input embedded directly in your checkout screen, not in a bottom sheet. For this, use `CardField`:

```typescript
// src/components/CustomPaymentForm.tsx
import { useState } from 'react';
import { View, Text, Pressable, StyleSheet, Alert } from 'react-native';
import {
  CardField,
  useStripe,
  CardFieldInput,
} from '@stripe/stripe-react-native';

interface CustomPaymentFormProps {
  amount: number;
  onSuccess: () => void;
}

export function CustomPaymentForm({ amount, onSuccess }: CustomPaymentFormProps) {
  const { confirmPayment } = useStripe();
  const [cardDetails, setCardDetails] = useState<CardFieldInput.Details | null>(null);
  const [loading, setLoading] = useState(false);

  const handlePay = async () => {
    if (!cardDetails?.complete) {
      Alert.alert('Error', 'Please enter complete card details');
      return;
    }

    setLoading(true);

    try {
      // Step 1: Create PaymentIntent on server
      const response = await fetch('/api/payments/create-intent', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ amount }),
      });

      const { clientSecret } = await response.json();

      // Step 2: Confirm the payment with card details
      // The SDK sends card data directly to Stripe -- it never hits your server
      const { error, paymentIntent } = await confirmPayment(clientSecret, {
        paymentMethodType: 'Card',
        paymentMethodData: {
          billingDetails: {
            email: 'customer@example.com',
            // Add more billing details as needed
          },
        },
      });

      if (error) {
        Alert.alert('Payment Failed', error.message);
      } else if (paymentIntent?.status === 'Succeeded') {
        onSuccess();
      }
    } catch (err) {
      Alert.alert('Error', 'Something went wrong. Please try again.');
    } finally {
      setLoading(false);
    }
  };

  return (
    <View style={styles.container}>
      <Text style={styles.label}>Card Details</Text>
      <CardField
        postalCodeEnabled={true}
        placeholders={{
          number: '4242 4242 4242 4242',
        }}
        cardStyle={{
          backgroundColor: '#FFFFFF',
          textColor: '#000000',
          borderColor: '#E0E0E0',
          borderWidth: 1,
          borderRadius: 8,
          fontSize: 16,
          placeholderColor: '#999999',
        }}
        style={styles.cardField}
        onCardChange={(details) => setCardDetails(details)}
      />

      <Pressable
        style={[
          styles.payButton,
          (!cardDetails?.complete || loading) && styles.payButtonDisabled,
        ]}
        onPress={handlePay}
        disabled={!cardDetails?.complete || loading}
      >
        <Text style={styles.payButtonText}>
          {loading ? 'Processing...' : `Pay $${(amount / 100).toFixed(2)}`}
        </Text>
      </Pressable>
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    padding: 16,
  },
  label: {
    fontSize: 16,
    fontWeight: '600',
    marginBottom: 8,
    color: '#333',
  },
  cardField: {
    width: '100%',
    height: 50,
    marginBottom: 24,
  },
  payButton: {
    backgroundColor: '#0066FF',
    paddingVertical: 16,
    borderRadius: 12,
    alignItems: 'center',
  },
  payButtonDisabled: {
    opacity: 0.5,
  },
  payButtonText: {
    color: '#FFFFFF',
    fontSize: 18,
    fontWeight: '600',
  },
});
```

**When to use CardField vs Payment Sheet:**

| Use Payment Sheet when... | Use CardField when... |
|--------------------------|----------------------|
| You want the fastest integration | You need the card form embedded in your screen |
| You want Apple Pay / Google Pay included | You have very specific design requirements |
| You want Stripe to handle UX edge cases | You are building a checkout flow with multiple steps |
| You want saved payment methods | You need full control over the form layout |
| You are starting a new project | You are integrating into an existing complex screen |

My recommendation: start with Payment Sheet. Switch to CardField only if you have a concrete design reason. The Payment Sheet handles dozens of edge cases (keyboard management, validation animations, error states, accessibility) that you would have to build yourself with CardField.

### 2.4 The Complete PaymentIntent Flow

Let me show you the full flow from server to client, end to end. This is the single most important code in this chapter.

**Server side (Next.js API Route):**

```typescript
// app/api/payments/create-intent/route.ts
import { NextRequest, NextResponse } from 'next/server';
import Stripe from 'stripe';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
  apiVersion: '2025-03-31.basil', // Always pin the API version
});

export async function POST(request: NextRequest) {
  try {
    const body = await request.json();
    const { amount, currency = 'usd', metadata = {} } = body;

    // Validate the amount on the server -- NEVER trust client-provided amounts
    // In a real app, calculate the amount from the cart/order on the server
    if (!amount || amount < 50) { // Stripe minimum is $0.50 USD
      return NextResponse.json(
        { error: 'Invalid amount' },
        { status: 400 }
      );
    }

    // Get or create a Stripe customer
    // In production, look up the customer from your database
    const customerId = await getOrCreateStripeCustomer(request);

    // Create an ephemeral key for the customer (needed for Payment Sheet)
    const ephemeralKey = await stripe.ephemeralKeys.create(
      { customer: customerId },
      { apiVersion: '2025-03-31.basil' }
    );

    // Create the PaymentIntent
    const paymentIntent = await stripe.paymentIntents.create({
      amount,
      currency,
      customer: customerId,
      // Attach metadata for tracking
      metadata: {
        ...metadata,
        orderId: metadata.orderId || generateOrderId(),
        source: 'mobile_app',
      },
      // Enable automatic payment methods (cards, wallets, etc.)
      automatic_payment_methods: {
        enabled: true,
      },
    });

    return NextResponse.json({
      clientSecret: paymentIntent.client_secret,
      ephemeralKey: ephemeralKey.secret,
      customerId,
      paymentIntentId: paymentIntent.id,
    });

  } catch (error) {
    console.error('Error creating PaymentIntent:', error);

    if (error instanceof Stripe.errors.StripeError) {
      return NextResponse.json(
        { error: error.message },
        { status: error.statusCode || 500 }
      );
    }

    return NextResponse.json(
      { error: 'Internal server error' },
      { status: 500 }
    );
  }
}

async function getOrCreateStripeCustomer(request: NextRequest): Promise<string> {
  // In production, you would:
  // 1. Get the authenticated user from the session/JWT
  // 2. Look up their Stripe customer ID in your database
  // 3. If they don't have one, create a new Stripe customer and save the ID

  // Simplified example:
  const userId = 'user_123'; // Get from auth middleware

  // Check your database for existing Stripe customer ID
  // const user = await db.user.findUnique({ where: { id: userId } });
  // if (user.stripeCustomerId) return user.stripeCustomerId;

  const customer = await stripe.customers.create({
    metadata: { userId },
  });

  // Save to your database:
  // await db.user.update({ where: { id: userId }, data: { stripeCustomerId: customer.id } });

  return customer.id;
}

function generateOrderId(): string {
  return `order_${Date.now()}_${Math.random().toString(36).substring(2, 8)}`;
}
```

**A critical note about amounts**: Never trust the amount the client sends you. In a real application, your server should calculate the amount from the user's cart or order. The client tells the server "I want to pay for order XYZ," and the server looks up order XYZ and knows the amount. If you let the client tell the server how much to charge, someone will inevitably modify the request and pay $0.50 for a $500 item.

### 2.5 Apple Pay Integration (React Native)

Apple Pay is already supported by the Payment Sheet. But if you want to offer Apple Pay as a standalone option (e.g., an "Apple Pay" button at the top of your checkout), here is how:

```typescript
// src/components/ApplePayButton.tsx
import { useCallback, useEffect, useState } from 'react';
import { View, StyleSheet, Platform } from 'react-native';
import {
  useStripe,
  isPlatformPaySupported,
  PlatformPay,
  PlatformPayButton,
} from '@stripe/stripe-react-native';

interface ApplePayButtonProps {
  amount: number; // in cents
  onSuccess: () => void;
  onError: (error: string) => void;
}

export function ApplePayButton({ amount, onSuccess, onError }: ApplePayButtonProps) {
  const { confirmPlatformPayPayment } = useStripe();
  const [isSupported, setIsSupported] = useState(false);

  useEffect(() => {
    (async () => {
      const supported = await isPlatformPaySupported();
      setIsSupported(supported);
    })();
  }, []);

  const handleApplePay = useCallback(async () => {
    try {
      // Step 1: Create PaymentIntent on server
      const response = await fetch('/api/payments/create-intent', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ amount }),
      });

      const { clientSecret } = await response.json();

      // Step 2: Confirm with Apple Pay / Google Pay
      const { error } = await confirmPlatformPayPayment(clientSecret, {
        applePay: {
          cartItems: [
            {
              label: 'Total',
              amount: (amount / 100).toFixed(2),
              paymentType: PlatformPay.PaymentType.Immediate,
            },
          ],
          merchantCountryCode: 'US',
          currencyCode: 'USD',
        },
        googlePay: {
          testEnv: __DEV__,
          merchantName: 'My Company',
          merchantCountryCode: 'US',
          currencyCode: 'USD',
          billingAddressConfig: {
            isRequired: true,
            format: PlatformPay.BillingAddressFormat.Full,
          },
        },
      });

      if (error) {
        onError(error.message);
      } else {
        onSuccess();
      }
    } catch (err) {
      onError('Payment failed. Please try again.');
    }
  }, [amount, confirmPlatformPayPayment, onSuccess, onError]);

  if (!isSupported) {
    return null; // Don't show the button if Apple Pay / Google Pay isn't available
  }

  return (
    <View style={styles.container}>
      <PlatformPayButton
        onPress={handleApplePay}
        type={
          Platform.OS === 'ios'
            ? PlatformPay.ButtonType.Pay
            : PlatformPay.ButtonType.Pay
        }
        appearance={PlatformPay.ButtonStyle.Black}
        style={styles.payButton}
      />
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    marginBottom: 16,
  },
  payButton: {
    width: '100%',
    height: 50,
  },
});
```

**Setup requirements for Apple Pay:**

1. An Apple Developer account with Apple Pay capability enabled
2. A merchant identifier registered in the Apple Developer portal (e.g., `merchant.com.mycompany.myapp`)
3. The merchant ID configured in your Stripe Dashboard under Settings > Payment Methods > Apple Pay
4. The `merchantIdentifier` in your Expo config plugin (shown earlier)

Apple Pay only works on real devices, not in the iOS Simulator. Plan your testing accordingly.

---

## 3. NEXT.JS + STRIPE (WEB)

### 3.1 SDK Setup

For web, you need two packages:

```bash
npm install @stripe/stripe-js @stripe/react-stripe-js stripe
```

- `@stripe/stripe-js` -- the Stripe.js loader (client-side)
- `@stripe/react-stripe-js` -- React components for Stripe Elements
- `stripe` -- the Node.js Stripe SDK (server-side only)

```typescript
// lib/stripe/client.ts
import { loadStripe, Stripe } from '@stripe/stripe-js';

let stripePromise: Promise<Stripe | null>;

export function getStripe(): Promise<Stripe | null> {
  if (!stripePromise) {
    stripePromise = loadStripe(
      process.env.NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY!
    );
  }
  return stripePromise;
}
```

```typescript
// lib/stripe/server.ts
// This file should ONLY be imported in server-side code
import Stripe from 'stripe';

export const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
  apiVersion: '2025-03-31.basil',
});
```

**Warning**: Make sure `lib/stripe/server.ts` is never imported in client-side code. Next.js will throw an error if you try to use `process.env.STRIPE_SECRET_KEY` (without `NEXT_PUBLIC_` prefix) in a Client Component, but be careful with shared utility files that might get imported on both sides. A good safeguard:

```typescript
// lib/stripe/server.ts
import 'server-only'; // This import ensures the file can only be used server-side

import Stripe from 'stripe';

export const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
  apiVersion: '2025-03-31.basil',
});
```

### 3.2 Stripe Elements: Embedded Payment Forms

Stripe Elements are pre-built, secure UI components that collect card details. They run in iframes hosted by Stripe, so card data never touches your server or your JavaScript code. This is how you stay PCI compliant on the web.

```typescript
// components/CheckoutForm.tsx
'use client';

import { useState, FormEvent } from 'react';
import {
  PaymentElement,
  useStripe,
  useElements,
} from '@stripe/react-stripe-js';

interface CheckoutFormProps {
  clientSecret: string;
  onSuccess: () => void;
}

export function CheckoutForm({ clientSecret, onSuccess }: CheckoutFormProps) {
  const stripe = useStripe();
  const elements = useElements();
  const [error, setError] = useState<string | null>(null);
  const [processing, setProcessing] = useState(false);

  const handleSubmit = async (event: FormEvent) => {
    event.preventDefault();

    if (!stripe || !elements) {
      // Stripe.js hasn't loaded yet
      return;
    }

    setProcessing(true);
    setError(null);

    const { error: submitError } = await stripe.confirmPayment({
      elements,
      confirmParams: {
        return_url: `${window.location.origin}/payment/success`,
      },
      redirect: 'if_required', // Only redirect for 3D Secure, etc.
    });

    if (submitError) {
      setError(submitError.message ?? 'An unexpected error occurred.');
      setProcessing(false);
    } else {
      // Payment succeeded without redirect
      onSuccess();
    }
  };

  return (
    <form onSubmit={handleSubmit} className="space-y-6">
      <PaymentElement
        options={{
          layout: 'tabs', // 'tabs' | 'accordion' | 'auto'
        }}
      />

      {error && (
        <div className="rounded-md bg-red-50 p-4 text-sm text-red-700">
          {error}
        </div>
      )}

      <button
        type="submit"
        disabled={!stripe || processing}
        className="w-full rounded-lg bg-blue-600 px-4 py-3 text-white font-semibold
                   hover:bg-blue-700 disabled:opacity-50 disabled:cursor-not-allowed
                   transition-colors"
      >
        {processing ? 'Processing...' : 'Pay Now'}
      </button>
    </form>
  );
}
```

Wrap it with the Elements provider:

```typescript
// app/checkout/page.tsx
'use client';

import { useEffect, useState } from 'react';
import { Elements } from '@stripe/react-stripe-js';
import { getStripe } from '@/lib/stripe/client';
import { CheckoutForm } from '@/components/CheckoutForm';

export default function CheckoutPage() {
  const [clientSecret, setClientSecret] = useState<string | null>(null);

  useEffect(() => {
    // Create PaymentIntent when the page loads
    fetch('/api/payments/create-intent', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        amount: 2999, // $29.99
      }),
    })
      .then((res) => res.json())
      .then((data) => setClientSecret(data.clientSecret));
  }, []);

  if (!clientSecret) {
    return (
      <div className="flex items-center justify-center min-h-screen">
        <div className="animate-spin h-8 w-8 border-4 border-blue-600 border-t-transparent rounded-full" />
      </div>
    );
  }

  return (
    <div className="max-w-md mx-auto py-12 px-4">
      <h1 className="text-2xl font-bold mb-8">Checkout</h1>
      <Elements
        stripe={getStripe()}
        options={{
          clientSecret,
          appearance: {
            theme: 'stripe', // 'stripe' | 'night' | 'flat'
            variables: {
              colorPrimary: '#0066FF',
              borderRadius: '8px',
            },
          },
        }}
      >
        <CheckoutForm
          clientSecret={clientSecret}
          onSuccess={() => {
            window.location.href = '/payment/success';
          }}
        />
      </Elements>
    </div>
  );
}
```

### 3.3 Stripe Checkout: The Zero-Code Option

If you do not want to build any payment UI at all, Stripe Checkout is a hosted payment page. You redirect the user to Stripe's domain, they pay there, and Stripe redirects them back.

This is great for:
- MVPs where speed matters more than customization
- Simple one-off payments or donations
- Situations where you want Stripe to handle 100% of the payment UX

```typescript
// app/api/checkout/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { stripe } from '@/lib/stripe/server';

export async function POST(request: NextRequest) {
  try {
    const body = await request.json();
    const { priceId, quantity = 1 } = body;

    const session = await stripe.checkout.sessions.create({
      mode: 'payment', // 'payment' | 'subscription' | 'setup'
      payment_method_types: ['card'],
      line_items: [
        {
          price: priceId, // A Stripe Price ID (price_xxx)
          quantity,
        },
      ],
      success_url: `${process.env.NEXT_PUBLIC_BASE_URL}/payment/success?session_id={CHECKOUT_SESSION_ID}`,
      cancel_url: `${process.env.NEXT_PUBLIC_BASE_URL}/payment/canceled`,
      metadata: {
        userId: 'user_123', // From auth
      },
    });

    return NextResponse.json({ url: session.url });
  } catch (error) {
    console.error('Checkout session error:', error);
    return NextResponse.json(
      { error: 'Failed to create checkout session' },
      { status: 500 }
    );
  }
}
```

```typescript
// components/CheckoutButton.tsx
'use client';

export function CheckoutButton({ priceId }: { priceId: string }) {
  const handleCheckout = async () => {
    const response = await fetch('/api/checkout', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ priceId }),
    });

    const { url } = await response.json();
    window.location.href = url; // Redirect to Stripe Checkout
  };

  return (
    <button
      onClick={handleCheckout}
      className="rounded-lg bg-blue-600 px-6 py-3 text-white font-semibold hover:bg-blue-700"
    >
      Buy Now
    </button>
  );
}
```

### 3.4 Server Actions for Payment Intents

If you prefer Next.js Server Actions over API Routes, here is how to create PaymentIntents:

```typescript
// app/actions/payments.ts
'use server';

import { stripe } from '@/lib/stripe/server';
import { auth } from '@/lib/auth'; // Your auth solution

export async function createPaymentIntent(amount: number, currency: string = 'usd') {
  // Authenticate the user
  const session = await auth();
  if (!session?.user) {
    throw new Error('Unauthorized');
  }

  // Validate the amount on the server
  if (amount < 50) {
    throw new Error('Amount must be at least $0.50');
  }

  // In production, calculate amount from cart/order, don't trust client
  const paymentIntent = await stripe.paymentIntents.create({
    amount,
    currency,
    metadata: {
      userId: session.user.id,
    },
    automatic_payment_methods: {
      enabled: true,
    },
  });

  // Only return the client_secret, never the full PaymentIntent
  return {
    clientSecret: paymentIntent.client_secret!,
  };
}
```

```typescript
// app/checkout/page.tsx
'use client';

import { useEffect, useState } from 'react';
import { Elements } from '@stripe/react-stripe-js';
import { getStripe } from '@/lib/stripe/client';
import { createPaymentIntent } from '@/app/actions/payments';
import { CheckoutForm } from '@/components/CheckoutForm';

export default function CheckoutPage() {
  const [clientSecret, setClientSecret] = useState<string | null>(null);

  useEffect(() => {
    createPaymentIntent(2999).then(({ clientSecret }) => {
      setClientSecret(clientSecret);
    });
  }, []);

  // ... rest is the same as before
}
```

Server Actions are convenient but have a subtle gotcha for payments: error handling. If a Server Action throws, Next.js will serialize the error and send it to the client. Make sure you are not leaking internal error details:

```typescript
// BAD -- leaks internal details
export async function createPaymentIntent(amount: number) {
  const paymentIntent = await stripe.paymentIntents.create({ ... });
  return { clientSecret: paymentIntent.client_secret! };
  // If Stripe throws, the raw Stripe error message goes to the client
}

// GOOD -- controlled error handling
export async function createPaymentIntent(amount: number) {
  try {
    const paymentIntent = await stripe.paymentIntents.create({ ... });
    return { clientSecret: paymentIntent.client_secret!, error: null };
  } catch (error) {
    console.error('PaymentIntent creation failed:', error);
    return { clientSecret: null, error: 'Payment initialization failed. Please try again.' };
  }
}
```

---

## 4. WEBHOOK ARCHITECTURE

### 4.1 Why Webhooks Are Essential

This is the part most developers get wrong, and it is the part that causes the worst bugs.

Here is the scenario: a user taps "Pay Now" on your app. The Payment Sheet appears. They enter their card. The payment processes. The Payment Sheet dismisses. The user sees a "Success!" screen. They are happy.

But what if:
- Their phone dies right after confirming the payment, before your app can update the order status?
- The network drops and the success response from Stripe never reaches the client?
- 3D Secure redirects them to their bank's app, and they never come back?
- The payment requires asynchronous processing (SEPA, bank transfers)?

In all these cases, the client never gets the success signal. If your order processing depends on the client telling your server "the payment worked," you will have paid orders that are never fulfilled.

**The solution: webhooks.** Stripe sends HTTP requests to your server when events happen. The `payment_intent.succeeded` webhook is the authoritative signal that a payment worked. Not the client. Not the API response. The webhook.

**The golden rule**: Trust webhooks for order fulfillment, not client-side callbacks. The client-side success is for UX only -- showing the user a nice "Thank you" screen. The webhook is for business logic -- updating your database, sending confirmation emails, triggering shipping.

### 4.2 Webhook Endpoint in Next.js

```typescript
// app/api/webhooks/stripe/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { stripe } from '@/lib/stripe/server';
import Stripe from 'stripe';

// CRITICAL: Disable Next.js body parsing for webhooks
// Stripe needs the raw body to verify the signature
export const runtime = 'nodejs'; // Webhooks need Node.js runtime, not Edge

const webhookSecret = process.env.STRIPE_WEBHOOK_SECRET!;

export async function POST(request: NextRequest) {
  const body = await request.text(); // Raw body, not JSON
  const signature = request.headers.get('stripe-signature');

  if (!signature) {
    return NextResponse.json(
      { error: 'Missing stripe-signature header' },
      { status: 400 }
    );
  }

  let event: Stripe.Event;

  // Step 1: Verify the webhook signature
  // This proves the request actually came from Stripe, not an attacker
  try {
    event = stripe.webhooks.constructEvent(body, signature, webhookSecret);
  } catch (err) {
    const message = err instanceof Error ? err.message : 'Unknown error';
    console.error(`Webhook signature verification failed: ${message}`);
    return NextResponse.json(
      { error: 'Invalid signature' },
      { status: 400 }
    );
  }

  // Step 2: Handle the event
  try {
    switch (event.type) {
      case 'payment_intent.succeeded':
        await handlePaymentSuccess(event.data.object as Stripe.PaymentIntent);
        break;

      case 'payment_intent.payment_failed':
        await handlePaymentFailure(event.data.object as Stripe.PaymentIntent);
        break;

      case 'charge.refunded':
        await handleRefund(event.data.object as Stripe.Charge);
        break;

      case 'invoice.paid':
        await handleInvoicePaid(event.data.object as Stripe.Invoice);
        break;

      case 'invoice.payment_failed':
        await handleInvoicePaymentFailed(event.data.object as Stripe.Invoice);
        break;

      case 'customer.subscription.created':
      case 'customer.subscription.updated':
        await handleSubscriptionUpdate(event.data.object as Stripe.Subscription);
        break;

      case 'customer.subscription.deleted':
        await handleSubscriptionCanceled(event.data.object as Stripe.Subscription);
        break;

      default:
        // Unhandled event type -- log but don't error
        console.log(`Unhandled webhook event: ${event.type}`);
    }

    // Step 3: Return 200 to acknowledge receipt
    // If you don't return 200, Stripe will retry the webhook
    return NextResponse.json({ received: true });

  } catch (error) {
    console.error(`Webhook handler error for ${event.type}:`, error);
    // Return 500 so Stripe retries
    return NextResponse.json(
      { error: 'Webhook handler failed' },
      { status: 500 }
    );
  }
}

// --- Event Handlers ---

async function handlePaymentSuccess(paymentIntent: Stripe.PaymentIntent) {
  const orderId = paymentIntent.metadata.orderId;

  if (!orderId) {
    console.error('PaymentIntent missing orderId metadata:', paymentIntent.id);
    return;
  }

  // Idempotency check: has this payment already been processed?
  // This is CRITICAL. Stripe may send the same event multiple times.
  // const order = await db.order.findUnique({ where: { id: orderId } });
  // if (order?.status === 'paid') {
  //   console.log(`Order ${orderId} already marked as paid. Skipping.`);
  //   return;
  // }

  // Update order status
  // await db.order.update({
  //   where: { id: orderId },
  //   data: {
  //     status: 'paid',
  //     stripePaymentIntentId: paymentIntent.id,
  //     paidAt: new Date(),
  //   },
  // });

  // Trigger post-payment actions
  // await sendOrderConfirmationEmail(orderId);
  // await notifyFulfillmentService(orderId);

  console.log(`Payment succeeded for order ${orderId}`);
}

async function handlePaymentFailure(paymentIntent: Stripe.PaymentIntent) {
  const orderId = paymentIntent.metadata.orderId;

  // await db.order.update({
  //   where: { id: orderId },
  //   data: {
  //     status: 'payment_failed',
  //     failureReason: paymentIntent.last_payment_error?.message,
  //   },
  // });

  console.log(`Payment failed for order ${orderId}`);
}

async function handleRefund(charge: Stripe.Charge) {
  // Handle full and partial refunds
  const paymentIntentId = charge.payment_intent as string;

  // await db.order.update({
  //   where: { stripePaymentIntentId: paymentIntentId },
  //   data: {
  //     status: charge.amount_refunded === charge.amount ? 'refunded' : 'partially_refunded',
  //     refundedAmount: charge.amount_refunded,
  //   },
  // });

  console.log(`Refund processed for charge ${charge.id}`);
}

async function handleInvoicePaid(invoice: Stripe.Invoice) {
  // Subscription payment succeeded
  const customerId = invoice.customer as string;
  const subscriptionId = invoice.subscription as string;

  // await db.subscription.update({
  //   where: { stripeSubscriptionId: subscriptionId },
  //   data: {
  //     status: 'active',
  //     currentPeriodEnd: new Date((invoice.lines.data[0]?.period?.end ?? 0) * 1000),
  //   },
  // });

  console.log(`Invoice paid for subscription ${subscriptionId}`);
}

async function handleInvoicePaymentFailed(invoice: Stripe.Invoice) {
  const subscriptionId = invoice.subscription as string;

  // await db.subscription.update({
  //   where: { stripeSubscriptionId: subscriptionId },
  //   data: { status: 'past_due' },
  // });

  // Send email: "Your payment failed. Please update your payment method."
  console.log(`Invoice payment failed for subscription ${subscriptionId}`);
}

async function handleSubscriptionUpdate(subscription: Stripe.Subscription) {
  // await db.subscription.upsert({
  //   where: { stripeSubscriptionId: subscription.id },
  //   update: {
  //     status: subscription.status,
  //     priceId: subscription.items.data[0]?.price.id,
  //     currentPeriodEnd: new Date(subscription.current_period_end * 1000),
  //     cancelAtPeriodEnd: subscription.cancel_at_period_end,
  //   },
  //   create: {
  //     stripeSubscriptionId: subscription.id,
  //     stripeCustomerId: subscription.customer as string,
  //     status: subscription.status,
  //     priceId: subscription.items.data[0]?.price.id,
  //     currentPeriodEnd: new Date(subscription.current_period_end * 1000),
  //     cancelAtPeriodEnd: subscription.cancel_at_period_end,
  //   },
  // });

  console.log(`Subscription updated: ${subscription.id} -> ${subscription.status}`);
}

async function handleSubscriptionCanceled(subscription: Stripe.Subscription) {
  // await db.subscription.update({
  //   where: { stripeSubscriptionId: subscription.id },
  //   data: {
  //     status: 'canceled',
  //     canceledAt: new Date(),
  //   },
  // });

  console.log(`Subscription canceled: ${subscription.id}`);
}
```

### 4.3 Webhook Signature Verification

**Never skip signature verification.** Without it, anyone who discovers your webhook URL can send fake events to your server. They could mark their orders as "paid" without paying, trigger refunds, or cause all sorts of chaos.

The signature verification works like this:

1. When you register a webhook endpoint in the Stripe Dashboard, Stripe generates a `whsec_` secret.
2. For every webhook event, Stripe signs the payload using this secret and includes the signature in the `stripe-signature` header.
3. Your server uses the same secret to verify the signature matches.
4. If the signature does not match, the request is fake. Reject it.

```typescript
// This is what's happening inside stripe.webhooks.constructEvent():
// 1. Extract timestamp and signatures from the header
// 2. Prepare the signed payload: `${timestamp}.${rawBody}`
// 3. Compute expected signature: HMAC-SHA256(webhookSecret, signedPayload)
// 4. Compare with the provided signature
// 5. Check that the timestamp is recent (prevents replay attacks)
```

You need the **raw request body** for signature verification. If your framework parses the body as JSON first, the signature will not match because JSON parsing and re-serialization can change whitespace and key ordering. That is why we use `request.text()` instead of `request.json()` in the webhook handler.

### 4.4 Idempotency: Handle Events Multiple Times Safely

Stripe guarantees **at-least-once delivery** for webhooks. This means they might send the same event multiple times. Your webhook handler must be idempotent -- processing the same event twice should have the same result as processing it once.

Common patterns for idempotency:

```typescript
// Pattern 1: Check before acting
async function handlePaymentSuccess(paymentIntent: Stripe.PaymentIntent) {
  const order = await db.order.findUnique({
    where: { stripePaymentIntentId: paymentIntent.id },
  });

  // Already processed? Skip.
  if (order?.status === 'paid') {
    console.log(`Already processed payment for ${paymentIntent.id}`);
    return;
  }

  await db.order.update({
    where: { stripePaymentIntentId: paymentIntent.id },
    data: { status: 'paid' },
  });
}

// Pattern 2: Use Stripe event ID as idempotency key
async function handleWebhookEvent(event: Stripe.Event) {
  // Check if we've already processed this event
  const existing = await db.processedWebhookEvent.findUnique({
    where: { stripeEventId: event.id },
  });

  if (existing) {
    console.log(`Event ${event.id} already processed`);
    return;
  }

  // Process the event...
  await processEvent(event);

  // Mark as processed
  await db.processedWebhookEvent.create({
    data: {
      stripeEventId: event.id,
      type: event.type,
      processedAt: new Date(),
    },
  });
}

// Pattern 3: Database transactions with unique constraints
async function handlePaymentSuccess(paymentIntent: Stripe.PaymentIntent) {
  try {
    await db.$transaction(async (tx) => {
      // The unique constraint on stripePaymentIntentId will prevent double-processing
      await tx.payment.create({
        data: {
          stripePaymentIntentId: paymentIntent.id, // unique
          amount: paymentIntent.amount,
          status: 'succeeded',
        },
      });

      await tx.order.update({
        where: { id: paymentIntent.metadata.orderId },
        data: { status: 'paid' },
      });
    });
  } catch (error) {
    // If it's a unique constraint violation, the event was already processed
    if (isPrismaUniqueConstraintError(error)) {
      console.log(`Payment ${paymentIntent.id} already recorded`);
      return;
    }
    throw error;
  }
}
```

I prefer Pattern 3. Using database constraints to enforce idempotency is more reliable than check-then-act (Pattern 1), which has a race condition if two webhook deliveries arrive simultaneously.

### 4.5 Webhook Retry Logic

Stripe retries failed webhooks (those that return non-2xx responses) with exponential backoff for up to 3 days. The retry schedule is approximately:

- 1st retry: ~1 minute after failure
- 2nd retry: ~5 minutes
- 3rd retry: ~30 minutes
- 4th retry: ~2 hours
- 5th retry: ~8 hours
- And so on, up to about 3 days of retries

This means:
- If your server is temporarily down, you will not miss events. Stripe will keep trying.
- Your webhook handlers need to be idempotent (as discussed above) because you will receive retries.
- If your webhook handler is consistently failing, you have 3 days to fix it before events are permanently dropped.
- Monitor your webhook failures in the Stripe Dashboard. Set up alerts.

---

## 5. SUBSCRIPTIONS

### 5.1 Stripe Billing Overview

Subscriptions are recurring payments. Stripe Billing handles the entire lifecycle: creating subscriptions, generating invoices, charging customers on schedule, handling failed payments, and sending dunning emails.

The key objects:

```
Product (e.g., "Pro Plan")
  └── Price (e.g., "$29/month", "$290/year")
        └── Subscription (a customer subscribed to this price)
              └── Invoice (generated each billing period)
                    └── PaymentIntent (the actual charge attempt)
```

### 5.2 Creating Products and Prices

You can create these in the Stripe Dashboard (recommended for most teams) or via the API:

```typescript
// scripts/setup-stripe-products.ts
// Run this once to set up your products and prices
import Stripe from 'stripe';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);

async function setupProducts() {
  // Create the product
  const product = await stripe.products.create({
    name: 'Pro Plan',
    description: 'Full access to all features',
    metadata: {
      tier: 'pro',
    },
  });

  // Create monthly price
  const monthlyPrice = await stripe.prices.create({
    product: product.id,
    unit_amount: 2999, // $29.99
    currency: 'usd',
    recurring: {
      interval: 'month',
    },
    metadata: {
      plan: 'pro_monthly',
    },
  });

  // Create yearly price (with discount)
  const yearlyPrice = await stripe.prices.create({
    product: product.id,
    unit_amount: 29900, // $299.00 (save ~$60/year)
    currency: 'usd',
    recurring: {
      interval: 'year',
    },
    metadata: {
      plan: 'pro_yearly',
    },
  });

  console.log('Product:', product.id);
  console.log('Monthly price:', monthlyPrice.id);
  console.log('Yearly price:', yearlyPrice.id);

  // Store these IDs in your environment variables or database
  // STRIPE_PRO_MONTHLY_PRICE_ID=price_xxx
  // STRIPE_PRO_YEARLY_PRICE_ID=price_xxx
}

setupProducts();
```

**Never hardcode prices in your app.** Always use Stripe Price IDs. This lets you change prices in the Stripe Dashboard without deploying code. Your app asks the server "what are the current plan options?" and the server returns the Price IDs and amounts from Stripe.

### 5.3 Creating Subscriptions

```typescript
// app/api/subscriptions/create/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { stripe } from '@/lib/stripe/server';

export async function POST(request: NextRequest) {
  try {
    const { priceId, customerId } = await request.json();

    // Create the subscription
    const subscription = await stripe.subscriptions.create({
      customer: customerId,
      items: [{ price: priceId }],
      payment_behavior: 'default_incomplete', // Don't charge immediately
      payment_settings: {
        save_default_payment_method: 'on_subscription',
      },
      expand: ['latest_invoice.payment_intent'],
    });

    const invoice = subscription.latest_invoice as Stripe.Invoice;
    const paymentIntent = invoice.payment_intent as Stripe.PaymentIntent;

    return NextResponse.json({
      subscriptionId: subscription.id,
      clientSecret: paymentIntent.client_secret,
    });
  } catch (error) {
    console.error('Subscription creation error:', error);
    return NextResponse.json(
      { error: 'Failed to create subscription' },
      { status: 500 }
    );
  }
}
```

With `payment_behavior: 'default_incomplete'`, the subscription is created but not yet active. It waits for the client to confirm the first payment. This is the recommended approach because it lets you use the Payment Sheet or Elements to collect payment on the client.

### 5.4 Managing Subscription Lifecycle

```typescript
// app/api/subscriptions/manage/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { stripe } from '@/lib/stripe/server';

// Cancel subscription (at period end -- the user keeps access until their current period expires)
export async function DELETE(request: NextRequest) {
  const { subscriptionId } = await request.json();

  const subscription = await stripe.subscriptions.update(subscriptionId, {
    cancel_at_period_end: true,
  });

  return NextResponse.json({
    status: subscription.status,
    cancelAtPeriodEnd: subscription.cancel_at_period_end,
    currentPeriodEnd: new Date(subscription.current_period_end * 1000),
  });
}

// Reactivate a subscription that was set to cancel at period end
export async function PATCH(request: NextRequest) {
  const { subscriptionId, action, newPriceId } = await request.json();

  if (action === 'reactivate') {
    const subscription = await stripe.subscriptions.update(subscriptionId, {
      cancel_at_period_end: false,
    });
    return NextResponse.json({ status: subscription.status });
  }

  if (action === 'change_plan') {
    // Upgrade or downgrade
    const subscription = await stripe.subscriptions.retrieve(subscriptionId);

    const updatedSubscription = await stripe.subscriptions.update(subscriptionId, {
      items: [
        {
          id: subscription.items.data[0].id,
          price: newPriceId,
        },
      ],
      // Prorate by default -- charge/credit the difference immediately
      proration_behavior: 'create_prorations',
    });

    return NextResponse.json({
      status: updatedSubscription.status,
      priceId: updatedSubscription.items.data[0].price.id,
    });
  }

  return NextResponse.json({ error: 'Invalid action' }, { status: 400 });
}
```

### 5.5 Displaying Subscription Status

```typescript
// app/api/subscriptions/status/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { stripe } from '@/lib/stripe/server';

export async function GET(request: NextRequest) {
  const customerId = 'cus_xxx'; // Get from authenticated user's profile

  const subscriptions = await stripe.subscriptions.list({
    customer: customerId,
    status: 'all',
    limit: 1,
    expand: ['data.default_payment_method'],
  });

  if (subscriptions.data.length === 0) {
    return NextResponse.json({ subscription: null });
  }

  const sub = subscriptions.data[0];
  const paymentMethod = sub.default_payment_method as Stripe.PaymentMethod | null;

  return NextResponse.json({
    subscription: {
      id: sub.id,
      status: sub.status,
      plan: sub.items.data[0].price.id,
      amount: sub.items.data[0].price.unit_amount,
      currency: sub.items.data[0].price.currency,
      interval: sub.items.data[0].price.recurring?.interval,
      currentPeriodEnd: new Date(sub.current_period_end * 1000).toISOString(),
      cancelAtPeriodEnd: sub.cancel_at_period_end,
      paymentMethod: paymentMethod
        ? {
            brand: paymentMethod.card?.brand,
            last4: paymentMethod.card?.last4,
            expMonth: paymentMethod.card?.exp_month,
            expYear: paymentMethod.card?.exp_year,
          }
        : null,
    },
  });
}
```

```typescript
// React Native or Web component
// components/SubscriptionStatus.tsx
import { useEffect, useState } from 'react';
import { View, Text, StyleSheet, Pressable } from 'react-native';

interface SubscriptionData {
  id: string;
  status: string;
  plan: string;
  amount: number;
  currency: string;
  interval: string;
  currentPeriodEnd: string;
  cancelAtPeriodEnd: boolean;
  paymentMethod: {
    brand: string;
    last4: string;
    expMonth: number;
    expYear: number;
  } | null;
}

export function SubscriptionStatus() {
  const [subscription, setSubscription] = useState<SubscriptionData | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetch('/api/subscriptions/status')
      .then((res) => res.json())
      .then((data) => {
        setSubscription(data.subscription);
        setLoading(false);
      });
  }, []);

  if (loading) {
    return <Text>Loading...</Text>;
  }

  if (!subscription) {
    return (
      <View style={styles.container}>
        <Text style={styles.title}>No active subscription</Text>
        <Pressable style={styles.button}>
          <Text style={styles.buttonText}>Subscribe to Pro</Text>
        </Pressable>
      </View>
    );
  }

  const statusColors: Record<string, string> = {
    active: '#22C55E',
    past_due: '#F59E0B',
    canceled: '#EF4444',
    trialing: '#3B82F6',
  };

  return (
    <View style={styles.container}>
      <View style={styles.header}>
        <Text style={styles.title}>Pro Plan</Text>
        <View style={[styles.badge, { backgroundColor: statusColors[subscription.status] || '#999' }]}>
          <Text style={styles.badgeText}>{subscription.status}</Text>
        </View>
      </View>

      <Text style={styles.price}>
        ${(subscription.amount / 100).toFixed(2)}/{subscription.interval}
      </Text>

      {subscription.paymentMethod && (
        <Text style={styles.detail}>
          {subscription.paymentMethod.brand} ending in {subscription.paymentMethod.last4}
        </Text>
      )}

      <Text style={styles.detail}>
        {subscription.cancelAtPeriodEnd
          ? `Cancels on ${new Date(subscription.currentPeriodEnd).toLocaleDateString()}`
          : `Renews on ${new Date(subscription.currentPeriodEnd).toLocaleDateString()}`}
      </Text>

      {subscription.cancelAtPeriodEnd && (
        <Pressable style={[styles.button, styles.reactivateButton]}>
          <Text style={styles.buttonText}>Reactivate Subscription</Text>
        </Pressable>
      )}
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    padding: 20,
    backgroundColor: '#FFFFFF',
    borderRadius: 16,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.05,
    shadowRadius: 8,
    elevation: 2,
  },
  header: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    marginBottom: 8,
  },
  title: {
    fontSize: 20,
    fontWeight: '700',
    color: '#111',
  },
  badge: {
    paddingHorizontal: 10,
    paddingVertical: 4,
    borderRadius: 12,
  },
  badgeText: {
    color: '#FFFFFF',
    fontSize: 12,
    fontWeight: '600',
    textTransform: 'capitalize',
  },
  price: {
    fontSize: 28,
    fontWeight: '700',
    color: '#111',
    marginBottom: 12,
  },
  detail: {
    fontSize: 14,
    color: '#666',
    marginBottom: 4,
  },
  button: {
    backgroundColor: '#0066FF',
    paddingVertical: 14,
    borderRadius: 10,
    alignItems: 'center',
    marginTop: 16,
  },
  reactivateButton: {
    backgroundColor: '#22C55E',
  },
  buttonText: {
    color: '#FFFFFF',
    fontSize: 16,
    fontWeight: '600',
  },
});
```

### 5.6 Grace Periods and Dunning

"Dunning" is the process of retrying failed subscription payments. Stripe handles this automatically, but you need to understand it and configure it:

1. **Automatic retries**: When a subscription payment fails, Stripe retries up to 4 times over about 3 weeks. You configure the retry schedule in the Stripe Dashboard under Settings > Billing > Subscriptions.

2. **Smart Retries**: Stripe uses machine learning to pick the optimal retry time based on when the card is most likely to succeed. Enable this in the Dashboard.

3. **Grace periods**: While Stripe is retrying, the subscription is in `past_due` status. You decide what this means for the user's access:

```typescript
// lib/subscription.ts
export function canAccessProFeatures(subscriptionStatus: string): boolean {
  // These statuses mean the user should have access
  const activeStatuses = [
    'active',
    'trialing',
    'past_due', // Grace period -- still retrying payment
  ];

  return activeStatuses.includes(subscriptionStatus);
}

export function shouldShowPaymentWarning(subscriptionStatus: string): boolean {
  return subscriptionStatus === 'past_due';
}

export function isSubscriptionEnded(subscriptionStatus: string): boolean {
  return ['canceled', 'unpaid', 'incomplete_expired'].includes(subscriptionStatus);
}
```

4. **Dunning emails**: Stripe can send automatic emails to customers when their payment fails. Configure these in the Dashboard. The emails include a link to update their payment method. This alone recovers a significant percentage of failed payments.

5. **After all retries fail**: The subscription moves to `unpaid` or `canceled` (you choose in Dashboard settings). At this point, revoke access and show the user a screen to resubscribe.

---

## 6. IN-APP PURCHASES (IAP)

### 6.1 When You Must Use IAP vs When You Can Use Stripe

This is the most important business decision in this chapter. Get it wrong and Apple will reject your app or remove it from the store.

**Apple's rules (and Google's are similar):**

| What You're Selling | Payment Method | Rule |
|---------------------|---------------|------|
| Digital goods / services consumed in the app | Apple IAP / Google Play Billing | **REQUIRED** |
| Physical goods or services delivered outside the app | Stripe (or any processor) | Allowed |
| Person-to-person services (Uber, Airbnb) | Stripe | Allowed |
| SaaS subscriptions (if only accessed in app) | Apple IAP | **REQUIRED** |
| SaaS subscriptions (accessed via web too) | Stripe or IAP | Grey area -- many companies use Stripe via web signup |

**Examples:**

- Selling premium filters in a photo app? **IAP required.**
- Selling a food delivery? **Stripe is fine.**
- Selling a monthly subscription to your productivity app? **IAP required** (if signup is in the app).
- Selling access to a SaaS that has a web app? **Stripe via web signup is usually fine.** Many companies (Netflix, Spotify, Amazon) direct users to their website to subscribe, avoiding the 30% commission.

The reason this matters: Apple and Google take a **30% commission** on IAP transactions (15% for small businesses under $1M/year and for subscriptions after the first year). That is $30 out of every $100. On Stripe, you pay about 2.9% + $0.30 -- roughly $3.20 on a $100 transaction.

This is a 10x difference in payment processing costs. It is a legitimate business consideration.

### 6.2 RevenueCat: The IAP Abstraction Layer

If you need IAP, use RevenueCat. It abstracts over Apple's StoreKit and Google Play Billing, providing a unified API, server-side receipt validation, subscription analytics, and a web dashboard.

Without RevenueCat, you would need to:
- Implement StoreKit 2 for iOS and Google Play Billing for Android separately
- Build server-side receipt validation for both platforms
- Handle subscription lifecycle events from both stores
- Build analytics for MRR, churn, trial conversions, etc.
- Deal with platform-specific edge cases (family sharing, grace periods, billing retry)

RevenueCat handles all of this.

```bash
# Install RevenueCat
npx expo install react-native-purchases
```

```typescript
// app.config.ts
const config: ExpoConfig = {
  // ...
  plugins: [
    'react-native-purchases',
    // ... other plugins
  ],
};
```

### 6.3 RevenueCat Setup

```typescript
// src/providers/RevenueCatProvider.tsx
import { createContext, useContext, useEffect, useState, ReactNode } from 'react';
import Purchases, {
  CustomerInfo,
  PurchasesOffering,
  LOG_LEVEL,
} from 'react-native-purchases';
import { Platform } from 'react-native';

const REVENUECAT_API_KEY_IOS = process.env.EXPO_PUBLIC_REVENUECAT_IOS_KEY!;
const REVENUECAT_API_KEY_ANDROID = process.env.EXPO_PUBLIC_REVENUECAT_ANDROID_KEY!;

interface RevenueCatContextType {
  customerInfo: CustomerInfo | null;
  offerings: PurchasesOffering | null;
  isPro: boolean;
  loading: boolean;
  purchasePackage: (packageId: string) => Promise<boolean>;
  restorePurchases: () => Promise<void>;
}

const RevenueCatContext = createContext<RevenueCatContextType | null>(null);

export function RevenueCatProvider({ children }: { children: ReactNode }) {
  const [customerInfo, setCustomerInfo] = useState<CustomerInfo | null>(null);
  const [offerings, setOfferings] = useState<PurchasesOffering | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    async function init() {
      // Configure RevenueCat
      if (__DEV__) {
        Purchases.setLogLevel(LOG_LEVEL.DEBUG);
      }

      const apiKey = Platform.OS === 'ios'
        ? REVENUECAT_API_KEY_IOS
        : REVENUECAT_API_KEY_ANDROID;

      await Purchases.configure({
        apiKey,
        // Optionally identify the user (if you have auth)
        // appUserID: userId,
      });

      // Fetch current customer info
      const info = await Purchases.getCustomerInfo();
      setCustomerInfo(info);

      // Fetch available offerings (products/prices)
      const allOfferings = await Purchases.getOfferings();
      setOfferings(allOfferings.current);

      setLoading(false);

      // Listen for customer info changes
      Purchases.addCustomerInfoUpdateListener((updatedInfo) => {
        setCustomerInfo(updatedInfo);
      });
    }

    init();
  }, []);

  const isPro = customerInfo?.entitlements.active['pro'] !== undefined;

  const purchasePackage = async (packageId: string): Promise<boolean> => {
    try {
      const offering = offerings;
      if (!offering) return false;

      const pkg = offering.availablePackages.find((p) => p.identifier === packageId);
      if (!pkg) return false;

      const { customerInfo: updatedInfo } = await Purchases.purchasePackage(pkg);
      setCustomerInfo(updatedInfo);

      return updatedInfo.entitlements.active['pro'] !== undefined;
    } catch (error: any) {
      if (error.userCancelled) {
        // User canceled -- not an error
        return false;
      }
      throw error;
    }
  };

  const restorePurchases = async () => {
    const info = await Purchases.restorePurchases();
    setCustomerInfo(info);
  };

  return (
    <RevenueCatContext.Provider
      value={{
        customerInfo,
        offerings,
        isPro,
        loading,
        purchasePackage,
        restorePurchases,
      }}
    >
      {children}
    </RevenueCatContext.Provider>
  );
}

export function useRevenueCat() {
  const context = useContext(RevenueCatContext);
  if (!context) {
    throw new Error('useRevenueCat must be used within a RevenueCatProvider');
  }
  return context;
}
```

### 6.4 Paywall Component

```typescript
// src/components/Paywall.tsx
import { View, Text, Pressable, StyleSheet, ActivityIndicator } from 'react-native';
import { useRevenueCat } from '@/providers/RevenueCatProvider';

export function Paywall() {
  const { offerings, isPro, loading, purchasePackage, restorePurchases } = useRevenueCat();

  if (loading) {
    return (
      <View style={styles.center}>
        <ActivityIndicator size="large" />
      </View>
    );
  }

  if (isPro) {
    return (
      <View style={styles.container}>
        <Text style={styles.title}>You are a Pro member!</Text>
        <Text style={styles.subtitle}>Thank you for your support.</Text>
      </View>
    );
  }

  if (!offerings) {
    return (
      <View style={styles.container}>
        <Text style={styles.error}>Unable to load subscription options.</Text>
      </View>
    );
  }

  return (
    <View style={styles.container}>
      <Text style={styles.title}>Upgrade to Pro</Text>
      <Text style={styles.subtitle}>Unlock all features</Text>

      {offerings.availablePackages.map((pkg) => (
        <Pressable
          key={pkg.identifier}
          style={styles.packageCard}
          onPress={() => purchasePackage(pkg.identifier)}
        >
          <View>
            <Text style={styles.packageTitle}>
              {pkg.product.title}
            </Text>
            <Text style={styles.packageDescription}>
              {pkg.product.description}
            </Text>
          </View>
          <Text style={styles.packagePrice}>
            {pkg.product.priceString}
          </Text>
        </Pressable>
      ))}

      <Pressable style={styles.restoreButton} onPress={restorePurchases}>
        <Text style={styles.restoreText}>Restore Purchases</Text>
      </Pressable>
    </View>
  );
}

const styles = StyleSheet.create({
  center: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
  },
  container: {
    flex: 1,
    padding: 24,
  },
  title: {
    fontSize: 28,
    fontWeight: '700',
    color: '#111',
    marginBottom: 8,
  },
  subtitle: {
    fontSize: 16,
    color: '#666',
    marginBottom: 32,
  },
  error: {
    fontSize: 16,
    color: '#EF4444',
    textAlign: 'center',
  },
  packageCard: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    padding: 20,
    backgroundColor: '#F8F9FA',
    borderRadius: 16,
    marginBottom: 12,
    borderWidth: 2,
    borderColor: '#E5E7EB',
  },
  packageTitle: {
    fontSize: 18,
    fontWeight: '600',
    color: '#111',
  },
  packageDescription: {
    fontSize: 14,
    color: '#666',
    marginTop: 4,
  },
  packagePrice: {
    fontSize: 20,
    fontWeight: '700',
    color: '#0066FF',
  },
  restoreButton: {
    alignItems: 'center',
    marginTop: 24,
    padding: 12,
  },
  restoreText: {
    fontSize: 14,
    color: '#0066FF',
  },
});
```

**Important**: Apple requires a "Restore Purchases" button. If you forget it, your app will be rejected. Users who reinstall the app or switch devices need a way to restore their previous purchases.

### 6.5 The 30% Commission and Pricing Strategy

The Apple/Google 30% commission fundamentally changes your pricing math:

| Your Price | You Receive (IAP) | You Receive (Stripe) |
|-----------|-------------------|---------------------|
| $9.99/mo | $6.99 (Apple keeps $3.00) | $9.40 (Stripe fee ~$0.59) |
| $29.99/mo | $20.99 | $28.82 |
| $99.99/yr | $69.99 | $96.79 |

After the first year of a subscription, Apple's cut drops to 15%. Google has a similar program. But that first year is brutal.

Common strategies:
1. **Price higher on IAP**: Charge $12.99/mo in-app but $9.99/mo on web. Many apps do this (even major ones like YouTube Premium).
2. **Web-first subscription**: Direct users to your website to subscribe. Spotify, Netflix, and Amazon all do this.
3. **Hybrid approach**: Offer IAP for convenience but encourage web signup with a lower price.

Note: Apple's rules around "anti-steering" have evolved. As of 2025, in many jurisdictions you can inform users about web-based subscription options, but the rules vary by country and are subject to ongoing legal changes. Research the current guidelines for your target markets.

---

## 7. APPLE PAY AND GOOGLE PAY

### 7.1 Why Wallet Payments Matter

Wallet payments (Apple Pay, Google Pay) have two massive advantages:

1. **Conversion**: Users complete payment in one tap. No typing card numbers. No typos. No "where's my wallet?" friction. Conversion rates for Apple Pay are typically 10-20% higher than card forms.
2. **Security**: The actual card number is never shared. A device-specific token is used instead. This is more secure than traditional card payments.

### 7.2 How Tokenized Payments Work

When a user pays with Apple Pay or Google Pay:

1. The user authenticates with Face ID / fingerprint / PIN
2. The device creates a one-time payment token from the card stored in the wallet
3. This token is sent to Stripe (not the actual card number)
4. Stripe processes the payment using the token
5. Your server never sees any card data at all

```
User's Device          Stripe              Your Server
     │                    │                      │
     │  Payment token     │                      │
     ├───────────────────>│                      │
     │                    │  PaymentIntent        │
     │                    │  succeeded webhook    │
     │                    ├─────────────────────>│
     │                    │                      │
     │  Success response  │                      │
     │<───────────────────│                      │
```

Your server is completely out of the card data flow. This is the ideal architecture.

### 7.3 Merchant ID Configuration

For Apple Pay specifically, you need:

1. **Apple Developer Account**: Register a Merchant ID (e.g., `merchant.com.mycompany.myapp`)
2. **Stripe Dashboard**: Upload the Apple Pay certificate (Settings > Payment Methods > Apple Pay)
3. **Expo Config**: Add the merchant identifier to your app config (shown in Section 2.1)

For Google Pay:
1. **Google Pay API**: Register your app in the Google Pay Business Console
2. **Stripe handles most of the integration** -- you just need to enable Google Pay in your Stripe Dashboard

### 7.4 When to Offer Wallet Payments

Always. If the device supports it, show it. The Payment Sheet does this automatically. If you are using a custom form, add the `ApplePayButton` / `PlatformPayButton` above your card form as a faster alternative.

The only exception is if you are collecting additional information that wallet payments skip (like a shipping address that is not in the wallet). In that case, collect the information first, then offer wallet payment for the final step.

---

## 8. PCI COMPLIANCE

### 8.1 What Stripe Handles for You

PCI DSS (Payment Card Industry Data Security Standard) is a set of security requirements for handling card data. Full PCI compliance is extremely expensive and complex -- it involves network segmentation, regular security audits, penetration testing, and extensive documentation.

Stripe handles the hard parts. When you use Stripe Elements (web) or the Stripe React Native SDK (mobile), card data goes directly from the user's device to Stripe's servers. Your servers never see, process, or store card numbers. This puts you at **SAQ A** (Self-Assessment Questionnaire A), the simplest level of PCI compliance.

### 8.2 SAQ A vs SAQ A-EP

| Level | What It Means | When It Applies |
|-------|---------------|-----------------|
| SAQ A | Simplest. You fully outsource card handling. | Using Stripe Elements, Payment Sheet, or Checkout |
| SAQ A-EP | Moderate. Your web page hosts the payment form. | Your website serves the page containing Stripe Elements |
| SAQ D | Full PCI audit. Very expensive. | You handle raw card data yourself |

For most web applications using Stripe Elements, you technically fall under SAQ A-EP because your server delivers the web page that contains the payment form. This is still very manageable -- it mostly means keeping your website secure (HTTPS, no mixed content, secure hosting).

For mobile apps using the Stripe React Native SDK, you are squarely at SAQ A because the native SDK is a self-contained module that handles everything.

### 8.3 The Rules That Keep You Safe

These are non-negotiable. Violating any of them puts you at risk of PCI penalties, Stripe account termination, and legal liability:

1. **Never log card numbers.** Not in console.log. Not in error reports. Not in analytics. Not in Sentry. Nowhere.

2. **Never store card numbers.** Not in your database. Not in localStorage. Not in AsyncStorage. Not in a file. Not even "temporarily."

3. **Never transmit card numbers through your server.** This is the whole point of Stripe Elements and the Stripe SDK. Card data goes directly from the client to Stripe. If you are building a system where card numbers pass through your API, stop and redesign.

4. **Use HTTPS everywhere.** Your API, your website, your webhook endpoints -- everything. This should be obvious in 2026, but I still see teams with HTTP endpoints in development that accidentally make it to staging.

5. **Never build your own card form with regular text inputs.** Do not create an `<input type="text">` for the card number. Use Stripe Elements or the Stripe SDK's CardField. These components run in secure iframes (web) or secure native views (mobile) that are isolated from your application code.

```typescript
// NEVER DO THIS
// This means card data flows through your JavaScript, which means
// it flows through your server if submitted via a form
function BadCardForm() {
  const [cardNumber, setCardNumber] = useState('');
  const [expiry, setExpiry] = useState('');
  const [cvc, setCvc] = useState('');

  return (
    <form>
      <input
        type="text"
        value={cardNumber}
        onChange={(e) => setCardNumber(e.target.value)}
        placeholder="Card number"
      />
      {/* THIS IS A PCI VIOLATION */}
    </form>
  );
}

// DO THIS INSTEAD
// Stripe Elements handles card input in a secure iframe
function GoodCardForm() {
  return (
    <Elements stripe={stripePromise}>
      <PaymentElement /> {/* Secure, PCI-compliant, hosted by Stripe */}
    </Elements>
  );
}
```

---

## 9. TESTING PAYMENTS

### 9.1 Stripe Test Mode

Stripe test mode is a fully functional sandbox. Every feature works -- PaymentIntents, subscriptions, webhooks, invoices -- but no real money moves. Always develop against test mode.

### 9.2 Test Card Numbers

Stripe provides special card numbers for testing:

| Card Number | Scenario |
|------------|----------|
| `4242 4242 4242 4242` | Succeeds |
| `4000 0000 0000 3220` | Requires 3D Secure authentication |
| `4000 0000 0000 9995` | Declined (insufficient funds) |
| `4000 0000 0000 0002` | Declined (generic) |
| `4000 0000 0000 0069` | Declined (expired card) |
| `4000 0000 0000 0127` | Declined (incorrect CVC) |
| `4000 0025 0000 3155` | Requires 3D Secure 2 (full flow) |
| `4000 0000 0000 3055` | 3D Secure required but optional |

For all test cards:
- Expiry: Any future date (e.g., 12/34)
- CVC: Any 3 digits (e.g., 123)
- ZIP: Any valid ZIP (e.g., 12345)

**Test every scenario.** Do not just test the happy path with 4242. Test declined cards. Test 3D Secure. Test what happens when the user closes the Payment Sheet mid-payment. Test what happens when the network drops. Payment code that only works on the happy path will break in production.

### 9.3 Testing Webhooks Locally with Stripe CLI

In production, Stripe sends webhooks to your public URL. During development, your server is running on localhost. The Stripe CLI solves this:

```bash
# Install the Stripe CLI
brew install stripe/stripe-cli/stripe

# Login to your Stripe account
stripe login

# Forward webhooks to your local server
stripe listen --forward-to localhost:3000/api/webhooks/stripe

# The CLI will print a webhook signing secret (whsec_...)
# Use this as STRIPE_WEBHOOK_SECRET in your .env.local
```

Now when you test payments in your local app, Stripe sends the webhook to the CLI, which forwards it to your local server. You can see every event in real time:

```bash
# In another terminal, trigger a test event
stripe trigger payment_intent.succeeded

# Or trigger a full subscription lifecycle
stripe trigger customer.subscription.created
stripe trigger invoice.paid
stripe trigger customer.subscription.updated
stripe trigger customer.subscription.deleted
```

**This is essential for development.** You cannot properly test payment flows without testing webhooks. And you cannot test webhooks without the Stripe CLI (or a tunnel like ngrok, but the CLI is better because it also lets you trigger events manually).

### 9.4 Testing Edge Cases

Here is a checklist of scenarios you must test before going live:

```
Payment Flow:
[ ] Successful payment (4242 card)
[ ] Declined card (insufficient funds)
[ ] Declined card (generic decline)
[ ] Expired card
[ ] 3D Secure (full authentication flow)
[ ] 3D Secure (authentication fails)
[ ] User cancels the Payment Sheet
[ ] User closes the app during payment
[ ] Network drops during payment confirmation
[ ] Double-tap on the pay button (idempotency)

Subscriptions:
[ ] New subscription creation
[ ] Subscription renewal (advance time in Stripe test clock)
[ ] Payment failure on renewal
[ ] Subscription upgrade (plan change)
[ ] Subscription downgrade
[ ] Subscription cancellation (at period end)
[ ] Subscription reactivation
[ ] Grace period behavior
[ ] Expired subscription access revocation

Webhooks:
[ ] Webhook receives and processes payment_intent.succeeded
[ ] Webhook handles duplicate events (idempotency)
[ ] Webhook rejects invalid signatures
[ ] Webhook returns 200 for unhandled event types
[ ] Webhook handles partial refunds
[ ] Webhook handles full refunds

Apple Pay / Google Pay:
[ ] Wallet payment succeeds (real device only for Apple Pay)
[ ] Wallet payment declined
[ ] Device without wallet configured (button should not appear)
```

### 9.5 Integration Tests with Mock Stripe

For CI, you do not want to hit the real Stripe API. Use a mock:

```typescript
// __tests__/payments/webhook.test.ts
import { POST } from '@/app/api/webhooks/stripe/route';
import { NextRequest } from 'next/server';
import Stripe from 'stripe';

// Mock the Stripe SDK
jest.mock('@/lib/stripe/server', () => ({
  stripe: {
    webhooks: {
      constructEvent: jest.fn(),
    },
  },
}));

import { stripe } from '@/lib/stripe/server';

describe('Stripe Webhook Handler', () => {
  const mockConstructEvent = stripe.webhooks.constructEvent as jest.Mock;

  it('handles payment_intent.succeeded', async () => {
    const mockEvent: Partial<Stripe.Event> = {
      id: 'evt_test_123',
      type: 'payment_intent.succeeded',
      data: {
        object: {
          id: 'pi_test_123',
          amount: 2999,
          currency: 'usd',
          status: 'succeeded',
          metadata: {
            orderId: 'order_123',
          },
        } as unknown as Stripe.PaymentIntent,
        previous_attributes: undefined,
      },
    };

    mockConstructEvent.mockReturnValue(mockEvent);

    const request = new NextRequest('http://localhost:3000/api/webhooks/stripe', {
      method: 'POST',
      body: JSON.stringify(mockEvent),
      headers: {
        'stripe-signature': 'test_signature',
      },
    });

    const response = await POST(request);
    expect(response.status).toBe(200);

    const body = await response.json();
    expect(body.received).toBe(true);
  });

  it('rejects invalid signatures', async () => {
    mockConstructEvent.mockImplementation(() => {
      throw new Error('Invalid signature');
    });

    const request = new NextRequest('http://localhost:3000/api/webhooks/stripe', {
      method: 'POST',
      body: 'invalid body',
      headers: {
        'stripe-signature': 'bad_signature',
      },
    });

    const response = await POST(request);
    expect(response.status).toBe(400);
  });

  it('returns 200 for unhandled event types', async () => {
    const mockEvent: Partial<Stripe.Event> = {
      id: 'evt_test_456',
      type: 'some.unknown.event' as any,
      data: { object: {} as any, previous_attributes: undefined },
    };

    mockConstructEvent.mockReturnValue(mockEvent);

    const request = new NextRequest('http://localhost:3000/api/webhooks/stripe', {
      method: 'POST',
      body: JSON.stringify(mockEvent),
      headers: {
        'stripe-signature': 'test_signature',
      },
    });

    const response = await POST(request);
    expect(response.status).toBe(200);
  });
});
```

### 9.6 Stripe Test Clocks

For subscription testing, Stripe provides Test Clocks -- a way to simulate the passage of time. This lets you test subscription renewals, trial expirations, and dunning flows without waiting days or weeks.

```typescript
// Create a test clock and attach a customer to it
const testClock = await stripe.testHelpers.testClocks.create({
  frozen_time: Math.floor(Date.now() / 1000),
});

const customer = await stripe.customers.create({
  test_clock: testClock.id,
});

// Create a subscription for this customer
const subscription = await stripe.subscriptions.create({
  customer: customer.id,
  items: [{ price: 'price_xxx' }],
});

// Advance time by 1 month (subscription should renew)
await stripe.testHelpers.testClocks.advance(testClock.id, {
  frozen_time: Math.floor(Date.now() / 1000) + 30 * 24 * 60 * 60,
});

// Check the subscription -- it should have a new invoice
```

Test Clocks are available in the Stripe Dashboard too. Go to Developers > Test Clocks. This is extremely valuable for testing subscription lifecycle without writing code.

---

## 10. REFUNDS AND DISPUTES

### 10.1 Issuing Refunds Programmatically

```typescript
// app/api/refunds/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { stripe } from '@/lib/stripe/server';

export async function POST(request: NextRequest) {
  try {
    const { paymentIntentId, amount, reason } = await request.json();

    // Full refund
    if (!amount) {
      const refund = await stripe.refunds.create({
        payment_intent: paymentIntentId,
        reason: reason || 'requested_by_customer',
        // 'requested_by_customer' | 'duplicate' | 'fraudulent'
      });

      return NextResponse.json({
        refundId: refund.id,
        status: refund.status,
        amount: refund.amount,
      });
    }

    // Partial refund
    const refund = await stripe.refunds.create({
      payment_intent: paymentIntentId,
      amount, // Amount in cents to refund
      reason: reason || 'requested_by_customer',
    });

    return NextResponse.json({
      refundId: refund.id,
      status: refund.status,
      amount: refund.amount,
    });
  } catch (error) {
    if (error instanceof Stripe.errors.StripeError) {
      return NextResponse.json(
        { error: error.message },
        { status: error.statusCode || 500 }
      );
    }
    return NextResponse.json(
      { error: 'Refund failed' },
      { status: 500 }
    );
  }
}
```

**Refund gotchas:**

1. Refunds take 5-10 business days to appear on the customer's statement. Tell your support team this.
2. Stripe's processing fee is not refunded. If you charged $100 (Stripe took $3.20 in fees), and you refund the full $100, you are out $103.20 total -- the $100 refund plus the $3.20 fee.
3. You can issue multiple partial refunds on the same charge, up to the original amount.
4. Refunds fail if the charge is too old (typically 180 days, varies by card network).

### 10.2 Handling Chargebacks / Disputes

A dispute (chargeback) happens when a customer contacts their bank to reverse a charge. This is different from a refund -- you did not initiate it, the customer's bank did.

Disputes are expensive. Stripe charges a $15 dispute fee (refunded if you win). And you lose the disputed amount until the dispute is resolved.

When a dispute is filed, Stripe sends a `charge.dispute.created` webhook:

```typescript
// In your webhook handler
case 'charge.dispute.created':
  await handleDispute(event.data.object as Stripe.Dispute);
  break;

case 'charge.dispute.closed':
  await handleDisputeClosed(event.data.object as Stripe.Dispute);
  break;

// ---

async function handleDispute(dispute: Stripe.Dispute) {
  console.error(`DISPUTE CREATED: ${dispute.id} for charge ${dispute.charge}`);
  console.error(`Amount: $${(dispute.amount / 100).toFixed(2)}`);
  console.error(`Reason: ${dispute.reason}`);

  // Alert your team immediately
  // await sendSlackAlert({
  //   channel: '#payments',
  //   text: `Dispute received: $${(dispute.amount / 100).toFixed(2)} - ${dispute.reason}`,
  // });

  // You have ~7 days to respond with evidence
}

async function handleDisputeClosed(dispute: Stripe.Dispute) {
  if (dispute.status === 'won') {
    console.log(`Dispute ${dispute.id} WON. Funds returned.`);
  } else {
    console.log(`Dispute ${dispute.id} LOST. Funds forfeited.`);
  }
}
```

### 10.3 Dispute Evidence Submission

To fight a dispute, you need to submit evidence. Stripe provides a structured way to do this:

```typescript
// Submit evidence for a dispute
async function submitDisputeEvidence(disputeId: string) {
  await stripe.disputes.update(disputeId, {
    evidence: {
      // Customer information
      customer_name: 'John Doe',
      customer_email_address: 'john@example.com',

      // Proof of delivery / service
      product_description: 'Premium subscription - monthly',
      service_date: '2026-03-15',

      // Activity logs showing the customer used the service
      access_activity_log: `
        2026-03-15 10:30 - User logged in
        2026-03-15 10:31 - User accessed premium feature X
        2026-03-16 14:22 - User accessed premium feature Y
        2026-03-20 09:15 - User logged in
        2026-03-20 09:16 - User accessed premium feature Z
      `,

      // Communication with customer
      customer_communication: 'file_upload_id', // Upload correspondence

      // Your cancellation/refund policy
      cancellation_policy: 'Subscriptions can be canceled at any time from the account settings page.',
      cancellation_policy_disclosure: 'Cancellation policy is displayed on the signup page and linked in the footer.',
      refund_policy: 'Full refunds within 14 days. Pro-rated refunds after 14 days.',
      refund_policy_disclosure: 'Refund policy is displayed during checkout and in the terms of service.',
    },
    submit: true, // Submit the evidence (you can update without submitting first)
  });
}
```

**Tips for winning disputes:**

1. Keep detailed logs of user activity. If you can prove the customer used your service after the charge, you have a strong case.
2. Send receipts and confirmation emails for every charge. These serve as evidence.
3. Have clear cancellation and refund policies, and make sure they are visible during signup.
4. Respond quickly. You typically have 7-21 days depending on the card network.
5. For subscription disputes, show that the customer was notified about each charge (via email).

---

## 11. COMMON MISTAKES

I have reviewed payment integrations for dozens of teams. These are the mistakes I see most often, ranked by how much damage they cause:

### Mistake 1: Creating PaymentIntents on the Client

```typescript
// NEVER DO THIS
// The client should NEVER have access to the Stripe secret key
const stripe = new Stripe('sk_live_...'); // SECRET KEY IN CLIENT CODE!!!
const paymentIntent = await stripe.paymentIntents.create({
  amount: 2999,
  currency: 'usd',
});
```

This means your secret key is in the client bundle. Anyone can extract it. With your secret key, they can:
- Issue refunds to themselves
- Create charges to arbitrary cards
- Read all your customer data
- Delete your products and prices

**Always create PaymentIntents on the server.** The client only receives the `client_secret`.

### Mistake 2: Trusting Client-Provided Amounts

```typescript
// BAD: The client tells the server how much to charge
app.post('/api/create-payment', async (req, res) => {
  const { amount } = req.body; // User can modify this!
  const paymentIntent = await stripe.paymentIntents.create({
    amount, // User sends amount: 1 (one cent)
    currency: 'usd',
  });
  res.json({ clientSecret: paymentIntent.client_secret });
});
```

```typescript
// GOOD: The server calculates the amount from the order
app.post('/api/create-payment', async (req, res) => {
  const { orderId } = req.body;
  const order = await db.order.findUnique({ where: { id: orderId } });

  // Calculate amount from order items on the server
  const amount = order.items.reduce(
    (sum, item) => sum + item.price * item.quantity,
    0
  );

  const paymentIntent = await stripe.paymentIntents.create({
    amount,
    currency: 'usd',
    metadata: { orderId },
  });

  res.json({ clientSecret: paymentIntent.client_secret });
});
```

### Mistake 3: Not Verifying Webhook Signatures

```typescript
// BAD: Accepts any POST request as a valid webhook
app.post('/api/webhooks/stripe', async (req, res) => {
  const event = req.body; // No verification! Anyone can send fake events
  if (event.type === 'payment_intent.succeeded') {
    await markOrderAsPaid(event.data.object.metadata.orderId);
  }
  res.json({ received: true });
});
```

An attacker discovers your webhook URL and sends:
```json
{
  "type": "payment_intent.succeeded",
  "data": {
    "object": {
      "metadata": { "orderId": "their_order_id" }
    }
  }
}
```

Congratulations, they just got free stuff. Always verify webhook signatures.

### Mistake 4: Not Handling Idempotency

```typescript
// BAD: No idempotency check
async function handlePaymentSuccess(paymentIntent) {
  // If Stripe sends this event twice (which it will, sometimes),
  // the customer gets charged once but receives two orders
  await createOrder(paymentIntent.metadata.userId, paymentIntent.amount);
  await sendConfirmationEmail(paymentIntent.metadata.userId);
}
```

Always check if the event has already been processed before taking action. See Section 4.4 for patterns.

### Mistake 5: Hardcoding Prices

```typescript
// BAD: Price is hardcoded in the client
const MONTHLY_PRICE = 29.99;

// What happens when you want to change the price?
// You have to deploy a new version of the app.
// What about users on old versions?
// What about different prices for different countries?
```

```typescript
// GOOD: Fetch prices from Stripe
const prices = await stripe.prices.list({
  product: 'prod_xxx',
  active: true,
});
```

Use Stripe Products and Prices. Change prices in the Dashboard. Your app fetches the current prices dynamically.

### Mistake 6: Not Testing the Full Flow

"I tested the happy path with 4242 4242 4242 4242 and it works!" is not sufficient testing. Test:
- Declined cards
- 3D Secure flows
- Webhook processing
- Network interruptions
- User cancellation mid-flow
- Subscription renewals
- Refunds

See Section 9.4 for the full checklist.

### Mistake 7: Ignoring Stripe Webhook Failures

Set up monitoring for your webhook endpoint. If webhooks start failing:
- Orders will not be fulfilled
- Subscriptions will not be activated
- Refunds will not be recorded

Add the webhook endpoint to your monitoring (Sentry, Datadog, etc.) and set up alerts for failure rates.

### Mistake 8: Not Using Stripe's Idempotency Keys for API Calls

When your server creates a PaymentIntent, the API call might timeout. If you retry without an idempotency key, you might create two PaymentIntents:

```typescript
// BAD: No idempotency key -- retry creates a duplicate
const paymentIntent = await stripe.paymentIntents.create({
  amount: 2999,
  currency: 'usd',
});

// GOOD: Idempotency key ensures retry returns the same result
const paymentIntent = await stripe.paymentIntents.create(
  {
    amount: 2999,
    currency: 'usd',
  },
  {
    idempotencyKey: `order_${orderId}_payment`, // Unique per logical operation
  }
);
```

Stripe's idempotency keys are valid for 24 hours. If you send the same key twice, Stripe returns the result of the first call instead of creating a duplicate.

### Mistake 9: Storing Subscription Status Only Locally

```typescript
// BAD: Only checking subscription status via client-side RevenueCat
if (revenueCatCustomerInfo.entitlements.active['pro']) {
  showProFeatures();
}
// What if the user modifies the local cache? What about web access?
```

Always verify subscription status on the server for sensitive operations. The client-side check is fine for UI gating (showing/hiding premium features), but for API endpoints that serve premium data, check on the server:

```typescript
// GOOD: Server-side subscription check
async function requireProSubscription(userId: string) {
  const subscription = await db.subscription.findFirst({
    where: {
      userId,
      status: { in: ['active', 'trialing', 'past_due'] },
    },
  });

  if (!subscription) {
    throw new Error('Pro subscription required');
  }

  return subscription;
}
```

### Mistake 10: Forgetting About Currency

```typescript
// BAD: Assumes USD everywhere
const price = '$' + (amount / 100).toFixed(2);

// GOOD: Format based on currency
function formatPrice(amount: number, currency: string): string {
  return new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency: currency.toUpperCase(),
  }).format(amount / 100);
}

formatPrice(2999, 'usd'); // "$29.99"
formatPrice(2999, 'eur'); // "EUR29.99" or "29,99 EUR" depending on locale
formatPrice(2999, 'jpy'); // "JP¥2,999" (JPY has no decimal places!)
```

Some currencies (JPY, KRW) do not use decimal places. $29.99 USD is 2999 cents, but 2999 JPY is 2999 yen (not 29.99 yen). Stripe handles this in its API, but your display code needs to handle it too.

---

## 12. PUTTING IT ALL TOGETHER

### 12.1 Architecture Checklist

Before you ship payments, verify each item:

```
Server Setup:
[x] Stripe secret key is in environment variables, NOT in code
[x] Stripe SDK initialized with pinned API version
[x] PaymentIntent creation endpoint requires authentication
[x] Amount is calculated server-side, never from client input
[x] Idempotency keys used for critical Stripe API calls

Client Setup:
[x] StripeProvider wraps the app with publishable key
[x] Payment Sheet OR Stripe Elements for card input (no custom inputs)
[x] Error handling for declined cards, network failures, cancellation
[x] Loading states during payment processing
[x] Success screen after payment (but order fulfillment depends on webhook)

Webhooks:
[x] Webhook endpoint registered in Stripe Dashboard
[x] Signature verification on every webhook request
[x] Idempotent event handling (safe to process same event twice)
[x] All relevant events handled (payment_intent.succeeded, charge.refunded, etc.)
[x] Returns 200 for unhandled event types (to prevent retries)
[x] Error monitoring on webhook failures

Security:
[x] Secret key never in client code
[x] HTTPS on all endpoints
[x] No card data logged, stored, or transmitted through your server
[x] Webhook secret stored in environment variables
[x] Raw body used for webhook signature verification

Testing:
[x] All test card scenarios verified
[x] Webhook flow tested with Stripe CLI
[x] Subscription lifecycle tested with Test Clocks
[x] Edge cases tested (network failure, 3D Secure, declined cards)
[x] CI tests with mocked Stripe
```

### 12.2 The Complete Example: E-Commerce Checkout

Here is how all the pieces fit together for a typical e-commerce flow:

```
1. User adds items to cart (client state)
2. User taps "Checkout" → app calls server to create order
3. Server creates order in database (status: "pending")
4. Server creates Stripe PaymentIntent with order ID in metadata
5. Server returns client_secret to the app
6. App initializes Payment Sheet with client_secret
7. User selects payment method (card, Apple Pay, Google Pay)
8. Stripe SDK sends card data directly to Stripe
9. Stripe processes payment
10. Payment Sheet shows success
11. App navigates to confirmation screen
12. Meanwhile: Stripe sends payment_intent.succeeded webhook
13. Server receives webhook, verifies signature
14. Server updates order status to "paid"
15. Server triggers fulfillment (email, shipping, etc.)
```

Steps 12-15 are the critical path. Even if the app crashes at step 10, the webhook at step 12 ensures the order is fulfilled. This is why webhooks are not optional.

### 12.3 Revenue Architecture Decision Tree

```
Are you selling digital goods/services consumed in the app?
├── YES → You must use Apple IAP / Google Play Billing
│         └── Use RevenueCat as abstraction layer
│             └── Server validates receipts
│             └── Accept the 30% commission (or 15% after year 1)
│
└── NO → You can use Stripe
    │
    ├── Do you need a custom checkout UI?
    │   ├── YES (Mobile) → @stripe/stripe-react-native with CardField
    │   ├── YES (Web) → Stripe Elements (PaymentElement)
    │   └── NO → Payment Sheet (mobile) or Stripe Checkout (web)
    │
    ├── One-time payments?
    │   └── PaymentIntents + webhook
    │
    └── Recurring payments?
        └── Stripe Billing (Subscriptions)
            └── Subscription webhooks for lifecycle management
```

---

## Summary

Payment code is different from every other code you write. It deals with real money, real trust, and real consequences. The key principles:

1. **Never handle raw card data.** Use Stripe's SDKs and pre-built UIs. They are PCI compliant. Your custom code is not.

2. **Server-side authority.** Create PaymentIntents on the server. Calculate amounts on the server. Verify payments on the server via webhooks. The client is for UX, not business logic.

3. **Webhooks are the source of truth.** The client-side success callback is for showing a nice UI. The webhook is for updating your database and triggering fulfillment.

4. **Idempotency everywhere.** Webhooks can be delivered multiple times. API calls can timeout and be retried. Every handler must be safe to run twice.

5. **Test relentlessly.** Happy path testing is not enough. Test declined cards, 3D Secure, network failures, duplicate events, and subscription lifecycle.

6. **Know the IAP rules.** If you are selling digital goods in a mobile app, you must use Apple IAP / Google Play Billing. If not, use Stripe. Get this wrong and your app gets rejected.

Get payment code right and your users trust you with their money. Get it wrong and they will never come back. There is no middle ground.

---

> Next chapter: [Chapter 25] continues with additional operational concerns for production applications.

> Previous chapter: [Chapter 23] covered CI/CD pipelines and deployment automation.
