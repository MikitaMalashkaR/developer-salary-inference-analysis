# Developer Salary Analysis — Stack Overflow 2025

How much of the salary difference between developer markets is explained by professional
experience, and how much by geography alone?

Inference statistics and multiple regression on the Stack Overflow Developer Survey 2025,
built from first principles: every metric is implemented from its formula, then verified
against `numpy`, `scipy` or `statsmodels`.

---

## Key findings

**1. Country explains salary roughly four times better than experience.**
Within salaried professional developers in the US, Germany and India, experience alone accounts for 15% of the variation in log salary (R² = 0.154). Adding country raises that to 59% (R² = 0.589).

**2. At equal experience, US salaries sit about 87% above German ones and Indian salaries about 73% below.**

**3. More than half of the apparent effect of experience was not experience at all.**
Without controlling for country, a year of experience appears to be worth +3.8% in salary. Once country enters the model, the same coefficient drops to **+1.8%**. The difference is omitted variable bias: Indian respondents are both less experienced and far lower paid, so a model without country attributes the whole pattern to experience. The bias is invisible from inside the first model — its R² is unremarkable but plausible, its coefficient highly significant, its residual diagnostics unremarkable.

**4. The return on experience is not the same in every market.**
Roughly **+10.6% per year in India** against **+1.4% in the US** and **+1.8% in Germany**. The Indian difference is significant at p < 0.001; the difference between the US and Germany is not (p = 0.053) and is reported as undetermined rather than absent.

**5. Experience is the dominant salary factor in India and a marginal one in the US.**
Within-country R² is 0.26 for India against 0.07 for the US — American salaries are driven mainly by characteristics this model does not contain.

---

## Data

| | |
|---|---|
| Source | [Stack Overflow Annual Developer Survey 2025](https://survey.stackoverflow.co/) |
| Raw size | 49,191 respondents × 172 columns |
| Response variable | `ConvertedCompYearly` — annual compensation, pre-converted to USD by the survey publisher |
| Licence | Open Database License (ODbL) |

Currency conversion is handled upstream by Stack Overflow, which removes the parsing problems typical of self-reported salary data and leaves the analysis free to focus on inference.

---

## Population and filtering

**Population: salaried professional developers in the United States, Germany and India.**

Germany is the market this analysis targets; the US and India are the two largest markets in the survey and differ from it in both salary level and structure.

The population is defined in words first, and the filters follow from the definition — not the other way round. Each rule is a separate boolean mask over the full index, applied together in a single `.loc` call.

| Step | Rule | US | Germany | India | Total |
|------|------|----|---------|-------|-------|
| 0 | Country | 7,233 | 3,025 | 2,547 | 12,805 |
| 1 | `MainBranch` = developer by profession, `Employment` = Employed | 4,558 | 1,917 | 1,512 | 7,987 |
| 2 | Salary reported and within [1,000; 6,000,000] USD | 3,862 | 1,530 | 805 | 6,197 |
| 3 | `WorkExp` reported and ≤ 68 years | 3,839 | 1,521 | 784 | **6,144** |

**On the salary threshold.** The lower bound is set at 1,000 USD rather than a rounder, higher figure: the 25th percentile of Indian salaries is about 8,100 USD, so a higher bound would remove a quarter of a real labour market as though it were noise. The bound is chosen against external knowledge of the markets involved, not against the shape of the distribution — standard outlier rules (z-score, Tukey's IQR) assume symmetry and fail on right-skewed income data, cutting valid high earners while leaving entries of 1 USD per year untouched.

**On freelancers.** Independent contractors and the self-employed are excluded. The income question is phrased in terms of salaried employment ("salary, bonuses, and perks"), so the quantity measured for a freelancer is undefined; their relative dispersion (P75/P25) is wider in all three countries without exception; and the freelance effect is not consistent across markets — negative in the US, positive in Germany, neutral in India. Capturing it properly would require a country × employment interaction and is out of scope here.

**On non-response.** Response rates to the salary question were 72% in the US, 71% in Germany and 43% in India. Within the filtered Indian population, 689 respondents reported no salary at all while only 18 were removed by the validity range — the data loss is driven by non-response, not by filtering, and no threshold can recover it.

---

## Methods

**Chapter A — whole-sample inference statistics** (n = 23,213, all countries, all employment types).

Standard error, confidence intervals, Welch's t-test with Satterthwaite degrees of freedom, Pearson and Spearman correlation, simple and log-linear regression — each implemented from its definition and then checked against the library implementation.

Two results from this chapter shape the rest of the analysis. Spearman's ρ (0.440) is more than twice Pearson's r (0.196), which says the salary–experience relationship is monotonic but not linear. And the remote-vs-in-person salary gap, though significant at p ≈ 4e-29, is confounded: remote work is more common in high-paying markets, so the comparison partly measures country rather than working mode.

**Chapter B — multiple regression** (n = 6,144, defined population).

Four models, each answering the question the previous one raised:

| Model | Specification | R² |
|-------|---------------|-----|
| 1 | `log(salary) ~ WorkExp + is_US + is_India` | 0.589 |
| 2 | `log(salary) ~ WorkExp` — same sample, for honest comparison | 0.154 |
| 3 | Separate regressions within each country | 0.07 / 0.09 / 0.26 |
| 4 | Model 1 plus `WorkExp × country` interactions | 0.619 |

Country is encoded as two dummy variables with **Germany as the explicit reference category**. The dummies are built by direct comparison rather than `pd.get_dummies(drop_first=True)`, which drops the first category alphabetically and would silently redefine every coefficient the moment another country were added.

Salary is modelled on the natural logarithm. Ordinary least squares fits the conditional *mean*, which a right-skewed distribution pulls upwards; fitting on logarithms and back-transforming with `exp()` yields the geometric mean, close to the median for such data. Coefficients then read as percentages: `exp(0.018) = 1.018`, i.e. +1.8% per year of experience, compounding rather than adding.

**Diagnostics.** Residuals against fitted values, a Q–Q plot, and residuals grouped by country. Medians sit at zero for all three countries, so the model is not systematically biased for any market. The tails are heavier than normal (kurtosis 11.05), driven by records implausible for their own labour market — a US data engineer with twelve years of experience reporting 1,200 USD per year — which is the price of the deliberately permissive lower threshold.

---

## Limitations

- **Non-response is not correctable.** Response rates vary by country and by experiencelevel; junior developers answer the income question less often, so the Indian sample is biased towards more experienced respondents and the estimated Indian salary level is likely too high.
- **Excluded groups.** Freelancers, the self-employed, students and the non-employed fall outside the population; the results say nothing about them.
- **Residual data-entry errors.** Some implausible records survive the lower threshold and make the left tail of the residual distribution heavier.
- **Extrapolation.** Indian and German fitted salaries converge at around 23 years of experience, but the Indian sample contains very few observations at that level. This is a projection of an observed trend, not a measured fact about senior developers.
- **Omitted factors.** Role, seniority, company size, industry and region within a country are not in the model. Role composition is similar across the three countries, so it is unlikely to distort the country coefficients, but these factors account for much of the unexplained variance — particularly in the US, where experience explains only 7% of it.
- **Association, not causation.** The model measures statistical association within one survey sample. It does not establish that gaining experience causes a salary increase of a given size, and it does not support predictions about individual developers.
- **Single survey, single year.** Stack Overflow respondents are self-selected and are not a random sample of the developer population in any of the three countries.

---

## Repository

```
developer-salary-inference/
├── notebooks/
│   └── salary_inference_statistics.ipynb   # full analysis
├── data/                                   # not tracked — see .gitignore
│   ├── results.txt                         # survey_results_public.csv
│   └── schema.txt                          # survey_results_schema.csv
├── STATISTICS_NOTES.md                     # study notes on the methods used
├── requirements.txt
└── README.md
```

The dataset is not committed. Download it from [survey.stackoverflow.co](https://survey.stackoverflow.co/), place both files in `data/`, and run the notebook top to bottom.

---

## Tools

Python · pandas · numpy · scipy · statsmodels · matplotlib · seaborn · Jupyter
