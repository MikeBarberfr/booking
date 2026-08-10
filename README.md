# Réservation en ligne — Barbershop (Supabase + GitHub Pages)

Même stack que NexTTrack : frontend statique sur GitHub Pages, backend sur Supabase
(base de données + Auth + Edge Functions), paiement Stripe, emails via Resend.

Différence clé par rapport à une solution "artifact" : le paiement de l'acompte est
**vérifié automatiquement** par un webhook Stripe qui confirme la réservation tout seul —
plus besoin de valider chaque paiement à la main.

## 1. Créer le projet Supabase

1. Sur [supabase.com](https://supabase.com), crée un **nouveau projet** dédié à cette activité
   (recommandé, pour ne pas mélanger avec les données NexTTrack).
2. Va dans **SQL Editor** → colle le contenu de `supabase/migrations/0001_init.sql` → Run.
3. Va dans **Authentication → Users → Add user** → crée ton compte admin (email + mot de passe).
   C'est ce compte qui te servira à te connecter sur `admin.html`.
4. Dans **Project Settings → API**, récupère :
   - `Project URL` → à mettre dans `config.js` (`SUPABASE_URL`)
   - `anon public key` → à mettre dans `config.js` (`SUPABASE_ANON_KEY`)

## 2. Créer le compte Stripe (ou réutiliser le tien)

1. Dans le Dashboard Stripe, reste en mode **Test** pour commencer.
2. Récupère ta **clé secrète** (Developers → API keys → Secret key).
3. On configurera le webhook après avoir déployé les fonctions (étape 4).

## 3. Créer le compte Resend (ou réutiliser celui de NexTTrack)

1. Sur [resend.com](https://resend.com), vérifie un domaine d'envoi (ou utilise le domaine
   de test fourni par Resend pour commencer).
2. Récupère ta clé API (`RESEND_API_KEY`).

## 4. Déployer les Edge Functions

Depuis le dossier du projet, avec la [CLI Supabase](https://supabase.com/docs/guides/cli) :

```bash
supabase login
supabase link --project-ref TON_PROJECT_REF

supabase functions deploy create-checkout-session
supabase functions deploy stripe-webhook --no-verify-jwt
supabase functions deploy notify-booking --no-verify-jwt

# Secrets partagés par toutes les fonctions
supabase secrets set STRIPE_SECRET_KEY=sk_test_xxx
supabase secrets set SUPABASE_URL=https://xxxxxxxx.supabase.co
supabase secrets set SUPABASE_SERVICE_ROLE_KEY=eyJxxxxx   # Project Settings > API > service_role
supabase secrets set SITE_URL=https://tonpseudo.github.io/ton-repo
supabase secrets set RESEND_API_KEY=re_xxxxx
supabase secrets set NOTIFY_FROM_EMAIL="Ton Barbershop <reservation@tondomaine.fr>"
# STRIPE_WEBHOOK_SECRET : voir étape 5
```

> `stripe-webhook` et `notify-booking` sont appelées par Stripe et par Supabase
> eux-mêmes (pas par un utilisateur connecté) : `--no-verify-jwt` est nécessaire.

## 5. Configurer le webhook Stripe

1. Dashboard Stripe → **Developers → Webhooks → Add endpoint**.
2. URL : `https://xxxxxxxx.supabase.co/functions/v1/stripe-webhook`
3. Événement à écouter : `checkout.session.completed`
4. Récupère le **Signing secret** (`whsec_...`) et ajoute-le :
   ```bash
   supabase secrets set STRIPE_WEBHOOK_SECRET=whsec_xxxxx
   ```

## 6. Configurer le Database Webhook (notification email automatique)

1. Dans Supabase → **Database → Webhooks → Create a new webhook**.
2. Table : `bookings` — Événement : `INSERT`
3. Type : `HTTP Request` → URL : `https://xxxxxxxx.supabase.co/functions/v1/notify-booking`
4. Ajoute le header `Authorization: Bearer TA_SERVICE_ROLE_KEY`.

Résultat : à chaque nouvelle réservation créée par un client, l'email part automatiquement,
sans rien faire de plus.

## 7. Configurer et déployer le site

1. Édite `config.js` avec ton `SUPABASE_URL`, `SUPABASE_ANON_KEY` et `FUNCTIONS_URL`
   (`https://xxxxxxxx.supabase.co/functions/v1`).
2. Pousse le dossier sur un dépôt GitHub.
3. Dans les Settings du repo → **Pages** → active GitHub Pages sur la branche `main`.
4. Ton site de réservation est disponible sur `index.html`, ton espace pro sur `admin.html`
   (ne le mets pas en avant dans la navigation publique — l'accès est protégé par mot de
   passe Supabase Auth, mais autant ne pas le lier depuis la page client).

## 8. Passer Stripe en mode production

Une fois les tests validés : remplace les clés `sk_test_...` par les clés live dans les
secrets Supabase, et recrée un webhook Stripe en mode live (les webhooks test/live sont
séparés).

## Structure du projet

```
├── index.html                  → page de réservation publique
├── admin.html                  → espace pro (agenda, prestations, horaires, réglages)
├── config.js                   → clés Supabase (à remplir)
├── supabase/
│   ├── migrations/0001_init.sql       → tables + RLS + fonction de dispo
│   └── functions/
│       ├── create-checkout-session/   → crée la session de paiement Stripe
│       ├── stripe-webhook/            → confirme la résa quand le paiement passe
│       └── notify-booking/            → envoie les emails (Resend)
```
