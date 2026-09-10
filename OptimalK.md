# Finding Optimal K in K-Means Clustering

K-Means requires specifying the number of clusters `k` upfront. Two common methods to find the optimal k are the **Elbow Method** and **Silhouette Score Analysis**.

---

## Method 1: Elbow Method

### Concept
The elbow method plots the **Within-Cluster Sum of Squares (WCSS)** against k. WCSS measures the total squared distance between each point and its assigned cluster centroid:

```
WCSS = Σ Σ ||x_i - μ_j||²
```

- As k increases, WCSS always decreases (more clusters = tighter groups)
- The optimal k is where the rate of decrease **slows down abruptly** — forming an "elbow" in the plot
- This point represents the best trade-off between minimizing WCSS and keeping k small

### Code

```python
from sklearn.cluster import KMeans

wcss = []

for i in range(1, 20):
    kmeans = KMeans(n_clusters=i, random_state=42)
    kmeans.fit(X)
    wcss.append(kmeans.inertia_)

plt.plot(range(1, 20), wcss, 'bo-')
plt.title('Elbow Method')
plt.xlabel('Number of Clusters (k)')
plt.ylabel('WCSS (Inertia)')
plt.show()
```

### Interpreting the Plot
- Look for the **bend** where WCSS stops dropping sharply
- Before the elbow: adding clusters significantly improves fit
- After the elbow: adding clusters gives diminishing returns
- The elbow is often ambiguous — that's why we use silhouette score as a second opinion

---

## Method 2: Silhouette Score

### Concept
The silhouette score measures how well each data point fits within its assigned cluster versus the nearest neighboring cluster. For each point *i*:

```
s(i) = (b(i) - a(i)) / max(a(i), b(i))
```

Where:
- **a(i)** = average distance from point *i* to all other points in the same cluster (intra-cluster cohesion)
- **b(i)** = average distance from point *i* to all points in the nearest neighboring cluster (inter-cluster separation)

**Score range: [-1, 1]**
- **+1** → point is far from neighboring clusters, well-assigned
- **0** → point is on the border between two clusters
- **-1** → point is likely assigned to the wrong cluster

### Code

```python
from sklearn.metrics import silhouette_score
from sklearn.cluster import KMeans

silhouette_scores = []
K_range = range(2, 11)

for k in K_range:
    kmeans = KMeans(n_clusters=k, random_state=42, n_init=10)
    labels = kmeans.fit_predict(X)
    score = silhouette_score(X, labels)
    silhouette_scores.append(score)

# Plot silhouette score vs k
plt.plot(K_range, silhouette_scores, 'bo-')
plt.xlabel('Number of clusters (k)')
plt.ylabel('Silhouette Score')
plt.title('Silhouette Score vs k')
plt.show()

# The k with the highest silhouette score is optimal
optimal_k = K_range[np.argmax(silhouette_scores)]
```

### Interpreting the Plot
- Look for the **peak** — the k value with the highest average silhouette score
- Unlike the elbow method (which looks for a bend), silhouette gives a clear maximum
- Score > 0.5 → reasonable clustering structure
- Score > 0.7 → strong clustering structure

**Tip:** Use `silhouette_samples()` to get per-point scores, then plot a silhouette diagram to see cluster quality visually — well-formed clusters show uniform-width bars, while overlapping or uneven bars signal poor separation.

---

## Which Method to Trust?

| Situation | Recommended Method |
|-----------|-------------------|
| Elbow is clear | Use elbow method |
| Elbow is ambiguous | Use silhouette score |
| Both agree | High confidence in result |
| They disagree | Try both k values, compare cluster interpretability |

In practice, use **both methods together**. If they point to the same k, you have strong evidence. If they differ, examine the clusters at each k and choose the one that makes more domain sense for your data.
