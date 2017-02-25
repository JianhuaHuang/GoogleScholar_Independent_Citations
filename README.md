-   [Web-scraping Independent Citations from Google Scholar](#web-scraping-independent-citations-from-google-scholar)
    -   [My Papers](#my-papers)
    -   [citing papers](#citing-papers)
    -   [Find Dependent/Independent Citations](#find-dependentindependent-citations)

Web-scraping Independent Citations from Google Scholar
======================================================

My Papers
---------

``` r
# https://cran.r-project.org/web/packages/RSelenium/vignettes/RSelenium-basics.html
library(RSelenium)
library(rvest)
library(dplyr)

rm(list = ls())
sleep <- function() Sys.sleep(1 + abs(rnorm(1, mean = 1)))
home_link <- 'https://scholar.google.com/citations?user=19Z-MdUAAAAJ&hl=en' # Liang


# double click the selenium-server-standalone-3.0.1.jar file to run it
# Or within the command line terminal, run:
# java -jar selenium-server-standalone-3.0.1.jar
rd <- remoteDriver(browserName = "firefox")

rd$open()
rd$navigate(home_link)

sleep()

# more_button <- rd$findElement(using = 'id', 'gsc_bpf_more')
# more_button$clickElement()  # Click multiple times, if you have lots of papers
sleep()

homepage <- read_html(rd$getPageSource()[[1]])

## My papers
my_papers <- rd$findElements(using = 'class', 'gsc_a_at')

get_my_paper_ref <- function(my_paper) {
  # Get the BibTex reference for my own papers
  my_paper$clickElement()
  sleep()
  
  export_button <- rd$findElement(using = 'id', 'gsc_btn_exp-bd')
  export_button$clickElement()
  sleep()
  
  bibtex <- rd$findElement(using = 'css selector', 'li')
  bibtex$clickElement()
  sleep()
  
  # paper_bibtex <- read_html(rd$getPageSource()[[1]]) %>% 
  #   html_text('body')
  body <- rd$findElement(using = 'css selector', 'body')
  paper_bibtex <- body$getElementText()[[1]]
  sleep()
  
  rd$goBack()
  Sys.sleep(.2)
  rd$goBack()
  Sys.sleep(abs(rnorm(1, mean = 2)))
  return(paper_bibtex)
}

# my_paper_refs <- c()
my_paper_refs <- get_my_paper_ref(my_papers[[1]])

for(p in 2:length(my_papers)) {
  # more_button is already clicked for p = 1
  # more_button <- rd$findElement(using = 'id', 'gsc_bpf_more')
  # more_button$clickElement()  # Click multiple times, if you have lots of papers
  sleep()
  
  ## My papers
  my_papers <- rd$findElements(using = 'class', 'gsc_a_at')
  my_paper_refs[p] <- get_my_paper_ref(my_papers[[p]])
}

rd$close()
saveRDS(my_paper_refs, file = 'Data/my_paper_refs.rds')
```

citing papers
-------------

``` r
rd$open()
rd$navigate(home_link)
sleep()

# more_button <- rd$findElement(using = 'id', 'gsc_bpf_more')
# more_button$clickElement()  # Click multiple times, if you have lots of papers
sleep()

homepage <- read_html(rd$getPageSource()[[1]])

citing_papers <- html_nodes(homepage, '.gsc_a_ac') %>% html_attr('href')

get_citing_paper_ref <- function(citing_paper) {
  # get eh BibTex reference for the citing papers
  rd <- remoteDriver(browserName = "firefox")
  rd$open()
  Sys.sleep(.5)
  rd$navigate(citing_paper)
  sleep()
  cite_links <- rd$findElements(using = 'link text', 'Cite')
  
  # get_BibTex function
  get_BibTex <- function(cite_link) {
    cite_link$clickElement()
    sleep()
    bibtex <- rd$findElement(using = 'class', 'gs_citi')
    bibtex$clickElement()
    sleep()
    
    body <- rd$findElement(using = 'css selector', 'body')
    paper_bibtex <- body$getElementText()[[1]]
    
    rd$goBack()
    sleep()
    
    close_popup <- rd$findElement(using = 'id', 'gs_cit-x')
    close_popup$clickElement()
    sleep()
    
    return(paper_bibtex)
  }
  
  cites_p1 <- sapply(cite_links, get_BibTex)  # cites in the first page
  
  # Some papers have more than 10 citations crossing multiple page
  # In this case, we need to click the next button to get all citations
  next_pages <- rd$findElements(using = 'class', 'gs_nma')
  cites_np <- list()  # cites in next pages
  
  if(length(next_pages) > 1) { 
    for(p in 2:length(next_pages)) {
      next_button <- rd$findElement(using = 'class', 'gs_ico_nav_next')
      next_button$clickElement()
      sleep()
      
      cite_links <- rd$findElements(using = 'link text', 'Cite')
      cites_np[[p-1]] <- sapply(cite_links, get_BibTex)
    }
  }
  
  rd$close()
  paper_bibtex <- c(cites_p1, unlist(cites_np))
}

# citing_paper_refs <- lapply(citing_papers[nchar(citing_papers) > 0],
#   get_citing_paper_ref)

citing_paper_refs <- list()
for(p in 1:sum(nchar(citing_papers) > 0)) {
  citing_paper_refs[[p]] <- get_citing_paper_ref(citing_papers[p])
}
saveRDS(citing_paper_refs, file = 'Data/citing_paper_refs.rds')
rd$close()
```

Find Dependent/Independent Citations
------------------------------------

``` r
mps <- readRDS('Data/my_paper_refs.rds')  # my papers
cps <- readRDS('Data/citing_paper_refs.rds')  # citing papers

no_cite <- length(mps) - length(cps)

mps.ncites <- sapply(cps, length)
mps.ncites <- c(mps.ncites, rep(1, no_cite))

refs <- data.frame(ID = rep(1:length(mps), mps.ncites), 
  MP = rep(mps, mps.ncites), 
  CP = c(unlist(cps), rep(NA, no_cite)))

# compare the authors
refs$MP_authors <- gsub('.*author=\\{', '', refs$MP) %>% 
  gsub('},.*', '', .) %>%
  strsplit(' and ')

refs$CP_authors <- gsub('.*author=\\{', '', refs$CP) %>% 
  gsub('},.*', '', .) %>%
  strsplit(' and ')

# the author formats are different in different papers, 
# keep only last name and the initial of other names, to make them consistent
Last_Initial <- function(names) {  
  last <- gsub(',.*', '', names)
  initials <- sapply(strsplit(names, ', '), function(x) gsub('[^A-Z]', '', x[2]))
  last_ini <- paste(last, initials, sep = ', ')
}

refs$MP_authors <- lapply(refs$MP_authors, Last_Initial)
refs$CP_authors <- lapply(refs$CP_authors, Last_Initial)

# check whether the authors are independent
my.authors = refs$MP_authors[3]
citing.authors = refs$CP_authors[3]

check.independence <- function(my.authors, citing.authors) { 
  ifelse(length(intersect(unlist(my.authors), unlist(citing.authors))) == 0,
    'Y', 'N')
  }

refs$Independent <- apply(refs[, c('MP_authors', 'CP_authors')], 1, function(x){
  check.independence(x[1], x[2])})
refs$Independent[is.na(refs$CP)] <- NA
saveRDS(refs, 'Data/paper.references.rds')
# write.csv(refs, 'Data/paper.references.csv', row.names = F)
```

**reference:**

-   <http://datascience-enthusiast.com/R/google_scholar_R.html>
