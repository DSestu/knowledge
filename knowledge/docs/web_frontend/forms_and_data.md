---
title: Forms, Data Fetching, and Validation
---

## Forms

- Start simple with controlled components
- For larger forms, use `react-hook-form` with Zod or Yup

```bash
pnpm add react-hook-form zod @hookform/resolvers
```

```tsx
import { useForm } from 'react-hook-form'
import { z } from 'zod'
import { zodResolver } from '@hookform/resolvers/zod'

const Schema = z.object({
  email: z.string().email(),
  name: z.string().min(2),
})
type FormData = z.infer<typeof Schema>

export function ContactForm() {
  const { register, handleSubmit, formState: { errors } } = useForm<FormData>({
    resolver: zodResolver(Schema),
  })
  const onSubmit = (data: FormData) => console.log(data)
  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('email')} placeholder="Email" />
      {errors.email && <p>{errors.email.message}</p>}
      <input {...register('name')} placeholder="Name" />
      {errors.name && <p>{errors.name.message}</p>}
      <button type="submit">Send</button>
    </form>
  )
}
```

## Data Fetching

- Start with `fetch` + simple `useEffect`
- For more features (caching, revalidation): TanStack Query

```bash
pnpm add @tanstack/react-query
```

```tsx
import { useQuery, QueryClient, QueryClientProvider } from '@tanstack/react-query'

const client = new QueryClient()

export function App() {
  return (
    <QueryClientProvider client={client}>
      <Users />
    </QueryClientProvider>
  )
}

function Users() {
  const { data, isLoading, error } = useQuery({
    queryKey: ['users'],
    queryFn: async () => {
      const res = await fetch('https://jsonplaceholder.typicode.com/users')
      if (!res.ok) throw new Error('Network error')
      return res.json() as Promise<Array<{ id: number; name: string }>>
    },
  })
  if (isLoading) return <p>Loading…</p>
  if (error) return <p>Error</p>
  return <ul>{data!.map((u) => <li key={u.id}>{u.name}</li>)}</ul>
}
```

## Runtime Validation

- Validate API responses with Zod when correctness matters
