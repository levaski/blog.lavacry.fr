---
title: "Wicket-atmosphere tutorial"
date: 2013-03-11T09:39:31+02:00
draft: false
---
Depuis wicket-6.0.0-beta2, il existe un module expérimental qui permet d'intégrer le framework atmosphere à wicket. Ce framework permet de faire des applications web asynchrones en utilisant les websockets ou les contournements des différents navigateurs.

J'ai donc voulu tester cette intégration pour faire 2 choses :

- l'affichage et la mise à jour de la date et heure du serveur sur la page cliente
- la saisie d'un message et affichage de celui-ci dans une ModalWindow chez tous les clients de l'application sans rechargement

La seconde fonctionnalité pourrait permettre de faire un service de notification sur une application Wicket en production afin de prévenir les clients connectés que le service va être mis en maintenant dans les 5 minutes par exemple.

# Création du projet

Pour créer le projet, j'utilise l'archetype maven pour générer une application wicket.

{{< highlight bash >}}
mvn archetype:generate -DarchetypeGroupId=org.apache.wicket -DarchetypeArtifactId=wicket-archetype-quickstart -DarchetypeVersion=6.6.0 -DgroupId=fr.lavacry.test -DartifactId=wicket-atmosphere-test -DarchetypeRepository=https://repository.apache.org/ -DinteractiveMode=false
{{< /highlight >}}

Ensuite, je modifie le pom.xml pour ajouter la dépendance vers wicket-atmosphere, ajouter la configuration du plugin wtp, et enlever les dépendances vers jetty (je vais utiliser tomcat depuis eclipse).

# Configuration d'atmosphere

Pour configurer atmosphere, j'ai du faire 2 choses :

- modifier le web.xml pour utiliser la servlet AtmosphereServlet
- créer le fichier atmosphere.xml dans src/main/webapp/WEB-INF/META-INF

web.xml :

{{< highlight xml >}}
<?xml version="1.0" encoding="ISO-8859-1"?>
<web-app xmlns="http://java.sun.com/xml/ns/javaee" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd"
	version="2.5">
	<display-name>wicket-atmosphere-test</display-name>
	<servlet>
		<servlet-name>AtmosphereApplication</servlet-name>
		<servlet-class>org.atmosphere.cpr.AtmosphereServlet</servlet-class>
		<init-param>
			<param-name>applicationClassName</param-name>
			<param-value>fr.levaski.test.WicketApplication</param-value>
		</init-param>
		<init-param>
			<param-name>org.atmosphere.useWebSocket</param-name>
			<param-value>true</param-value>
		</init-param>
		<init-param>
			<param-name>org.atmosphere.useNative</param-name>
			<param-value>true</param-value>
		</init-param>
		<init-param>
			<param-name>org.atmosphere.cpr.CometSupport.maxInactiveActivity</param-name>
			<param-value>30000</param-value>
		</init-param>
		<init-param>
			<param-name>filterMappingUrlPattern</param-name>
			<param-value>/*</param-value>
		</init-param>
		<init-param>
			<param-name>org.atmosphere.websocket.WebSocketProtocol</param-name>
			<param-value>org.atmosphere.websocket.protocol.EchoProtocol</param-value>
		</init-param>
		<init-param> 
        	<param-name>org.atmosphere.cpr.broadcastFilterClasses</param-name> 
        	<param-value>org.apache.wicket.atmosphere.TrackMessageSizeFilter</param-value>
        </init-param>
       <load-on-startup>1</load-on-startup>
	</servlet>
	<servlet-mapping>
		<servlet-name>AtmosphereApplication</servlet-name>
		<url-pattern>/*</url-pattern>
	</servlet-mapping>
</web-app>
{{< /highlight >}}

atmosphere.xml :

{{< highlight xml >}}
<atmosphere-handlers>
	<atmosphere-handler context-root="/*" class-name="org.atmosphere.handler.ReflectorServletProcessor">
		<property name="filterClassName" value="org.apache.wicket.protocol.http.WicketFilter" />
	</atmosphere-handler>
</atmosphere-handlers>
{{< /highlight >}}

# Ajout de la fonctionnalité affichant le temps du serveur

Pour cette fonctionnalité, je crée un EventBus pour poster un objet DateEvent. Un thread se charge de poster un DateEvent toutes les secondes.

{{< highlight java >}}
eventBus = new EventBus(this);
ScheduledExecutorService scheduler = Executors
		.newScheduledThreadPool(1);
final Runnable beeper = new Runnable() {
	@Override
	public void run() {
		try {
			eventBus.post(new DateEvent(new Date()));
		} catch (Exception e) {
			e.printStackTrace();
		}
	}
};
scheduler.scheduleWithFixedDelay(beeper, 1, 1, TimeUnit.SECONDS);
{{< /highlight >}}

La partie cliente se trouve dans la HomePage. La méthode écoutant les DateEvent est précédée d'une annotation @Subscribe. Cette méthode récupère donc le DateEvent et modifie le label avec la date récupérée.

{{< highlight java >}}
@Subscribe()
public void updateTime(AjaxRequestTarget target, DateEvent event) {
	timeLabel.setDefaultModelObject(event.getDate().toString());
	target.add(timeLabel);
}
{{< /highlight >}}

# Seconde fonctionnalité : affichage d'un message de notification

La partie d'envoi du message est faite depuis la page HomePage. Un formulaire post un AlertEvent dans l'EventBus.

{{< highlight java >}}
form.add(message = new TextField<String>("message", Model.of("")));
form.add(new AjaxSubmitLink("send", form) {

	@Override
	protected void onSubmit(AjaxRequestTarget target, Form<?> form) {
		EventBus.get().post(new AlertEvent(message.getModelObject()));
	}

	@Override
	protected void onError(AjaxRequestTarget target, Form<?> form) {
	}
});
{{< /highlight >}}

La partie client récupère l'AlertEvent et l'affiche dans une ModalWindow.

{{< highlight java >}}
@Subscribe()
public void displayMessage(AjaxRequestTarget target, AlertEvent event) {
	modal.addOrReplace(new ModalAlertPanel(modal.getContentId(), Model
			.of(event.getAlert())));
	modal.show(target);
}
{{< /highlight >}}

Les messages peuvent être filtrés en utilisant une référence vers une classe implémentant l'interface com.google.common.base.Predicate. Cette classe doit être référencée dans l'annotation @Subscribe.

Vous pouvez récupérer les sources du projet sur le repository git suivant :
https://github.com/levaski/wicket-atmosphere-test.git
