# LP25 TD C n°5

## Communication par files de messages

Ce TD a pour objectif l'utilisation de files de messages (Message Queues). Pour celà, nous utiliserons les files de message unix.

Les fonctions utilisées sont décrites ci dessous.

### Création d'une clé unique

Pour permettre l'utilisation de plusieurs MQ par le système, chaque file est associée à une clé unique qui l'identifie. La fonction pour demander au système une clé est `ftok` dont la signature est la suivante :
```c
#include <sys/ipc.h>
key_t ftok(const char *pathname, int proj_id);
```

Le paramètre `pathname` est un chemin vers un fichier pouvant être ouvert. Vous pouvez par exemple utiliser le chemin vers votre programme. Le paramètre `proj_id` est une valeur de votre choix, dont seul l'octet le moins significatif (les valeurs entre 0 et 255) sera utile. La fonction renvoie une clé qui sera utilisée pour accéder à la file de messages.

### Ouverture/Création de la file de messages

Pour créer la file de message, vous utiliserez la fonction `msgget` dont la signature est la suivante :
```c
#include <sys/ipc.h>
int msgget(key_t key, int msgflg);
```

Cette fonction va prendre comme premier paramètre la clé générée par `ftok`, et des flags pour les options d'ouverture de la file de messages. Ces flags définissent notamment les droits d'accès à la file de messages, assortis éventuellement d'une demande de la créer. Par exemple, l'appel suivant crée la file.

```c
key_t my_key = ftok("/path/to/a/file", 25);
int msg_id = msgget(my_key, 0666 | IPC_CREAT);
```

Et ce second appel, l'ouvre depuis un autre processus :

```c
key_t my_key = ftok("/path/to/a/file", 25);
int msg_id = msgget(my_key, 0666);
```

En cas d'erreur, vous obtenez en retour la valeur `-1`, sinon, l'identifiant de la file de messages est une valeur strictement positive.

### Envoyer des données

L'envoi et la réception des données repose sur une structure de données que vous pouvez étendre. La structure minimale est la suivante :

```c
struct msgbuf {
  long mtype; /* message type, must be > 0 */
  char mtext[1]; /* message data */
};
```

Elle doit toujours commencer par un `long` nommé `mtype` et être suivie par un champ de données. Le champ de données peut avoir la taille de votre choix. Par exemple, Imaginons qu'il soit nécessaire de transmettre une structure du type suivant :

```c
typedef struct _task {
	void (*callback)(struct _task *);
	char param1[1024];
	char param2[1024];
} task_t;
```

Vous utiliserez un _wrapper_ pour que votre structure soit de la forme attendue par `msgsnd` :

```c
typedef struct {
	long mtype;
	task_t message;
} task_wrapper_t;
```

À partir de ces définitions, vous enverrez le message avec la fonction `msgsnd` dont le prototype est le suivant :
```c
#include <sys/msg.h>
int msgsnd(int msqid, const void *msgp, size_t msgsz, int msgflg);
```

Cette fonction renvoie 0 en cas de succès. Elle prend en paramètres :

- `msqid` l'identifiant de la file de messages obtenue par `msgget`,
- `msgp` l'adresse de la structure à envoyer. Dans notre cas, ce serait l'adresse d'une variable de type `task_wrapper_t`,
- `msgsz` est la taille du message proprement dit, donc, dans notre cas, la taille de la structure `task_t` (et pas celle de la structure `task_wrapper_t`),
- `msgflg` est un ensemble de flags pouvant spécifier un comportement particulier. Dans le TP et le projet, nous n'utiliserons qu'une valeur de `0` pour un fonctionnement standard de l'envoi des messages.

```c
#include <sys/msg.h>
task_wrapper_t tw = {
	.mtype = 25,
	.message = {
		.callback = a_function,
		.param1 = "abc",
		.param2 = "def",
	},
};
msgsnd(msg_id, &tw, sizeof(task_t), 0);
```

### Recevoir des données

La réception des données se fait par la fonction `msgrcv` dont la signature est :

```c
#include <sys/msg.h>
ssize_t msgrcv(int msqid, void *msgp, size_t msgsz, long msgtyp, int msgflg);
```

Dont les paramètres sont les suivants :

- `msqid` est l'identifiant de la file de messages sur laquelle un message va être lu.
- `msgp` est un pointeur vers la structure où le message reçu sera stocké.
- `msgsz` est la taille du message proprement dit (sans le membre `mtype`).
- `msgtyp` est la valeur du `mtype` dont on recevra un message (les messages envoyés doivent avoir spécifié cette valeur pour le champ `mtype`). Si `msgtyp` vaut `0`, alors les messages seront lus quelque soit la valeur du champ `mtype`.

### Fermer la file de messages

La destruction de la file de messages ne doit être faite qu'une fois avec l'instruction :

```c
#include <sys/msg.h>
msgctl(msqid, IPC_RMID, NULL);
```

où `msqid` est l'identifiant obtenu avec `msgget`. Les processus qui n'ont pas créé la file n'ont rien à faire (s'ils sont en lecture ou écriture, la fonction en cours retournera avec l'erreur EIDRM).

## Exercices

### Exercice 1

Le premier exercice consiste à écrire deux programmes qui communiquent par une file de messages. Les deux programmes prennent chacun un identifiant qui sera utilisé pour envoyer/recevoir les messages qui leur sont destinés. Le programme `emetteur.c` lit les entrées au clavier, les envoie au programme `recepteur.c`. Ce dernier met la phrase reçue en majuscules et la renvoie à l'émetteur, qui attend alors à nouveau une saisie utilisateur.

Quand l'utilisateur saisit le texte `"EXIT"`, l'émetteur s'arrête en supprimant la file de messages.

### Exercice 2

Vous reprendrez le principe de l'exercice précédent avec un seul programme où le récepteur et l'émetteur seront des processus créés avec `fork()`.

### Exercice 3

Dans cet exercice, vous utiliserez `fork()` pour créer un processus enfant avec lequel le parent interagit à travers une file de messages. Cette file servira à passer une structure contenant un pointeur vers une fonction qui prend en paramètres deux entiers et qui renvoie un entier, et deux entiers.

Les fonctions pouvant être transmises seront :

```c
int somme(int a, int b); // renvoie a+b
int soustraction(int a, int b); // renvoie a-b
int produit(int a, int b); // renvoie a*b
int division(int a, int b); // renvoie a/b
int reste(int a, int b); //renvoie a%b
```

Vous testerez votre architecture avec un main qui envoie successivement les différentes opérations (fonctions +, -, etc.) et en attend les réponses. Une demande est envoyée via la file de messages, le résultat est attendu, puis on passe à l'opération suivante, etc.

## Bilan

Dans ce TD, vous avez utilisé les fonctions relatives aux files de messages unix :

- `ftok` pour générer une clé unique pour la création d'une file de messages.
- `msgget` pour obtenir un accès sur une file de messages.
- `msgsnd` pour envoyer des données sur une file de messages.
- `msgrcv` pour recevoir des données depuis une file de messages.
- `msgctl` pour fermer la file de messages.
