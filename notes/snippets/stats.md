---
layout: default
title: Snippets - Stats
permalink: /notes/snippets/stats/
---

```python
def calculate_outliers_iqr_method(values):
    """
    Calculate a list of outliers from a set of values using the IQR method

    :param values: pandas series of numerical values
    :returns: outliers: list of values that are outliers
    """
    q1 = values.quantile(q=0.25)
    q3 = values.quantile(q=0.75)
    iqr = q3 - q1
    
    outliers = []
    for value in values:
        if value > q3 + 1.5*iqr or value < q1 - 1.5*iqr:
            outliers.append(value)
    
    return outliers
```