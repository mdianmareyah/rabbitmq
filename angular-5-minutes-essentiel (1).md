# Angular en 5 Minutes - L'Essentiel pour Comprendre l'Architecture

## 🎯 Vue d'ensemble en 30 secondes

```
Angular = Framework Frontend Complet
├─ Components (UI) → Ce que l'utilisateur voit
├─ Services (Logique) → Traitement des données
├─ Modules (Organisation) → Regroupement logique
├─ Forms (Formulaires) → Saisie utilisateur
└─ HttpClient (API) → Communication backend
```

---

# 1. Architecture Angular (1 min)

## Le Pattern MVC adapté

```
┌─────────────────────────────────────────────┐
│              APPLICATION                     │
├─────────────────────────────────────────────┤
│                                             │
│  ┌──────────┐         ┌──────────┐         │
│  │Component │◄────────┤ Service  │         │
│  │  (Vue)   │         │(Business)│         │
│  └────┬─────┘         └────┬─────┘         │
│       │                    │                │
│       │ Affiche            │ HTTP           │
│       ▼                    ▼                │
│  ┌──────────┐         ┌──────────┐         │
│  │ Template │         │HttpClient│         │
│  │  (HTML)  │         │   (API)  │         │
│  └──────────┘         └────┬─────┘         │
│                            │                │
└────────────────────────────┼────────────────┘
                             │
                             ▼
                      Backend API
```

## Analogie simple

```
Restaurant:
├─ Component = Serveur (interface avec client)
├─ Template = Menu (ce que voit le client)
├─ Service = Cuisine (prépare les données)
└─ HttpClient = Livreur (apporte ingrédients du marché)
```

---

# 2. Components (1 min)

## Un Component = 3 fichiers

```typescript
// 📄 client.component.ts (TypeScript - Logique)
@Component({
  selector: 'app-client',     // Nom du tag HTML
  templateUrl: './client.html', // Fichier HTML
  styleUrls: ['./client.css']   // Fichier CSS
})
export class ClientComponent {
  clients: Client[] = [];      // Données
  
  constructor(
    private clientService: ClientService  // Injection service
  ) {}
  
  ngOnInit() {                 // Au démarrage
    this.loadClients();
  }
  
  loadClients() {
    this.clientService.getAll().subscribe(
      data => this.clients = data
    );
  }
}
```

```html
<!-- 📄 client.component.html (Template - Vue) -->
<div>
  <h2>Liste des Clients</h2>
  <div *ngFor="let client of clients">
    {{ client.nom }} - {{ client.email }}
  </div>
</div>
```

```css
/* 📄 client.component.css (Style) */
h2 { color: blue; }
```

## Cycle de vie simplifié

```
1. Constructor()      → Création
2. ngOnInit()        → Initialisation (charger données)
3. ngOnChanges()     → Quand @Input change
4. ngOnDestroy()     → Avant destruction
```

---

# 3. Services (1 min)

## Pourquoi un Service ?

```
❌ SANS Service (Mauvais):
Component A ──┐
              ├──> HttpClient.get() → API
Component B ──┘
→ Code dupliqué, difficile à maintenir

✅ AVEC Service (Bon):
Component A ──┐
              ├──> ClientService ──> HttpClient → API
Component B ──┘
→ Code centralisé, réutilisable
```

## Anatomie d'un Service

```typescript
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';

@Injectable({
  providedIn: 'root'  // Service disponible partout
})
export class ClientService {
  private apiUrl = 'http://localhost:8080/api/clients';

  constructor(private http: HttpClient) {}

  // GET tous les clients
  getAll(): Observable<Client[]> {
    return this.http.get<Client[]>(this.apiUrl);
  }

  // GET un client par ID
  getById(id: number): Observable<Client> {
    return this.http.get<Client>(`${this.apiUrl}/${id}`);
  }

  // POST créer un client
  create(client: Client): Observable<Client> {
    return this.http.post<Client>(this.apiUrl, client);
  }

  // PUT modifier un client
  update(id: number, client: Client): Observable<Client> {
    return this.http.put<Client>(`${this.apiUrl}/${id}`, client);
  }

  // DELETE supprimer un client
  delete(id: number): Observable<void> {
    return this.http.delete<void>(`${this.apiUrl}/${id}`);
  }
}
```

## Utilisation dans un Component

```typescript
export class ClientListComponent {
  clients: Client[] = [];

  constructor(private clientService: ClientService) {}

  ngOnInit() {
    // Subscribe = Écouter le résultat
    this.clientService.getAll().subscribe({
      next: (data) => {
        this.clients = data;  // Succès
      },
      error: (err) => {
        console.error(err);   // Erreur
      }
    });
  }
}
```

---

# 4. Forms (1 min)

## 2 types de formulaires

### A. Template-Driven (Simple)

```typescript
import { FormsModule } from '@angular/forms';

export class ClientFormComponent {
  client = {
    nom: '',
    prenom: '',
    email: ''
  };

  onSubmit() {
    console.log(this.client);
  }
}
```

```html
<form (ngSubmit)="onSubmit()">
  <input [(ngModel)]="client.nom" name="nom" required>
  <input [(ngModel)]="client.prenom" name="prenom" required>
  <input [(ngModel)]="client.email" name="email" type="email">
  <button type="submit">Envoyer</button>
</form>
```

### B. Reactive Forms (Recommandé)

```typescript
import { FormBuilder, Validators, ReactiveFormsModule } from '@angular/forms';

export class ClientFormComponent {
  clientForm = this.fb.group({
    nom: ['', [Validators.required, Validators.minLength(2)]],
    prenom: ['', Validators.required],
    email: ['', [Validators.required, Validators.email]]
  });

  constructor(private fb: FormBuilder) {}

  onSubmit() {
    if (this.clientForm.valid) {
      console.log(this.clientForm.value);
    }
  }
}
```

```html
<form [formGroup]="clientForm" (ngSubmit)="onSubmit()">
  <input formControlName="nom" 
         [class.error]="clientForm.get('nom')?.invalid">
  <div *ngIf="clientForm.get('nom')?.errors">
    Nom invalide
  </div>
  
  <input formControlName="email">
  <button [disabled]="clientForm.invalid">Envoyer</button>
</form>
```

## Validation rapide

```typescript
// Validations disponibles
Validators.required           // Obligatoire
Validators.minLength(n)       // Min n caractères
Validators.maxLength(n)       // Max n caractères
Validators.min(n)            // Valeur min
Validators.max(n)            // Valeur max
Validators.email             // Format email
Validators.pattern(regex)    // Pattern regex
```

---

# 5. HttpClient (1 min)

## Configuration

```typescript
// app.config.ts
import { provideHttpClient } from '@angular/common/http';

export const appConfig = {
  providers: [
    provideHttpClient()  // Activer HttpClient
  ]
};
```

## Les 5 méthodes essentielles

```typescript
constructor(private http: HttpClient) {}

// 1️⃣ GET - Récupérer
get(): Observable<Client[]> {
  return this.http.get<Client[]>('/api/clients');
}

// 2️⃣ POST - Créer
create(client: Client): Observable<Client> {
  return this.http.post<Client>('/api/clients', client);
}

// 3️⃣ PUT - Modifier
update(id: number, client: Client): Observable<Client> {
  return this.http.put<Client>(`/api/clients/${id}`, client);
}

// 4️⃣ DELETE - Supprimer
delete(id: number): Observable<void> {
  return this.http.delete<void>(`/api/clients/${id}`);
}

// 5️⃣ PATCH - Modifier partiellement
patch(id: number, changes: Partial<Client>): Observable<Client> {
  return this.http.patch<Client>(`/api/clients/${id}`, changes);
}
```

## Gestion des erreurs

```typescript
import { catchError } from 'rxjs/operators';
import { throwError } from 'rxjs';

getAll(): Observable<Client[]> {
  return this.http.get<Client[]>(this.apiUrl)
    .pipe(
      catchError(error => {
        console.error('Erreur:', error);
        return throwError(() => error);
      })
    );
}
```

## Headers et options

```typescript
const headers = new HttpHeaders({
  'Content-Type': 'application/json',
  'Authorization': 'Bearer token123'
});

this.http.get('/api/clients', { headers });

// Avec params
const params = new HttpParams()
  .set('page', '1')
  .set('size', '10');

this.http.get('/api/clients', { params });
```

---

# 6. Modules (30 secondes)

## Avant Angular 17 (Modules)

```typescript
@NgModule({
  declarations: [ClientComponent],  // Components
  imports: [CommonModule, FormsModule],  // Modules
  providers: [ClientService],       // Services
  exports: [ClientComponent]        // Exportés
})
export class ClientModule {}
```

## Depuis Angular 17 (Standalone)

```typescript
// Plus besoin de NgModule !
@Component({
  selector: 'app-client',
  standalone: true,  // ← Composant autonome
  imports: [CommonModule, FormsModule],  // Direct
  templateUrl: './client.html'
})
export class ClientComponent {}
```

---

# 7. Flux complet en 1 image

```
┌─────────────────────────────────────────────────────────┐
│                      BROWSER                             │
│  ┌───────────────────────────────────────────────────┐  │
│  │                  Component                         │  │
│  │  ┌──────────────────────────────────────────┐     │  │
│  │  │ TypeScript (.ts)                         │     │  │
│  │  │                                          │     │  │
│  │  │ clients: Client[] = [];                  │     │  │
│  │  │                                          │     │  │
│  │  │ constructor(                             │     │  │
│  │  │   private service: ClientService         │     │  │
│  │  │ ) {}                                     │     │  │
│  │  │                                          │     │  │
│  │  │ loadClients() {                          │     │  │
│  │  │   this.service.getAll()      ──────┐    │     │  │
│  │  │     .subscribe(                    │    │     │  │
│  │  │       data => this.clients = data  │    │     │  │
│  │  │     );                              │    │     │  │
│  │  │ }                                   │    │     │  │
│  │  └──────────────────────────────────────────┘     │  │
│  │                         │                         │  │
│  │                         ▼                         │  │
│  │  ┌──────────────────────────────────────────┐     │  │
│  │  │ Template (.html)                         │     │  │
│  │  │                                          │     │  │
│  │  │ <div *ngFor="let client of clients">     │     │  │
│  │  │   {{ client.nom }}                       │     │  │
│  │  │ </div>                                   │     │  │
│  │  └──────────────────────────────────────────┘     │  │
│  └───────────────────────────────────────────────────┘  │
│                         │                               │
│                         │ Utilise                       │
│                         ▼                               │
│  ┌───────────────────────────────────────────────────┐  │
│  │              Service (@Injectable)                 │  │
│  │  ┌──────────────────────────────────────────┐     │  │
│  │  │                                          │     │  │
│  │  │ constructor(                             │     │  │
│  │  │   private http: HttpClient               │     │  │
│  │  │ ) {}                                     │     │  │
│  │  │                                          │     │  │
│  │  │ getAll(): Observable<Client[]> {         │     │  │
│  │  │   return this.http.get(apiUrl)   ──────┐│     │  │
│  │  │ }                                       ││     │  │
│  │  └──────────────────────────────────────────┘     │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────┼───────────────────────────┘
                              │
                              │ HTTP Request
                              ▼
                   ┌──────────────────┐
                   │   Backend API    │
                   │  (Spring Boot)   │
                   └──────────────────┘
```

---

# 8. Checklist Rapide

## Pour créer une fonctionnalité complète :

```bash
# 1️⃣ Créer le modèle
# src/app/models/client.model.ts
export interface Client {
  id?: number;
  nom: string;
  email: string;
}

# 2️⃣ Créer le service
ng generate service services/client

# 3️⃣ Créer le component
ng generate component components/client-list

# 4️⃣ Ajouter la route
# app.routes.ts
{ path: 'clients', component: ClientListComponent }
```

## Template du Service (copier-coller)

```typescript
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class NomService {
  private apiUrl = 'http://localhost:8080/api/endpoint';

  constructor(private http: HttpClient) {}

  getAll(): Observable<Type[]> {
    return this.http.get<Type[]>(this.apiUrl);
  }

  getById(id: number): Observable<Type> {
    return this.http.get<Type>(`${this.apiUrl}/${id}`);
  }

  create(item: Type): Observable<Type> {
    return this.http.post<Type>(this.apiUrl, item);
  }

  update(id: number, item: Type): Observable<Type> {
    return this.http.put<Type>(`${this.apiUrl}/${id}`, item);
  }

  delete(id: number): Observable<void> {
    return this.http.delete<void>(`${this.apiUrl}/${id}`);
  }
}
```

## Template du Component (copier-coller)

```typescript
import { Component, OnInit } from '@angular/core';
import { CommonModule } from '@angular/common';
import { NomService } from '../../services/nom.service';

@Component({
  selector: 'app-nom',
  standalone: true,
  imports: [CommonModule],
  templateUrl: './nom.component.html'
})
export class NomComponent implements OnInit {
  items: Type[] = [];
  loading = false;
  error = '';

  constructor(private service: NomService) {}

  ngOnInit() {
    this.loadData();
  }

  loadData() {
    this.loading = true;
    this.service.getAll().subscribe({
      next: (data) => {
        this.items = data;
        this.loading = false;
      },
      error: (err) => {
        this.error = err.message;
        this.loading = false;
      }
    });
  }

  delete(id: number) {
    this.service.delete(id).subscribe({
      next: () => this.loadData(),
      error: (err) => alert(err.message)
    });
  }
}
```

---

# 9. Concepts clés à retenir

## Observable vs Promise

```typescript
// Promise (ancienne méthode)
fetch('/api/clients')
  .then(res => res.json())
  .then(data => console.log(data));

// Observable (Angular)
http.get('/api/clients')
  .subscribe(data => console.log(data));
```

**Différence** : Observable = flux de données continu (peut émettre plusieurs valeurs)

## Dependency Injection (DI)

```typescript
// ❌ Sans DI (Mauvais)
export class Component {
  service = new ClientService();  // Couplage fort
}

// ✅ Avec DI (Bon)
export class Component {
  constructor(private service: ClientService) {}
  // Angular injecte automatiquement
}
```

## Data Binding

```html
<!-- 1️⃣ Interpolation -->
<h1>{{ title }}</h1>

<!-- 2️⃣ Property Binding -->
<img [src]="imageUrl">

<!-- 3️⃣ Event Binding -->
<button (click)="save()">Sauvegarder</button>

<!-- 4️⃣ Two-Way Binding -->
<input [(ngModel)]="name">
```

## Directives structurelles

```html
<!-- *ngIf : Condition -->
<div *ngIf="isVisible">Contenu</div>

<!-- *ngFor : Boucle -->
<div *ngFor="let item of items">{{ item }}</div>

<!-- *ngSwitch : Switch -->
<div [ngSwitch]="status">
  <p *ngSwitchCase="'active'">Actif</p>
  <p *ngSwitchCase="'inactive'">Inactif</p>
  <p *ngSwitchDefault>Inconnu</p>
</div>
```

---

# 10. Résumé Ultra-Rapide

```
┌──────────────────────────────────────────────────┐
│  ANGULAR = 5 PILIERS                             │
├──────────────────────────────────────────────────┤
│                                                  │
│  1️⃣ COMPONENT                                    │
│     → Interface utilisateur (UI)                 │
│     → 3 fichiers : .ts, .html, .css             │
│                                                  │
│  2️⃣ SERVICE                                      │
│     → Logique métier réutilisable                │
│     → Communication HTTP                         │
│     → @Injectable()                              │
│                                                  │
│  3️⃣ FORMS                                        │
│     → Template-Driven (simple)                   │
│     → Reactive (avancé, recommandé)              │
│     → Validation intégrée                        │
│                                                  │
│  4️⃣ HTTPCLIENT                                   │
│     → GET, POST, PUT, DELETE                     │
│     → Retourne Observable<T>                     │
│     → .subscribe() pour écouter                  │
│                                                  │
│  5️⃣ ROUTING                                      │
│     → Navigation entre pages                     │
│     → Routes + RouterLink                        │
│     → <router-outlet>                            │
│                                                  │
└──────────────────────────────────────────────────┘
```

## Flux de données simplifié

```
User Input (Template)
        ↓
    Component
        ↓
     Service
        ↓
   HttpClient
        ↓
    Backend API
        ↓
   HttpClient
        ↓
     Service
        ↓
    Component
        ↓
Display (Template)
```

## Commandes essentielles

```bash
# Créer projet
ng new mon-projet

# Générer composant
ng generate component nom
ng g c nom  # raccourci

# Générer service
ng generate service services/nom
ng g s services/nom  # raccourci

# Démarrer app
ng serve

# Build production
ng build --configuration=production
```

---

# 🎯 Point Final : Le Pattern Complet

```typescript
// 1️⃣ MODEL (Interface)
export interface Client {
  id?: number;
  nom: string;
  email: string;
}

// 2️⃣ SERVICE (HTTP)
@Injectable({ providedIn: 'root' })
export class ClientService {
  constructor(private http: HttpClient) {}
  
  getAll(): Observable<Client[]> {
    return this.http.get<Client[]>('/api/clients');
  }
}

// 3️⃣ COMPONENT (Logique)
@Component({...})
export class ClientListComponent implements OnInit {
  clients: Client[] = [];
  
  constructor(private service: ClientService) {}
  
  ngOnInit() {
    this.service.getAll().subscribe(
      data => this.clients = data
    );
  }
}

// 4️⃣ TEMPLATE (Vue)
<div *ngFor="let client of clients">
  {{ client.nom }} - {{ client.email }}
</div>
```

**C'est tout ! Vous avez compris Angular en 5 minutes ! 🚀**

---

## Pour aller plus loin

- **Documentation officielle** : https://angular.io
- **RxJS** : https://rxjs.dev
- **Angular CLI** : https://angular.io/cli

**Maintenant, passez à la pratique avec le TP complet !** 💪
