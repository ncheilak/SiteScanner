# SiteLog v2.0
### Σύστημα Καταγραφής Παρουσίας Εργοταξίου

Εφαρμογή διαχείρισης παρουσιών για εργοτάξια, βασισμένη σε στατικά HTML αρχεία με Supabase ως backend. Χωρίς server, χωρίς build process — απλώς 5 αρχεία `.html` σε έναν φάκελο.

---

## Εργαλεία

| # | Αρχείο | Λειτουργία |
|---|--------|------------|
| — | `index.html` | Κεντρική σελίδα επιλογής εργαλείου |
| 01 | `0-worker-compiler-gr.html` | Καταχώρηση & διαχείριση εργαζομένων |
| 02 | `1-qr-generator-gr.html` | Αναζήτηση εργαζομένων & δημιουργία QR badges |
| 03 | `3-scanner-app-gr.html` | Σάρωση QR για καταγραφή παρουσίας |
| 04 | `2-attendance-gr.html` | Παρουσιολόγιο με εξαγωγή CSV |

---

## Αρχιτεκτονική

```
Browser (HTML/JS)
      │
      ▼
 Supabase (PostgreSQL)
      │
      ├── workers          ← στατικά στοιχεία εργαζομένων
      ├── sites            ← λίστα εργοταξίων
      └── attendance_log   ← παρουσίες (FK → workers, sites)
```

Κάθε HTML αρχείο επικοινωνεί απευθείας με το Supabase μέσω του JavaScript client SDK. Δεν υπάρχει κανένα middleware ή backend layer.

---

## Βάση Δεδομένων (Supabase Schema)

### `workers`
| Στήλη | Τύπος | Περιγραφή |
|-------|-------|-----------|
| `id` | `text` (PK) | Μοναδικό αναγνωριστικό (π.χ. `W-1001`) |
| `ama` | `text` | Αριθμός Μητρώου Ασφαλισμένου |
| `amka` | `text` | ΑΜΚΑ |
| `afm` | `text` | ΑΦΜ εργαζομένου |
| `ar_ad_ergasias` | `text` | Αριθμός Αδείας Εργασίας |
| `eponimo` | `text` | Επώνυμο |
| `onoma` | `text` | Όνομα |
| `patronimo` | `text` | Πατρώνυμο |
| `eidikotita` | `text` | Ειδικότητα |
| `default_phase` | `text` | Default φάση κατασκευής |
| `is_multi_phase` | `boolean` | Εργαζόμενος πολλαπλών φάσεων |
| `ergolabos` | `text` | Επωνυμία εργολάβου |
| `afm_ergolabou` | `text` | ΑΦΜ εργολάβου |
| `registered_at` | `timestamptz` | Ημερομηνία εγγραφής |

### `sites`
| Στήλη | Τύπος | Περιγραφή |
|-------|-------|-----------|
| `id` | `int4` (PK) | Αναγνωριστικό εργοταξίου |
| `name` | `text` | Όνομα εργοταξίου |
| `created_at` | `timestamptz` | — |

### `attendance_log`
| Στήλη | Τύπος | Περιγραφή |
|-------|-------|-----------|
| `id` | `int8` (PK) | Auto-increment |
| `worker_id` | `text` (FK → `workers.id`) | Εργαζόμενος |
| `site_id` | `int4` (FK → `sites.id`) | Εργοτάξιο |
| `time_in` | `text` | Ώρα προσέλευσης (`HH:MM`) |
| `log_date` | `date` | Ημερομηνία |
| `fasi` | `text` | Φάση κατασκευής για αυτό το check-in |
| `created_at` | `timestamptz` | — |

> **Σημείωση:** Το `attendance_log` αποθηκεύει μόνο `worker_id`, `site_id`, `fasi`, `time_in`, `log_date`. Τα υπόλοιπα στοιχεία (όνομα, ειδικότητα κ.λπ.) προκύπτουν μέσω JOIN με τον πίνακα `workers` κατά την ανάγνωση.

---

## Φάσεις Κατασκευής

```
01: Εκσκαφές & Οικοδομικός Σκελετός
02: Τοιχοποιίες
03: Επιχρίσματα (Σοβατίσματα)
04: Δάπεδα - Επενδύσεις
05: Διάφορες Εργασίες
06: Εργασίες Εγκαταστάσεων
```

---

## Ροή Εργασίας

```
1. Worker Compiler
   └── Καταχώρηση εργαζομένων στον πίνακα workers
        (default_phase + is_multi_phase flag)

2. QR Generator
   └── Αναζήτηση εργαζομένου (Επώνυμο / Όνομα / Εργολάβος dropdown)
        → 10 αποτελέσματα ανά σελίδα με pagination
        → δημιουργία QR badge (UTF-8 Byte mode)
        QR payload: { id, ep, on, ei, erg }
        (ep=επώνυμο, on=όνομα, ei=ειδικότητα, erg=εργολάβος)

3. Scanner (σε κινητό)
   └── Σάρωση QR badge
        → ανάγνωση payload { id, ep, on, ei, erg }
        → χειροκίνητη επιλογή φάσης κατασκευής (modal)
        └── Εγγραφή στο attendance_log

4. Παρουσιολόγιο
   └── JOIN attendance_log ⋈ workers ⋈ sites
        └── Εξαγωγή CSV με πλήρη στοιχεία
```

---

## Εγκατάσταση & Χρήση

### Απαιτήσεις
- Λογαριασμός [Supabase](https://supabase.com) (δωρεάν tier αρκεί)
- Οποιοσδήποτε web server ή τοπικό άνοιγμα αρχείων

### Βήματα

**1. Clone του repository**
```bash
git clone https://github.com/your-username/sitelog.git
cd sitelog
```

**2. Δημιουργία Supabase project**

Δημιουργήστε τους 3 πίνακες βάσει του schema παραπάνω. Ή εκτελέστε το παρακάτω SQL στο Supabase SQL Editor:

```sql
-- Sites
create table sites (
  id serial primary key,
  name text not null,
  created_at timestamptz default now()
);

-- Workers
create table workers (
  id text primary key,
  ama text, amka text, afm text,
  ar_ad_ergasias text,
  eponimo text, onoma text, patronimo text,
  eidikotita text,
  default_phase text,
  is_multi_phase boolean default false,
  ergolabos text, afm_ergolabou text,
  registered_at timestamptz default now()
);

-- Attendance Log
create table attendance_log (
  id bigint generated always as identity primary key,
  worker_id text references workers(id),
  site_id int references sites(id),
  time_in text,
  log_date date,
  fasi text,
  created_at timestamptz default now()
);
```

**3. Ρύθμιση Supabase credentials**

Σε κάθε HTML αρχείο, αντικαταστήστε:
```javascript
const SUPABASE_URL = 'https://your-project.supabase.co';
const SUPABASE_KEY = 'your-anon-public-key';
```

Τα credentials βρίσκονται στο Supabase Dashboard → Settings → API.

**4. Εκκίνηση**

Ανοίξτε το `index.html` σε browser — τοπικά ή μέσω οποιουδήποτε static hosting (GitHub Pages, Netlify κ.λπ.).

---

## CSV Import (Worker Compiler)

Το πρότυπο CSV έχει τις παρακάτω στήλες (η σειρά δεν έχει σημασία):

```
ama, amka, afm, arAdErgasias, eponimo, onoma, patronimo,
eidikotita, defaultPhase, isMultiPhase, ergolabos, afmErgolabou
```

Τιμές για `isMultiPhase`: `true` / `false`

Κατεβάστε το πρότυπο απευθείας από το εργαλείο Καταχώρησης Εργαζομένων.

---

## QR Generator — Λεπτομέρειες

- **Αναζήτηση** με τρία ανεξάρτητα πεδία: Επώνυμο (text), Όνομα (text), Εργολάβος (dropdown από τη βάση)
- **Pagination**: 10 αποτελέσματα ανά σελίδα, server-side με Supabase `.range()` + count query
- **QR encoding**: UTF-8 Byte mode — τα ελληνικά αποθηκεύονται σωστά και διαβάζονται από οποιονδήποτε QR reader
- **QR payload** (μόνο τα απαραίτητα για το check-in):

```json
{ "id": "W-1001", "ep": "Παπαδόπουλος", "on": "Γιάννης", "ei": "Εργάτης", "erg": "Εργολάβος ΑΕ" }
```

---

## Εξωτερικές Βιβλιοθήκες

| Βιβλιοθήκη | Χρήση |
|-----------|-------|
| [Supabase JS v2](https://github.com/supabase/supabase-js) | Επικοινωνία με βάση δεδομένων |
| [ZXing](https://github.com/zxing-js/library) | Σάρωση QR code μέσω κάμερας |
| [qrcode-generator](https://github.com/kazuhikoarase/qrcode-generator) | Δημιουργία QR code |
| [IBM Plex Sans/Mono, Bebas Neue](https://fonts.google.com) | Γραμματοσειρές |

Όλες φορτώνονται μέσω CDN — δεν απαιτείται `npm install`.

---

## Περιορισμοί & Σημειώσεις

- Χρησιμοποιείται το **Supabase anon/public key** — βεβαιωθείτε ότι οι πολιτικές RLS (Row Level Security) είναι ρυθμισμένες κατάλληλα για production χρήση.
- Το σύστημα καταγράφει μόνο **check-in** (είσοδο) — δεν υπάρχει check-out.
- Ο scanner είναι βελτιστοποιημένος για κινητές συσκευές με κάμερα.
- Η εφαρμογή είναι στα **ελληνικά**.

---

## Roadmap

- [ ] Στατιστικά & γραφήματα παρουσιών
- [ ] Check-out καταγραφή (ώρα αναχώρησης)
- [ ] Αποστολή ημερήσιας αναφοράς μέσω email
- [ ] PWA υποστήριξη για offline χρήση scanner

---

## License

MIT
