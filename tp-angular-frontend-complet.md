# TP Complet - Frontend Angular pour Microservices Bancaires

**Module** : Architecture Microservices - Frontend  
**Sujet** : Application Angular connectée aux microservices  
**Durée** : 4 heures  
**Niveau** : Master 2  
**Prérequis** : TypeScript, HTML/CSS, Angular basics

---

## 📚 Table des matières

1. [Introduction et architecture](#1-introduction)
2. [Installation et configuration](#2-installation)
3. [Partie 1 : Configuration du projet](#3-partie-1-configuration)
4. [Partie 2 : Service Client](#4-partie-2-service-client)
5. [Partie 3 : Service Compte](#5-partie-3-service-compte)
6. [Partie 4 : Composants et routing](#6-partie-4-composants)
7. [Partie 5 : Styling avec Bootstrap](#7-partie-5-styling)
8. [Partie 6 : Gestion des erreurs](#8-partie-6-erreurs)
9. [Bonus : Dashboard et statistiques](#9-bonus-dashboard)

---

# 1. Introduction et architecture

## 1.1 Architecture globale

```
┌─────────────────────────────────────────────────────────┐
│                   Angular Frontend                       │
│                   (Port 4200)                           │
│                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │   Client     │  │   Account    │  │   Dashboard  │ │
│  │  Component   │  │  Component   │  │  Component   │ │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘ │
│         │                  │                  │         │
│  ┌──────▼──────────────────▼──────────────────▼──────┐ │
│  │              Services Layer                        │ │
│  │  (ClientService, AccountService, HttpClient)      │ │
│  └──────────────────────┬────────────────────────────┘ │
└─────────────────────────┼──────────────────────────────┘
                          │
                          │ HTTP REST Calls
                          ▼
                ┌─────────────────┐
                │   API Gateway   │
                │   Port 8080     │
                └────────┬────────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│   Client     │  │   Account    │  │    Order     │
│   Service    │  │   Service    │  │   Service    │
│   (8082)     │  │   (8081)     │  │   (8085)     │
└──────────────┘  └──────────────┘  └──────────────┘
```

## 1.2 Fonctionnalités à développer

### Module Clients
- ✅ Liste des clients
- ✅ Création d'un client
- ✅ Modification d'un client
- ✅ Suppression d'un client
- ✅ Recherche de clients

### Module Comptes
- ✅ Liste des comptes
- ✅ Création d'un compte
- ✅ Dépôt sur un compte
- ✅ Retrait sur un compte
- ✅ Consultation du solde

### Dashboard
- ✅ Statistiques
- ✅ Graphiques
- ✅ Vue d'ensemble

---

# 2. Installation et configuration

## 2.1 Prérequis

### Installer Node.js et npm

**Windows** :
- Télécharger : https://nodejs.org/ (version LTS)
- Installer et vérifier :

```bash
node --version  # v20.x.x ou plus
npm --version   # v10.x.x ou plus
```

**macOS** :
```bash
brew install node
```

**Linux (Ubuntu)** :
```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs
```

### Installer Angular CLI

```bash
npm install -g @angular/cli

# Vérifier l'installation
ng version
```

**Résultat attendu** :
```
Angular CLI: 17.x.x
Node: 20.x.x
Package Manager: npm 10.x.x
```

---

# 3. Partie 1 : Configuration du projet

## 3.1 Créer le projet Angular

```bash
# Créer le projet
ng new banque-frontend

# Répondre aux questions :
? Would you like to add Angular routing? (y/N) → y (OUI)
? Which stylesheet format would you like to use? → CSS

# Accéder au projet
cd banque-frontend
```

## 3.2 Structure du projet

```
banque-frontend/
├── src/
│   ├── app/
│   │   ├── components/        # Composants UI
│   │   │   ├── clients/
│   │   │   ├── comptes/
│   │   │   └── dashboard/
│   │   ├── models/            # Interfaces TypeScript
│   │   ├── services/          # Services HTTP
│   │   ├── app.component.ts
│   │   ├── app.component.html
│   │   └── app.routes.ts
│   ├── assets/                # Images, fichiers statiques
│   ├── environments/          # Configuration par environnement
│   └── index.html
├── angular.json
├── package.json
└── tsconfig.json
```

## 3.3 Installer les dépendances

```bash
# Bootstrap pour le design
npm install bootstrap bootstrap-icons

# Chart.js pour les graphiques
npm install chart.js ng2-charts

# RxJS (déjà inclus normalement)
npm install rxjs
```

## 3.4 Configurer Bootstrap

Modifier `angular.json` :

```json
{
  "projects": {
    "banque-frontend": {
      "architect": {
        "build": {
          "options": {
            "styles": [
              "node_modules/bootstrap/dist/css/bootstrap.min.css",
              "node_modules/bootstrap-icons/font/bootstrap-icons.css",
              "src/styles.css"
            ],
            "scripts": [
              "node_modules/bootstrap/dist/js/bootstrap.bundle.min.js"
            ]
          }
        }
      }
    }
  }
}
```

## 3.5 Configuration des environnements

Créer `src/environments/environment.ts` :

```typescript
export const environment = {
  production: false,
  apiUrl: 'http://localhost:8080/api', // Gateway URL
  clientServiceUrl: 'http://localhost:8082/api',
  accountServiceUrl: 'http://localhost:8081/api'
};
```

Créer `src/environments/environment.prod.ts` :

```typescript
export const environment = {
  production: true,
  apiUrl: 'https://api.votre-domaine.com/api',
  clientServiceUrl: 'https://api.votre-domaine.com/api',
  accountServiceUrl: 'https://api.votre-domaine.com/api'
};
```

## 3.6 Configurer CORS dans le Gateway

Dans `gateway-service/src/main/resources/application.yml` :

```yaml
spring:
  cloud:
    gateway:
      globalcors:
        cors-configurations:
          '[/**]':
            allowedOrigins: 
              - "http://localhost:4200"
            allowedMethods:
              - GET
              - POST
              - PUT
              - DELETE
              - OPTIONS
            allowedHeaders: "*"
            allowCredentials: true
            maxAge: 3600
```

---

# 4. Partie 2 : Service Client

## 4.1 Créer les modèles

Créer `src/app/models/client.model.ts` :

```typescript
export interface Client {
  id?: number;
  nom: string;
  prenom: string;
  email: string;
  telephone: string;
  adresse?: string;
}

export interface CreateClientRequest {
  nom: string;
  prenom: string;
  email: string;
  telephone: string;
  adresse?: string;
}

export interface UpdateClientRequest {
  nom: string;
  prenom: string;
  email: string;
  telephone: string;
  adresse?: string;
}
```

## 4.2 Créer le service HTTP

```bash
ng generate service services/client
```

Modifier `src/app/services/client.service.ts` :

```typescript
import { Injectable } from '@angular/core';
import { HttpClient, HttpErrorResponse } from '@angular/common/http';
import { Observable, throwError } from 'rxjs';
import { catchError, retry } from 'rxjs/operators';
import { environment } from '../../environments/environment';
import { Client, CreateClientRequest, UpdateClientRequest } from '../models/client.model';

@Injectable({
  providedIn: 'root'
})
export class ClientService {
  private apiUrl = `${environment.apiUrl}/clients`;

  constructor(private http: HttpClient) { }

  /**
   * Récupérer tous les clients
   */
  getAllClients(): Observable<Client[]> {
    return this.http.get<Client[]>(this.apiUrl)
      .pipe(
        retry(1),
        catchError(this.handleError)
      );
  }

  /**
   * Récupérer un client par ID
   */
  getClientById(id: number): Observable<Client> {
    return this.http.get<Client>(`${this.apiUrl}/${id}`)
      .pipe(
        retry(1),
        catchError(this.handleError)
      );
  }

  /**
   * Créer un nouveau client
   */
  createClient(client: CreateClientRequest): Observable<Client> {
    return this.http.post<Client>(this.apiUrl, client)
      .pipe(
        catchError(this.handleError)
      );
  }

  /**
   * Mettre à jour un client
   */
  updateClient(id: number, client: UpdateClientRequest): Observable<Client> {
    return this.http.put<Client>(`${this.apiUrl}/${id}`, client)
      .pipe(
        catchError(this.handleError)
      );
  }

  /**
   * Supprimer un client
   */
  deleteClient(id: number): Observable<void> {
    return this.http.delete<void>(`${this.apiUrl}/${id}`)
      .pipe(
        catchError(this.handleError)
      );
  }

  /**
   * Compter le nombre de clients
   */
  countClients(): Observable<number> {
    return this.http.get<number>(`${this.apiUrl}/count`)
      .pipe(
        retry(1),
        catchError(this.handleError)
      );
  }

  /**
   * Gestion des erreurs HTTP
   */
  private handleError(error: HttpErrorResponse) {
    let errorMessage = 'Une erreur est survenue';
    
    if (error.error instanceof ErrorEvent) {
      // Erreur côté client
      errorMessage = `Erreur: ${error.error.message}`;
    } else {
      // Erreur côté serveur
      if (error.status === 404) {
        errorMessage = 'Client non trouvé';
      } else if (error.status === 409) {
        errorMessage = 'Email déjà utilisé';
      } else if (error.status === 400) {
        errorMessage = 'Données invalides';
      } else {
        errorMessage = `Erreur ${error.status}: ${error.message}`;
      }
    }
    
    console.error(errorMessage);
    return throwError(() => new Error(errorMessage));
  }
}
```

## 4.3 Créer le composant liste clients

```bash
ng generate component components/clients/client-list
```

Modifier `src/app/components/clients/client-list/client-list.component.ts` :

```typescript
import { Component, OnInit } from '@angular/core';
import { CommonModule } from '@angular/common';
import { RouterModule } from '@angular/router';
import { ClientService } from '../../../services/client.service';
import { Client } from '../../../models/client.model';

@Component({
  selector: 'app-client-list',
  standalone: true,
  imports: [CommonModule, RouterModule],
  templateUrl: './client-list.component.html',
  styleUrls: ['./client-list.component.css']
})
export class ClientListComponent implements OnInit {
  clients: Client[] = [];
  loading: boolean = false;
  error: string = '';

  constructor(private clientService: ClientService) { }

  ngOnInit(): void {
    this.loadClients();
  }

  /**
   * Charger la liste des clients
   */
  loadClients(): void {
    this.loading = true;
    this.error = '';
    
    this.clientService.getAllClients().subscribe({
      next: (data) => {
        this.clients = data;
        this.loading = false;
      },
      error: (err) => {
        this.error = err.message;
        this.loading = false;
      }
    });
  }

  /**
   * Supprimer un client
   */
  deleteClient(id: number | undefined): void {
    if (!id) return;
    
    if (confirm('Êtes-vous sûr de vouloir supprimer ce client ?')) {
      this.clientService.deleteClient(id).subscribe({
        next: () => {
          this.loadClients(); // Recharger la liste
          alert('Client supprimé avec succès');
        },
        error: (err) => {
          alert('Erreur lors de la suppression: ' + err.message);
        }
      });
    }
  }
}
```

Modifier `src/app/components/clients/client-list/client-list.component.html` :

```html
<div class="container mt-4">
  <div class="d-flex justify-content-between align-items-center mb-4">
    <h2>
      <i class="bi bi-people-fill"></i> Liste des Clients
    </h2>
    <a routerLink="/clients/create" class="btn btn-primary">
      <i class="bi bi-plus-circle"></i> Nouveau Client
    </a>
  </div>

  <!-- Alerte de chargement -->
  <div *ngIf="loading" class="alert alert-info">
    <div class="spinner-border spinner-border-sm me-2" role="status"></div>
    Chargement des clients...
  </div>

  <!-- Alerte d'erreur -->
  <div *ngIf="error" class="alert alert-danger alert-dismissible fade show" role="alert">
    <i class="bi bi-exclamation-triangle-fill me-2"></i>
    {{ error }}
    <button type="button" class="btn-close" (click)="error = ''"></button>
  </div>

  <!-- Liste vide -->
  <div *ngIf="!loading && clients.length === 0 && !error" class="alert alert-warning">
    <i class="bi bi-info-circle me-2"></i>
    Aucun client enregistré. Cliquez sur "Nouveau Client" pour en ajouter un.
  </div>

  <!-- Tableau des clients -->
  <div *ngIf="!loading && clients.length > 0" class="card shadow-sm">
    <div class="card-body">
      <div class="table-responsive">
        <table class="table table-hover">
          <thead class="table-light">
            <tr>
              <th>ID</th>
              <th>Nom</th>
              <th>Prénom</th>
              <th>Email</th>
              <th>Téléphone</th>
              <th>Adresse</th>
              <th class="text-center">Actions</th>
            </tr>
          </thead>
          <tbody>
            <tr *ngFor="let client of clients">
              <td>{{ client.id }}</td>
              <td>{{ client.nom }}</td>
              <td>{{ client.prenom }}</td>
              <td>
                <i class="bi bi-envelope me-1"></i>
                {{ client.email }}
              </td>
              <td>
                <i class="bi bi-telephone me-1"></i>
                {{ client.telephone }}
              </td>
              <td>{{ client.adresse || 'N/A' }}</td>
              <td class="text-center">
                <div class="btn-group btn-group-sm" role="group">
                  <a [routerLink]="['/clients/edit', client.id]" 
                     class="btn btn-outline-primary"
                     title="Modifier">
                    <i class="bi bi-pencil"></i>
                  </a>
                  <button (click)="deleteClient(client.id)" 
                          class="btn btn-outline-danger"
                          title="Supprimer">
                    <i class="bi bi-trash"></i>
                  </button>
                </div>
              </td>
            </tr>
          </tbody>
        </table>
      </div>
      
      <div class="mt-3 text-muted">
        <i class="bi bi-info-circle me-1"></i>
        Total : {{ clients.length }} client(s)
      </div>
    </div>
  </div>
</div>
```

## 4.4 Créer le composant formulaire client

```bash
ng generate component components/clients/client-form
```

Modifier `src/app/components/clients/client-form/client-form.component.ts` :

```typescript
import { Component, OnInit } from '@angular/core';
import { CommonModule } from '@angular/common';
import { FormBuilder, FormGroup, Validators, ReactiveFormsModule } from '@angular/forms';
import { Router, ActivatedRoute } from '@angular/router';
import { ClientService } from '../../../services/client.service';
import { Client } from '../../../models/client.model';

@Component({
  selector: 'app-client-form',
  standalone: true,
  imports: [CommonModule, ReactiveFormsModule],
  templateUrl: './client-form.component.html',
  styleUrls: ['./client-form.component.css']
})
export class ClientFormComponent implements OnInit {
  clientForm!: FormGroup;
  isEditMode: boolean = false;
  clientId: number | null = null;
  loading: boolean = false;
  error: string = '';
  submitted: boolean = false;

  constructor(
    private fb: FormBuilder,
    private clientService: ClientService,
    private router: Router,
    private route: ActivatedRoute
  ) {
    this.initForm();
  }

  ngOnInit(): void {
    // Vérifier si mode édition
    this.route.params.subscribe(params => {
      if (params['id']) {
        this.isEditMode = true;
        this.clientId = +params['id'];
        this.loadClient(this.clientId);
      }
    });
  }

  /**
   * Initialiser le formulaire
   */
  initForm(): void {
    this.clientForm = this.fb.group({
      nom: ['', [Validators.required, Validators.minLength(2), Validators.maxLength(50)]],
      prenom: ['', [Validators.required, Validators.minLength(2), Validators.maxLength(50)]],
      email: ['', [Validators.required, Validators.email]],
      telephone: ['', [Validators.required, Validators.pattern('^0[1-9][0-9]{8}$')]],
      adresse: ['', [Validators.maxLength(200)]]
    });
  }

  /**
   * Charger les données du client (mode édition)
   */
  loadClient(id: number): void {
    this.loading = true;
    
    this.clientService.getClientById(id).subscribe({
      next: (client) => {
        this.clientForm.patchValue(client);
        this.loading = false;
      },
      error: (err) => {
        this.error = err.message;
        this.loading = false;
      }
    });
  }

  /**
   * Raccourcis pour accéder aux contrôles du formulaire
   */
  get f() { 
    return this.clientForm.controls; 
  }

  /**
   * Soumettre le formulaire
   */
  onSubmit(): void {
    this.submitted = true;
    this.error = '';

    // Arrêter si le formulaire est invalide
    if (this.clientForm.invalid) {
      return;
    }

    this.loading = true;

    if (this.isEditMode && this.clientId) {
      // Mode édition
      this.clientService.updateClient(this.clientId, this.clientForm.value).subscribe({
        next: () => {
          alert('Client modifié avec succès');
          this.router.navigate(['/clients']);
        },
        error: (err) => {
          this.error = err.message;
          this.loading = false;
        }
      });
    } else {
      // Mode création
      this.clientService.createClient(this.clientForm.value).subscribe({
        next: () => {
          alert('Client créé avec succès');
          this.router.navigate(['/clients']);
        },
        error: (err) => {
          this.error = err.message;
          this.loading = false;
        }
      });
    }
  }

  /**
   * Annuler et retourner à la liste
   */
  onCancel(): void {
    this.router.navigate(['/clients']);
  }
}
```

Modifier `src/app/components/clients/client-form/client-form.component.html` :

```html
<div class="container mt-4">
  <div class="row justify-content-center">
    <div class="col-md-8">
      <div class="card shadow-sm">
        <div class="card-header bg-primary text-white">
          <h4 class="mb-0">
            <i class="bi bi-person-plus-fill me-2"></i>
            {{ isEditMode ? 'Modifier le Client' : 'Nouveau Client' }}
          </h4>
        </div>
        
        <div class="card-body">
          <!-- Alerte d'erreur -->
          <div *ngIf="error" class="alert alert-danger alert-dismissible fade show" role="alert">
            <i class="bi bi-exclamation-triangle-fill me-2"></i>
            {{ error }}
            <button type="button" class="btn-close" (click)="error = ''"></button>
          </div>

          <!-- Formulaire -->
          <form [formGroup]="clientForm" (ngSubmit)="onSubmit()">
            <!-- Nom -->
            <div class="mb-3">
              <label for="nom" class="form-label">
                Nom <span class="text-danger">*</span>
              </label>
              <input 
                type="text" 
                class="form-control" 
                id="nom" 
                formControlName="nom"
                [class.is-invalid]="submitted && f['nom'].errors"
                placeholder="Entrez le nom">
              <div *ngIf="submitted && f['nom'].errors" class="invalid-feedback">
                <div *ngIf="f['nom'].errors['required']">Le nom est obligatoire</div>
                <div *ngIf="f['nom'].errors['minlength']">Le nom doit contenir au moins 2 caractères</div>
                <div *ngIf="f['nom'].errors['maxlength']">Le nom ne peut pas dépasser 50 caractères</div>
              </div>
            </div>

            <!-- Prénom -->
            <div class="mb-3">
              <label for="prenom" class="form-label">
                Prénom <span class="text-danger">*</span>
              </label>
              <input 
                type="text" 
                class="form-control" 
                id="prenom" 
                formControlName="prenom"
                [class.is-invalid]="submitted && f['prenom'].errors"
                placeholder="Entrez le prénom">
              <div *ngIf="submitted && f['prenom'].errors" class="invalid-feedback">
                <div *ngIf="f['prenom'].errors['required']">Le prénom est obligatoire</div>
                <div *ngIf="f['prenom'].errors['minlength']">Le prénom doit contenir au moins 2 caractères</div>
                <div *ngIf="f['prenom'].errors['maxlength']">Le prénom ne peut pas dépasser 50 caractères</div>
              </div>
            </div>

            <!-- Email -->
            <div class="mb-3">
              <label for="email" class="form-label">
                Email <span class="text-danger">*</span>
              </label>
              <div class="input-group">
                <span class="input-group-text">
                  <i class="bi bi-envelope"></i>
                </span>
                <input 
                  type="email" 
                  class="form-control" 
                  id="email" 
                  formControlName="email"
                  [class.is-invalid]="submitted && f['email'].errors"
                  placeholder="exemple@email.com">
                <div *ngIf="submitted && f['email'].errors" class="invalid-feedback">
                  <div *ngIf="f['email'].errors['required']">L'email est obligatoire</div>
                  <div *ngIf="f['email'].errors['email']">L'email doit être valide</div>
                </div>
              </div>
            </div>

            <!-- Téléphone -->
            <div class="mb-3">
              <label for="telephone" class="form-label">
                Téléphone <span class="text-danger">*</span>
              </label>
              <div class="input-group">
                <span class="input-group-text">
                  <i class="bi bi-telephone"></i>
                </span>
                <input 
                  type="tel" 
                  class="form-control" 
                  id="telephone" 
                  formControlName="telephone"
                  [class.is-invalid]="submitted && f['telephone'].errors"
                  placeholder="0771234567">
                <div *ngIf="submitted && f['telephone'].errors" class="invalid-feedback">
                  <div *ngIf="f['telephone'].errors['required']">Le téléphone est obligatoire</div>
                  <div *ngIf="f['telephone'].errors['pattern']">
                    Le téléphone doit être au format sénégalais (10 chiffres commençant par 0)
                  </div>
                </div>
              </div>
              <small class="text-muted">Format : 0771234567</small>
            </div>

            <!-- Adresse -->
            <div class="mb-3">
              <label for="adresse" class="form-label">Adresse</label>
              <textarea 
                class="form-control" 
                id="adresse" 
                formControlName="adresse"
                rows="3"
                [class.is-invalid]="submitted && f['adresse'].errors"
                placeholder="Entrez l'adresse (optionnel)"></textarea>
              <div *ngIf="submitted && f['adresse'].errors" class="invalid-feedback">
                <div *ngIf="f['adresse'].errors['maxlength']">
                  L'adresse ne peut pas dépasser 200 caractères
                </div>
              </div>
            </div>

            <!-- Boutons -->
            <div class="d-flex justify-content-between">
              <button type="button" class="btn btn-secondary" (click)="onCancel()" [disabled]="loading">
                <i class="bi bi-x-circle me-1"></i>
                Annuler
              </button>
              
              <button type="submit" class="btn btn-primary" [disabled]="loading">
                <span *ngIf="loading" class="spinner-border spinner-border-sm me-2"></span>
                <i *ngIf="!loading" class="bi bi-check-circle me-1"></i>
                {{ isEditMode ? 'Modifier' : 'Créer' }}
              </button>
            </div>
          </form>
        </div>
      </div>
    </div>
  </div>
</div>
```

---

# 5. Partie 3 : Service Compte

## 5.1 Créer les modèles

Créer `src/app/models/compte.model.ts` :

```typescript
export interface Compte {
  id?: number;
  numeroCompte: string;
  solde: number;
  typeCompte: 'COURANT' | 'EPARGNE';
  clientId: number;
}

export interface CreateCompteRequest {
  numeroCompte: string;
  soldeInitial: number;
  typeCompte: 'COURANT' | 'EPARGNE';
  clientId: number;
}

export interface CompteWithClient {
  compte: Compte;
  client: {
    id: number;
    nom: string;
    prenom: string;
    email: string;
    telephone: string;
    adresse?: string;
  };
}
```

## 5.2 Créer le service

```bash
ng generate service services/compte
```

Modifier `src/app/services/compte.service.ts` :

```typescript
import { Injectable } from '@angular/core';
import { HttpClient, HttpErrorResponse } from '@angular/common/http';
import { Observable, throwError } from 'rxjs';
import { catchError, retry } from 'rxjs/operators';
import { environment } from '../../environments/environment';
import { Compte, CreateCompteRequest, CompteWithClient } from '../models/compte.model';

@Injectable({
  providedIn: 'root'
})
export class CompteService {
  private apiUrl = `${environment.apiUrl}/comptes`;

  constructor(private http: HttpClient) { }

  /**
   * Récupérer tous les comptes
   */
  getAllComptes(): Observable<Compte[]> {
    return this.http.get<Compte[]>(this.apiUrl)
      .pipe(
        retry(1),
        catchError(this.handleError)
      );
  }

  /**
   * Récupérer un compte par ID
   */
  getCompteById(id: number): Observable<Compte> {
    return this.http.get<Compte>(`${this.apiUrl}/${id}`)
      .pipe(
        retry(1),
        catchError(this.handleError)
      );
  }

  /**
   * Récupérer un compte avec les infos du client (via Feign)
   */
  getCompteWithClient(id: number): Observable<CompteWithClient> {
    return this.http.get<CompteWithClient>(`${this.apiUrl}/${id}/with-client`)
      .pipe(
        retry(1),
        catchError(this.handleError)
      );
  }

  /**
   * Créer un nouveau compte
   */
  createCompte(compte: CreateCompteRequest): Observable<Compte> {
    return this.http.post<Compte>(this.apiUrl, compte)
      .pipe(
        catchError(this.handleError)
      );
  }

  /**
   * Effectuer un dépôt
   */
  deposer(id: number, montant: number): Observable<Compte> {
    return this.http.post<Compte>(`${this.apiUrl}/${id}/depot`, null, {
      params: { montant: montant.toString() }
    }).pipe(
      catchError(this.handleError)
    );
  }

  /**
   * Effectuer un retrait
   */
  retirer(id: number, montant: number): Observable<Compte> {
    return this.http.post<Compte>(`${this.apiUrl}/${id}/retrait`, null, {
      params: { montant: montant.toString() }
    }).pipe(
      catchError(this.handleError)
    );
  }

  /**
   * Gestion des erreurs HTTP
   */
  private handleError(error: HttpErrorResponse) {
    let errorMessage = 'Une erreur est survenue';
    
    if (error.error instanceof ErrorEvent) {
      errorMessage = `Erreur: ${error.error.message}`;
    } else {
      if (error.status === 404) {
        errorMessage = 'Compte non trouvé';
      } else if (error.status === 400) {
        errorMessage = error.error?.message || 'Données invalides';
      } else {
        errorMessage = `Erreur ${error.status}: ${error.message}`;
      }
    }
    
    console.error(errorMessage);
    return throwError(() => new Error(errorMessage));
  }
}
```

## 5.3 Créer le composant liste comptes

```bash
ng generate component components/comptes/compte-list
```

Modifier `src/app/components/comptes/compte-list/compte-list.component.ts` :

```typescript
import { Component, OnInit } from '@angular/core';
import { CommonModule } from '@angular/common';
import { RouterModule } from '@angular/router';
import { FormsModule } from '@angular/forms';
import { CompteService } from '../../../services/compte.service';
import { Compte } from '../../../models/compte.model';

@Component({
  selector: 'app-compte-list',
  standalone: true,
  imports: [CommonModule, RouterModule, FormsModule],
  templateUrl: './compte-list.component.html',
  styleUrls: ['./compte-list.component.css']
})
export class CompteListComponent implements OnInit {
  comptes: Compte[] = [];
  loading: boolean = false;
  error: string = '';
  
  // Pour les opérations
  selectedCompteId: number | null = null;
  montant: number = 0;
  operationType: 'depot' | 'retrait' | null = null;

  constructor(private compteService: CompteService) { }

  ngOnInit(): void {
    this.loadComptes();
  }

  /**
   * Charger la liste des comptes
   */
  loadComptes(): void {
    this.loading = true;
    this.error = '';
    
    this.compteService.getAllComptes().subscribe({
      next: (data) => {
        this.comptes = data;
        this.loading = false;
      },
      error: (err) => {
        this.error = err.message;
        this.loading = false;
      }
    });
  }

  /**
   * Ouvrir le modal d'opération
   */
  openOperationModal(compteId: number | undefined, type: 'depot' | 'retrait'): void {
    if (!compteId) return;
    
    this.selectedCompteId = compteId;
    this.operationType = type;
    this.montant = 0;
  }

  /**
   * Effectuer l'opération (dépôt ou retrait)
   */
  effectuerOperation(): void {
    if (!this.selectedCompteId || !this.operationType || this.montant <= 0) {
      alert('Veuillez entrer un montant valide');
      return;
    }

    this.loading = true;

    if (this.operationType === 'depot') {
      this.compteService.deposer(this.selectedCompteId, this.montant).subscribe({
        next: () => {
          alert(`Dépôt de ${this.montant} FCFA effectué avec succès`);
          this.loadComptes();
          this.closeModal();
        },
        error: (err) => {
          alert('Erreur: ' + err.message);
          this.loading = false;
        }
      });
    } else {
      this.compteService.retirer(this.selectedCompteId, this.montant).subscribe({
        next: () => {
          alert(`Retrait de ${this.montant} FCFA effectué avec succès`);
          this.loadComptes();
          this.closeModal();
        },
        error: (err) => {
          alert('Erreur: ' + err.message);
          this.loading = false;
        }
      });
    }
  }

  /**
   * Fermer le modal
   */
  closeModal(): void {
    this.selectedCompteId = null;
    this.operationType = null;
    this.montant = 0;
    this.loading = false;
  }

  /**
   * Formater le solde
   */
  formatSolde(solde: number): string {
    return solde.toLocaleString('fr-FR') + ' FCFA';
  }
}
```

Modifier `src/app/components/comptes/compte-list/compte-list.component.html` :

```html
<div class="container mt-4">
  <div class="d-flex justify-content-between align-items-center mb-4">
    <h2>
      <i class="bi bi-wallet2"></i> Liste des Comptes
    </h2>
    <a routerLink="/comptes/create" class="btn btn-primary">
      <i class="bi bi-plus-circle"></i> Nouveau Compte
    </a>
  </div>

  <!-- Alerte de chargement -->
  <div *ngIf="loading && comptes.length === 0" class="alert alert-info">
    <div class="spinner-border spinner-border-sm me-2" role="status"></div>
    Chargement des comptes...
  </div>

  <!-- Alerte d'erreur -->
  <div *ngIf="error" class="alert alert-danger alert-dismissible fade show" role="alert">
    <i class="bi bi-exclamation-triangle-fill me-2"></i>
    {{ error }}
    <button type="button" class="btn-close" (click)="error = ''"></button>
  </div>

  <!-- Liste vide -->
  <div *ngIf="!loading && comptes.length === 0 && !error" class="alert alert-warning">
    <i class="bi bi-info-circle me-2"></i>
    Aucun compte enregistré. Cliquez sur "Nouveau Compte" pour en créer un.
  </div>

  <!-- Grille des comptes -->
  <div *ngIf="comptes.length > 0" class="row">
    <div *ngFor="let compte of comptes" class="col-md-6 col-lg-4 mb-4">
      <div class="card shadow-sm h-100">
        <div class="card-header" 
             [class.bg-primary]="compte.typeCompte === 'COURANT'"
             [class.bg-success]="compte.typeCompte === 'EPARGNE'"
             class="text-white">
          <h5 class="mb-0">
            <i class="bi bi-credit-card me-2"></i>
            {{ compte.numeroCompte }}
          </h5>
        </div>
        
        <div class="card-body">
          <div class="mb-3">
            <small class="text-muted d-block">Type de compte</small>
            <span class="badge" 
                  [class.bg-primary]="compte.typeCompte === 'COURANT'"
                  [class.bg-success]="compte.typeCompte === 'EPARGNE'">
              {{ compte.typeCompte }}
            </span>
          </div>

          <div class="mb-3">
            <small class="text-muted d-block">ID Client</small>
            <strong># {{ compte.clientId }}</strong>
          </div>

          <div class="mb-3">
            <small class="text-muted d-block">Solde actuel</small>
            <h4 class="mb-0" 
                [class.text-success]="compte.solde >= 0"
                [class.text-danger]="compte.solde < 0">
              {{ formatSolde(compte.solde) }}
            </h4>
          </div>

          <div class="d-grid gap-2">
            <button class="btn btn-success btn-sm" 
                    (click)="openOperationModal(compte.id, 'depot')"
                    data-bs-toggle="modal" 
                    data-bs-target="#operationModal">
              <i class="bi bi-plus-circle"></i> Dépôt
            </button>
            
            <button class="btn btn-warning btn-sm" 
                    (click)="openOperationModal(compte.id, 'retrait')"
                    data-bs-toggle="modal" 
                    data-bs-target="#operationModal">
              <i class="bi bi-dash-circle"></i> Retrait
            </button>
          </div>
        </div>
      </div>
    </div>
  </div>

  <div *ngIf="comptes.length > 0" class="mt-3 text-muted">
    <i class="bi bi-info-circle me-1"></i>
    Total : {{ comptes.length }} compte(s)
  </div>
</div>

<!-- Modal pour dépôt/retrait -->
<div class="modal fade" id="operationModal" tabindex="-1">
  <div class="modal-dialog">
    <div class="modal-content">
      <div class="modal-header" 
           [class.bg-success]="operationType === 'depot'"
           [class.bg-warning]="operationType === 'retrait'"
           class="text-white">
        <h5 class="modal-title">
          <i class="bi" 
             [class.bi-plus-circle]="operationType === 'depot'"
             [class.bi-dash-circle]="operationType === 'retrait'"></i>
          {{ operationType === 'depot' ? 'Effectuer un Dépôt' : 'Effectuer un Retrait' }}
        </h5>
        <button type="button" class="btn-close btn-close-white" data-bs-dismiss="modal"></button>
      </div>
      
      <div class="modal-body">
        <div class="mb-3">
          <label for="montant" class="form-label">Montant (FCFA)</label>
          <input type="number" 
                 class="form-control form-control-lg" 
                 id="montant" 
                 [(ngModel)]="montant"
                 [min]="0"
                 placeholder="Entrez le montant">
        </div>
      </div>
      
      <div class="modal-footer">
        <button type="button" class="btn btn-secondary" data-bs-dismiss="modal">
          Annuler
        </button>
        <button type="button" 
                class="btn"
                [class.btn-success]="operationType === 'depot'"
                [class.btn-warning]="operationType === 'retrait'"
                (click)="effectuerOperation()"
                [disabled]="montant <= 0 || loading"
                data-bs-dismiss="modal">
          <span *ngIf="loading" class="spinner-border spinner-border-sm me-2"></span>
          Valider
        </button>
      </div>
    </div>
  </div>
</div>
```

## 5.4 Créer le composant formulaire compte

```bash
ng generate component components/comptes/compte-form
```

Modifier `src/app/components/comptes/compte-form/compte-form.component.ts` :

```typescript
import { Component, OnInit } from '@angular/core';
import { CommonModule } from '@angular/common';
import { FormBuilder, FormGroup, Validators, ReactiveFormsModule } from '@angular/forms';
import { Router } from '@angular/router';
import { CompteService } from '../../../services/compte.service';
import { ClientService } from '../../../services/client.service';
import { Client } from '../../../models/client.model';

@Component({
  selector: 'app-compte-form',
  standalone: true,
  imports: [CommonModule, ReactiveFormsModule],
  templateUrl: './compte-form.component.html',
  styleUrls: ['./compte-form.component.css']
})
export class CompteFormComponent implements OnInit {
  compteForm!: FormGroup;
  clients: Client[] = [];
  loading: boolean = false;
  error: string = '';
  submitted: boolean = false;

  constructor(
    private fb: FormBuilder,
    private compteService: CompteService,
    private clientService: ClientService,
    private router: Router
  ) {
    this.initForm();
  }

  ngOnInit(): void {
    this.loadClients();
  }

  /**
   * Initialiser le formulaire
   */
  initForm(): void {
    this.compteForm = this.fb.group({
      numeroCompte: ['', [Validators.required, Validators.pattern('^[A-Z0-9]{6,12}$')]],
      soldeInitial: [0, [Validators.required, Validators.min(0)]],
      typeCompte: ['COURANT', Validators.required],
      clientId: ['', Validators.required]
    });
  }

  /**
   * Charger la liste des clients
   */
  loadClients(): void {
    this.clientService.getAllClients().subscribe({
      next: (data) => {
        this.clients = data;
      },
      error: (err) => {
        this.error = 'Impossible de charger la liste des clients';
      }
    });
  }

  /**
   * Raccourcis pour accéder aux contrôles du formulaire
   */
  get f() { 
    return this.compteForm.controls; 
  }

  /**
   * Soumettre le formulaire
   */
  onSubmit(): void {
    this.submitted = true;
    this.error = '';

    if (this.compteForm.invalid) {
      return;
    }

    this.loading = true;

    this.compteService.createCompte(this.compteForm.value).subscribe({
      next: () => {
        alert('Compte créé avec succès');
        this.router.navigate(['/comptes']);
      },
      error: (err) => {
        this.error = err.message;
        this.loading = false;
      }
    });
  }

  /**
   * Annuler et retourner à la liste
   */
  onCancel(): void {
    this.router.navigate(['/comptes']);
  }
}
```

Modifier `src/app/components/comptes/compte-form/compte-form.component.html` :

```html
<div class="container mt-4">
  <div class="row justify-content-center">
    <div class="col-md-8">
      <div class="card shadow-sm">
        <div class="card-header bg-primary text-white">
          <h4 class="mb-0">
            <i class="bi bi-wallet-fill me-2"></i>
            Nouveau Compte
          </h4>
        </div>
        
        <div class="card-body">
          <!-- Alerte d'erreur -->
          <div *ngIf="error" class="alert alert-danger alert-dismissible fade show" role="alert">
            <i class="bi bi-exclamation-triangle-fill me-2"></i>
            {{ error }}
            <button type="button" class="btn-close" (click)="error = ''"></button>
          </div>

          <!-- Formulaire -->
          <form [formGroup]="compteForm" (ngSubmit)="onSubmit()">
            <!-- Numéro de compte -->
            <div class="mb-3">
              <label for="numeroCompte" class="form-label">
                Numéro de Compte <span class="text-danger">*</span>
              </label>
              <input 
                type="text" 
                class="form-control" 
                id="numeroCompte" 
                formControlName="numeroCompte"
                [class.is-invalid]="submitted && f['numeroCompte'].errors"
                placeholder="Ex: ACC001"
                style="text-transform: uppercase;">
              <small class="text-muted">Format : 6-12 caractères alphanumériques (ex: ACC001)</small>
              <div *ngIf="submitted && f['numeroCompte'].errors" class="invalid-feedback">
                <div *ngIf="f['numeroCompte'].errors['required']">Le numéro de compte est obligatoire</div>
                <div *ngIf="f['numeroCompte'].errors['pattern']">
                  Format invalide (6-12 caractères alphanumériques)
                </div>
              </div>
            </div>

            <!-- Solde initial -->
            <div class="mb-3">
              <label for="soldeInitial" class="form-label">
                Solde Initial (FCFA) <span class="text-danger">*</span>
              </label>
              <input 
                type="number" 
                class="form-control" 
                id="soldeInitial" 
                formControlName="soldeInitial"
                [class.is-invalid]="submitted && f['soldeInitial'].errors"
                placeholder="0">
              <div *ngIf="submitted && f['soldeInitial'].errors" class="invalid-feedback">
                <div *ngIf="f['soldeInitial'].errors['required']">Le solde initial est obligatoire</div>
                <div *ngIf="f['soldeInitial'].errors['min']">Le solde doit être positif ou nul</div>
              </div>
            </div>

            <!-- Type de compte -->
            <div class="mb-3">
              <label for="typeCompte" class="form-label">
                Type de Compte <span class="text-danger">*</span>
              </label>
              <select 
                class="form-select" 
                id="typeCompte" 
                formControlName="typeCompte"
                [class.is-invalid]="submitted && f['typeCompte'].errors">
                <option value="COURANT">Compte Courant</option>
                <option value="EPARGNE">Compte Épargne</option>
              </select>
              <div *ngIf="submitted && f['typeCompte'].errors" class="invalid-feedback">
                Le type de compte est obligatoire
              </div>
            </div>

            <!-- Client -->
            <div class="mb-3">
              <label for="clientId" class="form-label">
                Client <span class="text-danger">*</span>
              </label>
              <select 
                class="form-select" 
                id="clientId" 
                formControlName="clientId"
                [class.is-invalid]="submitted && f['clientId'].errors">
                <option value="">Sélectionnez un client</option>
                <option *ngFor="let client of clients" [value]="client.id">
                  {{ client.prenom }} {{ client.nom }} ({{ client.email }})
                </option>
              </select>
              <div *ngIf="submitted && f['clientId'].errors" class="invalid-feedback">
                Veuillez sélectionner un client
              </div>
              <small class="text-muted">
                <a routerLink="/clients/create" class="text-decoration-none">
                  <i class="bi bi-plus-circle"></i> Créer un nouveau client
                </a>
              </small>
            </div>

            <!-- Boutons -->
            <div class="d-flex justify-content-between">
              <button type="button" class="btn btn-secondary" (click)="onCancel()" [disabled]="loading">
                <i class="bi bi-x-circle me-1"></i>
                Annuler
              </button>
              
              <button type="submit" class="btn btn-primary" [disabled]="loading">
                <span *ngIf="loading" class="spinner-border spinner-border-sm me-2"></span>
                <i *ngIf="!loading" class="bi bi-check-circle me-1"></i>
                Créer
              </button>
            </div>
          </form>
        </div>
      </div>
    </div>
  </div>
</div>
```

---

# 6. Partie 4 : Composants et routing

## 6.1 Créer le composant principal

Modifier `src/app/app.component.html` :

```html
<!-- Navigation -->
<nav class="navbar navbar-expand-lg navbar-dark bg-primary shadow">
  <div class="container-fluid">
    <a class="navbar-brand" routerLink="/">
      <i class="bi bi-bank2 me-2"></i>
      Banque App
    </a>
    
    <button class="navbar-toggler" type="button" data-bs-toggle="collapse" data-bs-target="#navbarNav">
      <span class="navbar-toggler-icon"></span>
    </button>
    
    <div class="collapse navbar-collapse" id="navbarNav">
      <ul class="navbar-nav me-auto">
        <li class="nav-item">
          <a class="nav-link" routerLink="/" routerLinkActive="active" [routerLinkActiveOptions]="{exact: true}">
            <i class="bi bi-house-door"></i> Accueil
          </a>
        </li>
        <li class="nav-item">
          <a class="nav-link" routerLink="/clients" routerLinkActive="active">
            <i class="bi bi-people"></i> Clients
          </a>
        </li>
        <li class="nav-item">
          <a class="nav-link" routerLink="/comptes" routerLinkActive="active">
            <i class="bi bi-wallet2"></i> Comptes
          </a>
        </li>
      </ul>
    </div>
  </div>
</nav>

<!-- Contenu principal -->
<main class="min-vh-100 bg-light">
  <router-outlet></router-outlet>
</main>

<!-- Footer -->
<footer class="bg-dark text-white text-center py-3">
  <div class="container">
    <p class="mb-0">
      © 2024 Banque App - Système de Gestion Bancaire
    </p>
  </div>
</footer>
```

## 6.2 Créer le composant Dashboard

```bash
ng generate component components/dashboard
```

Modifier `src/app/components/dashboard/dashboard.component.ts` :

```typescript
import { Component, OnInit } from '@angular/core';
import { CommonModule } from '@angular/common';
import { RouterModule } from '@angular/router';
import { ClientService } from '../../services/client.service';
import { CompteService } from '../../services/compte.service';
import { Client } from '../../models/client.model';
import { Compte } from '../../models/compte.model';

@Component({
  selector: 'app-dashboard',
  standalone: true,
  imports: [CommonModule, RouterModule],
  templateUrl: './dashboard.component.html',
  styleUrls: ['./dashboard.component.css']
})
export class DashboardComponent implements OnInit {
  totalClients: number = 0;
  totalComptes: number = 0;
  soldeTotal: number = 0;
  loading: boolean = true;
  
  recentClients: Client[] = [];
  recentComptes: Compte[] = [];

  constructor(
    private clientService: ClientService,
    private compteService: CompteService
  ) { }

  ngOnInit(): void {
    this.loadDashboardData();
  }

  /**
   * Charger les données du dashboard
   */
  loadDashboardData(): void {
    this.loading = true;

    // Charger les clients
    this.clientService.getAllClients().subscribe({
      next: (clients) => {
        this.totalClients = clients.length;
        this.recentClients = clients.slice(-5).reverse(); // 5 derniers clients
      }
    });

    // Charger les comptes
    this.compteService.getAllComptes().subscribe({
      next: (comptes) => {
        this.totalComptes = comptes.length;
        this.soldeTotal = comptes.reduce((sum, compte) => sum + compte.solde, 0);
        this.recentComptes = comptes.slice(-5).reverse(); // 5 derniers comptes
        this.loading = false;
      }
    });
  }

  /**
   * Formater le montant
   */
  formatMontant(montant: number): string {
    return montant.toLocaleString('fr-FR') + ' FCFA';
  }
}
```

Modifier `src/app/components/dashboard/dashboard.component.html` :

```html
<div class="container mt-4">
  <!-- En-tête -->
  <div class="mb-4">
    <h1 class="display-4">
      <i class="bi bi-speedometer2 me-3"></i>
      Tableau de Bord
    </h1>
    <p class="lead text-muted">Vue d'ensemble de votre banque</p>
  </div>

  <!-- Statistiques -->
  <div class="row mb-4">
    <!-- Total Clients -->
    <div class="col-md-4 mb-3">
      <div class="card border-primary shadow-sm h-100">
        <div class="card-body">
          <div class="d-flex justify-content-between align-items-center">
            <div>
              <p class="text-muted mb-1">Total Clients</p>
              <h2 class="mb-0">{{ totalClients }}</h2>
            </div>
            <div class="bg-primary bg-opacity-10 p-3 rounded">
              <i class="bi bi-people-fill text-primary fs-1"></i>
            </div>
          </div>
        </div>
        <div class="card-footer bg-transparent">
          <a routerLink="/clients" class="text-decoration-none">
            Voir tous les clients <i class="bi bi-arrow-right"></i>
          </a>
        </div>
      </div>
    </div>

    <!-- Total Comptes -->
    <div class="col-md-4 mb-3">
      <div class="card border-success shadow-sm h-100">
        <div class="card-body">
          <div class="d-flex justify-content-between align-items-center">
            <div>
              <p class="text-muted mb-1">Total Comptes</p>
              <h2 class="mb-0">{{ totalComptes }}</h2>
            </div>
            <div class="bg-success bg-opacity-10 p-3 rounded">
              <i class="bi bi-wallet2 text-success fs-1"></i>
            </div>
          </div>
        </div>
        <div class="card-footer bg-transparent">
          <a routerLink="/comptes" class="text-decoration-none">
            Voir tous les comptes <i class="bi bi-arrow-right"></i>
          </a>
        </div>
      </div>
    </div>

    <!-- Solde Total -->
    <div class="col-md-4 mb-3">
      <div class="card border-warning shadow-sm h-100">
        <div class="card-body">
          <div class="d-flex justify-content-between align-items-center">
            <div>
              <p class="text-muted mb-1">Solde Total</p>
              <h2 class="mb-0">{{ formatMontant(soldeTotal) }}</h2>
            </div>
            <div class="bg-warning bg-opacity-10 p-3 rounded">
              <i class="bi bi-cash-stack text-warning fs-1"></i>
            </div>
          </div>
        </div>
        <div class="card-footer bg-transparent">
          <span class="text-muted">Tous les comptes</span>
        </div>
      </div>
    </div>
  </div>

  <!-- Actions rapides -->
  <div class="row mb-4">
    <div class="col-12">
      <div class="card shadow-sm">
        <div class="card-header bg-light">
          <h5 class="mb-0">
            <i class="bi bi-lightning-fill me-2"></i>
            Actions Rapides
          </h5>
        </div>
        <div class="card-body">
          <div class="row g-3">
            <div class="col-md-3">
              <a routerLink="/clients/create" class="btn btn-outline-primary w-100">
                <i class="bi bi-person-plus fs-3 d-block mb-2"></i>
                Nouveau Client
              </a>
            </div>
            <div class="col-md-3">
              <a routerLink="/comptes/create" class="btn btn-outline-success w-100">
                <i class="bi bi-wallet-fill fs-3 d-block mb-2"></i>
                Nouveau Compte
              </a>
            </div>
            <div class="col-md-3">
              <a routerLink="/clients" class="btn btn-outline-info w-100">
                <i class="bi bi-search fs-3 d-block mb-2"></i>
                Rechercher Client
              </a>
            </div>
            <div class="col-md-3">
              <a routerLink="/comptes" class="btn btn-outline-warning w-100">
                <i class="bi bi-list-ul fs-3 d-block mb-2"></i>
                Liste Comptes
              </a>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>

  <!-- Derniers clients -->
  <div class="row">
    <div class="col-md-6 mb-4">
      <div class="card shadow-sm">
        <div class="card-header bg-primary text-white">
          <h5 class="mb-0">
            <i class="bi bi-clock-history me-2"></i>
            Derniers Clients
          </h5>
        </div>
        <div class="card-body p-0">
          <div *ngIf="recentClients.length === 0" class="p-3 text-center text-muted">
            Aucun client récent
          </div>
          <div class="list-group list-group-flush">
            <div *ngFor="let client of recentClients" class="list-group-item">
              <div class="d-flex justify-content-between align-items-center">
                <div>
                  <strong>{{ client.prenom }} {{ client.nom }}</strong>
                  <br>
                  <small class="text-muted">{{ client.email }}</small>
                </div>
                <a [routerLink]="['/clients/edit', client.id]" class="btn btn-sm btn-outline-primary">
                  <i class="bi bi-pencil"></i>
                </a>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- Derniers comptes -->
    <div class="col-md-6 mb-4">
      <div class="card shadow-sm">
        <div class="card-header bg-success text-white">
          <h5 class="mb-0">
            <i class="bi bi-clock-history me-2"></i>
            Derniers Comptes
          </h5>
        </div>
        <div class="card-body p-0">
          <div *ngIf="recentComptes.length === 0" class="p-3 text-center text-muted">
            Aucun compte récent
          </div>
          <div class="list-group list-group-flush">
            <div *ngFor="let compte of recentComptes" class="list-group-item">
              <div class="d-flex justify-content-between align-items-center">
                <div>
                  <strong>{{ compte.numeroCompte }}</strong>
                  <span class="badge ms-2"
                        [class.bg-primary]="compte.typeCompte === 'COURANT'"
                        [class.bg-success]="compte.typeCompte === 'EPARGNE'">
                    {{ compte.typeCompte }}
                  </span>
                  <br>
                  <small class="text-muted">Client #{{ compte.clientId }}</small>
                </div>
                <div class="text-end">
                  <strong [class.text-success]="compte.solde >= 0"
                          [class.text-danger]="compte.solde < 0">
                    {{ formatMontant(compte.solde) }}
                  </strong>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</div>
```

## 6.3 Configurer les routes

Modifier `src/app/app.routes.ts` :

```typescript
import { Routes } from '@angular/router';
import { DashboardComponent } from './components/dashboard/dashboard.component';
import { ClientListComponent } from './components/clients/client-list/client-list.component';
import { ClientFormComponent } from './components/clients/client-form/client-form.component';
import { CompteListComponent } from './components/comptes/compte-list/compte-list.component';
import { CompteFormComponent } from './components/comptes/compte-form/compte-form.component';

export const routes: Routes = [
  { path: '', component: DashboardComponent },
  { path: 'clients', component: ClientListComponent },
  { path: 'clients/create', component: ClientFormComponent },
  { path: 'clients/edit/:id', component: ClientFormComponent },
  { path: 'comptes', component: CompteListComponent },
  { path: 'comptes/create', component: CompteFormComponent },
  { path: '**', redirectTo: '' }
];
```

## 6.4 Configurer HttpClient

Modifier `src/app/app.config.ts` :

```typescript
import { ApplicationConfig, provideZoneChangeDetection } from '@angular/core';
import { provideRouter } from '@angular/router';
import { provideHttpClient, withInterceptorsFromDi } from '@angular/common/http';

import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [
    provideZoneChangeDetection({ eventCoalescing: true }),
    provideRouter(routes),
    provideHttpClient(withInterceptorsFromDi())
  ]
};
```

---

# 7. Partie 5 : Styling avec Bootstrap

## 7.1 Styles globaux

Modifier `src/styles.css` :

```css
/* Styles globaux */
body {
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  background-color: #f8f9fa;
}

/* Navigation */
.navbar-brand {
  font-weight: bold;
  font-size: 1.5rem;
}

/* Cards */
.card {
  border-radius: 10px;
  transition: transform 0.2s, box-shadow 0.2s;
}

.card:hover {
  transform: translateY(-5px);
  box-shadow: 0 10px 20px rgba(0, 0, 0, 0.1) !important;
}

/* Buttons */
.btn {
  border-radius: 5px;
  transition: all 0.3s;
}

.btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 5px 10px rgba(0, 0, 0, 0.2);
}

/* Tables */
.table {
  background: white;
}

.table-hover tbody tr:hover {
  background-color: #f1f3f5;
  cursor: pointer;
}

/* Forms */
.form-control:focus,
.form-select:focus {
  border-color: #0d6efd;
  box-shadow: 0 0 0 0.25rem rgba(13, 110, 253, 0.25);
}

/* Alerts */
.alert {
  border-radius: 10px;
  border: none;
}

/* Badges */
.badge {
  padding: 0.5rem 0.75rem;
  font-size: 0.85rem;
  border-radius: 5px;
}

/* Footer */
footer {
  margin-top: auto;
}

/* Loading spinner */
.spinner-border-sm {
  width: 1rem;
  height: 1rem;
}

/* Animations */
@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.fade-in {
  animation: fadeIn 0.5s ease-in;
}

/* Responsive */
@media (max-width: 768px) {
  .navbar-brand {
    font-size: 1.2rem;
  }
  
  .display-4 {
    font-size: 2rem;
  }
}
```

---

# 8. Partie 6 : Gestion des erreurs

## 8.1 Créer un intercepteur HTTP

```bash
ng generate interceptor interceptors/http-error
```

Modifier `src/app/interceptors/http-error.interceptor.ts` :

```typescript
import { HttpInterceptorFn, HttpErrorResponse } from '@angular/common/http';
import { inject } from '@angular/core';
import { catchError, throwError } from 'rxjs';
import { Router } from '@angular/router';

export const httpErrorInterceptor: HttpInterceptorFn = (req, next) => {
  const router = inject(Router);

  return next(req).pipe(
    catchError((error: HttpErrorResponse) => {
      let errorMessage = 'Une erreur est survenue';

      if (error.error instanceof ErrorEvent) {
        // Erreur côté client
        errorMessage = `Erreur: ${error.error.message}`;
      } else {
        // Erreur côté serveur
        switch (error.status) {
          case 0:
            errorMessage = 'Impossible de contacter le serveur. Vérifiez votre connexion.';
            break;
          case 400:
            errorMessage = error.error?.message || 'Données invalides';
            break;
          case 404:
            errorMessage = 'Ressource non trouvée';
            break;
          case 409:
            errorMessage = error.error?.message || 'Conflit de données';
            break;
          case 500:
            errorMessage = 'Erreur interne du serveur';
            break;
          default:
            errorMessage = `Erreur ${error.status}: ${error.message}`;
        }
      }

      console.error('Erreur HTTP:', errorMessage, error);
      
      // Afficher une notification (à implémenter avec un service)
      // this.notificationService.error(errorMessage);

      return throwError(() => new Error(errorMessage));
    })
  );
};
```

Modifier `src/app/app.config.ts` pour enregistrer l'intercepteur :

```typescript
import { ApplicationConfig, provideZoneChangeDetection } from '@angular/core';
import { provideRouter } from '@angular/router';
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { httpErrorInterceptor } from './interceptors/http-error.interceptor';

import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [
    provideZoneChangeDetection({ eventCoalescing: true }),
    provideRouter(routes),
    provideHttpClient(
      withInterceptors([httpErrorInterceptor])
    )
  ]
};
```

---

# 9. Bonus : Dashboard et statistiques

*(Déjà implémenté dans la Partie 4.2)*

---

# 10. Tests et démarrage

## 10.1 Démarrer l'application

```bash
# Dans le terminal, à la racine du projet Angular
ng serve

# Ou avec un port spécifique
ng serve --port 4200 --open
```

**L'application sera accessible sur** : http://localhost:4200

## 10.2 Ordre de démarrage complet

```bash
# Terminal 1 : Eureka Server
cd eureka-server
mvn spring-boot:run

# Terminal 2 : Client Service
cd client-service
mvn spring-boot:run

# Terminal 3 : Account Service
cd account-service
mvn spring-boot:run

# Terminal 4 : Gateway
cd gateway-service
mvn spring-boot:run

# Terminal 5 : Angular Frontend
cd banque-frontend
ng serve
```

## 10.3 Vérifications

1. **Eureka** : http://localhost:8761
   - ✅ 3 services enregistrés

2. **Gateway** : http://localhost:8080
   - ✅ Routes fonctionnelles

3. **Angular** : http://localhost:4200
   - ✅ Interface chargée
   - ✅ Dashboard visible

## 10.4 Scénario de test complet

```
1. Ouvrir http://localhost:4200
2. Cliquer sur "Nouveau Client"
3. Remplir le formulaire :
   - Nom : Diop
   - Prénom : Amadou
   - Email : amadou.diop@email.sn
   - Téléphone : 0771234567
   - Adresse : Dakar, Sénégal
4. Cliquer sur "Créer"
5. Vérifier la liste des clients
6. Cliquer sur "Nouveau Compte"
7. Remplir le formulaire :
   - Numéro : ACC001
   - Solde : 100000
   - Type : Compte Courant
   - Client : Amadou Diop
8. Cliquer sur "Créer"
9. Vérifier la liste des comptes
10. Cliquer sur "Dépôt" sur le compte
11. Entrer 50000 FCFA
12. Valider
13. Vérifier que le solde est mis à jour (150000 FCFA)
14. Retourner au Dashboard
15. Vérifier les statistiques
```

✅ **Si tout fonctionne, votre application est complète !**

---

# 11. Build pour la production

## 11.1 Compiler l'application

```bash
# Build production
ng build --configuration=production

# Les fichiers sont générés dans dist/banque-frontend/
```

## 11.2 Déployer sur un serveur web

```bash
# Installer un serveur HTTP simple
npm install -g http-server

# Servir les fichiers compilés
cd dist/banque-frontend
http-server -p 8080

# Ou avec nginx
sudo cp -r dist/banque-frontend/* /var/www/html/
```

---

# 12. Exercices et évaluation

## Exercice 1 : Recherche de clients (5 points)

**Énoncé** : Ajouter une barre de recherche dans la liste des clients pour filtrer par nom/prénom/email.

**Critères** :
- Input de recherche (1 pt)
- Filtrage en temps réel (2 pts)
- Affichage du nombre de résultats (1 pt)
- Reset du filtre (1 pt)

## Exercice 2 : Pagination (5 points)

**Énoncé** : Ajouter une pagination à la liste des comptes (10 éléments par page).

**Critères** :
- Boutons précédent/suivant (2 pts)
- Numéros de pages (2 pts)
- Affichage "Page X sur Y" (1 pt)

## Exercice 3 : Graphiques (10 points)

**Énoncé** : Créer une page de statistiques avec des graphiques Chart.js :
- Répartition des types de comptes (camembert)
- Évolution des soldes (courbe)
- Top 5 clients par solde total (barres)

**Bonus** : Export PDF des statistiques (+3 pts)

## Mini-projet : Module Transactions (20 points)

**Énoncé** : Créer un module complet de gestion des transactions :

1. **Backend** (10 pts)
   - Créer Transaction Service
   - Entité Transaction (id, compteId, type, montant, date)
   - CRUD complet
   - Publier événement dans Kafka

2. **Frontend** (10 pts)
   - Liste des transactions
   - Filtrage par compte/date
   - Historique avec timeline
   - Export CSV

---

# 13. Ressources

## Documentation
- **Angular** : https://angular.io/docs
- **Bootstrap** : https://getbootstrap.com/docs/5.3/
- **TypeScript** : https://www.typescriptlang.org/docs/
- **RxJS** : https://rxjs.dev/guide/overview

## Tutoriels
- Angular CLI : https://angular.io/cli
- Reactive Forms : https://angular.io/guide/reactive-forms
- HTTP Client : https://angular.io/guide/understanding-communicating-with-http

---

**Fin du TP - Félicitations ! 🎉**

Vous avez maintenant une application Angular complète connectée à vos microservices !
