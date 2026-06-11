---
paths:
  - "**/*.tsx"
  - "**/*.ts"
description: "Best practices for GraphQL data fetching with Apollo Client in Next.js App Router (RSC + Client Components)"
---

# GraphQL & Apollo Client Best Practices

This document defines the architecture and best practices for GraphQL data fetching in the Pop Logic application using Apollo Client with Next.js App Router.

---

## 1. Architecture Overview

### Data Fetching Strategy

| Use Case                    | Environment  | Tool                                | When to Use                                 |
| --------------------------- | ------------ | ----------------------------------- | ------------------------------------------- |
| Initial page/component data | Server (RSC) | `query()`, `getClient()`            | Page load, SEO content, static data         |
| User interactions           | Browser      | `useSuspenseQuery`, `useMutation`   | Clicks, form submissions, real-time updates |
| Hybrid (RSC → Client)       | Both         | `PreloadQuery` + `useSuspenseQuery` | Preload in RSC, hydrate in Client           |

### Why This Separation?

> ❗️ **RSC and SSR are completely separate use cases.**
>
> - Queries in RSC will NOT update in the browser without a full page rerender
> - Queries in Client Components CAN update dynamically (mutations, cache updates)
> - Overlapping queries between RSC and Client Components cause UI inconsistencies

---

## 2. File Structure

```
src/lib/apollo/
├── index.ts           # Barrel exports
├── apollo-client.ts   # Browser/SSR client (makeClient)
├── apollo-rsc.ts      # RSC client (query, getClient, PreloadQuery)
└── ApolloProvider.tsx # Client Component wrapper

src/graphql/
├── index.ts           # Barrel exports
├── fragments/         # Reusable GraphQL fragments
├── queries/           # GraphQL query definitions
└── mutations/         # GraphQL mutation definitions
```

> **Note:** Apollo setup is split into two files due to package conditional exports.
> `registerApolloClient` is only available in the RSC bundle.

---

## 3. Server Components (RSC) Patterns

### Basic Query

```tsx
// In a Server Component (page.tsx, layout.tsx)
import { query } from "@/lib/apollo/apollo-rsc";
import { GET_PRODUCTS } from "@/graphql";

export default async function ProductsPage() {
  const { data } = await query({
    query: GET_PRODUCTS,
    variables: { limit: 10 },
  });

  return (
    <ul>
      {data.products.map((product) => (
        <li key={product.id}>{product.name}</li>
      ))}
    </ul>
  );
}
```

### With Next.js Caching

```tsx
const { data } = await query({
  query: GET_PRODUCTS,
  context: {
    fetchOptions: {
      next: {
        tags: ["products"], // For revalidateTag()
        revalidate: 60, // Revalidate every 60 seconds
      },
    },
  },
});
```

### Using getClient() for Multiple Queries

```tsx
import { getClient } from "@/lib/apollo/apollo-rsc";

export default async function DashboardPage() {
  const client = getClient();

  const [usersResult, statsResult] = await Promise.all([
    client.query({ query: GET_USERS }),
    client.query({ query: GET_STATS }),
  ]);

  return (
    <div>
      <UserList users={usersResult.data.users} />
      <Stats stats={statsResult.data.stats} />
    </div>
  );
}
```

---

## 4. Client Components Patterns

### Setup: ApolloProvider in Root Layout

```tsx
// app/layout.tsx
import { ApolloProvider } from "@/lib";

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body>
        <ApolloProvider>{children}</ApolloProvider>
      </body>
    </html>
  );
}
```

### Queries with useSuspenseQuery

```tsx
"use client";

import { useSuspenseQuery } from "@apollo/client";
import { GET_USER_PROFILE } from "@/graphql";

export function UserProfile({ userId }: { userId: string }) {
  const { data } = useSuspenseQuery(GET_USER_PROFILE, {
    variables: { userId },
  });

  return <div>{data.user.name}</div>;
}
```

### Mutations

```tsx
"use client";

import { useMutation } from "@apollo/client";
import { CREATE_POST, GET_POSTS } from "@/graphql";

export function CreatePostButton() {
  const [createPost, { loading }] = useMutation(CREATE_POST, {
    // Update cache after mutation
    refetchQueries: [{ query: GET_POSTS }],
    // Or use cache.modify for fine-grained control
  });

  const handleCreate = async () => {
    await createPost({
      variables: { title: "New Post", content: "..." },
    });
  };

  return (
    <button onClick={handleCreate} disabled={loading}>
      {loading ? "Creating..." : "Create Post"}
    </button>
  );
}
```

### Optimistic Updates

```tsx
const [updatePost] = useMutation(UPDATE_POST, {
  optimisticResponse: {
    updatePost: {
      __typename: "Post",
      id: postId,
      title: newTitle,
    },
  },
});
```

---

## 5. Hybrid Pattern: PreloadQuery

Use `PreloadQuery` when you want to fetch data in RSC but need reactivity in Client Components.

### Server Component (Parent)

```tsx
// app/dashboard/page.tsx (Server Component)
import { PreloadQuery } from "@/lib/apollo/apollo-rsc";
import { GET_USER_STATS } from "@/graphql";
import { Suspense } from "react";
import { UserStats } from "@/components/UserStats";

export default async function DashboardPage() {
  return (
    <PreloadQuery query={GET_USER_STATS} variables={{ userId: "123" }}>
      <Suspense fallback={<div>Loading stats...</div>}>
        <UserStats />
      </Suspense>
    </PreloadQuery>
  );
}
```

### Client Component (Child)

```tsx
// components/UserStats.tsx
"use client";

import { useSuspenseQuery } from "@apollo/client";
import { GET_USER_STATS } from "@/graphql";

export function UserStats() {
  // Data is preloaded from RSC — no network request in browser!
  const { data } = useSuspenseQuery(GET_USER_STATS, {
    variables: { userId: "123" },
  });

  return <div>{data.stats.totalPosts} posts</div>;
}
```

### With useReadQuery (Advanced)

```tsx
// Server Component
<PreloadQuery query={GET_USER_STATS} variables={{ userId: "123" }}>
  {(queryRef) => (
    <Suspense fallback={<div>Loading...</div>}>
      <UserStats queryRef={queryRef} />
    </Suspense>
  )}
</PreloadQuery>;

// Client Component
("use client");

import { useReadQuery, useQueryRefHandlers, QueryRef } from "@apollo/client";

interface Props {
  queryRef: QueryRef<UserStatsData>;
}

export function UserStats({ queryRef }: Props) {
  const { refetch } = useQueryRefHandlers(queryRef);
  const { data } = useReadQuery(queryRef);

  return (
    <div>
      <p>{data.stats.totalPosts} posts</p>
      <button onClick={() => refetch()}>Refresh</button>
    </div>
  );
}
```

---

## 6. Query Organization

### Define Queries in Dedicated Files

```tsx
// src/graphql/queries/user.queries.ts
import { gql } from "@apollo/client";
import { USER_FRAGMENT } from "../fragments";

export const GET_USER = gql`
  ${USER_FRAGMENT}
  query GetUser($id: ID!) {
    user(id: $id) {
      ...UserFields
    }
  }
`;

export const GET_USERS = gql`
  ${USER_FRAGMENT}
  query GetUsers($limit: Int) {
    users(limit: $limit) {
      ...UserFields
    }
  }
`;
```

### Use Fragments for Reusability

```tsx
// src/graphql/fragments/user.fragments.ts
import { gql } from "@apollo/client";

export const USER_FRAGMENT = gql`
  fragment UserFields on User {
    id
    name
    email
    avatar
  }
`;
```

### Barrel Exports

```tsx
// src/graphql/index.ts
export * from "./fragments";
export * from "./queries";
export * from "./mutations";
```

---

## 7. Error Handling

### In RSC

```tsx
import { query } from "@/lib/apollo/apollo-rsc";

export default async function ProductsPage() {
  try {
    const { data } = await query({ query: GET_PRODUCTS });
    return <ProductList products={data.products} />;
  } catch (error) {
    // Log error server-side
    console.error("Failed to fetch products:", error);
    return <ErrorState message="Failed to load products" />;
  }
}
```

### In Client Components

```tsx
"use client";

import { useSuspenseQuery } from "@apollo/client";
import { ErrorBoundary } from "react-error-boundary";

function ProductList() {
  const { data } = useSuspenseQuery(GET_PRODUCTS);
  return <div>{/* render products */}</div>;
}

export function ProductsSection() {
  return (
    <ErrorBoundary fallback={<div>Something went wrong</div>}>
      <Suspense fallback={<div>Loading...</div>}>
        <ProductList />
      </Suspense>
    </ErrorBoundary>
  );
}
```

---

## 8. Cache Management

### Cache Policies

```tsx
const { data } = useSuspenseQuery(GET_USER, {
  variables: { id: userId },
  fetchPolicy: "cache-first", // Default: use cache, fetch if missing
  // fetchPolicy: 'network-only',  // Always fetch, update cache
  // fetchPolicy: 'cache-only',    // Only use cache, never fetch
  // fetchPolicy: 'no-cache',      // Fetch without caching
});
```

### Updating Cache After Mutations

```tsx
const [createPost] = useMutation(CREATE_POST, {
  update(cache, { data: { createPost } }) {
    cache.modify({
      fields: {
        posts(existingPosts = []) {
          const newPostRef = cache.writeFragment({
            data: createPost,
            fragment: POST_FRAGMENT,
          });
          return [...existingPosts, newPostRef];
        },
      },
    });
  },
});
```

---

## 9. Testing

### Reset Singletons Between Tests

```tsx
import { resetApolloClientSingletons } from "@apollo/client-integration-nextjs";

afterEach(() => {
  resetApolloClientSingletons();
});
```

### Mock Apollo Provider

```tsx
import { MockedProvider } from "@apollo/client/testing";

const mocks = [
  {
    request: { query: GET_USER, variables: { id: "1" } },
    result: { data: { user: { id: "1", name: "John" } } },
  },
];

render(
  <MockedProvider mocks={mocks} addTypename={false}>
    <UserProfile userId="1" />
  </MockedProvider>,
);
```

---

## 10. Do's and Don'ts

### ✅ Do

- Separate RSC queries from Client Component queries
- Use `PreloadQuery` for hybrid scenarios
- Define queries/mutations in dedicated files under `src/graphql/`
- Use fragments for reusable field selections
- Wrap Client Components with `ApolloProvider` at root layout
- Use `useSuspenseQuery` over `useQuery` for Suspense support
- Handle errors with Error Boundaries in Client Components

### ❌ Don't

- Don't overlap queries between RSC and Client Components
- Don't import from `apollo-rsc.ts` in Client Components
- Don't use `useQuery` for initial data in Client Components (use `useSuspenseQuery`)
- Don't fetch the same data in both RSC and Client — choose one
- Don't call `registerApolloClient` in files processed by both environments
- Don't use relative URLs in HttpLink — always use absolute URLs

---

## 11. Environment Variables

| Variable                      | Usage                            | Example                           |
| ----------------------------- | -------------------------------- | --------------------------------- |
| `NEXT_PUBLIC_GRAPHQL_API_URL` | GraphQL endpoint (browser + RSC) | `https://api.example.com/graphql` |

> **Note:** Use `NEXT_PUBLIC_` prefix so the variable is available in both server and browser.

---

## 12. Debugging

Enable Apollo Client logging for development:

```tsx
import { setLogVerbosity } from "@apollo/client";

if (process.env.NODE_ENV === "development") {
  setLogVerbosity("debug");
}
```
