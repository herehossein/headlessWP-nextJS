# فصل ۹: مدیریت مسیرها، صفحات پویا و رندرینگ ایستا (Routing, Dynamic Segments & Static Generation)

در هر وب‌سایت محتوایی، صفحات پویا (Dynamic Routes) هسته اصلی تجربه کاربری هستند: صفحه اختصاصی هر مقاله بر اساس نام مستعار (`/blog/[slug]`)، برگه‌های ایستا (`/[slug]`)، و صفحات آرشیو دسته‌بندی‌ها (`/category/[slug]`).

در **Next.js 16.4.5**، دسترسی به پارامترهای مسیر به صورت ناهمگام (Async Promise) انجام می‌پذیرد. همچنین با ترکیب تابع قدرتمند **`generateStaticParams`** با **Apollo Client 4.3.7**، می‌توانیم صدها صفحه را در زمان بیلد تولید کرده و صفحات جدید را با سرعت تولید افزایشی ایستا (ISR) رندر کنیم.

در این فصل، ساختار صفحات پویا، پیش‌رندرینگ SSG با Apollo Client و پیاده‌سازی صفحه ۴۰۴ را قدم‌به‌قدم پیاده می‌کنیم.

---

## ۱. مدیریت پارامترهای ناهمگام (Async Params) در Next.js 16

از نسخه Next.js 15 و با الزام قطعی در **Next.js 16**، خصوصیات `params` و `searchParams` در کامپوننت‌های صفحه و توابع متادیتا یک **Promise** هستند و باید با `await` حل شوند (دسترسی همگام قدیمی به طور کامل برچیده شده است):

```typescript
interface PageProps {
  params: Promise<{
    slug: string;
  }>;
}

export default async function BlogPostPage({ params }: PageProps) {
  // در Next.js 16.4.5 حتماً باید await شود:
  const { slug } = await params;
  
  // ادامه کار...
}
```

---

## ۲. پیاده‌سازی صفحه تک‌پست با Apollo Client (`app/blog/[slug]/page.tsx`)

این صفحه با دریافت Slug، اطلاعات مقاله و بلوک‌های محتوا را از طریق **Apollo Client 4.3.7** از سرور وردپرس 7.1.4 واکشی کرده و به کامپوننت `BlockRenderer` تحویل می‌دهد:

```tsx
// app/blog/[slug]/page.tsx
import { notFound } from "next/navigation";
import Image from "next/image";
import { gql } from "@apollo/client";
import { query } from "@/lib/apollo-client";
import { BlockRenderer } from "@/components/blocks/BlockRenderer";

interface BlogPostData {
  post: {
    id: string;
    databaseId: number;
    title: string;
    slug: string;
    date: string;
    author: {
      node: {
        name: string;
        avatar?: {
          url: string;
        };
      };
    };
    featuredImage?: {
      node: {
        sourceUrl: string;
        altText?: string;
      };
    };
    editorBlocks: any[];
  } | null;
}

const GET_POST_BY_SLUG = gql`
  query GetPostBySlug($slug: ID!) {
    post(id: $slug, idType: SLUG) {
      id
      databaseId
      title
      slug
      date
      author {
        node {
          name
          avatar {
            url
          }
        }
      }
      featuredImage {
        node {
          sourceUrl
          altText
        }
      }
      editorBlocks(flat: false) {
        __typename
        name
        ... on CoreParagraph {
          attributes {
            content
          }
        }
        ... on CoreHeading {
          attributes {
            content
            level
          }
        }
        ... on CoreImage {
          attributes {
            url
            alt
            width
            height
            caption
          }
        }
        ... on CoreGallery {
          innerBlocks {
            ... on CoreImage {
              attributes {
                url
                alt
                caption
              }
            }
          }
        }
        ... on CoreList {
          innerBlocks {
            ... on CoreListItem {
              attributes {
                content
              }
            }
          }
        }
      }
    }
  }
`;

interface PageProps {
  params: Promise<{
    slug: string;
  }>;
}

export default async function BlogPostPage({ params }: PageProps) {
  const { slug } = await params;
  const decodedSlug = decodeURIComponent(slug);

  const { data } = await query<BlogPostData>({
    query: GET_POST_BY_SLUG,
    variables: { slug: decodedSlug },
    context: {
      fetchOptions: {
        next: {
          tags: [`post-${decodedSlug}`, "posts"],
        },
      },
    },
  });

  if (!data?.post) {
    notFound();
  }

  const post = data.post;

  return (
    <article className="max-w-4xl mx-auto py-8">
      <header className="mb-8 border-b border-gray-200 dark:border-gray-800 pb-8">
        <h1 className="text-3xl sm:text-4xl lg:text-5xl font-black text-gray-900 dark:text-white leading-tight mb-6">
          {post.title}
        </h1>

        <div className="flex items-center gap-4 text-sm text-gray-600 dark:text-gray-400">
          {post.author?.node?.avatar && (
            <Image
              src={post.author.node.avatar.url}
              alt={post.author.node.name}
              width={40}
              height={40}
              className="rounded-full"
            />
          )}
          <div>
            <span className="font-semibold text-gray-900 dark:text-white block">
              {post.author?.node?.name}
            </span>
            <time dateTime={post.date}>
              {new Date(post.date).toLocaleDateString("fa-IR", {
                year: "numeric",
                month: "long",
                day: "numeric",
              })}
            </time>
          </div>
        </div>
      </header>

      {post.featuredImage?.node && (
        <div className="relative aspect-video w-full rounded-2xl overflow-hidden mb-10 shadow-md">
          <Image
            src={post.featuredImage.node.sourceUrl}
            alt={post.featuredImage.node.altText || post.title}
            fill
            priority
            sizes="(max-width: 1200px) 100vw, 1200px"
            className="object-cover"
          />
        </div>
      )}

      <section className="mt-8">
        <BlockRenderer blocks={post.editorBlocks} />
      </section>
    </article>
  );
}
```

---

## ۳. پیش‌رندرینگ در زمان بیلد با `generateStaticParams` و Apollo Client

```tsx
// افزودن به انتهای فایل app/blog/[slug]/page.tsx:

interface AllSlugsData {
  posts: {
    nodes: Array<{
      slug: string;
    }>;
  };
}

const GET_ALL_SLUGS = gql`
  query GetAllPostSlugs {
    posts(first: 100) {
      nodes {
        slug
      }
    }
  }
`;

export async function generateStaticParams() {
  try {
    const { data } = await query<AllSlugsData>({
      query: GET_ALL_SLUGS,
    });

    return data.posts.nodes.map((post) => ({
      slug: post.slug,
    }));
  } catch (error) {
    console.error("خطا در واکشی اسلاگ‌ها حین بیلد:", error);
    return [];
  }
}

export const dynamicParams = true;
```

---

## ۴. طراحی صفحه خطای ۴۰۴ سفارشی (`app/not-found.tsx`)

```tsx
// app/not-found.tsx
import Link from "next/link";

export default function NotFound() {
  return (
    <div className="py-24 text-center max-w-lg mx-auto">
      <span className="text-6xl font-black text-blue-600 dark:text-blue-400 block mb-4">
        ۴۰۴
      </span>
      <h1 className="text-3xl font-extrabold mb-3">صفحه مورد نظر یافت نشد</h1>
      <p className="text-gray-600 dark:text-gray-400 mb-8 text-sm leading-relaxed">
        مطلبی که به دنبال آن بودید ممکن است حذف شده یا آدرس آن تغییر یافته باشد.
      </p>
      <Link
        href="/blog"
        className="inline-flex items-center justify-center px-6 py-3 rounded-xl bg-blue-600 hover:bg-blue-700 text-white font-semibold text-sm transition-colors shadow-sm"
      >
        بازگشت به وبلاگ
      </Link>
    </div>
  );
}
```

---

## ۵. خلاصه فصل

در این فصل:
- کار با پارامترهای ناهمگام را در **Next.js 16.4.5** تثبیت کردیم.
- صفحه تک‌پست را با **Apollo Client 4.3.7** و تگ‌های کش برپا کردیم.
- با تابع `generateStaticParams` تولید ایستا (SSG) را برای مقالات پربازدید در زمان بیلد فعال ساختیم.

در ادامه، نحوه مدیریت پیش‌نمایش زنده در حالت **Draft Mode** را در فصل یازدهم و سپس سیستم جامع احراز هویت را در فصل دوازدهم بررسی خواهیم کرد.
