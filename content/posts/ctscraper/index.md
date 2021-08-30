---
title: "Creating a price tracker"
date: 2021-08-29T11:29:25-05:00
description: Scraping product pages to track price changes
hero: /images/posts/writing-posts/Idea.svg
menu:
  sidebar:
    name: E-Store Scraping
    identifier: ctscraper
    weight: 100
---

All summer long I have wanted to purchase a new grill. I did a ton of research, looking for the perfect combination of size, features, and longevity, and what I quickly found was that my last qualifier - longevity - was one of the most expensive qualities to buy for. Quality grills are _expensive_ at current regular prices, perhaps I needed to wait for a sale. I just needed to find one.

Many people are familiar with sites like [camelcamelcamel](https://www.camelcamelcamel.com) that provide historic prices for Amazon products, but what if I don't want to buy from Amazon? There are surprisingly few options for monitoring prices from other retailers, especially those that serve the Canadian market. What I could find had limited functionality and wouldn't do what I needed. I decided to create something for myself.

## Scraping a single product page

Building a web scraper is a common task with numerous how-to documents and videos around the internet supporting a variety of languages. I thought this meant my project would be relatively easy to build and get running, if only for one or two retailers. Unfortunately, modern web technologies stepped in and proved to me that things were more complex than I had anticipated.

I started [here](https://blog.rstudio.com/2014/11/24/rvest-easy-web-scraping-with-r/), "rvest: Easy Web Scraping with R" the article is called. How could it not be easy? First, I pulled a product page using the `rvest` package.

```
library(rvest)
product_page <- read_html("https://www.canadiantire.ca/en/pdp/shark-rocket-ultra-light-corded-stick-vacuum-0436795p.html")
```

Unfortunately, this didn't work. The session hung indefinitely and I had to stop it manually. It appears that the Canadian Tire website I was attempting to scrape decides what to send based on your user agent.

### User Agents

What is a user agent? When you visit a website, the user agent is a string that your browser sends to the website to tell it what kind of software you're running. This allows the website to offer different results based on your equipment and software, helping to eliminate compatibility issues. I'm using `rvest` which is reflected in the user agent sent to the website, and the website sees this and returns an "Access Denied" error back to me. I guess they only want browsers they're familiar with visiting the store.

To get around this, I changed my code slightly. I manually defined a user agent string so the website in question would see my query as coming from a regular browser and respond accordingly.

```
product_response <- GET("https://www.canadiantire.ca/en/pdp/shark-rocket-ultra-light-corded-stick-vacuum-0436795p.html", user_agent("Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:91.0) Gecko/20100101 Firefox/91.0")
product_page <- read_html(product_response)
```

Perfect, now it worked, I have an html page loaded in to the variable `product_page`.

### Selecting a field

Next, I poked through the site using the [SelectorGadget](https://selectorgadget.com/) plugin for Chrome to identify the field I wanted to scrape. I found the class `.price__reg-value_multisku` contained the price, so I continued to follow the article and modified their code to pull the price from the html.

```
product_page %>%
  html_nodes(".price__reg-value_multisku") %>%
  html_text()

#> character(0)
```

What? I did everything right this time, what happened??

I have to admit, this one took me a while to figure out. I needed to save the html file and sift through it manually before it made sense. There was no `.price__reg-value_multisku` in the HTML I saved! I loaded the page again into my browser and opened the source code to compare, and it was there, but it wasn't in the code I downloaded with R. What was going on?

### The magic of PhantomJS

Reviewing the HTML source downloaded by R I noticed that there was very little product content included in it at all. It was like all the fields that mattered to me were excluded when I downloaded the HTML manually. Then I had a lightbulb moment, Javascript!

I realized that what was happening was when loading the page in my browser, some javascript was being run that updated the page with all the details I was looking for. When I download the HTML I get references to these scripts, but they're not being run so the information I'm looking for isn't there. Enter PhantomJS.

[PhantomJS](https://phantomjs.org/) is a "Scriptable Headless Browser." Basically what that means is that it does the things your regular web browser would do, but without displaying the results to a screen. You can then write scripts telling it to do things with these results, such as capturing a screenshot of the fully rendered page, or saving the final HTML once all javascript has run.

I found [this blog](https://velaco.github.io/how-to-scrape-data-from-javascript-websites-with-R/) that gave me an example using PhantomJS to render the page and store the result as a .html file. I ran this file, modified to use my own link, and saved the page to `ctire.html`.

```
system("phantomjs.exe ctscrape.js")
product_page <- read_html("ctire.html")
product_page %>%
  html_nodes(".price__reg-value_multisku") %>%
  html_text()

#> character(0)
```

Even after all this, I still have no price! What went wrong now?

### Making PhantomJS do my bidding

The tricky thing with PhantomJS is timing. It's not just about _what_ you want it to do, but _when_. It's not always easy to know when a web page has "finished loading," so determining when to save the resulting html file can be a challenge.

After some searching, I came across a solution on [Github](https://gist.github.com/th3d0g/11234133). By implementing this `waitFor` function, I could tell PhantomJS to wait for the price information to be rendered before saving the file. Below is the line that covers the element to wait for;

```
return $("#signin-dropdown").is(":visible");
```

In this example, you can see that we want the CSS element with the ID `signin-dropdown` to have the property of `visible`. For my purpose, I changed this to look for the class ID `.add-to-cart__button` and re-ran the test.

```
system("phantomjs.exe ctscrape.js")
product_page <- read_html("ctire.html")
product_page %>%
  html_nodes(".price__reg-value_multisku") %>%
  html_text()

#> "$149.99"
```

Success! I can now scrape the price given a Canadian Tire product URL.
