### Wikipedia: Presidential returns by state (1864-)

``` r
library(tidyverse)
git_dir <- "/home/jtimm/jt_work/GitHub/packages/uspols/data-raw/"
setwd(git_dir)
pres <- read.csv('us_pres_1864.csv') 
```

``` r
base_url <- 'https://en.wikipedia.org/wiki/United_States_presidential_elections_in_'
options(tigris_use_cache = TRUE, tigris_class = "sf")
states_full <- tigris::states(cb = TRUE) %>% 
  data.frame() %>%
  select(NAME, STATEFP, STUSPS) %>%
  rename(state_abbrev = STUSPS)

states <- states_full %>%
   filter(!NAME %in%  c('Pennsylvania',  'California') &
           !STATEFP %in% c('78', '69', '66', '72', '60')) %>%
  mutate(which_table = ifelse(NAME %in% c('New York', 'Missouri'), 3, 2)) 
```

``` r
states_correct <- list()
for (i in 1:nrow(states)) {
  states_correct[[i]] <- 
    paste0(base_url, gsub(' ', '_', states$NAME[i])) %>%
    xml2::read_html() %>%
    rvest::html_node(
      xpath = paste0('//*[@id="mw-content-text"]/div/table[', 
                     states$which_table[i],']')) %>%
    rvest::html_table(fill = TRUE) 
  
  x <- states_correct[[i]][,c(1, 2, 4)]
  colnames(x) <- c('year', 'candidate', 'vote_share')
  y <- states_correct[[i]][,c(1, 5, 7)]
  colnames(y) <- c('year', 'candidate', 'vote_share')
  
  states_correct[[i]] <- rbind(x, y) %>%
    mutate(candidate = gsub('\\[.*\\]|\\(.*\\)', '', candidate),
           year = gsub("\\D+", "", year),
           year = as.integer(substr(year, 1,4)),
           vote_share = as.numeric(gsub('^$|-|%', 0, vote_share))
           )
  } 

names(states_correct) <- states$NAME
states_correct1 <- states_correct %>%
  bind_rows(.id = 'state_name')  %>%
  filter(candidate != 'TBD')
```

``` r
##### PA -- 
url <- 'https://en.wikipedia.org/wiki/United_States_presidential_elections_in_Pennsylvania'

returns <- url %>%
  xml2::read_html() %>%
  rvest::html_node(xpath = '//*[@id="mw-content-text"]/div/table[2]') %>%
  rvest::html_table(fill = TRUE)  

x <- returns[,c(1, 5, 6)]
colnames(x) <- c('year', 'candidate', 'vote_share')
y <- returns[,c(1, 9, 10)]
colnames(y) <- c('year', 'candidate', 'vote_share')

returns1 <- rbind(x, y) %>%
  filter(year > 1860) %>%
  mutate(vote_share = gsub('^.* \\(', '', vote_share),
         vote_share = as.numeric(gsub('%.*$', '', vote_share)),
         candidate = gsub(' Howard ', ' H\\. ', candidate),
         candidate = gsub('Charles Evans', 'Charles E\\.', candidate),
         candidate = gsub('Winfield Scott', 'Winfield S\\.', candidate),
         state_name = 'Pennsylvania') %>%
  select(state_name, year, candidate, vote_share) %>%
  unique() %>%
  arrange(desc(vote_share)) %>%
  group_by(year) %>%
  slice(1:2) %>% 
  ungroup()
```

``` r
##### CA -- 
url <- 'https://en.wikipedia.org/wiki/United_States_presidential_elections_in_California'

returns_ca <- url %>%
  xml2::read_html() %>%
  rvest::html_node(xpath = '//*[@id="mw-content-text"]/div/table[2]') %>%
  rvest::html_table(fill = TRUE) 

returns_ca <- returns_ca[-1,] 

x <- returns_ca[,c(1, 2, 5)]
colnames(x) <- c('year', 'candidate', 'vote_share')
y <- returns_ca[,c(1, 6, 9)]
colnames(y) <- c('year', 'candidate', 'vote_share')

returns_ca1 <- rbind(x, y) %>%
  filter(year > 1860) %>%
  mutate(vote_share = as.numeric(gsub('%', '', vote_share)),
         candidate = gsub('^Y|^N|\\[.*\\]', '', candidate), 
         year = gsub('\\[.*\\]', '', year), 
         state_name = 'California') %>%
  filter(candidate != 'TBD')  %>%
  select(state_name, year, candidate, vote_share)
```

``` r
setwd(git_dir)
## hand corrections on the names --
#pres <- write.csv(returns_ca1, 'ca_pres_1864xx.csv')
returns_ca2 <- read.csv('ca_pres_1864.csv')
```

``` r
###
full <- bind_rows(states_correct1,
                  returns1,
                  returns_ca2)%>%
  mutate(vote_share = ifelse(year %in% c('1864', '1868') &
                               vote_share == 0,
                             NA, vote_share),
         candidate = trimws(candidate)) %>%
  na.omit() %>% 
  left_join(pres %>% select(-electoral_votes)) %>%
  arrange(desc(vote_share)) %>%
  group_by(state_name, year) %>%
  mutate(winner = row_number()) %>%
  ungroup()%>%
  rename(NAME = state_name) %>%
  
  left_join(states_full) %>%
  select(STATEFP:state_abbrev, NAME:winner) %>%
  rename(GEOID = STATEFP, state = NAME)

# setwd(pdir)
# #saveRDS(pres, 'pres_1864.rds')
# #saveRDS(full, 'pres_elections_state.rds')
# write.csv(full, 'pres_elections_state.csv', 
#           row.names = F)
```