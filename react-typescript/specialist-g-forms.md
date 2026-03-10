# Specialist G: Forms

## Scope
React Hook Form with TypeScript, Zod schema integration, controlled inputs, validation error typing.

---

## 1. React Hook Form + Zod — Standard Pattern

```tsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

// 1. Define schema
const loginSchema = z.object({
  email: z.string().email('Invalid email address'),
  password: z.string().min(8, 'Password must be at least 8 characters'),
  rememberMe: z.boolean().default(false),
});

// 2. Derive TypeScript type from schema
type LoginFormValues = z.infer<typeof loginSchema>;

// 3. Component
function LoginForm() {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm<LoginFormValues>({
    resolver: zodResolver(loginSchema),
    defaultValues: {
      email: '',
      password: '',
      rememberMe: false,
    },
  });

  const onSubmit = async (data: LoginFormValues) => {
    await authService.login(data); // data is fully typed ✓
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <div>
        <label htmlFor="email">Email</label>
        <input id="email" type="email" {...register('email')} />
        {errors.email && <span>{errors.email.message}</span>}
      </div>

      <div>
        <label htmlFor="password">Password</label>
        <input id="password" type="password" {...register('password')} />
        {errors.password && <span>{errors.password.message}</span>}
      </div>

      <label>
        <input type="checkbox" {...register('rememberMe')} />
        Remember me
      </label>

      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Logging in...' : 'Log in'}
      </button>
    </form>
  );
}
```

---

## 2. Complex Form Schema with Nested Objects

```tsx
const profileSchema = z.object({
  personal: z.object({
    firstName: z.string().min(1, 'Required'),
    lastName: z.string().min(1, 'Required'),
    birthDate: z.string().regex(/^\d{4}-\d{2}-\d{2}$/, 'Format: YYYY-MM-DD'),
  }),
  contact: z.object({
    email: z.string().email(),
    phone: z.string().optional(),
  }),
  preferences: z.object({
    newsletter: z.boolean(),
    notifications: z.enum(['all', 'important', 'none']),
  }),
});

type ProfileFormValues = z.infer<typeof profileSchema>;

function ProfileForm() {
  const { register, handleSubmit, formState: { errors } } = useForm<ProfileFormValues>({
    resolver: zodResolver(profileSchema),
  });

  return (
    <form onSubmit={handleSubmit(data => updateProfile(data))}>
      <input {...register('personal.firstName')} />
      {errors.personal?.firstName && <span>{errors.personal.firstName.message}</span>}

      <select {...register('preferences.notifications')}>
        <option value="all">All</option>
        <option value="important">Important only</option>
        <option value="none">None</option>
      </select>
    </form>
  );
}
```

---

## 3. Dynamic Field Arrays

```tsx
import { useFieldArray } from 'react-hook-form';

const orderSchema = z.object({
  customerId: z.string(),
  items: z.array(z.object({
    productId: z.string().min(1, 'Required'),
    quantity: z.number().int().positive(),
    price: z.number().positive(),
  })).min(1, 'At least one item required'),
});

type OrderFormValues = z.infer<typeof orderSchema>;

function OrderForm() {
  const { register, control, handleSubmit, formState: { errors } } = useForm<OrderFormValues>({
    resolver: zodResolver(orderSchema),
    defaultValues: { items: [{ productId: '', quantity: 1, price: 0 }] },
  });

  const { fields, append, remove } = useFieldArray({ control, name: 'items' });

  return (
    <form onSubmit={handleSubmit(submitOrder)}>
      {fields.map((field, index) => (
        <div key={field.id}>
          <input {...register(`items.${index}.productId`)} placeholder="Product ID" />
          <input
            type="number"
            {...register(`items.${index}.quantity`, { valueAsNumber: true })}
          />
          {errors.items?.[index]?.quantity && (
            <span>{errors.items[index].quantity?.message}</span>
          )}
          <button type="button" onClick={() => remove(index)}>Remove</button>
        </div>
      ))}
      <button type="button" onClick={() => append({ productId: '', quantity: 1, price: 0 })}>
        Add Item
      </button>
      <button type="submit">Submit Order</button>
    </form>
  );
}
```

---

## 4. Controlled Input with `Controller`

Use `Controller` for custom/third-party inputs that don't support `ref`:

```tsx
import { Controller } from 'react-hook-form';

const formSchema = z.object({
  country: z.string().min(1, 'Required'),
  birthDate: z.date(),
});

type FormValues = z.infer<typeof formSchema>;

function MyForm() {
  const { control, handleSubmit } = useForm<FormValues>({
    resolver: zodResolver(formSchema),
  });

  return (
    <form onSubmit={handleSubmit(submitData)}>
      <Controller
        name="country"
        control={control}
        render={({ field, fieldState }) => (
          <CountrySelect
            value={field.value}
            onChange={field.onChange}
            onBlur={field.onBlur}
            error={fieldState.error?.message}
          />
        )}
      />
    </form>
  );
}
```

---

## 5. Shared Form Field Component

```tsx
import type { UseFormRegister, FieldErrors, Path, FieldValues } from 'react-hook-form';

type FormFieldProps<T extends FieldValues> = {
  name: Path<T>;
  label: string;
  register: UseFormRegister<T>;
  errors: FieldErrors<T>;
  type?: React.HTMLInputTypeAttribute;
};

function FormField<T extends FieldValues>({
  name,
  label,
  register,
  errors,
  type = 'text',
}: FormFieldProps<T>) {
  const error = errors[name];

  return (
    <div className="field">
      <label htmlFor={name}>{label}</label>
      <input id={name} type={type} {...register(name)} />
      {error && <span className="error">{String(error.message)}</span>}
    </div>
  );
}

// Usage
<FormField<LoginFormValues>
  name="email"
  label="Email"
  register={register}
  errors={errors}
  type="email"
/>
```

---

## 6. Rules Summary

- Always derive TypeScript types from Zod schema: `type T = z.infer<typeof schema>`. Never write the type separately.
- Use `register` for native HTML inputs. Use `Controller` for custom inputs.
- Use `valueAsNumber: true` for number inputs: `register('age', { valueAsNumber: true })`.
- Default values must match the schema shape — provide them in `useForm({ defaultValues })`, not as HTML `defaultValue`.
- Never use raw `useState` to manage form values — use React Hook Form's state.
