# TP1_JEE — Questions de compréhension



## 1. Les trois annotations regroupées par `@SpringBootApplication`

`@SpringBootApplication` est une annotation de commodité qui regroupe trois annotations :

| Annotation | Rôle |
|---|---|
| `@SpringBootConfiguration` (hérite de `@Configuration`) | Indique que la classe est une source de définitions de beans Spring (équivalent d'un fichier XML de configuration). Elle permet de déclarer des beans via des méthodes annotées `@Bean`. |
| `@EnableAutoConfiguration` | Active le mécanisme d'auto-configuration de Spring Boot : Spring analyse les dépendances présentes dans le classpath et configure automatiquement les beans nécessaires (ex. : Tomcat, DispatcherServlet, etc.). |
| `@ComponentScan` | Lance le scan des composants (`@Controller`, `@Service`, `@Repository`, `@Component`…) dans le **package courant et tous ses sous-packages**. |

### Que se passerait-il si le contrôleur était dans un package frère ?

Dans notre projet, la classe principale `MycoolappApplication` se trouve dans le package `com.tp1.mycoolapp`. Le `@ComponentScan` implicite scanne donc `com.tp1.mycoolapp` et ses sous-packages (ex. `com.tp1.mycoolapp.controllers`).

Si le contrôleur `FunRestController` était déplacé dans un **package frère** comme `com.tp1.controllers` (qui n'est **pas** un sous-package de `com.tp1.mycoolapp`), il **ne serait pas détecté** par le scan. Spring ne l'enregistrerait pas comme bean, et ses endpoints (`/`, `/teaminfo`, `/workout`) retourneraient une erreur **404 Not Found**.

Pour corriger cela, il faudrait soit :
- Ajouter `@ComponentScan(basePackages = {"com.tp1.mycoolapp", "com.tp1.controllers"})` sur la classe principale.
- Soit déplacer le contrôleur dans un sous-package de `com.tp1.mycoolapp`.

---

## 2. L'auto-configuration

L'auto-configuration est le mécanisme par lequel Spring Boot **configure automatiquement l'application** en fonction des dépendances présentes dans le classpath. Elle repose sur des classes `@AutoConfiguration` conditionnelles (`@ConditionalOnClass`, `@ConditionalOnMissingBean`…) qui ne s'activent que si certaines conditions sont remplies.

### Exemple concret avec `spring-boot-starter-webmvc`

En ajoutant simplement cette dépendance dans le `pom.xml` :

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webmvc</artifactId>
</dependency>
```

Spring Boot configure automatiquement, **sans aucune ligne de code supplémentaire** :

- Un **serveur Tomcat embarqué** qui écoute sur le port 8080
- Le **DispatcherServlet** (le front controller de Spring MVC), mappé sur `/`
- Un **convertisseur de messages** Jackson pour sérialiser/désérialiser du JSON
- La résolution des vues, la gestion des ressources statiques, la gestion des erreurs par défaut (`/error`)

C'est exactement ce qui se passe dans notre projet : on n'a configuré aucun serveur, mais l'application démarre sur le port 8080 avec Tomcat.

---

## 3. `./mvnw spring-boot:run` vs `java -jar target/....jar`

| | `./mvnw spring-boot:run` | `java -jar target/mycoolapp-0.0.1-SNAPSHOT.jar` |
|---|---|---|
| **Mécanisme** | Maven compile le projet puis lance l'application dans un processus enfant avec le classpath du projet | Exécute le JAR exécutable (fat JAR) déjà compilé et packagé par `mvn package` |
| **Compilation** | Compile automatiquement avant de lancer | Nécessite un `mvn package` préalable |
| **ClassLoader** | Utilise le classloader de Maven (avec le classpath du projet) | Utilise le classloader de Spring Boot (`LaunchedURLClassLoader`) |
| **DevTools actif ?** | ✅ **OUI** | ❌ **NON** |

### Pourquoi DevTools n'est actif qu'avec `spring-boot:run` ?

`spring-boot-devtools` est automatiquement **désactivé** lorsqu'il détecte que l'application est lancée depuis un JAR packagé. En effet, dans le fat JAR, DevTools se trouve dans `BOOT-INF/lib/` et le classloader de Spring Boot le charge via `LaunchedURLClassLoader`, ce qui est considéré comme un contexte de production.

Avec `spring-boot:run`, les classes du projet sont chargées par un `RestartClassLoader` séparé, ce qui permet le **redémarrage à chaud** (hot restart) : dès qu'un fichier `.class` change, Spring redémarre automatiquement l'application en quelques millisecondes. On a pu observer ce comportement dans les logs de notre projet :

```
Restarting due to 2 class path changes (0 additions, 0 deletions, 2 modifications)
```

---

## 4. Mot de passe Spring Security généré à chaque démarrage

Par défaut, quand Spring Security est sur le classpath sans configuration explicite, Spring Boot génère un **mot de passe aléatoire** affiché dans la console à chaque démarrage :

```
Using generated security password: a1b2c3d4-e5f6-...
```

### Pourquoi change-t-il à chaque démarrage ?

Le mot de passe est généré par `SecurityProperties` en utilisant `UUID.randomUUID()`. Il n'est jamais persisté.

### En quoi est-ce un bon comportement par défaut ?

- **Sécurité immédiate** : dès l'ajout de la dépendance, l'application est protégée. Aucun endpoint n'est accessible sans authentification.
- **Pas de mot de passe en dur par défaut** : évite le risque qu'un mot de passe par défaut connu de tous soit oublié en production.
- **Incite à configurer** : le caractère temporaire force le développeur à définir ses propres identifiants.

### Pourquoi le personnalise-t-on ?

En pratique, on a besoin de **credentials stables et connues**. Dans notre projet, on les a définies dans `application.properties` :

```properties
spring.security.user.name=abdou
spring.security.user.password=cHk3694
```

En production, on utiliserait plutôt une base de données d'utilisateurs, un annuaire LDAP ou un fournisseur OAuth2.

---

## 5. Ordre de priorité des sources de propriétés

Spring Boot charge les propriétés depuis de nombreuses sources, par **ordre de priorité croissante** (la dernière écrase la première) :

1. Valeurs par défaut internes de Spring Boot
2. `@PropertySource` sur les classes `@Configuration`
3. `application.properties` (ou `application.yml`) dans le classpath
4. `application-{profile}.properties` (profils spécifiques)
5. Variables d'environnement du système (`SPRING_APPLICATION_NAME`, etc.)
6. Propriétés système Java (`-Dserver.port=9090`)
7. **Arguments de ligne de commande** (`--server.port=9090`)

### Quel port est utilisé ?

Si `application.properties` définit `server.port=8080` et qu'on lance avec `--server.port=9090` :

> **Le port 9090 est utilisé**, car les arguments de ligne de commande (`--`) ont une **priorité supérieure** au fichier `application.properties`.

---

## 6. Rôle de `management.info.env.enabled=true`

Cette propriété active la contribution **environnement** de l'endpoint `/actuator/info`. Elle indique à Spring Boot de collecter et d'exposer toutes les propriétés préfixées par `info.*` définies dans `application.properties`.

Dans notre projet :

```properties
management.info.env.enabled=true
info.app.name=My Super Cool App
info.app.description=A crazy fun app, yoohoo!
info.app.version=1.0.0
```

### Avec `management.info.env.enabled=true`, `/actuator/info` retourne :

```json
{
  "app": {
    "name": "My Super Cool App",
    "description": "A crazy fun app, yoohoo!",
    "version": "1.0.0"
  }
}
```

### Sans cette ligne, `/actuator/info` retournerait :

```json
{}
```

Un objet JSON vide, car par défaut la contribution `env` est **désactivée** (depuis Spring Boot 2.6+). Les propriétés `info.*` existent dans le fichier mais ne sont pas exposées par l'endpoint.

---

## 7. Danger d'exposer `/actuator/beans` ou `/actuator/env` sans authentification

Ces endpoints exposent des **informations sensibles** sur l'application :

| Endpoint | Informations exposées |
|---|---|
| `/actuator/beans` | Liste **tous les beans** Spring avec leur type, scope, et dépendances → révèle l'architecture interne de l'application |
| `/actuator/env` | Affiche **toutes les propriétés de configuration**, y compris potentiellement des **mots de passe, clés API, tokens secrets, URLs de bases de données** |

### Risques concrets :

- **Fuite de secrets** : un attaquant pourrait récupérer les identifiants de la base de données, les clés JWT, les credentials de services tiers.
- **Reconnaissance** : la liste des beans révèle les frameworks utilisés, les versions, les bibliothèques → facilite la recherche de vulnérabilités connues (CVE).
- **Surface d'attaque** : connaître la structure interne permet de cibler des attaques précises.

C'est pourquoi dans notre projet, on a ajouté `spring-boot-starter-security` qui protège automatiquement **tous** les endpoints (y compris Actuator) derrière une authentification. Dans `application.properties`, on expose tous les endpoints (`management.endpoints.web.exposure.include=*`), mais ils restent protégés par les credentials configurées.

---

## 8. Différence entre `@RestController` et `@Controller`

| | `@Controller` | `@RestController` |
|---|---|---|
| **Annotation** | `@Controller` | `@RestController` = `@Controller` + `@ResponseBody` |
| **Retour des méthodes** | Retourne un **nom de vue** (template Thymeleaf, JSP…) qui sera résolu par un `ViewResolver` | Retourne directement l'**objet sérialisé** dans le corps de la réponse HTTP (JSON, XML, texte…) |
| **Usage typique** | Applications web avec rendu côté serveur (MVC classique) | APIs REST / microservices |

### Concrètement :

Avec `@Controller`, une méthode qui retourne `"hello"` cherchera un template `hello.html`. Avec `@RestController`, elle écrira la chaîne `"hello"` directement dans la réponse HTTP.

`@RestController` est un raccourci pour éviter d'annoter chaque méthode individuellement avec `@ResponseBody` :

```java
// Avec @Controller, il faut ajouter @ResponseBody sur chaque méthode
@Controller
public class MyController {
    @GetMapping("/")
    @ResponseBody   // ← nécessaire pour écrire dans le corps de la réponse
    public String hello() {
        return "Hello World!";
    }
}

// Avec @RestController, c'est implicite
@RestController
public class FunRestController {
    @GetMapping("/")
    public String sayHello() {
        return "Hello World!";   // ← directement écrit dans la réponse HTTP
    }
}
```

Dans notre projet, `FunRestController` utilise `@RestController` car il expose des endpoints REST qui retournent directement du texte (et non des vues HTML).