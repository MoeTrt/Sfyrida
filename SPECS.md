# Sfyrida — Spécifications & Architecture

> Application mobile de **chats éphémères géolocalisés**.
> Quand tu arrives dans un lieu (bibliothèque, festival…), l'app crée/rejoint un
> chat éphémère de **12 h** rattaché à ce lieu, et y réunit **tes amis présents**
> que tu n'aurais pas forcément vus.

- **Stack** : Flutter (iOS + Android)
- **Backend** : Firebase (Auth, Firestore, Cloud Functions, Cloud Messaging)
- **Modèle social** : amis uniquement (graphe d'amis interne)

---

## 1. Concept & vocabulaire

| Terme | Définition |
|---|---|
| **Lieu (place)** | Zone géographique détectée par clustering GPS (geohash). Pas de carte de POI au MVP. |
| **Session de lieu** | Un chat éphémère rattaché à un lieu, **durée de vie 12 h** à partir de sa création. |
| **Présence** | Le fait qu'un utilisateur soit « dans » une session (arrivée détectée). Expire vite (ex. 30 min sans signal). |
| **Ami** | Lien réciproque accepté entre deux utilisateurs. Seul un ami présent est visible. |

Boucle de valeur : **j'arrive → l'app détecte le lieu → je vois mes amis présents → on discute → tout disparaît au bout de 12 h.**

---

## 2. La décision centrale : « amis uniquement » + chat de lieu

C'est le point le plus délicat du produit. Le graphe d'amis **n'est pas transitif**
(A↔B, B↔C, mais pas A↔C), alors qu'un chat de groupe suppose une **conversation
partagée**. On a tranché en faveur de la cohérence de la conversation.

**Modèle retenu — « cluster d'amis » (composante connexe), profils complets :**

- Un salon = un **cluster d'amis** rattaché à un lieu : un groupe de présents reliés
  de proche en proche par l'amitié (ex. A-B-C-D forment un seul cluster via A et B).
- **Conséquence assumée** : on accepte de voir des **amis d'amis** présents. C, ami
  de A seulement, verra D (ami de B) dans le salon — et **avec son profil complet**,
  comme un participant normal. La conversation reste ainsi cohérente pour tous.

### Règle d'entrée qui garantit le cluster (clé d'implémentation)
On ne sait pas calculer une composante connexe en temps réel dans Firestore sans
complexité (union-find). On obtient le même résultat avec une **règle d'admission** :

> Tu ne peux **rejoindre** une session de lieu que si **au moins un de tes amis y est
> déjà présent**. Sinon, tu **crées** une nouvelle session sur ce lieu.

Par récurrence, chaque membre est donc relié au fondateur → **toute session est, par
construction, une seule composante connexe d'amis**. Deux inconnus (sans ami commun
présent) sur le même lieu créent **deux sessions distinctes** — ce qui est voulu.

> ⚠️ **Limite connue (→ V2)** : si une personne est amie avec des membres de **deux**
> sessions du même lieu, elle « ponterait » deux clusters. Au MVP elle rejoint la
> première trouvée (pas de **fusion** de sessions). La fusion de salons est repoussée.

### Bornage du groupe (anti-croissance non bornée)
La règle d'admission seule laisse la chaîne s'étendre à l'infini
(`A→B→C→D→…`) : dans un grand événement, **tout le monde finirait transitivement
relié = un seul chat géant inutile**. On borne donc sur deux axes :

1. **Distance sociale — `MAX_HOPS = 2`** (sauts depuis le fondateur).
   On attache à chaque présence un `hopLevel` figé à l'arrivée :
   - fondateur → `hopLevel = 0` ;
   - à l'arrivée : `monHop = 1 + min(hopLevel de mes amis présents dans la session)` ;
   - `monHop ≤ 2` → je rejoins ; sinon → je **fonde une nouvelle session**.

   ```
   A (fondateur) hop 0 → B ami de A : hop 1 ✅ → C ami de B : hop 2 ✅ → D ami de C : hop 3 ❌
   ```
   ✅ Amis + amis d'amis, puis stop.

2. **Taille — `MAX_PARTICIPANTS = 25`** : plafond dur, filet de sécurité
   indépendant de la topologie. Session pleine → on fonde une nouvelle session.

> ℹ️ Le `hopLevel` est **ancré au fondateur** (1ᵉʳ arrivé) et figé à l'entrée — choix
> assumé : le salon étant *partagé*, une mesure « 2 sauts depuis chacun » donnerait
> une vue différente par personne (= fragmentation, déjà écartée). On ne recalcule
> pas les hops si des membres partent (acceptable au MVP).

---

## 3. Détection de lieu & géolocalisation

### 3.1 Principe : geohash + fenêtre temporelle
- On encode la position en **geohash** (précision ~6 ≈ 1,2 km × 0,6 km ; ~7 ≈ 150 m).
- À l'arrivée, on liste les **sessions actives** (non expirées) sur ce geohash
  (+ voisins), puis on rejoint celle où **au moins un de mes amis est présent**
  (cf. §2, règle d'admission) ; sinon on **crée** une nouvelle session. Plusieurs
  sessions peuvent donc coexister sur un même lieu (un cluster d'amis chacune).
- Recommandation : **précision 7** pour un festival/bibliothèque (zone fine), avec
  recherche sur les cellules voisines pour éviter les coupures de bordure.

### 3.2 Niveaux de détection (par phase)
1. **MVP — check-in au premier plan** : détection quand l'app est ouverte
   (permission *« pendant l'utilisation »*). Simple, conforme, peu gourmand en batterie.
2. **V2 — geofencing arrière-plan** : déclenchement à l'arrivée même app fermée
   (permission *« toujours »*). C'est le cœur de l'expérience « magique » mais
   coûteux (batterie) et scruté à la revue des stores → justification d'usage requise.

### 3.3 Packages Flutter pressentis
- `geolocator` — position courante & permissions.
- `geoflutterfire_plus` (ou calcul geohash maison) — requêtes géo sur Firestore.
- `flutter_background_geolocation` ou `geofence_service` — phase V2 (arrière-plan).

---

## 4. Modèle de données (Firestore)

```
users/{uid}
  displayName: string
  username: string            // unique, pour ajout d'amis
  photoUrl: string?
  fcmTokens: string[]         // pour les notifications push
  createdAt: timestamp

friendships/{uid}
  friends/{friendUid}
    since: timestamp
    // présence d'un doc = amitié confirmée (réciproque, écrite des 2 côtés)

friendRequests/{requestId}
  from: uid
  to: uid
  status: "pending" | "accepted" | "declined"
  createdAt: timestamp

placeSessions/{sessionId}
  geohash: string             // ex. précision 7
  geohashPrefixes: string[]   // pour requêtes voisines
  center: geopoint
  label: string?              // nom optionnel saisi par le créateur
  founderUid: uid
  participantCount: number    // maintenu par Cloud Function, plafonné à MAX_PARTICIPANTS (25)
  createdAt: timestamp
  expiresAt: timestamp        // = createdAt + 12 h  (champ TTL Firestore)

  presence/{uid}
    hopLevel: number          // 0 = fondateur ; figé à l'arrivée ; doit rester ≤ MAX_HOPS (2)
    joinedAt: timestamp
    lastSeen: timestamp
    expiresAt: timestamp      // TTL court (ex. lastSeen + 30 min)

  messages/{msgId}
    authorUid: uid
    text: string
    createdAt: timestamp
    expiresAt: timestamp      // = session.expiresAt (TTL)
```

### Éphémérité automatique
- On active une **policy TTL Firestore** sur le champ `expiresAt` des collections
  `placeSessions`, `presence` et `messages` → suppression automatique côté serveur.
- TTL Firestore n'étant pas temps réel (purge dans les ~24 h), on **filtre aussi
  côté client** sur `expiresAt > now` pour ne jamais afficher de contenu périmé.

---

## 5. Flux utilisateurs

### 5.1 Onboarding & auth
1. Auth **Firebase** par **email + mot de passe** (décidé).
2. Choix d'un `username` unique + `displayName` + photo facultative.
3. Demande de permission **localisation** (avec écran d'explication *avant* le prompt système).

### 5.2 Gestion des amis
- Ajout par `username` (et/ou via contacts en V2).
- Demande → acceptation → écriture réciproque dans `friendships`.

### 5.3 Arrivée sur un lieu (cœur du produit)
1. L'app obtient la position → calcule le geohash.
2. Liste les `placeSession` actives sur ce geohash (+ voisins).
3. Cherche parmi elles une session où **un ami est déjà présent**, avec
   `monHop = 1 + min(hopLevel des amis présents) ≤ 2` **et** `participantCount < 25`
   → la rejoint (crée sa `presence` avec ce `hopLevel`). Sinon → **crée** une
   nouvelle session (`hopLevel = 0`) puis sa présence.
4. **Cloud Function** sur création de présence → notifie les **amis** déjà présents
   (et/ou prévient l'arrivant de la présence d'amis).
5. L'écran de chat affiche le **roster complet du cluster** (amis directs + amis
   d'amis, profils complets) + le fil de messages partagé.

### 5.4 Chat & sortie
- Messages temps réel via listener Firestore.
- `presence.lastSeen` rafraîchi périodiquement ; à la fermeture/éloignement, la
  présence expire (TTL court).
- À T+12 h : session + messages purgés. Plus rien n'est consultable.

---

## 6. Notifications (Cloud Functions + FCM)
- **Trigger** : création d'un doc `presence`.
- La fonction lit la liste d'amis de l'arrivant **présents** dans la même session
  et envoie un push : *« X est arrivé·e à [lieu] »*.
- Inversement, à l'arrivée, on peut résumer à l'utilisateur : *« 3 amis ici »*.
- Gestion des `fcmTokens` (multi-appareils, nettoyage des tokens invalides).

---

## 7. Confidentialité, sécurité & conformité
- **Minimisation** : on ne stocke **jamais l'historique de localisation**. Seule la
  présence courante (éphémère) existe ; tout est purgé sous 12 h. → argument produit fort.
- **RGPD** : consentement explicite localisation, droit à l'effacement (l'éphémérité aide),
  politique de confidentialité, export des données de compte.
- **Stores** : la permission localisation *« toujours »* (V2) doit être justifiée à la revue ;
  prévoir un texte d'usage clair. Le MVP en *« pendant l'utilisation »* passe plus facilement.
- **Firestore Security Rules** :
  - lire/écrire `friendships` uniquement pour soi ;
  - lire `presence`/`messages` d'une session **uniquement si on a une présence valide** ;
  - un message ne peut être écrit que par son `authorUid` authentifié.
- **Modération / abus** : signalement de message, blocage utilisateur, anti-spam (rate limit Functions).

---

## 8. Architecture applicative (Flutter)
- **State management** : **Riverpod** (recommandé) — providers pour auth, position, session courante, messages.
- **Découpage features** :
  ```
  lib/
    core/            (firebase init, theme, router, utils geohash)
    features/
      auth/
      friends/
      location/      (service de détection + geohash)
      session/       (résolution/jointure de session de lieu)
      chat/          (UI + stream messages)
      notifications/
  ```
- **Navigation** : `go_router`.
- **Modèles** : classes immuables + `freezed`/`json_serializable` (ou `dart mappable`).

---

## 9. Roadmap MVP → V1

**Phase 0 — Fondations**
- [ ] Init projet Flutter + Firebase (Auth, Firestore, FCM).
- [ ] Modèles de données + Security Rules de base.
- [ ] Auth + onboarding (username).

**Phase 1 — MVP testable (check-in au premier plan)**
- [ ] Gestion des amis (ajout par username, demandes).
- [ ] Service localisation + geohash + résolution de session.
- [ ] Écran chat temps réel + roster filtré aux amis.
- [ ] TTL 12 h (sessions/messages) + TTL court présence.

**Phase 2 — L'expérience « magique »**
- [ ] Geofencing en arrière-plan (arrivée détectée app fermée).
- [ ] Notifications push « ami arrivé / amis présents ».

**Phase 3 — Affinage**
- [ ] Modération (signalement/blocage), lieux nommés/POI, partage d'invitation,
      réglages de visibilité, métriques.

---

## 10. Risques & points à surveiller
- **Batterie & arrière-plan** : le geofencing continu est le principal risque techno (V2).
- **Cold start réseau** : sans amis présents, l'app paraît « vide » → soigner l'onboarding social.
- **Revue stores** : justifier la localisation « toujours ».
- **Bordures de geohash** : deux amis côte à côte mais sur deux cellules → requête sur voisins indispensable.
- **Coûts Firestore** : listeners temps réel + écritures de présence fréquentes → batcher les `lastSeen`.

---

## 11. Décisions ouvertes (à trancher)
1. ~~**Méthode d'auth**~~ → **email + mot de passe** (décidé).
2. **Précision geohash** : 7 (fin) confirmé ?
3. ~~**Modèle social**~~ → **cluster d'amis**, profils complets, admission « ≥1 ami présent » (décidé). Fusion de clusters repoussée en V2.
4. **Rayon de présence** : définir la distance max au centre pour rester « présent ».
