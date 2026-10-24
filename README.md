# Programmatic SEO City Pages

Generateur automatique de pages SEO locales (une page par ville) avec stockage Cloudflare KV et rendu SSR. Combine Supabase (base de donnees, cron) et Cloudflare Workers (KV, proxy) pour creer des milliers de pages uniques par ville.

## Architecture

```
pg_cron (nuit) → Edge Function Supabase → genere HTML par ville
                                        → ecrit dans Cloudflare KV
                                        → soumet IndexNow
Cloudflare Worker (domain-proxy) → sert le KV en SSR aux crawlers
```

## Ce que ca fait

- Lit une liste de villes depuis une table Supabase (code INSEE, population, departement, region)
- Pour chaque ville, genere une page HTML unique avec des donnees locales
- Ecrit la page dans Cloudflare KV au format `{title, meta, content, updated_at}`
- Soumet automatiquement l'URL a IndexNow
- Trace chaque page dans une table de suivi (`seo_programmatic_tracking`)
- Fonctionne en batch (10 villes/nuit par defaut, configurable)

## Schema de base

```sql
CREATE TABLE seo_cities (
  id SERIAL PRIMARY KEY,
  name TEXT NOT NULL,
  slug TEXT NOT NULL,
  department_code TEXT,
  department_name TEXT,
  region TEXT,
  population INTEGER,
  postal_codes TEXT[],
  insee_code TEXT UNIQUE,
  latitude NUMERIC,
  longitude NUMERIC,
  processed BOOLEAN DEFAULT false
);

CREATE TABLE seo_programmatic_tracking (
  id SERIAL PRIMARY KEY,
  domain TEXT NOT NULL,
  slug TEXT NOT NULL,
  theme TEXT NOT NULL,
  city TEXT NOT NULL,
  chars_count INTEGER,
  kv_written BOOLEAN DEFAULT false,
  indexnow_submitted BOOLEAN DEFAULT false,
  created_at TIMESTAMPTZ DEFAULT now(),
  UNIQUE(domain, slug)
);
```

## Configuration du cron

```sql
SELECT cron.schedule(
  'generate-programmatic-seo-nightly',
  '0 2 * * *',
  $$SELECT net.http_post(
    url := 'https://YOUR_PROJECT.supabase.co/functions/v1/generate-programmatic-seo',
    headers := '{"Content-Type": "application/json"}'::jsonb,
    body := '{}'::jsonb
  )$$
);
```

## Resultats en production

Ce systeme est utilise par [BusinessDigital.fr](https://businessdigital.fr) sur 7 combinaisons domaine/theme, avec 504 villes francaises (toutes les communes de plus de 10 000 habitants).

Objectif : 3 500 pages en 50 nuits, premiers resultats Google en 2-4 semaines.

En savoir plus sur notre approche SEO : [businessdigital.fr/academie](https://businessdigital.fr/academie).

## Prerequis

- Projet Supabase (plan Pro recommande pour pg_cron et pg_net)
- Compte Cloudflare avec Workers et KV
- Cle IndexNow

## Licence

MIT
