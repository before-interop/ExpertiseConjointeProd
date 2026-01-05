
# Expertise Conjointe en Production — Guide d’utilisation de l’API (Swagger)

**Version du modèle** : 1.0.0  
**Objet** : gestion des tickets *Expertise Conjointe en Production* (ECP), des notes, des pièces jointes et des événements.

> 👉 Cette API s’appuie sur des schémas communs (TroubleTicket, Note, Attachment, CloudEvent) et expose des ressources REST paginées.  
> 👉 Les erreurs **500** ne sont **pas** utilisées par convention dans cette spécification.

---

## 1) Bases de l’API

### URL de base
Le serveur est paramétrable via des variables :
```
{protocol}://{host}:{port}{basePath}
```
- `protocol` : `https` (défaut) ou `http`
- `host` : `localhost` (défaut)
- `port` : `8080` (défaut)
- `basePath` : `/api` (défaut)

### Formats
- **Requête/Réponse** : `application/json`
- **Téléchargement binaire** : `application/octet-stream`

### Codes de statut utilisés
- `200` OK / `201` Created / `204` No Content
- `400` Bad Request / `403` Forbidden / `404` Not Found / `412` Precondition Failed
- `default` (erreur applicative standardisée)

> **ETag & If-Match** : les mises à jour (`PUT`/`PATCH`) utilisent l’en-tête **`If-Match`** avec la valeur **`ETag`** lue au `GET`/`POST`.

---

## 2) Ressources principales

### 2.1 Tickets ECP
- `GET /expertise-conjointe-productions` : liste paginée des tickets  
- `HEAD /expertise-conjointe-productions` : comptage  
- `POST /expertise-conjointe-productions` : création d’un ticket  
- `GET /expertise-conjointe-productions/{ticketId}` : détail d’un ticket  
- `PUT /expertise-conjointe-productions/{ticketId}` : mise à jour complète (ETag/If-Match)  
- `PATCH /expertise-conjointe-productions/{ticketId}` : mise à jour partielle (ETag/If-Match)

**Modèle clé** : `TicketContradictoryProductionExpertise`
- Champs **obligatoires** : `@type`, `code_oc`, `code_oi`, `initialRequest`, `expertise`  
- `@type` : `expertiseConjointeProduction` / `AUTRE`  
- `code_oc`, `code_oi` : quadrigrammes opérateurs (codes ARCEP)  
- `initialRequest` : `OC` | `OI`  
- `expertise.accessOrder` :
  - `internalOCOrderReference` : **référence commande accès interne OC**  
  - `socketServiceReference` : **référence prestation prise**  
- `expertise.location` : `LocationPM`  
  - `pm` : schéma **PM** (Interop)  

**Autres modèles liés** (non exhaustif) :
- `Appointment` : `appointmentId` (idRDV), contacts OC/OI (principal/secondaire)
- `ExpertiseReport` : `presenceStatus`, `pboReading`, `pmReading`, `report`, `responsibility`
- `PresenceStatus` : `ocTechnicianPresent`, `oiTechnicianPresent`, `ocTechnicianAbsenceAcknowledgment`
- `PBOReading` & `PMReading` : références PBO/PM, mesures & valeurs
- `Report` : compte-rendu (RO conforme, éléments PM/PBO, connecteur prise, signal OK, ROA à modifier…)
- `statusChangeReason` : liste **française** (motifs selon transitions FINISHED/REJECTED/CANCELLED)

### 2.2 Notes
- `GET /expertise-conjointe-productions/{ticketId}/note` : liste paginée des notes  
- `HEAD /expertise-conjointe-productions/{ticketId}/note` : comptage  
- `POST /expertise-conjointe-productions/{ticketId}/note` : création d’une note

### 2.3 Pièces jointes
- `GET /expertise-conjointe-productions/{ticketId}/attachment` : liste paginée des pièces  
- `HEAD /expertise-conjointe-productions/{ticketId}/attachment` : comptage  
- `POST /expertise-conjointe-productions/{ticketId}/attachment` : **création des métadonnées** (multi-part)
- `GET /expertise-conjointe-productions/{ticketId}/attachment/{attachmentId}/content` : **téléchargement**  
- `POST /expertise-conjointe-productions/{ticketId}/attachment/{attachmentId}/content/{chunkIndex}` : **upload d’un tronçon**  
- `POST /expertise-conjointe-productions/{ticketId}/attachment/{attachmentId}/finalize` : **finalisation**

**Pattern multi-part (upload par tronçons)**
1. `POST /attachment` (métadonnées) → retourne `id`, `ETag`, `X-Chunk-Size`
2. boucle : `POST /attachment/{id}/content/{chunkIndex}` (tronçon binaire + métadonnées `Chunk`)
3. `POST /attachment/{id}/finalize`

### 2.4 Événements
- `GET /expertise-conjointe-productions/event` : liste paginée des événements  
- `HEAD /expertise-conjointe-productions/event` : comptage  
- `POST /expertise-conjointe-productions/event` : diffusion d’un événement

Modèle : `Event` (hérite de CloudEvent) — champs `time`, `subject` (UUID), `type`, `datacontenttype`.

---

## 3) Recherche, pagination, tri et filtrage
Toutes les listes prennent en charge :
- `limit` (défaut 50, max 100), `offset` (défaut 0)
- `sort` (ex. `-creationDate`)
- `fields` (projection partielle des champs)
- `filters` (filtres dynamiques sur propriétés)

> Le **HEAD** sur les collections retourne `X-Total-Count` ; les **GET** retournent `X-Total-Count` et `X-Result-Count`.

---

## 4) Exemples

### 4.1 Créer un ticket
```bash
curl -X POST   "{protocol}://{host}:{port}{basePath}/expertise-conjointe-productions"   -H "Content-Type: application/json"   -d '{
    "@type": "expertiseConjointeProduction",
    "code_oc": "ABCD",
    "code_oi": "WXYZ",
    "initialRequest": "OC",
    "expertise": {
      "accessOrder": {
        "internalOCOrderReference": "OC-INT-123456",
        "socketServiceReference": "PRESA-654321"
      },
      "location": {
        "type": "LocationPM",
        "pm": { /* … schéma Interop PM … */ },
        "geographicLocation": { /* … Interop.GeographicPoint … */ },
        "geographicAddress": { /* … Interop.GeographicAddress … */ }
      }
    }
  }'
```

### 4.2 Mettre à jour un ticket (If-Match)
```bash
curl -X PUT   "{protocol}://{host}:{port}{basePath}/expertise-conjointe-productions/{ticketId}"   -H "Content-Type: application/json"   -H "If-Match: "<ETag-lu-au-GET>""   -d '{
    "@type": "expertiseConjointeProduction",
    "code_oc": "ABCD",
    "code_oi": "WXYZ",
    "initialRequest": "OI",
    "expertise": { /* … */ }
  }'
```

### 4.3 Ajouter une note
```bash
curl -X POST   "{protocol}://{host}:{port}{basePath}/expertise-conjointe-productions/{ticketId}/note"   -H "Content-Type: application/json"   -d '{
    "text": "Techniciens présents, test de signal effectué.",
    "author": "oc.user@domain.tld"
  }'
```

### 4.4 Créer une pièce et envoyer le contenu par tronçons
1. **Métadonnées**
```bash
curl -X POST   "{protocol}://{host}:{port}{basePath}/expertise-conjointe-productions/{ticketId}/attachment"   -H "Content-Type: application/json"   -d '{
    "@type": "documentMultiPart",
    "name": "pv-ecp.pdf",
    "description": "PV signé",
    "size": 1048576,
    "sha256": "<hash>"
  }'
```
2. **Upload d’un tronçon** (répéter pour chaque index)
```bash
curl -X POST   "{protocol}://{host}:{port}{basePath}/expertise-conjointe-productions/{ticketId}/attachment/{attachmentId}/content/{chunkIndex}"   -F "file=@chunk.bin"   -F "metadata={"index":{chunkIndex},"size":N,...};type=application/json"
```
3. **Finaliser**
```bash
curl -X POST   "{protocol}://{host}:{port}{basePath}/expertise-conjointe-productions/{ticketId}/attachment/{attachmentId}/finalize"
```

### 4.5 Diffuser un événement
```bash
curl -X POST   "{protocol}://{host}:{port}{basePath}/expertise-conjointe-productions/event"   -H "Content-Type: application/json"   -d '{
    "id": "c7f2ec8a-2c19-4c7a-b8a6-8b6f2b2f1b7a",
    "source": "ecp/oc",
    "type": "fr.interop.ecp.update",
    "subject": "c5b0e2e8-6e2d-4b7a-8dc9-ffb3b6b8c001",
    "time": "2025-01-12T10:15:30Z",
    "datacontenttype": "application/json",
    "data": { /* … Ticket/Note/Attachment … */ }
  }'
```

> Les valeurs d’exemple sont indicatives ; adapter selon vos contraintes de sécurité et de routage.

---

## 5) Conventions & bonnes pratiques
- **Nommage champs** : `snake_case` (ex. `code_oc`, `code_oi`).
- **Contrôle de concurrence** : toujours utiliser `If-Match` lors des modifications.
- **Pagination** : privilégier `HEAD` pour obtenir le volume total avant de paginer.
- **Pièces jointes volumineuses** : suivre le **pattern multi-part** (tronçonnage + finalisation).
- **Événements** : respecter le format CloudEvents (`time`, `type`, `subject`, `datacontenttype`).

---

## 6) Références de la spécification
- Spécification **YAML** : `expertise-conjointe-production.yaml`
- Spécification **JSON** : `expertise-conjointe-production.json`

> Pour plus de détails de structure (ex. schémas PM, GeographicPoint/Address), se référer aux `$ref` Interop inclus dans la spécification.

---

## 7) Sécurité
Les mécanismes d’authentification/autorisation (ex. OAuth2/Bearer) ne sont **pas définis** dans ce document et doivent être **ajoutés** selon vos besoins d’intégration (en-têtes `Authorization`, flux d’échange de tokens, scopes, etc.).

---

## 8) Gestion de version & compatibilité
- La version courante est **1.0.0**.
- Les évolutions mineures peuvent ajouter des champs **optionnels**.  
- Toute suppression ou modification de contrat est traitée comme **rupture** (nouvelle version ou champ alternatif).

