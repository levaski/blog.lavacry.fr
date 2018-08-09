---
title: "Modifier la sérialisation Json avec Jackson"
date: 2013-10-02T09:39:45+02:00
draft: false
---
Lors de la mise en place d'un webservice REST Json via Jersey et Jackson, il est souvent nécessaire de modifier la façon dont sont sérialisées certaines valeurs.

Par exemple, lorsque l'on veut afficher une valeur monétaire représentée par un BigDecimal, il faut créer une classe qui surcharge la classe JsonSerializer, et faire référence à cette classe via une annotation sur la propriété du bean à sérialiser.

{{< highlight java >}}
public class MoneySerializer extends JsonSerializer<BigDecimal> {

    @Override
    public void serialize(BigDecimal bigDecimal, JsonGenerator jsonGenerator,
                          SerializerProvider serializerProvider)
            throws IOException, JsonProcessingException {

        if (bigDecimal == null) {
            jsonGenerator.writeNull();
            return;
        }
        jsonGenerator.writeString(bigDecimal.setScale(2, RoundingMode.HALF_UP).toString());
    }
}
{{< /highlight >}}

{{< highlight java >}}
    @javax.persistence.Column(name = "AQ_GAINS")
    @JsonSerialize(using = MoneySerializer.class)
    @Basic
    public BigDecimal getGains() {
        return gains;
    }
{{< /highlight >}}

