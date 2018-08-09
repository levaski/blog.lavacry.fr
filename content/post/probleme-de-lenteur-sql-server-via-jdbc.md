---
title: "Problème de lenteur SQL Server via JDBC"
date: 2012-06-13T09:29:49+02:00
draft: false
---
Je me suis heurté à un problème de performance assez gênant récemment concernant des requêtes SQL (dans mon cas c'était des requêtes vers des vues) sur un serveur Microsoft SQL Server via JDBC.

Ces requêtes ne sont pourtant pas gourmandes et ne sont que des select. Pourtant, il fallait compter plus d'une minute depuis notre page web pour avoir un résultat. Ce même comportement était visible par un développeur AS400 qui attaquait également le SQL Server via JDBC. En revanche, les utilisateurs (service comptabilité) utilisant cette base avec une connexion native n'avaient pas ce problème. Le driver JDBC semble être en cause donc, mais j'imagine mal un bug aussi voyant.

Après quelques investigations, j'ai découvert un détail non négligeable concernant SQL Server. En effet, SQL Server différencie les types de données qui supportent l'Unicode de ceux qui supportent seulement l'ASCII. Par exemple, les types de données qui supportent l'Unicode sont nchar, nvarchar, longnvarchar et leurs contreparties ASCII sont respectivement char, varchar et longvarchar. Par défaut, tous les drivers JDBC SQL Server envoient les les chaînes de caractère en Unicode au SQL Server et ce sans se soucier du fait que le type de données cible supporte ou non l'Unicode.

Quelle conséquence ?

Dans le cas où le type de données cible supporte l'Unicode, tout se passe bien évidemment. En revanche, si celui-ci ne supporte que l'ASCII, c'est une autre paire de manche. En effet, cela provoque bien souvent des problèmes de performance, plus particulièrement lors de recherches. SQL Server tente de convertir les données non Unicode de la table en Unicode pour faire les comparaisons, ce qui implique un traitement supplémentaire. Mais là n'est pas le gros problème. Le problème est que s'il existe un Index sur la colonne en question, celui-ci n'est pas utilisé. On peut donc imaginer les problèmes de performance qui en résultent sur les tables comportant un nombre important d'enregistrements. C'est évidemment notre cas.

Comment corriger cela ?

Pour une fois, c'est très simple dans notre cas, puisque toutes nos données sont stockées comme des types de données ne supportant que l'ASCII. Il existe donc un paramètre de connexion sur les différents drivers JDBC permettant de spécifier le comportement à adopter. Voici un tableau récapitulatif de ce paramètre sur les différents drivers :

| Driver  |  Paramètre |
|---|---|
| JSQLConnect  | asciiStringParameters  |
| JTDS | sendStringParametersAsUnicode  |
| DataDirectConnect  | sendStringParametersAsUnicode  |
| Microsoft JDBC  | sendStringParametersAsUnicode  |
| WebLogic Type 4 JDBC SQL Server driver  | sendStringParametersAsUnicode  |

Voici donc un exemple d'url de connexion utilisant cette propriété pour corriger le problème :
{{< highlight java >}}
jdbc:sqlserver://host:port;databaseName=DBNAME;sendStringParametersAsUnicode=false;
{{< /highlight >}}


