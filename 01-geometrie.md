# Geometrie

## Invalide Geometrien reparieren

**Wann:** Layer enthält self-intersecting polygons oder ring issues.

Invalide Geometrien im aktiven Layer reparieren.  
Verwendet QgsGeometry.makeValid(), basierend auf GEOS.  
Repariert: self-intersections, duplicate rings, bow-ties.



```python

def repair_invalid_geometries():
    target_layer = iface.activeLayer()
    data_provider = target_layer.dataProvider()

    total_feature_count = 0
    repaired_feature_count = 0

    for feature in target_layer.getFeatures():
        total_feature_count += 1

        geometry = feature.geometry()

        if geometry is not None and not geometry.isValid():
            repaired_geometry = geometry.makeValid()
            data_provider.changeGeometry(feature.id(), repaired_geometry)
            repaired_feature_count += 1

    target_layer.triggerRepaint()

    print(
        f"Geometrie-Reparatur abgeschlossen:\n"
        f"  Total:      {total_feature_count}\n"
        f"  Repariert:  {repaired_feature_count}\n"
        f"  Gültig:     {total_feature_count - repaired_feature_count}"
    )

repair_invalid_geometries()

```
  
**Ergebnis:** Alle Geometrien sind gültig. ST_IsValid() gibt danach True zurück.  
  



## Features mit NULL-Geometrie finden

**Wann:** Nach einem Import oder Join fehlt bei manchen Zeilen die Geometrie.  
Die Zeile existiert in der Attributtabelle, wird aber nicht kartiert.

Features ohne Geometrie (NULL) im aktiven Layer identifizieren.  
Nutze die Ausgabe, um gezielt zu löschen oder nachzuarbeiten.



```python

def find_features_without_geometry(max_ids_to_print: int = 10):
    target_layer = iface.activeLayer()

    null_geometry_feature_ids = [
        feature.id()
        for feature in target_layer.getFeatures()
        if feature.geometry() is None
    ]

    null_geometry_count = len(null_geometry_feature_ids)
    total_feature_count = target_layer.featureCount()

    print(
        f"Features ohne Geometrie:\n"
        f"  Betroffen:  {null_geometry_count} von {total_feature_count}\n"
        f"  Anteil:     {null_geometry_count / total_feature_count:.1%}\n"
        f"  IDs (erste {max_ids_to_print}): {null_geometry_feature_ids[:max_ids_to_print]}"
    )

    return null_geometry_feature_ids


#IDs sammeln und für weitere Verarbeitung nutzen
affected_feature_ids = find_features_without_geometry()

```


**Ergebnis:** Man weiß nun, wie viele betroffen sind, und kann gezielt löschen und nachbearbeiten.




## Geometrie vereinfachen

**Wann:** Der Layer ist sehr detailliert und braucht eine Generalisierung für eine  
kleinere Zeitskala oder Web-Export.


Geometrien im aktiven Layer mit dem Douglas-Peucker-Algorithmus vereinfachen.  
Vorher den Layer kopieren oder eine Transaktion nutzen, da die Geometrie 
direkt  
am Provider überschrieben wird.



```python
def simplify_geometries(simplification_tolerance: float = 5.0):
    target_layer = iface.activeLayer()
    data_provider = target_layer.dataProvider()
    layer_crs = target_layer.crs()

    #Toleranz anpasst: bei geografischem CRS (Grad) ist 5.0 zu grob,
    #bei projiziertem CRS (Meter) ist es üblich.
    if layer_crs.isGeographic():
        effective_tolerance = 0.00005  #5 m auf mittleren Breiten
    else:
        effective_tolerance = simplification_tolerance

    total_feature_count = 0
    simplified_feature_count = 0

    for feature in target_layer.getFeatures():
        total_feature_count += 1
        geometry = feature.geometry()

        if geometry is not None and not geometry.isEmpty():
            simplified_geometry = geometry.simplify(effective_tolerance)
            data_provider.changeGeometry(feature.id(), simplified_geometry)
            simplified_feature_count += 1

    target_layer.triggerRepaint()

    print(
        f"Generalisierung abgeschlossen:\n"
        f"  Toleranz:  {effective_tolerance} Layer-Einheiten\n"
        f"  CRS:       {layer_crs.authid()}\n"
        f"  Verarbeitet: {simplified_feature_count}/{total_feature_count} Features"
    )


#Aufruf (Toleranz nach Bedarf anpassen)
simplify_geometries(simplification_tolerance=5.0)
```

**Ergebnis:** Es gibt weniger Knotenpunkte, gleiche Form innerhalb der Toleranz.





