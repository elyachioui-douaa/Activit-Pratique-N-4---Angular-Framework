# Compte Rendu  de l'Activité pratique N°4  : Application Full-Stack avec Angular et Spring Boot

Ce rapport présente le TP 4, qui consistait à développer une application web full-stack avec Angular et Spring Boot, permettant de gérer des produits via une interface utilisateur réactive et une API REST.

## 1. Architecture de l'Application

L’application suit une architecture client-serveur moderne :

Backend (Spring Boot) : API REST gérant la logique métier, l’accès aux données et les opérations CRUD.

Frontend (Angular) : Application monopage (SPA) consommant l’API pour afficher les données et gérer les interactions utilisateur.

Cette séparation permet un développement et un déploiement indépendants.

## 2. Technologies Utilisées

**Backend (`backend-app`)**  
- Spring Boot, Spring Data JPA, Spring Web  
- Base de données H2 en mémoire  
- Lombok pour réduire le code boilerplate  
- Maven pour la compilation et les dépendances  

**Frontend (`angular-app`)**  
- Angular, HttpClientModule  
- Bootstrap et Bootstrap Icons pour le style  
- npm pour la gestion des dépendances  

## 3. Détails de l'Implémentation

### Partie Backend (Spring Boot)

- **Couche persistance (JPA)** : `Product` avec annotations JPA et validation.
- **Repository** : `ProductRepository` étend `JpaRepository` pour les CRUD.
- **Contrôleur REST** : `ProductRestAPI` expose les endpoints et gère le CORS (`@CrossOrigin("*")`).


```java
@Entity
@NoArgsConstructor @AllArgsConstructor @Getter @Setter
public class Product {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    @NotEmpty
    private String name;
    @Min(0)
    private double price;
    private Boolean selected;
}
```

**Repository `ProductRepository.java`**
Une interface simple qui étend `JpaRepository` pour fournir toutes les opérations CRUD standard sans code supplémentaire.

```java
public interface ProductRepository extends JpaRepository<Product,Long> {
}
```

#### Couche Web (API REST)

**Contrôleur `ProductRestAPI.java`**
Ce contrôleur fournit les endpoints REST permettant de gérer les produits. Il est configuré pour autoriser les requêtes provenant de n’importe quelle origine (@CrossOrigin("*")), ce qui assure une communication fluide avec l’application Angular.

```java
@RestController
@CrossOrigin("*")
@RequestMapping("/api")
@AllArgsConstructor
public class ProductRestAPI {
    private ProductService productService;

    @GetMapping("/products")
    public List<Product> getAllProducts(){
        return productService.findAll();
    }

    @DeleteMapping("/products/{id}")
    public void deleteProductById(@PathVariable(name = "id") Long id){
        productService.deleteById(id);
    }
    // ... autres endpoints
}
```

#### Initialisation des Données

La classe `BackendAppApplication.java` utilise un `CommandLineRunner` pour remplir automatiquement la base H2 avec des produits de test dès le démarrage de l’application.

```java
@Bean
CommandLineRunner init(ProductService productService) {
    return args -> {
        productService.addProduct(new Product(null, "ordinateur", 5000, true ));
        productService.addProduct(new Product(null, "telephone", 2500, true ));
        productService.addProduct(new Product(null, "tablette", 3000, false ));
        // ...
    };
}
```

### Partie Frontend (Angular)

Le frontend est organisé en composants et services, ce qui permet de séparer clairement la logique métier de l’interface utilisateur.

#### Service de données

**`ProductService.ts`**  
Ce service gère la communication avec l’API backend. Il utilise le `HttpClient` d’Angular pour envoyer et recevoir les requêtes HTTP.

```typescript
@Injectable({
  providedIn: 'root',
})
export class ProductService {
  constructor(private http: HttpClient) {}

  getAllProducts() {
      return this.http.get("http://localhost:8080/api/products");
  }

  deleteProduct(p: any) {
      return this.http.delete("http://localhost:8080/api/products/" + p.id);
  }
}
```

#### Composant d'affichage

**`products.ts`**  
Le composant `Products` utilise le `ProductService` pour récupérer et afficher la liste des produits. Il prend également en charge la suppression des produits directement depuis l’interface.

```typescript
@Component({ /* ... */ })
export class Products implements OnInit {
  products: any = [] ;

  constructor(
    private productService:ProductService,
    private cd: ChangeDetectorRef,
  ) {}

  ngOnInit () {
    this.getAllProducts();
  }

  getAllProducts() {
    this.productService.getAllProducts().subscribe({
      next: resp => {
        this.products = resp;
        this.cd.detectChanges(); // Forcer la détection de changement
      },
      // ...
    });
  }

  handleDelete(p: any) {
    this.productService.deleteProduct(p).subscribe({
      next: resp => {
        this.getAllProducts(); // Recharger la liste après suppression
      },
      // ...
    });
  }
}
```

#### Template `products.html`

Le template utilise la syntaxe de contrôle de flux d’Angular (`@for`, `@if`) pour afficher efficacement la liste des produits dans le tableau.

```html
<table class="table">
  <!-- ... thead ... -->
  @if (products) {
    <tbody>
      @for (p of products; track p){
        <tr>
          <td>{{p.id}}</td>
          <td>{{p.name}}</td>
          <td>{{p.price}}</td>
          <td>
            @if (p.selected) { <i class="bi bi-check-circle"></i> }
            @else { <i class="bi bi-circle"></i> }
          </td>
          <td>
            <button (click)="handleDelete(p)" class="btn btn-outline-danger">
              <i class="bi bi-trash"></i>
            </button>
          </td>
        </tr>
      }
    </tbody>
  }
</table>
```

## 4. Configuration de la base de données

Le backend utilise une base de données H2 en mémoire, ce qui simplifie le développement et les tests sans avoir besoin d’un serveur de base de données externe.

Fichier `application.properties` :
```properties
spring.datasource.url=jdbc:h2:mem:product-db
spring.datasource.driverClassName=org.h2.Driver
spring.datasource.username=sa
spring.datasource.password=password
spring.jpa.database-platform=org.hibernate.dialect.H2Dialect

# Activer la console H2 pour le débogage
spring.h2.console.enabled=true

# Stratégie de génération du schéma (create pour recréer la DB à chaque redémarrage)
spring.jpa.hibernate.ddl-auto=create
```
La console H2 est accessible à l'adresse `http://localhost:8080/h2-console` pour inspecter les données.

## 5. Difficultés rencontrées et solutions

* **Problème CORS (communication Frontend-Backend)** : Les premières requêtes de l’application Angular vers l’API Spring Boot étaient bloquées par le navigateur à cause du Cross-Origin Resource Sharing (CORS).  
  **Solution** : L’annotation `@CrossOrigin("*")` a été ajoutée au contrôleur REST pour autoriser les requêtes depuis n’importe quelle origine.

* **Mise à jour de la vue Angular** : Après la récupération des produits, la liste ne se rafraîchissait pas automatiquement.  
  **Solution** : Le `ChangeDetectorRef` a été utilisé dans le composant `Products` avec `detectChanges()` pour forcer la mise à jour de l’interface.

## 6. Conclusion

Ce TP a permis de se familiariser avec le développement d’applications full-stack et d’appliquer plusieurs concepts essentiels :  

* Créer une API REST fonctionnelle et bien organisée avec Spring Boot.  
* Développer une interface utilisateur réactive et dynamique avec Angular.  
* Gérer la communication asynchrone entre le frontend et le backend.  
* Résoudre des problèmes courants comme les erreurs CORS ou la mise à jour de la vue Angular.  

Le découplage entre frontend et backend offre une base solide pour développer des applications web modulaires et évolutives.

