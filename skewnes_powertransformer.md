# Skewness and Power Transformer

## What is skewness?

**Skewness tells us the direction in which the tail of a distribution stretches.**

### 1. Right-Skewed (Positive Skew)

- Most data is concentrated toward the **left**.
- A few unusually large values stretch the tail toward the **right**.
- **Skewness > 0** → Right-skewed.
- Examples:
  - `0.5` → right-skewed
  - `1.2` → right-skewed
  - `3.0` → strongly right-skewed

**Memory trick:**  
**Tail → right = Positive skewness**

---

### 2. Left-Skewed (Negative Skew)

- Most data is concentrated toward the **right**.
- A few unusually small values stretch the tail toward the **left**.
- **Skewness < 0** → Left-skewed.
- Examples:
  - `-0.5` → left-skewed
  - `-1.2` → left-skewed
  - `-3.0` → strongly left-skewed

**Memory trick:**  
**Tail → left = Negative skewness**

---

### 3. Approximately Symmetric

- The distribution is roughly balanced on both sides.
- **Skewness ≈ 0** → approximately symmetric.

---

## The Rule to Remember

| Skewness Value | Distribution |
|---|---|
| **> 0** | Right-skewed |
| **< 0** | Left-skewed |
| **≈ 0** | Approximately symmetric |

### Most Important Point

**Don't determine skewness by looking at where the tallest bars are.**

Instead, **look at the direction of the long tail**:

- **Long tail to the right → Right/Positive skew**
- **Long tail to the left → Left/Negative skew**


### Yeo-Johnson Power Transformer

Yeo-Johnson is useful because it can **automatically find a power transformation that makes a feature more symmetric**, including **left-skewed** features.

Unlike a simple log transform, it can use **different transformation behavior depending on the value and learned λ (lambda)**. This means it can correct either right or left skew.

**Example:**

* Right skew → transformation can compress the long right tail.
* Left skew → transformation can compress/stretch the appropriate side to make the distribution more symmetric.

### Important use cases

1. **Left-skewed features** — where `log()` isn't appropriate/effective.
2. **Features containing zero or negative values** — unlike Box-Cox, Yeo-Johnson **supports zero and negative values**.
3. **Automatically choosing the transformation** — it learns the best λ rather than you manually choosing a power.

```python
from sklearn.preprocessing import PowerTransformer

pt = PowerTransformer(method='yeo-johnson')
df[features] = pt.fit_transform(df[features])
```

**Key idea:** Yeo-Johnson is a good general-purpose choice when you want to make numerical features **more Gaussian/symmetric**, especially when they contain **zero/negative values**.
