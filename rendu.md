### TP — Maîtriser la gestion des identités et des rôles sur Google Cloud (Partie 1)
Rapport d’exécution et résultats validés en ligne de commande (PowerShell + gcloud)

## Préambule
- Compte principal: `julen.dubroca@gmail.com`
- Projet: `ynov-iam-tp-1545971109`
- Région: `europe-west1`
- Collaborateur: `aroufgangstacaptai@gmail.com`

---

## Exercice 1 — Créer les identités de base

- Création du projet et sélection:
```powershell
gcloud projects create ynov-iam-tp-1545971109 --name "YNOV IAM TP"
gcloud config set project ynov-iam-tp-1545971109
```

- Ajout d’utilisateurs IAM (tentatives avec emails fictifs ont échoué, ensuite remplacés par emails réels):
```powershell
gcloud projects add-iam-policy-binding $PROJECT_ID --member="user:julesdaramendi@gmail.com" --role="roles/viewer"
gcloud projects add-iam-policy-binding $PROJECT_ID --member="user:aroufgangstacaptai@gmail.com" --role="roles/editor"
```

- Compte de service “app-backend”:
```powershell
gcloud iam service-accounts create app-backend --display-name "app-backend"
# Email: app-backend@ynov-iam-tp-1545971109.iam.gserviceaccount.com
```

- Vérification SA:
```powershell
gcloud iam service-accounts list --project $PROJECT_ID --format="table(displayName, email)"
# OK: app-backend@...
```

---

## Exercice 2 — Explorer IAM et les rôles

- Policy IAM du projet:
```powershell
gcloud projects get-iam-policy $PROJECT_ID --format="table(bindings.role, bindings.members)"
```

- Inspection de rôles prédéfinis:
```powershell
gcloud iam roles describe roles/storage.objectViewer --format="value(includedPermissions)"
# => storage.objects.get; storage.objects.list; ...
```

- Permissions testables d’une ressource (per-projet):
```powershell
$PROJECT_NUMBER = (gcloud projects describe $PROJECT_ID --format="value(projectNumber)")
gcloud iam list-testable-permissions "//cloudresourcemanager.googleapis.com/projects/$PROJECT_NUMBER" --format="table(name)"
```

- Policy Analyzer (flags compatibles):
```powershell
gcloud asset analyze-iam-policy --project "$PROJECT_ID" --identity="user:$EDITOR_USER" --format="table(fullyQualifiedName,role)"
```

- Conclusion:
  - Différence rôles de base vs prédéfinis comprise, et usage de `analyze-iam-policy` validé.
  - Note: certains flags (`--scope`, `--permission`) varient selon version; adaptation faite.

---

## Exercice 3 — Portée des rôles et permissions

- Création bucket de test et fichier:
```powershell
# Après liaison du billing
$BUCKET_NAME="$PROJECT_ID-bucket-973805937" (puis remplacement par un bucket valide)
Set-Content .\test.txt 'hello' -Encoding ascii
gcloud storage buckets create "gs://$BUCKET_NAME" --location=EU
gcloud storage cp .\test.txt "gs://$BUCKET_NAME/test.txt"
```

- Lister permissions testables (bucket):
```powershell
$BUCKET_RESOURCE = "//storage.googleapis.com/projects/_/buckets/$BUCKET_NAME"
gcloud iam list-testable-permissions $BUCKET_RESOURCE --format="value(name)"
# => inclut storage.objects.get, storage.objects.list
```

- Accorder rôle sur le bucket (portée ressource):
```powershell
gcloud storage buckets add-iam-policy-binding "gs://$BUCKET_NAME" --member="user:$EDITOR_USER" --role="roles/storage.objectViewer"
gcloud storage buckets get-iam-policy "gs://$BUCKET_NAME" --format="table(bindings.role, bindings.members)"
```

- Test collaborateur (portée ressource):
```powershell
gcloud auth login $EDITOR_USER --no-launch-browser
gcloud config set account $EDITOR_USER
gcloud storage ls "gs://$BUCKET_NAME"             # OK
gcloud storage cp "gs://$BUCKET_NAME/test.txt" -  # OK
gcloud storage ls "gs://$PROJECT_ID-bucket-xxxx"  # 403 / Not found (attendu)
```

- Extension au niveau projet (portée projet):
```powershell
gcloud config set account "julen.dubroca@gmail.com"
gcloud projects add-iam-policy-binding $PROJECT_ID --member="user:$EDITOR_USER" --role="roles/storage.objectViewer"
# Note: 'buckets list' reste 403 (car storage.buckets.list ∉ storage.objectViewer); ajouter 'roles/storage.viewer' si on veut lister les buckets.
```

- Nettoyage:
```powershell
gcloud projects remove-iam-policy-binding $PROJECT_ID --member="user:$EDITOR_USER" --role="roles/storage.objectViewer"
gcloud storage buckets remove-iam-policy-binding "gs://$BUCKET_NAME" --member="user:$EDITOR_USER" --role="roles/storage.objectViewer"
```

- Conclusion test:
  - Portée ressource: accès uniquement au bucket ciblé.
  - Portée projet: lecture d’objets de tous les buckets connus, mais pas la liste des buckets sans `roles/storage.viewer`.
  - Illustration du moindre privilège validée.

---

## Exercice 4 — Rôle personnalisé Cloud Run

- YAML (créé dans %TEMP% pour éviter OneDrive):
```yaml
title: Cloud Run Minimal Deployer
description: Minimal permissions to deploy, list and delete Cloud Run services (project scope)
stage: GA
includedPermissions:
  - run.locations.list
  - run.operations.get
  - run.services.create
  - run.services.update
  - run.services.get
  - run.services.list
  - run.services.delete
  - iam.serviceAccounts.actAs
```

- Création du rôle:
```powershell
$ROLE_ID="runMinimalDeployer"
gcloud services enable run.googleapis.com
gcloud iam roles create $ROLE_ID --project=$PROJECT_ID --file="$env:TEMP\run.custom.deployer.yaml"
gcloud iam roles describe $ROLE_ID --project=$PROJECT_ID --format="yaml"
```

- Attribution au collaborateur:
```powershell
gcloud projects add-iam-policy-binding $PROJECT_ID --member="user:$EDITOR_USER" --role="projects/$PROJECT_ID/roles/$ROLE_ID"
```

- Tests collaborateur:
```powershell
$REGION="europe-west1"; $SERVICE="hello-custom-role"
gcloud auth login $EDITOR_USER --no-launch-browser
gcloud run deploy $SERVICE --image="us-docker.pkg.dev/cloudrun/container/hello" --region=$REGION --platform=managed --no-allow-unauthenticated
gcloud run services list --region=$REGION     # OK
gcloud run services delete $SERVICE --region=$REGION --quiet  # OK
```

- Nettoyage:
```powershell
gcloud config set account "julen.dubroca@gmail.com"
gcloud projects remove-iam-policy-binding $PROJECT_ID --member="user:$EDITOR_USER" --role="projects/$PROJECT_ID/roles/$ROLE_ID"
gcloud iam roles delete $ROLE_ID --project=$PROJECT_ID --quiet
```

- Résultat: déploiement/liste/suppression OK avec le jeu minimal de permissions listé ci-dessus.

---

## Exercice 5 — Comptes de service et droits applicatifs (Cloud Run → GCS)

- Compte de service applicatif:
```powershell
gcloud iam service-accounts create run-backend --display-name "run-backend"
$RUN_SA="run-backend@$PROJECT_ID.iam.gserviceaccount.com"
```

- Rôle minimal de lecture objets sur UN bucket (moindre privilège):
```powershell
# Bucket créé: ynov-iam-tp-1545971109-bucket-1162333581 (EU)
gcloud storage buckets add-iam-policy-binding "gs://ynov-iam-tp-1545971109-bucket-1162333581" `
  --member="serviceAccount:$RUN_SA" --role="roles/storage.objectViewer"
```

- Application Node.js (Express + @google-cloud/storage), route `/list`, var d’env `BUCKET_NAME`:
```json
// package.json
{ "name":"run-list-bucket","version":"1.0.0","type":"module","scripts":{"start":"node server.js"},
  "dependencies":{"@google-cloud/storage":"^7.12.1","express":"^4.19.2"} }
```
```javascript
// server.js
import express from "express";
import { Storage } from "@google-cloud/storage";
const app = express(); const port = process.env.PORT || 8080;
const bucketName = process.env.BUCKET_NAME; const storage = new Storage();
app.get("/list", async (_req, res) => {
  if (!bucketName) return res.status(500).json({ error: "BUCKET_NAME not set" });
  try { const [files] = await storage.bucket(bucketName).getFiles();
        res.json(files.map(f => f.name)); }
  catch (e) { res.status(500).json({ error: String(e) }); }
});
app.get("/", (_req,res)=>res.send("OK"));
app.listen(port, ()=>console.log(`Listening on ${port}`));
```

- Dockerfile (npm install; port 8080):
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install --omit=dev
COPY server.js .
ENV PORT=8080
EXPOSE 8080
CMD ["npm", "start"]
```

- Build & push (Artifact Registry) avec Cloud Build (droits ajoutés au SA Cloud Build):
```powershell
# Donner roles/artifactregistry.writer au SA Cloud Build:
$PROJECT_NUMBER=(gcloud projects describe $PROJECT_ID --format="value(projectNumber)")
gcloud projects add-iam-policy-binding $PROJECT_ID --member="serviceAccount:$PROJECT_NUMBER@cloudbuild.gserviceaccount.com" --role="roles/artifactregistry.writer"

# Build & push
$IMAGE="$REGION-docker.pkg.dev/$PROJECT_ID/app-repo/run-list-bucket:latest"
gcloud services enable artifactregistry.googleapis.com cloudbuild.googleapis.com
gcloud artifacts repositories create app-repo --repository-format=docker --location=$REGION 2>$null
gcloud builds submit --tag "$IMAGE" --quiet
```

- Déploiement Cloud Run avec le compte de service et la variable d’env:
```powershell
gcloud run deploy run-list-bucket `
  --image="$IMAGE" --region="$REGION" --platform=managed `
  --service-account="$RUN_SA" `
  --set-env-vars="BUCKET_NAME=ynov-iam-tp-1545971109-bucket-1162333581" `
  --allow-unauthenticated
# URL obtenue (ex): https://run-list-bucket-969776836293.europe-west1.run.app
```

- Test fonctionnel:
```powershell
# Bucket initialement vide => []
curl "$URL/list"         # => []

# Ajout d’un objet et re-test
[IO.File]::WriteAllText("$env:TEMP\sample.txt",'hello from cloud run')
gcloud storage cp "$env:TEMP\sample.txt" "gs://ynov-iam-tp-1545971109-bucket-1162333581/sample.txt"
curl "$URL/list"         # => ["sample.txt"]
```

- Logs (Cloud Run + accès Storage):
```powershell
gcloud logging read 'resource.type="cloud_run_revision" AND resource.labels.service_name="run-list-bucket"' `
  --limit=20 --format="table(timestamp,severity,textPayload)"
gcloud logging read 'protoPayload.serviceName="storage.googleapis.com" AND protoPayload.authenticationInfo.principalEmail:*' `
  --limit=20 --format="table(timestamp, protoPayload.authenticationInfo.principalEmail, protoPayload.methodName)"
# Observation: accès Storage visibles; côté Admin Activity, opérations Storage par l’owner (création/config). 
# Les lectures d’objets via l’API client utilisent l’identifiant du service selon le type de log consulté.
```

- Comportement moindre privilège:
  - Avec `roles/storage.objectViewer` sur UN bucket: la route `/list` ne renvoie que les objets de ce bucket; accès à un autre bucket => échec (403).
  - Le service s’authentifie automatiquement via le compte de service attaché (`--service-account=run-backend@...`).

- Nettoyage:
```powershell
gcloud run services delete run-list-bucket --region $REGION --quiet
gcloud storage buckets remove-iam-policy-binding "gs://ynov-iam-tp-1545971109-bucket-1162333581" `
  --member="serviceAccount:$RUN_SA" --role="roles/storage.objectViewer"
# (optionnel) gcloud iam service-accounts delete "$RUN_SA" --quiet
```

---

## Points clés et validations

- Création et configuration IAM au niveau projet et ressource validées.
- Différence de portée démontrée:
  - Bucket: accès limité au bucket ciblé (moindre privilège).
  - Projet: lecture d’objets tous buckets connus via `roles/storage.objectViewer`, mais pas “buckets list” (nécessite `roles/storage.viewer`).
- Rôle personnalisé Cloud Run minimal (`run.services.*`, `run.operations.get`, `run.locations.list`, `iam.serviceAccounts.actAs`) testé: déploiement/liste/suppression OK.
- Compte de service applicatif (`run-backend`) + droit minimal sur UN bucket; app Cloud Run lit correctement le contenu (`/list` → `["sample.txt"]`).
- Traçabilité: logs Cloud Run/Storage consultés, identité visible pour opérations d’admin; accès data visibles selon vues/logs activés.
