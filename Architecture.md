# Architecture du Projet Astro

## Vue d'ensemble

Ce document décrit l'architecture, les conventions de code et les bonnes pratiques à suivre pour maintenir la cohérence et la qualité du projet Astro.

## Structure des dossiers

```
src/
├── components/          # Composants Astro et framework
│   ├── astro/          # Composants Astro natifs (.astro)
│   │   ├── shared/     # Composants partagés
│   │   └── [Feature]/  # Composants par fonctionnalité
│   ├── react/          # Composants React (si utilisé)
│   ├── vue/            # Composants Vue (si utilisé)
│   └── svelte/         # Composants Svelte (si utilisé)
├── layouts/            # Layouts de pages
├── pages/              # Pages du site (routage basé sur fichiers)
│   ├── index.astro     # Page d'accueil
│   ├── blog/           # Section blog
│   │   ├── index.astro
│   │   └── [slug].astro
│   └── api/            # Routes API (endpoints)
├── content/            # Collections de contenu
│   ├── blog/           # Articles de blog
│   ├── authors/        # Auteurs
│   └── config.ts       # Configuration des collections
├── styles/             # Styles globaux
│   └── global.css
├── lib/                # Bibliothèques et utilitaires
├── utils/              # Fonctions utilitaires
├── types/              # Types TypeScript
├── constants/          # Constantes
├── data/               # Données statiques
└── env.d.ts           # Déclarations TypeScript pour Astro

public/                 # Fichiers statiques (non traités)
├── images/
├── fonts/
└── favicon.svg

astro.config.mjs        # Configuration Astro
tsconfig.json           # Configuration TypeScript
```

## Spécificités Astro

### Architecture Zero-JS par défaut

Astro envoie **zéro JavaScript au client par défaut** . Le JavaScript n'est envoyé que si vous utilisez des directives client explicites.

### Islands Architecture

Astro utilise l'architecture "Islands" : seuls les composants interactifs chargent du JavaScript.

```astro
---
// Composant statique (pas de JS)
import Header from '../components/Header.astro';

// Composant interactif (avec JS)
import Counter from '../components/Counter.react';
---

<Header />
<!-- Pas de JS envoyé -->

<Counter client:load />
<!-- JS envoyé uniquement pour Counter -->
```

### Directives client

- `client:load` - Charge immédiatement
- `client:idle` - Charge quand le navigateur est idle
- `client:visible` - Charge quand visible dans le viewport
- `client:media` - Charge selon une media query
- `client:only` - Skip SSR, rend uniquement côté client

## Pages et routage

### Routage basé sur fichiers

```
src/pages/
├── index.astro              → /
├── about.astro              → /about
├── blog/
│   ├── index.astro          → /blog
│   ├── [slug].astro         → /blog/mon-article
│   └── [...slug].astro      → /blog/2024/01/article (catch-all)
├── products/
│   └── [id].astro           → /products/123
└── api/
    └── newsletter.ts        → /api/newsletter (endpoint API)
```

### Page simple

```astro
---
// src/pages/about.astro
import Layout from '../layouts/Layout.astro';
import Hero from '../components/astro/Hero.astro';

const title = "À propos";
---

<Layout title={title}>
  <Hero title={title} />
  <section>
    <h1>À propos de nous</h1>
    <p>Contenu statique, zéro JavaScript envoyé.</p>
  </section>
</Layout>
```

### Page dynamique avec getStaticPaths

```astro
---
// src/pages/blog/[slug].astro
import { getCollection } from 'astro:content';
import Layout from '../../layouts/Layout.astro';
import BlogPost from '../../components/astro/BlogPost.astro';

export async function getStaticPaths() {
  const posts = await getCollection('blog');

  return posts.map(post => ({
    params: { slug: post.slug },
    props: { post },
  }));
}

const { post } = Astro.props;
const { Content } = await post.render();
---

<Layout title={post.data.title}>
  <BlogPost post={post}>
    <Content />
  </BlogPost>
</Layout>
```

### Page avec SSR (Server-Side Rendering)

```astro
---
// src/pages/products/[id].astro
export const prerender = false; // Active SSR pour cette page

const { id } = Astro.params;

// Appel API côté serveur
const response = await fetch(`https://api.example.com/products/${id}`);
const product = await response.json();
---

<Layout title={product.name}>
  <h1>{product.name}</h1>
  <p>{product.description}</p>
  <p>Prix : {product.price}€</p>
</Layout>
```

## Composants Astro

### Structure d'un composant .astro

```astro
---
// src/components/astro/Card.astro

// 1. Imports
import { Image } from 'astro:assets';

// 2. Interface des props (TypeScript)
interface Props {
  title: string;
  description: string;
  image?: string;
  href?: string;
}

// 3. Récupération des props
const { title, description, image, href } = Astro.props;

// 4. Logique côté serveur (s'exécute au build)
const formattedDate = new Date().toLocaleDateString('fr-FR');
---

<!-- 5. Template HTML -->
<div class="card">
  {image && (
    <Image src={image} alt={title} width={400} height={300} />
  )}
  <h3>{title}</h3>
  <p>{description}</p>
  {href && (
    <a href={href}>En savoir plus</a>
  )}
  <time>{formattedDate}</time>
</div>

<!-- 6. Styles scopés -->
<style>
  .card {
    padding: 1rem;
    border: 1px solid #ddd;
    border-radius: 8px;
  }

  h3 {
    margin: 0 0 0.5rem;
    color: var(--color-primary);
  }
</style>

<!-- 7. Script côté client (optionnel) -->
<script>
  // Ce JavaScript est envoyé au client et s'exécute dans le navigateur
  console.log('Card mounted');
</script>
```

### Composant avec slots

```astro
---
// src/components/astro/Card.astro
interface Props {
  title: string;
  variant?: 'primary' | 'secondary';
}

const { title, variant = 'primary' } = Astro.props;
---

<div class={`card card--${variant}`}>
  <header>
    <h3>{title}</h3>
    <slot name="header" />
  </header>

  <div class="card__content">
    <slot /> <!-- Slot par défaut -->
  </div>

  <footer>
    <slot name="footer" />
  </footer>
</div>

<style>
  .card {
    border: 1px solid #ddd;
  }

  .card--primary {
    border-color: var(--color-primary);
  }

  .card--secondary {
    border-color: var(--color-secondary);
  }
</style>
```

### Utilisation des slots

```astro
---
import Card from '../components/astro/Card.astro';
---

<Card title="Mon Titre">
  <div slot="header">
    <span>Badge</span>
  </div>

  <!-- Contenu principal (slot par défaut) -->
  <p>Contenu de la carte</p>

  <div slot="footer">
    <button>Action</button>
  </div>
</Card>
```

## Layouts

### Layout de base

```astro
---
// src/layouts/Layout.astro
import Header from '../components/astro/shared/Header.astro';
import Footer from '../components/astro/shared/Footer.astro';

interface Props {
  title: string;
  description?: string;
  image?: string;
}

const {
  title,
  description = 'Description par défaut',
  image = '/og-image.jpg'
} = Astro.props;

const canonicalURL = new URL(Astro.url.pathname, Astro.site);
---

<!DOCTYPE html>
<html lang="fr">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width" />
    <link rel="icon" type="image/svg+xml" href="/favicon.svg" />

    <!-- SEO -->
    <title>{title}</title>
    <meta name="description" content={description} />
    <link rel="canonical" href={canonicalURL} />

    <!-- Open Graph -->
    <meta property="og:title" content={title} />
    <meta property="og:description" content={description} />
    <meta property="og:image" content={new URL(image, Astro.site)} />
    <meta property="og:url" content={canonicalURL} />

    <!-- Twitter Card -->
    <meta name="twitter:card" content="summary_large_image" />
    <meta name="twitter:title" content={title} />
    <meta name="twitter:description" content={description} />
    <meta name="twitter:image" content={new URL(image, Astro.site)} />

    <slot name="head" />
  </head>
  <body>
    <Header />
    <main>
      <slot />
    </main>
    <Footer />
  </body>
</html>

<style is:global>
  :root {
    --color-primary: #0ea5e9;
    --color-text: #1f2937;
    --font-body: system-ui, sans-serif;
  }

  * {
    box-sizing: border-box;
  }

  body {
    margin: 0;
    font-family: var(--font-body);
    color: var(--color-text);
  }
</style>
```

### Layout pour blog

```astro
---
// src/layouts/BlogLayout.astro
import Layout from './Layout.astro';
import BlogHeader from '../components/astro/BlogHeader.astro';
import TableOfContents from '../components/astro/TableOfContents.astro';

interface Props {
  title: string;
  description: string;
  pubDate: Date;
  author: string;
  image?: string;
  headings?: Array<{ depth: number; text: string; slug: string }>;
}

const { title, description, pubDate, author, image, headings } = Astro.props;
---

<Layout title={title} description={description} image={image}>
  <article class="blog-post">
    <BlogHeader
      title={title}
      author={author}
      pubDate={pubDate}
      image={image}
    />

    <div class="blog-content">
      {headings && headings.length > 0 && (
        <aside class="toc">
          <TableOfContents headings={headings} />
        </aside>
      )}

      <div class="prose">
        <slot />
      </div>
    </div>
  </article>
</Layout>

<style>
  .blog-post {
    max-width: 1200px;
    margin: 0 auto;
    padding: 2rem 1rem;
  }

  .blog-content {
    display: grid;
    grid-template-columns: 250px 1fr;
    gap: 2rem;
    margin-top: 2rem;
  }

  .prose {
    max-width: 65ch;
  }

  @media (max-width: 768px) {
    .blog-content {
      grid-template-columns: 1fr;
    }

    .toc {
      display: none;
    }
  }
</style>
```

## Content Collections

### Configuration des collections

```typescript
// src/content/config.ts
import { defineCollection, z } from "astro:content";

const blog = defineCollection({
  type: "content",
  schema: z.object({
    title: z.string(),
    description: z.string(),
    pubDate: z.date(),
    updatedDate: z.date().optional(),
    author: z.string(),
    image: z.string().optional(),
    tags: z.array(z.string()).default([]),
    draft: z.boolean().default(false),
  }),
});

const authors = defineCollection({
  type: "data",
  schema: z.object({
    name: z.string(),
    bio: z.string(),
    avatar: z.string(),
    twitter: z.string().optional(),
    github: z.string().optional(),
  }),
});

export const collections = {
  blog,
  authors,
};
```

### Article de blog (Markdown)

```markdown
---
# src/content/blog/mon-article.md
title: "Mon Premier Article"
description: "Ceci est mon premier article de blog"
pubDate: 2024-01-15
author: "John Doe"
image: "/images/article-1.jpg"
tags: ["astro", "web", "javascript"]
---

# Mon Article

Contenu de l'article en Markdown...

## Section 1

Texte de la section 1.

## Section 2

Texte de la section 2.
```

### Utilisation des collections

```astro
---
// src/pages/blog/index.astro
import { getCollection } from 'astro:content';
import Layout from '../../layouts/Layout.astro';
import BlogCard from '../../components/astro/BlogCard.astro';

// Récupérer tous les articles publiés
const posts = (await getCollection('blog'))
  .filter(post => !post.data.draft)
  .sort((a, b) => b.data.pubDate.valueOf() - a.data.pubDate.valueOf());
---

<Layout title="Blog">
  <h1>Articles du blog</h1>

  <div class="posts-grid">
    {posts.map(post => (
      <BlogCard
        title={post.data.title}
        description={post.data.description}
        pubDate={post.data.pubDate}
        href={`/blog/${post.slug}`}
        image={post.data.image}
        tags={post.data.tags}
      />
    ))}
  </div>
</Layout>

<style>
  .posts-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
    gap: 2rem;
    margin-top: 2rem;
  }
</style>
```

## Composants de frameworks UI (React, Vue, Svelte)

### Configuration dans astro.config.mjs

```javascript
// astro.config.mjs
import { defineConfig } from "astro/config";
import react from "@astrojs/react";
import vue from "@astrojs/vue";
import svelte from "@astrojs/svelte";

export default defineConfig({
  integrations: [react(), vue(), svelte()],
});
```

### Composant React interactif

```tsx
// src/components/react/Counter.tsx
import { useState } from "react";

interface Props {
  initialCount?: number;
}

export default function Counter({ initialCount = 0 }: Props) {
  const [count, setCount] = useState(initialCount);

  return (
    <div className="counter">
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>Increment</button>
      <button onClick={() => setCount(count - 1)}>Decrement</button>
    </div>
  );
}
```

### Utilisation dans une page Astro

```astro
---
// src/pages/demo.astro
import Layout from '../layouts/Layout.astro';
import Counter from '../components/react/Counter';
import SearchBar from '../components/vue/SearchBar.vue';
---

<Layout title="Demo">
  <h1>Composants interactifs</h1>

  <!-- Charge immédiatement -->
  <Counter client:load initialCount={5} />

  <!-- Charge quand visible -->
  <SearchBar client:visible />

  <!-- Charge uniquement sur mobile -->
  <MobileMenu client:media="(max-width: 768px)" />

  <!-- Skip SSR, client uniquement -->
  <BrowserOnlyComponent client:only="react" />
</Layout>
```

## API Routes (Endpoints)

### Endpoint GET

```typescript
// src/pages/api/posts.ts
import type { APIRoute } from "astro";
import { getCollection } from "astro:content";

export const GET: APIRoute = async ({ params, request }) => {
  const posts = await getCollection("blog");

  return new Response(
    JSON.stringify({
      posts: posts.map((post) => ({
        slug: post.slug,
        title: post.data.title,
        description: post.data.description,
      })),
    }),
    {
      status: 200,
      headers: {
        "Content-Type": "application/json",
      },
    }
  );
};
```

### Endpoint POST

```typescript
// src/pages/api/newsletter.ts
import type { APIRoute } from "astro";

export const POST: APIRoute = async ({ request }) => {
  try {
    const data = await request.json();
    const { email } = data;

    // Validation
    if (!email || !email.includes("@")) {
      return new Response(JSON.stringify({ error: "Email invalide" }), {
        status: 400,
      });
    }

    // Logique d'inscription (API externe, DB, etc.)
    await subscribeToNewsletter(email);

    return new Response(JSON.stringify({ message: "Inscription réussie" }), {
      status: 200,
    });
  } catch (error) {
    return new Response(JSON.stringify({ error: "Erreur serveur" }), {
      status: 500,
    });
  }
};
```

### Utilisation côté client

```astro
---
// Aucun code serveur nécessaire
---

<form id="newsletter-form">
  <input type="email" name="email" required />
  <button type="submit">S'inscrire</button>
</form>

<script>
  const form = document.getElementById('newsletter-form');

  form?.addEventListener('submit', async (e) => {
    e.preventDefault();

    const formData = new FormData(e.target as HTMLFormElement);
    const email = formData.get('email');

    const response = await fetch('/api/newsletter', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({ email }),
    });

    const result = await response.json();
    alert(result.message || result.error);
  });
</script>
```

## Images avec Astro Assets

### Import et optimisation d'images

```astro
---
import { Image } from 'astro:assets';
import heroImage from '../images/hero.jpg';
---

<!-- Image locale optimisée -->
<Image
  src={heroImage}
  alt="Hero"
  width={1200}
  height={600}
  format="webp"
  quality={80}
/>

<!-- Image distante -->
<Image
  src="https://example.com/image.jpg"
  alt="Remote"
  width={800}
  height={400}
  inferSize
/>

<!-- Image responsive -->
<Image
  src={heroImage}
  alt="Hero"
  widths={[400, 800, 1200]}
  sizes="(max-width: 600px) 400px, (max-width: 900px) 800px, 1200px"
/>
```

### Composant Picture pour art direction

```astro
---
import { Picture } from 'astro:assets';
import desktopImage from '../images/hero-desktop.jpg';
import mobileImage from '../images/hero-mobile.jpg';
---

<Picture
  src={desktopImage}
  alt="Hero"
  widths={[400, 800, 1200]}
  formats={['avif', 'webp']}
  pictureAttributes={{
    style: 'width: 100%; height: auto;'
  }}
/>
```

## Styles

### Styles scopés

Les styles dans un composant `.astro` sont **automatiquement scopés** :

```astro
<div class="container">
  <h1>Titre</h1>
</div>

<style>
  /* Ces styles s'appliquent UNIQUEMENT à ce composant */
  .container {
    padding: 2rem;
  }

  h1 {
    color: blue;
  }
</style>
```

### Styles globaux

```astro
<style is:global>
  /* Ces styles sont globaux */
  body {
    margin: 0;
    font-family: system-ui;
  }

  h1, h2, h3 {
    line-height: 1.2;
  }
</style>
```

### Import de fichiers CSS

```astro
---
// Dans le frontmatter
import '../styles/global.css';
---
```

### CSS Modules (optionnel)

```astro
---
import styles from './Card.module.css';
---

<div class={styles.card}>
  <h3 class={styles.title}>Titre</h3>
</div>
```

### Tailwind CSS

```javascript
// astro.config.mjs
import { defineConfig } from "astro/config";
import tailwind from "@astrojs/tailwind";

export default defineConfig({
  integrations: [tailwind()],
});
```

```astro
<div class="bg-blue-500 text-white p-4 rounded-lg">
  <h1 class="text-2xl font-bold">Avec Tailwind</h1>
</div>
```

## Scripts côté client

### Script inline

```astro
<button id="toggle">Toggle</button>

<script>
  // Ce code s'exécute dans le navigateur
  const button = document.getElementById('toggle');

  button?.addEventListener('click', () => {
    document.body.classList.toggle('dark-mode');
  });
</script>
```

### Script is:inline (non traité)

```astro
<script is:inline>
  // Ce script n'est PAS traité par Vite
  // Utile pour du code qui doit s'exécuter immédiatement
  if (localStorage.getItem('theme') === 'dark') {
    document.documentElement.classList.add('dark');
  }
</script>
```

### Script externe

```astro
<script src="/scripts/analytics.js"></script>
```

### Script avec define:vars (passer des variables serveur)

```astro
---
const apiKey = import.meta.env.PUBLIC_API_KEY;
const config = { timeout: 5000 };
---

<script define:vars={{ apiKey, config }}>
  // Variables serveur disponibles côté client
  console.log(apiKey);
  console.log(config.timeout);
</script>
```

## Conventions de nommage

### Fichiers et dossiers

#### Pages et routes

- **Pages** : kebab-case (`about.astro`, `privacy-policy.astro`)
- **Pages dynamiques** : `[param].astro`, `[...path].astro`
- **Endpoints API** : kebab-case avec extension `.ts` (`newsletter.ts`, `contact-form.ts`)

#### Composants

- **Composants Astro** : PascalCase (`Header.astro`, `BlogPost.astro`, `UserCard.astro`)
- **Composants React/Vue/Svelte** : PascalCase avec extension appropriée (`.tsx`, `.vue`, `.svelte`)
- **Dossiers de composants** : PascalCase (`Header/`, `BlogPost/`)

#### Layouts

- **Layouts** : PascalCase avec suffixe Layout (`Layout.astro`, `BlogLayout.astro`)

#### Content Collections

- **Fichiers de contenu** : kebab-case (`mon-article.md`, `getting-started.mdx`)
- **Dossiers de collections** : kebab-case (`blog/`, `authors/`, `products/`)

#### Autres

- **Utilitaires** : camelCase (`formatDate.ts`, `validateEmail.ts`)
- **Types** : PascalCase (`User.ts`, `BlogPost.ts`)
- **Constantes** : UPPERCASE_SNAKE_CASE (`API_ENDPOINTS.ts`, `CONFIG.ts`)

### Variables et fonctions

- **Variables** : camelCase (`userName`, `isActive`, `postList`)
- **Constantes** : UPPERCASE_SNAKE_CASE (`API_BASE_URL`, `MAX_ITEMS`)
- **Fonctions** : camelCase (`formatDate`, `fetchPosts`, `validateInput`)
- **Types/Interfaces** : PascalCase (`User`, `BlogPost`, `ApiResponse`)

```typescript
// ✅ Bon
const userName = "John";
const API_KEY = "secret";

function formatDate(date: Date): string {
  return date.toLocaleDateString();
}

interface User {
  id: string;
  name: string;
}
```

### Composants

```astro
---
// ✅ Bon - Props typées avec interface
interface Props {
  title: string;
  description?: string;
  isActive?: boolean;
}

const { title, description, isActive = false } = Astro.props;
---
```

## Variables d'environnement

### Configuration

```env
# .env
PUBLIC_API_URL=https://api.example.com
SECRET_API_KEY=secret_key_here
```

### Utilisation

```astro
---
// Variables PUBLIC_ accessibles côté client
const apiUrl = import.meta.env.PUBLIC_API_URL;

// Variables sans PUBLIC_ accessibles uniquement côté serveur
const apiKey = import.meta.env.SECRET_API_KEY;
---

<script>
  // Côté client : seules les variables PUBLIC_ sont disponibles
  console.log(import.meta.env.PUBLIC_API_URL); // ✅ Fonctionne
  console.log(import.meta.env.SECRET_API_KEY); // ❌ undefined
</script>
```

### Types pour variables d'environnement

```typescript
// src/env.d.ts
/// <reference types="astro/client" />

interface ImportMetaEnv {
  readonly PUBLIC_API_URL: string;
  readonly SECRET_API_KEY: string;
}

interface ImportMeta {
  readonly env: ImportMetaEnv;
}
```

## Modes de rendu (Output)

### Static (par défaut)

```javascript
// astro.config.mjs
export default defineConfig({
  output: "static", // Par défaut
});
```

Toutes les pages sont générées au build. Idéal pour les sites statiques.

### Server (SSR)

```javascript
// astro.config.mjs
export default defineConfig({
  output: "server",
  adapter: vercel(), // ou netlify(), node(), etc.
});
```

Toutes les pages sont rendues à la demande. Idéal pour du contenu dynamique.

### Hybrid

```javascript
// astro.config.mjs
export default defineConfig({
  output: "hybrid",
  adapter: vercel(),
});
```

Par défaut statique, mais certaines pages peuvent être SSR :

```astro
---
// Cette page sera rendue à la demande
export const prerender = false;
---
```

## Configuration Astro

### astro.config.mjs complet

```javascript
import { defineConfig } from "astro/config";
import react from "@astrojs/react";
import tailwind from "@astrojs/tailwind";
import sitemap from "@astrojs/sitemap";
import mdx from "@astrojs/mdx";

export default defineConfig({
  // URL du site en production
  site: "https://example.com",

  // Base path (si sous-domaine)
  base: "/",

  // Mode de sortie
  output: "static",

  // Intégrations
  integrations: [react(), tailwind(), sitemap(), mdx()],

  // Markdown
  markdown: {
    syntaxHighlight: "shiki",
    shikiConfig: {
      theme: "github-dark",
    },
  },

  // Vite config
  vite: {
    ssr: {
      noExternal: ["some-package"],
    },
  },

  // Image service
  image: {
    service: {
      entrypoint: "astro/assets/services/sharp",
    },
  },
});
```

## Bonnes pratiques

### Performance

1. **Utiliser des composants Astro** : Préférer `.astro` pour les composants statiques
2. **Directives client appropriées** : Utiliser `client:visible` ou `client:idle` au lieu de `client:load`
3. **Optimiser les images** : Toujours utiliser le composant `<Image />` ou `<Picture />`
4. **Limiter le JavaScript** : N'ajouter que le JavaScript nécessaire
5. **Lazy load** : Charger les composants lourds avec `client:visible`

```astro
<!-- ❌ Éviter -->
<HeavyComponent client:load />

<!-- ✅ Mieux -->
<HeavyComponent client:visible />
```

### SEO

1. **Utiliser Layout avec SEO** : Centraliser les meta tags
2. **Sitemap** : Installer `@astrojs/sitemap`
3. **RSS Feed** : Créer un feed RSS pour le blog
4. **Canonical URLs** : Toujours définir l'URL canonique
5. **Structured Data** : Ajouter JSON-LD pour le référencement

```astro
<script type="application/ld+json" set:html={JSON.stringify({
  "@context": "https://schema.org",
  "@type": "BlogPosting",
  "headline": title,
  "datePublished": pubDate,
  "author": {
    "@type": "Person",
    "name": author
  }
})} />
```

### Accessibilité

1. **HTML sémantique** : Utiliser les bonnes balises (`<article>`, `<nav>`, `<aside>`)
2. **Alt text** : Toujours fournir un alt pour les images
3. **ARIA labels** : Ajouter des labels pour les éléments interactifs
4. **Focus visible** : S'assurer que le focus est visible
5. **Contraste** : Maintenir un bon contraste de couleurs

```astro
<nav aria-label="Navigation principale">
  <ul>
    <li><a href="/">Accueil</a></li>
    <li><a href="/about">À propos</a></li>
  </ul>
</nav>

<button aria-label="Fermer le menu" aria-expanded="false">
  <span aria-hidden="true">×</span>
</button>
```

### TypeScript

1. **Typer les props** : Toujours définir une interface Props
2. **Typer les collections** : Utiliser Zod pour les schémas
3. **Types utilitaires** : Créer des types réutilisables
4. **Strict mode** : Activer le mode strict dans tsconfig.json

```astro
---
interface Props {
  title: string;
  tags?: string[];
  publishedAt: Date;
}

const { title, tags = [], publishedAt } = Astro.props;
---
```

### Organisation du code

1. **Composants Astro pour le statique** : Pas besoin de React/Vue si pas d'interactivité
2. **Un composant = une responsabilité** : Garder les composants focalisés
3. **Content Collections** : Utiliser pour tout contenu structuré
4. **Utilitaires réutilisables** : Extraire la logique commune
5. **Layouts cohérents** : Utiliser des layouts pour la structure commune

## Middleware (Astro 3.0+)

### Création d'un middleware

```typescript
// src/middleware.ts
import { defineMiddleware } from "astro:middleware";

export const onRequest = defineMiddleware(async (context, next) => {
  // Logique avant la requête
  console.log(`Request: ${context.url.pathname}`);

  // Ajouter des données au contexte
  context.locals.user = await getUserFromToken(context.request);

  // Continuer vers la page
  const response = await next();

  // Logique après la requête
  response.headers.set("X-Custom-Header", "value");

  return response;
});
```

### Utilisation dans les pages

```astro
---
// Accès aux données du middleware
const user = Astro.locals.user;
---

{user ? (
  <p>Bienvenue {user.name}</p>
) : (
  <p>Non connecté</p>
)}
```

## View Transitions (Astro 3.0+)

### Activer les transitions

```astro
---
// Dans le Layout
import { ViewTransitions } from 'astro:transitions';
---

<head>
  <ViewTransitions />
</head>
```

### Animations personnalisées

```astro
---
import { fade, slide } from 'astro:transitions';
---

<div transition:animate={fade({ duration: '0.3s' })}>
  Contenu avec fade
</div>

<article transition:animate={slide({ duration: '0.5s' })}>
  Contenu avec slide
</article>
```

### Persister des éléments entre pages

```astro
<header transition:persist>
  <!-- Ce header persiste entre les navigations -->
  <nav>...</nav>
</header>

<div transition:persist="player">
  <!-- Lecteur audio qui continue entre les pages -->
  <audio src="song.mp3" controls></audio>
</div>
```

## Testing

### Tests unitaires avec Vitest

```typescript
// src/utils/formatDate.test.ts
import { describe, it, expect } from "vitest";
import { formatDate } from "./formatDate";

describe("formatDate", () => {
  it("formats date correctly", () => {
    const date = new Date("2024-01-15");
    expect(formatDate(date)).toBe("15 janvier 2024");
  });
});
```

### Tests de composants React

```typescript
// src/components/react/Counter.test.tsx
import { render, screen, fireEvent } from "@testing-library/react";
import { describe, it, expect } from "vitest";
import Counter from "./Counter";

describe("Counter", () => {
  it("increments count on click", () => {
    render(<Counter initialCount={0} />);

    const button = screen.getByText("Increment");
    fireEvent.click(button);

    expect(screen.getByText("Count: 1")).toBeInTheDocument();
  });
});
```

### Configuration Vitest

```typescript
// vitest.config.ts
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    globals: true,
    environment: "jsdom",
  },
});
```

## Déploiement

### Build de production

```bash
# Build statique
npm run build

# Preview du build
npm run preview

# Build avec SSR (nécessite un adapter)
npm run build
```

### Adapters pour SSR

```javascript
// Vercel
import vercel from '@astrojs/vercel/serverless';

export default defineConfig({
  output: 'server',
  adapter: vercel(),
});

// Netlify
import netlify from '@astrojs/netlify';

export default defineConfig({
  output: 'server',
  adapter: netlify(),
});

// Node.js
import node from '@astrojs/node';

export default defineConfig({
  output: 'server',
  adapter: node({
    mode: 'standalone',
  }),
});
```

### Checklist avant déploiement

- [ ] `npm run build` sans erreurs
- [ ] Tests passent
- [ ] Images optimisées
- [ ] SEO vérifié (meta tags, sitemap)
- [ ] Performance testée (Lighthouse)
- [ ] Variables d'environnement configurées
- [ ] 404 page personnalisée
- [ ] Accessibility audit
- [ ] Content Collections validées
- [ ] API routes testées (si SSR)

## Exemples de patterns courants

### Navigation active

```astro
---
// src/components/astro/shared/Navigation.astro
const currentPath = Astro.url.pathname;

const links = [
  { href: '/', label: 'Accueil' },
  { href: '/blog', label: 'Blog' },
  { href: '/about', label: 'À propos' },
];
---

<nav>
  <ul>
    {links.map(link => (
      <li>
        <a
          href={link.href}
          class:list={['nav-link', { active: currentPath === link.href }]}
        >
          {link.label}
        </a>
      </li>
    ))}
  </ul>
</nav>

<style>
  .nav-link {
    color: #666;
    text-decoration: none;
  }

  .nav-link.active {
    color: #0ea5e9;
    font-weight: bold;
  }
</style>
```

### Pagination

```astro
---
// src/pages/blog/[page].astro
import { getCollection } from 'astro:content';

export async function getStaticPaths({ paginate }) {
  const posts = await getCollection('blog');
  const sortedPosts = posts.sort((a, b) =>
    b.data.pubDate.valueOf() - a.data.pubDate.valueOf()
  );

  return paginate(sortedPosts, { pageSize: 10 });
}

const { page } = Astro.props;
---

<Layout>
  <h1>Blog - Page {page.currentPage}</h1>

  <div class="posts">
    {page.data.map(post => (
      <BlogCard post={post} />
    ))}
  </div>

  <nav class="pagination">
    {page.url.prev && (
      <a href={page.url.prev}>← Précédent</a>
    )}

    <span>Page {page.currentPage} sur {page.lastPage}</span>

    {page.url.next && (
      <a href={page.url.next}>Suivant →</a>
    )}
  </nav>
</Layout>
```

### Dark mode

```astro
---
// src/components/astro/shared/ThemeToggle.astro
---

<button id="theme-toggle" aria-label="Toggle dark mode">
  <span class="sun">☀️</span>
  <span class="moon">🌙</span>
</button>

<script>
  // Initialiser le thème
  const getTheme = () => {
    if (typeof localStorage !== 'undefined') {
      return localStorage.getItem('theme') || 'light';
    }
    return 'light';
  };

  const setTheme = (theme: string) => {
    document.documentElement.dataset.theme = theme;
    localStorage.setItem('theme', theme);
  };

  // Appliquer au chargement
  setTheme(getTheme());

  // Toggle
  document.getElementById('theme-toggle')?.addEventListener('click', () => {
    const currentTheme = getTheme();
    const newTheme = currentTheme === 'light' ? 'dark' : 'light';
    setTheme(newTheme);
  });
</script>

<style>
  button {
    background: none;
    border: 2px solid currentColor;
    border-radius: 8px;
    padding: 0.5rem;
    cursor: pointer;
  }

  [data-theme='light'] .moon,
  [data-theme='dark'] .sun {
    display: none;
  }
</style>
```

```css
/* src/styles/global.css */
:root {
  --bg: white;
  --text: black;
}

[data-theme="dark"] {
  --bg: #1a1a1a;
  --text: white;
}

body {
  background: var(--bg);
  color: var(--text);
  transition: background 0.3s, color 0.3s;
}
```

### Recherche

```astro
---
// src/pages/search.astro
import { getCollection } from 'astro:content';

const query = Astro.url.searchParams.get('q')?.toLowerCase() || '';

const allPosts = await getCollection('blog');
const results = query ? allPosts.filter(post =>
  post.data.title.toLowerCase().includes(query) ||
  post.data.description.toLowerCase().includes(query)
) : [];
---

<Layout title="Recherche">
  <form method="get">
    <input
      type="search"
      name="q"
      value={query}
      placeholder="Rechercher..."
      autofocus
    />
    <button type="submit">Rechercher</button>
  </form>

  {query && (
    <div>
      <p>{results.length} résultat(s) pour "{query}"</p>

      {results.map(post => (
        <article>
          <h2><a href={`/blog/${post.slug}`}>{post.data.title}</a></h2>
          <p>{post.data.description}</p>
        </article>
      ))}
    </div>
  )}
</Layout>
```

## Ressources

- [Documentation Astro officielle](https://docs.astro.build/)
- [Astro Theme Directory](https://astro.build/themes/)
- [Astro Integrations](https://astro.build/integrations/)
- [Astro Discord](https://astro.build/chat)
- [Awesome Astro](https://github.com/one-aalam/awesome-astro)

## Commandes utiles

```bash
# Créer un nouveau projet
npm create astro@latest

# Développement
npm run dev

# Build
npm run build

# Preview
npm run preview

# Ajouter une intégration
npx astro add react
npx astro add tailwind
npx astro add mdx

# Check du projet
npx astro check

# Sync des types (content collections)
npx astro sync
```

---

**Note** : Ce document doit être maintenu à jour au fur et à mesure que le projet évolue et que de nouvelles fonctionnalités Astro sont adoptées.
