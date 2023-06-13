---
layout: post
title:  "Bot to answer questions in Oracle Forums using Chatgpt 3.5 turbo model"
date:   2023-06-12 21:38:02 -0700
categories: jekyll update
permalink: /R/project1
---

---
stumbled upon parallel processing while working on Chatgpt API requests automation.

Project details:
              - Collect data from forums.oracle.com, manually scraped data from forums.oracle.com for category (Oracle Data base XE)
              - Automate chatgpt single chat completion API calls from R
              - Make atleast 3 requests per minute to chatgpt
                
Challenges faced: 
              - Data collection using RSelenium, was unsuccessful due to errors in going to next page in the pagination section.
                So, had to collect data manually (took 60 mins to collect urls for all open posts :-))
                1400 forums posts urls to be crawled and wrangled as an html page, is where the issue lies. 
                              As normal for loops in R taking lot of time, switched to apply functions like lapply in base packages
                              performance improved significantly. however, the http requests are independent to each other; 
                              ideally, parallel processing would be a good fit. 
                              Let's see the performance gains and the details of parallel processing in this post
Code optimizations:
              - Make atleast 3 requests per minute to chatgpt [Garbage collection, became a cocern after I noticed RStudio hanged few times, while running the program for all 1400 chat completions]
              - 1. growing data frame in size, instead of defining the predeterministic size (downside - R gets confused with memory allocation)
              - 2. For loops in R are too slow, instead use apply functions from base package.
              - 3. lapply functions with lot of intermediary variables. unnecessary variable usage inside the functions.
              - 4. use multiple processes/sessions to use the multicore CPUs to the fullest
              
Below are code snippets to solve the problem statement:

<R>
# read each html page in Data folder and retreive post URLS from them
# get the post details for each URL and store it in a data frame
# feed these post details to chatgpt

library(readr)
library(dplyr)
library(httr)
library(rvest)
library(jsonlite)
library(foreach)
library(future.apply)
library(future)
library(promises)

source("collect_chatpgt.R") # package contains the chatgpt request API automation

pgurls_df <- data.frame ("url", "pagenum")
colnames(pgurls_df) <- c("url", "pagenum")
data.files <- list.files(path = getwd(), pattern = "*.html", full.names = TRUE)
for (i in 1:length(data.files)) {
  read_html (read_file(file = data.files[i])) -> response_body_html
  response_body_html %>% html_nodes(".rw-ListItem-primaryText a") %>% html_attr(name = "href") -> pgUrls
  pgUrls %>% as.data.frame.character() %>% mutate(pagenum=i) -> pgUrls
  colnames(pgUrls) <- c("url", "pagenum")
  merge.data.frame (pgurls_df, pgUrls, by = intersect(names(pgurls_df), names(pgUrls)), all.x = TRUE, all.y = TRUE) -> pgurls_df
}

# remove unused objects from memory
rm(list=c("pgUrls", "response_body_html", "i"))

# remove duplicate urls from the pgurls_df, if any
df_unique <- pgurls_df[!duplicated(pgurls_df$url), ]
df_unique$gpt_response <- "gpt"

e_cnt =0
w_cnt = 0
nreq = 0

n = nrow(df_unique)
core_output = data.table::data.table('n' = 1:n
                                     , 'url' = rep("", n)
                                     , 'qa_title' = rep("", n)
                                     , 'qa_attrs' = rep("", n)
                                     , 'qa_body' = rep("", n)
                                     , 'prompt_content' = rep("", n)
                                     , 'chatgpt_response' = rep("", n))

for(i in 1:n){
  core_output$url[i] = df_unique[i, "url"]
}

meta_output <- data.table::data.table('n' = 1:n
                                      , 'url' = rep("", n)
                                      , 'error' = rep("" , n)
                                      , 'message' = rep ("" , n)
                                      , 'status_code' = rep("", n)
)

for(i in 1:n){
  meta_output$url[i] = df_unique[i, "url"]
}

collect_posts_data <- function(){  
  lapply(core_output$n, function(i){
  as.character(df_unique[i, "url"]) -> gurl
  core_output$url[i] = gurl
  gurl_data <- get_url_data(gurl)
  gurl_data_html <- content(gurl_data, as = "text") %>% read_html()
  qatitle <- gurl_data_html %>% html_nodes(".ds-Question-text .ds-Question-title") %>% html_text()
  qaattrs <- gurl_data_html %>% html_nodes(".ds-Question-text .ds-Question-attrs") %>% html_text2()
  qabody <- gurl_data_html %>% html_nodes(".ds-Question-text .ds-Question-body") %>% html_text()
  if(length(qatitle)>0) core_output$qa_title[i] = as.character(qatitle)
  if(length(qaattrs)>0) core_output$qa_attrs[i] = as.character(qaattrs)
  if(length(qabody)>0) core_output$qa_body[i] = as.character(qabody)
  })
}

system.time({ collect_posts_data() })


plan(multisession, workers = parallel::detectCores()) 
# Define the future loop
future_lapply(seq_along(2:nrow(df_unique)), function(i) {
  nreq = i
  v_rpm <- 3 # requests per minute
  v_tpm <- 40000 # token per minute
  tryCatch({
    if(w_cnt >0) {Sys.sleep(time = 40)} else {Sys.sleep(time=20)}
    if (length(core_output$qa_body[i]) > 0) {
      gpt_prompt <- paste0("Question title:\n", core_output$qa_title[i], "\nquestion details:", core_output$qa_body[i])
      gpt_response <- make_req_gpt(gpt_prompt)
      if(gpt_response$status_code != 200){
        warning(paste0("Request completed with error. Code: ", gpt_response$status_code
                       , ", message: ", gpt_response$error$message))
        meta_output$url[i] = gurl
        meta_output$error[i] = gpt_response$error$message
        meta_output$message[i] = gpt_response$error$message
        meta_output$status_code[i] = gpt_response$status_code
      }
      else {
        response_content <- parse_json(content(gpt_response, "text", encoding = "UTF-8"))$choices[[1]]$message$content
        core_output$url[i] = gurl
        core_output$prompt_content[i] = gpt_prompt
        core_output$chatgpt_response[i] = response_content
      }
      print(paste0("gpt response - " , response_content))
    }
  }, 
  error = function(e) {e_cnt = e_cnt+1
  print(e)}, 
  warning = function(w) {
    w_cnt = w_cnt+1
    print(w)}
  , finally = print("in Finally section"))  
}) -> gpt_responses 

# Clean up
future::plan("sequential")

t2 <- Sys.time()
print(paste0("time elapsed:" , t2-t1) 

</R>
