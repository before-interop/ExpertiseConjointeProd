# Expertise Conjointe en Production — Guide d'utilisation de l'API (Swagger)

**Version du modèle** : 1.0.0  
**Objet** : gestion des tickets *Expertise Conjointe en Production* (ECP), des notes, des pièces jointes et des événements.

> 👉 Cette API s'appuie sur des schémas communs (TroubleTicket, Note, Attachment, CloudEvent) et expose des ressources REST paginées.  
> 👉 Les erreurs **500** ne sont **pas** utilisées par convention dans cette spécification.

---

## 1) Bases de l'API

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
- **Upload multipart** : `multipart/form-data`

### Codes de statut utilisés
- `200` OK / `201` Created / `204` No Content
- `400` Bad Request / `403` Forbidden / `404` Not Found / `412` Precondition Failed / `413` Payload Too Large
- `default` (erreur applicative standardisée)

> **ETag & If-Match** : les mises à jour (`PUT`/`PATCH`) utilisent l'en-tête **`If-Match`** avec la valeur **`ETag`** lue au `GET`/`POST`.

---

## 2) Ressources principales

### 2.1 Tickets ECP
- `GET /expertise-conjointe-productions` : liste paginée des tickets  
- `HEAD /expertise-conjointe-productions` : comptage  
- `POST /expertise-conjointe-productions` : création d'un ticket  
- `GET /expertise-conjointe-productions/{ticketId}` : détail d'un ticket  
- `PUT /expertise-conjointe-productions/{ticketId}` : mise à jour complète (ETag/If-Match)  
- `PATCH /expertise-conjointe-productions/{ticketId}` : mise à jour partielle (ETag/If-Match)

**Modèle clé** : `TicketContradictoryProductionExpertise`
- Champs **obligatoires** : `@type`, `code_oc`, `code_oi`, `initialRequest`, `expertise`  
- `@type` : `expertiseConjointeProduction`  
- `code_oc`, `code_oi` : quadrigrammes opérateurs (codes ARCEP)  
- `initialRequest` : `OC` | `OI`
- `lastModificationBy` : `OI` | `OC` — identifie l'auteur du dernier changement
- `status` : état du ticket
  - Valeurs : `ACKNOWLEDGED`, `CANCELED`, `REJECTED`, `IN_PROGRESS`, `PLANNED`, `CONFIRMED`, `CLOSED`, `RESOLVED`
- `acknowledgedDate` : date de passage au status ACKNOWLEDGED (format `date`)
- `statusChangeDetail` : détail libre du changement d'état
- `statusChangeReason` : motif de changement d'état (voir section 2.1.1)
- `statusChangeReasonCode` : code du motif (voir section 2.1.2)

#### Expertise
- `expertise.accessOrder` :
  - `internalOCOrderReference` : **référence commande accès interne OC** (obligatoire)
  - `socketServiceReference` : **référence prestation prise** (obligatoire)
- `expertise.location` : `LocationPM` (obligatoire)
  - `pm` : schéma **PM** (Interop)

#### Appointment (rendez-vous)
- `appointmentId` : identifiant RDV (idRDV généré par l'API Plan de charge) — **obligatoire**
- `primaryOCContact` : contact principal OC (technicien ou hotline) — **obligatoire**
- `primaryOIContact` : contact principal OI
- `secondaryOCContact` : contact secondaire OC
- `secondaryOIContact` : contact secondaire OI

Chaque contact (`ContactMedium`) :
- `mediumType` : `EMAIL` | `PHONE` — **obligatoire**
- `value` : valeur du contact

#### ExpertiseReport (compte-rendu)
- `presenceStatus` : statut de présence des techniciens — **obligatoire**
- `responsibility` : `DISPUTE` | `OC` | `OI` — **obligatoire**
- `pboReading` : lecture au PBO
- `pmReading` : lecture au PM
- `report` : rapport détaillé

**PresenceStatus** :
- `ocTechnicianPresent` : technicien OC présent (boolean) — **obligatoire**
- `oiTechnicianPresent` : technicien OI présent (boolean) — **obligatoire**
- `ocTechnicianAbsenceAcknowledgment` : numéro de décharge pour absence OC

**PBOReading** (champs obligatoires marqués *) :
- `pboReference`* : référence PBO
- `cableToPMReference`* : référence câble vers PM
- `tubeColorToPM`* : couleur du tube vers PM
- `fiberColorToPM`* : couleur de la fibre vers PM
- `pboSignalMeasurement`* : mesure du signal au PBO réalisée (boolean)
- `downstreamSignalValue` : valeur du signal descendant (float)
- `upstreamSignalValue` : valeur du signal montant (float)

**PMReading** (champs obligatoires marqués *) :
- `pmReference`* : référence PM
- `garterStatus`* : état jarretière (`OK` | `KO`)
- `pmSignalMeasurement` : mesure du signal au PM réalisée (boolean)
- `upstreamSignalValue` : valeur du signal montant (float)

**Report** (tous champs obligatoires sauf mention contraire) :
- `identifiedDefect` : défaut identifié (boolean)
- `interventionDescription` : description de l'intervention
- `isAdditionalInterventionNecessary` : intervention complémentaire nécessaire (boolean)
- `opticalRouteCompliantWithReference` : route optique conforme au référentiel OI (boolean)
- `pboCableReference`, `pboFiberInformation`, `pboTubeInformation` : infos PBO
- `pmModuleCableReference`, `pmModuleFiberInformation`, `pmModuleName`, `pmModulePosition`, `pmModuleTubeInformation` : infos module PM
- `signalOKAfterExpertise` : signal OK après expertise (boolean)
- `socketConnectorColor` : `BLUE` | `GREEN` | `RED` | `YELLOW`
- `socketConnectorNumber` : numéro connecteur prise (integer)
- `isOpticalRouteToBeModified` : route optique à modifier ROA (boolean)
- `additionalIntervention` : intervention complémentaire (optionnel)
- `opticalRouteInformation` : info route optique (optionnel)

**AdditionalIntervention** :
- `intervenor` : `OC` | `OI` — opérateur devant intervenir
- `interventionNature` : `GC_PUBLIC`, `PROBLEME_ADDUCTION`, `MISE_A_JOUR_SI_OI`, `DESATURATION`, `AJOUT_CAPACITE`, `PBO_INTROUVABLE`, `PROBLEME_DE_CHEMINEMENT_SUR_PARTIE_PUBLIQUE`, `PROBLEME_DE_CONTINUITE`, `AUTRE_INTERVENTION_COMPLEMENTAIRE`
- `interventionId` : identifiant intervention
- `dischargeNumber` : numéro de décharge
- `sendFailureFlowRequired` : envoi flux échec nécessaire (boolean)

**OpticalRouteInformation** :
- `dischargeNumber` : numéro de décharge si envoi flux échec nécessaire
- `sendFailureFlowRequired` : envoi flux échec nécessaire (boolean)

#### 2.1.1 statusChangeReason (motifs de changement d'état)
| Motif |
|-------|
| `DEFAUT_NON_CORRIGE` |
| `DEFAUT_CONFIRME_ET_CORRIGE_PAR_OI` |
| `PRELOC_ERRONEE_DEFAUT_CORRIGE_PAR_OI` |
| `CLOTURE_DU_TICKET_D_INCIDENT_GENERALISE` |
| `DEFAUT_CAUSE_PAR_UN_TIERS_IDENTIFIE` |
| `DEFAUT_LIE_A_DES_CAUSES_EXCEPTIONNELLES` |
| `DEFAUT_LIE_A_UN_VANDALISME` |
| `PAS_DE_DEFAUT_CONSTATE_SUR_LA_PARTIE_OI` |
| `DEFAUT_DETECTE_RESPONSABILITE_OC` |
| `TECHNICIEN_OC_ABSENT` |
| `TECHNICIEN_OI_ABSENT` |
| `RESEAU_OI_INACCESSIBLE` |
| `TRAITEMENT_IMPOSSIBLE_IDENTIFIANT_COMMANDE_INTERNE_OC_INCONNU` |
| `TRAITEMENT_IMPOSSIBLE_CHAMPS_OBLIGATOIRES_MANQUANTS` |
| `TRAITEMENT_IMPOSSIBLE_CHAMPS_INCOHERENTS` |
| `TRAITEMENT_IMPOSSIBLE_COMMANDE_EN_REFERENCE_PAS_DANS_L_ETAT_ATTENDU` |
| `TRAITEMENT_IMPOSSIBLE_ID_EXPERTISE_OC_DEJA_UTILISE` |
| `TRAITEMENT_IMPOSSIBLE_DEMANDE_EXPERTISE_DEJA_EN_COURS` |
| `TRAITEMENT_IMPOSSIBLE_TRAVAUX_EN_COURS_OU_PLANIFIES` |
| `TRAITEMENT_IMPOSSIBLE_NOMBRE_ECHECS_NON_ATTEINT` |
| `TRAITEMENT_IMPOSSIBLE_REFUS_OC` |
| `RDV_ID_RDV_INCONNU_POUR_OC_DEMANDEUR` |
| `RDV_ETAT_RDV_NON_VALIDE` |
| `AUTRE_MOTIFS_LIBRES` |
| `RESILIATION_ANNULATION_RECUE` |
| `ANNULATION_EXPERTISE_SUR_DEMANDE_OC` |
| `IMPOSSIBILITE_PRISE_DE_RDV` |

#### 2.1.2 statusChangeReasonCode (codes motif)
| Code | Code | Code |
|------|------|------|
| `ERR08` | `ERR09` | `EXC02` |
| `EXC03` | `EXC04` | `EXC06` |
| `EXC07` | `FAUT01` | `FEXP01` |
| `FEXP02` | `FEXP03` | `FEXP04` |
| `FEXP05` | `FEXP06` | `FIMP14` |
| `FIMP15` | `FIMP16` | `FRDV03` |
| `FRDV04` | `RET01` | `RET02` |
| `RET03` | `RET04` | `RET05` |
| `RET06` | `STT01` | `STT02` |

### 2.2 Notes
- `GET /expertise-conjointe-productions/{ticketId}/note` : liste paginée des notes  
- `HEAD /expertise-conjointe-productions/{ticketId}/note` : comptage  
- `POST /expertise-conjointe-productions/{ticketId}/note` : création d'une note

### 2.3 Pièces jointes
- `GET /expertise-conjointe-productions/{ticketId}/attachment` : liste paginée des pièces  
- `HEAD /expertise-conjointe-productions/{ticketId}/attachment` : comptage  
- `POST /expertise-conjointe-productions/{ticketId}/attachment` : **création des métadonnées** (multi-part)
- `GET /expertise-conjointe-productions/{ticketId}/attachment/{attachmentId}/content` : **téléchargement**  
- `POST /expertise-conjointe-productions/{ticketId}/attachment/{attachmentId}/content/{chunkIndex}` : **upload d'un tronçon**  
- `POST /expertise-conjointe-productions/{ticketId}/attachment/{attachmentId}/finalize` : **finalisation**

**Types MIME autorisés** :
- `application/pdf`
- `application/vnd.oasis.opendocument.spreadsheet`
- `application/vnd.oasis.opendocument.text`
- `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`
- `application/vnd.openxmlformats-officedocument.wordprocessingml.document`
- `text/csv`
- `image/jpeg`, `image/jpg`, `image/png`, `image/svg+xml`

**Schémas Attachment** :
- `AttachmentECProductionMonoPart` : pièce jointe simple
- `AttachmentECProductionMultiPart` : pièce jointe multi-tronçons
- `AttachmentECProductionMonoPartStandard` : mono-part avec types MIME standards
- `AttachmentECProductionMultiPartStandard` : multi-part avec types MIME standards

**Pattern multi-part (upload par tronçons)**
1. `POST /attachment` (métadonnées avec `size` et `sha256` obligatoires) → retourne `id`, `ETag`, `X-Chunk-Size`
2. boucle : `POST /attachment/{id}/content/{chunkIndex}` (tronçon binaire + métadonnées `Chunk`)
3. `POST /attachment/{id}/finalize`

### 2.4 Événements
- `GET /expertise-conjointe-productions/event` : liste paginée des événements  
- `HEAD /expertise-conjointe-productions/event` : comptage  
- `POST /expertise-conjointe-productions/event` : diffusion d'un événement

**Modèle** : `Event` (hérite de CloudEvent)
- Champs requis : `time`, `subject` (UUID), `type`
- `datacontenttype` : `application/json` (défaut)

**Types d'événements** :
| Type | Modèle data |
|------|-------------|
| `fr.interop.ecp.create` | `TicketContradictoryProductionExpertise` |
| `fr.interop.ecp.update` | `TicketContradictoryProductionExpertise` |
| `fr.interop.ecp.note.create` | `Note` |
| `fr.interop.ecp.attachment.create` | `AttachmentECProduction` |
| `fr.interop.ecp.attachment.update` | `AttachmentECProduction` |

---

## 3) Recherche, pagination, tri et filtrage
Toutes les listes prennent en charge :
- `limit` (défaut 50, max 100), `offset` (défaut 0)
- `sort` (ex. `-creationDate`)
- `fields` (projection partielle des champs)
- `filters` (filtres dynamiques sur propriétés)

> Le **HEAD** sur les collections retourne `X-Total-Count` ; les **GET** retournent `X-Total-Count` et `X-Result-Count`.

---

## 4) Schéma Address

L'adresse utilise principalement du **camelCase** avec quelques exceptions en **snake_case** :

| Attribut | Format |
|----------|--------|
| `@baseType`, `@schemaLocation`, `@type` | camelCase |
| `city`, `country`, `locality`, `postcode` | camelCase |
| `stateOrProvince`, `streetName`, `streetNr`, `streetNrSuffix`, `streetSuffix`, `streetType` | camelCase |
| `code_ban`, `code_hexacle`, `code_insee`, `code_rivoli` | **snake_case** |

> `code_hexacle` : exactement 10 caractères (minLength: 10, maxLength: 10)

---

## 5) Exemples

### 5.1 Créer un ticket
```bash
curl -X POST \
  "{protocol}://{host}:{port}{basePath}/expertise-conjointe-productions" \
  -H "Content-Type: application/json" \
  -d '{
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
        "pm": { /* … schéma Interop PM … */ }
      }
    }
  }'
```

### 5.2 Mettre à jour un ticket (If-Match)
```bash
curl -X PUT \
  "{protocol}://{host}:{port}{basePath}/expertise-conjointe-productions/{ticketId}" \
  -H "Content-Type: application/json" \
  -H "If-Match: \"<ETag-lu-au-GET>\"" \
  -d '{
    "@type": "expertiseConjointeProduction",
    "code_oc": "ABCD",
    "code_oi": "WXYZ",
    "initialRequest": "OI",
    "expertise": { /* … */ }
  }'
```

### 5.3 Ajouter une note
```bash
curl -X POST \
  "{protocol}://{host}:{port}{basePath}/expertise-conjointe-productions/{ticketId}/note" \
  -H "Content-Type: application/json" \
  -d '{
    "text": "Techniciens présents, test de signal effectué.",
    "author": "oc.user@domain.tld"
  }'
```

### 5.4 Créer une pièce et envoyer le contenu par tronçons
1. **Métadonnées**
```bash
curl -X POST \
  "{protocol}://{host}:{port}{basePath}/expertise-conjointe-productions/{ticketId}/attachment" \
  -H "Content-Type: application/json" \
  -d '{
    "@type": "documentMultiPart",
    "name": "pv-ecp.pdf",
    "description": "PV signé",
    "size": 1048576,
    "sha256": "<hash>"
  }'
```
2. **Upload d'un tronçon** (répéter pour chaque index)
```bash
curl -X POST \
  "{protocol}://{host}:{port}{basePath}/expertise-conjointe-productions/{ticketId}/attachment/{attachmentId}/content/{chunkIndex}" \
  -F "file=@chunk.bin" \
  -F 'metadata={"index":{chunkIndex},"size":N,...};type=application/json'
```
3. **Finaliser**
```bash
curl -X POST \
  "{protocol}://{host}:{port}{basePath}/expertise-conjointe-productions/{ticketId}/attachment/{attachmentId}/finalize"
```

### 5.5 Diffuser un événement
```bash
curl -X POST \
  "{protocol}://{host}:{port}{basePath}/expertise-conjointe-productions/event" \
  -H "Content-Type: application/json" \
  -d '{
    "id": "c7f2ec8a-2c19-4c7a-b8a6-8b6f2b2f1b7a",
    "source": "ecp/oc",
    "type": "fr.interop.ecp.update",
    "subject": "c5b0e2e8-6e2d-4b7a-8dc9-ffb3b6b8c001",
    "time": "2025-01-12T10:15:30Z",
    "datacontenttype": "application/json",
    "data": { /* … Ticket/Note/Attachment … */ }
  }'
```

> Les valeurs d'exemple sont indicatives ; adapter selon vos contraintes de sécurité et de routage.

---

## 6) Conventions & bonnes pratiques
- **Nommage champs** : principalement `camelCase`, avec quelques attributs en `snake_case` :
  - `code_oc`, `code_oi` (ticket)
  - `code_ban`, `code_hexacle`, `code_insee`, `code_rivoli` (adresse)
- **Contrôle de concurrence** : toujours utiliser `If-Match` lors des modifications.
- **Pagination** : privilégier `HEAD` pour obtenir le volume total avant de paginer.
- **Pièces jointes volumineuses** : suivre le **pattern multi-part** (tronçonnage + finalisation).
- **Événements** : respecter le format CloudEvents (`time`, `type`, `subject`, `datacontenttype`).

---

## 7) Références de la spécification
- Spécification **YAML** : `expertise-conjointe-production.yaml`
- Spécification **JSON** : `expertise-conjointe-production.json`

> Pour plus de détails de structure (ex. schémas PM, TroubleTicket, Note, Attachment, CloudEvent), se référer aux `$ref` Interop inclus dans la spécification.

---