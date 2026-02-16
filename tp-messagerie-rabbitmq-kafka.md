# TP Complet - Microservices avec Messagerie Asynchrone

**Module** : Architecture Microservices  
**Sujet** : Communication asynchrone avec RabbitMQ et Apache Kafka  
**Durée** : 3 heures (2 TPs parallèles)  
**Niveau** : Master 2  
**Prérequis** : Microservices, Spring Boot, Docker

---

## 📚 Table des matières

1. [Introduction à la messagerie asynchrone](#1-introduction)
2. [TP 1 : RabbitMQ - Système bancaire](#tp-1-rabbitmq)
3. [TP 2 : Apache Kafka - E-commerce](#tp-2-kafka)
4. [Comparaison RabbitMQ vs Kafka](#comparaison)
5. [Exercices et évaluation](#exercices)

---

# 1. Introduction à la messagerie asynchrone

## 1.1 Communication synchrone vs asynchrone

### Communication SYNCHRONE (Vue précédemment)

```
Client Service                Account Service
      │                             │
      ├─── POST /create-account ────►
      │                             │
      │        (ATTEND...)          │
      │        (bloqué)             │
      │                             │
      ◄─────── 201 Created ─────────┤
      │                             │
   Continue                      Continue
```

**Problèmes** :
- ❌ Le client attend (bloquant)
- ❌ Si Account Service est down → erreur immédiate
- ❌ Couplage fort entre services
- ❌ Timeout si le traitement est long

### Communication ASYNCHRONE (Avec messagerie)

```
Client Service          Message Broker          Account Service
      │                       │                        │
      ├─── Envoie message ────►                        │
      │                       │                        │
   Continue              Stocke message                │
   (ne bloque pas)            │                        │
      │                       ◄──── Consomme ──────────┤
      │                       │                        │
      │                    Traite                   Continue
```

**Avantages** :
- ✅ Non-bloquant (asynchrone)
- ✅ Résilience (message persisté)
- ✅ Découplage des services
- ✅ Peak handling (absorbe les pics de charge)

## 1.2 Concepts de base

### Message Broker

Un **message broker** est un intermédiaire qui :
- Reçoit des messages des producteurs
- Stocke les messages
- Distribue les messages aux consommateurs

**Analogie** : C'est comme une **boîte postale** où on dépose des lettres (messages) que d'autres viendront récupérer.

### Producer (Producteur)

Service qui **envoie** des messages.

### Consumer (Consommateur)

Service qui **reçoit** et traite des messages.

### Queue (File)

Stockage temporaire des messages (FIFO : First In, First Out).

```
Producer → [Message 1 | Message 2 | Message 3] → Consumer
           └─────────── Queue ──────────────┘
```

### Topic

Canal de diffusion où plusieurs consommateurs peuvent s'abonner.

```
                    ┌─► Consumer A
Producer → Topic ───┼─► Consumer B
                    └─► Consumer C
```

---

# TP 1 : RabbitMQ - Système bancaire

**Durée** : 1h30  
**Objectif** : Créer un système de notifications bancaires avec RabbitMQ

## Cas d'usage

```
Scénario : Notification de transaction bancaire

1. Account Service effectue une transaction
2. Envoie un message à RabbitMQ
3. Notification Service consomme le message
4. Envoie un email/SMS au client
```

## Architecture

```
┌─────────────────┐
│ Account Service │ (Producer)
│   (Port 8081)   │
└────────┬────────┘
         │
         │ Envoie message
         │ "Transaction créée"
         ▼
┌─────────────────┐
│   RabbitMQ      │ (Message Broker)
│   (Port 5672)   │
│   Management    │
│   (Port 15672)  │
└────────┬────────┘
         │
         │ Consomme message
         ▼
┌─────────────────┐
│ Notification    │ (Consumer)
│    Service      │
│   (Port 8084)   │
└─────────────────┘
```

---

## Partie 1 : Installation de RabbitMQ

### Méthode 1 : Docker (Recommandé)

```bash
# Démarrer RabbitMQ avec interface de management
docker run -d --name rabbitmq \
  -p 5672:5672 \
  -p 15672:15672 \
  -e RABBITMQ_DEFAULT_USER=admin \
  -e RABBITMQ_DEFAULT_PASS=admin \
  rabbitmq:3-management

# Vérifier que RabbitMQ démarre
docker logs -f rabbitmq
```

**Interface Web** : http://localhost:15672
- Username : `admin`
- Password : `admin`

### Méthode 2 : Installation locale

**Windows** :
1. Télécharger RabbitMQ : https://www.rabbitmq.com/install-windows.html
2. Installer Erlang (prérequis)
3. Installer RabbitMQ
4. Activer le plugin management :
   ```bash
   rabbitmq-plugins enable rabbitmq_management
   ```

**macOS** :
```bash
brew install rabbitmq
brew services start rabbitmq
```

**Linux (Ubuntu)** :
```bash
sudo apt-get update
sudo apt-get install rabbitmq-server
sudo systemctl start rabbitmq-server
sudo rabbitmq-plugins enable rabbitmq_management
```

---

## Partie 2 : Modifier Account Service (Producer)

### Étape 1 : Ajouter la dépendance RabbitMQ

Dans `account-service/pom.xml` :

```xml
<dependencies>
    <!-- Dépendances existantes -->
    
    <!-- RabbitMQ -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-amqp</artifactId>
    </dependency>
</dependencies>
```

### Étape 2 : Configuration RabbitMQ

Dans `account-service/src/main/resources/application.properties` :

```properties
# Configuration existante...

# ========================================
# Configuration RabbitMQ
# ========================================
spring.rabbitmq.host=localhost
spring.rabbitmq.port=5672
spring.rabbitmq.username=admin
spring.rabbitmq.password=admin

# Nom de la queue
rabbitmq.queue.transaction=transaction.queue
rabbitmq.exchange.transaction=transaction.exchange
rabbitmq.routing.key.transaction=transaction.routing.key
```

### Étape 3 : Configuration des queues et exchanges

Créer `account-service/src/main/java/com/bank/accountservice/config/RabbitMQConfig.java` :

```java
package com.bank.accountservice.config;

import org.springframework.amqp.core.*;
import org.springframework.amqp.rabbit.connection.ConnectionFactory;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.amqp.support.converter.Jackson2JsonMessageConverter;
import org.springframework.amqp.support.converter.MessageConverter;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * Configuration RabbitMQ pour Account Service
 * 
 * Crée automatiquement :
 * - Une queue (file d'attente)
 * - Un exchange (point d'échange)
 * - Un binding (liaison entre queue et exchange)
 */
@Configuration
public class RabbitMQConfig {

    @Value("${rabbitmq.queue.transaction}")
    private String queueName;

    @Value("${rabbitmq.exchange.transaction}")
    private String exchangeName;

    @Value("${rabbitmq.routing.key.transaction}")
    private String routingKey;

    /**
     * Créer la queue
     * durable = true : la queue survit au redémarrage de RabbitMQ
     */
    @Bean
    public Queue transactionQueue() {
        return new Queue(queueName, true);
    }

    /**
     * Créer l'exchange de type Topic
     * Topic permet le routage avec patterns (ex: transaction.*)
     */
    @Bean
    public TopicExchange transactionExchange() {
        return new TopicExchange(exchangeName);
    }

    /**
     * Lier la queue à l'exchange avec une routing key
     */
    @Bean
    public Binding binding(Queue transactionQueue, TopicExchange transactionExchange) {
        return BindingBuilder
            .bind(transactionQueue)
            .to(transactionExchange)
            .with(routingKey);
    }

    /**
     * Convertir les objets Java en JSON pour l'envoi
     */
    @Bean
    public MessageConverter jsonMessageConverter() {
        return new Jackson2JsonMessageConverter();
    }

    /**
     * Template RabbitMQ pour envoyer des messages
     */
    @Bean
    public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory) {
        RabbitTemplate template = new RabbitTemplate(connectionFactory);
        template.setMessageConverter(jsonMessageConverter());
        return template;
    }
}
```

### Étape 4 : Créer le DTO de message

Créer `account-service/src/main/java/com/bank/accountservice/dto/TransactionMessage.java` :

```java
package com.bank.accountservice.dto;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.time.LocalDateTime;

/**
 * Message envoyé à RabbitMQ lors d'une transaction
 */
@Data
@NoArgsConstructor
@AllArgsConstructor
public class TransactionMessage {
    
    private Long accountId;
    private String accountNumber;
    private String transactionType;  // DEPOT, RETRAIT, VIREMENT
    private Double amount;
    private Double newBalance;
    private Long customerId;
    private LocalDateTime timestamp;
    private String description;
}
```

### Étape 5 : Créer le service de messagerie

Créer `account-service/src/main/java/com/bank/accountservice/service/RabbitMQProducer.java` :

```java
package com.bank.accountservice.service;

import com.bank.accountservice.dto.TransactionMessage;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

/**
 * Service pour envoyer des messages à RabbitMQ
 */
@Service
public class RabbitMQProducer {

    private static final Logger logger = LoggerFactory.getLogger(RabbitMQProducer.class);

    private final RabbitTemplate rabbitTemplate;

    @Value("${rabbitmq.exchange.transaction}")
    private String exchangeName;

    @Value("${rabbitmq.routing.key.transaction}")
    private String routingKey;

    public RabbitMQProducer(RabbitTemplate rabbitTemplate) {
        this.rabbitTemplate = rabbitTemplate;
    }

    /**
     * Envoyer un message de transaction à RabbitMQ
     */
    public void sendTransactionMessage(TransactionMessage message) {
        logger.info("📤 Envoi message à RabbitMQ: {}", message);
        
        rabbitTemplate.convertAndSend(
            exchangeName,
            routingKey,
            message
        );
        
        logger.info("✅ Message envoyé avec succès");
    }
}
```

### Étape 6 : Modifier AccountService pour envoyer des messages

Modifier `account-service/src/main/java/com/bank/accountservice/service/AccountService.java` :

```java
package com.bank.accountservice.service;

import com.bank.accountservice.dto.TransactionMessage;
import com.bank.accountservice.model.Account;
import com.bank.accountservice.repository.AccountRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.LocalDateTime;

@Service
@Transactional
public class AccountService {

    @Autowired
    private AccountRepository accountRepository;
    
    // ============================================
    // AJOUT pour RabbitMQ
    // ============================================
    @Autowired
    private RabbitMQProducer rabbitMQProducer;

    // Méthodes existantes...

    /**
     * Effectuer un dépôt sur un compte
     */
    public Account deposit(Long accountId, Double amount) {
        Account account = getAccountById(accountId);
        
        // Mettre à jour le solde
        account.setBalance(account.getBalance() + amount);
        Account savedAccount = accountRepository.save(account);
        
        // ============================================
        // NOUVEAU : Envoyer message à RabbitMQ
        // ============================================
        TransactionMessage message = new TransactionMessage(
            savedAccount.getId(),
            savedAccount.getAccountNumber(),
            "DEPOT",
            amount,
            savedAccount.getBalance(),
            savedAccount.getCustomerId(),
            LocalDateTime.now(),
            "Dépôt de " + amount + " FCFA"
        );
        
        rabbitMQProducer.sendTransactionMessage(message);
        
        return savedAccount;
    }

    /**
     * Effectuer un retrait sur un compte
     */
    public Account withdraw(Long accountId, Double amount) {
        Account account = getAccountById(accountId);
        
        if (account.getBalance() < amount) {
            throw new RuntimeException("Solde insuffisant");
        }
        
        // Mettre à jour le solde
        account.setBalance(account.getBalance() - amount);
        Account savedAccount = accountRepository.save(account);
        
        // ============================================
        // NOUVEAU : Envoyer message à RabbitMQ
        // ============================================
        TransactionMessage message = new TransactionMessage(
            savedAccount.getId(),
            savedAccount.getAccountNumber(),
            "RETRAIT",
            amount,
            savedAccount.getBalance(),
            savedAccount.getCustomerId(),
            LocalDateTime.now(),
            "Retrait de " + amount + " FCFA"
        );
        
        rabbitMQProducer.sendTransactionMessage(message);
        
        return savedAccount;
    }
}
```

---

## Partie 3 : Créer Notification Service (Consumer)

### Étape 1 : Créer le projet

**Via Spring Initializr** (https://start.spring.io/) :

- **Group** : `sn.ec2tl.banque`
- **Artifact** : `notification-service`
- **Dependencies** :
  - Spring Web
  - Spring for RabbitMQ (AMQP)
  - Lombok

### Étape 2 : pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.4.1</version>
        <relativePath/>
    </parent>
    
    <groupId>sn.ec2tl.banque</groupId>
    <artifactId>notification-service</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>notification-service</name>
    <description>Service de notifications bancaires</description>
    
    <properties>
        <java.version>17</java.version>
    </properties>
    
    <dependencies>
        <!-- Spring Web -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        
        <!-- RabbitMQ -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-amqp</artifactId>
        </dependency>
        
        <!-- Lombok -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        
        <!-- Test -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
    
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

### Étape 3 : Configuration

`notification-service/src/main/resources/application.properties` :

```properties
# Configuration du serveur
spring.application.name=notification-service
server.port=8084

# Configuration RabbitMQ
spring.rabbitmq.host=localhost
spring.rabbitmq.port=5672
spring.rabbitmq.username=admin
spring.rabbitmq.password=admin

# Nom de la queue (doit correspondre à Account Service)
rabbitmq.queue.transaction=transaction.queue
```

### Étape 4 : DTO (identique à Account Service)

`notification-service/src/main/java/com/bank/notificationservice/dto/TransactionMessage.java` :

```java
package com.bank.notificationservice.dto;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.time.LocalDateTime;

@Data
@NoArgsConstructor
@AllArgsConstructor
public class TransactionMessage {
    
    private Long accountId;
    private String accountNumber;
    private String transactionType;
    private Double amount;
    private Double newBalance;
    private Long customerId;
    private LocalDateTime timestamp;
    private String description;
}
```

### Étape 5 : Configuration RabbitMQ

`notification-service/src/main/java/com/bank/notificationservice/config/RabbitMQConfig.java` :

```java
package com.bank.notificationservice.config;

import org.springframework.amqp.support.converter.Jackson2JsonMessageConverter;
import org.springframework.amqp.support.converter.MessageConverter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RabbitMQConfig {

    /**
     * Convertir les messages JSON en objets Java
     */
    @Bean
    public MessageConverter jsonMessageConverter() {
        return new Jackson2JsonMessageConverter();
    }
}
```

### Étape 6 : Consumer (Consommateur)

`notification-service/src/main/java/com/bank/notificationservice/consumer/TransactionConsumer.java` :

```java
package com.bank.notificationservice.consumer;

import com.bank.notificationservice.dto.TransactionMessage;
import com.bank.notificationservice.service.NotificationService;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

/**
 * Consumer RabbitMQ
 * Écoute la queue et traite les messages de transaction
 */
@Component
public class TransactionConsumer {

    private static final Logger logger = LoggerFactory.getLogger(TransactionConsumer.class);

    @Autowired
    private NotificationService notificationService;

    /**
     * Écouter la queue transaction.queue
     * Chaque message reçu déclenche cette méthode
     */
    @RabbitListener(queues = "${rabbitmq.queue.transaction}")
    public void consumeTransaction(TransactionMessage message) {
        logger.info("📥 Message reçu de RabbitMQ: {}", message);
        
        try {
            // Traiter le message
            notificationService.processTransactionNotification(message);
            
            logger.info("✅ Message traité avec succès");
        } catch (Exception e) {
            logger.error("❌ Erreur lors du traitement du message: {}", e.getMessage());
        }
    }
}
```

### Étape 7 : Service de notification

`notification-service/src/main/java/com/bank/notificationservice/service/NotificationService.java` :

```java
package com.bank.notificationservice.service;

import com.bank.notificationservice.dto.TransactionMessage;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;

import java.time.format.DateTimeFormatter;

/**
 * Service de traitement des notifications
 */
@Service
public class NotificationService {

    private static final Logger logger = LoggerFactory.getLogger(NotificationService.class);
    private static final DateTimeFormatter formatter = DateTimeFormatter.ofPattern("dd/MM/yyyy HH:mm:ss");

    /**
     * Traiter une notification de transaction
     */
    public void processTransactionNotification(TransactionMessage message) {
        logger.info("🔔 Traitement de la notification...");
        
        // Construire le message de notification
        String notification = buildNotificationMessage(message);
        
        // Simuler l'envoi d'email/SMS
        sendEmail(message.getCustomerId(), notification);
        sendSMS(message.getCustomerId(), notification);
        
        // Sauvegarder dans la base (optionnel)
        // notificationRepository.save(notification);
    }

    /**
     * Construire le message de notification
     */
    private String buildNotificationMessage(TransactionMessage message) {
        StringBuilder sb = new StringBuilder();
        sb.append("═══════════════════════════════════════\n");
        sb.append("        NOTIFICATION BANCAIRE\n");
        sb.append("═══════════════════════════════════════\n\n");
        
        if ("DEPOT".equals(message.getTransactionType())) {
            sb.append("✅ Dépôt effectué avec succès\n\n");
        } else if ("RETRAIT".equals(message.getTransactionType())) {
            sb.append("💸 Retrait effectué avec succès\n\n");
        }
        
        sb.append("Compte: ").append(message.getAccountNumber()).append("\n");
        sb.append("Montant: ").append(message.getAmount()).append(" FCFA\n");
        sb.append("Nouveau solde: ").append(message.getNewBalance()).append(" FCFA\n");
        sb.append("Date: ").append(message.getTimestamp().format(formatter)).append("\n");
        sb.append("Description: ").append(message.getDescription()).append("\n\n");
        sb.append("═══════════════════════════════════════\n");
        
        return sb.toString();
    }

    /**
     * Simuler l'envoi d'un email
     */
    private void sendEmail(Long customerId, String message) {
        logger.info("📧 Envoi EMAIL au client {}", customerId);
        logger.info("\n{}", message);
        
        // TODO: Intégrer un service d'email (JavaMail, SendGrid, etc.)
    }

    /**
     * Simuler l'envoi d'un SMS
     */
    private void sendSMS(Long customerId, String message) {
        logger.info("📱 Envoi SMS au client {}", customerId);
        
        // Version courte pour SMS
        String sms = String.format(
            "Transaction effectuée. Nouveau solde: %.2f FCFA",
            extractBalanceFromMessage(message)
        );
        
        logger.info("SMS: {}", sms);
        
        // TODO: Intégrer un service SMS (Twilio, etc.)
    }

    private Double extractBalanceFromMessage(String message) {
        // Extraction simple pour la démo
        return 0.0;
    }
}
```

### Étape 8 : Classe principale

`notification-service/src/main/java/com/bank/notificationservice/NotificationServiceApplication.java` :

```java
package com.bank.notificationservice;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class NotificationServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(NotificationServiceApplication.class, args);
        
        System.out.println("\n╔════════════════════════════════════════╗");
        System.out.println("║ Notification Service démarré !         ║");
        System.out.println("║                                        ║");
        System.out.println("║ 📥 En écoute de RabbitMQ...            ║");
        System.out.println("║ Queue: transaction.queue               ║");
        System.out.println("╚════════════════════════════════════════╝\n");
    }
}
```

---

## Partie 4 : Tests du système RabbitMQ

### Étape 1 : Démarrage des services

**Ordre de démarrage** :

```bash
# 1. RabbitMQ (si Docker)
docker start rabbitmq

# 2. Account Service
cd account-service
mvn spring-boot:run

# 3. Notification Service
cd notification-service
mvn spring-boot:run
```

### Étape 2 : Vérifier RabbitMQ

Ouvrir http://localhost:15672
- Username : `admin`
- Password : `admin`

Dans l'onglet **Queues**, vous devriez voir :
- `transaction.queue` (créée automatiquement)

### Étape 3 : Test de dépôt

```bash
# Créer un compte d'abord
curl -X POST http://localhost:8081/api/comptes \
  -H "Content-Type: application/json" \
  -d '{
    "numeroCompte": "CPT001",
    "solde": 10000,
    "typeCompte": "COURANT",
    "clientId": 1
  }'

# Effectuer un dépôt
curl -X POST http://localhost:8081/api/comptes/1/depot?montant=5000
```

**Résultat attendu** :

1. **Account Service** (logs) :
```
📤 Envoi message à RabbitMQ: TransactionMessage(accountId=1, ...)
✅ Message envoyé avec succès
```

2. **RabbitMQ** (interface web) :
   - Le message apparaît brièvement dans la queue
   - Puis est consommé immédiatement

3. **Notification Service** (logs) :
```
📥 Message reçu de RabbitMQ: TransactionMessage(...)
🔔 Traitement de la notification...
📧 Envoi EMAIL au client 1
═══════════════════════════════════════
        NOTIFICATION BANCAIRE
═══════════════════════════════════════

✅ Dépôt effectué avec succès

Compte: CPT001
Montant: 5000.0 FCFA
Nouveau solde: 15000.0 FCFA
Date: 11/02/2024 10:30:15
Description: Dépôt de 5000.0 FCFA

═══════════════════════════════════════
📱 Envoi SMS au client 1
✅ Message traité avec succès
```

### Étape 4 : Test de retrait

```bash
curl -X POST http://localhost:8081/api/comptes/1/retrait?montant=3000
```

Même processus, avec notification de retrait.

### Étape 5 : Tester la résilience

**Scénario** : Notification Service est down

```bash
# 1. Arrêter Notification Service (Ctrl+C)

# 2. Faire un dépôt
curl -X POST http://localhost:8081/api/comptes/1/depot?montant=2000

# 3. Vérifier RabbitMQ
# → Le message est dans la queue (non consommé)

# 4. Redémarrer Notification Service
mvn spring-boot:run

# → Le message est automatiquement consommé !
# → La notification est envoyée
```

**C'est la MAGIE de RabbitMQ** : Les messages sont persistés et consommés même après un redémarrage ! 🎉

---

## Partie 5 : Patterns avancés RabbitMQ

### Pattern 1 : Dead Letter Queue (DLQ)

Pour gérer les messages en erreur :

```java
@Bean
public Queue transactionQueue() {
    return QueueBuilder.durable(queueName)
        .withArgument("x-dead-letter-exchange", "dlx.exchange")
        .withArgument("x-dead-letter-routing-key", "dlx.routing.key")
        .build();
}

@Bean
public Queue deadLetterQueue() {
    return new Queue("transaction.dlq", true);
}
```

### Pattern 2 : Retry avec TTL

```java
@RabbitListener(queues = "${rabbitmq.queue.transaction}")
public void consumeTransaction(TransactionMessage message) {
    try {
        notificationService.processTransactionNotification(message);
    } catch (Exception e) {
        // Réessayer après 5 secondes
        rabbitTemplate.convertAndSend("retry.exchange", "retry.key", message,
            msg -> {
                msg.getMessageProperties().setExpiration("5000");
                return msg;
            });
    }
}
```

### Pattern 3 : Priorité des messages

```java
@Bean
public Queue priorityQueue() {
    return QueueBuilder.durable(queueName)
        .withArgument("x-max-priority", 10)
        .build();
}

// Envoyer avec priorité
rabbitTemplate.convertAndSend(exchange, routingKey, message,
    msg -> {
        msg.getMessageProperties().setPriority(9);  // Haute priorité
        return msg;
    });
```

---

# TP 2 : Apache Kafka - E-commerce

**Durée** : 1h30  
**Objectif** : Créer un système de tracking de commandes avec Kafka

## Cas d'usage

```
Scénario : Suivi de commande e-commerce

1. Order Service crée une commande
2. Publie un événement dans Kafka topic "orders"
3. Plusieurs services consomment l'événement :
   - Inventory Service (gestion stock)
   - Shipping Service (expédition)
   - Analytics Service (statistiques)
```

## Architecture

```
┌─────────────────┐
│  Order Service  │ (Producer)
│   (Port 8085)   │
└────────┬────────┘
         │
         │ Publish event
         │ "Order Created"
         ▼
┌─────────────────────────────┐
│        Kafka Cluster        │
│      (Port 9092)            │
│   Topic: orders             │
│   Partitions: 3             │
└──────────┬──────────────────┘
           │
           ├─────────────┬──────────────┬───────────────┐
           │             │              │               │
           ▼             ▼              ▼               ▼
    ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
    │Inventory │  │ Shipping │  │Analytics │  │  Email   │
    │ Service  │  │ Service  │  │ Service  │  │ Service  │
    └──────────┘  └──────────┘  └──────────┘  └──────────┘
    (Consumer 1)  (Consumer 2)  (Consumer 3)  (Consumer 4)
```

---

## Partie 1 : Installation de Kafka

### Méthode 1 : Docker Compose (Recommandé)

Créer `docker-compose.yml` :

```yaml
version: '3.8'

services:
  # Zookeeper (requis pour Kafka)
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    container_name: zookeeper
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    ports:
      - "2181:2181"

  # Kafka Broker
  kafka:
    image: confluentinc/cp-kafka:7.5.0
    container_name: kafka
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1

  # Kafka UI (interface graphique)
  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    container_name: kafka-ui
    depends_on:
      - kafka
    ports:
      - "8090:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9092
```

**Démarrer** :

```bash
docker-compose up -d

# Vérifier les logs
docker-compose logs -f kafka
```

**Interface Kafka UI** : http://localhost:8090

### Méthode 2 : Installation locale

**Télécharger Kafka** : https://kafka.apache.org/downloads

```bash
# Décompresser
tar -xzf kafka_2.13-3.6.0.tgz
cd kafka_2.13-3.6.0

# Démarrer Zookeeper
bin/zookeeper-server-start.sh config/zookeeper.properties

# Démarrer Kafka (nouveau terminal)
bin/kafka-server-start.sh config/server.properties
```

---

## Partie 2 : Créer Order Service (Producer)

### Étape 1 : Créer le projet

**Spring Initializr** :
- **Group** : `sn.ec2tl.ecommerce`
- **Artifact** : `order-service`
- **Dependencies** :
  - Spring Web
  - Spring for Apache Kafka
  - Spring Data JPA
  - H2 Database
  - Lombok
  - Validation

### Étape 2 : pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.4.1</version>
        <relativePath/>
    </parent>
    
    <groupId>sn.ec2tl.ecommerce</groupId>
    <artifactId>order-service</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>order-service</name>
    <description>Service de gestion des commandes</description>
    
    <properties>
        <java.version>17</java.version>
    </properties>
    
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        
        <!-- Kafka -->
        <dependency>
            <groupId>org.springframework.kafka</groupId>
            <artifactId>spring-kafka</artifactId>
        </dependency>
        
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>
        
        <dependency>
            <groupId>com.h2database</groupId>
            <artifactId>h2</artifactId>
            <scope>runtime</scope>
        </dependency>
        
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        
        <dependency>
            <groupId>org.springframework.kafka</groupId>
            <artifactId>spring-kafka-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
    
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

### Étape 3 : Configuration

`order-service/src/main/resources/application.properties` :

```properties
# Configuration du serveur
spring.application.name=order-service
server.port=8085

# Configuration H2
spring.datasource.url=jdbc:h2:mem:orderdb
spring.datasource.driverClassName=org.h2.Driver
spring.h2.console.enabled=true
spring.jpa.hibernate.ddl-auto=create-drop

# Configuration Kafka
spring.kafka.bootstrap-servers=localhost:9092

# Producer configuration
spring.kafka.producer.key-serializer=org.apache.kafka.common.serialization.StringSerializer
spring.kafka.producer.value-serializer=org.springframework.kafka.support.serializer.JsonSerializer

# Topic name
kafka.topic.order=orders
```

### Étape 4 : Modèle Order

`order-service/src/main/java/com/ecommerce/orderservice/model/Order.java` :

```java
package com.ecommerce.orderservice.model;

import jakarta.persistence.*;
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.time.LocalDateTime;

@Entity
@Table(name = "orders")
@Data
@NoArgsConstructor
@AllArgsConstructor
public class Order {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false, unique = true)
    private String orderNumber;
    
    @Column(nullable = false)
    private Long customerId;
    
    @Column(nullable = false)
    private String productName;
    
    @Column(nullable = false)
    private Integer quantity;
    
    @Column(nullable = false)
    private Double price;
    
    @Column(nullable = false)
    private Double totalAmount;
    
    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private OrderStatus status;
    
    @Column(nullable = false)
    private LocalDateTime orderDate;
    
    private String shippingAddress;
    
    public enum OrderStatus {
        PENDING,
        CONFIRMED,
        PROCESSING,
        SHIPPED,
        DELIVERED,
        CANCELLED
    }
}
```

### Étape 5 : Event DTO

`order-service/src/main/java/com/ecommerce/orderservice/event/OrderEvent.java` :

```java
package com.ecommerce.orderservice.event;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.time.LocalDateTime;

/**
 * Événement publié dans Kafka lors de la création d'une commande
 */
@Data
@NoArgsConstructor
@AllArgsConstructor
public class OrderEvent {
    
    private String eventId;
    private String eventType;  // ORDER_CREATED, ORDER_UPDATED, ORDER_CANCELLED
    private LocalDateTime timestamp;
    
    // Détails de la commande
    private Long orderId;
    private String orderNumber;
    private Long customerId;
    private String productName;
    private Integer quantity;
    private Double totalAmount;
    private String status;
    private String shippingAddress;
}
```

### Étape 6 : Configuration Kafka

`order-service/src/main/java/com/ecommerce/orderservice/config/KafkaProducerConfig.java` :

```java
package com.ecommerce.orderservice.config;

import com.ecommerce.orderservice.event.OrderEvent;
import org.apache.kafka.clients.admin.NewTopic;
import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.common.serialization.StringSerializer;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.config.TopicBuilder;
import org.springframework.kafka.core.DefaultKafkaProducerFactory;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.kafka.core.ProducerFactory;
import org.springframework.kafka.support.serializer.JsonSerializer;

import java.util.HashMap;
import java.util.Map;

/**
 * Configuration Kafka Producer
 */
@Configuration
public class KafkaProducerConfig {

    @Value("${spring.kafka.bootstrap-servers}")
    private String bootstrapServers;

    @Value("${kafka.topic.order}")
    private String orderTopic;

    /**
     * Créer le topic automatiquement
     * 3 partitions pour paralléliser le traitement
     * Replication factor = 1 (single broker)
     */
    @Bean
    public NewTopic orderTopic() {
        return TopicBuilder.name(orderTopic)
            .partitions(3)
            .replicas(1)
            .build();
    }

    /**
     * Configuration du Producer
     */
    @Bean
    public Map<String, Object> producerConfigs() {
        Map<String, Object> props = new HashMap<>();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        return props;
    }

    /**
     * Producer Factory
     */
    @Bean
    public ProducerFactory<String, OrderEvent> producerFactory() {
        return new DefaultKafkaProducerFactory<>(producerConfigs());
    }

    /**
     * Kafka Template pour envoyer des messages
     */
    @Bean
    public KafkaTemplate<String, OrderEvent> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }
}
```

### Étape 7 : Kafka Producer Service

`order-service/src/main/java/com/ecommerce/orderservice/service/KafkaProducerService.java` :

```java
package com.ecommerce.orderservice.service;

import com.ecommerce.orderservice.event.OrderEvent;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.kafka.support.SendResult;
import org.springframework.stereotype.Service;

import java.util.concurrent.CompletableFuture;

/**
 * Service pour publier des événements dans Kafka
 */
@Service
public class KafkaProducerService {

    private static final Logger logger = LoggerFactory.getLogger(KafkaProducerService.class);

    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;

    @Value("${kafka.topic.order}")
    private String orderTopic;

    public KafkaProducerService(KafkaTemplate<String, OrderEvent> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    /**
     * Publier un événement de commande dans Kafka
     */
    public void publishOrderEvent(OrderEvent event) {
        logger.info("📤 Publication événement dans Kafka: {}", event.getEventType());
        
        CompletableFuture<SendResult<String, OrderEvent>> future = 
            kafkaTemplate.send(orderTopic, event.getOrderNumber(), event);
        
        future.whenComplete((result, ex) -> {
            if (ex == null) {
                logger.info("✅ Événement publié avec succès sur partition {} avec offset {}",
                    result.getRecordMetadata().partition(),
                    result.getRecordMetadata().offset());
            } else {
                logger.error("❌ Échec de publication de l'événement: {}", ex.getMessage());
            }
        });
    }
}
```

### Étape 8 : Order Service

`order-service/src/main/java/com/ecommerce/orderservice/service/OrderService.java` :

```java
package com.ecommerce.orderservice.service;

import com.ecommerce.orderservice.event.OrderEvent;
import com.ecommerce.orderservice.model.Order;
import com.ecommerce.orderservice.repository.OrderRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.LocalDateTime;
import java.util.List;
import java.util.UUID;

@Service
@Transactional
public class OrderService {

    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private KafkaProducerService kafkaProducerService;

    /**
     * Créer une nouvelle commande
     */
    public Order createOrder(Order order) {
        // Générer un numéro de commande unique
        order.setOrderNumber("ORD-" + UUID.randomUUID().toString().substring(0, 8).toUpperCase());
        order.setOrderDate(LocalDateTime.now());
        order.setStatus(Order.OrderStatus.PENDING);
        
        // Calculer le montant total
        order.setTotalAmount(order.getPrice() * order.getQuantity());
        
        // Sauvegarder en base
        Order savedOrder = orderRepository.save(order);
        
        // ============================================
        // Publier événement dans Kafka
        // ============================================
        OrderEvent event = new OrderEvent(
            UUID.randomUUID().toString(),
            "ORDER_CREATED",
            LocalDateTime.now(),
            savedOrder.getId(),
            savedOrder.getOrderNumber(),
            savedOrder.getCustomerId(),
            savedOrder.getProductName(),
            savedOrder.getQuantity(),
            savedOrder.getTotalAmount(),
            savedOrder.getStatus().toString(),
            savedOrder.getShippingAddress()
        );
        
        kafkaProducerService.publishOrderEvent(event);
        
        return savedOrder;
    }

    /**
     * Récupérer toutes les commandes
     */
    public List<Order> getAllOrders() {
        return orderRepository.findAll();
    }

    /**
     * Récupérer une commande par ID
     */
    public Order getOrderById(Long id) {
        return orderRepository.findById(id)
            .orElseThrow(() -> new RuntimeException("Commande non trouvée: " + id));
    }

    /**
     * Mettre à jour le statut d'une commande
     */
    public Order updateOrderStatus(Long id, Order.OrderStatus newStatus) {
        Order order = getOrderById(id);
        order.setStatus(newStatus);
        Order updatedOrder = orderRepository.save(order);
        
        // Publier événement de mise à jour
        OrderEvent event = new OrderEvent(
            UUID.randomUUID().toString(),
            "ORDER_UPDATED",
            LocalDateTime.now(),
            updatedOrder.getId(),
            updatedOrder.getOrderNumber(),
            updatedOrder.getCustomerId(),
            updatedOrder.getProductName(),
            updatedOrder.getQuantity(),
            updatedOrder.getTotalAmount(),
            updatedOrder.getStatus().toString(),
            updatedOrder.getShippingAddress()
        );
        
        kafkaProducerService.publishOrderEvent(event);
        
        return updatedOrder;
    }
}
```

### Étape 9 : Repository et Controller

`OrderRepository.java` :

```java
package com.ecommerce.orderservice.repository;

import com.ecommerce.orderservice.model.Order;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
}
```

`OrderController.java` :

```java
package com.ecommerce.orderservice.controller;

import com.ecommerce.orderservice.model.Order;
import com.ecommerce.orderservice.service.OrderService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @Autowired
    private OrderService orderService;

    @PostMapping
    public ResponseEntity<Order> createOrder(@RequestBody Order order) {
        Order createdOrder = orderService.createOrder(order);
        return ResponseEntity.status(HttpStatus.CREATED).body(createdOrder);
    }

    @GetMapping
    public ResponseEntity<List<Order>> getAllOrders() {
        return ResponseEntity.ok(orderService.getAllOrders());
    }

    @GetMapping("/{id}")
    public ResponseEntity<Order> getOrder(@PathVariable Long id) {
        return ResponseEntity.ok(orderService.getOrderById(id));
    }

    @PutMapping("/{id}/status")
    public ResponseEntity<Order> updateStatus(
        @PathVariable Long id,
        @RequestParam Order.OrderStatus status
    ) {
        return ResponseEntity.ok(orderService.updateOrderStatus(id, status));
    }
}
```

---

## Partie 3 : Créer Inventory Service (Consumer)

### Étape 1 : Créer le projet

Même structure que Order Service, mais port 8086.

### Étape 2 : Configuration

`inventory-service/src/main/resources/application.properties` :

```properties
spring.application.name=inventory-service
server.port=8086

# Kafka Consumer
spring.kafka.bootstrap-servers=localhost:9092
spring.kafka.consumer.group-id=inventory-group
spring.kafka.consumer.key-deserializer=org.apache.kafka.common.serialization.StringDeserializer
spring.kafka.consumer.value-deserializer=org.springframework.kafka.support.serializer.JsonDeserializer
spring.kafka.consumer.properties.spring.json.trusted.packages=*

kafka.topic.order=orders
```

### Étape 3 : Event DTO (identique)

Copier `OrderEvent.java` depuis order-service.

### Étape 4 : Configuration Kafka Consumer

`KafkaConsumerConfig.java` :

```java
package com.ecommerce.inventoryservice.config;

import com.ecommerce.inventoryservice.event.OrderEvent;
import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.common.serialization.StringDeserializer;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.annotation.EnableKafka;
import org.springframework.kafka.config.ConcurrentKafkaListenerContainerFactory;
import org.springframework.kafka.core.ConsumerFactory;
import org.springframework.kafka.core.DefaultKafkaConsumerFactory;
import org.springframework.kafka.support.serializer.JsonDeserializer;

import java.util.HashMap;
import java.util.Map;

@EnableKafka
@Configuration
public class KafkaConsumerConfig {

    @Value("${spring.kafka.bootstrap-servers}")
    private String bootstrapServers;

    @Value("${spring.kafka.consumer.group-id}")
    private String groupId;

    @Bean
    public ConsumerFactory<String, OrderEvent> consumerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        props.put(ConsumerConfig.GROUP_ID_CONFIG, groupId);
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, JsonDeserializer.class);
        props.put(JsonDeserializer.TRUSTED_PACKAGES, "*");
        props.put(JsonDeserializer.VALUE_DEFAULT_TYPE, OrderEvent.class.getName());
        
        return new DefaultKafkaConsumerFactory<>(props);
    }

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, OrderEvent> kafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, OrderEvent> factory =
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        return factory;
    }
}
```

### Étape 5 : Kafka Consumer

`OrderEventConsumer.java` :

```java
package com.ecommerce.inventoryservice.consumer;

import com.ecommerce.inventoryservice.event.OrderEvent;
import com.ecommerce.inventoryservice.service.InventoryService;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Component;

/**
 * Consumer Kafka pour les événements de commande
 */
@Component
public class OrderEventConsumer {

    private static final Logger logger = LoggerFactory.getLogger(OrderEventConsumer.class);

    @Autowired
    private InventoryService inventoryService;

    /**
     * Écouter le topic "orders"
     * Chaque événement déclenche cette méthode
     */
    @KafkaListener(
        topics = "${kafka.topic.order}",
        groupId = "${spring.kafka.consumer.group-id}"
    )
    public void consume(OrderEvent event) {
        logger.info("📥 Événement reçu de Kafka: {} - Order: {}",
            event.getEventType(), event.getOrderNumber());
        
        try {
            // Traiter selon le type d'événement
            switch (event.getEventType()) {
                case "ORDER_CREATED":
                    inventoryService.processOrderCreated(event);
                    break;
                case "ORDER_UPDATED":
                    inventoryService.processOrderUpdated(event);
                    break;
                case "ORDER_CANCELLED":
                    inventoryService.processOrderCancelled(event);
                    break;
                default:
                    logger.warn("⚠️ Type d'événement inconnu: {}", event.getEventType());
            }
            
            logger.info("✅ Événement traité avec succès");
            
        } catch (Exception e) {
            logger.error("❌ Erreur lors du traitement: {}", e.getMessage());
        }
    }
}
```

### Étape 6 : Inventory Service

`InventoryService.java` :

```java
package com.ecommerce.inventoryservice.service;

import com.ecommerce.inventoryservice.event.OrderEvent;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;

/**
 * Service de gestion du stock
 */
@Service
public class InventoryService {

    private static final Logger logger = LoggerFactory.getLogger(InventoryService.class);

    /**
     * Traiter la création d'une commande
     */
    public void processOrderCreated(OrderEvent event) {
        logger.info("📦 Traitement création commande: {}", event.getOrderNumber());
        logger.info("   Produit: {}", event.getProductName());
        logger.info("   Quantité: {}", event.getQuantity());
        
        // Vérifier le stock
        boolean stockAvailable = checkStock(event.getProductName(), event.getQuantity());
        
        if (stockAvailable) {
            // Réserver le stock
            reserveStock(event.getProductName(), event.getQuantity());
            logger.info("✅ Stock réservé pour la commande {}", event.getOrderNumber());
        } else {
            logger.warn("⚠️ Stock insuffisant pour {}", event.getProductName());
            // TODO: Publier événement STOCK_UNAVAILABLE
        }
    }

    /**
     * Traiter la mise à jour d'une commande
     */
    public void processOrderUpdated(OrderEvent event) {
        logger.info("🔄 Traitement mise à jour commande: {}", event.getOrderNumber());
        logger.info("   Nouveau statut: {}", event.getStatus());
        
        if ("SHIPPED".equals(event.getStatus())) {
            // Déduire définitivement du stock
            deductStock(event.getProductName(), event.getQuantity());
            logger.info("✅ Stock déduit pour commande expédiée");
        }
    }

    /**
     * Traiter l'annulation d'une commande
     */
    public void processOrderCancelled(OrderEvent event) {
        logger.info("❌ Traitement annulation commande: {}", event.getOrderNumber());
        
        // Libérer le stock réservé
        releaseStock(event.getProductName(), event.getQuantity());
        logger.info("✅ Stock libéré pour commande annulée");
    }

    // ============================================
    // Méthodes de simulation (à implémenter avec vraie DB)
    // ============================================
    
    private boolean checkStock(String product, int quantity) {
        // TODO: Vérifier en base de données
        logger.info("   Vérification stock: {} x {}", product, quantity);
        return true;  // Simulation: toujours disponible
    }
    
    private void reserveStock(String product, int quantity) {
        // TODO: Réserver en base de données
        logger.info("   Réservation: {} x {}", product, quantity);
    }
    
    private void deductStock(String product, int quantity) {
        // TODO: Déduire en base de données
        logger.info("   Déduction stock: {} x {}", product, quantity);
    }
    
    private void releaseStock(String product, int quantity) {
        // TODO: Libérer en base de données
        logger.info("   Libération stock: {} x {}", product, quantity);
    }
}
```

---

## Partie 4 : Tests du système Kafka

### Étape 1 : Démarrage

```bash
# 1. Kafka (Docker Compose)
docker-compose up -d

# 2. Order Service
cd order-service
mvn spring-boot:run

# 3. Inventory Service
cd inventory-service
mvn spring-boot:run
```

### Étape 2 : Vérifier Kafka UI

Ouvrir http://localhost:8090

Vous devriez voir :
- Topic **"orders"** créé automatiquement
- 3 partitions
- Consumer group **"inventory-group"**

### Étape 3 : Créer une commande

```bash
curl -X POST http://localhost:8085/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "customerId": 1,
    "productName": "MacBook Pro 14",
    "quantity": 2,
    "price": 2000000,
    "shippingAddress": "Dakar, Sénégal"
  }'
```

**Résultat attendu** :

1. **Order Service** (logs) :
```
📤 Publication événement dans Kafka: ORDER_CREATED
✅ Événement publié avec succès sur partition 1 avec offset 0
```

2. **Kafka UI** :
   - Messages count: 1
   - Partition: 0, 1 ou 2 (distribué)

3. **Inventory Service** (logs) :
```
📥 Événement reçu de Kafka: ORDER_CREATED - Order: ORD-ABCD1234
📦 Traitement création commande: ORD-ABCD1234
   Produit: MacBook Pro 14
   Quantité: 2
   Vérification stock: MacBook Pro 14 x 2
   Réservation: MacBook Pro 14 x 2
✅ Stock réservé pour la commande ORD-ABCD1234
✅ Événement traité avec succès
```

### Étape 4 : Test de parallélisme

**Démarrer 2 instances d'Inventory Service** :

```bash
# Terminal 1
mvn spring-boot:run

# Terminal 2
mvn spring-boot:run -Dspring-boot.run.arguments="--server.port=8087"
```

**Créer 10 commandes** :

```bash
for i in {1..10}; do
  curl -X POST http://localhost:8085/api/orders \
    -H "Content-Type: application/json" \
    -d "{
      \"customerId\": $i,
      \"productName\": \"Produit $i\",
      \"quantity\": 1,
      \"price\": 10000,
      \"shippingAddress\": \"Adresse $i\"
    }"
done
```

**Observer** : Les messages sont répartis entre les 2 instances grâce aux partitions ! 🚀

---

## Partie 5 : Ajouter d'autres consumers

### Shipping Service (simple)

```java
@Component
public class OrderEventConsumer {
    
    @KafkaListener(topics = "orders", groupId = "shipping-group")
    public void consume(OrderEvent event) {
        if ("ORDER_CREATED".equals(event.getEventType())) {
            logger.info("📮 Préparation expédition pour: {}", event.getOrderNumber());
            logger.info("   Adresse: {}", event.getShippingAddress());
            // TODO: Créer étiquette d'expédition, contacter transporteur
        }
    }
}
```

### Analytics Service (simple)

```java
@Component
public class OrderEventConsumer {
    
    @KafkaListener(topics = "orders", groupId = "analytics-group")
    public void consume(OrderEvent event) {
        logger.info("📊 Enregistrement statistique");
        logger.info("   Montant: {} FCFA", event.getTotalAmount());
        logger.info("   Produit: {}", event.getProductName());
        // TODO: Sauvegarder dans data warehouse pour BI
    }
}
```

---

# Comparaison RabbitMQ vs Kafka

## Tableau comparatif

| Critère | RabbitMQ | Apache Kafka |
|---------|----------|--------------|
| **Type** | Message Broker | Event Streaming Platform |
| **Pattern** | Point-to-Point, Pub/Sub | Pub/Sub uniquement |
| **Ordre des messages** | Garanti par queue | Garanti par partition |
| **Persistance** | Optionnelle | Toujours (par défaut) |
| **Rétention** | Jusqu'à consommation | Durée configurable (ex: 7 jours) |
| **Performance** | Millions msg/sec | Millions msg/sec (plus) |
| **Latence** | Très faible (ms) | Faible (ms-sec) |
| **Scalabilité** | Bonne | Excellente (horizontale) |
| **Complexité** | Simple | Plus complexe |
| **Cas d'usage** | Tasks asynchrones, Notifications | Event sourcing, Log aggregation, Streaming |
| **Consommation** | Destructive (msg supprimé) | Non-destructive (msg gardé) |
| **Replay** | ❌ Non | ✅ Oui |

## Quand utiliser RabbitMQ ?

✅ **Bon pour** :
- Notifications en temps réel
- Task queues (workers)
- Communication simple entre services
- Besoin de routing complexe
- Faible latence critique

**Exemples** :
- Envoi d'emails
- Traitement d'images
- Notifications push
- Jobs asynchrones

## Quand utiliser Kafka ?

✅ **Bon pour** :
- Event sourcing
- Log aggregation
- Streaming de données
- Analytics en temps réel
- Besoin de replay
- Très haut débit

**Exemples** :
- Tracking utilisateur
- IoT sensors data
- Financial transactions
- Audit logs
- Real-time analytics

---

# Exercices et évaluation

## Exercice 1 : RabbitMQ - Multi-consumers

**Énoncé** : Ajouter un Email Service qui consomme les messages de transaction et envoie des emails.

**Points** : 5

**Critères** :
- Service créé et démarré (1 pt)
- Consumer RabbitMQ configuré (2 pts)
- Traitement du message (1 pt)
- Logs appropriés (1 pt)

## Exercice 2 : Kafka - Nouveau topic

**Énoncé** : Créer un topic "inventory-alerts" pour les alertes de stock faible. Inventory Service publie, Analytics Service consomme.

**Points** : 7

**Critères** :
- Topic créé (1 pt)
- Producer configuré (2 pts)
- Consumer configuré (2 pts)
- Événement publié correctement (1 pt)
- Tests réussis (1 pt)

## Exercice 3 : Pattern avancé - Saga

**Énoncé** : Implémenter un pattern Saga pour gérer une commande :
1. Order Service crée commande
2. Inventory Service réserve stock
3. Payment Service traite paiement
4. Si échec → Compensation (annuler réservation)

**Points** : 10

**Bonus** : Dead Letter Queue (+2 pts)

## Mini-projet : Système complet

**Énoncé** : Créer un système de gestion de restaurant avec :

**Services** :
- Order Service (commandes)
- Kitchen Service (préparation)
- Delivery Service (livraison)
- Notification Service (SMS client)

**Technologies** :
- RabbitMQ pour notifications
- Kafka pour tracking des commandes

**Livrables** :
- 4 microservices fonctionnels
- Architecture documentée
- Tests de bout en bout

**Barème** : 20 points

---

## 📚 Ressources

### RabbitMQ
- Documentation : https://www.rabbitmq.com/documentation.html
- Spring AMQP : https://spring.io/projects/spring-amqp

### Kafka
- Documentation : https://kafka.apache.org/documentation/
- Spring Kafka : https://spring.io/projects/spring-kafka

### Patterns
- Enterprise Integration Patterns : https://www.enterpriseintegrationpatterns.com/

---

**Fin du TP - Bon courage ! 🚀**
