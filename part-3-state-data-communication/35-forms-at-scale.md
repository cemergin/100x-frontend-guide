<!--
  CHAPTER: 35
  TITLE: Forms at Scale — Multi-Step, Dynamic & Complex
  PART: III — State, Data & Communication
  PREREQS: Chapters 10, 0d
  KEY_TOPICS: React Hook Form, Zod, multi-step wizards, dynamic forms, conditional fields, file uploads, form state machines, server-side validation, useActionState, form arrays, dependent fields, autosave
  DIFFICULTY: Intermediate → Advanced
  UPDATED: 2026-04-07
-->

# Chapter 35: Forms at Scale — Multi-Step, Dynamic & Complex

> **Part III — State, Data & Communication** | Prerequisites: Chapters 10, 0d | Difficulty: Intermediate to Advanced

<details>
<summary><strong>TL;DR</strong></summary>

- Forms are the primary trust surface in your app -- a broken checkout form or a wizard that loses data on "back" costs you real revenue and real users
- React Hook Form + Zod is the production stack: uncontrolled inputs via refs for performance, Zod schemas for validation, and `useFieldArray` for dynamic lists -- do not reinvent this with `useState`
- Multi-step wizards need a state machine (XState or `useReducer`), per-step validation, persisted progress, and a guarantee that "back" never destroys user input
- Dynamic forms (JSON-schema-driven, CMS-generated) require a field registry pattern that maps schema types to React components -- keep the mapping declarative
- File uploads are their own beast: use signed URLs (S3/Vercel Blob) for large files, track progress with `XMLHttpRequest`, and always preview before uploading
- Server-side validation must use the same Zod schema as the client; `useActionState` in Next.js makes returning field-level errors from Server Actions ergonomic
- Autosave needs debouncing, dirty-field tracking, conflict detection, and a clear "saving..."/"saved" indicator -- never silently swallow save failures
- Controlled inputs cause re-renders on every keystroke; React Hook Form's ref-based architecture avoids this -- for forms with 50+ fields, this is the difference between usable and unusable

</details>

I need to be honest with you: I avoided writing this chapter for months. Forms feel mundane. Every tutorial covers `<input value={...} onChange={...} />` and calls it done. But if you have ever built a real application -- one with checkout flows, multi-step onboarding, dynamic survey builders, profile editors with image uploads, or admin dashboards with filter sidebars -- you know the truth: **forms are where the complexity lives.**

Forms are the primary interaction surface between your user and your application. They are where trust is built or destroyed. A checkout form that loses data when the user hits "back" costs you revenue. An onboarding wizard that shows validation errors only after the user clicks "next" costs you signups. A profile editor that silently fails to save costs you the user's confidence.

And yet, most teams treat forms as an afterthought. They reach for `useState`, wire up controlled inputs, sprinkle some inline validation, and wonder why their 30-field form re-renders the entire page on every keystroke. They build a multi-step wizard with a pile of boolean flags and wonder why the "back" button sometimes loses data. They implement file uploads with a bare `fetch` and wonder why there is no progress bar.

This chapter is the antidote. We are going to cover forms the way a 100x architect covers them: with the right abstractions, the right libraries, and the right patterns for every level of complexity.

### In This Chapter
- Why Forms Deserve Their Own Chapter (and their own architecture)
- React Hook Form Advanced Patterns (useFieldArray, watch, trigger, setError, formState)
- Zod Advanced Validation (refinements, transforms, discriminated unions, async validation)
- Multi-Step Wizard Architecture (state machines, persisted progress, per-step validation)
- Dynamic Forms (JSON-schema-driven, CMS-generated, field registry pattern)
- File Uploads (signed URLs, progress tracking, image preview, drag-and-drop)
- Server-Side Validation (shared Zod schemas, useActionState, field-level server errors)
- Autosave (debouncing, conflict detection, optimistic persistence)
- Form Performance (why controlled inputs kill perf, ref-based architecture, large forms)
- Complete Examples (checkout, survey builder, profile editor, search filters)

### Related Chapters
- [Ch 10: Data Fetching & Server Communication] -- API layer that forms submit to
- [Ch 9: State Management at Scale] -- where form state fits in the state taxonomy
- [Ch 36: Error Handling Architecture] -- how form errors feed into the global error system
- [Ch 4: TypeScript for Architects] -- the type system that makes Zod schemas type-safe

---

## 1. WHY FORMS DESERVE THEIR OWN CHAPTER

Let me paint a picture. You are building a SaaS onboarding flow. The user needs to:

1. Enter their company name, size, and industry (step 1)
2. Invite team members -- a dynamic list where they can add/remove rows (step 2)
3. Choose a plan with conditional fields (monthly/annual billing, enterprise needs a contact form) (step 3)
4. Upload a company logo (step 4)
5. Review everything and submit (step 5)

Now add these requirements:
- Email addresses in step 2 must be validated for format *and* checked against your API for existing accounts (async validation)
- The user can navigate freely between steps. "Back" must never lose data. Refreshing the page should restore progress.
- Step 3's fields change based on the plan selected (a discriminated union in your schema)
- The logo upload needs a preview, progress bar, and the ability to crop
- Server-side validation must return field-level errors that appear inline next to the right input
- The entire flow should autosave so the user can close the tab and come back later

This is not a contrived example. This is Tuesday at any growing SaaS company. And if you try to build it with `useState` and inline validation, you will produce a tangled mess that is impossible to test, impossible to extend, and full of subtle data-loss bugs.

Forms are a **domain** with their own patterns, their own state management concerns, their own performance characteristics, and their own UX requirements. They deserve dedicated architecture.

### 1.1 The Form Architecture Stack

Here is what we are going to build with:

```
┌─────────────────────────────────────────────────────────────┐
│                    THE FORM STACK                            │
│                                                              │
│  LAYER 1: SCHEMA (Zod)                                      │
│  ├── Single source of truth for validation rules             │
│  ├── Shared between client and server                        │
│  ├── Generates TypeScript types automatically                │
│  └── Handles sync, async, and cross-field validation         │
│                                                              │
│  LAYER 2: FORM STATE (React Hook Form)                       │
│  ├── Uncontrolled inputs via refs (no re-renders)            │
│  ├── Field arrays for dynamic lists                          │
│  ├── Watch for dependent fields                              │
│  ├── Dirty tracking for partial saves                        │
│  └── Integration with Zod via @hookform/resolvers            │
│                                                              │
│  LAYER 3: FORM UI (Your components)                          │
│  ├── Reusable field components (TextField, Select, etc.)     │
│  ├── Error display (inline, toast, summary)                  │
│  ├── Loading states (submitting, validating)                 │
│  └── Accessibility (labels, aria-describedby, focus mgmt)    │
│                                                              │
│  LAYER 4: SUBMISSION (API / Server Actions)                  │
│  ├── Server-side re-validation with same Zod schema          │
│  ├── Field-level error mapping back to client                │
│  ├── Optimistic UI during submission                         │
│  └── Retry logic for transient failures                      │
│                                                              │
│  LAYER 5: PERSISTENCE (Autosave / Multi-step)                │
│  ├── Debounced autosave on dirty fields                      │
│  ├── localStorage/AsyncStorage for wizard progress           │
│  ├── Conflict detection for concurrent edits                 │
│  └── State machine for wizard navigation                     │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 Why React Hook Form + Zod (and Not Formik, or Rolling Your Own)

I have used Formik, React Final Form, and hand-rolled solutions on production apps. Here is why React Hook Form (RHF) + Zod won:

**Performance.** RHF uses uncontrolled inputs. It registers inputs via `ref` and reads values directly from the DOM. This means typing in one field does *not* re-render every other field. For a 5-field login form, this does not matter. For a 50-field admin form with dynamic arrays, this is the difference between 60fps and "why is the cursor lagging."

**TypeScript integration.** Zod schemas infer TypeScript types. RHF's `useForm<z.infer<typeof schema>>` gives you full type safety across `register`, `watch`, `setValue`, `getValues`, and every other API. No manual type definitions that drift from your validation rules.

**Bundle size.** RHF is ~9KB gzipped. Formik is ~13KB. But the real difference is that RHF does not pull in a dependency tree -- it is self-contained.

**Ecosystem.** `@hookform/resolvers` connects RHF to Zod (and Yup, Superstruct, etc.) with a one-line resolver. `@hookform/devtools` gives you a visual inspector for form state during development.

```bash
# The full form stack
npm install react-hook-form zod @hookform/resolvers
```

---

## 2. REACT HOOK FORM ADVANCED PATTERNS

If you have used RHF for basic forms, you know `useForm`, `register`, `handleSubmit`, and `formState.errors`. This section goes deeper into the APIs that matter at scale.

### 2.1 The Foundation: Typed Forms with Zod

Every form starts with a Zod schema. The schema is the single source of truth.

```typescript
// schemas/profile.ts
import { z } from 'zod';

export const profileSchema = z.object({
  firstName: z
    .string()
    .min(1, 'First name is required')
    .max(50, 'First name must be 50 characters or less'),
  lastName: z
    .string()
    .min(1, 'Last name is required')
    .max(50, 'Last name must be 50 characters or less'),
  email: z
    .string()
    .min(1, 'Email is required')
    .email('Please enter a valid email address'),
  bio: z
    .string()
    .max(500, 'Bio must be 500 characters or less')
    .optional(),
  website: z
    .string()
    .url('Please enter a valid URL')
    .optional()
    .or(z.literal('')), // Allow empty string
  age: z
    .number({ invalid_type_error: 'Age must be a number' })
    .min(13, 'You must be at least 13 years old')
    .max(120, 'Please enter a valid age')
    .optional(),
});

// The TypeScript type is derived from the schema -- never define it separately
export type ProfileFormData = z.infer<typeof profileSchema>;
// Result: {
//   firstName: string;
//   lastName: string;
//   email: string;
//   bio?: string | undefined;
//   website?: string | undefined;
//   age?: number | undefined;
// }
```

Now the form component:

```tsx
// components/ProfileForm.tsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { profileSchema, type ProfileFormData } from '@/schemas/profile';

export function ProfileForm({ defaultValues }: { defaultValues?: Partial<ProfileFormData> }) {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting, isDirty, dirtyFields },
  } = useForm<ProfileFormData>({
    resolver: zodResolver(profileSchema),
    defaultValues: {
      firstName: '',
      lastName: '',
      email: '',
      bio: '',
      website: '',
      ...defaultValues,
    },
    mode: 'onBlur', // Validate on blur, not on every keystroke
  });

  const onSubmit = async (data: ProfileFormData) => {
    // `data` is fully typed and validated
    await api.updateProfile(data);
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <div>
        <label htmlFor="firstName">First Name</label>
        <input
          id="firstName"
          {...register('firstName')}
          aria-invalid={!!errors.firstName}
          aria-describedby={errors.firstName ? 'firstName-error' : undefined}
        />
        {errors.firstName && (
          <p id="firstName-error" role="alert">
            {errors.firstName.message}
          </p>
        )}
      </div>

      <div>
        <label htmlFor="email">Email</label>
        <input
          id="email"
          type="email"
          {...register('email')}
          aria-invalid={!!errors.email}
          aria-describedby={errors.email ? 'email-error' : undefined}
        />
        {errors.email && (
          <p id="email-error" role="alert">
            {errors.email.message}
          </p>
        )}
      </div>

      <div>
        <label htmlFor="age">Age</label>
        <input
          id="age"
          type="number"
          {...register('age', { valueAsNumber: true })}
          aria-invalid={!!errors.age}
          aria-describedby={errors.age ? 'age-error' : undefined}
        />
        {errors.age && (
          <p id="age-error" role="alert">
            {errors.age.message}
          </p>
        )}
      </div>

      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Saving...' : 'Save Profile'}
      </button>
    </form>
  );
}
```

**Key points:**
- `mode: 'onBlur'` validates when the user leaves a field, not on every keystroke. This is the right default for most forms. Use `mode: 'onChange'` only when you need instant feedback (e.g., password strength meters).
- `register('age', { valueAsNumber: true })` tells RHF to coerce the string from the DOM to a number before validation. Without this, Zod would receive a string and the `z.number()` check would fail.
- `aria-invalid` and `aria-describedby` are not optional. Screen readers need them. Accessibility is not a nice-to-have.

### 2.2 useFieldArray for Dynamic Lists

This is where RHF really earns its keep. Imagine you are building an invoice form where the user can add line items:

```typescript
// schemas/invoice.ts
import { z } from 'zod';

export const lineItemSchema = z.object({
  description: z.string().min(1, 'Description is required'),
  quantity: z.number().min(1, 'Quantity must be at least 1'),
  unitPrice: z.number().min(0, 'Price cannot be negative'),
});

export const invoiceSchema = z.object({
  clientName: z.string().min(1, 'Client name is required'),
  clientEmail: z.string().email('Invalid email'),
  dueDate: z.string().min(1, 'Due date is required'),
  lineItems: z
    .array(lineItemSchema)
    .min(1, 'At least one line item is required')
    .max(50, 'Maximum 50 line items'),
  notes: z.string().optional(),
  taxRate: z.number().min(0).max(100).default(0),
});

export type InvoiceFormData = z.infer<typeof invoiceSchema>;
```

```tsx
// components/InvoiceForm.tsx
import { useForm, useFieldArray, useWatch } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { invoiceSchema, type InvoiceFormData } from '@/schemas/invoice';

export function InvoiceForm() {
  const {
    register,
    control,
    handleSubmit,
    formState: { errors },
  } = useForm<InvoiceFormData>({
    resolver: zodResolver(invoiceSchema),
    defaultValues: {
      clientName: '',
      clientEmail: '',
      dueDate: '',
      lineItems: [{ description: '', quantity: 1, unitPrice: 0 }],
      notes: '',
      taxRate: 0,
    },
  });

  const { fields, append, remove, move, swap } = useFieldArray({
    control,
    name: 'lineItems',
  });

  const onSubmit = async (data: InvoiceFormData) => {
    await api.createInvoice(data);
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      {/* Client info fields omitted for brevity */}

      <fieldset>
        <legend>Line Items</legend>

        {fields.map((field, index) => (
          <div key={field.id} className="line-item-row">
            {/* IMPORTANT: use field.id as key, NOT index */}
            <div>
              <label htmlFor={`lineItems.${index}.description`}>
                Description
              </label>
              <input
                id={`lineItems.${index}.description`}
                {...register(`lineItems.${index}.description`)}
              />
              {errors.lineItems?.[index]?.description && (
                <p role="alert">
                  {errors.lineItems[index].description.message}
                </p>
              )}
            </div>

            <div>
              <label htmlFor={`lineItems.${index}.quantity`}>Qty</label>
              <input
                id={`lineItems.${index}.quantity`}
                type="number"
                {...register(`lineItems.${index}.quantity`, {
                  valueAsNumber: true,
                })}
              />
              {errors.lineItems?.[index]?.quantity && (
                <p role="alert">
                  {errors.lineItems[index].quantity.message}
                </p>
              )}
            </div>

            <div>
              <label htmlFor={`lineItems.${index}.unitPrice`}>
                Unit Price
              </label>
              <input
                id={`lineItems.${index}.unitPrice`}
                type="number"
                step="0.01"
                {...register(`lineItems.${index}.unitPrice`, {
                  valueAsNumber: true,
                })}
              />
              {errors.lineItems?.[index]?.unitPrice && (
                <p role="alert">
                  {errors.lineItems[index].unitPrice.message}
                </p>
              )}
            </div>

            {/* Line total (computed, not a form field) */}
            <LineItemTotal control={control} index={index} />

            <button
              type="button"
              onClick={() => remove(index)}
              disabled={fields.length <= 1}
              aria-label={`Remove line item ${index + 1}`}
            >
              Remove
            </button>

            {/* Reordering */}
            <button
              type="button"
              onClick={() => index > 0 && move(index, index - 1)}
              disabled={index === 0}
              aria-label="Move up"
            >
              ↑
            </button>
            <button
              type="button"
              onClick={() => index < fields.length - 1 && move(index, index + 1)}
              disabled={index === fields.length - 1}
              aria-label="Move down"
            >
              ↓
            </button>
          </div>
        ))}

        {/* Array-level error */}
        {errors.lineItems?.root && (
          <p role="alert">{errors.lineItems.root.message}</p>
        )}
        {typeof errors.lineItems?.message === 'string' && (
          <p role="alert">{errors.lineItems.message}</p>
        )}

        <button
          type="button"
          onClick={() =>
            append({ description: '', quantity: 1, unitPrice: 0 })
          }
        >
          + Add Line Item
        </button>
      </fieldset>

      {/* Invoice total */}
      <InvoiceTotal control={control} />

      <button type="submit">Create Invoice</button>
    </form>
  );
}

// Extracted component that only re-renders when its specific line item changes
function LineItemTotal({
  control,
  index,
}: {
  control: any;
  index: number;
}) {
  const quantity = useWatch({ control, name: `lineItems.${index}.quantity` });
  const unitPrice = useWatch({ control, name: `lineItems.${index}.unitPrice` });

  const total = (quantity || 0) * (unitPrice || 0);

  return (
    <span aria-label={`Line item ${index + 1} total`}>
      ${total.toFixed(2)}
    </span>
  );
}

// Only re-renders when any line item or tax rate changes
function InvoiceTotal({ control }: { control: any }) {
  const lineItems = useWatch({ control, name: 'lineItems' });
  const taxRate = useWatch({ control, name: 'taxRate' });

  const subtotal = (lineItems || []).reduce(
    (sum: number, item: any) => sum + (item.quantity || 0) * (item.unitPrice || 0),
    0,
  );
  const tax = subtotal * ((taxRate || 0) / 100);
  const total = subtotal + tax;

  return (
    <div>
      <p>Subtotal: ${subtotal.toFixed(2)}</p>
      <p>Tax ({taxRate || 0}%): ${tax.toFixed(2)}</p>
      <p><strong>Total: ${total.toFixed(2)}</strong></p>
    </div>
  );
}
```

**Critical details:**
- **Use `field.id` as the key**, not the array index. RHF generates stable IDs for each field. Using the index as a key causes fields to lose their state when items are reordered or removed.
- **`useWatch` isolates re-renders.** The `LineItemTotal` component only re-renders when its specific line item changes, not when any other line item changes. This is how you keep a 50-row invoice form responsive.
- **`append`, `remove`, `move`, `swap`, `insert`, `prepend`, `update`, `replace`** -- these are all available. `move` is particularly useful for drag-and-drop reordering.

### 2.3 watch vs useWatch

This trips up a lot of developers.

```typescript
// watch: triggers re-render of the ENTIRE form component
const { watch } = useForm();
const email = watch('email'); // Component re-renders on every email change

// useWatch: triggers re-render of ONLY the component that calls it
function EmailPreview({ control }) {
  const email = useWatch({ control, name: 'email' });
  return <p>Preview: {email}</p>; // Only this component re-renders
}
```

**Rule of thumb:** Use `watch` in event handlers and effects where you need the value once. Use `useWatch` in render paths where you need the value to display. In practice, `useWatch` is almost always what you want.

### 2.4 Dependent Fields with watch

Sometimes a field's options or visibility depend on another field's value. This is where `watch` shines:

```tsx
// components/ShippingForm.tsx
function ShippingForm() {
  const { register, watch, control, formState: { errors } } = useForm({
    resolver: zodResolver(shippingSchema),
    defaultValues: {
      country: '',
      state: '',
      postalCode: '',
      shippingMethod: 'standard',
    },
  });

  // Watch the country to conditionally show state/province
  const country = watch('country');

  // Watch shipping method to show delivery estimate
  const shippingMethod = watch('shippingMethod');

  return (
    <form>
      <select {...register('country')}>
        <option value="">Select country</option>
        <option value="US">United States</option>
        <option value="CA">Canada</option>
        <option value="GB">United Kingdom</option>
      </select>

      {/* State/Province only shown for US and Canada */}
      {(country === 'US' || country === 'CA') && (
        <div>
          <label>{country === 'US' ? 'State' : 'Province'}</label>
          <select {...register('state')}>
            {(country === 'US' ? US_STATES : CA_PROVINCES).map((s) => (
              <option key={s.code} value={s.code}>
                {s.name}
              </option>
            ))}
          </select>
        </div>
      )}

      {/* Postal code format depends on country */}
      <div>
        <label>
          {country === 'US' ? 'ZIP Code' : 'Postal Code'}
        </label>
        <input
          {...register('postalCode')}
          placeholder={
            country === 'US'
              ? '12345'
              : country === 'CA'
                ? 'A1A 1A1'
                : 'Enter postal code'
          }
        />
      </div>

      {/* Delivery estimate based on shipping method */}
      <div>
        <select {...register('shippingMethod')}>
          <option value="standard">Standard (5-7 days)</option>
          <option value="express">Express (2-3 days)</option>
          <option value="overnight">Overnight</option>
        </select>
        <DeliveryEstimate method={shippingMethod} control={control} />
      </div>
    </form>
  );
}
```

### 2.5 trigger for Cross-Field Validation

Sometimes you need to re-validate one field when another changes. For example, "confirm password" should re-validate when "password" changes:

```tsx
function PasswordForm() {
  const { register, trigger, watch, formState: { errors } } = useForm({
    resolver: zodResolver(passwordSchema),
  });

  // When password changes, re-validate confirmPassword
  const password = watch('password');
  useEffect(() => {
    if (password) {
      trigger('confirmPassword');
    }
  }, [password, trigger]);

  return (
    <form>
      <input type="password" {...register('password')} />
      <input type="password" {...register('confirmPassword')} />
    </form>
  );
}
```

You can also trigger validation programmatically:

```typescript
// Validate a single field
await trigger('email');

// Validate multiple fields
await trigger(['email', 'password']);

// Validate the entire form (returns boolean)
const isValid = await trigger();

// Trigger with specific options
await trigger('email', { shouldFocus: true }); // Focus the field if invalid
```

### 2.6 setError and clearErrors for Server Errors

After form submission, the server might return errors that your client-side validation could not catch (e.g., "email already taken"). You need to display these inline, next to the relevant field:

```tsx
function SignupForm() {
  const {
    register,
    handleSubmit,
    setError,
    clearErrors,
    formState: { errors, isSubmitting },
  } = useForm<SignupFormData>({
    resolver: zodResolver(signupSchema),
  });

  const onSubmit = async (data: SignupFormData) => {
    try {
      await api.signup(data);
    } catch (error) {
      if (error instanceof ApiError && error.details?.fieldErrors) {
        // Server returned field-level errors
        const fieldErrors = error.details.fieldErrors as Record<string, string>;

        Object.entries(fieldErrors).forEach(([field, message]) => {
          setError(field as keyof SignupFormData, {
            type: 'server',
            message,
          });
        });
      } else {
        // Generic server error -- set on root
        setError('root.serverError', {
          type: 'server',
          message: 'Something went wrong. Please try again.',
        });
      }
    }
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input
        {...register('email')}
        onChange={(e) => {
          // Clear server error when user starts typing
          clearErrors('email');
          register('email').onChange(e);
        }}
      />
      {errors.email && <p role="alert">{errors.email.message}</p>}

      <input type="password" {...register('password')} />
      {errors.password && <p role="alert">{errors.password.message}</p>}

      {/* Root-level server error */}
      {errors.root?.serverError && (
        <div role="alert" className="server-error">
          {errors.root.serverError.message}
        </div>
      )}

      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Creating account...' : 'Sign Up'}
      </button>
    </form>
  );
}
```

### 2.7 formState.dirtyFields for Partial Saves

When editing an existing record, you often want to send only the changed fields to the server (a PATCH, not a PUT). RHF tracks exactly which fields the user has modified:

```tsx
function ProfileEditor({ profile }: { profile: ProfileFormData }) {
  const {
    register,
    handleSubmit,
    formState: { dirtyFields, isDirty },
  } = useForm<ProfileFormData>({
    defaultValues: profile,
  });

  const onSubmit = async (data: ProfileFormData) => {
    // Build a partial update with only dirty fields
    const changedFields: Partial<ProfileFormData> = {};

    (Object.keys(dirtyFields) as Array<keyof ProfileFormData>).forEach((key) => {
      if (dirtyFields[key]) {
        changedFields[key] = data[key] as any;
      }
    });

    if (Object.keys(changedFields).length === 0) {
      // Nothing changed -- no need to hit the API
      return;
    }

    await api.patchProfile(changedFields);
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      {/* ... fields ... */}
      <button type="submit" disabled={!isDirty}>
        {isDirty ? 'Save Changes' : 'No Changes'}
      </button>
    </form>
  );
}
```

This is a real performance win on the backend too -- you are not sending unchanged fields over the wire, and your API can skip no-op updates.

---

## 3. ZOD ADVANCED PATTERNS

Basic Zod is `z.string().min(1)`. Advanced Zod is where you encode real business logic into your validation layer. Let's go deep.

### 3.1 Refinements (.refine and .superRefine)

`.refine` adds custom validation logic to any schema:

```typescript
// Simple refinement: custom validation with a message
const passwordSchema = z.string()
  .min(8, 'Password must be at least 8 characters')
  .refine(
    (val) => /[A-Z]/.test(val),
    'Password must contain at least one uppercase letter',
  )
  .refine(
    (val) => /[0-9]/.test(val),
    'Password must contain at least one number',
  )
  .refine(
    (val) => /[^A-Za-z0-9]/.test(val),
    'Password must contain at least one special character',
  );
```

`.superRefine` gives you full control over error reporting -- multiple errors, custom paths, and conditional logic:

```typescript
const signupSchema = z.object({
  password: z.string().min(8),
  confirmPassword: z.string(),
  email: z.string().email(),
  role: z.enum(['user', 'admin']),
  adminCode: z.string().optional(),
}).superRefine((data, ctx) => {
  // Cross-field validation: passwords must match
  if (data.password !== data.confirmPassword) {
    ctx.addIssue({
      code: z.ZodIssueCode.custom,
      message: 'Passwords do not match',
      path: ['confirmPassword'], // Error appears on confirmPassword field
    });
  }

  // Conditional validation: admins need an admin code
  if (data.role === 'admin' && !data.adminCode) {
    ctx.addIssue({
      code: z.ZodIssueCode.custom,
      message: 'Admin code is required for admin accounts',
      path: ['adminCode'],
    });
  }

  // Multiple errors on the same field
  if (data.password.includes(data.email.split('@')[0])) {
    ctx.addIssue({
      code: z.ZodIssueCode.custom,
      message: 'Password cannot contain your email username',
      path: ['password'],
    });
  }
});
```

**When to use which:**
- `.refine`: Single validation rule, single error message, on a single field
- `.superRefine`: Cross-field validation, multiple errors, conditional logic, custom paths

### 3.2 Transforms (.transform)

Transforms let you normalize data during validation. The input type and output type can differ:

```typescript
// Transform: clean up data during validation
const contactSchema = z.object({
  name: z.string()
    .min(1)
    .transform((val) => val.trim()), // Remove whitespace

  phone: z.string()
    .min(10)
    .transform((val) => val.replace(/[\s\-\(\)]/g, '')), // Strip formatting

  email: z.string()
    .email()
    .transform((val) => val.toLowerCase()), // Normalize case

  website: z.string()
    .url()
    .optional()
    .or(z.literal(''))
    .transform((val) => val || undefined), // Convert empty string to undefined

  age: z.string()
    .transform((val) => parseInt(val, 10)) // String from input → number
    .pipe(z.number().min(13).max(120)),    // Then validate the number
});

// Input type:  { name: string; phone: string; email: string; ... }
// Output type: { name: string; phone: string; email: string; age: number; ... }
type ContactInput = z.input<typeof contactSchema>;
type ContactOutput = z.output<typeof contactSchema>;
```

**The `.pipe` pattern is critical.** When you transform a string to a number, the `.min(13)` check needs to run on the *output* of the transform, not the input. `.pipe` chains a second schema that validates the transformed value.

### 3.3 Discriminated Unions for Polymorphic Forms

This is one of Zod's most powerful features for forms. When your form shape changes based on a "type" field, use `z.discriminatedUnion`:

```typescript
// A payment form that changes based on payment method
const creditCardSchema = z.object({
  method: z.literal('credit_card'),
  cardNumber: z.string().regex(/^\d{16}$/, 'Card number must be 16 digits'),
  expiryMonth: z.number().min(1).max(12),
  expiryYear: z.number().min(2026),
  cvv: z.string().regex(/^\d{3,4}$/, 'CVV must be 3 or 4 digits'),
  cardholderName: z.string().min(1, 'Cardholder name is required'),
});

const bankTransferSchema = z.object({
  method: z.literal('bank_transfer'),
  bankName: z.string().min(1, 'Bank name is required'),
  accountNumber: z.string().min(1, 'Account number is required'),
  routingNumber: z.string().regex(/^\d{9}$/, 'Routing number must be 9 digits'),
});

const paypalSchema = z.object({
  method: z.literal('paypal'),
  paypalEmail: z.string().email('Please enter a valid PayPal email'),
});

// The discriminated union: method field determines which schema applies
const paymentSchema = z.discriminatedUnion('method', [
  creditCardSchema,
  bankTransferSchema,
  paypalSchema,
]);

type PaymentFormData = z.infer<typeof paymentSchema>;
// PaymentFormData is a proper union type:
// | { method: 'credit_card'; cardNumber: string; ... }
// | { method: 'bank_transfer'; bankName: string; ... }
// | { method: 'paypal'; paypalEmail: string; }
```

Now the form component adapts to the selected payment method:

```tsx
function PaymentForm() {
  const { register, watch, handleSubmit, formState: { errors } } = useForm<PaymentFormData>({
    resolver: zodResolver(paymentSchema),
    defaultValues: { method: 'credit_card' } as any,
  });

  const method = watch('method');

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <select {...register('method')}>
        <option value="credit_card">Credit Card</option>
        <option value="bank_transfer">Bank Transfer</option>
        <option value="paypal">PayPal</option>
      </select>

      {method === 'credit_card' && (
        <>
          <input {...register('cardNumber')} placeholder="Card Number" />
          {errors.cardNumber && <p role="alert">{errors.cardNumber.message}</p>}
          <input {...register('cardholderName')} placeholder="Name on Card" />
          <div className="flex gap-2">
            <input
              type="number"
              {...register('expiryMonth', { valueAsNumber: true })}
              placeholder="MM"
            />
            <input
              type="number"
              {...register('expiryYear', { valueAsNumber: true })}
              placeholder="YYYY"
            />
            <input {...register('cvv')} placeholder="CVV" />
          </div>
        </>
      )}

      {method === 'bank_transfer' && (
        <>
          <input {...register('bankName')} placeholder="Bank Name" />
          <input {...register('accountNumber')} placeholder="Account Number" />
          <input {...register('routingNumber')} placeholder="Routing Number" />
        </>
      )}

      {method === 'paypal' && (
        <>
          <input {...register('paypalEmail')} placeholder="PayPal Email" />
          {errors.paypalEmail && <p role="alert">{errors.paypalEmail.message}</p>}
        </>
      )}

      <button type="submit">Pay</button>
    </form>
  );
}
```

**Why discriminated unions, not conditional `.refine`?** Because discriminated unions give you proper TypeScript narrowing. When you check `method === 'credit_card'`, TypeScript knows the other fields exist. With `.refine`, you just get a boolean -- no type narrowing.

### 3.4 Async Validation (Username Availability)

Real-world forms often need to check something against the server during validation: is this username taken? Is this coupon code valid? Zod supports async refinements:

```typescript
const usernameSchema = z.string()
  .min(3, 'Username must be at least 3 characters')
  .max(20, 'Username must be 20 characters or less')
  .regex(/^[a-zA-Z0-9_]+$/, 'Username can only contain letters, numbers, and underscores')
  .refine(
    async (username) => {
      // Call your API to check availability
      const response = await fetch(`/api/check-username?username=${username}`);
      const { available } = await response.json();
      return available;
    },
    'This username is already taken',
  );
```

**Warning:** Async validation in Zod works with RHF, but you need to be careful about timing. The validation will fire based on your `mode` setting. With `mode: 'onBlur'`, the async check runs when the user leaves the field -- that is usually what you want. With `mode: 'onChange'`, it would fire on every keystroke, hammering your API.

For better control, debounce the check yourself:

```tsx
function UsernameField({ control }: { control: Control<SignupFormData> }) {
  const [checking, setChecking] = useState(false);
  const [available, setAvailable] = useState<boolean | null>(null);

  const username = useWatch({ control, name: 'username' });

  useEffect(() => {
    if (!username || username.length < 3) {
      setAvailable(null);
      return;
    }

    setChecking(true);
    const timeoutId = setTimeout(async () => {
      try {
        const res = await fetch(`/api/check-username?username=${username}`);
        const { available: isAvailable } = await res.json();
        setAvailable(isAvailable);
      } catch {
        setAvailable(null);
      } finally {
        setChecking(false);
      }
    }, 500); // 500ms debounce

    return () => clearTimeout(timeoutId);
  }, [username]);

  return (
    <div>
      <input {...control.register('username')} />
      {checking && <span>Checking...</span>}
      {available === true && <span className="text-green-600">Available!</span>}
      {available === false && <span className="text-red-600">Taken</span>}
    </div>
  );
}
```

### 3.5 Conditional Schemas (Dynamic Validation)

Sometimes the validation rules themselves change based on context (not just the form shape):

```typescript
// Schema that adapts based on the user's country
function createAddressSchema(country: string) {
  const base = z.object({
    street: z.string().min(1, 'Street is required'),
    city: z.string().min(1, 'City is required'),
    country: z.literal(country),
  });

  switch (country) {
    case 'US':
      return base.extend({
        state: z.string().length(2, 'Please select a state'),
        zipCode: z.string().regex(/^\d{5}(-\d{4})?$/, 'Invalid ZIP code'),
      });
    case 'CA':
      return base.extend({
        province: z.string().min(1, 'Please select a province'),
        postalCode: z.string().regex(
          /^[A-Za-z]\d[A-Za-z]\s?\d[A-Za-z]\d$/,
          'Invalid postal code (format: A1A 1A1)',
        ),
      });
    case 'GB':
      return base.extend({
        county: z.string().optional(),
        postcode: z.string().regex(
          /^[A-Z]{1,2}\d[A-Z\d]?\s?\d[A-Z]{2}$/i,
          'Invalid postcode',
        ),
      });
    default:
      return base.extend({
        region: z.string().optional(),
        postalCode: z.string().optional(),
      });
  }
}

// In the component: recreate the resolver when country changes
function AddressForm() {
  const [country, setCountry] = useState('US');

  const schema = useMemo(() => createAddressSchema(country), [country]);

  const form = useForm({
    resolver: zodResolver(schema),
    defaultValues: { country, street: '', city: '' },
  });

  // When country changes, update the form and schema
  const handleCountryChange = (newCountry: string) => {
    setCountry(newCountry);
    form.setValue('country', newCountry);
    form.clearErrors(); // Clear errors from the old schema
  };

  // ...
}
```

---

## 4. MULTI-STEP WIZARDS

Multi-step forms are the most architecturally demanding form pattern. The challenges:

1. **State persistence**: Data entered in step 1 must survive navigation to step 3 and back
2. **Per-step validation**: Each step should validate independently
3. **Free navigation**: Users should be able to jump between completed steps
4. **Progress persistence**: Refreshing the page should not lose progress
5. **Summary screen**: A final review step that shows all data before submission

### 4.1 The State Machine Approach

A wizard is a state machine. Each step is a state, and transitions between states have guards (validation must pass before moving forward). Here is the approach with `useReducer`:

```typescript
// types/wizard.ts

// Each step has its own schema
import { z } from 'zod';

export const step1Schema = z.object({
  companyName: z.string().min(1, 'Company name is required'),
  companySize: z.enum(['1-10', '11-50', '51-200', '201-1000', '1000+']),
  industry: z.string().min(1, 'Industry is required'),
});

export const step2Schema = z.object({
  teamMembers: z.array(
    z.object({
      name: z.string().min(1, 'Name is required'),
      email: z.string().email('Invalid email'),
      role: z.enum(['admin', 'member', 'viewer']),
    }),
  ).min(1, 'At least one team member is required'),
});

export const step3Schema = z.object({
  plan: z.enum(['starter', 'professional', 'enterprise']),
  billingCycle: z.enum(['monthly', 'annual']),
  // Conditional: only required for enterprise
  enterpriseContact: z.string().optional(),
});

export const step4Schema = z.object({
  logo: z.string().optional(), // URL after upload
});

// Combined schema for final submission
export const onboardingSchema = step1Schema
  .merge(step2Schema)
  .merge(step3Schema)
  .merge(step4Schema);

export type OnboardingData = z.infer<typeof onboardingSchema>;

// Step definitions
export const STEPS = [
  { id: 'company', label: 'Company Info', schema: step1Schema },
  { id: 'team', label: 'Team Members', schema: step2Schema },
  { id: 'plan', label: 'Choose Plan', schema: step3Schema },
  { id: 'branding', label: 'Branding', schema: step4Schema },
  { id: 'review', label: 'Review', schema: null }, // No validation on review
] as const;
```

```typescript
// hooks/useWizard.ts
import { useReducer, useCallback, useEffect } from 'react';

type WizardState = {
  currentStep: number;
  completedSteps: Set<number>;
  data: Partial<OnboardingData>;
  direction: 'forward' | 'backward';
};

type WizardAction =
  | { type: 'GO_TO_STEP'; step: number }
  | { type: 'NEXT' }
  | { type: 'PREVIOUS' }
  | { type: 'UPDATE_DATA'; data: Partial<OnboardingData> }
  | { type: 'MARK_COMPLETED'; step: number }
  | { type: 'RESTORE'; state: WizardState };

function wizardReducer(state: WizardState, action: WizardAction): WizardState {
  switch (action.type) {
    case 'NEXT':
      return {
        ...state,
        currentStep: Math.min(state.currentStep + 1, STEPS.length - 1),
        direction: 'forward',
      };
    case 'PREVIOUS':
      return {
        ...state,
        currentStep: Math.max(state.currentStep - 1, 0),
        direction: 'backward',
      };
    case 'GO_TO_STEP':
      // Only allow jumping to completed steps or the next step
      if (
        action.step <= state.currentStep ||
        state.completedSteps.has(action.step - 1)
      ) {
        return {
          ...state,
          currentStep: action.step,
          direction: action.step > state.currentStep ? 'forward' : 'backward',
        };
      }
      return state;
    case 'UPDATE_DATA':
      return {
        ...state,
        data: { ...state.data, ...action.data },
      };
    case 'MARK_COMPLETED':
      return {
        ...state,
        completedSteps: new Set([...state.completedSteps, action.step]),
      };
    case 'RESTORE':
      return {
        ...action.state,
        completedSteps: new Set(action.state.completedSteps),
      };
    default:
      return state;
  }
}

const STORAGE_KEY = 'onboarding-wizard-state';

export function useWizard() {
  const [state, dispatch] = useReducer(wizardReducer, {
    currentStep: 0,
    completedSteps: new Set<number>(),
    data: {},
    direction: 'forward',
  });

  // Restore from storage on mount
  useEffect(() => {
    try {
      const saved = localStorage.getItem(STORAGE_KEY);
      if (saved) {
        const parsed = JSON.parse(saved);
        dispatch({
          type: 'RESTORE',
          state: {
            ...parsed,
            completedSteps: new Set(parsed.completedSteps),
          },
        });
      }
    } catch {
      // Ignore corrupt storage
    }
  }, []);

  // Persist to storage on every change
  useEffect(() => {
    try {
      localStorage.setItem(
        STORAGE_KEY,
        JSON.stringify({
          ...state,
          completedSteps: [...state.completedSteps],
        }),
      );
    } catch {
      // Storage full or unavailable
    }
  }, [state]);

  const next = useCallback(() => dispatch({ type: 'NEXT' }), []);
  const previous = useCallback(() => dispatch({ type: 'PREVIOUS' }), []);
  const goToStep = useCallback(
    (step: number) => dispatch({ type: 'GO_TO_STEP', step }),
    [],
  );
  const updateData = useCallback(
    (data: Partial<OnboardingData>) => dispatch({ type: 'UPDATE_DATA', data }),
    [],
  );
  const markCompleted = useCallback(
    (step: number) => dispatch({ type: 'MARK_COMPLETED', step }),
    [],
  );
  const clearStorage = useCallback(() => {
    localStorage.removeItem(STORAGE_KEY);
  }, []);

  return {
    ...state,
    next,
    previous,
    goToStep,
    updateData,
    markCompleted,
    clearStorage,
    isFirstStep: state.currentStep === 0,
    isLastStep: state.currentStep === STEPS.length - 1,
    progress: ((state.currentStep + 1) / STEPS.length) * 100,
  };
}
```

### 4.2 The Wizard Container

```tsx
// components/OnboardingWizard.tsx
import { AnimatePresence, motion } from 'framer-motion';

export function OnboardingWizard() {
  const wizard = useWizard();

  const handleStepSubmit = async (stepData: any) => {
    wizard.updateData(stepData);
    wizard.markCompleted(wizard.currentStep);
    wizard.next();
  };

  const handleFinalSubmit = async () => {
    try {
      await api.completeOnboarding(wizard.data as OnboardingData);
      wizard.clearStorage();
      router.push('/dashboard');
    } catch (error) {
      // Handle server errors
    }
  };

  return (
    <div className="wizard-container">
      {/* Progress indicator */}
      <nav aria-label="Onboarding progress">
        <ol className="step-indicators">
          {STEPS.map((step, index) => (
            <li key={step.id}>
              <button
                onClick={() => wizard.goToStep(index)}
                disabled={
                  index > wizard.currentStep &&
                  !wizard.completedSteps.has(index - 1)
                }
                aria-current={index === wizard.currentStep ? 'step' : undefined}
                className={
                  index === wizard.currentStep
                    ? 'active'
                    : wizard.completedSteps.has(index)
                      ? 'completed'
                      : 'pending'
                }
              >
                <span className="step-number">{index + 1}</span>
                <span className="step-label">{step.label}</span>
              </button>
            </li>
          ))}
        </ol>
        <div
          className="progress-bar"
          role="progressbar"
          aria-valuenow={wizard.progress}
          aria-valuemin={0}
          aria-valuemax={100}
          style={{ width: `${wizard.progress}%` }}
        />
      </nav>

      {/* Animated step transitions */}
      <AnimatePresence mode="wait">
        <motion.div
          key={wizard.currentStep}
          initial={{
            x: wizard.direction === 'forward' ? 100 : -100,
            opacity: 0,
          }}
          animate={{ x: 0, opacity: 1 }}
          exit={{
            x: wizard.direction === 'forward' ? -100 : 100,
            opacity: 0,
          }}
          transition={{ duration: 0.2 }}
        >
          {wizard.currentStep === 0 && (
            <CompanyInfoStep
              defaultValues={wizard.data}
              onSubmit={handleStepSubmit}
              onBack={wizard.previous}
              isFirstStep={wizard.isFirstStep}
            />
          )}
          {wizard.currentStep === 1 && (
            <TeamMembersStep
              defaultValues={wizard.data}
              onSubmit={handleStepSubmit}
              onBack={wizard.previous}
            />
          )}
          {wizard.currentStep === 2 && (
            <PlanSelectionStep
              defaultValues={wizard.data}
              onSubmit={handleStepSubmit}
              onBack={wizard.previous}
            />
          )}
          {wizard.currentStep === 3 && (
            <BrandingStep
              defaultValues={wizard.data}
              onSubmit={handleStepSubmit}
              onBack={wizard.previous}
            />
          )}
          {wizard.currentStep === 4 && (
            <ReviewStep
              data={wizard.data as OnboardingData}
              onSubmit={handleFinalSubmit}
              onBack={wizard.previous}
              onEditStep={wizard.goToStep}
            />
          )}
        </motion.div>
      </AnimatePresence>
    </div>
  );
}
```

### 4.3 Individual Step Components

Each step is its own form with its own schema. The critical pattern: **default values come from the wizard state, not from the component**.

```tsx
// components/steps/CompanyInfoStep.tsx
function CompanyInfoStep({
  defaultValues,
  onSubmit,
  onBack,
  isFirstStep,
}: StepProps) {
  const {
    register,
    handleSubmit,
    formState: { errors, isValid },
  } = useForm({
    resolver: zodResolver(step1Schema),
    defaultValues: {
      companyName: defaultValues.companyName ?? '',
      companySize: defaultValues.companySize ?? '1-10',
      industry: defaultValues.industry ?? '',
    },
    mode: 'onBlur',
  });

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <h2>Tell us about your company</h2>

      <div>
        <label htmlFor="companyName">Company Name</label>
        <input id="companyName" {...register('companyName')} />
        {errors.companyName && (
          <p role="alert">{errors.companyName.message}</p>
        )}
      </div>

      <div>
        <label htmlFor="companySize">Company Size</label>
        <select id="companySize" {...register('companySize')}>
          <option value="1-10">1-10 employees</option>
          <option value="11-50">11-50 employees</option>
          <option value="51-200">51-200 employees</option>
          <option value="201-1000">201-1000 employees</option>
          <option value="1000+">1000+ employees</option>
        </select>
      </div>

      <div>
        <label htmlFor="industry">Industry</label>
        <input id="industry" {...register('industry')} />
        {errors.industry && (
          <p role="alert">{errors.industry.message}</p>
        )}
      </div>

      <div className="step-actions">
        {!isFirstStep && (
          <button type="button" onClick={onBack}>
            Back
          </button>
        )}
        <button type="submit">Continue</button>
      </div>
    </form>
  );
}
```

### 4.4 The "Back Doesn't Lose Data" Problem

This is the number one bug in wizard implementations. Here is what goes wrong:

1. User fills out step 1, clicks "Next"
2. Step 2 renders, user fills it out, clicks "Back"
3. Step 1 re-mounts. If `defaultValues` are empty strings, the user's data is gone.

The fix is in the pattern above: **default values always come from the wizard's accumulated state**. When the wizard's `updateData` is called on each step's submit, the data is stored. When the user goes back, the step receives that stored data as `defaultValues`.

But there is a subtlety: what if the user edits step 1, does NOT click "Next" (no submit), and clicks "Back" from step 2? The step 1 edits were never submitted. They are lost.

The solution: save on navigation, not just on submit:

```tsx
function CompanyInfoStep({ defaultValues, onSubmit, onBack, onSaveProgress }: StepProps) {
  const form = useForm({
    resolver: zodResolver(step1Schema),
    defaultValues: { /* ... */ },
  });

  // Save current values when navigating away (even without validation)
  const handleBack = () => {
    const currentValues = form.getValues();
    onSaveProgress(currentValues); // Save even if invalid
    onBack();
  };

  // Full validation only on "Next"
  const handleNext = form.handleSubmit((validData) => {
    onSubmit(validData);
  });

  return (
    <form onSubmit={handleNext}>
      {/* ... fields ... */}
      <button type="button" onClick={handleBack}>Back</button>
      <button type="submit">Continue</button>
    </form>
  );
}
```

### 4.5 The Review Step

```tsx
function ReviewStep({
  data,
  onSubmit,
  onBack,
  onEditStep,
}: {
  data: OnboardingData;
  onSubmit: () => Promise<void>;
  onBack: () => void;
  onEditStep: (step: number) => void;
}) {
  const [submitting, setSubmitting] = useState(false);

  const handleSubmit = async () => {
    setSubmitting(true);
    try {
      await onSubmit();
    } catch {
      setSubmitting(false);
    }
  };

  return (
    <div>
      <h2>Review Your Information</h2>

      <section>
        <div className="section-header">
          <h3>Company Info</h3>
          <button type="button" onClick={() => onEditStep(0)}>
            Edit
          </button>
        </div>
        <dl>
          <dt>Company Name</dt>
          <dd>{data.companyName}</dd>
          <dt>Size</dt>
          <dd>{data.companySize}</dd>
          <dt>Industry</dt>
          <dd>{data.industry}</dd>
        </dl>
      </section>

      <section>
        <div className="section-header">
          <h3>Team Members</h3>
          <button type="button" onClick={() => onEditStep(1)}>
            Edit
          </button>
        </div>
        <ul>
          {data.teamMembers.map((member, i) => (
            <li key={i}>
              {member.name} ({member.email}) — {member.role}
            </li>
          ))}
        </ul>
      </section>

      <section>
        <div className="section-header">
          <h3>Plan</h3>
          <button type="button" onClick={() => onEditStep(2)}>
            Edit
          </button>
        </div>
        <dl>
          <dt>Plan</dt>
          <dd>{data.plan}</dd>
          <dt>Billing</dt>
          <dd>{data.billingCycle}</dd>
        </dl>
      </section>

      <div className="step-actions">
        <button type="button" onClick={onBack}>
          Back
        </button>
        <button onClick={handleSubmit} disabled={submitting}>
          {submitting ? 'Setting up your account...' : 'Complete Setup'}
        </button>
      </div>
    </div>
  );
}
```

---

## 5. DYNAMIC FORMS

Dynamic forms are forms whose fields are not known at compile time. They come from a JSON schema, a CMS, an API response, or a form builder. The key architectural pattern is a **field registry** that maps schema types to React components.

### 5.1 The Field Registry Pattern

```typescript
// types/dynamic-form.ts
export type FieldType =
  | 'text'
  | 'email'
  | 'number'
  | 'textarea'
  | 'select'
  | 'multiselect'
  | 'checkbox'
  | 'radio'
  | 'date'
  | 'file'
  | 'group';

export interface FieldDefinition {
  name: string;
  type: FieldType;
  label: string;
  placeholder?: string;
  required?: boolean;
  validation?: {
    min?: number;
    max?: number;
    minLength?: number;
    maxLength?: number;
    pattern?: string;
    patternMessage?: string;
  };
  options?: Array<{ label: string; value: string }>;
  dependsOn?: {
    field: string;
    value: string | string[];
  };
  fields?: FieldDefinition[]; // For 'group' type
  defaultValue?: any;
}

export interface FormDefinition {
  id: string;
  title: string;
  description?: string;
  fields: FieldDefinition[];
  submitLabel?: string;
}
```

### 5.2 Schema Generation from JSON

Convert a JSON form definition to a Zod schema at runtime:

```typescript
// utils/schema-from-definition.ts
import { z, ZodTypeAny } from 'zod';
import { FieldDefinition, FormDefinition } from '@/types/dynamic-form';

function fieldToZod(field: FieldDefinition): ZodTypeAny {
  let schema: ZodTypeAny;

  switch (field.type) {
    case 'text':
    case 'email':
    case 'textarea': {
      let s = z.string();
      if (field.validation?.minLength) {
        s = s.min(field.validation.minLength, `Minimum ${field.validation.minLength} characters`);
      }
      if (field.validation?.maxLength) {
        s = s.max(field.validation.maxLength, `Maximum ${field.validation.maxLength} characters`);
      }
      if (field.type === 'email') {
        s = s.email('Please enter a valid email');
      }
      if (field.validation?.pattern) {
        s = s.regex(
          new RegExp(field.validation.pattern),
          field.validation.patternMessage ?? 'Invalid format',
        );
      }
      schema = field.required ? s.min(1, `${field.label} is required`) : s.optional();
      break;
    }

    case 'number': {
      let n = z.number({ invalid_type_error: `${field.label} must be a number` });
      if (field.validation?.min !== undefined) n = n.min(field.validation.min);
      if (field.validation?.max !== undefined) n = n.max(field.validation.max);
      schema = field.required ? n : n.optional();
      break;
    }

    case 'select':
    case 'radio': {
      const values = (field.options ?? []).map((o) => o.value);
      if (values.length > 0) {
        schema = z.enum(values as [string, ...string[]]);
        if (!field.required) schema = schema.optional();
      } else {
        schema = z.string().optional();
      }
      break;
    }

    case 'multiselect': {
      const values = (field.options ?? []).map((o) => o.value);
      schema = z.array(z.enum(values as [string, ...string[]]));
      if (field.required) {
        schema = (schema as z.ZodArray<any>).min(1, 'Please select at least one option');
      }
      break;
    }

    case 'checkbox':
      schema = field.required
        ? z.boolean().refine((v) => v === true, `${field.label} must be checked`)
        : z.boolean();
      break;

    case 'date': {
      let s = z.string();
      if (field.required) s = s.min(1, `${field.label} is required`);
      schema = field.required ? s : s.optional();
      break;
    }

    case 'file':
      schema = field.required
        ? z.string().min(1, `${field.label} is required`)
        : z.string().optional();
      break;

    case 'group':
      schema = z.object(
        (field.fields ?? []).reduce(
          (acc, subField) => {
            acc[subField.name] = fieldToZod(subField);
            return acc;
          },
          {} as Record<string, ZodTypeAny>,
        ),
      );
      break;

    default:
      schema = z.any();
  }

  return schema;
}

export function createSchemaFromDefinition(definition: FormDefinition): z.ZodObject<any> {
  const shape: Record<string, ZodTypeAny> = {};

  for (const field of definition.fields) {
    shape[field.name] = fieldToZod(field);
  }

  return z.object(shape);
}
```

### 5.3 The Dynamic Form Renderer

```tsx
// components/DynamicForm.tsx
import { useForm, Controller } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { FormDefinition, FieldDefinition } from '@/types/dynamic-form';
import { createSchemaFromDefinition } from '@/utils/schema-from-definition';

export function DynamicForm({
  definition,
  onSubmit,
  defaultValues,
}: {
  definition: FormDefinition;
  onSubmit: (data: Record<string, any>) => Promise<void>;
  defaultValues?: Record<string, any>;
}) {
  const schema = useMemo(() => createSchemaFromDefinition(definition), [definition]);

  const {
    register,
    control,
    handleSubmit,
    watch,
    formState: { errors, isSubmitting },
  } = useForm({
    resolver: zodResolver(schema),
    defaultValues: defaultValues ?? buildDefaultValues(definition.fields),
  });

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <h2>{definition.title}</h2>
      {definition.description && <p>{definition.description}</p>}

      {definition.fields.map((field) => (
        <DynamicField
          key={field.name}
          field={field}
          register={register}
          control={control}
          errors={errors}
          watch={watch}
        />
      ))}

      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Submitting...' : definition.submitLabel ?? 'Submit'}
      </button>
    </form>
  );
}

function DynamicField({
  field,
  register,
  control,
  errors,
  watch,
}: {
  field: FieldDefinition;
  register: any;
  control: any;
  errors: any;
  watch: any;
}) {
  // Handle conditional visibility
  if (field.dependsOn) {
    const dependentValue = watch(field.dependsOn.field);
    const expectedValues = Array.isArray(field.dependsOn.value)
      ? field.dependsOn.value
      : [field.dependsOn.value];

    if (!expectedValues.includes(dependentValue)) {
      return null; // Field is hidden
    }
  }

  const error = errors[field.name];

  switch (field.type) {
    case 'text':
    case 'email':
      return (
        <div>
          <label htmlFor={field.name}>{field.label}</label>
          <input
            id={field.name}
            type={field.type}
            placeholder={field.placeholder}
            {...register(field.name)}
            aria-invalid={!!error}
          />
          {error && <p role="alert">{error.message}</p>}
        </div>
      );

    case 'number':
      return (
        <div>
          <label htmlFor={field.name}>{field.label}</label>
          <input
            id={field.name}
            type="number"
            {...register(field.name, { valueAsNumber: true })}
            aria-invalid={!!error}
          />
          {error && <p role="alert">{error.message}</p>}
        </div>
      );

    case 'textarea':
      return (
        <div>
          <label htmlFor={field.name}>{field.label}</label>
          <textarea
            id={field.name}
            placeholder={field.placeholder}
            {...register(field.name)}
            aria-invalid={!!error}
          />
          {error && <p role="alert">{error.message}</p>}
        </div>
      );

    case 'select':
      return (
        <div>
          <label htmlFor={field.name}>{field.label}</label>
          <select id={field.name} {...register(field.name)}>
            <option value="">Select...</option>
            {field.options?.map((opt) => (
              <option key={opt.value} value={opt.value}>
                {opt.label}
              </option>
            ))}
          </select>
          {error && <p role="alert">{error.message}</p>}
        </div>
      );

    case 'checkbox':
      return (
        <div>
          <label>
            <input type="checkbox" {...register(field.name)} />
            {field.label}
          </label>
          {error && <p role="alert">{error.message}</p>}
        </div>
      );

    case 'radio':
      return (
        <fieldset>
          <legend>{field.label}</legend>
          {field.options?.map((opt) => (
            <label key={opt.value}>
              <input
                type="radio"
                value={opt.value}
                {...register(field.name)}
              />
              {opt.label}
            </label>
          ))}
          {error && <p role="alert">{error.message}</p>}
        </fieldset>
      );

    case 'group':
      return (
        <fieldset>
          <legend>{field.label}</legend>
          {field.fields?.map((subField) => (
            <DynamicField
              key={`${field.name}.${subField.name}`}
              field={{ ...subField, name: `${field.name}.${subField.name}` }}
              register={register}
              control={control}
              errors={errors[field.name] || {}}
              watch={watch}
            />
          ))}
        </fieldset>
      );

    default:
      return null;
  }
}

function buildDefaultValues(fields: FieldDefinition[]): Record<string, any> {
  const defaults: Record<string, any> = {};
  for (const field of fields) {
    if (field.defaultValue !== undefined) {
      defaults[field.name] = field.defaultValue;
    } else {
      switch (field.type) {
        case 'checkbox':
          defaults[field.name] = false;
          break;
        case 'number':
          defaults[field.name] = 0;
          break;
        case 'multiselect':
          defaults[field.name] = [];
          break;
        case 'group':
          defaults[field.name] = field.fields
            ? buildDefaultValues(field.fields)
            : {};
          break;
        default:
          defaults[field.name] = '';
      }
    }
  }
  return defaults;
}
```

### 5.4 Example: CMS-Driven Form

Here is what a form definition from your CMS might look like:

```json
{
  "id": "contact-sales",
  "title": "Contact Sales",
  "description": "Tell us about your needs and we'll get back to you within 24 hours.",
  "fields": [
    {
      "name": "fullName",
      "type": "text",
      "label": "Full Name",
      "required": true,
      "validation": { "minLength": 2, "maxLength": 100 }
    },
    {
      "name": "email",
      "type": "email",
      "label": "Work Email",
      "required": true
    },
    {
      "name": "companySize",
      "type": "select",
      "label": "Company Size",
      "required": true,
      "options": [
        { "label": "1-50 employees", "value": "small" },
        { "label": "51-500 employees", "value": "medium" },
        { "label": "500+ employees", "value": "enterprise" }
      ]
    },
    {
      "name": "enterpriseRequirements",
      "type": "textarea",
      "label": "Enterprise Requirements",
      "placeholder": "Tell us about your specific needs...",
      "dependsOn": { "field": "companySize", "value": "enterprise" },
      "required": false
    },
    {
      "name": "budget",
      "type": "radio",
      "label": "Budget Range",
      "required": true,
      "options": [
        { "label": "Under $10k/year", "value": "under-10k" },
        { "label": "$10k-$50k/year", "value": "10k-50k" },
        { "label": "$50k+/year", "value": "50k-plus" }
      ]
    },
    {
      "name": "agreeToTerms",
      "type": "checkbox",
      "label": "I agree to the terms of service",
      "required": true
    }
  ],
  "submitLabel": "Request Demo"
}
```

The `DynamicForm` component renders this automatically, complete with conditional fields (enterprise requirements only appears when company size is "enterprise"), validation, and proper accessibility attributes.

---

## 6. FILE UPLOADS

File uploads are the form feature that trips up the most developers. The challenges: progress tracking, preview before upload, size limits, type validation, and choosing between direct uploads and signed URLs.

### 6.1 The Upload Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                FILE UPLOAD STRATEGIES                        │
│                                                              │
│  STRATEGY 1: Direct Upload (small files < 5MB)              │
│  Client → Your API → Storage                                │
│  Simple but your server is a bottleneck                      │
│                                                              │
│  STRATEGY 2: Signed URL Upload (large files, recommended)   │
│  Client → Your API (get signed URL) → Storage directly      │
│  Client never sends the file through your server             │
│                                                              │
│  STRATEGY 3: Vercel Blob (if on Vercel)                     │
│  Client → Vercel Blob API → Vercel Edge Storage             │
│  Zero config, built-in CDN                                   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 6.2 Signed URL Upload with Progress

```typescript
// api/upload.ts -- Server-side: generate signed URL
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';
import { v4 as uuid } from 'uuid';

const s3 = new S3Client({ region: process.env.AWS_REGION });

export async function generateUploadUrl(
  contentType: string,
  folder: string = 'uploads',
) {
  const key = `${folder}/${uuid()}-${Date.now()}`;

  const command = new PutObjectCommand({
    Bucket: process.env.S3_BUCKET!,
    Key: key,
    ContentType: contentType,
  });

  const signedUrl = await getSignedUrl(s3, command, { expiresIn: 300 }); // 5 minutes

  return {
    uploadUrl: signedUrl,
    fileUrl: `https://${process.env.S3_BUCKET}.s3.amazonaws.com/${key}`,
    key,
  };
}
```

```typescript
// utils/upload.ts -- Client-side: upload with progress
export interface UploadProgress {
  loaded: number;
  total: number;
  percentage: number;
}

export async function uploadWithProgress(
  file: File,
  uploadUrl: string,
  onProgress?: (progress: UploadProgress) => void,
): Promise<void> {
  return new Promise((resolve, reject) => {
    const xhr = new XMLHttpRequest();

    xhr.upload.addEventListener('progress', (event) => {
      if (event.lengthComputable && onProgress) {
        onProgress({
          loaded: event.loaded,
          total: event.total,
          percentage: Math.round((event.loaded / event.total) * 100),
        });
      }
    });

    xhr.addEventListener('load', () => {
      if (xhr.status >= 200 && xhr.status < 300) {
        resolve();
      } else {
        reject(new Error(`Upload failed with status ${xhr.status}`));
      }
    });

    xhr.addEventListener('error', () => reject(new Error('Upload failed')));
    xhr.addEventListener('abort', () => reject(new Error('Upload cancelled')));

    xhr.open('PUT', uploadUrl);
    xhr.setRequestHeader('Content-Type', file.type);
    xhr.send(file);
  });
}
```

### 6.3 File Upload Component with React Hook Form

```tsx
// components/FileUploadField.tsx
import { useState, useRef, useCallback } from 'react';
import { useController, Control } from 'react-hook-form';
import { uploadWithProgress, UploadProgress } from '@/utils/upload';

interface FileUploadFieldProps {
  name: string;
  control: Control<any>;
  accept?: string;
  maxSizeMB?: number;
  label: string;
}

export function FileUploadField({
  name,
  control,
  accept = 'image/*',
  maxSizeMB = 10,
  label,
}: FileUploadFieldProps) {
  const { field, fieldState } = useController({ name, control });
  const [preview, setPreview] = useState<string | null>(field.value || null);
  const [uploading, setUploading] = useState(false);
  const [progress, setProgress] = useState<UploadProgress | null>(null);
  const [dragOver, setDragOver] = useState(false);
  const inputRef = useRef<HTMLInputElement>(null);

  const handleFile = useCallback(
    async (file: File) => {
      // Client-side validation
      if (file.size > maxSizeMB * 1024 * 1024) {
        alert(`File must be smaller than ${maxSizeMB}MB`);
        return;
      }

      // Show preview immediately
      const objectUrl = URL.createObjectURL(file);
      setPreview(objectUrl);

      setUploading(true);
      setProgress({ loaded: 0, total: file.size, percentage: 0 });

      try {
        // 1. Get signed URL from your API
        const { uploadUrl, fileUrl } = await fetch('/api/upload/signed-url', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ contentType: file.type, folder: 'avatars' }),
        }).then((r) => r.json());

        // 2. Upload directly to storage
        await uploadWithProgress(file, uploadUrl, setProgress);

        // 3. Set the form value to the permanent URL
        field.onChange(fileUrl);
        setPreview(fileUrl);
      } catch (error) {
        setPreview(null);
        field.onChange('');
        alert('Upload failed. Please try again.');
      } finally {
        setUploading(false);
        setProgress(null);
        URL.revokeObjectURL(objectUrl);
      }
    },
    [field, maxSizeMB],
  );

  const handleDrop = useCallback(
    (e: React.DragEvent) => {
      e.preventDefault();
      setDragOver(false);
      const file = e.dataTransfer.files[0];
      if (file) handleFile(file);
    },
    [handleFile],
  );

  const handleInputChange = useCallback(
    (e: React.ChangeEvent<HTMLInputElement>) => {
      const file = e.target.files?.[0];
      if (file) handleFile(file);
    },
    [handleFile],
  );

  return (
    <div>
      <label>{label}</label>
      <div
        className={`upload-zone ${dragOver ? 'drag-over' : ''}`}
        onDragOver={(e) => { e.preventDefault(); setDragOver(true); }}
        onDragLeave={() => setDragOver(false)}
        onDrop={handleDrop}
        onClick={() => inputRef.current?.click()}
        role="button"
        tabIndex={0}
        onKeyDown={(e) => {
          if (e.key === 'Enter' || e.key === ' ') {
            inputRef.current?.click();
          }
        }}
        aria-label={`Upload ${label}`}
      >
        <input
          ref={inputRef}
          type="file"
          accept={accept}
          onChange={handleInputChange}
          className="hidden"
          aria-hidden="true"
        />

        {preview ? (
          <div className="preview">
            <img src={preview} alt="Preview" className="preview-image" />
            {uploading && progress && (
              <div className="upload-overlay">
                <div
                  className="progress-bar"
                  style={{ width: `${progress.percentage}%` }}
                />
                <span>{progress.percentage}%</span>
              </div>
            )}
          </div>
        ) : (
          <div className="upload-placeholder">
            <p>Drop a file here, or click to browse</p>
            <p className="text-sm text-gray-500">
              Max size: {maxSizeMB}MB
            </p>
          </div>
        )}
      </div>

      {field.value && !uploading && (
        <button
          type="button"
          onClick={() => {
            field.onChange('');
            setPreview(null);
          }}
        >
          Remove
        </button>
      )}

      {fieldState.error && (
        <p role="alert">{fieldState.error.message}</p>
      )}
    </div>
  );
}
```

### 6.4 React Native File Upload (expo-image-picker)

```tsx
// components/ImagePickerField.native.tsx
import * as ImagePicker from 'expo-image-picker';
import { useController, Control } from 'react-hook-form';
import { Image, Pressable, Text, View, ActivityIndicator } from 'react-native';
import { useState } from 'react';

export function ImagePickerField({
  name,
  control,
  label,
}: {
  name: string;
  control: Control<any>;
  label: string;
}) {
  const { field, fieldState } = useController({ name, control });
  const [uploading, setUploading] = useState(false);

  const pickImage = async () => {
    // Request permissions
    const { status } = await ImagePicker.requestMediaLibraryPermissionsAsync();
    if (status !== 'granted') {
      alert('Camera roll permission is required');
      return;
    }

    const result = await ImagePicker.launchImageLibraryAsync({
      mediaTypes: ImagePicker.MediaTypeOptions.Images,
      allowsEditing: true,
      aspect: [1, 1],
      quality: 0.8,
    });

    if (result.canceled) return;

    const asset = result.assets[0];
    setUploading(true);

    try {
      // Get signed URL
      const { uploadUrl, fileUrl } = await api.getUploadUrl({
        contentType: 'image/jpeg',
        folder: 'avatars',
      });

      // Upload using fetch (React Native supports FormData natively)
      const response = await fetch(asset.uri);
      const blob = await response.blob();

      await fetch(uploadUrl, {
        method: 'PUT',
        body: blob,
        headers: { 'Content-Type': 'image/jpeg' },
      });

      field.onChange(fileUrl);
    } catch (error) {
      alert('Upload failed');
    } finally {
      setUploading(false);
    }
  };

  return (
    <View>
      <Text>{label}</Text>
      <Pressable onPress={pickImage} disabled={uploading}>
        {field.value ? (
          <Image
            source={{ uri: field.value }}
            style={{ width: 100, height: 100, borderRadius: 50 }}
          />
        ) : (
          <View style={{
            width: 100,
            height: 100,
            borderRadius: 50,
            backgroundColor: '#e5e7eb',
            alignItems: 'center',
            justifyContent: 'center',
          }}>
            {uploading ? (
              <ActivityIndicator />
            ) : (
              <Text>+ Photo</Text>
            )}
          </View>
        )}
      </Pressable>
      {fieldState.error && (
        <Text style={{ color: 'red' }}>{fieldState.error.message}</Text>
      )}
    </View>
  );
}
```

---

## 7. SERVER-SIDE VALIDATION

Client-side validation is for UX. Server-side validation is for security. You need both, and the best pattern is **sharing the same Zod schema**.

### 7.1 Shared Schema Architecture

```
shared/
  schemas/
    profile.ts       ← Zod schema, used by both client and server
    invoice.ts
    auth.ts
app/
  (client)/
    components/
      ProfileForm.tsx  ← imports from shared/schemas/profile
  api/
    profile/
      route.ts         ← imports from shared/schemas/profile
```

### 7.2 Server-Side Validation in a Route Handler

```typescript
// app/api/profile/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { profileSchema } from '@/shared/schemas/profile';
import { ZodError } from 'zod';

export async function PUT(request: NextRequest) {
  try {
    const body = await request.json();

    // Validate with the SAME schema the client uses
    const data = profileSchema.parse(body);

    // If we get here, data is valid and typed
    const updated = await db.profile.update({
      where: { userId: session.userId },
      data,
    });

    return NextResponse.json(updated);
  } catch (error) {
    if (error instanceof ZodError) {
      // Transform Zod errors into field-level errors
      const fieldErrors: Record<string, string> = {};
      for (const issue of error.issues) {
        const field = issue.path.join('.');
        if (!fieldErrors[field]) {
          fieldErrors[field] = issue.message;
        }
      }

      return NextResponse.json(
        {
          code: 'VALIDATION_ERROR',
          message: 'Validation failed',
          details: { fieldErrors },
        },
        { status: 400 },
      );
    }

    return NextResponse.json(
      { code: 'INTERNAL_ERROR', message: 'Something went wrong' },
      { status: 500 },
    );
  }
}
```

### 7.3 Server Actions with useActionState

Next.js Server Actions + `useActionState` (React 19+) give you a first-class way to handle server-side form validation:

```typescript
// actions/profile.ts
'use server';

import { profileSchema } from '@/shared/schemas/profile';

export type ProfileActionState = {
  success: boolean;
  fieldErrors?: Record<string, string>;
  message?: string;
};

export async function updateProfileAction(
  prevState: ProfileActionState,
  formData: FormData,
): Promise<ProfileActionState> {
  // Parse FormData into an object
  const raw = {
    firstName: formData.get('firstName'),
    lastName: formData.get('lastName'),
    email: formData.get('email'),
    bio: formData.get('bio'),
    website: formData.get('website'),
  };

  // Validate
  const result = profileSchema.safeParse(raw);

  if (!result.success) {
    const fieldErrors: Record<string, string> = {};
    for (const issue of result.error.issues) {
      const field = issue.path.join('.');
      if (!fieldErrors[field]) {
        fieldErrors[field] = issue.message;
      }
    }
    return { success: false, fieldErrors };
  }

  // Additional server-only validation
  const existingUser = await db.user.findUnique({
    where: { email: result.data.email },
  });
  if (existingUser && existingUser.id !== session.userId) {
    return {
      success: false,
      fieldErrors: { email: 'This email is already in use' },
    };
  }

  // Save
  await db.profile.update({
    where: { userId: session.userId },
    data: result.data,
  });

  return { success: true, message: 'Profile updated successfully' };
}
```

```tsx
// components/ProfileFormServerAction.tsx
'use client';

import { useActionState } from 'react';
import { updateProfileAction, type ProfileActionState } from '@/actions/profile';

export function ProfileForm({ profile }: { profile: any }) {
  const [state, formAction, isPending] = useActionState<ProfileActionState, FormData>(
    updateProfileAction,
    { success: false },
  );

  return (
    <form action={formAction}>
      <div>
        <label htmlFor="firstName">First Name</label>
        <input
          id="firstName"
          name="firstName"
          defaultValue={profile.firstName}
          aria-invalid={!!state.fieldErrors?.firstName}
        />
        {state.fieldErrors?.firstName && (
          <p role="alert">{state.fieldErrors.firstName}</p>
        )}
      </div>

      <div>
        <label htmlFor="email">Email</label>
        <input
          id="email"
          name="email"
          type="email"
          defaultValue={profile.email}
          aria-invalid={!!state.fieldErrors?.email}
        />
        {state.fieldErrors?.email && (
          <p role="alert">{state.fieldErrors.email}</p>
        )}
      </div>

      <div>
        <label htmlFor="bio">Bio</label>
        <textarea
          id="bio"
          name="bio"
          defaultValue={profile.bio}
        />
      </div>

      {state.success && (
        <div role="status" className="success-message">
          {state.message}
        </div>
      )}

      {!state.success && state.message && (
        <div role="alert" className="error-message">
          {state.message}
        </div>
      )}

      <button type="submit" disabled={isPending}>
        {isPending ? 'Saving...' : 'Save Profile'}
      </button>
    </form>
  );
}
```

**Why this matters:** `useActionState` works *without JavaScript* on the client. The form submits as a standard HTML form, the server action runs, and the page re-renders with the new state. Progressive enhancement for free.

### 7.4 Combining RHF with Server Actions

You can get the best of both worlds: client-side validation via RHF + Zod for instant feedback, and server-side validation via Server Actions for security:

```tsx
'use client';

import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { useActionState, useRef } from 'react';
import { profileSchema, type ProfileFormData } from '@/shared/schemas/profile';
import { updateProfileAction } from '@/actions/profile';

export function ProfileFormHybrid({ profile }: { profile: ProfileFormData }) {
  const [serverState, formAction, isPending] = useActionState(
    updateProfileAction,
    { success: false },
  );

  const formRef = useRef<HTMLFormElement>(null);

  const {
    register,
    handleSubmit,
    formState: { errors },
  } = useForm<ProfileFormData>({
    resolver: zodResolver(profileSchema),
    defaultValues: profile,
  });

  // Client-side validation first, then submit to server action
  const onSubmit = handleSubmit(() => {
    // RHF validated successfully -- now submit to the server action
    const formData = new FormData(formRef.current!);
    formAction(formData);
  });

  // Merge client and server errors
  const getError = (field: keyof ProfileFormData) =>
    errors[field]?.message || serverState.fieldErrors?.[field];

  return (
    <form ref={formRef} onSubmit={onSubmit}>
      <div>
        <label htmlFor="firstName">First Name</label>
        <input
          id="firstName"
          {...register('firstName')}
          aria-invalid={!!getError('firstName')}
        />
        {getError('firstName') && (
          <p role="alert">{getError('firstName')}</p>
        )}
      </div>

      <div>
        <label htmlFor="email">Email</label>
        <input
          id="email"
          type="email"
          {...register('email')}
          aria-invalid={!!getError('email')}
        />
        {getError('email') && (
          <p role="alert">{getError('email')}</p>
        )}
      </div>

      {serverState.success && (
        <div role="status">Profile updated!</div>
      )}

      <button type="submit" disabled={isPending}>
        {isPending ? 'Saving...' : 'Save'}
      </button>
    </form>
  );
}
```

---

## 8. AUTOSAVE

Autosave is the feature users love and developers fear. Get it right and your users never lose work. Get it wrong and you corrupt data, hammer your API, or show a "saved" indicator when it actually failed.

### 8.1 Debounced Autosave Hook

```typescript
// hooks/useAutosave.ts
import { useRef, useEffect, useState, useCallback } from 'react';
import { UseFormReturn, FieldValues } from 'react-hook-form';

type AutosaveStatus = 'idle' | 'saving' | 'saved' | 'error';

interface UseAutosaveOptions<T extends FieldValues> {
  form: UseFormReturn<T>;
  onSave: (data: Partial<T>, dirtyFields: Record<string, boolean>) => Promise<void>;
  debounceMs?: number;
  enabled?: boolean;
}

export function useAutosave<T extends FieldValues>({
  form,
  onSave,
  debounceMs = 2000,
  enabled = true,
}: UseAutosaveOptions<T>) {
  const [status, setStatus] = useState<AutosaveStatus>('idle');
  const timeoutRef = useRef<ReturnType<typeof setTimeout>>();
  const lastSavedRef = useRef<string>('');

  const { watch, formState: { dirtyFields, isDirty } } = form;

  const save = useCallback(async () => {
    if (!isDirty) return;

    const currentValues = form.getValues();
    const serialized = JSON.stringify(currentValues);

    // Don't save if nothing changed since last save
    if (serialized === lastSavedRef.current) return;

    // Only send dirty fields
    const changedData: Partial<T> = {};
    Object.keys(dirtyFields).forEach((key) => {
      if (dirtyFields[key as keyof typeof dirtyFields]) {
        (changedData as any)[key] = currentValues[key as keyof T];
      }
    });

    if (Object.keys(changedData).length === 0) return;

    setStatus('saving');

    try {
      await onSave(changedData, dirtyFields as Record<string, boolean>);
      lastSavedRef.current = serialized;
      setStatus('saved');

      // Reset to idle after 3 seconds
      setTimeout(() => setStatus('idle'), 3000);
    } catch (error) {
      setStatus('error');
      console.error('Autosave failed:', error);
    }
  }, [form, onSave, isDirty, dirtyFields]);

  // Watch all fields and debounce saves
  useEffect(() => {
    if (!enabled) return;

    const subscription = watch(() => {
      if (timeoutRef.current) {
        clearTimeout(timeoutRef.current);
      }
      timeoutRef.current = setTimeout(save, debounceMs);
    });

    return () => {
      subscription.unsubscribe();
      if (timeoutRef.current) {
        clearTimeout(timeoutRef.current);
      }
    };
  }, [watch, save, debounceMs, enabled]);

  // Save on unmount (if there are pending changes)
  useEffect(() => {
    return () => {
      if (timeoutRef.current) {
        clearTimeout(timeoutRef.current);
        // Attempt a final save -- fire-and-forget
        save();
      }
    };
  }, [save]);

  return { status, save };
}
```

### 8.2 Autosave Status Indicator

```tsx
// components/AutosaveIndicator.tsx
function AutosaveIndicator({ status }: { status: AutosaveStatus }) {
  return (
    <div
      role="status"
      aria-live="polite"
      className="autosave-indicator"
    >
      {status === 'idle' && null}
      {status === 'saving' && (
        <span className="text-gray-500">
          <Spinner size="sm" /> Saving...
        </span>
      )}
      {status === 'saved' && (
        <span className="text-green-600">
          ✓ Saved
        </span>
      )}
      {status === 'error' && (
        <span className="text-red-600">
          Failed to save. <button onClick={save}>Retry</button>
        </span>
      )}
    </div>
  );
}
```

### 8.3 Using Autosave in a Form

```tsx
function DocumentEditor({ document }: { document: Document }) {
  const form = useForm({
    defaultValues: {
      title: document.title,
      content: document.content,
      tags: document.tags,
    },
  });

  const { status } = useAutosave({
    form,
    onSave: async (data) => {
      await api.patchDocument(document.id, data);
    },
    debounceMs: 1500,
  });

  return (
    <div>
      <div className="toolbar">
        <AutosaveIndicator status={status} />
      </div>

      <form>
        <input {...form.register('title')} placeholder="Document title" />
        <textarea {...form.register('content')} rows={20} />
      </form>
    </div>
  );
}
```

### 8.4 Conflict Detection

When multiple users (or tabs) can edit the same resource, you need conflict detection:

```typescript
// hooks/useAutosaveWithConflictDetection.ts
interface VersionedSaveOptions<T extends FieldValues> extends UseAutosaveOptions<T> {
  currentVersion: number;
  onConflict: (serverVersion: number, serverData: T) => void;
}

export function useAutosaveWithConflict<T extends FieldValues>({
  form,
  onSave,
  currentVersion,
  onConflict,
  ...options
}: VersionedSaveOptions<T>) {
  const versionRef = useRef(currentVersion);

  const versionedSave = useCallback(
    async (data: Partial<T>, dirtyFields: Record<string, boolean>) => {
      try {
        const response = await api.patchWithVersion(
          data,
          versionRef.current,
        );

        // Update our version
        versionRef.current = response.version;
      } catch (error) {
        if (error instanceof ApiError && error.code === 'VERSION_CONFLICT') {
          // Someone else edited the resource
          onConflict(error.details.serverVersion, error.details.serverData);
          throw error; // Let the autosave hook show the error status
        }
        throw error;
      }
    },
    [onConflict],
  );

  return useAutosave({
    form,
    onSave: versionedSave,
    ...options,
  });
}
```

On the server:

```typescript
// Optimistic concurrency control
async function patchDocument(id: string, data: Partial<Document>, expectedVersion: number) {
  const result = await db.document.updateMany({
    where: {
      id,
      version: expectedVersion, // Only update if version matches
    },
    data: {
      ...data,
      version: { increment: 1 },
      updatedAt: new Date(),
    },
  });

  if (result.count === 0) {
    // Version mismatch -- someone else edited it
    const current = await db.document.findUnique({ where: { id } });
    throw new ApiError(409, 'VERSION_CONFLICT', {
      serverVersion: current.version,
      serverData: current,
    });
  }
}
```

---

## 9. FORM PERFORMANCE

### 9.1 Why Controlled Inputs Kill Performance

Here is the problem with the naive approach:

```tsx
// BAD: Every keystroke re-renders the ENTIRE component
function BadForm() {
  const [formData, setFormData] = useState({
    field1: '', field2: '', field3: '', /* ... 50 more fields */
  });

  return (
    <form>
      {/* All 50+ inputs re-render on EVERY keystroke in ANY field */}
      <input
        value={formData.field1}
        onChange={(e) => setFormData({ ...formData, field1: e.target.value })}
      />
      <input
        value={formData.field2}
        onChange={(e) => setFormData({ ...formData, field2: e.target.value })}
      />
      {/* ... 50 more */}
    </form>
  );
}
```

When the user types a character in field 1, React:
1. Creates a new state object (spreading all 50+ fields)
2. Re-renders the form component
3. Re-renders all 50+ input components
4. Diffs all 50+ DOM elements
5. Updates only the one that actually changed

For 5 fields, this is invisible. For 50 fields with complex rendering (conditional visibility, computed values, custom components), this causes visible lag.

### 9.2 React Hook Form's Ref-Based Architecture

RHF solves this by not using React state for input values at all:

```tsx
// GOOD: Only the modified input updates the DOM
function GoodForm() {
  const { register, handleSubmit } = useForm();

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      {/* Each input manages its own value via ref -- zero re-renders */}
      <input {...register('field1')} />
      <input {...register('field2')} />
      {/* ... 50 more -- no re-renders when typing */}
    </form>
  );
}
```

`register` returns `{ ref, name, onChange, onBlur }`. The `ref` lets RHF read and set the input value directly. The `onChange` and `onBlur` handlers update RHF's internal store (not React state). The form component never re-renders just because the user typed.

### 9.3 Isolating Re-Renders with useWatch

When you do need reactive values (computed totals, conditional fields), isolate them:

```tsx
// BAD: watch in parent causes entire form to re-render
function BadInvoiceForm() {
  const { register, watch } = useForm();
  const lineItems = watch('lineItems'); // Re-renders on EVERY line item change
  const total = calculateTotal(lineItems); // Expensive

  return (
    <form>
      {/* All fields re-render when any line item changes */}
      <input {...register('clientName')} />
      {/* ... 30 more fields ... */}
      <p>Total: {total}</p>
    </form>
  );
}

// GOOD: useWatch in a child component isolates re-renders
function GoodInvoiceForm() {
  const { register, control } = useForm();

  return (
    <form>
      <input {...register('clientName')} /> {/* Never re-renders from total changes */}
      {/* ... 30 more fields ... */}
      <InvoiceTotal control={control} /> {/* Only this re-renders */}
    </form>
  );
}

function InvoiceTotal({ control }: { control: Control }) {
  const lineItems = useWatch({ control, name: 'lineItems' });
  const total = calculateTotal(lineItems);
  return <p>Total: {total}</p>;
}
```

### 9.4 Performance Checklist for Large Forms

```
┌─────────────────────────────────────────────────────────────┐
│            LARGE FORM PERFORMANCE CHECKLIST                  │
│                                                              │
│  ✓ Use React Hook Form (uncontrolled inputs)                │
│  ✓ Use mode: 'onBlur' (not 'onChange') for validation       │
│  ✓ Isolate reactive values with useWatch in child components│
│  ✓ Use React.memo on custom field components                │
│  ✓ Use useFieldArray (not manual array state) for lists     │
│  ✓ Virtualize long field arrays (react-window)              │
│  ✓ Debounce async validation (500ms+)                       │
│  ✓ Lazy-load conditional field groups                       │
│  ✓ Profile with React DevTools -- look for unnecessary      │
│    re-renders in the form tree                              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 9.5 Virtualizing Large Field Arrays

If your form has a dynamic list with 100+ items (e.g., a spreadsheet-like data entry form), render only the visible rows:

```tsx
import { FixedSizeList as List } from 'react-window';
import { useFieldArray, useForm } from 'react-hook-form';

function LargeListForm() {
  const { register, control } = useForm({
    defaultValues: {
      items: Array.from({ length: 500 }, (_, i) => ({
        name: `Item ${i + 1}`,
        value: 0,
      })),
    },
  });

  const { fields } = useFieldArray({ control, name: 'items' });

  const Row = ({ index, style }: { index: number; style: React.CSSProperties }) => (
    <div style={style} className="flex gap-2">
      <input {...register(`items.${index}.name`)} />
      <input
        type="number"
        {...register(`items.${index}.value`, { valueAsNumber: true })}
      />
    </div>
  );

  return (
    <form>
      <List
        height={600}
        itemCount={fields.length}
        itemSize={40}
        width="100%"
      >
        {Row}
      </List>
    </form>
  );
}
```

---

## 10. COMPLETE EXAMPLES

### 10.1 Multi-Step Checkout Form

This ties together everything: multi-step wizard, discriminated unions, file upload (receipt), server validation, and autosave.

```typescript
// schemas/checkout.ts
import { z } from 'zod';

const shippingSchema = z.object({
  firstName: z.string().min(1, 'Required'),
  lastName: z.string().min(1, 'Required'),
  address: z.string().min(1, 'Required'),
  city: z.string().min(1, 'Required'),
  state: z.string().min(2, 'Required'),
  zip: z.string().regex(/^\d{5}(-\d{4})?$/, 'Invalid ZIP'),
  phone: z.string().min(10, 'Phone number is required'),
});

const creditCardPayment = z.object({
  method: z.literal('card'),
  cardNumber: z.string().regex(/^\d{16}$/, 'Invalid card number'),
  expiry: z.string().regex(/^\d{2}\/\d{2}$/, 'Use MM/YY format'),
  cvv: z.string().regex(/^\d{3,4}$/, 'Invalid CVV'),
  nameOnCard: z.string().min(1, 'Required'),
});

const paypalPayment = z.object({
  method: z.literal('paypal'),
  paypalEmail: z.string().email('Invalid email'),
});

const paymentSchema = z.discriminatedUnion('method', [
  creditCardPayment,
  paypalPayment,
]);

export const checkoutSchema = z.object({
  shipping: shippingSchema,
  payment: paymentSchema,
  sameAsBilling: z.boolean(),
  giftMessage: z.string().max(200).optional(),
  agreeToTerms: z.boolean().refine((v) => v, 'You must agree to the terms'),
});

export type CheckoutData = z.infer<typeof checkoutSchema>;
```

```tsx
// components/checkout/CheckoutWizard.tsx
const CHECKOUT_STEPS = [
  { id: 'shipping', label: 'Shipping', schema: shippingSchema },
  { id: 'payment', label: 'Payment', schema: paymentSchema },
  { id: 'review', label: 'Review', schema: null },
];

export function CheckoutWizard({ cartItems }: { cartItems: CartItem[] }) {
  const wizard = useWizard();

  const handlePlaceOrder = async () => {
    const result = checkoutSchema.safeParse(wizard.data);
    if (!result.success) {
      // Find which step has errors and navigate there
      const firstErrorPath = result.error.issues[0]?.path[0];
      if (firstErrorPath === 'shipping') wizard.goToStep(0);
      else if (firstErrorPath === 'payment') wizard.goToStep(1);
      return;
    }

    try {
      const order = await api.placeOrder({
        ...result.data,
        items: cartItems,
      });
      wizard.clearStorage();
      router.push(`/order-confirmation/${order.id}`);
    } catch (error) {
      if (error instanceof ApiError) {
        // Handle specific errors
        if (error.code === 'CARD_DECLINED') {
          wizard.goToStep(1); // Go back to payment
          // Show error on payment step
        }
      }
    }
  };

  return (
    <div className="checkout-layout">
      <div className="checkout-main">
        <StepIndicator steps={CHECKOUT_STEPS} currentStep={wizard.currentStep} />

        {wizard.currentStep === 0 && (
          <ShippingStep
            defaultValues={wizard.data.shipping}
            onSubmit={(shipping) => {
              wizard.updateData({ shipping });
              wizard.markCompleted(0);
              wizard.next();
            }}
          />
        )}

        {wizard.currentStep === 1 && (
          <PaymentStep
            defaultValues={wizard.data.payment}
            onSubmit={(payment) => {
              wizard.updateData({ payment });
              wizard.markCompleted(1);
              wizard.next();
            }}
            onBack={wizard.previous}
          />
        )}

        {wizard.currentStep === 2 && (
          <ReviewStep
            data={wizard.data}
            cartItems={cartItems}
            onPlaceOrder={handlePlaceOrder}
            onBack={wizard.previous}
            onEditStep={wizard.goToStep}
          />
        )}
      </div>

      {/* Persistent order summary sidebar */}
      <aside className="checkout-sidebar">
        <OrderSummary items={cartItems} />
      </aside>
    </div>
  );
}
```

### 10.2 Dynamic Survey Builder

A form that builds other forms:

```tsx
// components/SurveyBuilder.tsx
function SurveyBuilder() {
  const form = useForm<SurveyDefinition>({
    defaultValues: {
      title: 'Untitled Survey',
      description: '',
      questions: [
        { type: 'text', label: '', required: false, options: [] },
      ],
    },
  });

  const { fields, append, remove, move } = useFieldArray({
    control: form.control,
    name: 'questions',
  });

  return (
    <div className="survey-builder">
      <input
        {...form.register('title')}
        className="text-2xl font-bold"
        placeholder="Survey Title"
      />
      <textarea
        {...form.register('description')}
        placeholder="Description (optional)"
      />

      <div className="questions-list">
        {fields.map((field, index) => (
          <QuestionEditor
            key={field.id}
            index={index}
            control={form.control}
            register={form.register}
            onRemove={() => remove(index)}
            onMoveUp={() => index > 0 && move(index, index - 1)}
            onMoveDown={() => index < fields.length - 1 && move(index, index + 1)}
          />
        ))}
      </div>

      <button
        type="button"
        onClick={() =>
          append({ type: 'text', label: '', required: false, options: [] })
        }
      >
        + Add Question
      </button>

      {/* Preview the survey as users would see it */}
      <SurveyPreview definition={form.watch()} />
    </div>
  );
}

function QuestionEditor({
  index,
  control,
  register,
  onRemove,
  onMoveUp,
  onMoveDown,
}: QuestionEditorProps) {
  const questionType = useWatch({
    control,
    name: `questions.${index}.type`,
  });

  const { fields: optionFields, append, remove } = useFieldArray({
    control,
    name: `questions.${index}.options`,
  });

  return (
    <div className="question-card">
      <div className="question-header">
        <select {...register(`questions.${index}.type`)}>
          <option value="text">Short Text</option>
          <option value="textarea">Long Text</option>
          <option value="number">Number</option>
          <option value="select">Dropdown</option>
          <option value="radio">Multiple Choice</option>
          <option value="checkbox">Checkboxes</option>
          <option value="rating">Rating (1-5)</option>
        </select>

        <label>
          <input
            type="checkbox"
            {...register(`questions.${index}.required`)}
          />
          Required
        </label>

        <div className="question-actions">
          <button type="button" onClick={onMoveUp}>↑</button>
          <button type="button" onClick={onMoveDown}>↓</button>
          <button type="button" onClick={onRemove}>Delete</button>
        </div>
      </div>

      <input
        {...register(`questions.${index}.label`)}
        placeholder="Question text"
        className="question-label-input"
      />

      {/* Options editor for select/radio/checkbox types */}
      {['select', 'radio', 'checkbox'].includes(questionType) && (
        <div className="options-list">
          {optionFields.map((opt, optIndex) => (
            <div key={opt.id} className="option-row">
              <input
                {...register(
                  `questions.${index}.options.${optIndex}.label`,
                )}
                placeholder={`Option ${optIndex + 1}`}
              />
              <button type="button" onClick={() => remove(optIndex)}>
                ×
              </button>
            </div>
          ))}
          <button
            type="button"
            onClick={() => append({ label: '', value: '' })}
          >
            + Add Option
          </button>
        </div>
      )}
    </div>
  );
}
```

### 10.3 Search with Filter Sidebar

Forms are not just for data entry. Search filters are forms too, and they have their own patterns: instant application (no submit button), URL synchronization, and remembering preferences.

```tsx
// components/SearchFilters.tsx
import { useForm } from 'react-hook-form';
import { useRouter, useSearchParams } from 'next/navigation';
import { useEffect, useCallback, useRef } from 'react';

interface FilterValues {
  query: string;
  category: string;
  priceMin: number | undefined;
  priceMax: number | undefined;
  inStock: boolean;
  sortBy: 'relevance' | 'price-asc' | 'price-desc' | 'newest';
  rating: number | undefined;
}

export function SearchFilters({ onFiltersChange }: {
  onFiltersChange: (filters: FilterValues) => void;
}) {
  const router = useRouter();
  const searchParams = useSearchParams();

  // Initialize from URL params
  const defaultValues: FilterValues = {
    query: searchParams.get('q') ?? '',
    category: searchParams.get('category') ?? 'all',
    priceMin: searchParams.get('priceMin')
      ? Number(searchParams.get('priceMin'))
      : undefined,
    priceMax: searchParams.get('priceMax')
      ? Number(searchParams.get('priceMax'))
      : undefined,
    inStock: searchParams.get('inStock') === 'true',
    sortBy: (searchParams.get('sortBy') as FilterValues['sortBy']) ?? 'relevance',
    rating: searchParams.get('rating')
      ? Number(searchParams.get('rating'))
      : undefined,
  };

  const form = useForm<FilterValues>({ defaultValues });
  const debounceRef = useRef<ReturnType<typeof setTimeout>>();

  // Watch all values and sync to URL + callback
  useEffect(() => {
    const subscription = form.watch((values) => {
      if (debounceRef.current) clearTimeout(debounceRef.current);

      debounceRef.current = setTimeout(() => {
        // Update URL params
        const params = new URLSearchParams();
        if (values.query) params.set('q', values.query);
        if (values.category && values.category !== 'all') {
          params.set('category', values.category);
        }
        if (values.priceMin) params.set('priceMin', String(values.priceMin));
        if (values.priceMax) params.set('priceMax', String(values.priceMax));
        if (values.inStock) params.set('inStock', 'true');
        if (values.sortBy !== 'relevance') params.set('sortBy', values.sortBy!);
        if (values.rating) params.set('rating', String(values.rating));

        router.replace(`?${params.toString()}`, { scroll: false });
        onFiltersChange(values as FilterValues);
      }, 300);
    });

    return () => {
      subscription.unsubscribe();
      if (debounceRef.current) clearTimeout(debounceRef.current);
    };
  }, [form, router, onFiltersChange]);

  const resetFilters = useCallback(() => {
    form.reset({
      query: '',
      category: 'all',
      priceMin: undefined,
      priceMax: undefined,
      inStock: false,
      sortBy: 'relevance',
      rating: undefined,
    });
  }, [form]);

  return (
    <aside className="filter-sidebar">
      <div className="filter-header">
        <h3>Filters</h3>
        <button type="button" onClick={resetFilters}>
          Clear All
        </button>
      </div>

      <div className="filter-group">
        <label htmlFor="search-query">Search</label>
        <input
          id="search-query"
          {...form.register('query')}
          placeholder="Search products..."
        />
      </div>

      <div className="filter-group">
        <label htmlFor="category">Category</label>
        <select id="category" {...form.register('category')}>
          <option value="all">All Categories</option>
          <option value="electronics">Electronics</option>
          <option value="clothing">Clothing</option>
          <option value="home">Home & Garden</option>
        </select>
      </div>

      <div className="filter-group">
        <label>Price Range</label>
        <div className="flex gap-2">
          <input
            type="number"
            {...form.register('priceMin', { valueAsNumber: true })}
            placeholder="Min"
          />
          <span>to</span>
          <input
            type="number"
            {...form.register('priceMax', { valueAsNumber: true })}
            placeholder="Max"
          />
        </div>
      </div>

      <div className="filter-group">
        <label>
          <input type="checkbox" {...form.register('inStock')} />
          In Stock Only
        </label>
      </div>

      <div className="filter-group">
        <label htmlFor="sortBy">Sort By</label>
        <select id="sortBy" {...form.register('sortBy')}>
          <option value="relevance">Relevance</option>
          <option value="price-asc">Price: Low to High</option>
          <option value="price-desc">Price: High to Low</option>
          <option value="newest">Newest First</option>
        </select>
      </div>
    </aside>
  );
}
```

### 10.4 Profile Editor with Image Upload

```tsx
// components/ProfileEditor.tsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { profileSchema, type ProfileFormData } from '@/schemas/profile';
import { FileUploadField } from '@/components/FileUploadField';
import { useAutosave } from '@/hooks/useAutosave';

export function ProfileEditor({ profile }: { profile: ProfileFormData & { avatar: string } }) {
  const form = useForm({
    resolver: zodResolver(
      profileSchema.extend({
        avatar: z.string().url().optional().or(z.literal('')),
      }),
    ),
    defaultValues: profile,
  });

  const { status } = useAutosave({
    form,
    onSave: async (data) => {
      await api.patchProfile(data);
    },
    debounceMs: 2000,
  });

  const {
    register,
    control,
    handleSubmit,
    formState: { errors },
  } = form;

  return (
    <div>
      <div className="flex justify-between items-center">
        <h2>Edit Profile</h2>
        <AutosaveIndicator status={status} />
      </div>

      <form onSubmit={handleSubmit(async (data) => {
        await api.updateProfile(data);
      })}>
        <FileUploadField
          name="avatar"
          control={control}
          accept="image/jpeg,image/png,image/webp"
          maxSizeMB={5}
          label="Profile Photo"
        />

        <div className="grid grid-cols-2 gap-4">
          <div>
            <label htmlFor="firstName">First Name</label>
            <input id="firstName" {...register('firstName')} />
            {errors.firstName && <p role="alert">{errors.firstName.message}</p>}
          </div>
          <div>
            <label htmlFor="lastName">Last Name</label>
            <input id="lastName" {...register('lastName')} />
            {errors.lastName && <p role="alert">{errors.lastName.message}</p>}
          </div>
        </div>

        <div>
          <label htmlFor="email">Email</label>
          <input id="email" type="email" {...register('email')} />
          {errors.email && <p role="alert">{errors.email.message}</p>}
        </div>

        <div>
          <label htmlFor="bio">Bio</label>
          <textarea id="bio" {...register('bio')} rows={4} />
          <BioCharCount control={control} max={500} />
          {errors.bio && <p role="alert">{errors.bio.message}</p>}
        </div>

        <div>
          <label htmlFor="website">Website</label>
          <input id="website" type="url" {...register('website')} />
          {errors.website && <p role="alert">{errors.website.message}</p>}
        </div>

        <button type="submit">Save Profile</button>
      </form>
    </div>
  );
}

function BioCharCount({ control, max }: { control: Control; max: number }) {
  const bio = useWatch({ control, name: 'bio' }) ?? '';
  const remaining = max - bio.length;

  return (
    <span className={remaining < 50 ? 'text-red-500' : 'text-gray-400'}>
      {remaining} characters remaining
    </span>
  );
}
```

---

## WRAPPING UP

Forms at scale are not about knowing the React Hook Form API. They are about understanding the layers:

1. **Schema layer** (Zod): Define your validation once, share it everywhere, derive your types from it
2. **Form state layer** (React Hook Form): Let the library manage input state via refs, use `useFieldArray` for dynamic lists, use `useWatch` to isolate re-renders
3. **UI layer** (your components): Build reusable field components with proper accessibility, error display, and loading states
4. **Submission layer** (API/Server Actions): Validate again on the server with the same schema, return field-level errors
5. **Persistence layer** (autosave/wizards): Debounce saves, track dirty fields, detect conflicts, persist wizard progress

If you internalize these five layers, every form you build -- from a simple login to a 50-field admin dashboard to a multi-step onboarding wizard -- will follow the same architecture. The complexity scales linearly because the patterns compose.

The next chapter covers what happens when things go wrong: Error Handling Architecture. Because the best form in the world is useless if the submission error shows a white screen instead of a helpful message.

---

> **Next:** [Chapter 36: Error Handling Architecture — Graceful Failures Everywhere](/part-4-architecture-at-scale/36-error-handling.md)
>
> **Previous:** [Chapter 34](/part-3-state-data-communication/34-previous-chapter.md)
