Accepting sockets
The first thing our Redis server needs to do is to accept inbound TCP sockets. This is done by binding tokio::net::TcpListener to port 6379.

Nous souhaitons que notre serveur Redis puisse traiter de nombreuses requêtes simultanées. Pour ce faire, nous devons introduire une certaine forme de concurrence.

La concurrence et le parallélisme ne sont pas la même chose. Si vous alternez entre deux tâches, alors vous travaillez sur les deux tâches de manière concurrente, mais pas en parallèle. Pour que cela soit considéré comme du parallélisme, vous auriez besoin de deux personnes, une dédiée à chaque tâche.

Un des avantages d'utiliser Tokio est 
que le code asynchrone vous permet de travailler sur de nombreuses tâches de manière concurrente, sans avoir à les traiter en parallèle en utilisant des threads ordinaires. En fait, Tokio peut exécuter de nombreuses tâches de manière concurrente sur un seul thread !

Les tâches sont l'unité d'exécution gérée
par le planificateur. La création de la tâche la soumet au planificateur Tokio, qui s'assure ensuite que la tâche s'exécute lorsqu'elle a du travail à faire. La tâche créée peut être exécutée sur le même fil que celui où elle a été créée, ou elle peut s'exécuter sur un fil d'exécution différent. La tâche peut également être
déplacée entre les fils après avoir été créée.


État partagé
Jusqu'à présent, nous avons un serveur clé-valeur qui fonctionne. Cependant, il y a un défaut majeur : l'état n'est pas partagé entre les connexions. Nous allons corriger cela dans cet article.

Stratégies
Il existe plusieurs façons différentes de partager l'état dans Tokio.

Protégez l'état partagé avec un Mutex.
Générez une tâche pour gérer l'état et utilisez la transmission de messages pour y opérer.
En général, vous voulez utiliser la première approche pour des données simples, et la deuxième approche pour des choses qui nécessitent un travail asynchrone comme les primitives d'E/S. Dans ce chapitre, l'état partagé est un HashMap et les opérations sont insert et get. Aucune de ces opérations n'est asynchrone, donc nous allons utiliser un Mutex.

La dernière approche est abordée dans le chapitre suivant.

Tâches, threads et contention
L'utilisation d'un mutex bloquant pour protéger de courtes sections 
critiques est une stratégie acceptable lorsque la contention est minime. 
Lorsqu'un verrou fait l'objet d'une contention, le thread exécutant la tâche 
doit se bloquer et attendre le mutex. Cela bloque non seulement la tâche en cours, 
mais également toutes les autres tâches planifiées sur le thread actuel.

Par défaut, le runtime Tokio utilise un planificateur multithread. 
Les tâches sont planifiées sur un nombre quelconque de threads gérés par le runtime. 
Si un grand nombre de tâches sont planifiées pour s’exécuter et qu’elles nécessitent 
toutes l’accès au mutex, il y aura alors des conflits d’accès. En revanche, 
si l’option de runtime « current_thread » est utilisée, le mutex ne fera jamais 
l’objet de conflits d’accès.