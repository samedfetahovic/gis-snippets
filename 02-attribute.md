# Attribute

## Whitespace und Leerwerte aufräumen

**Wann:** Attribute aus Excel-Import oder CSV enthalten unsichtbare Leerzeichen,  
NULL-Strings statt echter NULLs, oder inkonsistente Groß- und Kleinschreibung.

Whitespace trimmen und Platzhalter-Strings in echte NULLs umwandeln. Arbeitet  
über eine Edit-Session, damit die Änderungen persistiert werden.



```python

from collections import Counter

# Strings, die als Platzhalter für "kein Wert" interpretiert werden sollen
NULL_PLACEHOLDER_STRINGS = {"null", "none", "keine angabe", "-", ""}


def clean_string_attributes():
    # Trimmt Whitespace in allen String-Feldern und ersetzt
    # Platzhalter-Strings (z. B. 'null', '-') durch echte NULLs.
    active_layer = iface.activeLayer()
    layer_fields = active_layer.fields()
    data_provider = active_layer.dataProvider()

    data_provider.startEdit()

    # Sammelt alle zu ändernden Features: {Feature-ID: [Werte für alle Felder]}
    modified_features = {}

    for feature in active_layer.getFeatures():
        field_updates = {}

        for field_index, layer_field in enumerate(layer_fields):
            cell_value = feature[field_index]

            if not isinstance(cell_value, str):
                continue

            trimmed_value = cell_value.strip()

            # Platzhalter → echte NULL
            if trimmed_value.lower() in NULL_PLACEHOLDER_STRINGS:
                field_updates[field_index] = None
            # Nur Whitespace entfernt (Inhalt bleibt)
            elif trimmed_value != cell_value:
                field_updates[field_index] = trimmed_value

        if field_updates:
            # changeAttributeValues erwartet eine vollständige Zeile
            # (None = Feld bleibt unverändert)
            full_row_values = [None] * len(layer_fields)
            for field_index, new_value in field_updates.items():
                full_row_values[field_index] = new_value
            modified_features[feature.id()] = full_row_values

    if modified_features:
        data_provider.changeAttributeValues(modified_features)

    data_provider.commitChanges()
    active_layer.triggerRepaint()

    print(
        f"Attribut-Reinigung abgeschlossen:\n"
        f"  Geänderte Features: {len(modified_features)}"
    )


clean_string_attributes()

```

**Ergebnis:** Saubere Strings. Joins und Filter funktionieren danach zuverlässig.




## Duplikate per Attribut erkennen

**Wann:** Nach einem Join oder Merge existieren Duplikate (z. B. gleiche flaeche_id mehrfach).

Häufigste doppelte Werte in einem Feld auflisten.



```python

from collections import Counter


def find_duplicate_values(field_name: str = "flaeche_id", max_results: int = 10) -> dict:
    # Zählt Vorkommen aller Werte im angegebenen Feld und
    # gibt Werte mit mehr als einem Vorkommen zurück.
    #
    # Returns:
    #   dict: {Wert: Anzahl_Vorkommen} – nur Duplikate.
    active_layer = iface.activeLayer()
    field_index = active_layer.fields().indexOf(field_name)

    if field_index == -1:
        print(f"Feld '{field_name}' nicht gefunden.")
        return {}

    # Zählt wie oft jeder Wert vorkommt
    value_occurrence_counts = Counter(
        feature[field_index] for feature in active_layer.getFeatures()
    )

    # Nur Werte mit mehr als einem Vorkommen
    duplicate_values = {
        value: occurrence_count
        for value, occurrence_count in value_occurrence_counts.items()
        if occurrence_count > 1
    }

    print(f"Duplikate in Feld '{field_name}':")
    print(f"  Betroffene Werte: {len(duplicate_values)}")
    print(f"  Top {max_results}:")

    for value, occurrence_count in sorted(
        duplicate_values.items(),
        key=lambda item: -item[1],
    )[:max_results]:
        print(f"    {value}: {occurrence_count}×")

    return duplicate_values


find_duplicate_values("flaeche_id")

```

**Ergebnis:** Du siehst sofort, welche Werte betroffen sind und wie häufig.



