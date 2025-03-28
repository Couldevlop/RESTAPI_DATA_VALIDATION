# RESTAPI_DATA_VALIDATION

## Maîtriser la validation dans Spring Boot : Guide complet avec cas d'utilisation concrets

### INTRODUCTION

La validation des données est une étape cruciale et indispensable dans le développement d'applications web modernes, notamment dans une API REST où les données entrantes peuvent provenir de sources non fiables. Spring Boot, avec son intégration native de la spécification Bean Validation (JSR-380), offre des outils puissants pour garantir l'intégrité des données, améliorer la sécurité et offrir une expérience utilisateur plus fluide. Dans cet article, nous allons analyser en profondeur les mécanismes de validation dans Spring Boot, dépuis les annotations simples jusqu'aux approches personnalisées et complexes, le tout, illustrés par des cas d'utilisation concrets issus de mon expérience, et vous donner(proposer) les clés pour implémenter une validation robuste et adaptée à vos besoins.

1. **Les fondamentaux de la validation des données**  
   **_Pourquoi la validation est indispensable?_**

La validation n'est pas seulement une action purement technique bien au contraire, elle est un bouclier solide contre:

- Les entrées malveillantes
- Les données incomplètes ou incorrectes
- Les risques de sécurités potentiels(les injections)
- Les erreurs d'integrations système

2. **La validation avec springboot**
   A- **_Les annotations et validation standards_**

Spring Boot utilise Hibernate Validator(annotation de Jakarta Bean Validation) comme implémentation par défaut de Bean Validation. Cela permet de valider les objets Java en annotant leurs champs avec des contraintes comme **_@NotNull_** , **_@Min_** , **_notBlank_**, **_patern(regex)_**, **_positive_**, **_negative_**, **_@Size_** , **_@Email_** etc. Prenons un exemple simple :

```java
public class UserRequest {
@NotNull(message = "Le nom est obligatoire")
    @Size(min = 2, max = 50, message = "Le nom doit contenir entre 2 et 50 caractères")
    private String name;

    @NotNull(message = "L’âge est requis")
    @Min(value = 18, message = "L’âge doit être supérieur ou égal à 18")
    private Integer age;

    @Email(message = "L’email doit être valide")
    private String email;

    // Getters et setters

}
```

- **_Les annotations imbriguées_**

```java
 public class Commande {
    @Valid
    private Utilisateur utilisateur;

    @Valid
    private List<Produit> produits;

}
```

Dans l'exemple précédent, l'on suppose que les objets **Utilisateur** et **Produit** contiennent des annotations de validation

Dans un contrôleur REST, l'annotation **@Valid** déclenche la validation de l'objet :

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @PostMapping
    public ResponseEntity<String> createUser(@Valid @RequestBody UserRequest user, BindingResult result) {
        if (result.hasErrors()) {
            return ResponseEntity.badRequest().body(result.getAllErrors().toString());
        }
        return ResponseEntity.ok("Utilisateur créé avec succès");
    }

}
```

Si une donnée invalide est soumise (par exemple, un âge de 15 ans), Spring renvoie automatiquement une erreur, capturée dans **BindingResult**.

A1- **Cas d'utilisations**

**Cas 1** : Validation d'une inscription multi-étapes
Considérons une application d'inscription où un utilisateur doit fournir des informations en plusieurs étapes : informations personnelles, adresse, puis préférences. Chaque étape a ses propres contraintes.

```java
public class RegistrationStep1 {
    @NotBlank(message = "Le prénom est requis")
    private String firstName;

    @NotBlank(message = "Le nom de famille est requis")
    private String lastName;

    @Past(message = "La date de naissance doit être dans le passé")
    private LocalDate birthDate;

}
```

```java
public class RegistrationStep2 {
    @NotBlank(message = "La rue est requise")
    private String street;

    @Pattern(regexp = "\\d{5}", message = "Le code postal doit contenir 5 chiffres")
    private String postalCode;

}
```

Dans le contrôleur, chaque étape peut être validée séparément :

```java
@PostMapping("/step1")
public ResponseEntity<String> submitStep1(@Valid @RequestBody RegistrationStep1 step1, BindingResult result) {
    if (result.hasErrors()) {
    return ResponseEntity.badRequest().body(result.getAllErrors().toString());
    }
    // Logique métier
    return ResponseEntity.ok("Étape 1 validée");
}
```

**Avantage** : La validation est modulaire et chaque étape reste indépendante, ce qui améliore la lisibilité et la maintenance.

**Cas 2** : Validation d'une commande
Considérons une plateforme d'e-commerce dans laquelle on veut garantir l'integrité des données entrantes:

. Des prix positifs (@Positive).
. Des quantités en stock suffisantes (@Min(1)).
. Des adresses email valides (@Email).

```java
public class OrderRequest {
    @Positive(message = "Le prix doit être positif")
    private BigDecimal price;

    @Min(value = 1, message = "La quantité minimale est 1")
    private int quantity;

    @Email(message = "Format d'email invalide")
    private String customerEmail;

}
```

**Avantage**: On peut valider certaines contraintes sans toucher à la logique métier en question.

**Cas 3** : Validation conditionnelle dans une API de réservation

Pour une API de réservation de vols, la validation peut dépendre du contexte. Par exemple, un billet "aller simple" ne nécessite pas de date de retour, contrairement à un "aller-retour".

```java
public class FlightBooking {
    @NotNull(message = "Le type de billet est requis")
    private String ticketType;

    @NotNull(message = "La date de départ est requise")
    private LocalDate departureDate;

    @AssertTrue(message = "La date de retour est requise pour un aller-retour")
    public boolean isReturnDateValid() {
        if ("ROUND_TRIP".equals(ticketType)) {
            return returnDate != null;
        }
        return true;
    }

    private LocalDate returnDate;

}
```

**Avantage**: Ici, **_@AssertTrue_** permet une validation personnalisée basée sur une logique métier. Cela montre la flexibilité de Bean Validation pour des scénarios complexes.

**Cas 3** : Validation des fichiers uploadés
Dans une application permettant l'upload de fichiers (par exemple, une photo de profil), on peut valider la taille et le type du fichier avant traitement :

```java
public class ProfileUpdate {
    @NotNull(message = "La photo est requise")
    @Size(max = 5 _ 1024 _ 1024, message = "La photo ne doit pas dépasser 5 Mo")
    private MultipartFile profilePicture;

    // Validation personnalisée
    public void validateFileType() {
        if (!Arrays.asList("image/jpeg", "image/png").contains(profilePicture.getContentType())) {
            throw new IllegalArgumentException("Seuls les formats JPEG et PNG sont acceptés");
        }
    }

}

```

NB: Bien que j'ai utilisé la validation personnalisée dans cet exemple mais ce chapitre fera l'objet du second grand point.

A2- **Que peut-on retenir?**

- **Avantages des annotations standards et l'utilisation de @Valid**
  . simple et déclaratif
  . Intégration native avec spring
  . Validation automatique
  . Gestion des erreurs simplifiées
  . Lisible et proche du modèle de données

- **Inconvenients**
  . Limités aux règles standards
  . peu ou pas flexibles pour les logiques metiers complexes
  . Couplage avec le modèle de données
  . Une gestion d'erreur verbeuse quand il y'a assez de champs à annoter

3- **La validation personnalisée**

**Pourquoi recourir à la validation personnalisée?**
Il faut comprendre qu'on ne personnalise pas la validation des données pour le plaisir de coder mais pour des raisons bien spécifiques. Je vais vous en partager celles que j'ai pu rencontrer.

- Logique métier complexe
  . Dépendence des champs
  Lorsque la validation d'un champs depend de la valeur d'un autre champs
  Par: la date de fin doit être postérieur à la date de debut(cas d'achat de biellets aller/retour)

  . Règles métiers spécifiques
  Lorsque la validation implique nécessairement la vérification de la cohérence de cesdonnées avec d'autres informations provenants d'autres sources de données(base de données ou sources externes)

  . Validation avec des logiques conditionnelles avancées
  Lorsque la validation tiens comptes de plusieurs facteurs.
  Par si l'âge est compris en 25 ans et 60 ans et qu'il du departement RH avec 5 ans de présence et n'ayant jamais été traduit en conseille de diciplibe alors...

  . Contrôles nécessitant des calculs ou des algorithmes spécifiques
  Par exemple: Validation de Capacité d'Emprunt (LoanEligibilityValidator)
  Objectif : Évaluer l'éligibilité d'un client à un prêt via des calculs complexes

      Calculs et algorithmes spécifiques :

            Calcul du ratio d'endettement
            Vérification de la stabilité d'emploi
            Évaluation du score de crédit
            Calcul du montant maximum de prêt autorisé

            Critères de validation :

            Ratio d'endettement inférieur à 33%
            Ancienneté professionnelle d'au moins 2 ans
            Score de crédit supérieur à 700
            Montant du prêt proportionnel aux revenus

- Validation de format personnalisée
  . Validation basée sur des conditions
  Lorsque la validation d'un ou des champs varie en fonction de certaines conditions.
  Par exemple: un champ peut être uniquement obligatoire dans certains cas

  . Validation de groupes de données
  Lorsqu'un champ peut changer les rçegles de validations en fonction du context:
  Par exemple: Lors de la création d'un objet, certains champs sont validés et d'autres le sont à la mise à jour du même objet.

- Sécurité renforcée pour ajouter des couches supplémentaires de validation de sécurité
  . Détection de tentatives d'injection
  . Validation de jetons ou de signatures
  . Contrôle des plages de valeurs sensibles

- Internationalisation quand les validations doivent prendre en compte des spécificités
  . Formats de date différents selon les pays
  . Règles de validation variant selon les réglementations locales
  . Gestion de jeux de caractères spécifiques

- Intégrations avec des systèmes externes lorsque vous devez effectuer des vérifications impliquant des appels à des services externes.
  . Validation d'un numéro de carte bancaire
  . Vérification de l'existence d'un utilisateur dans un autre système
  . Contrôle de conformité avec des référentiels externes

A2- **Cas d'utilisations**
**_Création de Validateurs Personnalisés_**

- **Messages d'erreur dynamiques**

Pour une expérience utilisateur optimale, personnalisez les messages d'erreur dans un fichier de messages.properties :

```propertie
NotNull.userRequest.name=Le nom ne peut pas être vide
Size.userRequest.name=Le nom doit contenir entre {min} et {max} caractères
```

- **Utilisation de l'interface Validator**

Ici le principe consiste à créer une classe de validation personnalisée qui implemente l'interface Validator.

```java
@Component
public class CommandeValidator implements Validator {
    // Méthode qui vérifie si le validateur supporte la classe à valider
    @Override
    public boolean supports(Class<?> clazz) {
        return Commande.class.equals(clazz);
    }

    // Méthode principale de validation
    @Override
    public void validate(Object target, Errors errors) {
        // Conversion de l'objet générique en objet Commande spécifique
        Commande commande = (Commande) target;

        // Validation personnalisée : vérification des produits
        if (commande.getProduits().isEmpty()) {
            // Ajout d'une erreur si aucun produit n'est présent
            errors.rejectValue("produits", "commande.produits.vide",
                               "La commande doit contenir au moins un produit");
        }
    }
}
```

**Explications**

**supports()**
Méthode qui indique quels types de classes peuvent être validés
Ici, uniquement les instances de **Commande**
Permet de filtrer et n'appliquer le validateur que sur les bonnes classes

**_validate()_**
Méthode core qui réalise la validation personnalisée
Reçoit l'objet à valider et un objet Errors pour collecter les erreurs
Utilise **errors.rejectValue()** pour ajouter des erreurs de validation

**Cas1**:

```java
@Service
public class CommandeService {
    @Autowired
    private CommandeValidator commandeValidator;

    public void passerCommande(Commande commande) {
        // Validation manuelle avec le validateur personnalisé
        DataBinder binder = new DataBinder(commande);
        binder.setValidator(commandeValidator);
        binder.validate();

        BindingResult results = binder.getBindingResult();

        if (results.hasErrors()) {
            throw new ValidationException("Commande invalide");
        }

        // Logique de traitement de la commande
        verifierStockProduits(commande);
        calculerPrixTotal(commande);
        enregistrerCommande(commande);
    }

    private void verifierStockProduits(Commande commande) {
        for (Produit produit : commande.getProduits()) {
            if (produit.getQuantiteEnStock() < produit.getQuantiteCommandee()) {
                throw new StockInsuffisantException(
                    "Stock insuffisant pour le produit : " + produit.getNom()
                );
            }
        }
    }
}
```

- **Validation de groupes**
  La validation de groupes est une fonctionnalité avancée de la spécification Bean Validation (JSR-380), intégrée à Spring Boot via Hibernate Validator. Elle permet de définir des ensembles de contraintes qui ne s'appliquent que dans des contextes spécifiques, comme la création ou la mise à jour d'une entité. Cette approche est particulièrement utile lorsque les règles de validation diffèrent selon l'opération effectuée sur un objet.

**Pourquoi utiliser les groupes de validation ?**

Dans une application, un même objet (souvent un DTO) peut être utilisé dans plusieurs scénarios, mais les contraintes de validation ne sont pas toujours identiques. Par exemple :

. Lors de la création d'un utilisateur, le **nom** du champ doit être obligatoire, mais pas l' **identifiant** (car il sera généré).

. Lors de la **_mise à jour_** , l' **identifiant** devient obligatoire pour identifier l'entité, tandis que le **nom** peut être optionnel.

Sans groupes, toutes les contraintes s'appliquent en même temps, ce qui peut entraîner des erreurs indésirables ou un code difficile à maintenir. Les groupes permettent de segmenter ces contraintes.

**Comment faire?**

1- **Créer des interfaces de groupes**

Les groupes sont simplement des interfaces vides qui servent de marqueurs. Par convention, les surnommer selon leur contexte (ex. OnCreate , OnUpdate ).

```java
public interface OnCreate {}
public interface OnUpdate {}
```

2- **_Associer des contraintes à des groupes :_**

Dans votre classe, utilisez l'attribut groups des annotations de validation pour lier une contrainte à un ou plusieurs groupes.

```java
public class UserRequest {
    @NotNull(groups = OnCreate.class, message = "Le nom est requis lors de la création")
    @Size(min = 2, max = 50, groups = {OnCreate.class, OnUpdate.class},
          message = "Le nom doit contenir entre 2 et 50 caractères")
    private String name;

    @NotNull(groups = OnUpdate.class, message = "L’ID est requis pour la mise à jour")
    private Long id;

    @Email(groups = {OnCreate.class, OnUpdate.class}, message = "L’email doit être valide")
    private String email;

    // Getters et setters
}
```

- Ici, le **_nom_** est obligatoire uniquement à la création ( OnCreate ).
- **_id_** est obligatoire uniquement à la mise à jour ( OnUpdate ).
- **_email_** est validé dans les deux cas.

3- **_Utilisation dans un controlleur_**

Pour activer la validation avec des groupes, utilisez l'annotation @Validated (fournie par Spring) au lieu de **@Valid** .
**@Validated** permet de préciser les groupes à appliquer.

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @PostMapping
    public ResponseEntity<String> createUser(
            @Validated(OnCreate.class) @RequestBody UserRequest user,
            BindingResult result) {
        if (result.hasErrors()) {
            return ResponseEntity.badRequest().body(result.getAllErrors().toString());
        }
        return ResponseEntity.ok("Utilisateur créé");
    }

    @PutMapping
    public ResponseEntity<String> updateUser(
            @Validated(OnUpdate.class) @RequestBody UserRequest user,
            BindingResult result) {
        if (result.hasErrors()) {
            return ResponseEntity.badRequest().body(result.getAllErrors().toString());
        }
        return ResponseEntity.ok("Utilisateur mis à jour");
    }
}
```

- **Création** : Seules les contraintes associées à OnCreate (ex. **_nom_** obligatoire, **_email_** valide) sont vérifiées. Si je suis fourni, il est ignoré.
- **Mise à jour** : Seules les contraintes associées à OnUpdate (ex. **_id_** obligatoire, **_email_** valide) sont vérifiées. Si **name** est omis, aucune erreur n'est levée.

**ALLER PLUS LOIN**
**_Exemple avancé :_** Validation imbriquée avec groupes

Les groupes fonctionnent également avec des objets imbriqués. Imaginons une classe OrderRequest contenant une liste de ItemRequest :

```java
public class ItemRequest {
    @NotNull(groups = OnCreate.class, message = "Le nom de l’article est requis")
    private String itemName;

    @Min(value = 1, groups = {OnCreate.class, OnUpdate.class},
         message = "La quantité doit être positive")
    private Integer quantity;
}

public class OrderRequest {
    @NotNull(groups = OnCreate.class, message = "Le client est requis")
    private String customerName;

    @Valid // Nécessaire pour valider les objets imbriqués
    @NotEmpty(groups = OnCreate.class, message = "La commande doit contenir au moins un article")
    private List<ItemRequest> items;
}
```

Dans le controlleur

```java
@PostMapping("/orders")
public ResponseEntity<String> createOrder(
        @Validated(OnCreate.class) @RequestBody OrderRequest order,
        BindingResult result) {
    if (result.hasErrors()) {
        return ResponseEntity.badRequest().body(result.getAllErrors().toString());
    }
    return ResponseEntity.ok("Commande créée");
}

```

Ici, **@Valid** sur la liste items propage la validation aux objets ItemRequest , mais uniquement pour les contraintes du groupe OnCreate .

**Cas pratique(autre)** : Gestion d'un profil utilisateur
Considerons une API de gestion de profils avec trois opérations : inscription, mise à jour partielle, et mise à jour complète.

```java
public interface OnRegister {}
public interface OnPartialUpdate {}
public interface OnFullUpdate {}

public class UserProfile {
    @NotNull(groups = OnRegister.class, message = "Le nom est requis à l’inscription")
    @Size(min = 2, max = 50, groups = {OnRegister.class, OnFullUpdate.class},
          message = "Le nom doit contenir entre 2 et 50 caractères")
    private String name;

    @NotNull(groups = {OnPartialUpdate.class, OnFullUpdate.class},
             message = "L’ID est requis pour une mise à jour")
    private Long id;

    @NotNull(groups = OnFullUpdate.class, message = "L’adresse est requise pour une mise à jour complète")
    private String address;
}
```

Controlleur

```java
@PostMapping("/register")
public ResponseEntity<String> register(@Validated(OnRegister.class) @RequestBody UserProfile profile, BindingResult result) {
    // ...
}

@PatchMapping("/partial-update")
public ResponseEntity<String> partialUpdate(@Validated(OnPartialUpdate.class) @RequestBody UserProfile profile, BindingResult result) {
    // ...
}

@PutMapping("/full-update")
public ResponseEntity<String> fullUpdate(@Validated(OnFullUpdate.class) @RequestBody UserProfile profile, BindingResult result) {
    // ...
}
```

**Inscription** : Vérifie uniquement le nom .
**Mise à jour partielle** : Vérifie uniquement l'identifiant .
**Mise à jour complète** : Vérifie l'identifiant , le nom et l'adresse .

**Avantages des groupes de validation**
**_Flexibilité_** : Permet de réutiliser le même DTO dans différents contextes sans dupliquer les classes.
**_Granularité_** : Appliquez uniquement les règles pertinentes à chaque opération.
**_Clarté_** : Les intentions de validation sont explicites dans le code, ce qui facilite la maintenance.

**Limites et précautions**
**_Complexité accumulée_** : Trop de groupes peuvent rendre le code difficile à suivre. Limitez leur nombre et documentez leur utilisation.
**Dépendance à @Validated** : Contrairement à @Valid , @Validated est spécifique à Spring, ce qui peut poser problème si vous cherchez une portabilité hors de Spring.
**Validation par défaut** : Si aucun groupe n'est spécifié dans @Validated , aucune contrainte ne sera validée, sauf celles sans groups . Attention à ne pas omettre les groupes par erreur.

**Différence entre @Valid, @Validated et une validation personnalisée**

1. **_Utilisation de @Valid_**
   . **Description** : @Valid est une annotation standard de JSR-380 qui déclenche la validation des contraintes définies sur un objet (ex. @NotNull , @Size ). Elle est utilisée dans les contrôleurs pour valider les objets passés en paramètres (souvent des DTO).

   . **Avantages** :
   Intégration native avec Spring et Hibernate Validator.
   Simplicité et lisibilité : les règles sont directement dans le modèle.
   Gestion automatique des erreurs via BindingResult ou @ExceptionHandler .

   . **Limites** :
   Moins flexible pour des validations complexes ou dynamiques (ex. dépendance entre champs).
   Ne convient pas aux cas où la logique de validation nécessite des appels externes (base de données, API).

2. **_Utilisation de @Validated_**

   . **Description** : Une variante de @Valid fournie par Spring, qui supporte les groupes de validation (ex. OnCreate , OnUpdate ). Elle peut être utilisée au niveau des méthodes ou des classes.
   . **Avantages** :
   Permet de contextualiser la validation (par exemple, valider uniquement certains champs selon l'opération).
   Plus granulaire que @Valid .
   . **Limites** :
   Toujours limité aux contraintes statiques définies dans le modèle.
   Nécessite une compréhension des groupes, ce qui peut compliquer le code.

3. **_Fonction utilitaire personnalisée_**

   . **Description** : Une méthode ou classe dédiée pour valider les données, indépendante des annotations Bean Validation. Exemple :

```java
public class UserValidator {
    public static void validateUser(UserRequest user) {
        if (user.getName() == null || user.getName().isBlank()) {
            throw new IllegalArgumentException("Le nom est requis");
        }
        if (user.getAge() != null && user.getAge() < 18) {
            throw new IllegalArgumentException("L’âge doit être supérieur ou égal à 18");
        }
    }
}
```

. **Avantages** :
**Flexibilité totale** : peut inclure des appels à une base de données, des API, ou des règles complexes.
Indépendant du framework, donc portable.
. **Limites** :
Code plus verbeux et moins standardisé.
Pas d'intégration automatique avec les mécanismes de Spring (ex. BindingResult ).

**Quelle est la meilleure pratique ?**

. **_Recommandation(ma proposition)_** : Utilisez @Valid ou @Validated comme approche par défaut, et complétez avec une fonction utilitaire personnalisée uniquement pour des cas spécifiques.
. **Pourquoi :**

- **Simplicité et standardisation** : Les annotations sont intuitives, maintenables et bien intégrées à Spring. Elles réduisent le code passe-partout et centralisent les règles dans le modèle.

- **Évolutivité** : Pour des cas complexes (ex. validation dépendant d'une base de données), une fonction utilitaire est plus adaptée, mais elle doit être une exception, pas la règle.

- **Lisibilité** : Mélanger les deux approches de manière cohérente (annotations pour les règles simples, fonctions pour les cas avancés) garder le code clair et structuré.

En résumé, **@Valid** et **@Validated** sont les meilleurs choix pour 80-90 % des cas grâce à leur intégration et leur simplicité. Une fonction utilitaire est un complément puissant pour les 10-20 % restants, où la flexibilité prime sur la standardisation.

**NB:** Je n'ai pas pu traiter chaque cas d'utilisation afin d'éviter de trop allonger l'article.
