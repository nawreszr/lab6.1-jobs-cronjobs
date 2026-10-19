# Lab 6.1 : Jobs et CronJobs (K3s)

## 📖 Description du Lab
Ce laboratoire explore l'exécution de tâches ponctuelles (Jobs) et planifiées (CronJobs) sur Kubernetes. Nous avons observé la gestion des échecs (retry/backoff), la parallélisation des tâches, la planification temporelle (cron), ainsi que la persistance des sauvegardes sur un volume (PVC).

## 📂 Fichiers du projet
- `calcul.yaml` : Job simple (succès).
- `echec.yaml` : Job en échec (`restartPolicy: Never`).
- `echec-onfailure.yaml` : Job en échec (`restartPolicy: OnFailure`).
- `parallele.yaml` : Job parallèle avec nettoyage automatique (`ttlSecondsAfterFinished`).
- `sauvegarde.yaml` : CronJob simple (toutes les minutes).
- `lent.yaml` : CronJob pour tester la concurrence (`Forbid`).
- `sauvegarde-vol.yaml` : PVC + CronJob écrivant sur un volume persistant.
- `lecteur.yaml` : Pod lisant les fichiers du volume persistant.

---

## 🧠 Questions de réflexion (Réponses)

**1. Quelle différence de comportement entre `restartPolicy: Never` et `OnFailure` pour un Job qui échoue ?**
> Avec `Never`, chaque tentative crée un **nouveau Pod** (les Pods en erreur restent visibles). Avec `OnFailure`, le conteneur est **relancé dans le même Pod** (la colonne `RESTARTS` augmente, et le Pod est supprimé une fois le `backoffLimit` atteint).

**2. À quoi sert `ttlSecondsAfterFinished` ?**
> À supprimer automatiquement un Job terminé et ses Pods après un délai (par exemple 120 secondes). Sans cette option, les Jobs et leurs Pods s'accumulent indéfiniment dans le namespace.

**3. Que se passerait-il avec `concurrencyPolicy: Allow` dans l'étape 6 ?**
> Une nouvelle exécution démarrerait à chaque minute **sans attendre la fin de la précédente**. Plusieurs Jobs tourneraient en parallèle, ce qui est dangereux (risque de corruption de données ou de saturation).

**4. Pourquoi `kubectl create job --from=cronjob/...` est-il utile ?**
> Il permet de **tester immédiatement** le contenu du CronJob sans attendre l'échéance du calendrier. C'est idéal pour valider la configuration avant de laisser tourner la tâche en production.

**5. Une sauvegarde stockée sur le même volume local que la base protège-t-elle contre la perte d'un nœud ? Que proposeriez-vous ?**
> **Non** : si le nœud tombe, la base et sa sauvegarde disparaissent ensemble. Il faut copier les sauvegardes **ailleurs** (stockage réseau, autre nœud, stockage objet type S3, ou machine distante).

---

## 🚀 Commandes clés du Lab

### 1. Créer un Job simple
```bash
kubectl apply -f calcul.yaml
kubectl wait --for=condition=complete job/calcul --timeout=60s
kubectl logs job/calcul
2. Tester un Job en échec (retry)
bash
kubectl apply -f echec.yaml
kubectl get pods -l job-name=echec -w
3. Job parallèle avec TTL
bash
kubectl apply -f parallele.yaml
kubectl get job parallele -w
4. CronJob (Planification)
bash
kubectl apply -f sauvegarde.yaml
kubectl create job --from=cronjob/sauvegarde test-manuel
kubectl get cronjob
5. Suspension et Concurrence
bash
kubectl patch cronjob sauvegarde -p '{"spec":{"suspend":true}}'
kubectl get cronjob
kubectl patch cronjob sauvegarde -p '{"spec":{"suspend":false}}'
6. Sauvegarde sur volume persistant
bash
kubectl apply -f sauvegarde-vol.yaml
# (Attendre quelques minutes)
kubectl apply -f lecteur.yaml
kubectl exec lecteur -- ls -l /backup
kubectl exec lecteur -- sh -c 'cat /backup/$(ls /backup | tail -n 1)'
🧹 Nettoyage
bash
kubectl delete namespace tp-k8s
kubectl config set-context --current --namespace=default
