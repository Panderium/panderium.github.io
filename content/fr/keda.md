---
title: "Keda: l’autoscaler qui va sauver votre production"
date: 2025-03-14T20:15:23+01:00
draft: true
categories: [kubernetes]
tags: [Kubernetes, scaling]
---

Vous avez soigneusement conçu une architecture distribuée basée sur des microservices, convaincu que cela assurerait la scalabilité de vos applications. Mais voilà, après la mise en ligne de votre article phare les commandes s’enchainent mais la performance se fait attendre… jusqu’au blackout... Ce qui devait être un grand moment de succès se transforme en véritable casse-tête, votre infrastructure est été incapable de gérer l’arrivé massive d’utilisateur sur votre site. Ce scénario aurait sans doute pu être évité si vous aviez mise en place de **l’autoscaling** événementiel avec **Keda**.

## Les limites de l’autoscaling “traditionnelle”

**L'autoscaling** est désormais essentiel pour une gestion dynamique des ressources et pour garantir la performance des applications. Cependant, **l'autoscaling** traditionnel reposant sur des métriques comme l'utilisation du CPU ou de la mémoire, peut ne pas être suffisant lorsque la charge des applications est influencée par des facteurs externes. S’appuyer uniquement sur ces métriques peut empêcher une application de répondre efficacement à l’augmentation de la demande des utilisateurs. Dans ces situations, il devient crucial d’adapter le **scaling** des applications en fonction de **métriques** plus adaptées capables d’anticiper au mieux les besoins des utilisateurs. Par exemple, une application qui réagit à des événements spécifiques — tels que des messages dans une file d'attente ou des requêtes provenant de services externes — nécessitera un **autoscaling** plus flexible et réactif afin de répondre rapidement aux variations de la demande.

C’est ici qu’intervient [**KEDA**](https://keda.sh/) (Kubernetes Event-Driven Autoscaling), un outil pour **Kubernetes** spécialement conçu afin **d'autoscaler** en fonction **d’événements**. KEDA permet à Kubernetes de réagir de manière proactive aux événements qui affectent les applications, comme l’arrivée de nouveaux messages, l’ajout de données dans une base de données ou des métriques provenant du cloud, par exemple. Cela permet d'ajuster la capacité des *pods* en temps réel en fonction de la demande, et ainsi d’éviter la surcharge du système ou le gaspillage de ressources. En d'autres termes, **KEDA** vous aide à garantir que vos applications soient toujours dimensionnées correctement, en fonction des événements qu'elles rencontrent. Cet article vous montrera comment installer et configurer KEDA sur Kubernetes pour améliorer **l'autoscaling** dans vos environnements de production.

## De la théorie à la pratique

### A. Présentation de l’outil

Si vous êtes familier avec Kubernetes, vous avez probablement déjà utilisé les Horizontal Pod Autoscalers (HPA). Keda va plus loin en ajoutant à Kubernetes un ensemble de Custom Resource Definitions (CRD), permettant d’étendre les capacités d’autoscaling au-delà de ce que propose le HPA. En fait, Keda agit comme un serveur de métriques qui fournit des données au HPA (en utilisant les APIs *custom.metrics.k8s.io* et *external.metrics.k8s.io* fournis par Kubernetes). Ces métriques sont collectées via des [scalers](https://keda.sh/docs/2.16/scalers/), qui sont utilisés dans des ressources comme les [ScaledObjects](https://keda.sh/docs/2.16/concepts/scaling-deployments/) (pour les Deployments, StatefulSets et ressources personnalisées) ou les [ScaledJobs](https://keda.sh/docs/2.16/concepts/scaling-jobs/) (pour les Jobs).

Un des grands avantages de **Keda** est sa capacité à mettre un *Deployment* à l'échelle de zéro, ce qui est particulièrement utile dans une optique de **FinOps** et de **GreenIT** 😊.

Pour en savoir plus sur le fonctionnement interne de Keda je vous invite à [visiter cette documentation](https://keda.sh/docs/2.16/concepts/).

### B. HPA vs Keda

| **Critère** | **Autoscaling basé sur la charge (HPA)** | **Autoscaling événementiel (Keda)** |
| --- | --- | --- |
| **Métriques utilisées** 
 
 | CPU, mémoire, ou toute autre métrique définie par l'utilisateur (mais souvent limitée à celles disponibles sur les pods). | Toute métrique ou événement provenant de sources internes ou externes comme RabbitMQ, Kafka, Azure Service Bus, etc. |
| **Réactivité** 
 
 | Moins réactif aux événements en temps réel. L'autoscaling se déclenche après un certain seuil d'utilisation de ressources. 
 
 | Très réactif aux événements en temps réel, permettant un ajustement immédiat en fonction de la demande. |
| **Flexibilité** 
 
 | Limitée à des métriques internes et bien définies (comme CPU/mémoire).

Implémenter soit même des métriques customs peut s’avérer périlleux. 
 | Très flexible, permet d’intégrer une grande variété de sources et de types d’événements pour déclencher l’autoscaling. |
| **Gestion du scaling à zéro** | Impossible, il reste toujours au minimum un pod. | Keda permet de scaler à zéro, utile dans des scénarios où des ressources qui doivent être libérées lorsque la demande est nulle. |
| **Prise en charge des événements externes** | Nécessite une implémentation de l’api *external.metrics.k8s.io* | Prise en charge native des événements externes via des scalers (ex : RabbitMQ, Kafka, etc.). |

### C. Mise en pratique

Pour tester son fonctionnement, j’ai effectué un PoC mettant en œuvre un *ScaledJob* écoutant une queue d’un cluster **RabbitMQ**. N’étant pas expert en programmation, le *job* en question sera un simple serveur Nginx mais libre à vous d’implémenter une application consommant les messages de la queue.

Les prérequis:

- Un cluster Kubernetes
- Helm
- Kubectl

**1. *Installation de RabbitMQ***

Plusieurs opérateurs existent, pour des raisons de simplicité j’ai pris celui disponible sur le [Github de RabbitMQ.](https://github.com/rabbitmq/cluster-operator)

**a. Installation de l’opérateur**

kubectl apply -f [https://github.com/rabbitmq/clusteroperator/releases/latest/download/cluster-operator.yml](https://github.com/rabbitmq/cluster-operator/releases/latest/download/cluster-operator.yml)

**b. Installation du Cluster**

Le manifeste suivant crée un cluster **RabbitMQ** dans un nouveau *NameSpace*. Le *ConfigMap* permet d’injecter une configuration créant une queue au démarrage de **RabbitMQ**.

```yaml
apiVersion: v1
kind: namespace
metadata:
  name: rabbitmq-cluster
---
apiVersion: rabbitmq.com/v1beta1
kind: RabbitmqCluster
metadata:
  name: rmq-cluster
  namespace: rabbitmq-cluster
spec:
  resources:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: 800m
      memory: 1Gi
  override:
    statefulSet:
      spec:
        template:
          spec:
            containers:
            - name: rabbitmq
              volumeMounts:
              - mountPath: /etc/rabbitmq/definitions.json
                subPath: definitions.json
                name: definitions
            volumes:
            - name: definitions
              configMap:
                name: rmq-cluster-definitions
  rabbitmq:
    additionalConfig: |
      load_definitions = /etc/rabbitmq/definitions.json
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: rmq-cluster-definitions
  namespace: rabbitmq-cluster
data:
  definitions.json: |
    {
      "vhosts": [
        {
          "name": "/"
        }
      ],
      "topic_permissions": [],
      "parameters": [],
      "queues": [
        {
          "name": "keda",
          "vhost": "/",
          "durable": true,
          "auto_delete": false,
          "arguments": {
            "x-queue-type": "classic"
          }
        }
      ]
    }
```

**c. Ajout d’un utilisateur**

Afin de permettre à **Keda** de se connecter au cluster j’ai créé un utilisateur auquel je donne les droits d’administration.

```bash
// après avoir ouvert un shell sur le cluster
$ rabbitmqctl add_user keda keda
$ rabbitmqctl set_user_tags keda administrator
$ rabbitmqctl set_permissions -p / keda ".*" ".*" ".*"
```

***2. Installer Keda***

**a. Installation de l’opérateur**

Rien de plus simple en suivant la documentation [Deploying KEDA | KEDA.](https://keda.sh/docs/2.16/deploy/)

```bash
helm repo add kedacore https://kedacore.github.io/charts
helm repo update
helm install keda kedacore/keda --namespace keda -–create-namespace
```

**b. Déploiement d’un *ScaledJob***

Ajout du *ScaledJob* avec un trigger utilisant le [scaler RabbitMQ](https://keda.sh/docs/2.16/scalers/rabbitmq-queue/).

Notez que j’utilise un secret que j’importe dans le *ScaledJob* pour pouvoir authentifier le *Scaler* à mon cluster **RabbitMQ**. Bien que cela soit fonctionnel **Keda** propose une gestion bien plus poussée de l’authentification avec des ressources de [type *TriggerAuthentication* et *ClusterTriggerAuthentication.*](https://keda.sh/docs/2.16/concepts/authentication/#re-use-credentials-and-delegate-auth-with-triggerauthentication)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: rabbitmq
  namespace: default
data:
  RABBITMQ_HOST: YW1xcDovL3JtcS1jbHVzdGVyLnJhYmJpdG1xLWNsdXN0ZXI6NTY3Mi92aG9zdA== # amqp://rmq-cluster.rabbitmq-cluster:5672/vhost
  RABBITMQ_USERNAME: a2VkYQ== # keda en base64
  RABBITMQ_PASSWORD: a2VkYQ== # keda en base64
---
apiVersion: keda.sh/v1alpha1
kind: ScaledJob
metadata:
  name: nginx-keda
  namespace: default
spec:
  jobTargetRef:
    template:
      spec:
        containers:
        - name: nginx
          image: nginx
          imagePullPolicy: Always
          command: ["/usr/bin/bash", "-c", "sleep 30"]
          envFrom:
            - secretRef:
                name: rabbitmq
        restartPolicy: Never
    backoffLimit: 4  
  triggers:
  - type: rabbitmq
    metadata:
      queueName: keda
      mode: QueueLength
      value  : '1'
      vhostName: /
      hostFromEnv: RABBITMQ_HOST
      usernameFromEnv: RABBITMQ_USERNAME
      passwordFromEnv: RABBITMQ_PASSWORD
      unsafeSsl: "true"
```

**3. *Démonstration***

![demo-keda.gif](/static/demo-keda.gif)

## Conclusion

À travers cette démonstration, il est clair que l’implémentation de Keda sur un cluster Kubernetes est relativement simple. En s’appuyant sur le serveur de métriques Kubernetes, l’équipe de Keda nous offre une interface unifiée permettant de mettre en place une stratégie d’autoscaling basée sur des événements. Cette solution légère couvre une large gamme de source à partir desquels on pourrait envisager le scaling. De plus, il est tout à fait possible de [contribuer à l’ajout de nouveaux scalers](https://github.com/kedacore/keda/blob/main/CONTRIBUTING.md#contributing-scalers).

Si vous utilisez déjà Kubernetes et que vous cherchez à améliorer la gestion de la charge de vos applications, Keda mérite clairement votre attention. Avez-vous déjà testé cet outil en production ? N’hésitez pas à partager vos retours d’expérience et vos cas d’usage !

## Et si vous passiez à l’échelle avec ACENSI ?

Chez **ACENSI**, nous accompagnons nos clients dans l’optimisation de leurs infrastructures cloud et Kubernetes. Grâce à notre expertise en **DevOps** nous aidons les entreprises à tirer pleinement parti des solutions comme Keda pour garantir une infrastructure réactive, performante et économique. Vous souhaitez implémenter une stratégie d’autoscaling efficace et adaptée à vos besoins ? **Nos experts sont là pour vous conseiller et vous accompagner**. N’hésitez pas à nous contacter pour en discuter !