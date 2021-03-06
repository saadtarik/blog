The weekend before Valentine's Day, my friend Pippa and I were sitting at our kitchen table trying to decide how to celebrate. We wanted a way to show all of our friends how much we love and appreciate them! Complicating the Valentine's a bit is the fact that we have friends in multiple countries, so delivering something in person would be impossible and sending something through the mail would add up quickly. Fairly quickly we decided to send something via email. We put out a post on Facebook asking people to respond if they wanted a valentine from us. With those responses plus people we added, we had 34 people on the list!

*add image!*

We were talking about the poem generator I'd built a few weeks ago and thought somewhere similar to that would be a good place to start. I'd read Sarah Lotspeich and Lucy D'Agostino McGowan's [secret sampling](http://livefreeordichotomize.com/2017/11/15/secret-sampling/) post about sending emails through R will `ponyexpress`. They'd used it to send messages about secret santa over the holidays, but it seemed like we could use that idea for sending valentines! The final idea was to randomly generate a conversation heart for each of our friends with markov chains and then send them via email with `ponyexpress`. This would also give me a chance to use the `markovifyR` package that I'd heard such good things about!

For the rest of the afternoon (we're quite skilled at procrastination), powered by Whitney Houston and Mary Chapin Carpenter, we entered nearly 500 possible conversation heart sayings into a google sheet so we could both be working at the same time without too many duplicates. Some were much better than others; our creativity really started to fade around the 300 mark. The 300 to 500 range included gems such as "discount candy" and "wuts ur sign" (there are some astrology fans in my house, I unfortunately am not one of them).

To start, we needed to read in the sheets from google with `googlesheets` package. Then we cleaned it, due to some predictable errors that go along with human-entered data.

``` r
# packages

library(googlesheets) # read in the sheet 
library(tidyverse) # clean the text
library(markovifyR) # make the markov chains
library(magick) # image processing
library(ponyexpress) # send the emails

# data

hearts <- gs_title("conversation_hearts") # read in my conversation_hearts sheet

sayings <- hearts %>%
  gs_read(ws = 1) # the text we thought of was in the first sheet

# cleaning the text

sayings$text <- sayings$text %>%
  str_replace_all("[[:punct:]]", "") %>% # standardize the words
  str_replace_all(c("what" = "wut",
                    " are " = "r",
                    "love" = "luv",
                    " for " = "4",
                    "too " = "2",
                    " to " = "2",
                    "your" = "ur",
                    "you're" = "ur",
                    "thanks" = "thx",
                    "night" = "nite",
                    " you are " = "ur",
                    "you " = "u ",
                    " you " = " u ")) # all the abbreviations we agreed to use to be more ~hip~
```

Next I needed to create a model that generate the text! Luckily the `markovifyR` package has pretty good documentation and I could look to Malcolm Barrett's [Stochastic Shakespeare: Sonnets Produced by Markov Chains in R](https://malco.io/2018/01/28/shakespeare/) post for some guidance as well.

``` r
model <- generate_markovify_model( # build the text model
  input_text = sayings$text, # use the text as the input
  max_overlap_ratio = 0.85,
  max_overlap_total = 40
)
```

The only other step for the text was to actually generate the messages. It was difficult because each input saying was so short that there weren't a whole lot of probabilities to work from. It was a challenge to create enough distinct messages, but we made a workaround for that.

``` r
texts_to_annotate <- markovify_text( # make the strings
  markov_model = model, # use the model we just made
  maximum_sentence_length = NULL, # without a max sentence length
  output_column_name = 'heart_sayings',
  count = 100,
  tries = 600,
  only_distinct = TRUE, # we still just wanted distinct messages 
  return_message = FALSE
)

head(texts_to_annotate)
```

    ## # A tibble: 6 x 2
    ##   idRow heart_sayings                
    ##   <int> <chr>                        
    ## 1     1 like u as a button           
    ## 2     2 u turn me on tinder          
    ## 3     3 u have a sweet valentines day
    ## 4     4 i like u as a button         
    ## 5     5 object of my life            
    ## 6     6 i like like u as a friend

These are the sayings it built! Evidently some of our text cleaning didn't quite work but honestly that's okay with me for this type of project. None of my friends know enough R to call me out on it, I think :D And I didn't want to spend too much time doing all of the cleaning.

We decided to sample enough messages from this list for each of the people on our email list.

``` r
friends <- hearts %>%
  gs_read(ws = 3) %>%
  mutate(email = address) %>% # i wanted to change the name of this variable
  select(-address)

final <- texts_to_annotate %>%
  sample_n(nrow(friends), replace = TRUE) # take as many samples as we have friends in the dataset i.e. friends to email

friends$text <- final$heart_sayings # add the text to the friends data for when we send them
```

While Pippa finished up the brainstorming, I started to work on the image processing part. From blog posts I'd read `magick` has always seemed like a cool package, but until now I hadn't done much that required it. We decided to superimpose the generated text onto a heart to create the final image to send. We'd include it in the email with a bit of text for context!

Warning, the part ahead is definitely the least elegant! I use the dreaded for loop because I learned Python first and admittedly haven't ever really taken the time to get comfortable for `map` or `lapply`. Some day I will...

I wrote a loop to create an image for each person and then saved that image as `{name}.png`

``` r
heart <- image_read("C:\\Users\\katie\\Documents\\fun_projects\\conversation_hearts\\pink-heart.png") # the base heart image we used

all_paths <- c() # create an empty vector for local paths to the images

for (i in 1:nrow(friends)){
  path <- paste0("C:\\Users\\katie\\Documents\\conversation_hearts\\images\\", friends$name[[i]], ".png") # reference the images by the person's first name so I can track them easily
  all_paths <- c(all_paths, path) # add this path to the vector. For only 32 this doesn't get too slow.
  img <- heart %>% # start with the base image
            image_scale("600") %>% # make it big! <3
            image_annotate(friends$text[[i]], color = "white", gravity = "center", size = 40) %>% # write the generated text in white in the middle of the heart with size 40 font
            image_write(path = path, format = "png") # save the image as {name}.png 

}

friends$path <- all_paths # add paths to the dataset
```

While playing around with the `ponyexpress` package, I figured it would be easier to include the images if they were stored online and GitHub seemed like the natural option. So I wrote another similar loop to create GitHub paths for the images.

``` r
github_paths <- c()

for (i in 1:nrow(friends)){
  p <- paste0("https://raw.githubusercontent.com/katiejolly/conversation-hearts/master/images/", friends$name[[i]], ".png")
  github_paths <- c(github_paths, p) # this time we don't have to create another image, just the path!
}

friends$github_paths <- github_paths # add github paths to images
```

At this point we were done with all of the prep work! The next task was formatting the emails and then sending them!

As a note, I used two separate R scripts for these two halves of the work. It helped me think in two separate sections.

We wrote the skeleton of the email and saved it as `body`. We used the GitHub paths to include the images! We also included a gif at the end just for extra fun.

``` r
body <- "Dear {name},


Happy Valentine's Day!! We made a special heart just for you :D

<img src = '{github_paths}'> </img>

(As an aside, we randomly generated the text with R and have no idea what we sent to you. Ask Katie if you want to know more!)


We love having you in our lives and wanted to take the time to tell you! 


Much love to you <3

Katie, Pippa, and Matthew


<img src = 'https://media.giphy.com/media/26xBRiIYbyjCzYMAU/giphy.gif'> </img>"
```

I then added my name and email as a blank row in the table so that ponyexpress would recognize my account.

``` r
friends <- rbind(friends, c("Katie Jolly", "katiejolly6@gmail.com", "", ""))
```

And it was an easy decision to use the glitter template!

``` r
our_template <- glue::glue(glitter_template)
```

Then we just put the pieces together in the `parcel_create()` function!

``` r
parcel <- parcel_create(friends,
                        sender_name = "Katie Jolly",
                        sender_email = "katiejolly6@gmail.com",
                        subject = "a valentine for you!",
                        template = our_template)
```

To see our great creation we ran `parcel_preview()`

``` r
parcel_preview(parcel)

# parcel_send(parcel)
```

It produced this card!

*insert card image*

We loved the cards we were able to make! And we got lots of replies from friends who similarly loved the cards :D

Now my house is now in an R mood so we are working on using this general outline to do some projects with the student run food co-op on campus! Right now we are just in the idea phase, but I'm having a lot of fun seeing little projects turn into impactful tools for people and causes I care about. I'm hoping to be able to write about that partnership in the near future!
