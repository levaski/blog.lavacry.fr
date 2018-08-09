---
title: "Compter et trier les ips d'un fichier de log"
date: 2013-07-11T09:39:38+02:00
draft: false
---
Il m'arrive très souvent de devoir aller analyser les logs de production des différents environnements de l'entreprise dans laquelle je travaille actuellement (Java, PHP, etc…). Lors de mon analyse, je prend le temps de remonter les différentes exceptions ou erreurs aux personnes concernées, mais je passe également un peu de temps à checker les comportements des utilisateurs. J'utilise donc une commande bash afin de compter et trier les ips qui se connectent sur nos serveurs. Ceci me permet de voir si certaines ips ressortent du lot et ainsi analyser leur comportement (tentative de hack, robots, etc…).

Cette commande permet :

- de filtrer un fichier de log (ici le log de haproxy) sur une information (ici la page index.html)
- d'en extraire les adresses ip et le port via awq (dans le log haproxy, l'ip et le port sont présents en 6ème colonne)
- de ne garder que l'ip avec la commande cut (ici -d: signifie que le délimiteur que nous utilisons est ':' et -f1 indique que nous ne gardons que la première partie de la chaîne)
- de trier les ips de façon croissante avec la commande sort (ici -n signifie que nous trions des nombres)
- de compter le nombre d'occurence des ips avec la commande uniq -c
- et enfin de trier le résultat par nombre d'occurence croissante avec la commande sort -n -k1,1 (-k1,1 signifie que nous basons notre tri uniquement sur la première colonne)

{{< highlight bash >}}
grep test.html /var/log/haproxy/haproxy-info.log | awk '{print $6}' | cut -d: -f1 | sort -n | uniq -c | sort -n -k1,1
{{< /highlight >}}

Il ne vous reste plus qu'à partir à la chasse aux ips douteuses !