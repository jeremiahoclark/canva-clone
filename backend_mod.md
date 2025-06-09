Below is a “from-code-to-prod” roadmap that shows what must change, why it matters, and one pragmatic way to get it done when you move Davronov-Alimardon/canva-clone to Firebase.

1. Know What You’re Migrating
Layer	Current tech (repo)	Firebase-friendly substitute	Why it must change
Runtime & build	Next 14.2.4 (package.json) 
github.com
Next 15.4 (latest LTS)	you asked for a version bump; 15 is now stable and adds Turbopack, async/streaming Router APIs 
nextjs.org
Auth	next-auth + Drizzle adapter	Firebase Auth (client SDK) + Firebase Admin (Functions)	removes need for Drizzle tables that just back sessions/accounts
ORM / DB	Drizzle ORM + Neon Postgres (drizzle.config.ts, schema.ts) 
github.com
github.com
Option A (document) Firestore, good fit for users/{uid}/projects JSON
Option B (relational) keep Neon but switch to Prisma	you said “no Drizzle” and are open to Neon or Cloud SQL
File storage	UploadThing (stores on Vercel’s S3-compatible bucket)	Firebase Storage (GCS)	native to Firebase; fine-grained security rules
Serverless APIs	Hono handlers in /src/app/api	Cloud Functions for Firebase (Node 20) or App Hosting’s Cloud Run “backend”	Hosting rewrites cleanly to Functions/Run
Hosting	Vercel	Firebase App Hosting (GA) – first-class for Next 15 
firebase.blog

2. Upgrade & Refactor Locally
Branch → firebase-migration

Upgrade Next.js & React

bash
Copy
Edit
npx @next/codemod upgrade latest   # runs official codemod
npm i next@15.4.0 react@18.3.0 react-dom@18.3.0
Fix any codemod TODOs (mainly revalidate, async handlers).

Replace Drizzle

If you choose Firestore

npm rm drizzle-orm drizzle-kit @neondatabase/serverless pg

npm i firebase firebase-admin

Rewrite the tiny db helper functions (create/read/update project JSON, subscription status) to Firestore queries—no joins needed because all relations in schema.ts map 1-to-many.

If you stay on Postgres

npm i prisma @prisma/client then npx prisma init, transliterate schema.ts to schema.prisma, point DATABASE_URL at Neon.

Rip out Drizzle adapters in Auth

Switch to Firebase Auth client SDK in the front-end (apps/providers.tsx).

In Functions, verify ID-tokens with admin.auth().verifyIdToken().

Swap UploadThing for Firebase Storage

Client side:

ts
Copy
Edit
import { getStorage, ref, uploadBytes, getDownloadURL } from "firebase/storage";
Server side (if you still need a signed URL generator):

ts
Copy
Edit
import { getStorage } from "firebase-admin/storage";
const url = await getStorage().bucket().file(path).getSignedUrl({ action:"read", expires:"1y"});
``` :contentReference[oaicite:4]{index=4}  
Package tidy-up (npm prune && npm dedupe) and run next build.

3. Bootstrap Firebase Project
bash
Copy
Edit
firebase login
firebase init apphosting  # GA product; zero-config for Next 15
Choose TypeScript, App Hosting, and Functions.

Accept automatic detection of Next.js. The CLI scaffolds:

bash
Copy
Edit
/firebase.json
/app.firebaserc
/functions/  (Node 20 runtime)
/app/        (your Next.js source untouched)
4. Wire Up Back-end Pieces
Concern	What to do in /functions
Stripe webhooks	Move /api/webhook/route.ts into functions/stripeWebhook.ts, export as onRequest. Add a rewrites rule in firebase.json so /api/webhook hits the Function.
Image AI / Replicate calls	Keep them in Functions to avoid leaking API keys to the browser.
SSR / API routes	App Hosting auto-builds a Cloud Run service that serves app and proxies any app/api/* routes. No extra config needed.

Example firebase.json snippet:

json
Copy
Edit
{
  "apphosting": {
    "target": "canva-clone",
    "source": "."
  },
  "functions": {
    "source": "functions",
    "runtime": "nodejs20",
    "ignore": ["**/node_modules/**"]
  },
  "rewrites": [
    { "source": "/api/webhook", "function": "stripeWebhook" }
  ]
}
5. Continuous Deployment
Step	Command / setting
Manual deploy	firebase deploy --only apphosting,functions
GitHub Actions	Use the official Firebase App Hosting workflow; it builds Next.js with Turbopack cache and deploys on every push to main.
Env-vars	firebase functions:config:set stripe.secret=... (available as process.env.STRIPE_SECRET in Functions). For App Hosting, set variables in Firebase → App Hosting → Backends → Environment variables.

6. Post-migration Checklist
Billing plan – move project to Blaze (needed for Functions, Storage, and Firestore quotas).

Security Rules – tighten Storage (allow read,write: if request.auth != null && request.auth.uid == resource.name.split('/')[1]).

Performance – enable Cloud Run min-instances to 0 for dev, add a CDN TTL of 1 day for /static/*.

Monitoring – open Cloud Logging and set up an error-alert on Function exceptions.

Roll-back strategy – App Hosting keeps previous revisions; a single click rolls back if something goes wrong.

7. Time-boxed Execution Plan
Day	Milestone
1	Fork repo, create Firebase project, run Next 15 codemod, green build locally.
2–3	Remove Drizzle, stand up Firestore or Prisma-Neon layer, write adapters.
4	Replace file uploads, verify Storage rules, regenerate thumbnails.
5	Move Stripe webhook & Replicate calls to Functions, run local firebase emulators:start.
6	First deploy to Staging Firebase project; run e2e tests.
7	Production switch-over, DNS + SSL, set up monitoring dashboards.

Recommendations & Tips
Start with Firestore unless you genuinely need SQL joins. Your current schema.ts is already mostly hierarchical (user → projects). It will keep cold-start latency lower on Functions.

Use App Hosting over “web frameworks preview” because it’s GA, supports Next 15 out-of-the-box, and removes most bespoke Cloud Functions wiring.

Keep Neon as a thin read-only replica if you need complex analytics later; you can sync writes from Firestore to Neon via Cloud Functions triggers without polluting app code.

For local dev parity, install the Firebase Emulator Suite so you can run Firestore, Auth and Storage offline (firebase emulators:start --import=.emulatorData).

That’s it—follow the steps in order and you’ll have Canva-Clone running on Firebase with modern Next 15, zero Drizzle, and images safely living in Google Cloud Storage while still keeping options open for Neon or Cloud SQL down the road.











Sources
