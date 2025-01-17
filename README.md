***21/03/26. Note to my future self***: [lu0](https://github.com/lu0), listen, you forked it to implement it using Python on Taylor's newer albums (*Lover*, *folklore* and *evermore*) 💛💛 Please don't forget it :)

Original Readme file:
> # Text analysis and data visualization with Taylor Swift songs
> 
> 
> With only three days to go for Taylor Swift's much awaited 7th studio album -aka TS7 aka Lover-, there's quite a lot to be excited about. And as there's nothing I love more during the week preceding a Taylor's album release than listening over and over to all her old songs, I though I may as well go a bit further this time and dig deeper into her previous albums (have I mentioned I'm very excited?). So it's time to share with you what I've been doing for the past few days: **a not-so-short tutorial on text analysis and data visualization with Taylor Swift's lyrics**.
> 
> To be honest, it all started as way less ambitious project. I had her lyrics on my laptop and I thought I could do a couple of nice pink plots counting words. But this woman inspires me too much, ideas kept flowing and all of a sudden all this Taylor Swift analysis was fully out of hand. To try to keep some order, here are some of the topics I'll be covering below:
> 
> * Using the [**genius**](https://cran.r-project.org/web/packages/genius/genius.pdf) package to download lyrics
> * Using [**tidytext**](https://cran.r-project.org/web/packages/tidytext/tidytext.pdf) to tidy the lyrics so they're easy to analyze
> * Using ggplot to create **bar  plots with glitter** (you can't imagine how excited I am about this) - [**ggtextures**](https://github.com/clauswilke/ggtextures) will help us with that
> * Perform some basic sentiment analysis on the lyrics
> * Check to what extent her previous albums are related among them. And plotting that, of course
> * And last but not least and maybe my favorite part... we'll compute the correlation between all her songs to plot a network map of the way in which they're related to one another.
> 
> (Kind reminder: you'll be able to apply this to any kind of texts you want. It won't be as cool, though)
> 
> Are you... [Ready for it?](https://www.youtube.com/watch?v=wIft-t-MQuE)
> 
> ### First things first
> 
> First, of course, libraries. You may want to set your own working directory now if you're going to save your  plots with ggsave.
> 
> ```` r
> library(tidyverse) #can we actually do R without it? I'd say no
> library(genius) #we'll need this for the lyrics
> library(tidytext) #for text tidying
> library(ggtextures) #for glittery plots
> library(extrafont) #to add personalizzed fonts to ggplot output
> library(scales) #will be needed for percentage scales in ggplot
> library(widyr) #to find correlations between songs
> library(ggraph) #plotting network maps
> library(igraph) #same
> 
> 
> setwd("yourwd")
> ````
> 
> ### 1. Downloading the lyrics  with the Genius package
> 
> Getting lyrics from an artist is quite easy in R thanks to the Genius package. I won't get in detail here about how it works, but you can learn more about it [here](https://github.com/josiahparry/genius).
> 
> 
> Here, I'm using the `genius_album` function to download, one by one (not great, I know) her albums. All you need to specify to the function is the name of the artist and the album you want, and it'll get the lyrics for you. What you'll get back is a tibble with one row per sentence and information on the track title, track number and song line. Notice that the function won't return the  album's name, that we'll need later, so I'm adding it with a `mutate`call - happily Genius is `dplyr` compatible!
> 
> 
> You'll see that I've also added a filter call to the Fearless album, as Genius website stores the content of it with three extra songs which were bonus tracks but actually belong to the first Taylor album. I don't want them twice in there! (shoutout to my friend Nerea who was the one spotting it - swifties can't be tricked that easily). 
> 
> After downloading everything, I put it all together in a tibble called `tay`. Now we have all the data we need! You'll see that I have also deleted some things after joining - mainly the 1989 voice memos and some acoustic versions, to once again avoid having anything twice (or texts that are not songs).
> 
> 
> You'll notice a `save` call at the end of this chunk. I added this because retrieving all the data is a bit time consuming, and some times bad things happen, after all. Having it stored avoids having to do everything again and you can just access the data again with an easy `load`call.
> 
> ```` r
> #First, downloading each TS album
> ts1 <- genius_album(artist = "Taylor Swift", album = "Taylor Swift")%>%
>   mutate(album = "Taylor Swift")
> 
> ts2 <- genius_album(artist = "Taylor Swift", album = "Fearless")%>%
>   mutate(album = "Fearless")%>%
>   filter(!track_n %in% 14:16)
> 
> ts3 <- genius_album(artist = "Taylor Swift", album = "Speak Now")%>%
>   mutate(album = "Speak Now")
> 
> ts4 <- genius_album(artist = "Taylor Swift", album = "Red")%>%
>   mutate(album = "Red")
> 
> ts5 <- genius_album(artist = "Taylor Swift", album =  "1989")%>%
>   mutate(album = "1989")
> 
> ts6 <- genius_album(artist = "Taylor Swift", album = "Reputation")%>%
>   mutate(album = "Reputation")
> 
> #Putting all togetherin the same df
> tay <- rbind(ts1, ts2, ts3, ts4, ts5, ts6)
> 
> #Patterns to be removed from the data
> remove.list <- paste(c("Demo Recording", "Voice Memo", "Pop Version", "Acoustic Version"), collapse = '|')
> 
> #Applying the changes
> tay <- tay%>%
>  filter(!grepl(remove.list, track_title)) 
> 
> #Just in case: save!
> save(tay, file = "taytay.Rdata")
> ````
> 
> 
> ### 2. Tidying up the lyrics
> 
> So now we have a tibble with more that five thousand words featuring Taylor Swift lyrics _by line_. This does not seem very practical, but what would be? The [tidy text format](https://www.tidytextmining.com/tidytext.html) tells us that what we need is "one-token-per-document-per-row", with a token meaning, in our case, a word. Luckily, converting our songs to this format is quite easy thanks to the `unnest_tokens()` function from `tidytext`, that will take our data frame and split every sentence word by word:
> 
> ```` r
> #Tokenizing our data:
> tay_tok <- tay%>%
>   #word is the new column, lyric the column to retrieve the information from
>   unnest_tokens(word, lyric) 
> ````
> 
> Et voilà! Our new data, `tay_tok` now has almost 40k observations: one per word, while keeping all the information from the other columns. This is what it looks like:
> 
> ```` r
> tay_tok
> ````
> ![](https://i.ibb.co/LNW7fC2/image.png)
> 
> (Don't you just love [Tim McGraw?](https://www.youtube.com/watch?v=GkD20ajVxnY))
> 
> Notice that `unnest_tokens()` has also set all words to lowercase and removed punctuation. You can ask the function not to do it, but it's actually quite handy.
> 
> Now that the songs are in tidy format, we can start analyzing them.Let's do a simple `count()`call to see which are the most frequent words:
> 
> ````{r}
> tay_tok %>%
>   count(word, sort = TRUE) 
> ````
> ![](https://i.ibb.co/VJ5CmBP/image.png)
> 
> As you can see, this is not  very informative. It makes sense that Taylor's most repeated word is "you", but most of what we're getting here -pronouns, articles and prepositions- is quite information-empty. Of course  these words are very common, but that's not Taylor Swift-specific.
> 
> Luckily enough, `tidytext`provides a helpful tool to get rid of these kind of common terms, that in text analysis jargon are called _stop words_. We can take these kind of words by applying an `anti_join()` call with the `stopwords`dictionary from the package (of course you can use another dictionary if you want, this will depend on your needs). Let's do this and see what happens to our lyrics now:
> 
> ```` r
> tidy_taylor <- tay_tok %>%
>   anti_join(stop_words)
> 
> tidy_taylor%>%
>   count(word, sort = TRUE) 
> ````
> ![](https://i.ibb.co/gTb5Wzy/image.png)
> 
> This looks much better!! And with her upcoming album being called **Lover**, I can't deny that I'm quite excited to see that the word Taylor uses more frequently is not other than **love**. This deserves some over-the-top plotting to celebrate!
> 
> ### 3. Producing bar plots with an image as fill in ggplot: glitter time!
> 
> When I was first thinking about this plot, one of my first thoughts was "normal color palettes are not  enough for Taylor: I need glitter". During a quite desperate Google search, I got to know that [ggplot doesn't really like this](https://stackoverflow.com/questions/2895319/how-to-add-texture-to-fill-colors-in-ggplot2/2901210#2901210), but that there's a way  out. I won't get in detail of how this works because luckily for you and me, the `ggtextures` package has automated this process making it actually quite easy. 
> 
> All you need to do is find a texture you like -in my case, this was pink glitter, store it (doesn't matter whether you save it in your system or use a link from the internet) and provide it as an `image` argument to `geom_textured_col()`. With that and quite a bit of editing, you can get a nice glittery plot. 
> 
> All the code is commented, but these two elements may deserve some extra attention:
> 
> * Remember we loaded the `extrafont` package earlier on? This will allow us to use any font in our system in our plots, but we need to load them first with `loadfonts()`
> 
> * Notice that I'm filtering for songs that appear more than 70 times to get only the most frequent ones, and taking out the "di" and "oh" rows because well, those aren't actually words, right?
> 
> ````r 
> #Loading fonts
> loadfonts(device = "win")
> 
> #The pink pattern I will be using for the plot
> img = "pink.jpg"
> 
> #Let's plot!
> tidy_taylor %>%
>   count(word, sort = TRUE) %>%
>   #filtering to get only the information we want on the plot
>   filter(n > 70,
>          word != "di",
>          word != "ooh",
>          word != "ey")%>%
>   ggplot(aes(x = reorder(word, n), y = n)) +
>   geom_textured_col(image = img, color = "white", width = 0.8)+
>   geom_text(aes(label = reorder(word, n)), 
>             hjust = 1.2,vjust = 0.3, color = "white", 
>             size = 5,  family="Harlow Solid Italic")+
>   labs(y = "Number  of times mentioned", 
>        x = NULL,
>        title = "Most frequent words in Taylor Swift lyrics",
>        caption = "                                                                                                                                    Ariane Aumaitre - @ariamsita")+
>   coord_flip()+
>   ylim(c(0, 210))+ # I didn't want to have the bars covering the whole plotting area
>   theme_minimal()+
>   #now making more visually appealing
>   theme(plot.title = element_text( hjust = 0.5,vjust = 3, color = "maroon3", size = 14,  family="Forte"),
>         axis.text.y = element_blank(),
>         axis.text.x = element_text(size = 8, color = "grey40"),
>         axis.title.x = element_text(size = 10, color = "grey40"),
>         plot.caption = element_text(size = 7.5, color = "grey40"),
>         plot.margin=unit(c(2,1,1.5,1.2),"cm"))+
>   ggsave("song_count.png")
> 
> ````
> 
> ![](https://i.ibb.co/G5RpmyD/image.png)
> 
> ### 4. Comparing Taylor's albums with sentiment analysis
> 
> How can we go deeper into analyzing the content of Taylor's songs with R? An interesting technique to apply here is _sentiment analysis_. This can be done systematically with the `tidytext` package thanks to the `get_sentiments()`function, that provides 4 different sentiment lexicons that can be applied to our data with a simple `inner_join`.
> 
> The analysis performed here uses code inspired from [this section](https://www.tidytextmining.com/sentiment.html) and uses one of `get_sentiments()`lexicons (in this case I'm using the "bing" one but you can try others to see the differences) to compute the total amount of positive and negative sentiments in every Taylor song, by computing the difference between positive and negative ones.
> 
> 
> * First, I do an inner join with the lexicon selected -> This adds a column assigning whether a word has a positive or a negative sentiment related to it. Words that do not show up in the lexicon are discarded from the data.
> 
> * Then, I count the number of total positive and negative words showing up in a song with the `count()` call. I keep the information on album for easier plotting later.
> 
> * What comes next is just some data wrangling to create the desired output: a tibble containing a 'sentiment' column that assigns a sentiment value for each song. You can see the output below.
> 
> If this sounds a bit confusing to you, I highly recommend consulting the book linked above - I don't think I can provide a better explanation than what you'll find there.
> 
> ```` r
> tay_sentiment <- tidy_taylor%>%
>   inner_join(get_sentiments("bing"))%>% 
>   count(album, track_title, sentiment) %>%
>   spread(sentiment, n, fill = 0) %>%
>   mutate(sentiment = positive - negative)
> 
> tay_sentiment
> ````
> ![](https://i.ibb.co/GJqtSvx/image.png)
> 
> Now we have the data as we want it, we can go and do a nice bar plot of sentiments. Some things I've done to make it look nicer:
> 
> * We want albums to be in the right (chronological) order. Some factor reordering is needed for that.
> 
> * Colors, colors, colors. I have assigned to each album a color that I relate to it (still not convinced by my 1989 choice, but probably you don't care as much as I do about this)
> 
> * There's a lot of information showing up in this plot and I struggled a bit with finding the right values for ggsave in order for it to look proportionate. If this happens for you, just keep trying till you find what works for you""
> 
> * I'm setting the `scales` argument in `facet_wrap()` to free because sentiment values have very different ranges from one album to another, but this means paying the price of less comparability. Be aware of that!
> 
> ```` r
> #Right order for albums:
> tay_order <- c("Taylor Swift", "Fearless", "Speak Now", "Red", "1989", "Reputation")
> tay_sentiment$album <- factor(tay_sentiment$album, levels = tay_order)
> 
> #Plot:
> tay_sentiment%>%
>   ggplot(aes(reorder(track_title, sentiment), sentiment, fill = album)) +
>   geom_col(show.legend = FALSE) +
>   facet_wrap(~album, ncol = 3, scales = "free")+
>   scale_fill_manual(values = c("skyblue1", "lightgoldenrod1", "mediumorchid3", "red2", "plum1", "slategray"))+
>   labs(x = NULL,
>        y = "Sentiment",
>        title = "Taylor Swift's songs ranked by sentiment",
>        caption = "                                                                                                                                    Ariane Aumaitre - @ariamsita")+
>   theme_minimal()+
>   theme(plot.title = element_text(size = 13, hjust = 0.4, face = "bold"),
>         axis.title.y = element_text(hjust = 0.05, size = 7, color = "grey40", angle = 0),
>         axis.title.x =  element_text(size = 8, color = "grey40"),
>         axis.text.x = element_text(size = 6.5, color = "grey40"),
>         axis.text.y = element_text(size = 6.5, color = "grey40"), 
>         strip.text = element_text(size = 9, color = "grey40", face = "bold"),
>         plot.caption = element_text(size = 7.5, color = "grey40"))+
>   coord_flip()+
>   ggsave("sentiment.png", width = 10, height = 5.5)
> ````
> ![](https://i.ibb.co/SBf8vSq/image.png)
> 
> So what is this output telling us? First of all, that we should be careful with sentiment analysis and be always aware of the text we're analyzing. After all, it's rather weird to think of [Shake it off](https://www.youtube.com/watch?v=nfWlot6h_JM) as the most negative Taylor song... but it does repeat the word _hate_ a lot of times!! ("shake" and "break" are also assigned negative values). So check your lexicon well before jumping into conclusions!
> 
> ### 5. Checking the relationship among albums
> 
> One of the first questions that popped into my mind when I started analyzing Taylor's songs was how similar her albums were to one another, lyric-wise. After all, Taylor has changed a lot her style during the last 10 years, travelling from country to pop queen. But us Swifties tend to say that even if the sound changes, they're still the same kind of songs. Is this true?
> 
> To test this, I decided to compare word frequency in three of Taylor's albums: Fearless -full country-, Red -somewhere in the middle of country and pop- and 1989 -full pop-. Three of these albums deserved the Album of the Year Grammy, only two got it. Life is unfair, after all, but let's not get into that topic.
> 
> Once again, I can't recommend enough the [Text Mining with R](https://www.tidytextmining.com/) book if you want to learn how to do this properly. It's the best reference I've found.
> 
> To do this, I will create a data frame to get the frequency of words _by album_. Remember that the aim is to compare 1989 (the pop album) to Fearless and to Red. 
> 
> * First, I create a column of the frequency of every word in the data in every album
> * I then delete the "n" column to have cleaner data
> * Once I have this, I transform the data into wide format, to have one frequency column per album, and select only the albums I'm interested in
> * Finally, I transform again into long format, but only half way. This is not an ideal format nor do I love it, but it works for what I need, which is having one column with the 1989 frequency and then to 'joint columns' for Fearless and red, to be able to wrap by album and plot their frequencies in the other axis. It'll become clearer once plotted, anyway.
> 
> ```` r
> #First, factor reordering
> tidy_taylor$album <- factor(tidy_taylor$album, levels = tay_order)
> 
> #Frequency df
> tay_frequency <- tidy_taylor%>%
> count(album, word) %>%
>   group_by(album) %>%
>   mutate(proportion = n / sum(n)) %>% 
>   select(-n) %>% 
>   spread(album, proportion)%>%
>   select(-c(`Taylor Swift`, `Speak Now`, `Reputation`))%>%
>   gather(album, proportion, c(Fearless,Red))
> ````
> 
> There isn't that much to comment about the plot, but as usual, some remarks:
> 
> * This is quite a basic, nicely formatted (at least I hope so) combination of `geom_text()`and `geom_jitter()`. I'm using `geom_jitter` instead of `geom_point` because it looks more spread and thus nicer. No other reason.
> 
> * The dotted diagonal line is there to show where words would show up if used with the same frequency in the albums compared
> 
> * Note that I've logged the scales!! This allows to not have all words on top of another, but careful with interpretation
> 
> * Something quite annoying about `facet_wrap()` is that it won't let you have individual axis titles for your plots. I didn't want to do two separate plots so I went for the solution you can see below. I know it's not ideal, but I think it works.
> 
> ```` r
> tay_frequency%>%
> ggplot(aes(x = proportion, y = `1989`)) +
>   geom_abline(color = "maroon3", lty = 2) +
>   geom_jitter(alpha = 0.1, size = 2.5, 
>               width = 0.3, height = 0.3, color = "maroon3") +
>   geom_text(aes(label = word), check_overlap = TRUE, 
>             vjust = 1.5, color = "grey40") +
>   scale_x_log10(labels = percent_format()) +
>   scale_y_log10(labels = percent_format()) +
>   facet_wrap(~album, nrow = 1, strip.position = "bottom") +
>   coord_equal()+
>   theme_minimal()+
>   labs(x = "Word frequency",
>        y = "Word frequency 1989",
>        title = "Comparing Taylor Swift's albums",
>               caption = "                                                                                                                                    Ariane Aumaitre - @ariamsita")+
>    theme(plot.title = element_text(size = 13, hjust = 0.4, face = "bold"),
>         axis.title.y = element_text(hjust = 0.5, size =9 , color = "grey40"),
>         axis.title.x =  element_text(size = 8, color = "grey40"),
>         axis.text.x = element_text(size = 6.5, color = "grey40"),
>         axis.text.y = element_text(size = 6.5, color = "grey40"), 
>         strip.text = element_text(size = 9, color = "grey40", face = "bold"),
>         plot.caption = element_text(size = 7.5, color = "grey40"))+
>   ggsave("frequency.png")
> 
> ````
> ![](https://i.ibb.co/tH4k98s/image.png)
> 
> What can we say from this? Well, our girl does talk about love **a lot**, something that we already suspected. She also spent quite an important part of both Fearless and 1989 addressing a "baby", and a nice share of both Red and 1989 asking someone to stay (but that we knew right? What we don't know is whether [Stay, Stay, Stay](https://www.youtube.com/watch?v=s-mBGUWf4rg) and [All You Had To Do Was Stay](https://www.youtube.com/watch?v=Wo93B0s7QcY) were written for the same person).
> 
> Some more album-specific words arise if we look closer. "Shake" was obviously an important word for 1989, as were "bad" and "hate" - (am I getting some Bad Blood vibes from here?) But all in all, a lot of words seem to come together towards the diagonal line, which suggests that topics are not that different across albums, after all.
> 
> ### 6. Drawing a network map of Taylor Swift songs
> 
> We've come a long way until here. Lyrics have been downloaded, data has been cleaned, words have been counted, glitter has been created, and we have even done some sentiment analysis and looked at the correlation between different albums! But what if we wanted to do one last thing, let's say... a network map of how songs are related to each other?
> 
> Luckily for us, this is quite simple. I'll start by creating a last data frame, one that takes all songs and applies the `pairwise_cor()` function to all of them by comparing the words they contain. This will return a long df where we'll be able to find every pair of songs and their correlation. It's just two lines of code:
> 
> ```` r
> tay_cors <- tidy_taylor %>%
>   pairwise_cor(track_title, word, sort = TRUE)
> 
> tay_cors
> ````
> ![](https://i.ibb.co/myZ09qQ/image.png)
> 
> We could map this as a huge heatmap (I tried to do that), but with so many elements it's almost impossible to produce anything barely nice-looking.I think that a better solution is to go for a network map to see how songs are related to each other. This can be done with `ggraph` with the code below:
> 
> * I didn't want very low correlations to overcrowd the map and make it illegible so I filtered them out. The choice of .13 as my cutting point will be obvious to any Swiftie
> * The computer draws these king of plots differently every time you run them, so I'd suggest setting a seed if you want to be able to replicate!
> * Once again, it can get tricky with sizes and fitting everything to the plot area. Just try until you find what fits your data first
> 
> ```` r
> set.seed(123)
> 
> tay_cors %>%
>   filter(correlation > .13) %>%
>   graph_from_data_frame() %>%
>   ggraph(layout = "fr") +
>   geom_edge_link( show.legend = FALSE, aes(edge_alpha = correlation)) +
>   geom_node_point(color = "pink", size = 5) +
>   geom_node_text(aes(label = name), repel = TRUE, size = 3.5, color = "grey40") +
>   theme_void()+
>   ggsave("taymap.png", width = 15, height = 11)
> ````
> 
> ![](https://i.ibb.co/1TKpsGt/image.png)
> 
