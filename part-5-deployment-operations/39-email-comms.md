<!--
  CHAPTER: 39
  TITLE: Email, Transactional Comms & In-App Messaging
  PART: V — Deployment & Operations
  PREREQS: Chapters 27, 34
  KEY_TOPICS: Resend, React Email, SendGrid, email templates, transactional email, SMTP, email deliverability, in-app messaging, Knock, Novu, notification center, email verification, welcome emails
  DIFFICULTY: Intermediate
  UPDATED: 2026-04-07
-->

# Chapter 39: Email, Transactional Comms & In-App Messaging

> **Part V — Deployment & Operations** | Prerequisites: Chapters 27, 34 | Difficulty: Intermediate

> "Email is the cockroach of the internet. It survived social media, Slack, push notifications, and every 'email killer' ever launched. Your app will send email. Make it good."

---

<details>
<summary><strong>TL;DR</strong></summary>

- Transactional emails (welcome, password reset, order confirmation) are not marketing emails -- they are part of your product experience and deserve the same quality as your UI
- Use Resend for sending and React Email for templates; the DX is leagues ahead of legacy providers, you write JSX instead of fighting table-based HTML from 2003
- Domain verification (SPF, DKIM, DMARC) is not optional -- skip it and your emails land in spam; set up all three DNS records before sending a single production email
- For multi-channel notifications (email + push + in-app + SMS), use Knock or Novu instead of building your own notification router; the state management alone will eat weeks
- Always queue emails from Server Actions -- never send inline; a failed email send should not block your user's checkout flow

</details>

## The Email Problem

Here is a story I have seen play out at least a dozen times.

A team builds a beautiful app. The UI is polished, the animations are smooth, the API is fast. Then someone says "we need to send a welcome email." And the engineering team -- the same team that agonized over button border-radius and loading skeleton shimmer effects -- ships an email that looks like it was written in Notepad in 2004.

White background. Black text. No logo. The subject line says "Welcome to AppName." The body says "Your account has been created." There is a link at the bottom that says "Click here to verify your email." It is unstyled. It looks like a phishing attempt.

This is the first thing your user sees after signing up. The first touchpoint outside your app. And it looks like spam.

The reason this happens is that email is genuinely hard. Not conceptually hard -- the concept is simple. You render some HTML, you send it through an API. But the execution is hard because email HTML is stuck in 1999. No flexbox. No grid. No modern CSS. You are writing table-based layouts with inline styles, and you are testing across 90+ email clients that all render differently.

This chapter fixes that. We are going to build a proper transactional email system with modern tools that make it almost -- almost -- as pleasant as building web UIs. We are going to cover the providers, the templates, the deliverability, and the multi-channel notification story. By the end, your emails will be as polished as your app.

### In This Chapter
- Transactional email vs marketing email -- why you need both, why this chapter covers transactional
- Resend -- the modern email API built by the React Email team
- React Email -- email templates as React components
- Email patterns -- welcome, verification, password reset, order confirmation, digest, alert
- SendGrid, Postmark, AWS SES -- when each makes sense
- Email deliverability -- SPF, DKIM, DMARC, sender reputation, spam filters
- In-app notification center -- Knock, Novu, multi-channel notifications
- Server-side implementation -- Next.js Server Actions, queuing, rate limiting
- Complete example -- Resend + React Email in a real Next.js app

### Related Chapters
- [Ch 27: Server Architecture] -- API routes and Server Actions that trigger emails
- [Ch 34: Background Jobs & Queues] -- queuing emails for reliability
- [Ch 22: Security & Data Protection] -- protecting API keys and user data in emails

---

## 1. TRANSACTIONAL EMAIL

### 1.1 What Is Transactional Email?

Let me be very precise about what we are building in this chapter. There are two categories of email your app sends:

| | Transactional Email | Marketing Email |
|---|---|---|
| **Triggered by** | User action (signed up, bought something, reset password) | Business decision (campaign, newsletter, promotion) |
| **Expected by** | Yes -- the user just did something and expects a response | Maybe -- the user opted in at some point |
| **Content** | Specific to that user and their action | Same content to many users |
| **Timing** | Immediately after the trigger | Scheduled for optimal engagement |
| **CAN-SPAM** | Exempt from unsubscribe requirements (mostly) | Must include unsubscribe link |
| **Examples** | Welcome, password reset, receipt, shipping, alerts | Newsletter, product updates, promotions |
| **Provider** | Resend, Postmark, SES | Mailchimp, ConvertKit, Loops |

This chapter covers transactional email. If you need marketing email, use a dedicated marketing platform -- they handle list management, unsubscribe, A/B testing, and analytics that you do not want to build yourself.

### 1.2 The Emails Every App Needs

Here is the minimum set of transactional emails most apps need, roughly in the order you will build them:

1. **Email verification** -- "Verify your email address" with a time-limited link
2. **Welcome email** -- Sent after verification, introduces the app, has a CTA
3. **Password reset** -- "Reset your password" with a time-limited link
4. **Password changed confirmation** -- "Your password was just changed" (security notice)
5. **Login from new device** -- "Someone logged in from Chrome on Windows" (security notice)
6. **Order / action confirmation** -- "Your order #1234 is confirmed" or equivalent
7. **Receipt** -- Itemized receipt after payment
8. **Weekly digest** -- "Here's what happened this week" (optional but high-value)
9. **Alert / notification** -- "Your deploy failed" or "Someone commented on your post"

You will not build all of these on day one. Start with verification and password reset (because auth requires them), then add the others as your product matures.

### 1.3 The Technical Flow

Every transactional email follows this flow:

```
User action (sign up, purchase, etc.)
    │
    ▼
Server Action / API route handles the action
    │
    ▼
Queue an email job (DO NOT send inline)
    │
    ▼
Worker picks up the job
    │
    ▼
Render React Email template to HTML string
    │
    ▼
Send via Resend API
    │
    ▼
Resend delivers via verified domain
    │
    ▼
Track delivery status (delivered, bounced, complained)
```

The key insight here: **never send email inline in the request/response cycle.** If you send an email inside a Server Action and the email API is slow or down, your user's action fails. The user clicked "Place Order" and got a timeout because SendGrid had a hiccup. That is unacceptable.

Queue the email. Let the primary action succeed. Send the email asynchronously. If the email fails, retry. The user should never know or care about your email delivery pipeline.

---

## 2. RESEND (RECOMMENDED)

### 2.1 Why Resend

I am going to recommend Resend the same way I recommended Stripe in the payments chapter: with conviction and without pretending to be neutral.

Resend was built by the same team that built React Email. They understood that the two biggest problems with transactional email were (1) the sending API was stuck in the SOAP era, and (2) the templating was stuck in the table-layout era. So they built solutions for both.

Here is why Resend wins for our stack:

1. **React Email integration** -- First-class support for React Email templates. You write JSX, they render it to email-safe HTML. No other provider has this.
2. **Modern API** -- Clean REST API, excellent TypeScript SDK, no XML, no SOAP, no legacy baggage.
3. **Domain verification** -- Simple DNS record setup with clear documentation.
4. **Webhook support** -- Get notified about delivery, bounce, complaint events.
5. **Reasonable pricing** -- Free tier (100 emails/day), then $20/month for 5,000 emails. More than enough for most startups.
6. **Fast delivery** -- Emails arrive in seconds, not minutes. This matters for verification and password reset emails.

The alternatives are fine. SendGrid has been around forever and works at massive scale. Postmark is obsessively focused on deliverability. AWS SES is the cheapest at volume. We will cover all of them. But if you are building a new app with Next.js and React, Resend is the obvious starting point.

### 2.2 Setting Up Resend

**Step 1: Create an account and get an API key**

```bash
# Install the Resend SDK
npm install resend
```

Go to [resend.com](https://resend.com), create an account, and generate an API key from the dashboard. Add it to your environment:

```bash
# .env.local
RESEND_API_KEY=re_123456789abcdef
```

**Step 2: Send your first email**

```typescript
// lib/email.ts
import { Resend } from 'resend';

if (!process.env.RESEND_API_KEY) {
  throw new Error('RESEND_API_KEY environment variable is required');
}

export const resend = new Resend(process.env.RESEND_API_KEY);
```

```typescript
// Test it — a simple API route
// app/api/test-email/route.ts
import { resend } from '@/lib/email';
import { NextResponse } from 'next/server';

export async function POST() {
  const { data, error } = await resend.emails.send({
    from: 'Acme <onboarding@resend.dev>', // Use resend.dev for testing
    to: ['your@email.com'],
    subject: 'Hello from Resend',
    html: '<p>This is a test email.</p>',
  });

  if (error) {
    return NextResponse.json({ error }, { status: 500 });
  }

  return NextResponse.json({ data });
}
```

That is it. You can send an email with 10 lines of code. No SMTP configuration. No connection pooling. No transport setup.

### 2.3 Domain Verification

Before you send from your own domain (not `@resend.dev`), you need to verify it. This is where most teams get confused, so let me be very clear about what is happening and why.

When you send an email "from" `notifications@yourapp.com`, the receiving email server (Gmail, Outlook, etc.) checks whether you are actually authorized to send email for `yourapp.com`. If you are not, the email goes to spam or gets rejected entirely.

You prove authorization by adding DNS records. There are three:

**SPF (Sender Policy Framework)** -- "These servers are allowed to send email for this domain."

```
Type: TXT
Name: @ (or yourapp.com)
Value: v=spf1 include:resend.com ~all
```

**DKIM (DomainKeys Identified Mail)** -- "This email was actually sent by us and hasn't been tampered with." Resend gives you a CNAME record to add:

```
Type: CNAME
Name: resend._domainkey
Value: (provided by Resend dashboard — a long string)
```

**DMARC (Domain-based Message Authentication, Reporting & Conformance)** -- "Here's what to do if SPF or DKIM fails."

```
Type: TXT
Name: _dmarc
Value: v=DMARC1; p=quarantine; rua=mailto:dmarc@yourapp.com
```

**The DMARC policy options:**

| Policy | Meaning | When to use |
|--------|---------|-------------|
| `p=none` | Do nothing, just report | When you first set up -- monitor before enforcing |
| `p=quarantine` | Put in spam folder | After you have confirmed legitimate email is passing |
| `p=reject` | Reject entirely | When you are fully confident in your setup |

Start with `p=none` while you verify everything works, then move to `p=quarantine`, then eventually `p=reject`.

### 2.4 Resend API Deep Dive

Here is the full shape of what you can do with the Resend API:

```typescript
// Send a basic email
const { data, error } = await resend.emails.send({
  from: 'Acme <notifications@yourapp.com>',
  to: ['user@example.com'],
  subject: 'Your order is confirmed',
  html: '<h1>Order Confirmed</h1><p>Thanks for your purchase.</p>',
});

// Send with React Email component (we'll cover this in section 3)
import { OrderConfirmation } from '@/emails/order-confirmation';
import { render } from '@react-email/render';

const html = await render(OrderConfirmation({ orderId: '1234', total: 99.99 }));

const { data, error } = await resend.emails.send({
  from: 'Acme <notifications@yourapp.com>',
  to: ['user@example.com'],
  subject: 'Order #1234 confirmed',
  html,
});

// Send to multiple recipients
const { data, error } = await resend.emails.send({
  from: 'Acme <notifications@yourapp.com>',
  to: ['user1@example.com', 'user2@example.com'],
  subject: 'Team update',
  html: '<p>New deployment succeeded.</p>',
});

// CC and BCC
const { data, error } = await resend.emails.send({
  from: 'Acme <notifications@yourapp.com>',
  to: ['user@example.com'],
  cc: ['manager@example.com'],
  bcc: ['audit@yourapp.com'],
  subject: 'Invoice #5678',
  html: '<p>Your invoice is attached.</p>',
});

// Reply-to address (useful when "from" is a no-reply but you want replies to go somewhere)
const { data, error } = await resend.emails.send({
  from: 'Acme <no-reply@yourapp.com>',
  replyTo: 'support@yourapp.com',
  to: ['user@example.com'],
  subject: 'Your support ticket #42',
  html: '<p>We received your request.</p>',
});

// Attachments
const { data, error } = await resend.emails.send({
  from: 'Acme <billing@yourapp.com>',
  to: ['user@example.com'],
  subject: 'Invoice #5678',
  html: '<p>Your invoice is attached.</p>',
  attachments: [
    {
      filename: 'invoice-5678.pdf',
      content: pdfBuffer, // Buffer or base64 string
    },
  ],
});

// Tags for tracking / filtering in Resend dashboard
const { data, error } = await resend.emails.send({
  from: 'Acme <notifications@yourapp.com>',
  to: ['user@example.com'],
  subject: 'Your weekly digest',
  html: digestHtml,
  tags: [
    { name: 'category', value: 'digest' },
    { name: 'user_id', value: '12345' },
  ],
});

// Schedule for later
const { data, error } = await resend.emails.send({
  from: 'Acme <notifications@yourapp.com>',
  to: ['user@example.com'],
  subject: 'Your trial ends tomorrow',
  html: trialEndingHtml,
  scheduledAt: '2026-04-08T09:00:00Z', // ISO 8601
});
```

### 2.5 Resend Webhooks

Resend can notify your app about email events via webhooks:

```typescript
// app/api/webhooks/resend/route.ts
import { NextRequest, NextResponse } from 'next/server';
import crypto from 'crypto';

const RESEND_WEBHOOK_SECRET = process.env.RESEND_WEBHOOK_SECRET!;

function verifySignature(payload: string, signature: string): boolean {
  const expected = crypto
    .createHmac('sha256', RESEND_WEBHOOK_SECRET)
    .update(payload)
    .digest('hex');
  return crypto.timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(expected)
  );
}

export async function POST(request: NextRequest) {
  const body = await request.text();
  const signature = request.headers.get('resend-signature') ?? '';

  if (!verifySignature(body, signature)) {
    return NextResponse.json({ error: 'Invalid signature' }, { status: 401 });
  }

  const event = JSON.parse(body);

  switch (event.type) {
    case 'email.sent':
      // Email accepted by Resend
      console.log(`Email ${event.data.email_id} sent`);
      break;

    case 'email.delivered':
      // Email delivered to recipient's mail server
      await db.emailLog.update({
        where: { emailId: event.data.email_id },
        data: { status: 'delivered', deliveredAt: new Date() },
      });
      break;

    case 'email.bounced':
      // Email bounced — mark the address as invalid
      await db.user.update({
        where: { email: event.data.to },
        data: { emailBounced: true },
      });
      break;

    case 'email.complained':
      // User marked as spam — stop sending to them immediately
      await db.user.update({
        where: { email: event.data.to },
        data: { emailOptOut: true },
      });
      break;

    case 'email.opened':
      // Email was opened (if tracking is enabled)
      break;

    case 'email.clicked':
      // A link in the email was clicked
      break;
  }

  return NextResponse.json({ received: true });
}
```

**Important**: Always handle `email.bounced` and `email.complained`. If you keep sending to bounced addresses, your sender reputation drops. If you ignore complaints, email providers will start blocking your domain entirely.

---

## 3. REACT EMAIL

### 3.1 The Email HTML Problem

Before React Email, building email templates was genuinely painful. Here is why:

Email clients do not render HTML like browsers. Not even close. There is no standard. Gmail strips `<style>` tags in the `<head>` (sometimes). Outlook uses Microsoft Word's rendering engine (yes, really -- Word. In 2026.). Apple Mail is relatively modern. Yahoo does unpredictable things.

What this means in practice:

| CSS Feature | Browser Support | Email Support |
|------------|-----------------|---------------|
| Flexbox | Universal | Almost nowhere |
| CSS Grid | Universal | Nowhere |
| `<div>` for layout | Universal | Inconsistent |
| `<table>` for layout | Deprecated for layout | The only reliable option |
| External stylesheets | Universal | Stripped by most clients |
| `<style>` in `<head>` | Universal | Stripped by Gmail (sometimes) |
| Inline styles | Universal | The only reliable option |
| `border-radius` | Universal | Works in most, not Outlook |
| `background-image` | Universal | Unreliable |
| `media queries` | Universal | Some support (Apple Mail, iOS) |
| `max-width` | Universal | Inconsistent |
| CSS variables | Universal | Nowhere |
| `gap` | Universal | Nowhere |

You are building layouts with `<table>`, `<tr>`, and `<td>`. You are using inline styles on every element. You are targeting a rendering engine from 2007. This is the reality.

### 3.2 What React Email Does

React Email gives you a set of React components that abstract away the table-based hell:

```bash
npm install @react-email/components react-email
```

Instead of writing this:

```html
<!-- The old way: table-based email HTML -->
<table role="presentation" border="0" cellpadding="0" cellspacing="0" width="100%">
  <tr>
    <td align="center" style="padding: 20px 0;">
      <table role="presentation" border="0" cellpadding="0" cellspacing="0" width="600">
        <tr>
          <td style="background-color: #ffffff; padding: 40px 30px;">
            <table role="presentation" border="0" cellpadding="0" cellspacing="0" width="100%">
              <tr>
                <td style="font-family: Arial, sans-serif; font-size: 24px; font-weight: bold; color: #333333;">
                  Welcome to Acme
                </td>
              </tr>
              <tr>
                <td style="padding-top: 20px; font-family: Arial, sans-serif; font-size: 16px; color: #666666; line-height: 24px;">
                  Thanks for signing up. We're glad to have you.
                </td>
              </tr>
              <tr>
                <td style="padding-top: 30px;">
                  <table role="presentation" border="0" cellpadding="0" cellspacing="0">
                    <tr>
                      <td style="background-color: #000000; border-radius: 4px; padding: 12px 24px;">
                        <a href="https://yourapp.com/dashboard" style="color: #ffffff; text-decoration: none; font-family: Arial, sans-serif; font-size: 16px; font-weight: bold;">
                          Go to Dashboard
                        </a>
                      </td>
                    </tr>
                  </table>
                </td>
              </tr>
            </table>
          </td>
        </tr>
      </table>
    </td>
  </tr>
</table>
```

You write this:

```tsx
// emails/welcome.tsx
import {
  Body,
  Button,
  Container,
  Head,
  Heading,
  Html,
  Preview,
  Section,
  Text,
} from '@react-email/components';

interface WelcomeEmailProps {
  userName: string;
  dashboardUrl: string;
}

export function WelcomeEmail({ userName, dashboardUrl }: WelcomeEmailProps) {
  return (
    <Html>
      <Head />
      <Preview>Welcome to Acme, {userName}!</Preview>
      <Body style={main}>
        <Container style={container}>
          <Heading style={heading}>Welcome to Acme</Heading>
          <Text style={text}>
            Hey {userName}, thanks for signing up. We are glad to have you.
          </Text>
          <Section style={buttonSection}>
            <Button style={button} href={dashboardUrl}>
              Go to Dashboard
            </Button>
          </Section>
        </Container>
      </Body>
    </Html>
  );
}

// Styles as plain objects — React Email converts them to inline styles
const main = {
  backgroundColor: '#f6f9fc',
  fontFamily: '-apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif',
};

const container = {
  backgroundColor: '#ffffff',
  margin: '0 auto',
  padding: '40px 30px',
  maxWidth: '600px',
  borderRadius: '8px',
};

const heading = {
  fontSize: '24px',
  fontWeight: '700' as const,
  color: '#1a1a1a',
  margin: '0 0 20px',
};

const text = {
  fontSize: '16px',
  color: '#666666',
  lineHeight: '24px',
  margin: '0 0 30px',
};

const buttonSection = {
  textAlign: 'center' as const,
};

const button = {
  backgroundColor: '#000000',
  color: '#ffffff',
  padding: '12px 24px',
  borderRadius: '4px',
  fontSize: '16px',
  fontWeight: '600' as const,
  textDecoration: 'none',
};
```

Same result. One-tenth the pain. And it is type-safe -- your email templates have props, just like your React components.

### 3.3 The React Email Component Library

Here are the components React Email gives you and when to use each:

```tsx
import {
  Html,        // Wraps everything, sets doctype
  Head,        // <head> tag — meta tags, fonts
  Preview,     // Preview text (shown in inbox before opening)
  Body,        // <body> tag
  Container,   // Centered content area (max-width wrapper)
  Section,     // Groups related content (<table> under the hood)
  Row,         // Horizontal layout (<tr> under the hood)
  Column,      // Columns within a row (<td> under the hood)
  Heading,     // <h1> through <h6>
  Text,        // <p> tag
  Link,        // <a> tag
  Button,      // Styled <a> that looks like a button
  Img,         // <img> tag
  Hr,          // Horizontal rule
  Font,        // Custom font loading (limited email client support)
  CodeBlock,   // Syntax-highlighted code
  CodeInline,  // Inline code
  Markdown,    // Render markdown content
  Tailwind,    // Use Tailwind classes in email (yes, really)
} from '@react-email/components';
```

**The `<Preview>` component** is especially important. It controls the preview text shown in the inbox list -- the gray text after the subject line:

```
Subject: Your order is confirmed
Preview: Order #1234 — 3 items, $147.00. Arriving by April 10.   <-- This is <Preview>
```

Without `<Preview>`, email clients will grab the first text content in your email body, which is often "View in browser" or your company name -- useless information.

### 3.4 Preview in Browser

React Email comes with a dev server that lets you preview your templates in the browser:

```json
// package.json
{
  "scripts": {
    "email:dev": "email dev --dir emails --port 3001"
  }
}
```

```bash
npm run email:dev
```

This opens a browser at `localhost:3001` where you can:
- See all your email templates listed
- Preview each one with different props
- View the rendered HTML source
- Send a test email to yourself
- Toggle between desktop and mobile preview widths

This is genuinely game-changing. Before React Email, testing email templates meant sending yourself an email every time you changed a margin. Now you get hot-reload.

### 3.5 Rendering to HTML

When it is time to actually send the email, you render the React component to an HTML string:

```typescript
import { render } from '@react-email/render';
import { WelcomeEmail } from '@/emails/welcome';

// Render to HTML string
const html = await render(WelcomeEmail({ userName: 'Alice', dashboardUrl: 'https://app.acme.com/dashboard' }));

// Render to plain text (for email clients that don't support HTML)
const text = await render(WelcomeEmail({ userName: 'Alice', dashboardUrl: 'https://app.acme.com/dashboard' }), {
  plainText: true,
});

// Send both versions
await resend.emails.send({
  from: 'Acme <notifications@acme.com>',
  to: ['alice@example.com'],
  subject: 'Welcome to Acme!',
  html,
  text, // Fallback for plain text email clients
});
```

Always send both `html` and `text` versions. Some email clients (especially corporate ones) display plain text by default. Some users prefer it. And spam filters look more favorably on emails that include both.

### 3.6 Using Tailwind in Email

React Email has a `<Tailwind>` component that lets you use Tailwind utility classes in your email templates. This is borderline magical:

```tsx
import { Html, Head, Body, Container, Text, Button, Tailwind } from '@react-email/components';

export function WelcomeEmail({ userName }: { userName: string }) {
  return (
    <Html>
      <Head />
      <Tailwind
        config={{
          theme: {
            extend: {
              colors: {
                brand: '#007bff',
              },
            },
          },
        }}
      >
        <Body className="bg-gray-100 font-sans">
          <Container className="bg-white mx-auto p-10 rounded-lg max-w-xl">
            <Text className="text-2xl font-bold text-gray-900 mb-4">
              Welcome, {userName}!
            </Text>
            <Text className="text-base text-gray-600 leading-relaxed mb-8">
              Thanks for joining Acme. We are excited to have you on board.
            </Text>
            <Button
              className="bg-brand text-white px-6 py-3 rounded font-semibold"
              href="https://app.acme.com/dashboard"
            >
              Go to Dashboard
            </Button>
          </Container>
        </Body>
      </Tailwind>
    </Html>
  );
}
```

React Email converts these Tailwind classes to inline styles at render time. The email client never sees a class name -- it sees inline `style` attributes. This means Tailwind works everywhere email works.

---

## 4. EMAIL PATTERNS

Here are the email templates every app needs. Each one is a complete, production-ready React Email component.

### 4.1 Email Verification

The most important email you will ever send. If this lands in spam, users cannot complete signup.

```tsx
// emails/verify-email.tsx
import {
  Body,
  Button,
  Container,
  Head,
  Heading,
  Hr,
  Html,
  Img,
  Preview,
  Section,
  Text,
} from '@react-email/components';

interface VerifyEmailProps {
  userName: string;
  verificationUrl: string;
  expiresInHours: number;
}

export function VerifyEmail({
  userName,
  verificationUrl,
  expiresInHours = 24,
}: VerifyEmailProps) {
  return (
    <Html>
      <Head />
      <Preview>Verify your email address for Acme</Preview>
      <Body style={main}>
        <Container style={container}>
          <Img
            src="https://yourapp.com/logo.png"
            width="48"
            height="48"
            alt="Acme"
            style={logo}
          />
          <Heading style={heading}>Verify your email address</Heading>
          <Text style={text}>
            Hey {userName}, thanks for signing up for Acme. Please verify your
            email address by clicking the button below.
          </Text>
          <Section style={buttonContainer}>
            <Button style={button} href={verificationUrl}>
              Verify Email Address
            </Button>
          </Section>
          <Text style={secondaryText}>
            This link expires in {expiresInHours} hours. If you did not create an
            account with Acme, you can safely ignore this email.
          </Text>
          <Hr style={hr} />
          <Text style={footerText}>
            If the button does not work, copy and paste this URL into your browser:
          </Text>
          <Text style={linkText}>{verificationUrl}</Text>
        </Container>
      </Body>
    </Html>
  );
}

const main = {
  backgroundColor: '#f6f9fc',
  fontFamily: '-apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif',
  padding: '40px 0',
};

const container = {
  backgroundColor: '#ffffff',
  margin: '0 auto',
  padding: '40px',
  maxWidth: '560px',
  borderRadius: '8px',
  border: '1px solid #e5e7eb',
};

const logo = {
  margin: '0 0 24px',
};

const heading = {
  fontSize: '22px',
  fontWeight: '700' as const,
  color: '#111827',
  margin: '0 0 16px',
};

const text = {
  fontSize: '15px',
  color: '#374151',
  lineHeight: '24px',
  margin: '0 0 24px',
};

const buttonContainer = {
  textAlign: 'center' as const,
  margin: '0 0 24px',
};

const button = {
  backgroundColor: '#111827',
  color: '#ffffff',
  padding: '12px 32px',
  borderRadius: '6px',
  fontSize: '15px',
  fontWeight: '600' as const,
  textDecoration: 'none',
};

const secondaryText = {
  fontSize: '13px',
  color: '#6b7280',
  lineHeight: '20px',
  margin: '0 0 24px',
};

const hr = {
  borderColor: '#e5e7eb',
  margin: '0 0 24px',
};

const footerText = {
  fontSize: '12px',
  color: '#9ca3af',
  lineHeight: '20px',
  margin: '0',
};

const linkText = {
  fontSize: '12px',
  color: '#3b82f6',
  lineHeight: '20px',
  wordBreak: 'break-all' as const,
};
```

**Key design decisions:**
- Always include the raw URL as text below the button. Some email clients block styled links.
- Always say "If you did not create an account, ignore this email." This prevents support tickets from confused people.
- Keep the expiration time visible. Users need to know they have 24 hours.

### 4.2 Welcome Email

Sent after verification succeeds. This is your chance to make a good first impression.

```tsx
// emails/welcome.tsx
import {
  Body,
  Button,
  Container,
  Head,
  Heading,
  Hr,
  Html,
  Img,
  Link,
  Preview,
  Section,
  Text,
} from '@react-email/components';

interface WelcomeEmailProps {
  userName: string;
  dashboardUrl: string;
}

export function WelcomeEmail({ userName, dashboardUrl }: WelcomeEmailProps) {
  return (
    <Html>
      <Head />
      <Preview>Welcome to Acme -- here is how to get started</Preview>
      <Body style={main}>
        <Container style={container}>
          <Img
            src="https://yourapp.com/logo.png"
            width="48"
            height="48"
            alt="Acme"
            style={logo}
          />
          <Heading style={heading}>Welcome to Acme!</Heading>
          <Text style={text}>
            Hey {userName}, your account is all set up. Here are three things
            you can do right now:
          </Text>

          <Section style={stepSection}>
            <Text style={stepNumber}>1</Text>
            <Text style={stepText}>
              <strong>Complete your profile</strong> -- Add a photo and bio so
              your team knows who you are.
            </Text>
          </Section>

          <Section style={stepSection}>
            <Text style={stepNumber}>2</Text>
            <Text style={stepText}>
              <strong>Create your first project</strong> -- It takes 30 seconds
              and you can invite teammates.
            </Text>
          </Section>

          <Section style={stepSection}>
            <Text style={stepNumber}>3</Text>
            <Text style={stepText}>
              <strong>Explore the docs</strong> -- We have guides for every
              feature at{' '}
              <Link href="https://docs.yourapp.com" style={link}>
                docs.yourapp.com
              </Link>
              .
            </Text>
          </Section>

          <Section style={buttonContainer}>
            <Button style={button} href={dashboardUrl}>
              Go to Your Dashboard
            </Button>
          </Section>

          <Hr style={hr} />
          <Text style={footerText}>
            Questions? Just reply to this email -- it goes straight to our
            support team.
          </Text>
        </Container>
      </Body>
    </Html>
  );
}

const main = {
  backgroundColor: '#f6f9fc',
  fontFamily: '-apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif',
  padding: '40px 0',
};

const container = {
  backgroundColor: '#ffffff',
  margin: '0 auto',
  padding: '40px',
  maxWidth: '560px',
  borderRadius: '8px',
  border: '1px solid #e5e7eb',
};

const logo = { margin: '0 0 24px' };

const heading = {
  fontSize: '24px',
  fontWeight: '700' as const,
  color: '#111827',
  margin: '0 0 16px',
};

const text = {
  fontSize: '15px',
  color: '#374151',
  lineHeight: '24px',
  margin: '0 0 24px',
};

const stepSection = {
  marginBottom: '16px',
  paddingLeft: '8px',
};

const stepNumber = {
  display: 'inline-block' as const,
  backgroundColor: '#111827',
  color: '#ffffff',
  borderRadius: '50%',
  width: '24px',
  height: '24px',
  textAlign: 'center' as const,
  lineHeight: '24px',
  fontSize: '13px',
  fontWeight: '700' as const,
  marginRight: '12px',
  verticalAlign: 'top' as const,
};

const stepText = {
  display: 'inline-block' as const,
  fontSize: '15px',
  color: '#374151',
  lineHeight: '24px',
  verticalAlign: 'top' as const,
  width: '85%',
  margin: '0',
};

const link = {
  color: '#3b82f6',
  textDecoration: 'underline',
};

const buttonContainer = {
  textAlign: 'center' as const,
  margin: '32px 0 24px',
};

const button = {
  backgroundColor: '#111827',
  color: '#ffffff',
  padding: '12px 32px',
  borderRadius: '6px',
  fontSize: '15px',
  fontWeight: '600' as const,
  textDecoration: 'none',
};

const hr = { borderColor: '#e5e7eb', margin: '0 0 24px' };

const footerText = {
  fontSize: '13px',
  color: '#6b7280',
  lineHeight: '20px',
  margin: '0',
};
```

**Key design decisions:**
- Three concrete next steps. Do not leave the user staring at a blank email.
- "Reply to this email" -- set a `replyTo` address that goes to support.
- Keep it short. Welcome emails with 10 paragraphs do not get read.

### 4.3 Password Reset

Security-critical. Must be clear, trustworthy, and include anti-phishing cues.

```tsx
// emails/password-reset.tsx
import {
  Body,
  Button,
  Container,
  Head,
  Heading,
  Hr,
  Html,
  Img,
  Preview,
  Section,
  Text,
} from '@react-email/components';

interface PasswordResetProps {
  userName: string;
  resetUrl: string;
  ipAddress: string;
  userAgent: string;
}

export function PasswordResetEmail({
  userName,
  resetUrl,
  ipAddress,
  userAgent,
}: PasswordResetProps) {
  return (
    <Html>
      <Head />
      <Preview>Reset your Acme password</Preview>
      <Body style={main}>
        <Container style={container}>
          <Img
            src="https://yourapp.com/logo.png"
            width="48"
            height="48"
            alt="Acme"
            style={{ margin: '0 0 24px' }}
          />
          <Heading style={heading}>Reset your password</Heading>
          <Text style={text}>
            Hey {userName}, we received a request to reset your password. Click
            the button below to choose a new one.
          </Text>
          <Section style={{ textAlign: 'center' as const, margin: '0 0 24px' }}>
            <Button style={button} href={resetUrl}>
              Reset Password
            </Button>
          </Section>
          <Text style={secondaryText}>
            This link expires in 1 hour. If you did not request a password reset,
            you can safely ignore this email. Your password will not be changed.
          </Text>

          <Hr style={hr} />

          <Text style={metaText}>
            This request was made from:
          </Text>
          <Text style={metaText}>
            IP: {ipAddress}
          </Text>
          <Text style={metaText}>
            Device: {userAgent}
          </Text>

          <Hr style={hr} />
          <Text style={footerText}>
            If the button does not work, copy and paste this URL:
          </Text>
          <Text style={linkText}>{resetUrl}</Text>
        </Container>
      </Body>
    </Html>
  );
}

const main = {
  backgroundColor: '#f6f9fc',
  fontFamily: '-apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif',
  padding: '40px 0',
};

const container = {
  backgroundColor: '#ffffff',
  margin: '0 auto',
  padding: '40px',
  maxWidth: '560px',
  borderRadius: '8px',
  border: '1px solid #e5e7eb',
};

const heading = {
  fontSize: '22px',
  fontWeight: '700' as const,
  color: '#111827',
  margin: '0 0 16px',
};

const text = {
  fontSize: '15px',
  color: '#374151',
  lineHeight: '24px',
  margin: '0 0 24px',
};

const button = {
  backgroundColor: '#dc2626',
  color: '#ffffff',
  padding: '12px 32px',
  borderRadius: '6px',
  fontSize: '15px',
  fontWeight: '600' as const,
  textDecoration: 'none',
};

const secondaryText = {
  fontSize: '13px',
  color: '#6b7280',
  lineHeight: '20px',
  margin: '0 0 24px',
};

const hr = { borderColor: '#e5e7eb', margin: '0 0 16px' };

const metaText = {
  fontSize: '12px',
  color: '#9ca3af',
  lineHeight: '18px',
  margin: '0',
  fontFamily: 'monospace',
};

const footerText = {
  fontSize: '12px',
  color: '#9ca3af',
  lineHeight: '20px',
  margin: '16px 0 0',
};

const linkText = {
  fontSize: '12px',
  color: '#3b82f6',
  lineHeight: '20px',
  wordBreak: 'break-all' as const,
};
```

**Key design decisions:**
- Red button (not black/blue) -- signals "this is a security action."
- Include IP address and device info -- helps users identify if the request is legitimate.
- Shorter expiration (1 hour, not 24) -- password reset links should be short-lived.

### 4.4 Order Confirmation

```tsx
// emails/order-confirmation.tsx
import {
  Body,
  Column,
  Container,
  Head,
  Heading,
  Hr,
  Html,
  Img,
  Preview,
  Row,
  Section,
  Text,
} from '@react-email/components';

interface OrderItem {
  name: string;
  quantity: number;
  price: number;
  imageUrl: string;
}

interface OrderConfirmationProps {
  userName: string;
  orderId: string;
  orderDate: string;
  items: OrderItem[];
  subtotal: number;
  shipping: number;
  tax: number;
  total: number;
  shippingAddress: string;
  estimatedDelivery: string;
}

export function OrderConfirmationEmail({
  userName,
  orderId,
  orderDate,
  items,
  subtotal,
  shipping,
  tax,
  total,
  shippingAddress,
  estimatedDelivery,
}: OrderConfirmationProps) {
  const formatCurrency = (amount: number) =>
    `$${amount.toFixed(2)}`;

  return (
    <Html>
      <Head />
      <Preview>
        Order #{orderId} confirmed -- {items.length} item{items.length > 1 ? 's' : ''},{' '}
        {formatCurrency(total)}
      </Preview>
      <Body style={main}>
        <Container style={container}>
          <Img
            src="https://yourapp.com/logo.png"
            width="48"
            height="48"
            alt="Acme Store"
            style={{ margin: '0 0 24px' }}
          />
          <Heading style={heading}>Order confirmed!</Heading>
          <Text style={text}>
            Hey {userName}, thanks for your order. Here is your receipt.
          </Text>

          <Section style={orderMeta}>
            <Row>
              <Column>
                <Text style={metaLabel}>Order number</Text>
                <Text style={metaValue}>#{orderId}</Text>
              </Column>
              <Column>
                <Text style={metaLabel}>Order date</Text>
                <Text style={metaValue}>{orderDate}</Text>
              </Column>
              <Column>
                <Text style={metaLabel}>Estimated delivery</Text>
                <Text style={metaValue}>{estimatedDelivery}</Text>
              </Column>
            </Row>
          </Section>

          <Hr style={hr} />

          {/* Line items */}
          {items.map((item, index) => (
            <Section key={index} style={itemRow}>
              <Row>
                <Column style={{ width: '64px' }}>
                  <Img
                    src={item.imageUrl}
                    width="56"
                    height="56"
                    alt={item.name}
                    style={itemImage}
                  />
                </Column>
                <Column style={{ paddingLeft: '12px' }}>
                  <Text style={itemName}>{item.name}</Text>
                  <Text style={itemQuantity}>Qty: {item.quantity}</Text>
                </Column>
                <Column style={{ textAlign: 'right' as const }}>
                  <Text style={itemPrice}>
                    {formatCurrency(item.price * item.quantity)}
                  </Text>
                </Column>
              </Row>
            </Section>
          ))}

          <Hr style={hr} />

          {/* Totals */}
          <Section style={totalsSection}>
            <Row>
              <Column><Text style={totalLabel}>Subtotal</Text></Column>
              <Column style={{ textAlign: 'right' as const }}>
                <Text style={totalValue}>{formatCurrency(subtotal)}</Text>
              </Column>
            </Row>
            <Row>
              <Column><Text style={totalLabel}>Shipping</Text></Column>
              <Column style={{ textAlign: 'right' as const }}>
                <Text style={totalValue}>
                  {shipping === 0 ? 'Free' : formatCurrency(shipping)}
                </Text>
              </Column>
            </Row>
            <Row>
              <Column><Text style={totalLabel}>Tax</Text></Column>
              <Column style={{ textAlign: 'right' as const }}>
                <Text style={totalValue}>{formatCurrency(tax)}</Text>
              </Column>
            </Row>
            <Hr style={{ ...hr, margin: '12px 0' }} />
            <Row>
              <Column><Text style={grandTotalLabel}>Total</Text></Column>
              <Column style={{ textAlign: 'right' as const }}>
                <Text style={grandTotalValue}>{formatCurrency(total)}</Text>
              </Column>
            </Row>
          </Section>

          <Hr style={hr} />

          <Text style={metaLabel}>Shipping to</Text>
          <Text style={addressText}>{shippingAddress}</Text>

          <Hr style={hr} />
          <Text style={footerText}>
            Questions about your order? Reply to this email or contact us at
            support@yourapp.com.
          </Text>
        </Container>
      </Body>
    </Html>
  );
}

const main = {
  backgroundColor: '#f6f9fc',
  fontFamily: '-apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif',
  padding: '40px 0',
};

const container = {
  backgroundColor: '#ffffff',
  margin: '0 auto',
  padding: '40px',
  maxWidth: '560px',
  borderRadius: '8px',
  border: '1px solid #e5e7eb',
};

const heading = {
  fontSize: '24px',
  fontWeight: '700' as const,
  color: '#111827',
  margin: '0 0 16px',
};

const text = {
  fontSize: '15px',
  color: '#374151',
  lineHeight: '24px',
  margin: '0 0 24px',
};

const orderMeta = {
  backgroundColor: '#f9fafb',
  padding: '16px',
  borderRadius: '6px',
  marginBottom: '24px',
};

const metaLabel = {
  fontSize: '12px',
  color: '#6b7280',
  textTransform: 'uppercase' as const,
  letterSpacing: '0.5px',
  margin: '0 0 4px',
  fontWeight: '600' as const,
};

const metaValue = {
  fontSize: '14px',
  color: '#111827',
  fontWeight: '600' as const,
  margin: '0',
};

const hr = { borderColor: '#e5e7eb', margin: '0 0 24px' };

const itemRow = { marginBottom: '16px' };

const itemImage = {
  borderRadius: '6px',
  objectFit: 'cover' as const,
};

const itemName = {
  fontSize: '14px',
  color: '#111827',
  fontWeight: '600' as const,
  margin: '0 0 4px',
};

const itemQuantity = {
  fontSize: '13px',
  color: '#6b7280',
  margin: '0',
};

const itemPrice = {
  fontSize: '14px',
  color: '#111827',
  fontWeight: '600' as const,
  margin: '0',
};

const totalsSection = { marginBottom: '24px' };

const totalLabel = {
  fontSize: '14px',
  color: '#6b7280',
  margin: '4px 0',
};

const totalValue = {
  fontSize: '14px',
  color: '#374151',
  margin: '4px 0',
};

const grandTotalLabel = {
  fontSize: '16px',
  color: '#111827',
  fontWeight: '700' as const,
  margin: '4px 0',
};

const grandTotalValue = {
  fontSize: '16px',
  color: '#111827',
  fontWeight: '700' as const,
  margin: '4px 0',
};

const addressText = {
  fontSize: '14px',
  color: '#374151',
  lineHeight: '22px',
  margin: '0 0 24px',
  whiteSpace: 'pre-line' as const,
};

const footerText = {
  fontSize: '13px',
  color: '#6b7280',
  lineHeight: '20px',
  margin: '0',
};
```

### 4.5 Weekly Digest

```tsx
// emails/weekly-digest.tsx
import {
  Body,
  Button,
  Column,
  Container,
  Head,
  Heading,
  Hr,
  Html,
  Link,
  Preview,
  Row,
  Section,
  Text,
} from '@react-email/components';

interface DigestItem {
  title: string;
  description: string;
  url: string;
  metric?: string;
}

interface WeeklyDigestProps {
  userName: string;
  weekRange: string;          // "March 31 — April 6, 2026"
  highlights: DigestItem[];
  stats: { label: string; value: string }[];
  dashboardUrl: string;
  unsubscribeUrl: string;
}

export function WeeklyDigestEmail({
  userName,
  weekRange,
  highlights,
  stats,
  dashboardUrl,
  unsubscribeUrl,
}: WeeklyDigestProps) {
  return (
    <Html>
      <Head />
      <Preview>Your weekly Acme digest for {weekRange}</Preview>
      <Body style={main}>
        <Container style={container}>
          <Text style={preheading}>Weekly Digest</Text>
          <Heading style={heading}>{weekRange}</Heading>
          <Text style={text}>
            Hey {userName}, here is what happened this week.
          </Text>

          {/* Stats row */}
          <Section style={statsSection}>
            <Row>
              {stats.map((stat, i) => (
                <Column key={i} style={statCol}>
                  <Text style={statValue}>{stat.value}</Text>
                  <Text style={statLabel}>{stat.label}</Text>
                </Column>
              ))}
            </Row>
          </Section>

          <Hr style={hr} />

          {/* Highlights */}
          <Heading as="h3" style={subheading}>Highlights</Heading>
          {highlights.map((item, i) => (
            <Section key={i} style={highlightItem}>
              <Link href={item.url} style={highlightTitle}>
                {item.title}
              </Link>
              <Text style={highlightDesc}>{item.description}</Text>
              {item.metric && (
                <Text style={highlightMetric}>{item.metric}</Text>
              )}
            </Section>
          ))}

          <Section style={{ textAlign: 'center' as const, margin: '32px 0 0' }}>
            <Button style={button} href={dashboardUrl}>
              View Full Dashboard
            </Button>
          </Section>

          <Hr style={hr} />
          <Text style={footerText}>
            You are receiving this because you are subscribed to weekly digests.{' '}
            <Link href={unsubscribeUrl} style={unsubLink}>
              Unsubscribe
            </Link>
          </Text>
        </Container>
      </Body>
    </Html>
  );
}

const main = {
  backgroundColor: '#f6f9fc',
  fontFamily: '-apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif',
  padding: '40px 0',
};

const container = {
  backgroundColor: '#ffffff',
  margin: '0 auto',
  padding: '40px',
  maxWidth: '560px',
  borderRadius: '8px',
  border: '1px solid #e5e7eb',
};

const preheading = {
  fontSize: '12px',
  color: '#6b7280',
  textTransform: 'uppercase' as const,
  letterSpacing: '1px',
  fontWeight: '600' as const,
  margin: '0 0 8px',
};

const heading = {
  fontSize: '22px',
  fontWeight: '700' as const,
  color: '#111827',
  margin: '0 0 16px',
};

const subheading = {
  fontSize: '16px',
  fontWeight: '700' as const,
  color: '#111827',
  margin: '0 0 16px',
};

const text = {
  fontSize: '15px',
  color: '#374151',
  lineHeight: '24px',
  margin: '0 0 24px',
};

const statsSection = {
  backgroundColor: '#f9fafb',
  padding: '20px',
  borderRadius: '6px',
  marginBottom: '24px',
};

const statCol = { textAlign: 'center' as const };

const statValue = {
  fontSize: '24px',
  fontWeight: '700' as const,
  color: '#111827',
  margin: '0 0 4px',
};

const statLabel = {
  fontSize: '12px',
  color: '#6b7280',
  textTransform: 'uppercase' as const,
  letterSpacing: '0.5px',
  margin: '0',
};

const hr = { borderColor: '#e5e7eb', margin: '24px 0' };

const highlightItem = { marginBottom: '20px' };

const highlightTitle = {
  fontSize: '15px',
  fontWeight: '600' as const,
  color: '#3b82f6',
  textDecoration: 'none',
};

const highlightDesc = {
  fontSize: '14px',
  color: '#374151',
  lineHeight: '22px',
  margin: '4px 0 0',
};

const highlightMetric = {
  fontSize: '13px',
  color: '#6b7280',
  margin: '4px 0 0',
};

const button = {
  backgroundColor: '#111827',
  color: '#ffffff',
  padding: '12px 32px',
  borderRadius: '6px',
  fontSize: '15px',
  fontWeight: '600' as const,
  textDecoration: 'none',
};

const footerText = {
  fontSize: '12px',
  color: '#9ca3af',
  lineHeight: '20px',
  margin: '0',
};

const unsubLink = {
  color: '#9ca3af',
  textDecoration: 'underline',
};
```

**Key design decisions:**
- Digest emails MUST have an unsubscribe link. Unlike pure transactional emails, digests are periodic and user-initiated opt-out is expected.
- Stats at the top -- give people a reason to care before they scroll.
- Keep highlights to 3-5 items. More than that and nobody reads them.

### 4.6 Alert / Notification Email

```tsx
// emails/alert.tsx
import {
  Body,
  Button,
  Container,
  Head,
  Heading,
  Html,
  Preview,
  Section,
  Text,
} from '@react-email/components';

type AlertSeverity = 'info' | 'warning' | 'error' | 'success';

interface AlertEmailProps {
  userName: string;
  title: string;
  message: string;
  severity: AlertSeverity;
  actionUrl: string;
  actionLabel: string;
  timestamp: string;
}

const severityConfig: Record<AlertSeverity, { color: string; bgColor: string; emoji: string }> = {
  info: { color: '#1d4ed8', bgColor: '#eff6ff', emoji: 'ℹ️' },
  warning: { color: '#b45309', bgColor: '#fffbeb', emoji: '⚠️' },
  error: { color: '#dc2626', bgColor: '#fef2f2', emoji: '🚨' },
  success: { color: '#059669', bgColor: '#ecfdf5', emoji: '✅' },
};

export function AlertEmail({
  userName,
  title,
  message,
  severity,
  actionUrl,
  actionLabel,
  timestamp,
}: AlertEmailProps) {
  const config = severityConfig[severity];

  return (
    <Html>
      <Head />
      <Preview>{config.emoji} {title}</Preview>
      <Body style={main}>
        <Container style={container}>
          <Section style={{
            backgroundColor: config.bgColor,
            padding: '16px 20px',
            borderRadius: '6px',
            borderLeft: `4px solid ${config.color}`,
            marginBottom: '24px',
          }}>
            <Heading style={{ ...alertTitle, color: config.color }}>
              {title}
            </Heading>
            <Text style={alertMessage}>{message}</Text>
            <Text style={alertTimestamp}>{timestamp}</Text>
          </Section>

          <Section style={{ textAlign: 'center' as const }}>
            <Button
              style={{ ...button, backgroundColor: config.color }}
              href={actionUrl}
            >
              {actionLabel}
            </Button>
          </Section>

          <Text style={footerText}>
            You are receiving this alert because of your notification
            preferences. Manage them in your account settings.
          </Text>
        </Container>
      </Body>
    </Html>
  );
}

const main = {
  backgroundColor: '#f6f9fc',
  fontFamily: '-apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif',
  padding: '40px 0',
};

const container = {
  backgroundColor: '#ffffff',
  margin: '0 auto',
  padding: '40px',
  maxWidth: '560px',
  borderRadius: '8px',
  border: '1px solid #e5e7eb',
};

const alertTitle = {
  fontSize: '18px',
  fontWeight: '700' as const,
  margin: '0 0 8px',
};

const alertMessage = {
  fontSize: '14px',
  color: '#374151',
  lineHeight: '22px',
  margin: '0 0 8px',
};

const alertTimestamp = {
  fontSize: '12px',
  color: '#9ca3af',
  margin: '0',
};

const button = {
  color: '#ffffff',
  padding: '12px 32px',
  borderRadius: '6px',
  fontSize: '15px',
  fontWeight: '600' as const,
  textDecoration: 'none',
};

const footerText = {
  fontSize: '12px',
  color: '#9ca3af',
  lineHeight: '20px',
  margin: '24px 0 0',
  textAlign: 'center' as const,
};
```

---

## 5. SENDGRID / POSTMARK / SES

### 5.1 When to Use Each

Resend is great. But it is not always the right choice. Here is when to consider the alternatives:

| Provider | Best For | Pricing | Key Strength | Key Weakness |
|----------|----------|---------|-------------|-------------|
| **Resend** | Startups, Next.js/React teams | $20/mo for 5K emails | DX, React Email integration | Newer, smaller scale track record |
| **SendGrid** (Twilio) | Scale, combined transactional + marketing | Free 100/day, then volume-based | Massive scale, marketing tools too | API is dated, DX is mediocre |
| **Postmark** | Maximum deliverability | $15/mo for 10K emails | Best-in-class deliverability | Strictly transactional (no marketing) |
| **AWS SES** | High volume, cost-sensitive | $0.10 per 1,000 emails | Cheapest at scale by far | DIY everything, worst DX |
| **Mailgun** | Developers who want SMTP | $35/mo for 50K emails | Good SMTP support, logs | Deliverability can be inconsistent |

### 5.2 SendGrid

SendGrid is the incumbent. It has been around since 2009. It handles billions of emails. If you need both transactional and marketing email from one provider, or if you are sending millions of emails per month, SendGrid makes sense.

```typescript
// Using SendGrid with @sendgrid/mail
import sgMail from '@sendgrid/mail';

sgMail.setApiKey(process.env.SENDGRID_API_KEY!);

export async function sendEmail({
  to,
  subject,
  html,
  text,
}: {
  to: string;
  subject: string;
  html: string;
  text?: string;
}) {
  const msg = {
    to,
    from: {
      email: 'notifications@yourapp.com',
      name: 'Acme',
    },
    subject,
    html,
    text: text ?? '',
  };

  try {
    await sgMail.send(msg);
  } catch (error: any) {
    console.error('SendGrid error:', error.response?.body);
    throw error;
  }
}
```

SendGrid also supports dynamic templates (created in their web UI) referenced by template ID:

```typescript
await sgMail.send({
  to: 'user@example.com',
  from: 'notifications@yourapp.com',
  templateId: 'd-abc123def456',
  dynamicTemplateData: {
    userName: 'Alice',
    orderId: '1234',
    total: '$99.99',
  },
});
```

The downside: those templates are edited in SendGrid's web-based WYSIWYG editor, which is clunky. You cannot version-control them. You cannot code-review them. They live outside your codebase. This is why I prefer React Email templates rendered locally and sent as raw HTML.

### 5.3 Postmark

Postmark's entire identity is deliverability. They refuse to send marketing email. They only allow transactional messages. This means their IP reputation is extremely clean -- your emails are less likely to land in spam.

```typescript
// Using Postmark with postmark SDK
import { ServerClient } from 'postmark';

const client = new ServerClient(process.env.POSTMARK_SERVER_TOKEN!);

export async function sendEmail({
  to,
  subject,
  html,
  text,
}: {
  to: string;
  subject: string;
  html: string;
  text?: string;
}) {
  await client.sendEmail({
    From: 'notifications@yourapp.com',
    To: to,
    Subject: subject,
    HtmlBody: html,
    TextBody: text ?? '',
    MessageStream: 'outbound',
  });
}
```

When to use Postmark over Resend:
- You are sending emails where deliverability is life-or-death (financial services, healthcare, legal)
- You want the most established track record for transactional email
- You do not need React Email integration (you have your own templates)

### 5.4 AWS SES

AWS Simple Email Service is the cheapest option at scale. $0.10 per 1,000 emails. If you are sending 10 million emails per month, that is $1,000 on SES versus $2,000+ on other providers.

```typescript
// Using AWS SES v3 SDK
import { SESv2Client, SendEmailCommand } from '@aws-sdk/client-sesv2';

const ses = new SESv2Client({ region: 'us-east-1' });

export async function sendEmail({
  to,
  subject,
  html,
  text,
}: {
  to: string;
  subject: string;
  html: string;
  text?: string;
}) {
  const command = new SendEmailCommand({
    FromEmailAddress: 'notifications@yourapp.com',
    Destination: {
      ToAddresses: [to],
    },
    Content: {
      Simple: {
        Subject: { Data: subject },
        Body: {
          Html: { Data: html },
          Text: { Data: text ?? '' },
        },
      },
    },
  });

  await ses.send(command);
}
```

The tradeoff: SES gives you nothing beyond raw sending. No dashboard analytics. No template management. No webhook delivery tracking (you set up SNS topics yourself). No bounce handling (you build it). No suppression list management (you build it). You are paying less money and more engineering time.

**My recommendation:** Start with Resend. If you outgrow it (sending 100K+ emails/month and cost matters), migrate to SES. The sending interface is just an HTTP call -- switching providers is a one-file change if you architect it right.

### 5.5 Provider Abstraction

Here is how to keep your email code provider-agnostic:

```typescript
// lib/email/types.ts
export interface EmailPayload {
  to: string | string[];
  subject: string;
  html: string;
  text?: string;
  from?: string;
  replyTo?: string;
  cc?: string[];
  bcc?: string[];
  attachments?: Array<{
    filename: string;
    content: Buffer | string;
  }>;
  tags?: Array<{ name: string; value: string }>;
}

export interface EmailProvider {
  send(payload: EmailPayload): Promise<{ id: string }>;
}
```

```typescript
// lib/email/providers/resend.ts
import { Resend } from 'resend';
import type { EmailPayload, EmailProvider } from '../types';

export class ResendProvider implements EmailProvider {
  private client: Resend;
  private defaultFrom: string;

  constructor(apiKey: string, defaultFrom: string) {
    this.client = new Resend(apiKey);
    this.defaultFrom = defaultFrom;
  }

  async send(payload: EmailPayload): Promise<{ id: string }> {
    const { data, error } = await this.client.emails.send({
      from: payload.from ?? this.defaultFrom,
      to: Array.isArray(payload.to) ? payload.to : [payload.to],
      subject: payload.subject,
      html: payload.html,
      text: payload.text,
      replyTo: payload.replyTo,
      cc: payload.cc,
      bcc: payload.bcc,
      attachments: payload.attachments,
      tags: payload.tags,
    });

    if (error) {
      throw new Error(`Resend error: ${error.message}`);
    }

    return { id: data!.id };
  }
}
```

```typescript
// lib/email/providers/ses.ts
import { SESv2Client, SendEmailCommand } from '@aws-sdk/client-sesv2';
import type { EmailPayload, EmailProvider } from '../types';

export class SESProvider implements EmailProvider {
  private client: SESv2Client;
  private defaultFrom: string;

  constructor(region: string, defaultFrom: string) {
    this.client = new SESv2Client({ region });
    this.defaultFrom = defaultFrom;
  }

  async send(payload: EmailPayload): Promise<{ id: string }> {
    const toAddresses = Array.isArray(payload.to) ? payload.to : [payload.to];

    const command = new SendEmailCommand({
      FromEmailAddress: payload.from ?? this.defaultFrom,
      Destination: {
        ToAddresses: toAddresses,
        CcAddresses: payload.cc,
        BccAddresses: payload.bcc,
      },
      Content: {
        Simple: {
          Subject: { Data: payload.subject },
          Body: {
            Html: { Data: payload.html },
            Text: { Data: payload.text ?? '' },
          },
        },
      },
      ReplyToAddresses: payload.replyTo ? [payload.replyTo] : undefined,
    });

    const response = await this.client.send(command);
    return { id: response.MessageId ?? 'unknown' };
  }
}
```

```typescript
// lib/email/index.ts
import type { EmailProvider } from './types';
import { ResendProvider } from './providers/resend';
import { SESProvider } from './providers/ses';

function createEmailProvider(): EmailProvider {
  const provider = process.env.EMAIL_PROVIDER ?? 'resend';

  switch (provider) {
    case 'resend':
      return new ResendProvider(
        process.env.RESEND_API_KEY!,
        'Acme <notifications@yourapp.com>'
      );
    case 'ses':
      return new SESProvider(
        process.env.AWS_REGION ?? 'us-east-1',
        'Acme <notifications@yourapp.com>'
      );
    default:
      throw new Error(`Unknown email provider: ${provider}`);
  }
}

export const emailProvider = createEmailProvider();
```

Now swapping providers is a one-line environment variable change. Your application code calls `emailProvider.send()` and does not know or care whether it is Resend, SES, or anything else.

---

## 6. EMAIL DELIVERABILITY

### 6.1 Why Your Emails Land in Spam

You built a beautiful email template. You set up Resend. You sent a test email. It landed in your inbox. Ship it.

Two weeks later, your support team is getting flooded with "I never got the verification email" tickets. Your emails are going to spam for 30% of users. What happened?

Email deliverability is not binary. It is not "emails work or they don't." It is a reputation system. Every email you send affects your reputation. And once your reputation drops, recovering it takes weeks.

Here are the factors:

### 6.2 Authentication Records (The Foundation)

We covered SPF, DKIM, and DMARC in the Resend section. Let me go deeper.

**SPF (Sender Policy Framework)**

SPF tells receiving servers which IP addresses are authorized to send email for your domain. Here is how it works:

```
v=spf1 include:resend.com include:_spf.google.com ~all
```

- `v=spf1` -- This is an SPF record
- `include:resend.com` -- Resend's servers can send for us
- `include:_spf.google.com` -- Google Workspace can send for us (if you use Gmail)
- `~all` -- Soft-fail anything else (treat as suspicious but don't reject)

Common mistakes:
- Having multiple SPF records (you can only have ONE per domain -- combine with `include:`)
- Using `-all` (hard fail) before you have verified everything works
- Forgetting to include your email hosting provider (Google Workspace, Microsoft 365)
- Exceeding 10 DNS lookups (SPF has a 10-lookup limit -- `include:` counts as a lookup)

**DKIM (DomainKeys Identified Mail)**

DKIM adds a cryptographic signature to your emails. The receiving server checks the signature against a public key in your DNS. If it matches, the email has not been tampered with and came from an authorized sender.

Your email provider (Resend, SendGrid, etc.) generates a key pair and gives you the public key as a DNS record. You do not need to understand the cryptography. You just need to add the record.

```
Type: CNAME
Name: resend._domainkey.yourapp.com
Value: (long string from Resend dashboard)
```

**DMARC (Domain-based Message Authentication, Reporting & Conformance)**

DMARC ties SPF and DKIM together and tells receiving servers what to do when authentication fails.

```
v=DMARC1; p=quarantine; rua=mailto:dmarc-reports@yourapp.com; pct=100
```

- `p=quarantine` -- Put unauthenticated emails in spam
- `rua=mailto:...` -- Send aggregate reports here (you will get XML reports showing who is sending email using your domain)
- `pct=100` -- Apply this policy to 100% of messages

**The rollout plan:**

1. Week 1: Set `p=none` with `rua=` reporting. Send test emails. Check reports.
2. Week 2-3: Review DMARC reports. Make sure all legitimate senders pass.
3. Week 4: Move to `p=quarantine` with `pct=25` (apply to 25% of traffic).
4. Week 5-6: Increase to `pct=50`, then `pct=100`.
5. Week 7+: Move to `p=reject` if you are confident.

### 6.3 Sender Reputation

Your sender reputation is like a credit score for email. It is based on:

| Factor | Impact | How to Maintain |
|--------|--------|-----------------|
| Bounce rate | High | Remove invalid addresses immediately |
| Spam complaints | Very high | Honor unsubscribes, only send relevant email |
| Sending volume consistency | Medium | Ramp up gradually, avoid sudden spikes |
| Engagement (opens, clicks) | Medium | Send relevant content, good subject lines |
| Spam trap hits | Very high | Never buy email lists, clean your list |
| Authentication (SPF/DKIM/DMARC) | High | Set up all three correctly |

**Bounce handling** is critical. There are two types:

- **Hard bounce** -- The address does not exist. Stop sending immediately. Never retry.
- **Soft bounce** -- Temporary issue (mailbox full, server down). Retry a few times, then stop.

```typescript
// Handle bounces in your webhook handler
case 'email.bounced':
  const { to, bounce_type } = event.data;

  if (bounce_type === 'hard') {
    // Permanently mark this address as invalid
    await db.user.update({
      where: { email: to },
      data: {
        emailBounced: true,
        emailBouncedAt: new Date(),
      },
    });
  }

  // Check your sending function to skip bounced addresses
  break;
```

```typescript
// In your sending logic, skip bounced addresses
export async function sendTransactionalEmail(userId: string, template: string, data: any) {
  const user = await db.user.findUnique({ where: { id: userId } });

  if (!user) return;
  if (user.emailBounced) {
    console.warn(`Skipping email to bounced address: ${user.id}`);
    return;
  }
  if (user.emailOptOut) {
    console.warn(`Skipping email to opted-out user: ${user.id}`);
    return;
  }

  // ... send the email
}
```

### 6.4 Dedicated IP vs Shared IP

Most email providers put you on a shared IP by default. Your emails are sent from the same IP addresses as thousands of other customers.

| | Shared IP | Dedicated IP |
|---|---|---|
| **Cost** | Included | Extra ($20-$50/month) |
| **Reputation** | Shared with other senders | Yours alone |
| **Warm-up** | Pre-warmed by provider | YOU must warm it up |
| **Best for** | < 50K emails/month | > 100K emails/month |
| **Risk** | Other senders can hurt your rep | Only you can hurt your rep |

**Dedicated IP warm-up** is a real process. You cannot buy a dedicated IP and send 100K emails on day one. The IP has no reputation. ISPs will throttle or block you.

Warm-up schedule (approximate):

| Day | Daily volume |
|-----|-------------|
| 1-3 | 50-100 |
| 4-7 | 200-500 |
| 8-14 | 500-2,000 |
| 15-21 | 2,000-10,000 |
| 22-30 | 10,000-50,000 |
| 30+ | Full volume |

Send to your most engaged users first (people who open and click). Their positive engagement signals build reputation for the new IP.

### 6.5 Content Best Practices

Beyond authentication and reputation, the content of your email matters:

**Things that trigger spam filters:**
- ALL CAPS subject lines
- Excessive exclamation marks (!!!)
- Phrases like "Act now!", "Limited time!", "Free!", "Click here!"
- Huge images with little text (image-to-text ratio)
- URL shorteners in links (bit.ly, etc.)
- Single-image emails with no text
- Missing unsubscribe link (for non-transactional email)
- Mismatched "From" name and domain

**Things that help deliverability:**
- Consistent "From" name and address
- Text-to-image ratio of at least 60/40
- Both HTML and plain text versions
- Proper HTML structure (doctype, head, body)
- Real content that matches the subject line
- Links to your own domain (not third-party tracking domains)

---

## 7. IN-APP NOTIFICATION CENTER

### 7.1 Beyond Email: Multi-Channel Notifications

At some point, email is not enough. Users want notifications where they already are:

- **In-app** -- A notification bell with unread count and a dropdown
- **Push** -- Mobile push notifications (iOS, Android)
- **Email** -- Still important for things the user needs to see even if they are not in the app
- **SMS** -- For urgent, time-sensitive notifications
- **Slack/Discord/Teams** -- For developer tools and B2B products

Building this yourself is tempting. It sounds simple. "Just send a push notification AND an email." But then you need:

- Per-user notification preferences ("email me about comments but not likes")
- Per-channel preferences ("push for urgent, email for digest")
- Deduplication ("don't send both push and email if the user already saw it in-app")
- Batching ("don't send 10 emails for 10 comments in 5 minutes -- send one digest")
- Read/unread state for in-app notifications
- Notification history
- Template management for each channel
- Delivery tracking and retry

This is a product, not a feature. If you build it yourself, it will take weeks and it will be full of edge cases.

### 7.2 Knock

Knock is a notification infrastructure platform. You define notification workflows, and Knock handles the multi-channel delivery, user preferences, batching, and in-app feed.

**Setting up Knock:**

```bash
npm install @knocklabs/node @knocklabs/react
```

```typescript
// lib/knock.ts (server-side)
import { Knock } from '@knocklabs/node';

export const knock = new Knock(process.env.KNOCK_API_KEY!);
```

**Triggering a notification:**

```typescript
// When a user gets a new comment
import { knock } from '@/lib/knock';

export async function notifyNewComment({
  commentId,
  commentAuthor,
  postOwner,
  postTitle,
  commentText,
}: {
  commentId: string;
  commentAuthor: { id: string; name: string; avatarUrl: string };
  postOwner: { id: string };
  postTitle: string;
  commentText: string;
}) {
  await knock.workflows.trigger('new-comment', {
    // Who triggered the notification
    actor: commentAuthor.id,
    // Who receives the notification
    recipients: [postOwner.id],
    // Data available in all templates
    data: {
      commentId,
      postTitle,
      commentText,
      commentAuthorName: commentAuthor.name,
      commentAuthorAvatar: commentAuthor.avatarUrl,
      commentUrl: `https://yourapp.com/posts/${commentId}`,
    },
  });
}
```

In the Knock dashboard, you define the "new-comment" workflow:

```
Workflow: new-comment
  │
  ├── Step 1: In-app notification (immediate)
  │   Template: "{{ commentAuthorName }} commented on {{ postTitle }}"
  │
  ├── Step 2: Wait 5 minutes (batch window)
  │   If user has not seen the in-app notification...
  │
  ├── Step 3: Push notification
  │   Template: "{{ commentAuthorName }}: {{ commentText | truncate: 50 }}"
  │
  ├── Step 4: Wait 1 hour
  │   If user has not engaged...
  │
  └── Step 5: Email
      Template: React Email component rendered to HTML
```

This workflow sends an in-app notification immediately, then escalates to push if the user has not seen it, then escalates to email. This is much smarter than blasting all channels simultaneously.

**In-app notification feed (React component):**

```tsx
// components/notification-feed.tsx
'use client';

import {
  KnockProvider,
  KnockFeedProvider,
  NotificationIconButton,
  NotificationFeedPopover,
} from '@knocklabs/react';

// Import Knock's default styles
import '@knocklabs/react/dist/index.css';

interface NotificationFeedProps {
  userId: string;
  userToken: string; // Generated server-side with Knock signing key
}

export function NotificationFeed({ userId, userToken }: NotificationFeedProps) {
  return (
    <KnockProvider
      apiKey={process.env.NEXT_PUBLIC_KNOCK_PUBLIC_API_KEY!}
      userId={userId}
      userToken={userToken}
    >
      <KnockFeedProvider feedId={process.env.NEXT_PUBLIC_KNOCK_FEED_ID!}>
        <NotificationIconButton />
        <NotificationFeedPopover />
      </KnockFeedProvider>
    </KnockProvider>
  );
}
```

That gives you a notification bell icon with an unread badge and a dropdown feed -- like the one in GitHub, Linear, or Notion. Out of the box. With real-time updates via WebSocket.

**Generating the user token (server-side):**

```typescript
// app/api/knock-token/route.ts
import { Knock } from '@knocklabs/node';
import { auth } from '@/lib/auth';
import { NextResponse } from 'next/server';

const knock = new Knock(process.env.KNOCK_API_KEY!);

export async function GET() {
  const session = await auth();
  if (!session?.user?.id) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  const token = await knock.signUserToken(session.user.id, {
    signingKey: process.env.KNOCK_SIGNING_KEY!,
  });

  return NextResponse.json({ token });
}
```

### 7.3 Novu

Novu is the open-source alternative to Knock. If you want to self-host your notification infrastructure, Novu is the way to go.

```bash
npm install @novu/node @novu/notification-center
```

```typescript
// lib/novu.ts (server-side)
import { Novu } from '@novu/node';

export const novu = new Novu(process.env.NOVU_API_KEY!);
```

**Triggering a notification:**

```typescript
import { novu } from '@/lib/novu';

await novu.trigger('new-comment', {
  to: {
    subscriberId: postOwner.id,
    email: postOwner.email,
  },
  payload: {
    commentAuthorName: commentAuthor.name,
    postTitle,
    commentText,
    commentUrl: `https://yourapp.com/posts/${commentId}`,
  },
  actor: {
    subscriberId: commentAuthor.id,
  },
});
```

**In-app notification center:**

```tsx
// components/notification-center.tsx
'use client';

import {
  NovuProvider,
  PopoverNotificationCenter,
  NotificationBell,
} from '@novu/notification-center';

interface NotificationCenterProps {
  subscriberId: string;
}

export function NotificationCenter({ subscriberId }: NotificationCenterProps) {
  return (
    <NovuProvider
      subscriberId={subscriberId}
      applicationIdentifier={process.env.NEXT_PUBLIC_NOVU_APP_ID!}
    >
      <PopoverNotificationCenter>
        {({ unseenCount }) => <NotificationBell unseenCount={unseenCount} />}
      </PopoverNotificationCenter>
    </NovuProvider>
  );
}
```

### 7.4 Knock vs Novu

| | Knock | Novu |
|---|---|---|
| **Hosting** | Managed (cloud only) | Cloud or self-hosted |
| **Pricing** | Free to 10K notifications/mo, then usage-based | Free cloud tier, or free self-hosted |
| **Workflow builder** | Visual, drag-and-drop | Visual, drag-and-drop |
| **React components** | Polished, styled components | Good components, more customizable |
| **Batching/digests** | Built-in | Built-in |
| **Preferences UI** | Pre-built | Pre-built |
| **Best for** | Teams that want managed infra | Teams that want control / self-hosting |

My recommendation: Use Knock if you want things to Just Work with minimal setup. Use Novu if you need to self-host or want deeper customization.

### 7.5 User Notification Preferences

Both Knock and Novu support per-user, per-channel preferences. Here is what a preference UI typically looks like:

```
Notification Preferences
─────────────────────────────────────────────
                     In-App   Push   Email
Comments on my posts   ✅      ✅      ✅
Likes on my posts      ✅      ❌      ❌
New followers          ✅      ✅      ❌
Weekly digest          ──      ──      ✅
System alerts          ✅      ✅      ✅
─────────────────────────────────────────────
```

With Knock, you set preferences via the API:

```typescript
await knock.users.setPreferences(userId, {
  channel_types: {
    email: true,
    push: true,
    in_app_feed: true,
  },
  workflows: {
    'new-comment': {
      channel_types: {
        email: true,
        push: true,
        in_app_feed: true,
      },
    },
    'new-like': {
      channel_types: {
        email: false,
        push: false,
        in_app_feed: true,
      },
    },
    'weekly-digest': {
      channel_types: {
        email: true,
        push: false,
        in_app_feed: false,
      },
    },
  },
});
```

Knock also provides a pre-built `<PreferencesModal>` React component so you do not have to build the UI yourself.

---

## 8. SERVER-SIDE IMPLEMENTATION

### 8.1 Sending from Server Actions

Here is how to trigger emails from Next.js Server Actions:

```typescript
// app/actions/auth.ts
'use server';

import { render } from '@react-email/render';
import { emailProvider } from '@/lib/email';
import { VerifyEmail } from '@/emails/verify-email';
import { db } from '@/lib/db';
import { generateVerificationToken } from '@/lib/tokens';

export async function signUp(formData: FormData) {
  const email = formData.get('email') as string;
  const password = formData.get('password') as string;
  const name = formData.get('name') as string;

  // 1. Create the user
  const user = await db.user.create({
    data: { email, password: await hashPassword(password), name },
  });

  // 2. Generate verification token
  const token = await generateVerificationToken(user.id);
  const verificationUrl = `${process.env.NEXT_PUBLIC_APP_URL}/verify-email?token=${token}`;

  // 3. Queue the email (don't await — fire and forget to queue)
  await queueEmail({
    type: 'verify-email',
    to: email,
    data: { userName: name, verificationUrl, expiresInHours: 24 },
  });

  return { success: true };
}
```

### 8.2 Queuing Emails

Never send emails inline. Always queue them. Here are your options:

**Option 1: Upstash QStash (recommended for Vercel)**

QStash is a serverless message queue that works perfectly with Vercel's serverless functions. You publish a message to a URL, and QStash calls that URL with the message payload. It handles retries, deduplication, and scheduling.

```bash
npm install @upstash/qstash
```

```typescript
// lib/email-queue.ts
import { Client } from '@upstash/qstash';

const qstash = new Client({
  token: process.env.QSTASH_TOKEN!,
});

interface QueuedEmail {
  type: string;
  to: string;
  data: Record<string, any>;
}

export async function queueEmail(email: QueuedEmail) {
  await qstash.publishJSON({
    url: `${process.env.NEXT_PUBLIC_APP_URL}/api/email/send`,
    body: email,
    retries: 3,
    // Deduplication: don't send the same email twice within 60 seconds
    deduplicationId: `${email.type}-${email.to}-${Date.now()}`,
  });
}
```

```typescript
// app/api/email/send/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { verifySignatureAppRouter } from '@upstash/qstash/nextjs';
import { render } from '@react-email/render';
import { emailProvider } from '@/lib/email';
import { VerifyEmail } from '@/emails/verify-email';
import { WelcomeEmail } from '@/emails/welcome';
import { PasswordResetEmail } from '@/emails/password-reset';
import { OrderConfirmationEmail } from '@/emails/order-confirmation';

const emailTemplates: Record<string, {
  component: (data: any) => React.ReactElement;
  subject: (data: any) => string;
}> = {
  'verify-email': {
    component: (data) => VerifyEmail(data),
    subject: () => 'Verify your email address',
  },
  'welcome': {
    component: (data) => WelcomeEmail(data),
    subject: (data) => `Welcome to Acme, ${data.userName}!`,
  },
  'password-reset': {
    component: (data) => PasswordResetEmail(data),
    subject: () => 'Reset your password',
  },
  'order-confirmation': {
    component: (data) => OrderConfirmationEmail(data),
    subject: (data) => `Order #${data.orderId} confirmed`,
  },
};

async function handler(request: NextRequest) {
  const body = await request.json();
  const { type, to, data } = body;

  const template = emailTemplates[type];
  if (!template) {
    return NextResponse.json(
      { error: `Unknown email type: ${type}` },
      { status: 400 }
    );
  }

  const html = await render(template.component(data));
  const text = await render(template.component(data), { plainText: true });

  await emailProvider.send({
    to,
    subject: template.subject(data),
    html,
    text,
  });

  return NextResponse.json({ success: true });
}

// QStash signature verification middleware
export const POST = verifySignatureAppRouter(handler);
```

**Option 2: Vercel Cron + Database Queue**

If you do not want to add QStash, you can use your database as a simple queue:

```typescript
// lib/email-queue-db.ts
import { db } from '@/lib/db';

export async function queueEmail(email: {
  type: string;
  to: string;
  data: Record<string, any>;
}) {
  await db.emailQueue.create({
    data: {
      type: email.type,
      to: email.to,
      data: email.data,
      status: 'pending',
      attempts: 0,
      maxAttempts: 3,
    },
  });
}
```

```typescript
// app/api/cron/send-emails/route.ts
// Triggered by Vercel Cron every minute
import { NextResponse } from 'next/server';
import { db } from '@/lib/db';
import { render } from '@react-email/render';
import { emailProvider } from '@/lib/email';
import { emailTemplates } from '@/lib/email-templates';

export async function GET(request: Request) {
  // Verify cron secret
  const authHeader = request.headers.get('authorization');
  if (authHeader !== `Bearer ${process.env.CRON_SECRET}`) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  // Pick up pending emails (up to 10 per run)
  const pendingEmails = await db.emailQueue.findMany({
    where: {
      status: 'pending',
      attempts: { lt: db.emailQueue.fields.maxAttempts },
    },
    take: 10,
    orderBy: { createdAt: 'asc' },
  });

  for (const email of pendingEmails) {
    try {
      const template = emailTemplates[email.type];
      if (!template) throw new Error(`Unknown template: ${email.type}`);

      const html = await render(template.component(email.data));
      const text = await render(template.component(email.data), { plainText: true });

      await emailProvider.send({
        to: email.to,
        subject: template.subject(email.data),
        html,
        text,
      });

      await db.emailQueue.update({
        where: { id: email.id },
        data: { status: 'sent', sentAt: new Date() },
      });
    } catch (error) {
      await db.emailQueue.update({
        where: { id: email.id },
        data: {
          attempts: { increment: 1 },
          lastError: (error as Error).message,
          status: email.attempts + 1 >= email.maxAttempts ? 'failed' : 'pending',
        },
      });
    }
  }

  return NextResponse.json({
    processed: pendingEmails.length,
  });
}
```

```json
// vercel.json
{
  "crons": [
    {
      "path": "/api/cron/send-emails",
      "schedule": "* * * * *"
    }
  ]
}
```

### 8.3 Rate Limiting

Do not let your system send unlimited emails. Rate limit at multiple levels:

```typescript
// lib/email-rate-limit.ts
import { Ratelimit } from '@upstash/ratelimit';
import { Redis } from '@upstash/redis';

const redis = new Redis({
  url: process.env.UPSTASH_REDIS_REST_URL!,
  token: process.env.UPSTASH_REDIS_REST_TOKEN!,
});

// Per-user: max 10 emails per hour
export const userEmailRateLimit = new Ratelimit({
  redis,
  limiter: Ratelimit.slidingWindow(10, '1 h'),
  prefix: 'email:user',
});

// Global: max 1000 emails per hour
export const globalEmailRateLimit = new Ratelimit({
  redis,
  limiter: Ratelimit.slidingWindow(1000, '1 h'),
  prefix: 'email:global',
});

export async function checkEmailRateLimit(userId: string): Promise<boolean> {
  const [userResult, globalResult] = await Promise.all([
    userEmailRateLimit.limit(userId),
    globalEmailRateLimit.limit('global'),
  ]);

  if (!userResult.success) {
    console.warn(`User ${userId} hit email rate limit`);
    return false;
  }

  if (!globalResult.success) {
    console.warn('Global email rate limit reached');
    return false;
  }

  return true;
}
```

```typescript
// Use in your email sending function
export async function sendTransactionalEmail(
  userId: string,
  type: string,
  to: string,
  data: Record<string, any>
) {
  // Check rate limits
  const allowed = await checkEmailRateLimit(userId);
  if (!allowed) {
    console.warn(`Email rate limited for user ${userId}, type: ${type}`);
    return; // Silently drop — don't error
  }

  // Check bounce/opt-out status
  const user = await db.user.findUnique({
    where: { id: userId },
    select: { emailBounced: true, emailOptOut: true },
  });

  if (user?.emailBounced || user?.emailOptOut) {
    return;
  }

  // Queue the email
  await queueEmail({ type, to, data });
}
```

---

## 9. COMPLETE EXAMPLE

Let me put it all together. Here is a complete, production-ready email system for a Next.js app using Resend and React Email.

### 9.1 Project Structure

```
your-app/
├── emails/                          # React Email templates
│   ├── components/                  # Shared email components
│   │   ├── email-layout.tsx         # Base layout wrapper
│   │   ├── email-button.tsx         # Reusable button
│   │   └── email-footer.tsx         # Footer with unsubscribe
│   ├── verify-email.tsx
│   ├── welcome.tsx
│   ├── password-reset.tsx
│   ├── order-confirmation.tsx
│   ├── weekly-digest.tsx
│   └── alert.tsx
├── lib/
│   ├── email/
│   │   ├── index.ts                 # Email provider singleton
│   │   ├── types.ts                 # Shared types
│   │   ├── templates.ts             # Template registry
│   │   ├── queue.ts                 # Queue helper
│   │   └── rate-limit.ts            # Rate limiting
│   └── ...
├── app/
│   ├── api/
│   │   ├── email/
│   │   │   └── send/route.ts        # QStash webhook handler
│   │   └── webhooks/
│   │       └── resend/route.ts      # Resend event webhook
│   └── ...
└── package.json
```

### 9.2 Shared Email Layout

```tsx
// emails/components/email-layout.tsx
import {
  Body,
  Container,
  Head,
  Html,
  Img,
  Preview,
  Section,
  Text,
  Hr,
  Link,
} from '@react-email/components';
import { ReactNode } from 'react';

interface EmailLayoutProps {
  previewText: string;
  children: ReactNode;
  footerText?: string;
  unsubscribeUrl?: string;
}

export function EmailLayout({
  previewText,
  children,
  footerText,
  unsubscribeUrl,
}: EmailLayoutProps) {
  return (
    <Html>
      <Head />
      <Preview>{previewText}</Preview>
      <Body style={main}>
        <Container style={container}>
          {/* Header */}
          <Section style={header}>
            <Img
              src="https://yourapp.com/email-logo.png"
              width="120"
              height="32"
              alt="Acme"
            />
          </Section>

          {/* Content */}
          <Section style={content}>
            {children}
          </Section>

          {/* Footer */}
          <Hr style={hr} />
          <Section style={footer}>
            {footerText && <Text style={footerTextStyle}>{footerText}</Text>}
            <Text style={footerTextStyle}>
              Acme, Inc. | 123 Main St, San Francisco, CA 94102
            </Text>
            {unsubscribeUrl && (
              <Text style={footerTextStyle}>
                <Link href={unsubscribeUrl} style={unsubscribeLink}>
                  Unsubscribe
                </Link>
              </Text>
            )}
          </Section>
        </Container>
      </Body>
    </Html>
  );
}

const main = {
  backgroundColor: '#f6f9fc',
  fontFamily: '-apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif',
  padding: '40px 0',
};

const container = {
  backgroundColor: '#ffffff',
  margin: '0 auto',
  maxWidth: '560px',
  borderRadius: '8px',
  border: '1px solid #e5e7eb',
  overflow: 'hidden' as const,
};

const header = {
  padding: '24px 40px',
  borderBottom: '1px solid #e5e7eb',
};

const content = {
  padding: '40px',
};

const hr = {
  borderColor: '#e5e7eb',
  margin: '0',
};

const footer = {
  padding: '24px 40px',
  backgroundColor: '#f9fafb',
};

const footerTextStyle = {
  fontSize: '12px',
  color: '#9ca3af',
  lineHeight: '20px',
  margin: '0 0 4px',
  textAlign: 'center' as const,
};

const unsubscribeLink = {
  color: '#9ca3af',
  textDecoration: 'underline',
};
```

### 9.3 Order Confirmation with Layout

```tsx
// emails/order-confirmation-v2.tsx
import { Button, Column, Heading, Hr, Img, Row, Section, Text } from '@react-email/components';
import { EmailLayout } from './components/email-layout';

// ... (same props as the order confirmation above)

export function OrderConfirmationV2(props: OrderConfirmationProps) {
  const { userName, orderId, items, total } = props;
  const formatCurrency = (amount: number) => `$${amount.toFixed(2)}`;

  return (
    <EmailLayout
      previewText={`Order #${orderId} confirmed -- ${formatCurrency(total)}`}
      footerText="Questions? Reply to this email or contact support@yourapp.com."
    >
      <Heading style={heading}>Order confirmed!</Heading>
      <Text style={text}>Hey {userName}, thanks for your order.</Text>

      {/* ... same content as section 4.4 ... */}

      <Section style={{ textAlign: 'center' as const, margin: '32px 0 0' }}>
        <Button style={button} href={`https://yourapp.com/orders/${orderId}`}>
          View Order Details
        </Button>
      </Section>
    </EmailLayout>
  );
}

const heading = { fontSize: '24px', fontWeight: '700' as const, color: '#111827', margin: '0 0 16px' };
const text = { fontSize: '15px', color: '#374151', lineHeight: '24px', margin: '0 0 24px' };
const button = {
  backgroundColor: '#111827',
  color: '#ffffff',
  padding: '12px 32px',
  borderRadius: '6px',
  fontSize: '15px',
  fontWeight: '600' as const,
  textDecoration: 'none',
};
```

### 9.4 Template Registry

```typescript
// lib/email/templates.ts
import { VerifyEmail } from '@/emails/verify-email';
import { WelcomeEmail } from '@/emails/welcome';
import { PasswordResetEmail } from '@/emails/password-reset';
import { OrderConfirmationEmail } from '@/emails/order-confirmation';
import { WeeklyDigestEmail } from '@/emails/weekly-digest';
import { AlertEmail } from '@/emails/alert';

export interface EmailTemplate {
  component: (data: any) => React.ReactElement;
  subject: (data: any) => string;
}

export const emailTemplates: Record<string, EmailTemplate> = {
  'verify-email': {
    component: (data) => VerifyEmail(data),
    subject: () => 'Verify your email address',
  },
  'welcome': {
    component: (data) => WelcomeEmail(data),
    subject: (data) => `Welcome to Acme, ${data.userName}!`,
  },
  'password-reset': {
    component: (data) => PasswordResetEmail(data),
    subject: () => 'Reset your password',
  },
  'order-confirmation': {
    component: (data) => OrderConfirmationEmail(data),
    subject: (data) => `Order #${data.orderId} confirmed`,
  },
  'weekly-digest': {
    component: (data) => WeeklyDigestEmail(data),
    subject: (data) => `Your weekly digest: ${data.weekRange}`,
  },
  'alert': {
    component: (data) => AlertEmail(data),
    subject: (data) => `${data.severity === 'error' ? '🚨' : 'ℹ️'} ${data.title}`,
  },
};
```

### 9.5 Server Action That Triggers Everything

Here is the complete flow. User places an order. The Server Action creates the order, charges the payment, and queues the confirmation email:

```typescript
// app/actions/orders.ts
'use server';

import { auth } from '@/lib/auth';
import { db } from '@/lib/db';
import { stripe } from '@/lib/stripe';
import { queueEmail } from '@/lib/email/queue';
import { redirect } from 'next/navigation';

export async function placeOrder(formData: FormData) {
  const session = await auth();
  if (!session?.user) throw new Error('Unauthorized');

  const cartItems = await db.cartItem.findMany({
    where: { userId: session.user.id },
    include: { product: true },
  });

  if (cartItems.length === 0) {
    throw new Error('Cart is empty');
  }

  // Calculate totals
  const subtotal = cartItems.reduce(
    (sum, item) => sum + item.product.price * item.quantity,
    0
  );
  const shipping = subtotal >= 100 ? 0 : 9.99;
  const tax = subtotal * 0.0875; // 8.75% tax
  const total = subtotal + shipping + tax;

  // Create the order in the database
  const order = await db.order.create({
    data: {
      userId: session.user.id,
      status: 'confirmed',
      subtotal,
      shipping,
      tax,
      total,
      items: {
        create: cartItems.map((item) => ({
          productId: item.productId,
          quantity: item.quantity,
          price: item.product.price,
        })),
      },
    },
    include: { items: { include: { product: true } } },
  });

  // Charge the payment (Stripe — see Chapter 24)
  // ... payment logic here ...

  // Clear the cart
  await db.cartItem.deleteMany({
    where: { userId: session.user.id },
  });

  // Queue the confirmation email — DO NOT AWAIT THE ACTUAL SEND
  await queueEmail({
    type: 'order-confirmation',
    to: session.user.email!,
    data: {
      userName: session.user.name ?? 'there',
      orderId: order.id,
      orderDate: new Date().toLocaleDateString('en-US', {
        year: 'numeric',
        month: 'long',
        day: 'numeric',
      }),
      items: order.items.map((item) => ({
        name: item.product.name,
        quantity: item.quantity,
        price: item.price,
        imageUrl: item.product.imageUrl,
      })),
      subtotal,
      shipping,
      tax,
      total,
      shippingAddress: '123 Main St\nSan Francisco, CA 94102',
      estimatedDelivery: 'April 12-15, 2026',
    },
  });

  redirect(`/orders/${order.id}/confirmation`);
}
```

### 9.6 Testing Emails in Development

```typescript
// lib/email/providers/console.ts
// In development, log emails to the console instead of sending them
import type { EmailPayload, EmailProvider } from '../types';

export class ConsoleProvider implements EmailProvider {
  async send(payload: EmailPayload): Promise<{ id: string }> {
    console.log('\n📧 ═══════════════════════════════════════');
    console.log(`  To:      ${payload.to}`);
    console.log(`  Subject: ${payload.subject}`);
    console.log(`  From:    ${payload.from ?? 'default'}`);
    if (payload.cc) console.log(`  CC:      ${payload.cc.join(', ')}`);
    if (payload.bcc) console.log(`  BCC:     ${payload.bcc.join(', ')}`);
    console.log('═══════════════════════════════════════════\n');

    // Optionally write the HTML to a file for preview
    if (process.env.EMAIL_DEBUG_DIR) {
      const fs = await import('fs');
      const path = await import('path');
      const filename = `${Date.now()}-${payload.subject?.replace(/\W+/g, '-')}.html`;
      const filepath = path.join(process.env.EMAIL_DEBUG_DIR, filename);
      fs.writeFileSync(filepath, payload.html);
      console.log(`  HTML saved to: ${filepath}\n`);
    }

    return { id: `console-${Date.now()}` };
  }
}
```

```typescript
// lib/email/index.ts — updated with development provider
function createEmailProvider(): EmailProvider {
  if (process.env.NODE_ENV === 'development') {
    return new ConsoleProvider();
  }

  const provider = process.env.EMAIL_PROVIDER ?? 'resend';
  switch (provider) {
    case 'resend':
      return new ResendProvider(/* ... */);
    case 'ses':
      return new SESProvider(/* ... */);
    default:
      throw new Error(`Unknown email provider: ${provider}`);
  }
}
```

### 9.7 Putting It All Together

Here is the complete dependency map:

```
User clicks "Place Order"
    │
    ▼
Server Action: placeOrder()
    ├── Creates order in database
    ├── Charges payment via Stripe
    ├── Clears the cart
    ├── Queues email via QStash ──────────┐
    └── Redirects to confirmation page     │
                                           │
    ┌──────────────────────────────────────┘
    │
    ▼
QStash calls /api/email/send (with retries)
    │
    ▼
Route handler:
    ├── Verifies QStash signature
    ├── Looks up template by type
    ├── Renders React Email component to HTML
    └── Sends via Resend API
            │
            ▼
    Resend delivers the email
            │
            ▼
    Resend webhook → /api/webhooks/resend
            │
            ▼
    Update delivery status in database
```

This architecture is robust. The user's action (placing an order) never waits for email delivery. The email is queued and sent asynchronously. If Resend is down, QStash retries. If the email bounces, the webhook handler records it. If the user has opted out, the rate limiter catches it.

---

## Key Takeaways

1. **Transactional email is part of your product.** Treat it with the same care as your UI. Use React Email to build proper templates.

2. **Resend + React Email is the modern stack.** You write JSX, you get email-safe HTML. The DX gap between Resend and legacy providers is enormous.

3. **Set up SPF, DKIM, and DMARC before sending a single production email.** Skip this and 30% of your emails land in spam.

4. **Never send email inline in a request.** Queue it with QStash or a database queue. A slow email API should never block your user.

5. **Handle bounces and complaints.** Your sender reputation depends on it. Stop sending to bounced addresses immediately.

6. **Use Knock or Novu for multi-channel notifications.** Building your own notification router is weeks of work for a worse result.

7. **Abstract your email provider.** Write a provider interface, implement it for Resend, and swap to SES when you need to. One-file change.

8. **Rate limit everything.** Per-user limits prevent abuse. Global limits prevent accidents. Both prevent your sender reputation from being destroyed.

---

*Next chapter: We will cover analytics, event tracking, and understanding how users actually use your app -- because sending the right email at the right time requires knowing what your users are doing.*
