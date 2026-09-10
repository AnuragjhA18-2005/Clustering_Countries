# Clustering Project Plan: Country Development Indicators

## Dataset Overview
- **167 countries**, 9 numerical features (excluding country name)
- Features: `child_mort`, `exports`, `health`, `imports`, `income`, `inflation`, `life_expec`, `total_fer`, `gdpp`
- Goal: Group countries with similar socio-economic profiles

---

## Phase 1: Exploratory Data Analysis (EDA)

### 1.1 Load & Inspect
- [ ] Load CSV, check shape, dtypes, missing values
- [ ] **Question**: Why might `country` column need special handling?

### 1.2 Univariate Analysis
- [ ] Plot distributions for each feature (histograms/boxplots)
- [ ] **Question**: Which features are skewed? What does that imply for distance-based algorithms?

### 1.3 Bivariate Analysis
- [ ] Correlation heatmap
- [ ] Pairplot for top 4-5 correlated features
- [ ] **Question**: High correlation between features → what problem does this cause for clustering?

### 1.3.1 EDA Findings → Implications for Later Phases

| Finding | Impact on Clustering | Solution / Mitigation |
|---------|---------------------|----------------------|
| **Multicollinearity**: `income` & `gdpp` (r=0.90), `child_mort` & `life_expec` (r=-0.89) | Redundant features dominate Euclidean distance; K-means centroids pulled toward correlated dimensions; silhouette score inflated artificially | Drop one of each highly correlated pair (keep `gdpp`, `child_mort`) |
| **Severe right-skew**: `income`, `gdpp`, `child_mort`, `total_fer` | Mean/Std-based scaling (StandardScaler) ineffective; outliers (Luxembourg, Singapore) dominate distances; clusters biased toward dense low-end | Use RobustScaler (median/IQR) or log-transform skewed features before scaling; test both |
| **Left-skew**: `life_expec` (clustered at 70-80) | Compressed variance at high end → small distance differences between developed countries | Consider power transform (Yeo-Johnson) or binning for interpretability |
| **Non-linear (L-shaped) wealth-health relationships** | K-means assumes spherical clusters (linear boundaries); hierarchical with Ward linkage assumes convex clusters; both will split along linear cuts missing true structure | Try log-transform on wealth features (`log(income)`, `log(gdpp)`) to linearize; or use GMM (Phase 5.3) for elliptical clusters |
| **Wealth disparity scaling**: `gdpp` grows faster than `income` at high end | Single "wealth" dimension conflates aggregate vs per-capita prosperity | Create ratio feature `income/gdpp` (labor share) as engineered feature (Phase 5.1) |

**How Each Solution Fixes the Problem:**

1. **Dropping correlated features**: When two features are highly correlated (e.g., `income` & `gdpp`), they essentially encode the same information. Euclidean distance treats them as independent dimensions, so countries are measured twice along the same underlying "wealth" axis. This inflates the importance of that dimension, distorting cluster boundaries. Dropping one feature (keeping `gdpp` and `child_mort`) eliminates redundancy, ensuring each dimension contributes unique information to the distance calculation.

2. **RobustScaler / Log-transform for right-skew**: Right-skewed features like `income` have a long tail where a few wealthy countries (Luxembourg, Singapore) have extreme values. StandardScaler uses mean and standard deviation, which are themselves pulled toward these outliers, compressing the scaling for the majority of countries. RobustScaler uses median and IQR, which are unaffected by extreme values, giving the bulk of countries meaningful separation. Log-transform further compresses the high end and stretches the low end, making the distribution more symmetric before scaling.

3. **Power transform for left-skew**: `life_expec` values cluster at 70-80, with few countries below 50. The compressed range means developed countries appear almost identical in this dimension, even though there are meaningful differences (e.g., 75 vs 82 years). Yeo-Johnson power transform reshapes the distribution to spread out the clustered end and compress the sparse end, restoring discriminative power across the full range.

4. **Log-transform for non-linear relationships**: The L-shaped relationship between wealth and health means that at low income, small increases in income correspond to large gains in life expectancy, but at high income, additional wealth yields diminishing returns. K-means and Ward linkage assume linear (straight-line) boundaries between clusters. Log-transforming wealth features converts the exponential relationship into a linear one, allowing distance-based algorithms to draw meaningful straight boundaries that better separate country groups.

5. **Ratio feature for wealth disparity**: `gdpp` measures per-capita economic output while `income` measures aggregate national income. At high values, `gdpp` grows faster because wealthy nations have smaller populations or higher productivity. Using both as separate features doesn't capture this divergence. The `income/gdpp` ratio captures the structural difference—whether a country's wealth comes from large aggregate output or high per-capita productivity—adding a dimension that raw features miss.

**Key Decision Before Phase 1.4**: Choose scaling strategy based on above. Recommended pipeline:
1. Drop correlated features: remove `income` and `life_expec` (keep `gdpp`, `child_mort`)
2. Log-transform skewed features: `gdpp`, `child_mort`, `total_fer`, `exports`, `imports`
3. Apply RobustScaler to all remaining features
4. Verify transformed distributions are roughly symmetric

### 1.4 Feature Scaling
- [ ] Apply StandardScaler and MinMaxScaler
- [ ] **Question**: K-means uses Euclidean distance. Why is scaling critical here? Compare feature ranges (e.g., `gdpp` ~300-105000 vs `health` ~1-17).

---

## Phase 2: K-Means Clustering

### 2.1 Baseline Model
- [ ] Fit KMeans with k=3 (arbitrary start)
- [ ] **Question**: What does the inertia_ value tell you? Is lower always better?

### 2.2 Finding Optimal K
- [ ] Elbow method (inertia vs k)
- [ ] Silhouette score vs k
- [ ] **Question**: Elbow often ambiguous. How does silhouette score help? What's a "good" silhouette score?

### 2.3 Cluster Interpretation
- [x] Compute cluster centroids (in original feature space)
- [x] Create profile for each cluster (mean values per feature)
- [x] Label clusters: Developed (cluster 0), Underdeveloped (cluster 1), Developing (cluster 2)
- [ ] **Question**: Can you label clusters meaningfully (e.g., "Developed", "Developing", "Underdeveloped")?

**How Cluster Labeling Works:**

Cluster labeling is **not** done by the algorithm — K-means only assigns numerical labels (0, 1, 2). You interpret the clusters by examining the **centroid profiles** (mean feature values per cluster) in the original feature space.

**Step 1: Examine centroid profiles**

| Feature | Cluster 0 (40 countries) | Cluster 1 (62 countries) | Cluster 2 (57 countries) |
|---------|------------------------|------------------------|------------------------|
| child_mort | 7.15 | 75.12 | 24.47 |
| exports | 37.63 | 24.70 | 56.10 |
| health | 9.40 | 5.91 | 5.93 |
| imports | 40.48 | 36.33 | 58.82 |
| inflation | 2.42 | 12.22 | 8.04 |
| total_fer | 1.76 | 4.36 | 2.44 |
| gdpp | 29,212 | 2,138 | 11,069 |

**Step 2: Identify the most discriminative features**

Not all features contribute equally to labeling. Here, two features separate clusters most clearly:
- **`gdpp`**: The three clusters sit at very different wealth levels (29K vs 2K vs 11K)
- **`child_mort`**: Strong inverse relationship with wealth (7 vs 75 vs 24)

Other features (`exports`, `health`, `imports`) help confirm but are less decisive on their own.

**Step 3: Assign meaningful labels using domain logic**

The labeling follows a logical chain:

1. **Cluster 0 → "Developed"**: Highest gdpp (29,212), lowest child_mort (7.15), highest health spending (9.40%), lowest fertility (1.76), lowest inflation (2.42%). These are wealthy nations with strong institutions, low mortality, and stable economies — e.g., Western Europe, USA, Japan, Australia.

2. **Cluster 1 → "Underdeveloped"**: Lowest gdpp (2,138), highest child_mort (75.12), lowest health spending (5.91%), highest fertility (4.36), highest inflation (12.22%). These are low-income nations struggling with high mortality, volatile economies, and rapid population growth — e.g., Sub-Saharan Africa, parts of South Asia.

3. **Cluster 2 → "Developing"**: Middle gdpp (11,069), middle child_mort (24.47), highest trade activity (exports 56.10%, imports 58.82%). These are middle-income nations with moderate development — e.g., Latin America, Southeast Asia, some Eastern European countries.

**Key insight**: The clusters form a clear socioeconomic gradient. The distance between centroids in the transformed feature space reflects real-world development tiers.

### 2.4 K-Means Limitations
- [ ] Try k=2, k=5, k=10 — observe changes
- [ ] **Question**: K-means assumes spherical clusters of similar size. Does this hold for country data?

---

## Phase 3: Agglomerative (Hierarchical) Clustering

### 3.1 Dendrogram Analysis
- [ ] Plot dendrogram (use `scipy.cluster.hierarchy`)
- [ ] **Question**: How do you choose cut height? What does "distance" on y-axis represent?

### 3.2 Linkage Methods Comparison
- [ ] Test: `ward`, `complete`, `average`, `single`
- [ ] **Question**: How does linkage choice affect cluster shape? When would you use each?

### 3.3 Model Fitting
- [ ] Fit AgglomerativeClustering with optimal n_clusters from dendrogram
- [ ] Compare clusters with K-means results
- [ ] **Question**: Hierarchical doesn't require specifying k upfront (dendrogram guides it). When is this advantageous?

### 3.4 Computational Considerations
- [ ] **Question**: Agglomerative is O(n²) or O(n³). At what dataset size does it become impractical?

---

## Phase 4: Evaluation & Comparison

### 4.1 Internal Metrics
- [ ] Silhouette score, Calinski-Harabasz, Davies-Bouldin for both methods
- [ ] **Question**: These metrics have biases (e.g., silhouette favors convex clusters). How to interpret?

### 4.2 External Validation (if labels existed)
- [ ] **Question**: No ground truth here. How would you validate if you had "development status" labels?

### 4.3 Stability Analysis
- [ ] Run K-means 10 times with different random_state
- [ ] **Question**: How much do cluster assignments vary? What does this imply?

### 4.4 India's Position Relative to Other Countries
- [ ] Identify which cluster India falls into and analyze its position within the cluster
- [ ] Compare India's feature values against cluster centroids and other countries in the same cluster
- [ ] Determine where India leads other countries (advantages/strengths) and where it lags behind (weaknesses/improvement areas)
- [ ] **Question**: What socio-economic indicators place India ahead of or behind similar developing nations?

---

## Phase 5: Advanced Topics (Stretch Goals)

### 5.1 Feature Engineering
- [ ] Create ratios (e.g., `exports/imports`, `health/gdpp`)
- [ ] **Question**: Domain-informed features vs raw features — which works better?

### 5.2 Handling Outliers
- [ ] Identify outliers (e.g., Luxembourg, Singapore)
- [ ] Test clustering with/without outliers
- [ ] **Question**: K-means is sensitive to outliers. How does hierarchical handle them?

### 5.3 Soft Clustering (GMM)
- [ ] Try GaussianMixture for probabilistic assignments
- [ ] **Question**: When might a country belong to multiple clusters?

---

## Learning Checkpoints

After each phase, answer in writing:
1. What surprised you about the results?
2. What assumptions did the algorithm make that may not hold?
3. How would you explain your clusters to a non-technical stakeholder?

---

## Deliverables

1. **Jupyter Notebook** with all code, visualizations, and markdown explanations
2. **Cluster Profile Table** — mean feature values per cluster (original scale)
3. **Country-to-Cluster Mapping** CSV
4. **2-page Summary** — methodology, key findings, limitations

---

## Recommended Reading Order

| Topic | Resource |
|-------|----------|
| K-means intuition | StatQuest YouTube "K-means Clustering" |
| Hierarchical clustering | StatQuest "Hierarchical Clustering" |
| Silhouette score | sklearn docs + original paper (Rousseeuw 1987) |
| Linkage methods | scipy docs + "Comparison of Linkage Methods" blog posts |

---

## Reflection Questions (End of Project)

1. Which algorithm gave more interpretable clusters for this domain? Why?
2. How did feature scaling change your results?
3. If you had to deploy this, what preprocessing pipeline would you save?
4. What would you do differently with a 10,000-country dataset?

---

*Start with Phase 1. Commit after each phase. Push yourself to answer the **Question** prompts before moving on.*