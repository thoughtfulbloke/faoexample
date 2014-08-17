faoexample
==========

Data from the FAO for demonstrating reading and merging datasets, to give some more practice to things learned in week three of the Getting and Cleaning Data course on Coursera.

The plan is people can fork this repo, and try and clean up the data in it using techniques seen in week 3. You could also browse other peoples forks to see how they did it. If you aren't comfortable working with Github yet, the code below includes steps for downloading the data files from this repository, and you could work with them just on your own machine.

I think Quiz 3, Questions 3-5 (and the week 4 revisiting of the data) are great. The are nifty examples of the kind of data people regularly have to work with. But you can't really discuss solutions to the quiz questions in the forums, so I thought it would be a good idea to pose an example challenge with some data that people can discuss code with in the forums. So I've got a few challenges, and people who want to practice can try those or give their own. The main goal is actually to clean up and combine the two data sets, which then makes answering the questions easy

This is data from faostat3.fao.org, and I will acknowledge I could have picked slightly easier arrangements to download the data as, but this is a genuine arrangement available for download and rather a good example for needing cleanup.

The appleorange file contains production volumes of apples and oranges, the stability file contains country stability indicators. Yes, we are comparing apples to oranges. 

So a couple of opening questions 

* do the top ten most stable countries grow more apples or oranges (in total volume) 
* what is the average amount of railroad in countries that grow more apples than oranges, and vis versa. 

So, entirely optional. Just a chance to practice and show off. 

If you see a question that illustrates a nice point about organising clean data, by all means pose it.

## Example Processing

If downloading the data files from the original repository I would use the code

For direct downloads from github (where I have just put it), this works pretty well (at least on my machine)

    library(downloader)
    download("https://raw.githubusercontent.com/thoughtfulbloke/faoexample/master/appleorange.csv", destfile="appleorange.csv")
    download("https://raw.githubusercontent.com/thoughtfulbloke/faoexample/master/stability.csv", destfile="stability.csv")


Here is how I would clean up the appleorange data before merging (this is just my way, there are lots of other approaches):

    ao <- read.csv("appleorange.csv")
    str(ao)

Well, that's a bit of a mess, let's be a bit slower about setting things up

    aoraw <- read.csv("appleorange.csv", stringsAsFactors=FALSE, header=FALSE)
    head(aoraw,10)
    tail(aoraw,10)

So the data is actually between rows 3 and 700

    aodata <- aoraw[3:700,]
    names(aodata) <- c("country", "countrynumber", "products", "productnumber", "tonnes", "year")
    aodata$countrynumber <- as.integer(aodata$countrynumber)

I could have assigned the names of the file to row 2 in the data with names(aodata) <- aoraw\[2,\] but thought giving my own would be better.

If you want to see how things could go horribly wrong doing a numeric conversion, compare what you get if you do a conversion to integer of the second column of ao (the orginal one on the automatic make it a factor settings).

Now there are rather a lot of lines saying Food supply quantity (tonnes) (tonnes) we do not need.

    fslines <- which(aodata$country == "Food supply quantity (tonnes) (tonnes)")
    aodata <- aodata[(-1 * fslines),]
    
I suppose I could have selected those observations != to Food etc., but I felt like using the negative index numbers to remove rows method.

Next, the tonnes needs serious find and replacing of text to get it into numbers only before conversion before numeric.

    aodata$tonnes <- gsub("\xca", "", aodata$tonnes)
    aodata$tonnes <- gsub(", tonnes \\(\\)", "", aodata$tonnes)
    aodata$tonnes <- as.numeric(aodata$tonnes)

and, for completeness, let's put in the year that was up the top of the file

    aodata$year <- 2009

Now people can have a go at cleaning up the stability data, and carefully see what happens when you try and merge the data with various settings.

For merging files together, the issue is always "what are the observations in the data". In this example, we would be wanting to merge the data together to form combined information about countries, however because we have looked at the data we have realised there multiple entries per country, so to merge to something that forms a combined set about countries we are going to need a unit of observation of individual countries in each data set (otherwise the two country records in on table combine with the three country records in the other table to give use 6 records per country & a horrible mess). Fortunately the week 3 lectures give us a lot of power in reorganising our data.

Now, I want to stress our data is tidy data at the moment. It is just that it is tidy data where each row is "a country and the fruit it grows" and for a merge we would be better to have (still) tidy data for each country. There are three options for the reorganising: 1 horrible, 1 reasonable, 1 great. The overall all goal is to turn it from tidy long data (several occurrences of a country) into tidy wide data (one instance of a country with several columns). 

The general rule that this falls under is "get all you data cleaning and arrangement sorted out before you do stuff to it, and then things are much easier".

### the horrible way, do not do this in the real world

You could subset the apples, then subset the oranges as a new column after it

    cleanao1 <- aodata[aodata$productnumber == 2617,]
    cleanao1$oranges <- aodata$tonnes[aodata$productnumber == 2611]
    cleanao1 <- cleanao1[,c(1,2,5,7)]
    names(cleanao1)[names(cleanao1) == "tonnes"] <- "apples"

The reason this is so very bad is that I have just added the orange subset onto the left of the apple subset without making any effort to check that the individual rows are for the same countries. In this case it happened to work, but that is more a matter of good luck. Imagine that there was no entry for oranges for Albania and no entry for apples for Venezuela. My lists of apples and oranges would both be the same length and R would stick them together, but the places in the list would be for different countries so my data would be hopelessly scrambled.

### It is OK to merge data with itself

If you make a subset of the apple tonnes, and a subset of the orange tonnes, you can use the merge function, joining them together by the country identifier, and you will get wide data.

    apples <- aodata[aodata$productnumber == 2617, c(1,2,5)]
    names(apples)[3] <- "apples"
    oranges <- aodata[aodata$productnumber == 2611, c(2,5)]
    names(oranges)[2] <- "oranges"
    cleanao2 <- merge(apples, oranges, by="countrynumber", all=TRUE)

In this case, all=TRUE was set so that if you only have an apple or and orange value then the country will still appear in the wide data list.

### reshape2 makes shifting around data easy

As a general rule of thumb, if you need to rearrange data, it is easy to do with reshape2 if you have had the practice in using it.

    library(reshape2)
    cleanao3 <- dcast(aodata[,c(1:3,5)], formula = country + countrynumber ~ products, value.var="tonnes")
    names(cleanao3)[3:4] <- c("apples", "oranges")

Now, if you have just run the above commands in RStudio, I want you to stop right now and look at the Environment tab. You will see clean1 has 174 rows while clean2 and clean3 have 175\. So what happened?

Remember that big warning about missing entries? Turns out there are some (I didn't actually know this when I picked the data sets). We can do something useful about seeing what is up with investigating those records with na values by going:

    cleanao2[!(complete.cases(cleanao2)),]
    cleanao3[!(complete.cases(cleanao3)),]

This shows potential problem entries. However it includes both countries which were in the dataset but with a blank entry (Chad, which because it is present but has no value is not the source of the problem) and Countries that had an entry entirely missing (Suriname). It does, in passing, also show a second problem with with the merged data- because we are only drawing our country names from entries with an apple row, Suriname got a blank for country name (as it did not have an apple entry). We could go to fancy lengths to fix this, but I'd rather just use reshape2\.

Coming back to the NULL value problem, another way of identifying those entries that have an apple or an orange entry, but not both, is to find all the entries with a single entry, which we can do by making a frequency table of country names and subsetting it to find the names that only occur once

    table(aodata$country)[table(aodata$country) == 1]

So from that we want to be really careful to check what happens with Myanmar (which has no orange entry) and Suriname (which has no apple entry).

The take-home of all that- reshape2 is a good, **reliable** way of reorganising our data.

Second take-home- after doing something dramatic to your data, you might want to actually check what the results were.

In both the tidying up of the data, and the reshaping of it, the same kind of rules and techniques apply to the stability data.

## Acknowledgements

As per the terms and conditions on the FAO website <http://www.fao.org/contact-us/terms/en/> I am acknowledging the source of the data in order to be able to freely use it.

_Except where otherwise indicated, content may be copied, printed and downloaded for private study, research and teaching purposes, or for use in non-commercial products or services, provided that appropriate acknowledgement of FAO as the source and copyright holder is given and that FAO's endorsement of users' views, products or services is not stated or implied in any way._


