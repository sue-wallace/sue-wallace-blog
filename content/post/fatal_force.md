---
autoThumbnailImage: false
categories:
- crosstalk
- flexdashboard
coverImage: /img/blue.jpg
date: "2019-07-28"
keywords:
- crosstalk
- flexdashboard
- DT
- leaflet
- Tidyverse
metaAlignment: center
tags:
- crosstalk
- flexdashboard
- leaflet
- shiny
thumbnailImage: /img/fatal-force.jpg
thumbnailImagePosition: left
title: Fatal Force with Cross Talk
---
Using crosstalk and html widgets such as flexdashboard, leaflet and DT to create an interactive application
<!--more-->

![Fatal Force](/img/fatal-force.jpg)

Crosstalk lets you use R Markdown to utilise multiple html widgets to develop an [interactive application](https://sue-wallace.github.io/fatal-force-with-crosstalk/). Unlike Shiny you don't need a server to host the application, and personally I find it a bit easier to use than Shiny. In this example I'm going to use data collated by the Washington Post which details fatal police shootings in the US from 2015 to 2018. 

If you're planning on using this blog to understand how crosstalk works I would recommend cloning the project from github and following through what's written in this guidence by running the code yourself in R. 

Otherwise this blog will give you some more general information on how crosstalk works!

<!-- toc --> 

# Fatal Force

[![Fatal Force](/img/fatal-force.jpg)](https://sue-wallace.github.io/fatal-force-with-crosstalk/)

## Getting started

[Crosstalk](https://rstudio.github.io/crosstalk/) developed by [Joe Cheng](https://twitter.com/jcheng?lang=en) extends html widgets with interactivity using sliders, radio buttons and filters.  

To get started you can create a new R project and install the github repo by typing the following into the command line:

```
git clone https://github.com/mrmoleje/fatal-force-with-crosstalk
```
You'll also need the following libaries:

```
dplyr, leaflet, DT, crosstalk, RColorBrewer , readr, ggplot2
```
## What's 'fatal force'

Police officers in the United States shoot and kill hundreds of people every year, far more than in other developed countries such as the UK, Germany and South Korea.

Black people are disproportionately affected by police shootings, with black people making up 23.1% of the proportion killed.

Between 2015 and 2017 nearly 3,000 people were shot and killed by police, and in 2018 so far 707 people have been killed.

Nearly 7% of the people shot and killed by police in 2017 were unarmed. Currently in 2018 this figure stands at 5.8%.

The data used here is taken from the Washington Post: Fatal Force website which details police shootings in the US from 2015-2018. You would need a subscription to look at the Washinton Post website but the data for the project is also available on [Kaggle](https://www.kaggle.com/washingtonpost/police-shootings).


## Tidy the data

If you've cloned the github repo as per the instructions above you'll see the data in csv format in the 'Data' folder. 

You'll notice that the data there is quite messy. There are also some joins that need to be made before proceeding. I'm going to use Tidyverse principles to clean the data and join it all into one data table. 

If you're not interested in how the data has been cleaned you might want to skip this step. To do this you can go ahead and use the `killing.csv` which is saved in the 'Data' folder and skip to the [Using Crosstalk](#usingcrosstalk) section of the blog.

First let's install the libraries we'll use to tidy the data.

```
library(dplyr) #data manipulation
library(readr) #read in and read out data
```
Now we can load the data sets that we want to tidy.

````
killing <- read_csv("Data/fatal-police-shootings-data.csv") #From Washinton Post

us_cities <- read_csv("Data/us_cities.csv")

states <- read_csv("Data/state_code.csv")
````

Next we'll change the 'Year' varaible so that it shows only the year of the incident, rather than the whole date.

```
killing %>%
  mutate(Year=format(as.Date(killing$date, format="%d/%m/%Y"),"%Y")) -> killing
```

Next we need to join the data which details the deaths with some information on states. At the same time we can rename some of the variables so that they look more user friendly in the application. 

```
dplyr::left_join(x = killing, 
                 y = states, 
                 by = "state") %>%
  dplyr::select(name, date, manner_of_death, armed_1, age, gender, race, city,
                signs_of_mental_illness, Year, state_name, flee, body_camera) %>% 
  dplyr::rename(Name = name, Date = date, `Manner of Death` = manner_of_death,
                Age = age, Gender = gender, Race = race, State = state_name, 
                City = city, `Signs of mental illness` = signs_of_mental_illness,
                Armed = armed_1) -> killing
```
We also need to join some data on cities with the states data. 

```
us_cities %>%
  dplyr::select(city, state_id, lat, lng) %>%
  dplyr::rename(City = city, state = state_id) -> us_cities

us_cities %>% 
  dplyr::left_join(
    x = us_cities,
    y = states,
    by =  "state"
  ) -> us_cities
```
Next we need to rename some of the values within the data. Again the purpose of this is so that the values are easy to understand in the application. 

```
killing$Gender[killing$Gender == "M"] <- "Male"
killing$Gender[killing$Gender == "F"] <- "Female"
killing$Race[killing$Race == "A"] <- "Asian"
killing$Race[killing$Race == "B"] <- "Black"
killing$Race[killing$Race == "H"] <- "Hispanic"
killing$Race[killing$Race == "W"] <- "White"
killing$Race[killing$Race == "O"] <- "Other"
killing$Race[killing$Race == "NA"] <- "Unknown"
killing$Race[killing$Race == "N"] <- "First Nations"
killing$Race[is.na(killing$Race)] <- "Unknown"
killing$`Manner of Death`[killing$`Manner of Death` == "shot"] <- "Shot"
killing$`Manner of Death`[killing$`Manner of Death` == "shot and Tasered"] <- "Shot and tasered"
killing$body_camera[killing$body_camera == "FALSE"] <- "No"
killing$body_camera[killing$body_camera == "TRUE"] <- "Yes"
killing$`Signs of mental illness`[killing$`Signs of mental illness` == "FALSE"] <- "No"
killing$`Signs of mental illness`[killing$`Signs of mental illness` == "TRUE"] <- "Yes"
killing$Armed[killing$Armed == "armed"] <- "Armed"
killing$Armed[killing$Armed == "unarmed"] <- "Unarmed"
killing$Armed[killing$Armed == "unknown"] <- "Unknown"
killing$Armed[killing$Armed == "undetermined"] <- "Undetermined"
killing$Armed[killing$Armed == "unknown weapon"] <- "Unknown Weapon"
```
Okay now we have all the data we need and it's in a decent user friendly format. We just need to join the data which details the deaths with the data detailing US cities and states. In the US there are some cities with the same name that are in different geographical location. We need to bear this in mind when joining the states data with the cities data. 

To compensate we need to use `paste` and `mutate` on the 'killing' data to develop a new variable which shows the city name followed by the state name. This means when we join to the killing data to the cities data there wont be any confusion between cities that are named the same but are in different locations. 

```
killing %>%
  mutate(city_state=paste(killing$City, killing$State, sep=", ")) %>%
  select(Name, Date, `Manner of Death`, Age, Gender, Race, Armed, State,
         City, Year, city_state, flee, body_camera, 
         `Signs of mental illness`) -> killing

us_cities %>%
  mutate(city_state=paste(us_cities$City, 
                          us_cities$state_name, sep=", ")) -> us_cities2
                          
```
Now we can join the cities data which we'll use to get the latitude and longitude information to the data which details the deaths. 

```
killing <- dplyr::left_join(
  x = killing,  # to this table...
  y = us_cities2,   # ...join this table
  by = "city_state"  # on this key
) 
```

Okay great! Finally for this section we need to write the data to a csv.

```
write_csv(killing, "Data/killing.csv")

```


## Using Crosstalk

Now we have our data in a nice, tidy format, and we have all the data we need to build the application using crosstalk!

Alright. To do this we need to start developing the application in Rmd. If you've not used [RMarkdown](https://rmarkdown.rstudio.com/) before then I would suggest having a read about that first before proceeding. RMarkdown is useful for developing reproducable documents in an array of formats. In this example we're using RMarkdown in conjunction with crosstalk to develop a html output. 

You don't need to know a lot of html to do this. I peronally don't know much html at all, but managed to get through by referencing other people's projects. In fact hopefully what I'm detailing in this blog with be enough information!

First we need to load some more libraries.

```
library(flexdashboard) - for creating the dashbaord
library(dplyr) - data wrangling
library(leaflet) - for mapping
library(DT) - for developing interactive data tables
library(crosstalk) - well, you guessed it, so we can use the crosstalk features
library(RColorBrewer) - for nice comlour templates
library(readr) - for reading data into and out of R
library(ggplot2) - for generating plots

```
Let's have a closer look at the the banner at the head of the RMarkdown document. Here you can add the title of your application and set some preferences. Here I'm using [Flexdashboard](https://rmarkdown.rstudio.com/flexdashboard/) to develop a, well a dashboard! Again I would suggest having a read of the flexdashboard documentation if you want to know a bit more about that. 

I'm using the 'cerulean' theme here taken from [Bootswatch](https://bootswatch.com/). If you've downloaded the project from Github then have a go at playing around with the themes and layouts using the '01.crosstalk.Rmd' document. 

```
---
title: "Police Shootings in the United States: 2015 - 2018"
output: 
  flexdashboard::flex_dashboard:
    orientation: columns
    vertical_layout: fill
    theme: cerulean
---
```
Next we can load in the data that we developed in the 'Tidy the data' section.

```
killing <- read_csv("Data/killing.csv") #As developed in '00.tidydata.R'

killing$Year <- as.character(killing$Year)

```

Now comes quite an important step which is crucial to understand how crosstalk and flexdashboard work. We need to wrap the data frame we have developed and pass it to a Crosstalk-compatible widget where a data frame would normally be expected. After doing this, everytime we want to call the data to be utilised in the application we will use this wrapped data.

```
sd <- SharedData$new(killing)
```
We're now going to asign a colour palette to the 'Race' variable in the data. This means that each 'Race' value will be assigned it's own colour. For this application I've used the 'Spectral' palette. 

```
pal <- colorFactor(
  palette = 'Spectral',
  domain = killing$Race
)
```

Next we're going to use `leaflet` to develop a map. So far leaflet is my favourite library in R. Namely because I love playing around with maps! What's important to note here is that the `sd` data frame is referenced, rather than the csv file!  

```
sd %>% 
  leaflet() %>%
  setView(lng = -100.94, lat = 38.94 , zoom = 3) %>% 
  addTiles() %>%
  addCircleMarkers(lng = ~lng, lat = ~lat, weight = 3, color = ~pal(Race),
                   stroke = TRUE, fillOpacity = 0.5, 
                   radius = ~ifelse(Race == "White", 6, 10),
                   popup = ~paste0("<h5>", killing$Name, "</h5>",
                                   
                                   
                                   "<table style='width:100%'>",
                                   
                                   "<tr>",
                                   "<tr>",
                                   "<th>Date</th>",
                                   "<th>", killing$Date, "</th>",
                                   "</tr>",
                                   
                                   "<tr>",
                                   "<tr>",
                                   "<th>Gender</th>",
                                   "<th>", killing$Gender, "</th>",
                                   "</tr>",
                                   
                                   "<tr>",
                                   "<tr>",
                                   "<th>Age</th>",
                                   "<th>", killing$Age, "</th>",
                                   "</tr>",
                                   
                                   "<tr>",
                                   "<tr>",
                                   "<th>Race</th>",
                                   "<th>", killing$Race, "</th>",
                                   "</tr>"
                   )) %>%
  addLegend("bottomleft", pal = pal, values = ~Race,
            title = "Race of victim",
            labFormat = labelFormat(prefix = "$"),
            opacity = 1
  ) -> map

```
Cool now we have a map to use in our application! As we're using leaflet here the map will be interactive. Next we're going to develop some charts to give some insight into the data we're using. The charts will be a static feature within the application. 

```

# Proportion killed by state

killing %>%
  dplyr::group_by(State) %>%
  #dplyr::filter(Year == "2017") %>%
  dplyr::summarise(count = n()) %>%
  dplyr::mutate(proportion = count/sum(count)*100) %>%
  dplyr::filter(proportion >= 2.5) %>%
  ggplot(aes(x = reorder(State, -proportion), y=proportion,
                          label = (round(proportion,1)))) +
  geom_col(position = "dodge", fill="#FDAE61") +
  geom_text(position = position_dodge(width = .9), vjust=-0.5)+
  theme_classic()+
  #ggtitle("Highest ranking states by proportion of total killed: 2017")+
  xlab("State")+
  ylab("Proportion") +
  theme(text = element_text(size=18), 
        axis.text.x = element_text(angle = 75, hjust = 1)) -> states_chart


killing %>%
  dplyr::group_by(Year, `Signs of mental illness`) %>%
  dplyr::summarise(count = n()) %>%
  dplyr::mutate(proportion = (count / sum(count)*100)) %>% 
  ggplot(aes(x = `Signs of mental illness`, y = proportion, fill = Year, label = (round(proportion,1))))+ 
  geom_col(position = "dodge") +
  scale_fill_brewer(palette = "Spectral", direction = -1)+
  geom_text(position = position_dodge(width = .9), vjust=-0.5)+
  theme_classic()+
  theme(text = element_text(size=18))+
  #ggtitle("Proption of fatalities showing signs of mental illness")+
  xlab("Victim showed signs of mental illness")+
  ylab("Proportion") -> mental_health_ch

#proportion by gender
 
killing %>%
  dplyr::filter(!is.na(Gender)) %>% #small proportion of unknown gender so take out
  dplyr::group_by(Year, Gender) %>%
  dplyr::summarise(count = n() ) %>%
  dplyr::mutate(proportion = count / sum(count)*100) %>% 
  ggplot(aes(reorder(x = Gender, -proportion), y = proportion, fill = Year, label = (round(proportion,1))))+ 
  geom_col(position = "dodge") +
  scale_fill_brewer(palette = "Spectral", direction = -1)+
  geom_text(position = position_dodge(width = .9), vjust=-0.5)+
  theme_classic()+
  theme(text = element_text(size=18))+
  #ggtitle("Proption of fatalities by gender")+
  xlab("Gender of victim")+
  ylab("Proportion") -> gen_chart

#proportion by race

killing %>%
  dplyr::group_by(Race) %>%
  dplyr::summarise(count = n() ) %>%
  dplyr::mutate(proportion = count / sum(count)*100) %>% 
  ggplot(aes(x=reorder(Race, -proportion), y=proportion, 
                      label = (round(proportion, 1))))+
  geom_col(position = "dodge", fill="#ABDDA4") +
  geom_text(position = position_dodge(width = .9), vjust=-0.5)+
  theme_classic()+
  theme(text = element_text(size=18))+
  #ggtitle("Proption of fatalities by race/ethnicity")+
  xlab("Ethnicity of victim")+
  ylab("Proportion") -> eth_ch

```

Finally we'll use 'DT' to make an interacrtive data table. This will detail all of the data in killing.csv in the application. Having an interactive table in your application lets the user play around with and apply filters to the underlying data. 

```
DT::datatable(
  filter = "top",
  head(killing_dt, 3000), 
  options = list(
  columnDefs = list(list(className = 'dt-center', targets = c(2:5))),
  pageLength = 100,
  lengthMenu = c(100, 250, 500, 1000),
  initComplete = JS(
    "function(settings, json) {",
    "$(this.api().table().header()).css({'background-color': '#FFFFE0', 'color': '#000'});",
    "}"),
  (searchHighlight = TRUE)
)) -> data_table
```
Okay now we have an interactive map which details where shootings took place, which can be filtered by race, gender, age and date. We also have some static graphs and an interactive table which shows the underlying data. 

Now we're going to take what we've developed above and put it into an application.

It's important that you have a think about how you want your application to look in terms of layout before you start developing it. How many pages do you want, and what content do you want on the pages? How do you want the pages to be laid out?

Each page of your application is defined by the title of the page followed by a row of equal symbols, as below. 

```
Overview
=====================================
```
Once you've decided the layout of your page you can define how wide you want the columns to be. For this application the map is pretty important, and looks better the more rectangular it is. So I've chosen a width of 550 for the column on the left of the page. The total width of the page is 1000, so this leaves 450 for the column on the right side. 

```
Column {data-width=550}
-----------------------------------------------------------------------
```
You can then assign headings within your page. 

```
### Map
```
And then call the map. When developing the map I assigned it to 'map' so this is what I'll use to call it into the application. You also need to tell flexdashboard what kind of widget you're putting into the application. In this case I used `{r map}`.


Okay great that's our map set up. Next we'll define the right hand colummn. 

```
Column {data-width=450}
-----------------------------------------------------------------------
```
On the right side of the application I want to apply some filters to interact with the map and to narrow down teh data to make it easier to interpret. if you've used [Shiny](https://shiny.rstudio.com/) before then this concept will be familiar, however crosstalk woks in static html documents.

So for example if the user wants to look at shootings that occured in 2015, they can apply a filter to that effect, and the map will update to show shootings that occured on that date.

For me, this is where crosstalk becomes really useful. There are lots of different filters that you can apply such as check boxes and sliders, and it's really quite simple to add interactivity into a dashboard as you'll see below. 

First let's give the section a title. 

```
### Filters
```

We then need to define what the column is going to detail in r.

```
{r filters}
```

We can then use our `sd` dataframe, which is being used in the map to apply the filters. The first chunk of code here defines what kind of interactivity is required by using `filter_select`. The label id is set to "Race", this is the title of the filter. The SharedData is then called by using `=sd`. Lastly the `sd` data frame is grouped by Race using `~Race`. 

```
filter_select(
  id = "Race",
  label = "Race",
  sharedData = sd,
  group = ~Race
)

filter_select(
  id = "State",
  label = "State",
  sharedData = sd,
  group = ~State
)

bscols(
  filter_checkbox(
    id = "Armed",
    label = "Armed",
    sharedData = sd,
    group = ~Armed
  ),
  filter_checkbox(
    id = "Year",
    label = "Year of incident",
    sharedData = sd,
    group = ~Year
  ),
  filter_checkbox(
    id = "Gender",
    label = "Gender",
    sharedData = sd,
    group = ~Gender
  ), 
   filter_checkbox(
    id = "flee",
    label = "Fleeing",
    sharedData = sd,
    group = ~`flee`
  ))
```
Now the map and filters are set I'm going to use the bottom right panel to add in a static chart. We've already defined the width of the right panel, so we don't need to do this again. 

Define the title. 

```
### Police shootings by state: 2015 - 2018
```

Set the height and width of the chart, and call the chart to the application.

```
{r fig.width=8, fig.height=6}

states_chart

```
So that's it, we have our first page set out! You should be able to knit to flexdashboard and run the application. 

Just lastly I'm going to highlight how to add the interactive table we created earlier using the `DT` package. 

Simply start a new page in the application within your RMarkdown document.

```
Data
===================================== 
```

Now call the data table.

```
[r}
data_table

```

So that's it, we have an up and running html application! In developing applications in R I started with Shiny and the had a go using crosstalk using flexdashboard. Personally I found flexdashboard easier to get to grips with than Shiny. For that reason I think it's good to start out using crosstalk, and then move onto trying Shiny. 

That said it is possible to use [Shiny within crosstalk!](https://rstudio.github.io/crosstalk/shiny.html)



