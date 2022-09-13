
### Setup

In einem ersten Schritt müssen einige wichtige R-Bibliotheken importiert
werden. Wichtig für die Analyse der Zitationen über Crossref ist die
[`rcrossref`-Bibliothek](https://github.com/ropensci/rcrossref), die den
Datenabzug über einie der [von Crossref bereitgestellten Schnittstellen
(API)](https://www.crossref.org/documentation/retrieve-metadata/rest-api/)
erleichtert.

``` r
# tidyverse packages https://www.tidyverse.org/ 
library(dplyr) 
library(tidyr)
library(ggplot2)
# ropensci client for crossref
library(rcrossref) # R Client Crossref remote::install_github("ropensci/rcrossref)
# sna
library(sna)
library(network)
library(ggnet)
```

### Data

Zunächst sollen JASIST Publikationsmetadaten über die Crossref
Schnittstelle gezogen werden. Im folgenden Code-Schnippet wird die
Funktion `cr_works` aus der `rcrossref`-Bibliothek aufgerufen. Die
Funktion hat zwei benannte Argumente: `filter` und `limit`. Während
`limit` die Zahl der zurückgegebenen Treffer begrenzt, wird mit `filter`
angegeben, welche Werke aus der Gesamtmenge der Titel auf crossref
ausgegeben werden soll. [Weitere
Filtereinstellungen](https://docs.ropensci.org/rcrossref/reference/filters.html)
sind in der entsprechenden Dokumentation zur R-Bibliothek zu finden.

Die Treffermenge wird mit dem `<-`-Operator der Variable `jasist`
zugewiesen. `jasist`, d.h. der Output der Funktion `cr_works`, gibt eine
Liste mit drei Elementen zurück, von denen aber für die weitere Analyse
nur das Element `data` interessiert, das die Publikationsmetadaten
enthält.

Der Einfachheit halber wird also die Liste mit dem Namen `data` aus
`jasist`-Liste in eine eigene Variable (`jasist_md`) geschrieben.

``` r
jasist <- rcrossref::cr_works(filter = list(
  issn = "2330-1643",
  from_pub_date = "2020-01-01",
  until_pub_date = "2020-12-31"
), limit = 250)
jasist_md <- jasist$data
jasist_md
#> # A tibble: 126 × 35
#>    alternative.id    archive container.title   created deposited published.print
#>    <chr>             <chr>   <chr>             <chr>   <chr>     <chr>          
#>  1 10.1002/asi.24432 Portico Journal of the A… 2020-1… 2021-04-… 2021-05        
#>  2 10.1002/asi.24363 Portico Journal of the A… 2020-0… 2020-04-… 2020-05        
#>  3 10.1002/asi.24435 Portico Journal of the A… 2020-1… 2020-11-… 2020-12        
#>  4 10.1002/asi.24429 Portico Journal of the A… 2020-1… 2021-04-… 2021-05        
#>  5 10.1002/asi.24279 Portico Journal of the A… 2019-0… 2021-06-… 2020-05        
#>  6 10.1002/asi.24348 Portico Journal of the A… 2020-0… 2020-11-… 2020-12        
#>  7 10.1002/asi.24407 Portico Journal of the A… 2020-0… 2020-08-… 2020-09        
#>  8 10.1002/asi.24395 Portico Journal of the A… 2020-0… 2021-03-… 2021-02        
#>  9 10.1002/asi.24362 Portico Journal of the A… 2020-0… 2020-12-… 2021-01        
#> 10 10.1002/asi.24366 Portico Journal of the A… 2020-0… 2020-12-… 2021-01        
#> # … with 116 more rows, and 29 more variables: published.online <chr>,
#> #   doi <chr>, indexed <chr>, issn <chr>, issue <chr>, issued <chr>,
#> #   member <chr>, page <chr>, prefix <chr>, publisher <chr>, score <chr>,
#> #   source <chr>, reference.count <chr>, references.count <chr>,
#> #   is.referenced.by.count <chr>, subject <chr>, title <chr>, type <chr>,
#> #   update.policy <chr>, url <chr>, volume <chr>, language <chr>,
#> #   short.container.title <chr>, assertion <list>, author <list>, …
```

### Referenzanalyse

Wie viele und welche Publikationen referenzierten JASIST Artikel 2020?

`jasist_md` enthält die Publikationsmetadaten zu Aufsätzen aus der
Zeitschrift JASIST aus dem oben als Filter angegebenen Zeitraum. Um den
folgenden Code-Schnipsel zu verstehen, bietet es sich an, die
`jasist_md`-Daten etwas genauer anzuschauen. In RStudio geht dies leicht
über den Reiter *Environment*. Mit einem Klick auf die jeweilige
Variable lässt sich der Inhalt näher untersuchen.

![](img/table_symbol.png)

Mit einem Klick auf das Tabellensymbol am rechten Rand der Zeile der
Variable lässt sich der Variableninhalt in einer eigenen Tabelle
einsehen.

In dieser Ansicht entsprich eine Zeile einem Datensatz (=
Zeitschriftenaufsatz). Die letzte Spalte mit dem Titel *reference*
enthält ihrerseits eine Liste. Es handelt sich hierbei um eine Liste in
einer Liste.

Diese Verschachtelung ist wichtig, um den folgenden Code-Schnipsel zu
verstehen.

Hier werden aus `jasist_md` drei Spalten selektiert: `doi`, `title` und
`reference`. Die ersten beiden enthalten schlicht Strings (also
einfachen Text). Der komplexe Datentyp in der `reference`-Spalte lässt
sich nicht ohne Weiteres in einer Tabellenform darstellen; man müsste
hier eigentlich eine Tabelle in eine Zelle einfügen, was die Handhabung
erheblich erschweren würde.

Hier kommt nun die Funktion `unnest` ins Spiel, die die verschachtelte
Liste (*nested list* auf Englisch) gewissermaßen auspackt.

Die Logik dahinter entspricht folgendem Schema:

    | A | B | a b c | --> unnest --> | A | B | a |
    | X | Y | x y z |                | A | B | b |
                                     | A | B | c |
                                     | X | Y | x |
                                     | X | Y | y |
                                     | X | Y | z |

Es wird also für jedes Element in der verschachtelten Liste der
übergeordnete Datensatz kopiert.

Im konkreten Beispiel führt dies dazu, dass `jasist_cit` letztlich eine
Zeile je in einem JASIST-Aufsatz zitierten Titel aufweist, wobei die
ersten beiden Spalten DOI und Titel des JASIST-Aufsatzes beinhalten (und
so oft wiederholt werden, wie es Referenzen im jeweiligen Aufsatz gibt),
die restlichen Spalten jedoch die bibliographischen Details des
zitierten Titels.

``` r
# referenzen
jasist_cit <- jasist_md %>%
  select(doi, title, reference) %>%
  unnest(reference)
jasist_cit
#> # A tibble: 4,443 × 15
#>    doi    title     key   unstructured   issue doi.asserted.by first.page DOI   
#>    <chr>  <chr>     <chr> <chr>          <chr> <chr>           <chr>      <chr> 
#>  1 10.10… Describi… e_1_… Andersen J.(2… <NA>  <NA>            <NA>       <NA>  
#>  2 10.10… Describi… e_1_… Atlassian Cor… <NA>  <NA>            <NA>       <NA>  
#>  3 10.10… Describi… e_1_… Atlassian Cor… <NA>  <NA>            <NA>       <NA>  
#>  4 10.10… Describi… e_1_… <NA>           2     crossref        139        10.22…
#>  5 10.10… Describi… e_1_… <NA>           <NA>  <NA>            <NA>       <NA>  
#>  6 10.10… Describi… e_1_… <NA>           <NA>  <NA>            <NA>       <NA>  
#>  7 10.10… Describi… e_1_… <NA>           <NA>  <NA>            <NA>       <NA>  
#>  8 10.10… Describi… e_1_… <NA>           2     crossref        135        10.10…
#>  9 10.10… Describi… e_1_… <NA>           <NA>  <NA>            <NA>       <NA>  
#> 10 10.10… Describi… e_1_… Entertainment… <NA>  <NA>            <NA>       <NA>  
#> # … with 4,433 more rows, and 7 more variables: article.title <chr>,
#> #   volume <chr>, author <chr>, year <chr>, journal.title <chr>,
#> #   volume.title <chr>, series.title <chr>
```

Die Vorschau gibt schon die Zahl der Zeilen (4443) an. Möchte man sich
jedoch per Funktion explizit ausgeben lassen, kann man hierfür die
`nrow()` verwenden:

``` r
nrow(jasist_cit)
#> [1] 4443
```

Verteilung

``` r
# referenzen per artikel
cit_stat <- jasist_cit %>%
  group_by(doi) %>%
  summarise(ref_n = n_distinct(key),
            ref_cr = sum(!is.na(DOI))) %>%
  mutate(prop = ref_cr / ref_n)
cit_stat
#> # A tibble: 92 × 4
#>    doi               ref_n ref_cr  prop
#>    <chr>             <int>  <int> <dbl>
#>  1 10.1002/asi.24256   116     71 0.612
#>  2 10.1002/asi.24257    46     33 0.717
#>  3 10.1002/asi.24258    29     22 0.759
#>  4 10.1002/asi.24259    71     45 0.634
#>  5 10.1002/asi.24260     2      1 0.5  
#>  6 10.1002/asi.24262    71     55 0.775
#>  7 10.1002/asi.24279    40     14 0.35 
#>  8 10.1002/asi.24280    80     37 0.462
#>  9 10.1002/asi.24282    33     18 0.545
#> 10 10.1002/asi.24285     3      3 1    
#> # … with 82 more rows
```

top 10

``` r
cit_stat %>%
  arrange(desc(ref_n))
#> # A tibble: 92 × 4
#>    doi               ref_n ref_cr  prop
#>    <chr>             <int>  <int> <dbl>
#>  1 10.1002/asi.24387   122     67 0.549
#>  2 10.1002/asi.24256   116     71 0.612
#>  3 10.1002/asi.24354   113     44 0.389
#>  4 10.1002/asi.24339   105     63 0.6  
#>  5 10.1002/asi.24342   103     55 0.534
#>  6 10.1002/asi.24362    96     76 0.792
#>  7 10.1002/asi.24390    96     51 0.531
#>  8 10.1002/asi.24358    95     47 0.495
#>  9 10.1002/asi.24367    89     59 0.663
#> 10 10.1002/asi.24415    87     72 0.828
#> # … with 82 more rows
```

Parameter

``` r
# referenzen
summary(cit_stat$ref_n)
#>    Min. 1st Qu.  Median    Mean 3rd Qu.    Max. 
#>    2.00   31.75   45.00   48.29   63.00  122.00
```

``` r
# anteil referenzen mit crossref doi
summary(cit_stat$prop)
#>    Min. 1st Qu.  Median    Mean 3rd Qu.    Max. 
#>  0.1429  0.5097  0.5969  0.6050  0.7228  1.0000
```

Verteilung Referenzen je Artikel

``` r
ggplot(cit_stat, aes(ref_n)) +
  geom_density(fill = "#56b4e9") +
  theme_minimal() +
  labs(x = "Anzahl Referenzen je JASIST-Artikel")
```

![](uebung_files/figure-gfm/unnamed-chunk-9-1.png)<!-- -->

### Netzwerkanalyse

Zitationsdaten

``` r
cit_df <- jasist_cit %>%
  filter(!is.na(DOI)) %>%
  select(doi, ref_doi = DOI) %>%
  mutate(ref_doi = tolower(ref_doi)) %>%
  distinct()
```

Meist zitierte Arbeiten

``` r
cit_df %>%
  count(ref_doi, sort = TRUE)
#> # A tibble: 2,599 × 2
#>    ref_doi                           n
#>    <chr>                         <int>
#>  1 10.1002/asi.24232                 6
#>  2 10.1002/asi.20019                 4
#>  3 10.1191/1478088706qp063oa         4
#>  4 10.1016/0147-1767(85)90062-8      3
#>  5 10.1016/j.lisr.2009.10.003        3
#>  6 10.1016/s0306-4573(99)00027-8     3
#>  7 10.1108/eum0000000007145          3
#>  8 10.1145/2740908.2742839           3
#>  9 10.11645/11.1.2188                3
#> 10 10.2307/41409970                  3
#> # … with 2,589 more rows
```

#### Zitationsmatrix und visualisierung

``` r
# nur Artikel mit mehr als einer Zitation
dois_cit <- cit_df %>%
  count(ref_doi, sort = TRUE) %>%
  filter(n > 1)
my_cit <- cit_df %>%
  filter(ref_doi %in% dois_cit$ref_doi)
my_mat <- as.matrix(table(my_cit$doi, my_cit$ref_doi))
dim(my_mat)
#> [1]  64 107
```

Netzwerkobjekt für die Visualisierung

#### Welche Artikel sind bibliographisch gekoppelt

``` r
mat_t <- my_mat %*% t(my_mat)
dim(mat_t)
#> [1] 64 64
```

Visualisierung

``` r
net <- network::as.network.matrix(mat_t, directed = FALSE)
ggnet::ggnet2(net, size = "degree", color = "#56b4e9", alpha = 0.8) +
  geom_point(aes(color = color), color = "grey90") +
  guides(color = "none", size = "none") +
  labs(title = "Bibliographic coupling JASIST 2020")
```

![](uebung_files/figure-gfm/unnamed-chunk-14-1.png)<!-- -->

#### Welche Artikel werden co-zitiert in JASIST-Artikeln von 2020

``` r
mat_t <- t(my_mat) %*% (my_mat)
dim(mat_t)
#> [1] 107 107
```

Visualisierung

``` r
net <- network::as.network.matrix(mat_t, directed = FALSE)
ggnet::ggnet2(net, size = "degree", color = "#56b4e9", alpha = 0.8) +
  geom_point(aes(color = color), color = "grey90") +
  guides(color = "none", size = "none") +
  labs(title = "Co-Citation network JASIST 2020")
```

![](uebung_files/figure-gfm/unnamed-chunk-16-1.png)<!-- -->
