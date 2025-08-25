## Next.js In-Depth Notes for the Senior Frontend Developer: Navigation

Navigation is the backbone of any multi-page application. Next.js provides a specialized routing system that optimizes performance and developer experience. Understanding the nuances between declarative navigation with the `<Link>` component and programmatic navigation is key for a senior developer.

### 1. Implementing Client-Side Navigation with the `Link` Component

The primary method for navigating between routes in Next.js is the built-in `<Link>` component. It's crucial to understand that using `<Link>` is fundamentally different from using a standard `<a>` tag.

**Key Concept: `<Link>` vs. `<a>`**
*   **`<a>` tag:** A standard anchor tag causes a full page reload. The browser requests a new HTML document from the server, all client-side state (like React component state) is lost, and the entire page is re-rendered. This is inefficient for Single-Page Application (SPA)-like experiences.
*   **`<Link>` component:** The Next.js `<Link>` component enables client-side navigation. When a link is clicked, Next.js prevents the browser's default full-page reload. Instead, it fetches only the necessary components for the new route and intelligently swaps them on the client side. This results in faster transitions and preservation of shared layout state.

**Prefetching:** By default, Next.js prefetches the code for any route that appears within the user's viewport. This means that when the user clicks the link, the transition is nearly instantaneous because the required JavaScript is likely already loaded.

#### Adding Links to Navigate Between Pages

Import the `Link` component from `next/link` and use it with an `href` prop.

**Example (`/components/Header.tsx`):**
```typescript
import Link from 'next/link';

export default function Header() {
  return (
    <header className="flex justify-between items-center p-4 bg-gray-800 text-white">
      <Link href="/" className="text-xl font-bold">
        MyApp
      </Link>
      <nav className="flex gap-4">
        <Link href="/about">About</Link>
        <Link href="/products">Products</Link>
        <Link href="/blog">Blog</Link>
      </nav>
    </header>
  );
}
```

#### Understanding the `replace` Prop

By default, `Link` uses `router.push()`, which adds a new entry to the browser's history stack. The `replace` prop changes this behavior to `router.replace()`, which replaces the current history entry instead.

*   **When to use `replace`:** This is useful for flows where you don't want the user to be able to go "back" to the previous page, such as after a login redirect or when updating filters on a search page.

**Example:**
```typescript
// After a successful action, navigating to the dashboard
// The user shouldn't be able to click "back" to the login form.
<Link href="/dashboard" replace>
  Go to Dashboard
</Link>
```

#### Accessing `params` and `searchParams`

How you access route parameters depends on whether you are in a Server or Client Component.

**In Server Components (`page.tsx`, `layout.tsx`):**
Route parameters are passed directly as props to the component. This is the most efficient method as it requires no client-side hooks.

```typescript
// app/blog/[slug]/page.tsx
// URL: /blog/my-first-post?showComments=true

type Props = {
  params: { slug: string };
  searchParams: { [key: string]: string | string[] | undefined };
};

export default function BlogPostPage({ params, searchParams }: Props) {
  const showComments = searchParams?.showComments === 'true';

  return (
    <div>
      <h1>Blog Post: {params.slug}</h1>
      {showComments && <p>Displaying comments...</p>}
    </div>
  );
}
```

**In Client Components:**
You must use hooks from `next/navigation` to access route information.

*   `useParams()`: Returns an object containing the dynamic route parameters.
*   `useSearchParams()`: Returns a read-only `URLSearchParams` object.
*   `usePathname()`: Returns the current URL pathname as a string.

```typescript
// app/components/ClientComponent.tsx
'use client';

import { useParams, useSearchParams, usePathname } from 'next/navigation';

export default function ClientRouteInfo() {
  const params = useParams(); // For /blog/my-post, returns { slug: 'my-post' }
  const searchParams = useSearchParams(); // For ?sort=asc, use searchParams.get('sort')
  const pathname = usePathname(); // Returns '/blog/my-post'

  return (
    <div>
      <p>Current Path: {pathname}</p>
      <p>Post Slug: {params.slug}</p>
      <p>Sort Order: {searchParams.get('sort')}</p>
    </div>
  );
}
```

### 2. Programmatic Navigation

Programmatic navigation is essential for redirecting users based on an event, such as a form submission, a timer, or an authentication check.

#### Server-Side Redirection: `redirect` function

When you need to redirect on the server *before* any HTML is rendered, use the `redirect` function from `next/navigation`. This is typically used in Server Actions, route handlers, or at the beginning of a Server Component's rendering logic. It works by sending an HTTP 307 (Temporary Redirect) response to the browser.

*   **Important:** This function throws a special `NEXT_REDIRECT` error, so it must be called outside of a `try...catch` block.

**Use Case: Protecting a Server-Side Route**
```typescript
// app/dashboard/page.tsx
import { redirect } from 'next/navigation';
import { getCurrentUser } from '@/lib/auth'; // Your auth logic

export default async function DashboardPage() {
  const user = await getCurrentUser();

  if (!user) {
    // Redirect unauthenticated users to the login page
    redirect('/login');
  }

  return (
    <div>
      <h1>Welcome to your Dashboard, {user.name}</h1>
      {/* ...dashboard content */}
    </div>
  );
}```

#### Client-Side Navigation: `useRouter` Hook

For all client-side programmatic navigation, you use the `useRouter` hook from `next/navigation`. This hook must be used within a Client Component (`'use client'`).

*   `router.push(href)`: Navigates to a new URL and adds it to the history stack.
*   `router.replace(href)`: Navigates to a new URL but replaces the current entry in the history stack.
*   `router.back()`: Navigates to the previous entry in the history stack.
*   `router.forward()`: Navigates to the next entry in the history stack.
*   `router.refresh()`: A powerful new feature in the App Router. It performs a "soft refresh" of the current route. This re-fetches data requests on the server (for Server Components) and re-renders them without losing the state of unrelated Client Components on the page. It's excellent for invalidating server data after a mutation (e.g., updating a user's profile).

**Example: Handling a Form Submission**
```typescript
// app/settings/ProfileForm.tsx
'use client';

import { useRouter } from 'next/navigation';

export default function ProfileForm() {
  const router = useRouter();

  async function handleSubmit(event: React.FormEvent<HTMLFormElement>) {
    event.preventDefault();
    const formData = new FormData(event.currentTarget);
    
    // API call to update the user profile
    await fetch('/api/user', { method: 'PUT', body: formData });

    // Option 1: Navigate to a different page
    // router.push('/dashboard');

    // Option 2: Refresh the current page to show the updated data
    // This will re-run the server data fetches for the current route.
    router.refresh(); 
  }

  return (
    <form onSubmit={handleSubmit}>
      {/* ...form fields... */}
      <button type="submit">Update Profile</button>
    </form>
  );
}
```