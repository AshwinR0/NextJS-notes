## Next.js In-Depth Notes for the Senior Frontend Developer: SEO & Metadata

Search Engine Optimization (SEO) is not an afterthought; it's a critical component of building successful web applications. Next.js is architected with SEO as a primary concern, providing a powerful and flexible Metadata API to manage your page's `<head>` section effectively.

### 1. Introduction to SEO with Next.js

From a senior developer's perspective, Next.js's main advantage for SEO stems from its rendering strategies. Because Next.js can pre-render pages on the server (using SSR or SSG), search engine crawlers receive fully-formed HTML documents, rather than an empty `<body>` tag that needs to be filled by client-side JavaScript. This dramatically improves a site's ability to be crawled and indexed effectively.

Beyond rendering, Next.js provides two primary ways to manage metadata:
*   **Config-based Metadata:** Exporting a special `metadata` object or a `generateMetadata` function from your `layout.tsx` or `page.tsx` files.
*   **File-based Metadata:** Adding special files like `robots.txt` or `sitemap.xml` to your project to give instructions to search engine crawlers.

This guide focuses on the config-based approach, which handles the most common and critical SEO tags like `title` and `description`.

### 2. Understanding Metadata in Next.js

The Metadata API in the App Router allows you to export an object that defines your page's metadata. Next.js then automatically generates the corresponding `<head>` elements.

**Key Principles:**
*   **Server-Side Only:** The `metadata` object and `generateMetadata` function are only supported in Server Components. This is crucial because metadata needs to be present in the initial HTML document sent from the server.
*   **Placement:** Metadata can be defined in both `layout.tsx` and `page.tsx` files.
*   **Inheritance and Overriding:** Metadata is resolved hierarchically. Metadata defined in a page will override metadata from its parent layout. If a page doesn't define a specific metadata field (like `title`), it will inherit the value from the nearest parent layout that does.

#### Static Metadata Configuration

For pages where the metadata is known at build time (e.g., an "About Us" or "Contact" page), you can export a static `metadata` object.

**Implementation (`app/about/page.tsx`):**
```typescript
import type { Metadata } from 'next';

export const metadata: Metadata = {
  title: 'About Us | Acme Inc.',
  description: 'Learn more about the mission and team at Acme Inc.',
  keywords: ['company', 'about', 'mission', 'team'],
  openGraph: {
    title: 'About Us | Acme Inc.',
    description: 'Learn more about the mission and team at Acme Inc.',
    // You'd also add images, url, etc.
  },
};

export default function AboutPage() {
  return <h1>About Our Company</h1>;
}
```

#### Dynamic Metadata Configuration

For pages where metadata depends on dynamic data (e.g., a product page or a blog post), you must export an async function called `generateMetadata`. This function receives the route's `params` and `searchParams` as arguments, allowing you to fetch data and construct the metadata dynamically.

**Implementation (`app/products/[productId]/page.tsx`):**
```typescript
import type { Metadata, ResolvingMetadata } from 'next';

type Props = {
  params: { productId: string };
};

// This function is called by Next.js on the server during a request.
export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const { productId } = params;

  // Fetch product data based on the ID
  const product = await fetch(`https://api.example.com/products/${productId}`).then((res) => res.json());

  // If the product doesn't exist, you can return default metadata or handle it
  if (!product) {
    return {
      title: 'Product Not Found',
    };
  }

  return {
    title: product.name,
    description: product.description,
  };
}

export default async function ProductPage({ params }: Props) {
  // Page component still needs to fetch its own data.
  // Next.js automatically deduplicates the fetch requests.
  const product = await fetch(`https://api.example.com/products/${params.productId}`).then((res) => res.json());

  return <h1>{product.name}</h1>;
}
```
> **Performance Note:** `fetch` requests made inside `generateMetadata` are automatically memoized. If you make the exact same `fetch` request in your page component, Next.js will use the cached result, avoiding a duplicate network call.

#### Handling Metadata in Client Components

You **cannot** export `metadata` or `generateMetadata` from a file marked with the `'use client'` directive. This is by design, as metadata must be rendered on the server.

If a page requires client-side interactivity, the correct pattern is to:
1.  Keep the `page.tsx` file as a Server Component to handle metadata.
2.  Extract the interactive parts of the page into a separate component.
3.  Mark the new component file with `'use client'`.
4.  Import and render the new Client Component within your `page.tsx`.

**Example:**
```typescript
// app/interactive-page/page.tsx (Server Component)
import type { Metadata } from 'next';
import InteractiveCounter from './_components/InteractiveCounter'; // The Client Component

export const metadata: Metadata = {
  title: 'Interactive Page',
};

// This page remains a Server Component
export default function InteractivePage() {
  return (
    <div>
      <h1>My Page with an Interactive Element</h1>
      <InteractiveCounter />
    </div>
  );
}
``````typescript
// app/interactive-page/_components/InteractiveCounter.tsx (Client Component)
'use client';

import { useState } from 'react';

export default function InteractiveCounter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>Count: {count}</button>;
}
```

### 3. The `metadata.title` Field in Detail

The `title` tag is arguably the most important piece of metadata for SEO. The Metadata API offers advanced control over how titles are generated across your application. The `title` can be a simple string or a more powerful object.

#### Setting Title with a String

The most basic usage. This directly sets the `<title>` tag for the page.
```typescript
// app/contact/page.tsx
export const metadata = {
  title: 'Contact Us', // Renders: <title>Contact Us</title>
};
```

#### Advanced Title Control with an Object

For more complex scenarios, you can define `title` as an object with the following properties: `default`, `template`, and `absolute`.

**1. `default` - The Fallback Title**

Used in a `layout.tsx`, `title.default` provides a fallback title for any child routes that do not define their own title.
```typescript
// app/layout.tsx
export const metadata = {
  title: {
    default: 'Acme Inc. - The Future of Innovation',
  },
};
```
If `app/blog/page.tsx` has no `title` defined, it will inherit this default value.

**2. `template` - For Consistent Title Formatting**

The `template` property is incredibly useful for maintaining a consistent brand identity in your titles. It defines a pattern where `%s` is replaced by the child segment's title. This is best used in a root layout.

```typescript
// app/layout.tsx
export const metadata = {
  title: {
    template: '%s | Acme Inc.', // The child title will be injected here
    default: 'Acme Inc.', // Fallback for pages without a title (e.g., the homepage)
  },
};
```

```typescript
// app/products/page.tsx
export const metadata = {
  title: 'Our Products',
};
// Final Rendered Title: <title>Our Products | Acme Inc.</title>
```

**3. `absolute` - For Complete Overrides**

Sometimes, you need a title that completely ignores the `template` defined in a parent layout. This is where `absolute` comes in. It's perfect for the homepage or special campaign pages.

```typescript
// Assuming layout.tsx has a title.template of '%s | Acme Inc.'

// app/page.tsx (Homepage)
export const metadata = {
  title: {
    absolute: 'Acme Inc. - The Future of Innovation, Today.',
  },
};
// Final Rendered Title: <title>Acme Inc. - The Future of Innovation, Today.</title>
// The parent template is completely ignored.
```